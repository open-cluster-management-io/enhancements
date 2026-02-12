# ConfigurationPolicy ComplianceTypes for different merge behaviors

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

We propose to add new options for the `spec["object-templates][*].complianceType` field in
ConfigurationPolicies to allow the user to specify new merge strategies: `musthaveapply`, 
`musthavestrategic` and `musthavemerge`. These new options correspond to existing options in
`kubectl`, which users may be familiar with. Along with the existing `musthave`, `mustonlyhave`, and
`mustnothave` options, this should provide more flexibility and control to users, and be able to
provide a more predictable experience.

## Motivation

The existing `musthave` complianceType has a bespoke algorithm for merging content from the cluster
state with what is defined in the policy. One specific difference is when the policy includes a list
in the object template: `musthave` will check the existing list for any items specified in the
policy, and if any are missing, it will *add* them to the list and apply the change. Tools like
`kubectl apply` can do something similar on some objects, via a "strategic merge", but not on custom
resources.

The existing `mustonlyhave` complianceType is simpler to understand: any fields on the object not
defined in the policy will be removed from the object on the cluster (possibly just reverting them
to default values). This can be used to avoid other merge behaviors, but requires the policy author
to include more of the object in the policy, and does not allow the unset fields to be set by other
controllers. This complianceType has been known to cause issues by repeatedly updating objects on
the cluster, causing high API server load.

New complianceTypes would give more tools to policy authors to specify the exact behavior they want.

### Goals

1. Policy authors should be able to create a policy to *replace* a specific list in an object on the
   cluster with what is defined in the policy, *without* that policy reverting other customizations
   on the object.
2. Policy authors should be able to create a policy that has the exact same behavior as `kubectl`
   commands like `apply`, `patch --type=strategic` and `patch --type=merge`.
3. Clarify in documentation the exact behaviors of the complianceType options.

### Non-Goals

1. Support strategic merge on custom resources (`kubectl` does not support this).

## Proposal

Add a `musthavemerge` complianceType, corresponding to the `kubectl` command `patch --type=merge`.
In particular, this allows for replacement of specific fields / objects / lists in an object without
reverting other "unrelated" changes in the object. This merge strategy works on all resource types,
including custom ones.

Add a `musthavestrategic` complianceType, corresponding to `kubectl patch --type=strategic`. This
strategy works only on "built-in" types on the cluster, but allows for more intelligent merges using
"patchStrategy" information available in those type definitions. For example, it allows for adding
an additional environment variable to a Deployment, as long as it has a new name.

Add a `musthaveapply` complianceType, corresponding to `kubectl apply`. The patch type is chosen
based on the involved resource's kind: custom resources use `--type=merge`, and built-in types use
`--type=strategic`.

These new types will only change the behavior when *updating* a resource on the cluster.

### User Stories

#### Story 1

A policy author can define a ConfigurationPolicy to *replace* the `env` list in a container in a
Deployment, without specifying *all* of the other non-default fields of the Deployment in the
ConfigurationPolicy.

Say this is the existing Deployment:
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: example
  namespace: default
spec:
  replicas: 0
  selector:
    matchLabels:
      app: example
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: example
    spec:
      containers:
        - name: container
          image: 'image-registry.openshift-image-registry.svc:5000/openshift/httpd:latest'
          ports:
            - containerPort: 8080
              protocol: TCP
          env:
            - name: hello
              value: world
          resources: {}
          terminationMessagePath: /dev/my-termination-log # (customized)
          terminationMessagePolicy: File
          imagePullPolicy: Always
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirstWithHostNet # (customized)
      securityContext: {}
      schedulerName: default-scheduler
  strategy:
    type: Recreate
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
```

This ConfigurationPolicy should change *only* the environment variable section:
```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: ConfigurationPolicy
metadata:
  name: example
