# Expand Hub Template Access on Policies

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
      [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

Open Cluster Management policy hub templates are limited to referencing objects in the same namespace as the root
policy. The proposal is to add an additional field of `spec.hubTemplateOptions.serviceAccountName` that must reference a
service account in the same namespace as the root policy. The hub templates would execute using that service account.
The user is responsible for providing `list` and `watch` permissions on the objects that the hub templates reference for
that service account.

## Motivation

An Open Cluster Management policy can contain hub templates that get executed as replicated policies are created in each
managed cluster namespace on the hub. To prevent privilege escalation and data leakage, there is a current restriction
that only the `ManagedCluster` object and objects in the same namespace as the root policy can be referenced. This is
limiting in situations such as copying a `Secret` from another namespace to the managed cluster. The current workaround
is to have a policy applied to `local-cluster` (Hub) that copies the `Secret` to the root policy namespace. Workarounds
for cluster-scoped objects or objects that have side effects are more complicated.

### Goals

1. A user can specify a service account for hub templates to access objects outside of the root policy namespace.
1. The current event-driven nature of hub templates is retained with a custom service account.
1. Tokens for the service account have an expiration of 2 hours.

### Non-Goals

1. Allowing service accounts outside of the root policy namespace.

## Proposal

### Design

Define a new field at `spec.hubTemplateOptions.serviceAccountName` to specify a service account that exists in the root
policy namespace. When specified, the service account is used instead of the service account of the Governance Policy
Propagator for hub template execution. Additionally, no namespace or cluster-scoped restrictions will be enforced by the
Governance Policy Propagator, and instead, it will rely on the Kubernetes API performing authorization checks of the
service account. Note that due to the event-driven nature of hub templates, the service account must have `list` and
`watch` permissions for the objects that the hub templates reference.

Note that for backwards compatibility, if the namespace is not specified in a hub template object lookup, the root
policy namespace will be the default, however, the service account must still have the appropriate permissions in the
root policy namespace to perform the lookup.

See an example of such a policy below:

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: test-config
  namespace: test-policies
spec:
  disabled: false
  remediationAction: enforce
  hubTemplateOptions:
    serviceAccountName: testsa
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: test-config
        spec:
          remediationAction: enforce
          severity: high
          object-templates:
            - complianceType: musthave
              objectDefinition:
                apiVersion: v1
                data:
                  field1: '{{hub fromConfigMap "some-other-namespace" "test-configmap" "test-field1" hub}}'
                  field2: '{{hub fromConfigMap "some-other-namespace" "test-configmap" "test-field1" hub}}'
                kind: ConfigMap
                metadata:
                  name: my-configmap
                  namespace: test
```

### User Stories

#### Story 1

As a policy user, I would like to specify hub templates that reference objects outside of the root policy namespace so
that I can stop defining additional policies on `local-cluster` to sync the required information to the root policy
namespace.

### Implementation Details/Notes/Constraints

Although the user experience is rather simple, there's complexity in the implementation. One aspect is that the number
of service accounts is not known at startup and can constantly change. It is also a bit complex to prevent duplication
of Kubernetes API watches, underlying TCP connections, and tokens if multiple policies reference the same service
account.

Currently, the Governance Policy Propagator only uses a single `TemplateResolver` client from the
[go-template-utils](https://github.com/stolostron/go-template-utils) library and sets restrictions based on the root
policy namespace. That will still remain the default, however, we'll need exactly one `TemplateResolver` client per
service account. To do this, I recommend having a new `sync.Map` field named `TemplateResolvers` on the
`ReplicatedPolicyReconciler` struct and each key would be a `NamespacedName` from `k8s.io/apimachinery/pkg/types`
representing the service account. Each value would be an instance of the following struct:

```go
type templateResolverWithCancel struct {
	lock           *sync.RWMutex
	cancel         context.CancelFunc
	resolver       *templates.TemplateResolver
	dynamicWatcher k8sdepwatches.DynamicWatcher
}
```

When the replicated policy controller with the Governance Policy Propagator encounters a
`spec.hubTemplateOptions.serviceAccountName` field, it'll perform an operation similar to this to either create the
`TemplateResolver` if it's the first reconcile for that `serviceAccountName` or to use the preexisting one:

```go
var templateResolver *templates.TemplateResolver

nsName := types.NamespacedName{
	Namespace: rootPlc.Namespace, Name: replicatedPlc.Spec.HubTemplateOptions.ServiceAccountName,
}

resolver := &templateResolverWithCancel{
	lock: &sync.RWMutex{},
}

resolver.lock.Lock()

oldResolver, loaded := r.TemplateResolvers.LoadOrStore(nsName, resolver)
if loaded {
	resolver.lock.Unlock()

	oldResolverTyped := oldResolver.(*templateResolverWithCancel)
	oldResolverTyped.lock.RLock()

	// Handle the case where resolver is getting cleaned up while the lock is held.
	// Just try again in this case.
	if oldResolverTyped.resolver == nil {
		oldResolverTyped.lock.RUnlock()

		// TODO: Write code to handle repeating the above code block
	}

	defer oldResolverTyped.lock.RUnlock()

	templateResolver = oldResolverTyped.resolver

	break
}

// TODO: Code to get a token and instantiate a TemplateResolver instance for the service account and
// set the values on `resolver`.
```

There should be a goroutine that periodically cleans up entries in `TemplateResolvers` that are for service accounts no
longer referenced by any root policy. This is simpler than managing a reference count for each service account.

Another complication is that when the policy's service account name changes, the watches need to be removed for the
replicated policy on the previous service account `TemplateResolver`. The suggestion is to have another `sync.Map`
called `policyToServiceAccount` on the `ReplicatedPolicyController` struct to keep track of the service account the
replicated policy had. The keys are the `ObjectIdentifier` type from the
[kubernetes-dependency-watches](https://github.com/stolostron/kubernetes-dependency-watches) library representing the
replicated policy. The values are `types.NamespacedName` values of the service account used. Note that for the default
template resolver, we will use the special name of `<default>` with an empty namespace. In the `Reconcile` method, if
the service account changes, the `DynamicWatcher` of the old service account is retrieved from `r.TemplateResolvers` and
`resolver.DynamicWatcher.RemoveWatcher` is called. This should be done regardless of if hub templates are detected since
the update of the service account name may have removed hub templates as well. Note that this `policyToServiceAccount`
map can also be used to clean up watches when a replicated policy is deleted since the original replicated policy spec
is no longer available.

The other complication is the service account must be periodically refreshed before expiration. This can be handled when
the token is generated using the
[TokenRequest API](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/) and
launching a goroutine that replaces the token before expiration. Note that if the token is provided in a token file,
client-go will periodically check for a new token in the file and use that for API requests. Taking advantage of this
prevents the Governance Policy Propagator from needing to recreate the `TemplateResolver` instance when the token
changes. See an example function to get a service account token that gets automatically refreshed:

```go
func (r *ReplicatedPolicyReconciler) getToken(ctx context.Context, namespace string, saName string) (string, error) {
	var expiration int64 = 7200

	req := &authv1.TokenRequest{
		Spec: authv1.TokenRequestSpec{
			// TODO: Is this always the correct value?
			Audiences: []string{"https://kubernetes.default.svc.cluster.local"},
			// 2 hours
			ExpirationSeconds: &expiration,
		},
	}

	returnedTokenReq, err := r.HubClient.CoreV1().ServiceAccounts(namespace).CreateToken(
		ctx, saName, req, metav1.CreateOptions{},
	)
	if err != nil {
		return "", err
	}

	tokenFile, err := os.CreateTemp("", fmt.Sprintf("token-%s.%s-*", namespace, saName))
	if err != nil {
		return "", err
	}

	_, err = tokenFile.Write([]byte(returnedTokenReq.Status.Token))
	if err != nil {
		return "", err
	}

	tokenFilePath, err := filepath.Abs(tokenFile.Name())
	if err != nil {
		return "", err
	}

	expirationTimestamp := returnedTokenReq.Status.ExpirationTimestamp

	go func() {
		defer func() {
			err := os.Remove(tokenFilePath)
			if err != nil {
				log.Error(err, "Failed to clean up the service account token file", "serviceAccount", saName, "namespace", namespace, "path", tokenFilePath)
			}
		}()

		for {
			deadlineCtx, deadlineCtxCancel := context.WithDeadline(ctx, expirationTimestamp.Add(-10*time.Minute))

			<-deadlineCtx.Done()
			if !errors.Is(deadlineCtx.Err(), context.DeadlineExceeded) {
				// This really does nothing but this satisfies the linter that cancel is called.
				deadlineCtxCancel()

				return
			}

			returnedTokenReq, err := r.HubClient.CoreV1().ServiceAccounts(namespace).CreateToken(
				ctx, saName, req, metav1.CreateOptions{},
			)
			if err != nil {
				log.Error(err, "Failed to renew the token. Will retry in 5 seconds.", "serviceAccount", saName, "namespace", namespace)

				time.Sleep(5 * time.Second)

				continue
			}

			_, err = tokenFile.Write([]byte(returnedTokenReq.Status.Token))
			if err != nil {
				log.Error(err, "Failed to write the token. Will retry in 5 seconds.", "serviceAccount", saName, "namespace", namespace, "path", tokenFilePath)

				time.Sleep(5 * time.Second)

				continue
			}

			expirationTimestamp = returnedTokenReq.Status.ExpirationTimestamp

			// This really does nothing but this satisfies the linter that cancel is called.
			deadlineCtxCancel()
		}
	}()

	return tokenFilePath, nil
}
```

### Risks and Mitigation

### Open Questions [optional]

None yet.

### Test Plan

**Note:** _Section not required until targeted at a release._

### Graduation Criteria

N/A

### Upgrade / Downgrade Strategy

Downgrades would cause the `spec.serviceAccountName` field to be removed and fall back to the default service account.

### Version Skew Strategy

## Implementation History

N/A

## Drawbacks

- Complex implementation

## Alternatives

One could use `SubjectAccessReview` to see if the service account has access and leverage the existing default
`TemplateResolver` but this is expensive on the API server, it would not take into account group membership, and it
would not be responsive to RBAC changes on the service account.

## Infrastructure Needed [optional]

N/A
