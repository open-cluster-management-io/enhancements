# Core Policy Module

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

A common repository / module will contain common API validation, structs, and
methods that multiple policy controllers can use. This will simplify creating
new controllers, adding features that all the controllers can benefit from, and
maintaining the controllers.

## Motivation

Current policy controllers rely on detailed knowledge from the policy framework 
implementation, that is not clear to new developers. Most notably, the format
for events to be picked up by the policy framework is not well documented. Other
details will be more consistent if maintained in a single place.

### Goals

1. Common API validation: fields like `.spec.severity` which use enums will have
consistent validation through kubebuilder when custom policy types embed them.
1. Required API features: fields like `.spec.remediationAction` are required on
all policy types so they can be passed down from the parent policy. Putting
these in a struct that policy types embed will ensure they are implemented.
1. Common methods: behavior like selecting namespaces and emitting compliance
events for the policy framework will have a common implementation. This will
reduce bugs from inconsistency, and allow policy controllers to focus on their
specific behavior, not the details of interacting with the framework.
1. Automated tests to ensure the features/methods listed above work properly
when used by custom controllers.
1. Custom Controller tutorial: we will create and maintain documentation showing
how to create a new custom policy controller, and use the main features of this
module.

### Non-Goals

1. Custom Controller Template: we will not maintain a template/sdk like the 
operator-sdk for new policy controllers.

## Proposal

Create a new repository in open-cluster-mangement-io to provide implementations
of common APIs and functions for controllers to work with the policy framework,
and describe the behavior of the policy framework, similar to some of the docs
in https://github.com/stolostron/governance-policy-framework.

### User Stories

#### Story 1

As a developer of a policy controller, I can ensure consistent behavior with
other policies by incorporating the common API types. For example, if the
behavior of wildcards in the common `spec.namespaceSelector` field changes, I
should not have to adjust my controller's code to be consistent, other than
updating the version of the common library.

#### Story 2

As a creator of a new policy controller, I can focus on my controller's 
specialized detection of violations, and rely on common implementations of
functions in order to interact with the policy framework. For example, the
reporting of compliance through events should not require a new implementation.

#### Story 3

As a developer of multiple policy controllers, I can fix a bug or add a feature
to common policy behavior in one place, and not duplicate it in each controller.

### Implementation Details/Notes/Constraints 

Although kubebuilder will automatically pull API validation from embedded
structs (like when types embed `metav1.TypeMeta`), it will not automatically
pull RBAC permissions on methods into the project. So RBAC requirements on
common functions (eg `get namespaces` for selecting namespaces) will need to
be well-documented.

This module can include features that aren't strictly required or currently used
by the policy framework. For example, storing additional structured information
about k8s objects that the policy looks at in a `status.relatedObjects` field.
Experimental features can be added, but should be marked as such by the
versioning of the repository.

### Risks and Mitigation

(unknown)

## Design Details

Here is a part of the common struct and validation from a
[prototype](https://github.com/JustinKuli/policycore-test/blob/main/api/v1/policycore_types.go)
of this proposal:

```golang
type PolicyCoreSpec struct {
	// Severity defines how serious the situation is when the policy is not
	// compliant. The severity should not change the behavior of the policy, but
	// may be read and used by other tools. Accepted values include: low,
	// medium, high, and critical.
	Severity Severity `json:"severity,omitempty"`

	// RemediationAction indicates what the policy controller should do when the
	// policy is not compliant. Note that policy controllers are not required to
	// automatically remediate a policy, even when set to "enforce". Accepted 
	// values include inform, and enforce.
	RemediationAction RemediationAction `json:"remediationAction,omitempty"`

	// NamepaceSelector indicates which namespaces on the cluster this policy
	// should apply to, when the policy applies to namespaced objects.
	NamespaceSelector NamespaceSelector `json:"namespaceSelector,omitempty"`
}

//+kubebuilder:validation:Enum=Low;low;Medium;medium;High;high;Critical;critical
type Severity string

//+kubebuilder:validation:Enum=Inform;inform;Enforce;enforce
type RemediationAction string

type NamespaceSelector struct {
	// Include is a list of namespaces the policy should apply to. UNIX style
	// wildcards will be expanded, for example "kube-*" will include both
	// "kube-system" and "kube-public".
	Include []string `json:"include,omitempty"`

	// Exclude is a list of namespaces the policy should _not_ apply to. UNIX
	// style wildcards will be expanded, for example "*openshift*" will exclude
	// any namespace that includes with "openshift" in its name.
	Exclude []string `json:"exclude,omitempty"`
}
```

With this struct, a new policy controller could implement their type like this,
in order to get the same validation automatically:

```golang
import policycore "github.com/open-cluster-management-io/policycore/api/v1" // example import - this repo does not exist

type MyNewPolicySpec struct {
    policycore.PolicyCoreSpec `json:",inline"`

    Foo string `json:"foo"`
}
```

Common behavior could be implemented in the common library through methods on 
these new types, for example, utilizing the `spec.namespaceSelector` field could
be a method with this signature and doc:

```golang
// GetNamespaces fetches all namespaces in the cluster and returns a list of the
// namespaces that match the NamespaceSelector. The client.Reader needs access
// for viewing namespaces, like the access given by this kubebuilder tag:
// `//+kubebuilder:rbac:groups=core,resources=namespaces,verbs=get;list;watch`
func (sel NamespaceSelector) GetNamespaces(ctx context.Context, r client.Reader) ([]string, error) {
    // ... implementation ...
}
```

At least one additional function would cover creating the compliance event that
the status-sync component watches for.

### Open Questions

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

New features and adjustments to existing features should be made in a backwards
compatible way. In particular, this means that new items in common structs
to be embedded by policy types can not be required.

Since the intent is to have other policy controllers import this for common
functions, those functions should maintain their behavior, except for bugfixes.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

Each piece of the policy framework could expose types/methods for behavior that
that piece requires. For example, the status-sync could expose a function for
creating the events that it is watching for, and the template-sync could expose
a type with the common fields it requires like `remediationAction`. In this
case, the policy controllers would need to import each piece of the framework,
and keep each updated to match each other if changes were made. There would
also not be a clear place for non-required, but commonly used fields like the
`spec.namespaceSelector`, and their behavior.

## Infrastructure Needed

A new repository, name TBD. Maybe `policy-framework`?
