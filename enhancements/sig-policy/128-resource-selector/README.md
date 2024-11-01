# Resource selector for `ConfigurationPolicy`

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
      [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

Add a resource selector to pair with the existing `namespaceSelector` in order to allow a
`ConfigurationPolicy` to filter and select objects when an `objectDefinition` doesn't have a name
provided. The resource selector would also enable the enforcement of `objectDefinitions` without a
name.

## Motivation

The `ConfigurationPolicy` currently has a `namespaceSelector` to configure the controller to filter
and range over namespaces relevant to a particular namespaced `objectDefinition`. Furthermore,
currently an `objectDefinition` without a name is feasible but is not enforceable. A
`resourceSelector` would both pair well with the `namespaceSelector` for further object filtering as
well as enable an enforceable unnamed `objectDefinition` by providing the controller with a list of
names to handle.

Adding a `resourceSelector` would lower the bar to entry for `ConfigurationPolicy`, adding an
additional configuration available to users without requiring knowledge of Go templating as is
required with the alternative, `object-templates-raw`.

### Goals

- Add a `resourceSelector` to `ConfigurationPolicy` to allow the controller to handle objects
  without requiring `object-templates-raw`
- Consolidate compliance messages to list out objects per namespace (one downside to the alternative
  `object-templates-raw` is there's a compliance message for each object)
- Make unnamed `objectDefinition`s enforceable

### Non-Goals

- Establishing complex logic in the `resourceSelector` that could not be handled readily by the
  Kubernetes API and, by extension, the `kubernetes-dependency-watches` library.

## Proposal

### User Stories

#### Story 1

As a policy user, I want to be able to create namespaces based on the `namespaceSelector`.

#### Story 2

As a policy user, I want to be able to select objects by label and have the controller iterate over
them, both to `inform` and `enforce`.

### Implementation Details/Notes/Constraints

Since the `resourceSelector` refers to a specific object, the `resourceSelector` would be embedded
in each element of the `object-templates` array. The `resourceSelector` would ONLY be composed of a
`LabelSelector` since that can be handled by the Kubernetes API. The `namespaceSelector` has
`include` and `exclude` filters for filtering by name, but such a feature requires that it run in
its own `Namespace` reconciler, which would be unwieldy with a `resourceSelector` where the `Kind`
is arbitrary.

In a YAML manifest, this new field would appear as follows:

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: ConfigurationPolicy
metadata:
  name: resource-selector-policy
spec:
  remediationAction: inform
  namespaceSelector:        # <---- Existing namespaceSelector
    exclude: []             #
    include: []             #
    matchLabels: {}         #
    matchExpressions: []    #
  object-templates:
    - complianceType: musthave
      resourceSelector:         # <---- New resourceSelector
        matchLabels: {}         #
        matchExpressions: []    #
      objectDefinition:
        apiVersion: v1
        kind: Pod
        metadata:
          labels:
            selected: true
    - ...
```

The `config-policy-controller` would list the provided `Kind` across all namespaces using the
provided `resourceSelector` and iterate over each matching object in each namespace to determine
compliance. Namespaces would be determined from either the namespace defined in the
`objectDefinition` or the `namespaceSelector` results.

The label selectors in `resourceSelector` will be a pointer to determine whether the user has
provided it to distinguish between fetching all objects vs fetching no objects.

### Risks and Mitigation

- Currently `mustnothave` doesn't consider fields when a name is provided, but _does_ consider
  fields when a name is not provided.
  - Caution should be taken (and tests written) to refactor `mustnothave` to consider fields as part
    of the `resourceSelector`.
- The implementation could increase load on the Kubernetes API.
  - Depending on how it's implemented, I wouldn't think the load would be any greater than if
    `object-templates-raw` were in play. Ideally the initial `List` request would populate the cache
    and any subsequent `Get` (if any) would fetch from the cache.
- Adding a `resourceSelector` could enable enforcement where enforcement wasn't previously
  available, which could have unintended consequences for users.
  - One possibility here would be to not enable enforcement, but I think there would be more demand
    for enforcement than not. And a change would need to be made to the policy in order to enable
    it, which hopefully users would test beforehand.

### Open Questions

1. Should `remediationAction: enforce` be allowed?
   - Some users may rely on the current behavior that an unnamed `objectDefinition` can only be
     `inform` and may cause unexpected results when a `resourceSelector` is newly applied to a
     policy they didn't realize was enforced.
2. Should the `resourceSelector` be able to determine the namespace if the namespace is not
   provided?
   - Currently if the namespace is not provided in the `objectDefinition` and the
     `namespaceSelector` returns no results, no objects are returned. I think this behavior should
     be preserved since it would follow the behaviors of tools like `kubectl`, but I could be
     convinced otherwise.
3. Should the `resourceSelector` have include/exclude name filtering?
   - This question was raised previously in standup.

### Test Plan

All code will have tests meeting coverage targets. All code will have e2e tests.

### Graduation Criteria

--

## Implementation History

--

## Alternatives

Users can accomplish this currently using `objects-templates-raw`.