spec:
  namespaceSelector:
    exclude: []
    include:
    - default
  object-templates:
  - complianceType: musthavemerge
    objectDefinition:
      kind: Deployment
      apiVersion: apps/v1
      metadata:
        name: example
        namespace: default
      spec:
        template:
          spec:
            containers:
            - name: container
              image: 'image-registry.openshift-image-registry.svc:5000/openshift/httpd:latest'
              ports:
              - containerPort: 8080
                protocol: TCP
              env:
              - name: foo
                value: bar
              terminationMessagePath: /dev/my-termination-log
    recreateOption: None
  pruneObjectBehavior: None
  remediationAction: enforce
  severity: medium
```

Note: effectively, the merge is replacing the entire `containers` list, so the `image`, `ports`, and
`terminationMessagePath` fields must be set to preserve them. However, this is far fewer fields than
would be required to accomplish this via a `mustonlyhave` policy, and allows for other processes to
customize things like the number of replicas on the Deployment.

For comparison, a `mustonlyhave` policy to accomplish this would look like:
```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: ConfigurationPolicy
metadata:
  name: example
spec:
  namespaceSelector:
    exclude: []
    include:
    - default
  object-templates:
  - complianceType: mustonlyhave
    objectDefinition:
      kind: Deployment
      apiVersion: apps/v1
      metadata:
        name: example
        namespace: default
      spec:
        replicas: 0
        selector:
          matchLabels:
            app: example
        template:
          metadata:
            labels:
              app: example
          spec:
            containers:
            - name: container
              image: 'image-registry.openshift-image-registry.svc:5000/openshift/httpd:latest'
              ports:
              - containerPort: 8080
                protocol: TCP
              env:
              - name: hello
                value: world
              terminationMessagePath: /dev/my-termination-log
            dnsPolicy: ClusterFirstWithHostNet
        strategy:
          type: Recreate
    recreateOption: None
  pruneObjectBehavior: None
  remediationAction: enforce
  severity: medium
```

#### Story 2

A policy author can define a ConfigurationPolicy to update an item in the `ports` list of a
Deployment, without specifying other (non-identifying) fields in the policy. More generally, users
should be able to benefit from arbitrary `patchMergeKey` settings on built-in types.

With the same initial cluster state as Story 1, this ConfigurationPolicy should make the correct
update:
```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: ConfigurationPolicy
metadata:
  name: example
spec:
  namespaceSelector:
    exclude: []
    include:
    - default
  object-templates:
  - complianceType: musthavestrategic
    objectDefinition:
      kind: Deployment
      apiVersion: apps/v1
      metadata:
        name: example
        namespace: default
      spec:
        template:
          spec:
            containers:
            - name: container
              ports:
              - containerPort: 8080
                protocol: UDP
    recreateOption: None
  pruneObjectBehavior: None
  remediationAction: enforce
  severity: medium
