# Namespace `LabelSelector` for `ConfigurationPolicies`

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
      [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

Implement `MatchLabels` and `MatchExpressions` (collectively known as `LabelSelector`) for
`ConfigurationPolicies` in order to further filter the list of namespaces to which an object should
be applied.

## Motivation

Currently the `ConfigurationPolicy` has a `spec.namespaceSelector` that includes `include` and
`exclude` arrays of strings that optionally accept a leading and/or trailing wildcard to filter by
the name of the namespace. In order to more closely align with Kubernetes use cases, the proposal is
to implement a `LabelSelector` along with these arrays in order to filter namespaces by their label.
The `LabelSelector` struct includes `MatchLabels` and `MatchExpressions`. In addition to aligning
with Kubernetes practices, this will cut down on processing the namespace list for clusters with a
large number of namespaces since a shorter list would be returned from the Kubernetes API.

Futhermore, an additional proposal is to use the `filepath` Go library rather than a custom wildcard
implementation for the `include` and `exclude` lists. This will bring maintainability and additional
power to the namespace filtering. This will not be a breaking change since the current behavior is
to only support leading and trailing wildcards, and a pattern like `foo*bar` is treated as a literal
string, which is not a valid namespace name.

### Goals

1. Filter the namespace list by `LabelSelector` for the `NamespaceSelector`
2. Allow `LabelSelector` and `include`/`exclude` arrays to operate together when both are provided
   or separately when provided individually
3. Replace custom wildcard code with `filepath` (See
   [filepath](https://pkg.go.dev/path/filepath#Match) for reference)

## Proposal

Implement `MatchLabels` and `MatchExpressions` alongside the `include` and `exclude` arrays:

```yaml
namespaceSelector:
  include: []
  exclude: []
  matchLabels: {}
  matchExpressions: []
```

These can easily be used to construct a `LabelSelector` for the Kubernetes API to consume.

The `MatchLabels` and `MatchExpressions` are AND-ed by default, so the implementation will carry
this forward to find the intersection of the `LabelSelector` and, if provided, the `include` and
`exclude` arrays. Either the `include` and `exclude` arrays or the `LabelSelector` can be provided.
If the `LabelSelector` is not provided, all namespaces are retrieved and filtered with the `include`
and `exclude` arrays. If only a `LabelSelector` is provided and not `include`/`exclude`, it will be
as if `include` were set to `["*"]` and the results will only be filtered by the `LabelSelector`.
(See the [LabelSelector](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#LabelSelector)
struct for reference.)

### User Stories

Suppose clusters have namespaces named `test1` through `test5` labeled with a `name` key and value
matching its name and an `all: namespaces` label.

#### Story 1

As an administrator of `ConfigurationPolicies`, I can choose namespaces in which to deploy defined
objects by the namespace label.

```yaml
namespaceSelector:
  matchLabels:
    name: test2
  matchExpressions:
    key: name
    operator: In
    values: ["test1", "test2"]
```

**Result**: `test2`

**Logic Flow**: `MatchExpressions` will return namespaces `test1` and `test2` according to their
`name` labels while `MatchLabels` will return `test2` according to its label. As defined by
`LabelSelector`, these results are AND-ed and namespace `test2` would be returned from the API.

#### Story 2

As an administrator of `ConfigurationPolicies`, I can use `include`/`exclude` filepath patterns to
filter out by namespace name, and without needing to provide `MatchLabels`/`MatchExpressions`.

```yaml
namespaceSelector:
  include: ["test[3-5]"]
  exclude: ["test4"]
```

**Result**: `test3`, `test5`

**Logic Flow**: With no `LabelSelector` defined, all namespace are returned from the API. `include`
would filter the namespaces to `test3`, `test4`, and `test5` according to the provided `filepath`
character range pattern. `test4` is excluded by the literal string provided in `exclude`, leaving
`test3` and `test5` returned.

#### Story 3

As an administrator of `ConfigurationPolicies`, I can choose namespaces in which to deploy defined
objects by the namespace label and then filter out by name using filepath patterns.

```yaml
namespaceSelector:
  exclude: ["test[2-3]"]
  matchLabels:
    all: namespaces
```

**Result**: `test1`, `test4`, `test5`

**Logic Flow**: `MatchLabels` will return all five namespaces from the API since all have the
`all: namespaces` label. `test2` and `test3` are excluded by the filepath character range pattern
provided in `exclude`, leaving `test1`, `test4`, and `test5` returned.

#### Story 4

As an administrator of `ConfigurationPolicies`, I can override the `NamespaceSelector` by providing
`metadata.namespace` in the object itself. (This should match current behavior--putting here for
clarity.)

```yaml
namespaceSelector:
  include: ["test?"]
  matchExpressions:
    key: all
    operator: Exists
object-templates:
  - objectDefinition:
    metadata:
      ...
      namespace: this-other-namespace
```

**Result**:

- Namespaces from `namespaceSelector`: `test1`, `test2`, `test3`, `test4`, `test5`
- Object defined to check namespace: `this-other-namespace`

**Logic Flow**: `MatchExpressions` will return all five namespaces from the API since all have the
label key `all` defined. `include` will also not filter any namespace out because all namespaces
match the `test?` single-character wildcard. However, the object has a namespace defined, so the
controller will use the provided namespace, disregarding the `namespaceSelector`.

### Implementation Details/Notes/Constraints

Current behaviors of `NamespaceSelector` won't be disrupted. When namespaces are fetched, if a
`LabelSelector` is provided, it will be added to the API call to return matching namespaces.
Filtering by `include`/`exclude` will take place against the fetched namespaces as before. If
`include`/`exclude` are not provided but `MatchLabels`/`MatchExpressions` are provided, it will
behave as if `include: ["*"]` were provided and will return all of the namespaces fetched by the
`LabelSelector`. The `NamespaceSelector` can be overridden by providing `metadata.namespace` in the
object.

The struct would be defined as:

```golang
type NamespaceSelector struct {
	Include          []NonEmptyString                   `json:"include,omitempty"`
	Exclude          []NonEmptyString                   `json:"exclude,omitempty"`
	MatchLabels      *map[string]string                 `json:"matchLabels,omitempty"`
	MatchExpressions *[]metav1.LabelSelectorRequirement `json:"matchExpressions,omitempty"`
}
```

There is no change to the `include`/`exclude` definitions, but by defining
`MatchLabels`/`MatchExpressions` as pointers, we are able to tell whether they have been defined in
the object and can handle `nil` vs an empty type such as `map[string]string{}` (which would be
indistinguishable if it were defined without the pointer). This allows us to handle
`include`/`exclude` and `LabelSelector` independently since we're able to tell when they've been
defined by the user and can skip over the `LabelSelector` when it is not defined and fetch all
namespace for `include`/`exclude` or use the `LabelSelector` as it has been defined.

A `LabelSelector` is constructed from `MatchLabels` and `MatchExpressions` as:

```golang
metav1.LabelSelector{
  MatchLabels:      matchLabels,
  MatchExpressions: matchExpressions,
}
```

This is then converted to a string by `metav1.LabelSelectorAsSelector()` in order to pass to
`metav1.ListOptions{}`.

**References**:

- [`metav1.LabelSelector`](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#LabelSelector)
- [`metav1.LabelSelectorAsSelector`](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#LabelSelectorAsSelector)
- [`metav1.ListOptions`](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#ListOptions)

### Risks and Mitigation

(unknown)

### Open Questions

### Test Plan

**Note:** _Section not required until targeted at a release._

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

**Note:** _Section not required until targeted at a release._

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
