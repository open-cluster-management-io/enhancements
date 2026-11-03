# Provide managed-cluster credentials to hosted addons

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
[website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

A hosted addon runs on a hosting cluster but often needs to call the API server of a different
managed cluster. The addon framework gives the hosted workload a stable kubeconfig Secret name, but
it does not define how a scoped credential will be placed in that Secret. Addons that need this
access must otherwise carry their own credential provisioner or receive a broader bootstrap
credential than the workload needs.

This proposal will add a creation-time Go option to AgentAddon. An addon that opts in will declare
one target ServiceAccount and a requested token lifetime. The framework will
send that declaration to the hosting cluster in the addon's ManifestWork. The hosting cluster's
klusterlet operator will authenticate the request through its AppliedManifestWork, use its existing
managed-cluster connection to issue a short-lived TokenRequest for the declared ServiceAccount, and
write a kubeconfig Secret in the addon's isolated runtime namespace.

The generated kubeconfig will use tokenFile, so routine token replacement will not require a pod
restart. ManagedClusterAddOn status will report whether the credential is ready and will identify
the exact Secret through RelatedObjects. The entire feature will be optional. Existing addon types,
existing ManagedClusterAddOn objects, Default-mode deployments, and hosted addons that do not
select the option will keep their existing behavior and require no migration.

### Terms

- **Target cluster**: the managed cluster whose API the hosted addon needs to call.
- **Hosting cluster**: the managed cluster where the hosted addon's workload runs.
- **Runtime namespace**: the deterministic, per-addon-instance namespace on the hosting cluster
  defined by [191-hosted-addon-runtime-namespace](../191-hosted-addon-runtime-namespace/README.md).
- **Credential request**: the fixed-name ConfigMap delivered to the runtime namespace through
  ManifestWork.
- **Target ServiceAccount**: the ServiceAccount on the target cluster whose authorization will be
  carried by the generated token.
- **Credential Secret**: the stable Secret in the runtime namespace containing a generated
  kubeconfig and its sibling token file.

## Motivation

[Issue #190](https://github.com/open-cluster-management-io/enhancements/issues/190) asks OCM to
provide a managed-cluster credential to hosted addons. A hosted workload cannot use its hosting
cluster's in-cluster identity to reach the target cluster. Giving the workload the klusterlet
operator's external managed-cluster kubeconfig would solve connectivity, but would also give the
workload substantially more authority than its addon ServiceAccount needs.

The required operation crosses two control planes:

1. The hub-side addon framework knows the addon instance, its target ServiceAccount, and the
   ManifestWork that deploys the hosted workload.
2. The hosting-cluster klusterlet operator has the network path and bootstrap connection needed to
   request a token from the target cluster.

Neither side alone has enough context. The framework must make an authenticated declaration, and
the klusterlet operator must turn only that declaration into a scoped, short-lived credential.

Token rotation also needs a stable consumption contract. Replacing an embedded bearer token in a
kubeconfig does not update a client that parsed that kubeconfig at startup. A kubeconfig auth entry
using tokenFile lets client-go periodically read the token from a separate file. Updating the
Secret's token key will then update the mounted file without changing the kubeconfig or restarting
the addon pod.

### Goals

- Add a fully optional, creation-time Go AgentAddon declaration for one managed-cluster
  ServiceAccount credential.
- Keep the klusterlet operator's broad managed-cluster connection out of the hosted workload.
- Mint a short-lived TokenRequest token carrying only the declared ServiceAccount's permissions.
- Present the credential under a stable Secret name in a deterministic, isolated runtime namespace.
- Rotate the token before expiry without requiring an addon pod restart.
- Detect target ServiceAccount UID and managed-cluster connection changes, and clear the published
  token before verifying and issuing a replacement.
- Authenticate every privileged request field through ManifestWork and AppliedManifestWork
  identity, ownership, resource UID, and content hash checks.
- Surface readiness or failure through a ManagedClusterCredentialReady condition and an exact
  Secret RelatedObject.
- Remove the credential through normal Kubernetes ownership when its addon instance is deleted.
- Preserve all behavior and defaults for every addon that does not select the option.

### Non-Goals

- Creating the target ServiceAccount or granting it RBAC. The addon will continue to declare those
  managed-cluster resources in its own manifests.
- Providing a broad administrator credential to the hosted workload.
- Changing Default-mode addon registration or credential behavior.
- Adding an AddOnTemplate API in the first version. This proposal covers Go AgentAddon
  implementations.
- Supporting more than one managed-cluster credential per addon instance.
- Supporting custom token audiences, client certificates, exec plugins, or arbitrary authentication
  plugins in the generated kubeconfig.
- Mutating the selected opt-in or credential declaration after an addon instance is created. The
  effective declaration will be immutable for that instance.
- Automatically reloading a changed API server address or CA bundle inside an arbitrary running
  addon process. Routine token rotation will be restart-free; connection reload behavior remains an
  addon responsibility.

## Proposal

### User Stories

#### Addon developer

I want to declare the managed-cluster ServiceAccount my hosted addon needs and mount one stable
Secret, without writing a second controller that handles TokenRequest, rotation, and cleanup.

#### Hub administrator

I want a hosted addon to receive only its declared managed-cluster identity, and I want
ManagedClusterAddOn status to tell me whether that credential is usable.

#### Security administrator

I want the broad external managed-cluster kubeconfig to remain inside the klusterlet operator, and
I want a runtime namespace user to be unable to change which ServiceAccount receives a token.

#### Operator who does not select the feature

I want upgrades to preserve every existing default and require no migration.

### Risks and Mitigations

**A runtime namespace user could forge a request for a more privileged ServiceAccount.** The
klusterlet operator will not trust the ConfigMap by name, label, namespace, or content alone. It
will require an AppliedManifestWork owner with the same UID recorded in the ConfigMap owner
reference, verify that the AppliedManifestWork status records the exact ConfigMap UID, verify the
owning ManifestWork identity, and recompute a hash over every request field. The hosted workload's
RBAC will not permit create, update, patch, or delete on the request ConfigMap.

**A runtime namespace user could forge Ready feedback.** Credential status will be written as
annotations on the request ConfigMap, so the workload must not have mutation rights to that object.
The hub will accept Ready, Pending, or Failed only when the feedback includes the exact authenticated
request hash expected for the current ManifestWork generation.

**A pre-existing Secret could be overwritten.** The controller will never adopt a Secret merely
because its namespace and name match. It will update or delete only a Secret whose controller owner
is the authenticated request ConfigMap and whose credential marker and immutable request hash match.
A collision with any other Secret will produce Failed status.

**The klusterlet operator needs Secret permissions across runtime namespaces.** Today its ClusterRole
can create any Secret but can read, update, or delete only a fixed list of Secret names. The credential
Secret will therefore have one fixed name, `managed-cluster-kubeconfig`, in every runtime namespace, so
that list gains one name instead of the rule covering every Secret on the hosting cluster. Request
authentication, per-instance namespaces, strict owner checks, and non-adoption of collisions will
constrain its use.
The klusterlet operator remains the trusted component that already holds the broader target-cluster
connection.

**Clearing a Secret does not revoke tokens already read by clients.** An empty token file is a read
error, and [client-go continues using its last successfully read token](https://github.com/kubernetes/client-go/blob/v0.36.4/transport/token_source.go#L180-L189).
Cached or copied tokens can remain usable until the target API server rejects them or they expire.
Token lifetimes will be bounded to at least 10 minutes and at most 24 hours, with early renewal
managed by the framework. Target ServiceAccount RBAC should remain the primary authorization
boundary. Applications that need to stop requests when the credential becomes not ready must
handle that separately; tokenFile polling does not provide that guarantee.

**A changed target identity or connection could leave an old identity usable.** If the observed
target ServiceAccount UID or managed-cluster connection identity changes, the controller will first
blank the owned Secret's token and remove its expiration. It will mint a replacement only after the
new identity and connection have been verified. A routine refresh failure with unchanged identity
may retain a still-valid short-lived token until its natural expiration.

**Secret-volume propagation is asynchronous.** The token will be updated atomically in the Secret,
but kubelet propagation and client-go token-file polling are not instantaneous. The framework's
refresh policy must leave enough time for both. The minimum supported token lifetime will be 10 minutes.

## Design Details

This design will have four cooperating pieces:

1. The addon type will opt in through AgentAddonOptions.
2. The addon framework will place an authenticated credential request in the hosted runtime
   ManifestWork.
3. The hosting-cluster klusterlet operator will validate the request and reconcile a scoped
   TokenRequest credential.
4. ManifestWork feedback will drive ManagedClusterAddOn condition and RelatedObject status.

The option and all of its selected values will be creation-time configuration. An addon type that
does not set the option will generate no request and observe no behavior change. An addon developer
must not change the effective declaration while ManagedClusterAddOn instances created with it
exist. Behavior after changing that declaration is outside this proposal.

### 1. Opt-in AgentAddon declaration

AgentAddonOptions will gain an optional managed-cluster credential declaration:

~~~go
type AgentAddonOptions struct {
    HostedModeEnabled       bool
    HostingNamespaceEnabled bool

    ManagedClusterCredential *ManagedClusterCredentialOption
}

type ManagedClusterCredentialOption struct {
    ServiceAccountNamespace string
    ServiceAccountName      string
    TokenFile               string
    ExpirationSeconds       int64
}
~~~

Addon factory users will be able to select it with
WithManagedClusterCredentialOption. The option will be valid only together with HostedModeEnabled
and HostingNamespaceEnabled. The isolated namespace requirement is mandatory because a shared
hosting namespace would let unrelated addon instances mount, collide with, or attempt to mutate
each other's credential objects.

The fields will have these meanings:

| Field | Required | Meaning |
|---|---|---|
| ServiceAccountNamespace | No | Target-cluster ServiceAccount namespace. Empty means the addon's managed-cluster install namespace. |
| ServiceAccountName | Yes | Target-cluster ServiceAccount name. |
| TokenFile | Yes | Path used by the generated kubeconfig to read the mounted token key. |
| ExpirationSeconds | Yes | Requested TokenRequest lifetime, from 600 through 86400 seconds. |

TokenFile is a path inside the addon container after the Secret is mounted, not a path on the
klusterlet operator host. For example, a Secret mounted at /managed/config can use
/managed/config/token.

The framework will choose when to refresh the credential. Addon authors will not need to specify a
second interval for rotation; that policy will account for the actual expiration, Secret-volume
propagation, client-go token-file polling, and retries.

This will remain a Go configuration surface. It will not add a mutable field to
ManagedClusterAddOn, ClusterManagementAddOn, or AddOnTemplate. An addon type must select the option
before creating addon instances that depend on it and must keep the selected values stable for
those instances.

### 2. Isolated runtime namespace dependency

The framework will require the runtime namespace contract from
[191-hosted-addon-runtime-namespace](../191-hosted-addon-runtime-namespace/README.md). For a
ManagedClusterAddOn identified by target-cluster namespace and addon name, that contract will
produce a deterministic DNS-1123 namespace:

~~~text
open-cluster-management-<sanitized-addon-prefix>-<instance-id>
~~~

The instance ID will be the first 20 hexadecimal characters of a domain-separated SHA-256 digest
of the ManagedClusterAddOn namespace and name. The human-readable addon prefix will be sanitized
and truncated so the complete namespace is no longer than 63 characters.

ManagedClusterAddOn status.hostingNamespace will publish the resolved namespace. The credential
request, credential Secret, hosted workload, hub kubeconfig Secret, and hosted addon Lease will use
that same runtime namespace. Credential provisioning will not fall back to the legacy shared
namespace when status.hostingNamespace is empty.

The instance digest prevents two target clusters with the same addon name from sharing credential
objects on one hosting cluster. Sanitization and truncation will not be relied on for uniqueness.

### 3. Credential request

After the isolated runtime Namespace claim is current and safely applied as defined by
KEP 191, the framework will add one fixed-name ConfigMap to the ManifestWork that carries the hosted
runtime manifests. The runtime Work, including this request, will remain gated while that Namespace
claim is pending or conflicting:

~~~yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: managed-cluster-credential-request
  namespace: <runtime-namespace>
  labels:
    addon.open-cluster-management.io/managed-credential-request: "true"
    addon.open-cluster-management.io/managed-credential-request-hash: <request-hash>
data:
  targetCluster: <managed-cluster-name>
  addonName: <addon-name>
  serviceAccountNamespace: <target-serviceaccount-namespace>
  serviceAccountName: <target-serviceaccount-name>
  tokenFile: <mounted-token-path>
  expirationSeconds: <base-10-seconds>
~~~

The name managed-cluster-credential-request and the six data keys will be fixed. Unknown data
keys will never influence a credential decision. The request hash will be a Kubernetes label-safe
value made from SHA-256. It will start with the domain bytes
`open-cluster-management.io/addon-credential-request/v1\x00`. For each fixed key in lexical order,
the hash input will append the key and its value, each preceded by its byte length encoded as an
unsigned 64-bit big-endian integer. The label value will be `v1-` followed by the first 40 lowercase
hexadecimal digest characters. Missing and empty required values will be rejected.

The framework will place the same marker and hash labels on the ManifestWork. The hosting runtime
Work name will follow this identity convention:

~~~text
addon-<addon-name>-deploy-hosting-<target-cluster>-<suffix>
~~~

It will also carry the standard labels:

~~~text
open-cluster-management.io/addon-name=<addon-name>
open-cluster-management.io/addon-namespace=<target-cluster>
~~~

The work agent will apply the ConfigMap and create an AppliedManifestWork on the hosting cluster.
The ManifestWork controller will keep the Work labels synchronized to that AppliedManifestWork.
The work agent's normal ownership will add the AppliedManifestWork owner reference to the
ConfigMap and record applied resource identity in AppliedManifestWork status.

The request ConfigMap will use non-forcing server-side apply with an addon-instance-specific field
manager. Conflicting data held by another field manager will fail instead of being forced.

### 4. Request authentication

The klusterlet operator will watch ConfigMaps bearing the credential request marker, but the marker
will only select queue events. It will not authorize a request. Before reading any field as a
privileged instruction, the controller will require all of the following:

1. The ConfigMap name is exactly managed-cluster-credential-request.
2. The marker and request hash labels are present.
3. Exactly one owner reference names an AppliedManifestWork, with a non-empty name and UID.
4. The referenced AppliedManifestWork exists, has the same UID, and is not being deleted.
5. AppliedManifestWork status.appliedResources contains the exact ConfigMap group, resource,
   namespace, name, and UID.
6. The AppliedManifestWork carries the credential marker and the same request hash.
7. Its standard addon name and addon namespace labels match the request identity.
8. Every fixed request data field is present, and recomputing the request hash produces the label
   value.
9. The ConfigMap namespace equals the deterministic runtime namespace for that target cluster and
   addon name.

The Work name will remain useful for diagnostics, but its prefix will not authorize a credential
request. The ownership chain, content hash, and addon identity provide that binding.

An AppliedManifestWork that has not yet reported the request UID will produce Pending rather than
authorization. A missing owner, wrong UID, hash mismatch, identity mismatch, or unexpected namespace
will never be used to mint a token.

This chain means ordinary users who can write objects in the runtime namespace cannot select a
different target ServiceAccount. Users authorized to create arbitrary ManifestWork for the hosting
cluster are within the trusted hosting-cluster administration boundary: that permission can also
deploy a pod that mounts the klusterlet's external managed-cluster kubeconfig.

### 5. Target connection and TokenRequest

After authenticating the request, the controller will locate Hosted-mode Klusterlet objects whose
cluster name matches targetCluster. spec.clusterName is optional; when it is empty, the controller will
use the name the operator already reads from the agent's hub kubeconfig Secret. Exactly one match will
be required. No match will report Pending; multiple matches will report Failed rather than choosing one.

The controller will read the fixed external-managed-kubeconfig Secret from that Klusterlet's
runtime namespace. This broad credential will remain inside the klusterlet operator process and
will never be copied to the addon runtime namespace. A target-cluster client created from it will
use a bounded request timeout.

The controller will then:

1. Get the declared target ServiceAccount and record its UID.
2. Call the ServiceAccount token subresource with the declared expiration.
3. Build a kubeconfig containing the target server, CA and TLS connection settings, and tokenFile.
4. Create or update the stable credential Secret in the runtime namespace.
5. Schedule the next refresh from the actual expiration returned by the target API server.

The klusterlet operator already writes hosted agent kubeconfig Secrets this way, with
`helpers.SATokenGetter` and `helpers.SyncKubeConfigSecret` in ocm `pkg/operator/helpers`: the same data
keys, a tokenFile kubeconfig, and the same connection change checks. The credential controller will
extend that helper instead of adding a second one. Unlike the agent path, it will not use legacy
ServiceAccount token Secrets, will use the declared lifetime and framework refresh policy, and will
check Secret ownership before every update instead of applying by name.

The target ServiceAccount and its RBAC will remain addon-owned. TokenRequest will not add any
authorization beyond that ServiceAccount. The target API server may shorten the requested lifetime;
the controller will use the returned expiration and will reject a token that leaves insufficient
time for its refresh policy.

### 6. Credential Secret contract

The stable Secret will be type Opaque and will contain:

| Data key | Purpose |
|---|---|
| kubeconfig | Generated target-cluster kubeconfig using tokenFile. |
| token | Short-lived bearer token mounted at the path named by tokenFile. |
| expiration | Actual RFC3339 token expiration. |
| serviceaccount_uid | Recorded target ServiceAccount UID. |

The Secret will carry the credential marker, immutable request hash, and addon instance identity.
Its controller owner reference will point to the request ConfigMap. A valid existing Secret must
have that exact controller owner and request identity before it can be read, updated, invalidated,
or deleted.

The authenticated request already contains the ServiceAccount name, tokenFile path, and requested
lifetime, so the Secret will not copy them into separate data keys. Its request hash identifies that
declaration. Connection identity will be compared through the generated kubeconfig rather than
stored again as a separate hash.

The Secret will always be named `managed-cluster-kubeconfig`. The runtime namespace is already unique
per addon instance, so the name does not need to vary, and a fixed name lets the operator's RBAC list it.
If another object occupies the name, the controller will leave it unchanged and report Failed.

For a Hosted-mode instance with `ManagedClusterCredential` selected, the Helm factory will set
`.Values.managedKubeConfigSecret` and the Go template factory will set `.ManagedKubeConfigSecret` to
`managed-cluster-kubeconfig`. These will be authoritative built-in values in this mode, taking
precedence over user-supplied values. Runtime and pre-delete hook templates will use the factory
value instead of constructing a Secret name from the addon name. Both factories will retain the
existing `<addon-name>-managed-kubeconfig` default and user override behavior for instances without
the credential option and for Default-mode instances, even when their addon type selects the option.

The kubeconfig auth info will contain tokenFile and will not embed token. The addon will mount both
kubeconfig and token from the same Secret volume so the configured token path resolves to the
sibling token file. client-go will periodically read that file and prefer its latest successful
contents, allowing routine rotation without a pod restart.

### 7. Freshness, rotation, and credential replacement

A credential will be fresh only when all of these remain true:

- the actual expiration leaves enough time under the framework's refresh policy;
- the Secret has the expected controller owner and authenticated request hash;
- the ServiceAccount selected by that request still has the recorded UID; and
- the kubeconfig matches the expected tokenFile and target connection, including server, CA data,
  insecure TLS setting, TLS server name, proxy setting, and connection behavior.

The shared credential helper will choose a refresh time before the actual expiration returned by
the target API server, leaving time for propagation, token-file polling, and retry. This is an
internal framework policy, not another addon option or a fixed interval in this API contract.
Reconciliation will also run at a bounded identity-check interval to detect ServiceAccount
recreation without a hosting-cluster event. After a restart the controller will reevaluate the
current Secret and expiration; it will not depend on a saved timer or lifecycle phase.

The controller will verify ownership before every mutation. If a requested identity or connection
change is detected, or if the target ServiceAccount cannot be found, it will first set token to an
empty value and remove expiration from the owned Secret. Only then will it attempt to mint and
publish a replacement. If ownership cannot be proven, invalidation itself will fail and the
controller will not adopt the Secret.

The controller will report the credential as not ready until it has verified the target and
published a replacement.

Routine expiry refresh is different from an identity change. If the target identity and connection
remain unchanged and a refresh call temporarily fails, the still-valid old token may remain
available until its server-enforced expiration. The condition will report the reconciliation
failure and retries will continue.

A connection change will replace the generated kubeconfig as well as the token. Secret volume
projection will update the files, but an addon process that caches server or CA settings must watch
the kubeconfig and reload or restart itself. The restart-free guarantee in this proposal applies to
routine token rotation.

### 8. Status and discoverability

The controller will write three feedback annotations to the authenticated request ConfigMap:

~~~text
addon.open-cluster-management.io/managed-credential-status
addon.open-cluster-management.io/managed-credential-message
addon.open-cluster-management.io/managed-credential-observed-hash
~~~

Status will be one of Pending, Ready, or Failed. The observed hash will identify the authenticated
request for which that status was produced. ManifestWork feedback rules will return all three
values to the hub.

The addon framework will accept any of the three statuses only when:

- the ManifestWork is Applied for its current generation;
- the Work marker and immutable request hash match the declaration; and
- managed-credential-observed-hash exactly equals that request hash.

Otherwise the framework will keep the condition Pending. This prevents feedback from an older Work
generation or a mutable runtime object from being interpreted as current success.

ManagedClusterAddOn will receive:

| Controller state | Condition status | Reason |
|---|---|---|
| Pending | False | ManagedClusterCredentialPending |
| Ready | True | ManagedClusterCredentialReady |
| Failed | False | ManagedClusterCredentialFailed |

The condition type will be ManagedClusterCredentialReady. Its message will identify the pending
dependency, validation error, or ready Secret and expiration.

While the option is selected, status.relatedObjects will include the exact core Secret reference:

~~~yaml
- group: ""
  resource: secrets
  namespace: <runtime-namespace>
  name: managed-cluster-kubeconfig
~~~

The reference will be stable because the runtime namespace and Secret name are immutable. It lets
users discover the credential without reconstructing naming rules.

### 9. RBAC and trust boundary

The hosting-cluster klusterlet operator will need:

- get, list, watch, and patch on credential request ConfigMaps, which its ClusterRole already allows;
- create on Secrets, which is already allowed, and `managed-cluster-kubeconfig` added to its existing
  named-Secret rule, which grants get, list, watch, update, and delete. That rule matches the name in
  every namespace, which is one more reason the checks below are mandatory;
- get on AppliedManifestWork, added next to its existing list, update, and patch; and
- its existing access to Hosted Klusterlet objects and external-managed-kubeconfig.

The runtime namespaces are dynamic, so RBAC cannot limit these permissions to them. Authentication
and ownership checks in the controller are therefore mandatory, not defense in depth that can be
omitted.

Hosted workload RBAC must not grant create, update, patch, or delete on
managed-cluster-credential-request. Addon Roles should name the individual ConfigMaps they need to
read and should use Leases, not a generally writable ConfigMap rule, for leader election. The
workload may get and mount its credential Secret, but it does not need to update the Secret.

Cluster administrators and users allowed to create arbitrary ManifestWork targeting the hosting
cluster remain trusted. Ordinary namespace users and the hosted addon ServiceAccount are not
trusted to choose credential request fields or status.

### 10. Ownership and cleanup

The request ConfigMap will be owned by AppliedManifestWork. The credential Secret will be owned by
the request ConfigMap. Deleting the ManagedClusterAddOn will remove its ManifestWork, the work agent
will remove the request ConfigMap, and Kubernetes garbage collection will remove the Secret.
Deleting the isolated runtime namespace will provide a second lifecycle boundary.

A hosting pre-delete hook that mounts the credential Secret can still run during deletion, because
[191-hosted-addon-runtime-namespace](../191-hosted-addon-runtime-namespace/README.md) waits for the
hook's result before deleting the runtime Work that owns the request ConfigMap and, through it, the
Secret.

All explicit invalidation or cleanup will verify the expected controller owner, marker, immutable
hash, and current UID. Deletes will use a UID precondition so a newly created object at the same
name cannot be removed after a stale read.

The framework will remove the credential condition and Secret RelatedObject when the owning
ManagedClusterAddOn is deleted. There will be no cross-instance Secret cleanup, label-only sweep,
or name-pattern deletion.

### Architecture

~~~text
Hub cluster

  ManagedClusterAddOn <target>/<addon>
      |
      | AgentAddon creation-time opt-in
      v
  addon-framework
      |
      | ManifestWork:
      | - hosted workload
      | - managed-cluster-credential-request
      | - request marker and immutable hash
      v

Hosting cluster

  work-agent --------------------> AppliedManifestWork
      |                                  |
      | applies request ConfigMap        | owner UID and applied resource UID
      v                                  |
  isolated runtime namespace             |
      |                                  |
      +--> request ConfigMap <------------+
      |         |
      |         | authenticated declaration
      |         v
      |    klusterlet operator
      |         |
      |         | external-managed-kubeconfig stays here
      |         | ServiceAccount TokenRequest
      |         v
      |    target managed cluster
      |         |
      |         | scoped short-lived token
      |         v
      +--> credential Secret
                |
                | kubeconfig + tokenFile
                v
           hosted addon pod

  request status annotations
      |
      +--> ManifestWork feedback
              |
              v
         ManagedClusterAddOn
         condition + RelatedObject
~~~

### Test Plan

- Unit tests for deterministic request hashes and namespaces, including delimiter ambiguity,
  sanitization, truncation, long names, invalid characters, and stable output.
- Unit tests for option validation: feature absent, all required opt-ins present, target
  ServiceAccount defaults, required tokenFile, and minimum and maximum lifetime.
- Factory tests for Helm and Go template credential Secret values, including pre-delete hooks,
  proving the opted-in Hosted path uses the fixed name despite a user override, while Default mode
  and instances without the option retain their existing defaults and overrides.
- Controller tests for every request authentication failure: wrong fixed name, missing marker,
  missing or multiple AppliedManifestWork owners, owner UID mismatch, deleting owner, missing exact
  resource UID, addon label mismatch, namespace mismatch, missing data, and recomputed hash mismatch.
  Unknown keys must not influence credential decisions.
- Authentication tests proving that changing a Work name prefix does not change authorization when
  the verified ownership chain, request hash, and addon identity are unchanged.
- Controller tests for no matching Hosted Klusterlet, one match, multiple matches, missing or
  malformed external-managed-kubeconfig, target connection failures, missing ServiceAccount, and
  incomplete TokenRequest responses.
- Secret ownership tests proving that a colliding unowned Secret, wrong controller UID, wrong
  marker, wrong hash, and recreated object UID are never adopted, updated, invalidated, or deleted.
- Rotation tests for the helper's refresh scheduling using actual expiration, target lifetime
  shortening, transient refresh failure, insufficient renewal margin, and reevaluation after a
  controller restart without a saved timer.
- Credential replacement tests proving the published token is blanked and expiration removed before
  replacement when the ServiceAccount UID or connection identity changes, and when the target
  ServiceAccount disappears, with Ready withheld until a verified replacement has been published.
- An integration test that mounts the Secret, keeps the addon pod UID unchanged, rotates token,
  and proves subsequent client-go requests use the new tokenFile contents.
- Integration coverage for both factories rendering the workload's Secret volume from their
  credential Secret value and mounting the Secret created by the credential controller.
- A client-go consumption test showing that an empty token file retains the cached token, so Secret
  invalidation is not treated as client-side revocation.
- RBAC tests proving the hosted workload can read its mounted Secret but cannot create, update,
  patch, or delete the credential request ConfigMap.
- Status tests for Pending, Ready, Failed, message propagation, Applied observedGeneration, and
  mandatory observed-hash equality for every accepted status.
- Lifecycle tests proving ManagedClusterAddOn deletion removes the Work, request ConfigMap,
  credential Secret, condition, and RelatedObject without touching a colliding or unrelated Secret.
- Compatibility tests proving an absent option produces no request, no new condition, no new
  RelatedObject, and no Default-mode behavior change.

### Graduation Criteria

#### Alpha

- The creation-time Go option, authenticated request transport, hosting-cluster credential
  controller, tokenFile rotation, condition, and RelatedObject are available as an explicit
  opt-in.
- Unit and integration coverage includes the authentication, takeover, rotation, credential
  replacement, and cleanup cases above.
- Documentation explains target ServiceAccount RBAC, Secret mounting, tokenFile, and the immutable
  configuration contract.

#### Beta

- At least one hosted addon uses the option with a narrowly scoped target ServiceAccount.
- Rotation and ServiceAccount recreation have been exercised over at least one release.
- A security review has confirmed the ManifestWork trust boundary, hosted workload RBAC, Secret
  non-adoption rules, and klusterlet operator permissions.
- Operational feedback requires no change to the immutable option shape.

#### GA

- Multiple hosted addon types have used the opt-in across at least one release.
- Upgrade and downgrade guidance has been exercised without requiring migration of existing addon
  objects.
- No security incident has been attributed to request authentication, Secret ownership, or token
  rotation behavior.

### Upgrade / Downgrade Strategy

**Upgrade:** all new behavior will default to off. Upgrading api, sdk-go, addon-framework, or ocm
will not create a request, credential Secret, condition, or RelatedObject for any addon type that
does not select ManagedClusterCredential. Existing ManagedClusterAddOn and Default-mode objects
will require no data migration, annotation, namespace move, or manifest rewrite.

The klusterlet operator ClusterRole will include the credential-controller permissions on upgrade
because Kubernetes RBAC cannot condition rules on whether an addon has selected a Go option. With no
authenticated credential request, the controller will make no credential mutation and the new
permissions will remain unused.

An addon type that plans to use the feature will select all required options before creating the
addon instances that depend on it. The hosting-cluster klusterlet operator version that understands
the request must be available before those instances are created. Existing instances will not be
silently enrolled by an upgrade.

**Downgrade:** because existing defaults and stored objects are unchanged, addon types that never
selected the option can downgrade with no migration. For addon instances created with the option,
the selected configuration remains immutable and the supported lifecycle operation is deletion of
the owning ManagedClusterAddOn. Its normal Work and owner-reference cleanup will remove the
credential. Any token copied outside Kubernetes ownership will expire within its bounded lifetime.

The Go option introduces no new persisted CRD field. ManagedClusterCredentialReady uses the
existing conditions list, and the Secret uses the existing relatedObjects list. The separate
status.hostingNamespace field and runtime namespace behavior are owned by KEP 191.

### Version Skew

- A newer klusterlet operator with an older addon framework will see no request and take no action.
- A newer addon framework must not create an opted-in instance until the hosting cluster runs a
  klusterlet operator that supports the request. If it does, the request will remain Pending and no
  broad credential will be exposed.
- Older clients will ignore the additive condition reason and RelatedObject.
- Both served addon API versions will use the same condition, label, and annotation strings.
- No component may treat an unauthenticated request or feedback value as valid during version skew.

## Implementation History

- 2026-09-14: Initial proposal.

## Alternatives

**Give each hosted addon the external managed-cluster kubeconfig.** This is simple but gives every
workload the klusterlet operator's broader identity. A compromised addon could then act outside its
declared ServiceAccount permissions.

**Let every addon run its own credential provisioner.** This duplicates security-sensitive
TokenRequest, rotation, ownership, and connection handling. It also requires each provisioner to
receive a bootstrap credential and makes readiness and cleanup inconsistent across addons.

**Mint the token from the hub.** The hub-side addon framework is not guaranteed to have a network
path to the target managed cluster. The hosting-cluster klusterlet operator has both that path and
the established external connection.

**Use a long-lived ServiceAccount token Secret.** Legacy token Secrets are not an appropriate
rotation mechanism and can remain valid indefinitely. TokenRequest provides bounded lifetime and
server-enforced expiration.

**Embed the token directly in kubeconfig.** A running client usually parses an embedded token once.
tokenFile supports routine Secret rotation without forcing the addon pod to restart.

**Create a new hosting-cluster CRD for credential requests.** A typed API could provide validation,
but it would add installation and versioning requirements to the hosting cluster. ManifestWork,
AppliedManifestWork ownership, and a fixed ConfigMap are sufficient for the one immutable
declaration in the initial scope.

**Add AddOnTemplate support immediately.** That would require a persisted API and conversion
design before the Go use case can be evaluated. A later proposal can add a declarative surface
without changing this request and credential protocol.

**Use the managed-serviceaccount addon to distribute this credential.** That API addresses a
broader multi-cluster ServiceAccount lifecycle. This proposal only needs a credential tied to one
hosted addon instance and one ServiceAccount the addon already owns.

**Report status from the hosted workload.** The workload must not be trusted to assert that a
privileged credential was minted correctly. Feedback will originate from the klusterlet operator's
annotations on an authenticated request and will be accepted only with the observed hash.

## Infrastructure Needed

- No new repository or CI infrastructure.
- Changes will be coordinated across api, sdk-go, addon-framework, and ocm. The API constants and
  shared SDK contract will precede the framework request producer and klusterlet operator
  controller.

## References

- Issue: https://github.com/open-cluster-management-io/enhancements/issues/190
- [19-projected-serviceaccount-token](../19-projected-serviceaccount-token/README.md) - TokenRequest
  and rotation precedent
- [29-manifestwork-status-feedback](../29-manifestwork-status-feedback/README.md) - ManifestWork
  feedback
- [33-hosted-deploy-mode](../33-hosted-deploy-mode/README.md) - klusterlet Hosted mode
- [63-hosted-addon](../63-hosted-addon/README.md) - addon Hosted mode
- [188-hosted-addon-follow-klusterlet](../188-hosted-addon-follow-klusterlet/README.md) - hosting
  cluster discovery and validation
- [191-hosted-addon-runtime-namespace](../191-hosted-addon-runtime-namespace/README.md) - isolated
  per-instance runtime namespace