```

If the complianceType was `musthave`, the resulting deployment would specify two ports: one UDP
and one TCP. The bespoke implementation for `musthave` does not recognize `containerPort` as the
`patchMergeKey` for that list - it only uses `name` for that purpose.

A `mustonlyhave` or `musthavemerge` policy would require many more fields to be set in the policy.

### Implementation Details/Notes/Constraints [optional]

The existing `musthave` complianceType is likely the best general choice, working on both built-in
types and custom resources the same way, and allowing the policy author to be sparse on the fields
they define in the policy. The new types would allow policy authors to have *some* more control over
the behavior, without requiring the full object definition like in `mustonlyhave`.

The `kubectl patch` command has three types of patches that can be specified. The names can be
confusing:
- `--type=json` [JSON Patch, RFC 6902](https://datatracker.ietf.org/doc/html/rfc6902) is not 
  covered by this proposal. Patches look like `[{"op": "replace", "path": "/a/b/c", "value": 42}]`.
- `--type=merge` [JSON Merge Patch, RFC 7386](https://datatracker.ietf.org/doc/html/rfc7386).
  Patches look like a snippet of the resource in question, with omitted fields (generally) being
  merged with the state of the object on the cluster.
- `--type=strategic` is kubernetes-specific. It can use additional API information on built-in types
  in order to make more intelligent merge decisions. Patches look like a snippet of the resource.

#### Server-side apply

Kubernetes also supports [server-side apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/).
This makes heavy use of the `metadata.managedFields` field in the object being managed, and utilizes
a [structured merge](https://github.com/kubernetes-sigs/structured-merge-diff), which is closest to
the strategic merge above. One major advantage is that this technique can use the OpenAPI info from
custom resources for more intelligent merges. A disadvantage is that the `managedFields` section
must be carefully maintained.

Some of the server-side apply functionality might not be of benefit to ConfigurationPolicy: after
applying a partial object, the API server configures the `managedFields` to reflect that the fields
in that request are now owned by the specific field-manager of that request. Applying another
partial object with fewer fields later (for example if the policy changes) will result in the
missing fields being **removed** from the object on the cluster. In other words, server-side apply
is not always idempotent. In order to ensure idempotency and proper management of fields, the
controller would need to first patch the `managedFields`, relinquishing control of the previous
fields, and then submit the server-side apply request.

Another function enabled by server-side apply which ConfigurationPolicy would need to consider is
conflict detection and resolution. [The reference doc](https://kubernetes.io/docs/reference/using-api/server-side-apply/#using-server-side-apply-in-a-controller)
specifically recommends that controllers always "force" conflicts, always taking control of the
fields that they specify and overriding any other existing configuration. That is most similar to
what ConfigurationPolicy currently does when it is enforcing. It might be interesting to allow
policy authors to stop enforcement if a conflict is detected, and just have the policy become
NonCompliant, but that might deserve an entirely separate discussion.

One more potential advantage with server-side apply is that the requests trigger validating and 
mutating webhooks, even if there aren't any actual "changes" that a client-side merge would notice.
In order for ConfigurationPolicy to utilize this, it might need to always do a dry-run request to
check for changes in this mode (assuming a server-side apply dry-run triggers the webhooks). In
contrast, the current implementation only runs dry-run updates if it already believes a change is
necessary, the dry-run only preventing marking the policy as non-compliant when the update would
have no actual effect. 

### Risks and Mitigation

Adding new options makes the API more complex. Documentation with examples will help clarify what
each option does, including making the existing `musthave` behavior more clear.

As mentioned in User Story 1, the proposed `musthavemerge` will still require some additional fields
to be specified in the policy to prevent them from reverting to defaults. However, since this is the
same behavior as a `kubectl patch` command, it may be more familiar to some users. It also requires
fewer additional fields in general than `mustonlyhave`.

We do not forsee any increased use of resources (cpu, memory, API calls) by using these types. In
fact, using `musthavemerge` instead of `mustonlyhave` may reduce the total number of API calls by
preventing some cases of "thrashing". (Server-side apply could require additional API calls)

## Design Details

### Open Questions [optional]

1. Is `musthaveapply` particularly useful? It may have to guess whether a strategic merge is
   possible, for example on types in the OpenShift API which might support it. It may be more clear
   to simply require the policy author to choose. There might not be a case where the strategy
   should dynamically be chosen per-cluster, where this option would be helpful.
2. Is server-side apply another possibly useful option?

### Test Plan

**Note:** *Section not required until targeted at a release.*

<!--

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations). 

All code is expected to have sufficient e2e tests.

-->

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

<!--

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:

- [Maturity levels][maturity-levels]
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),
or by redefining what graduation means.

In general, we try to use the same stages (alpha, beta, stable), regardless how the functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

-->

### Upgrade / Downgrade Strategy

As new options, there are no concerns with upgrades. After upgrading to a version that supports
these options, they should just work. Downgrading is only a concern if a policy has been created
that uses a new option. In that case, it may be required to delete the policy, or change the field
to a supported value.

### Version Skew Strategy

Through CRD validation, the policy framework will not be able to apply a ConfigurationPolicy using
one of the new `complianceType` options on a cluster that does not support it yet. The framework
will remove the existing ConfigurationPolicy (if it exists) and emit a template error.

## Implementation History

<!--
Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.
-->

## Drawbacks

See "Risks and Mitigation"

## Alternatives

Using `object-templates-raw` allows a policy author to effectively "roll their own" merge strategy,
but the syntax could be very complex, and it would likely be error-prone.

## Infrastructure Needed [optional]

None.
