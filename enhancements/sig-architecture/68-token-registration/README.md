## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

OCM's use of a CSR based mechanism for registering spoke clusters with the hub cluster is incompatible with Kubernetes environments that cannot issue client auth certificates such as Amazon Elastic Kubernetes Service (EKS).
This enhancement provides a secondary Service Account Token based registration mechanism that is universally supported, and can be used as an alternative CSR in such environments. 

## Motivation

For cluster administrators, it is often preferable to leverage their cloud provider’s managed Kubernetes service (e.g. AKS, EKS, IKS, GKE etc) rather than self-managing the Kubernetes control plane and worker nodes - reducing cluster management overhead and complexity.
However, in some of these environments (for example EKS), the use of CSRs for client authentication is not permissible.

Adopting OCM should not require its users to change how they deploy and manage their hub Kubernetes cluster.

As such, OCM needs to support an alternative registration mechanism to CSRs that is expected to work in all environments, providing maximum compatibility with managed Kubernetes services or restricted/locked down environments. 

### Goals

- The OCM hub cluster can run on EKS, and in other environments where CSR registration cannot be used.
- `clusteradm` tooling provides options to select token based registration when joining a worker to the hub.
- Suitable warnings/error messages/conditions are added to OCM such that it is easily identifiable when CSRs cannot be used.

### Non-Goals

- Support for leveraging native cloud specific IAM providers or external credential providers such as [Vault](https://www.vaultproject.io/) in the registration process (which will be raised as a separate enhancement)

## Proposal

### User Stories

#### Story 1

#### Story 2

### Implementation Details/Notes/Constraints [optional]

### Risks and Mitigation

TODO service account token concerns/mitigations

## Design Details

### Open Questions [optional]

This is where to call out areas of the design that require closure before deciding
to implement the design.  For instance,
> 1. This requires exposing previously private resources which contain sensitive
     information.  Can we do this?

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

All code is expected to have sufficient e2e tests.

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

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

### Upgrade / Downgrade Strategy

If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to make use of the enhancement?

Upgrade expectations:
- Each component should remain available for user requests and
  workloads during upgrades. Any exception to this should be
  identified and discussed here.
- Micro version upgrades - users should be able to skip forward versions within a
  minor release stream without being required to pass through intermediate
  versions - i.e. `x.y.N->x.y.N+2` should work without requiring `x.y.N->x.y.N+1`
  as an intermediate step.
- Minor version upgrades - you only need to support `x.N->x.N+1` upgrade
  steps. So, for example, it is acceptable to require a user running 0.1.0 to
  upgrade to 0.3.0 with a `0.1.0->0.2.0` step followed by a `0.2.0->0.3.0` step.
- While an upgrade is in progress, new component versions should
  continue to operate correctly in concert with older component
  versions (aka "version skew"). For example, if an agent operator on a managed cluster
  is rolling out a new agent, the old and new agent in different managed clusters
  must continue to work correctly with old or new version of the hub while hub and agents
  remain in this partially upgraded state for some time.

Downgrade expectations:
- If an `N->N+1` upgrade fails mid-way through, or if the `N+1` agent is
  misbehaving, it should be possible for the user to rollback to `N`. It is
  acceptable to require some documented manual steps in order to fully restore
  the downgraded agent to its previous state. Examples of acceptable steps
  include:
    - Deleting any resources added by the new version of the agent operator in the
      managed cluster. The agent operator does not currently delete resources that
      no longer exist in the target version.

### Version Skew Strategy

How will the component handle version skew with other components?
What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- During an upgrade, we will always have skew among components, how will this impact your work?
- Does this enhancement involve coordinating behavior in the hub and
  in the klusterlet? How does an n-2 klusterlet without this feature available behave
  when this feature is used?
- Will any other components on the managed cluster change? For example, changes to infrastructure
  configuration of the managed cluster may be needed before the updating klusterlet or other
  agent components.

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used to
highlight and record other possible approaches to delivering the value proposed
by an enhancement.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.