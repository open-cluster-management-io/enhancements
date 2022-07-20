# Selective Policy Enforcement

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

A Selective Policy Enforcement feature aims to provide users
additional levels of control over the way in which Configuration
Policy is enforced across a large set (fleet) of
ManagedClusters. Configuration Policy allows a user to declare the
desired state of configuration and bind that policy to a set of
ManagedClusters. This policy provides the user with both visibility
into the state of compliance of their clusters as well as a mechanism
to drive clusters into compliance. When the remediation action is set
to "enforce" the configuration will be applied to all bound clusters
immediately.

In some use cases the immediate remediation behavior of an "enforce"
policy across the fleet of clusters may be unacceptable. For example,
if the set of clusters are collectively meeting a Service Level
Agreement (SLA) uptime spec, a single change in a Configuration Policy
may create an unacceptable service downtime as all of the clusters are
simultaneously updated. In another example a set of clusters may
provide overlapping service coverage and changes to configuration
needs to be done in progressive waves to ensure continuous coverage. A
third example is when the cluster operator wants to soak a change on a
handful of clusters prior to rolling the change out to the entire
fleet.

With Selective Policy Enforcement a feature is introduced which allows
the user control over the timing of the "enforce" remediationAction
taking effect on a selected subset of the bound clusters.

## Motivation

Selective Policy Enforcement allows fine grained control over the
timing and application of a policy to clusters. It is intended to
support this level of control natively within the policy framework. By
supporting this as a feature users, or higher level
controllers/orchestrators, do not have to implement a procedural
pattern of copying policies and manipulating bindings/labels to
progressivly enforce the policy on clusters.

*Progressive Policy Rollout*

One of the key results of this feature is allowing the impact of a
Configuration Policy change to be "rolled out" to a fleet of clusters
in a progressive way rather than all at once. The timing and choice of
clusters to which the change is applied are typically use case
dependent but this feature allows scenarios such as
+ Application to a small set of "soak" clusters before rolling the
  change out to the entire fleet in progressively larger groups
+ Staggered updates to overlapping (geographical or logical) clusters
  to ensure continuous service coverage
+ Meeting Service Level Agreements for availability by limiting
  concurrent updates
+ Applying changes to groups of clusters with varying "maintenance
  windows" when SLAs allow changes to be made
+ Applying changes to one region while deferring other regions

### Goals

The goals of this feature are to
1. Enable users to select when a policy will be enforced
1. Enable users to select what subset of bound clusters the policy
   will be enforced to
1. Support progressive policy rollouts as described above
1. Give users visibility into the effect of a Configuration Policy
   change prior to remediation
1. Reduce complexity and scalability issues with alternative
   approaches

### Non-Goals

Out of scope for this feature
1. A mechanism/feature to capture/define the timing of enforcement
1. A feature for managing progressive rollouts. The timing and cluster
   selection are typically use-case specific. There may be enough
   commonality to define a feature to manage progressive rollouts but
   that is deferred to a separate discussion/enhancement.

## Proposal

The Selective Policy Enforcement feature extends the policy APIs to
allow enforcement of inform policies to selected clusters. The effect
of this would be the equivalent of setting the remediationAction of
child (cluster namespace copy) policies to enforce for those selected
clusters.

The proposal extends the API with `remediationActionOverride` and
includes two configurable elements as shown in this sample:

```
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
...
remediationActionOverride:
    remediationAction: [null|enforce]
    subFilter: [fale|true]
placementRef:
  apiGroup: apps.open-cluster-management.io
  kind: PlacementRule
  name: override-placementrule
subjects:
- apiGroup: policy.open-cluster-management.io
  kind: Policy
  name: test-policy-1
```

### Action override in PlacementBinding
The `remediationActionOverride.remediationAction` field allows users
to override the remediationAction specified in the bound Policy or
PolicySet. Valid values for this field include unset (no override) and
`enforce`.

The behavior of this field is:

+ An `enforce` override sets the remediationAction of the bound Policy
  to `enforce` for all clusters bound by this PlacementBinding.
+ The behavior of multiple PlacementBindings for the same policy and
  cluster remains unchanged (they are additive).
  + For reducing resources the implementation should de-duplicate any
    child policies created
+ A policy bound to the same cluster under multiple PlacementBindings
  will be set to `enforce` if _any_ of those bindings has
  `remediationActionOverride.remediationAction` set to enforce.
+ A policy with `remediationAction` already set to `enforce` is
  unaffected by a PlacementBinding with
  `remediationActionOverride.remediationAction` set to enforce.

### Sub filtering option in PlacementBinding
The `remediationActionOverride.remediationAction` field allows users
to control what clusters can be selected for a
remediationActionOverride. This field is a boolean with valid values
of `remediationActionOverride.subFilter: [false|true]`. To preserve
the existing semantics of multiple placement bindings attached to a
single policy this option defaults to `false`.

The behavior of the `remediationActionOverride` when the `subFilter`
is `true` is:

+ For the bound Policy or PolicySet, only clusters selected by another
  PlacementBinding where `subFilter` is false will be considered for
  `remediationAction` override.
  + A cluster must be bound by both this `subFilter: true` binding
    _and_ at least one `subFilter: false` PlacementBinding for the
    override to be applied.

The subFilter option is specified to allow a user to ensure that a
single PlacementBinding can be used to define the set of potentially
affected clusters. Without the subFilter option, it is possible to
erroneously select additional clusters in the "override"
PlacementBinding and accidentally enforce the policy to those
additional clusters. The `subFilter` option, when set true, ensures
that a second PlacementBinding used to enforce the policy to a smaller
number of clusters will respect the intent of the initial binding.

The behavior of the `remediationActionOverride` when the `subFilter`
is `false` is:

+ For the bound Policy or PolicySet, all selected clusters will have
  the `remediationAction` overridden.

`true` this option restricts the set of clusters considered for a
remediationActionOverride. When true, the set of clusters that are
considered when evaluating the PlacementRule for the specific
PlacementBinding is restricted to the set that are selected by all
other PlacementBindings for the policy where the `subFilter` field is
false (examples below).

### Examples
In the following examples an inform Policy is bound to
`placementrule-initial` which selects clusters [ "A", "B", "C", "D"
]. Additional placement rules are defined and used in the examples
below:
+ `placementrule-sub` selects clusters [ "A", "B" ]
+ `placementrule-sub-2` selects clusters [ "E", "F" ]
+ `placementrule-extended` selects clusters [ "A", "B", "E", "F" ]

#### Example 1: Simple selection
```
kind: PlacementBinding
placementRef:
  apiGroup: apps.open-cluster-management.io
  kind: PlacementRule
  name: placementrule-sub
remediationActionOverride:
    remediationAction: enforce
subjects:
- apiGroup: policy.open-cluster-management.io
  kind: Policy
  name: test-policy-1
```

The initial policy binds to clusters A, B, C, and D. The
placementrule-sub selects clusters A and B. Based on the override the
policy is switched to `enforce` mode for clusters A and B. The policy
remains in `inform` mode for clusters C and D.

#### Example 2: No sub filtering
```
kind: PlacementBinding
placementRef:
  apiGroup: apps.open-cluster-management.io
  kind: PlacementRule
  name: placementrule-extended
remediationActionOverride:
    remediationAction: enforce
    subFilter: false
subjects:
- apiGroup: policy.open-cluster-management.io
  kind: Policy
  name: test-policy-1
```

The `subFilter` field is false so all clusters in all placementrules
which are bound to the policy are considered. In this case the bound
PlacementRules are placementrule-initial and placementrule-extended
(in this binding).

Clusters A, B, C, D, E, and F are all set to enforce mode.

#### Example 3: With sub filtering
```
kind: PlacementBinding
placementRef:
  apiGroup: apps.open-cluster-management.io
  kind: PlacementRule
  name: placementrule-extended
remediationActionOverride:
    remediationAction: enforce
    subFilter: true
subjects:
- apiGroup: policy.open-cluster-management.io
  kind: Policy
  name: test-policy-1
```

The `subFilter` field is true in this example so only clusters in
placementrules with `subFilter` that is false are considered. In this
case that is only the placementrule-initial with clusters A, B, C, and
D. The PlacementRule in this CR is `placementrule-extended` which
includes A, B, E, and F.

The set of clusters set to enforce mode is the intersection of these
two sets. Clusters A and B are set to enforce mode.

#### Example 4: Multiple placements with sub filtering
```
kind: PlacementBinding
placementRef:
  apiGroup: apps.open-cluster-management.io
  kind: PlacementRule
  name: placementrule-sub-2
subjects:
- apiGroup: policy.open-cluster-management.io
  kind: Policy
  name: test-policy-1
---
kind: PlacementBinding
placementRef:
  apiGroup: apps.open-cluster-management.io
  kind: PlacementRule
  name: placementrule-extended
remediationActionOverride:
    remediationAction: enforce
    subFilter: true
subjects:
- apiGroup: policy.open-cluster-management.io
  kind: Policy
  name: test-policy-1
```

The `subFilter` field is true on the extended PlacementBinding and
false on both the initial and sub-2 PlacementBindings. The union of
the initial and sub-2 PlacementRules become the set from which the
extended PlacementBinding is subFiltering.

Clusters A, B, E, and F are set to enforce mode. C and D remain in
inform mode.

#### Example 5: No overlap
```
kind: PlacementBinding
placementRef:
  apiGroup: apps.open-cluster-management.io
  kind: PlacementRule
  name: placementrule-sub-2
remediationActionOverride:
    remediationAction: enforce
    subFilter: true
subjects:
- apiGroup: policy.open-cluster-management.io
  kind: Policy
  name: test-policy-1
```

The PlacementBinding is set to `subFilter: true`. The clusters under
consideration are those in placementrule-initial (A, B, C, and D). The
override PlacementBinding uses placementrule-sub-2 which includes
clusters E and F. There is no overlap between the non-subFilter
clusters and the subFilter set.

No clusters are set to enforce mode.

### Effect of policy changes
The behavior of policies when changes are made does not change from
current behavior when a remediationActionOverride is in effect. For
clusters switched to enforce mode by the override the change made in
the policy will "flow through" and be immediately enforced on the
selected clusters. For clusters bound with inform mode the policy will
immediately inform based on the changed content.

### User Stories

#### Story 1
I have a custom controller/operator which is responsible for applying
a configuration change to my fleet of clusters. SLAs require a
progressive rollout to my network. The rollout proceeds as follows:
1. I define the desired configuration of my clusters in a
   Configuration Policy which I bind to a large set of clusters in my
   network.
1. The policy is created in inform mode and I can see/verify the scope
   and impact of the changes.
1. My controller/orchestrator identifies that one or more clusters
   need to be remediated.
1. My controller/orchestrator creates a PlacementRule which is
   initially empty (selects no clusters)
1. My controller/orchestrator creates a a PlacementBinding with
   `remediationActionOverride.remediationAction: enforce` and
   `remediationActionOverride.subFilter: true`. The
   `placementRef.name` is set to the created PlacementRule.
1. My controller/orchestrator selects, based on its criteria, a set of
   clusters which should receive the configuration update.
1. My controller/orchestrator updates the created PlacementRule to
   this set of clusters
1. The GRC policy controller remediates the policy on only the
   selected clusters according to existing "enforce" mode rules. The
   other bound clusters continue evaluating the policy under "inform"
   mode rules.
1. My controller recognizes the child policy(ies) moving to the
   compliant state and removes the set of clusters from the created
   PlacementRule
1. The GRC policy controller reverts to existing "inform" mode
   handling of the policy for the selected, and now compliant,
   clusters.
1. My controller goes back to the cluster selection state and repeats,
   according to its defined timing/criteria, until all desired
   clusters are brought to compliance.

#### Story 2
I have a Configuration Policy which is bound to a large number of
ManagedClusters. I want to continuously see the compliance state of
this policy for all bound clusters. I want to, at a time of my
choosing, enforce this policy against a subset of those clusters. The
selection of clusters and timing of enforcing the policy must be under
my control.

#### Story 3
I want to roll out a new configuration to my network. The
configuration is captured in an inform policy so that the scope and
impact of the change can be observed and validated prior to applying
the change to the network. I want to propagate the change out to a
small number of clusters as an initial test and allow the change to
“soak” for a period of time. At a later time I want to propagate the
change to the rest of the network.

#### Story 4
I want to roll out a new configuration to my network. The
configuration is captured in an inform policy so that the scope and
impact of the change can be observed and validated prior to applying
the change to the network. I want to propagate the change to the
network in progressive waves to minimize overall network
impact/downtime.

### Implementation Details/Notes/Constraints [optional]

Several factors were considered when choosing this mechanism for
selecting clusters for enforcement.

+ Avoiding multiple controllers managing a CR. With approaches where
  the child Policy is directly manipulated by the user/orchestrator
  there are multiple owners of that portion of the CR which must be
  managed.
+ Supporting multiple users/controllers enabling enforcement. In the
  anticipated, large scale, use cases it is likely that multiple
  controllers (or a single controller with multiple instances) will
  seek to enforce policies. This approach allows each of those
  controllers to own a PlacementBinding/PlacementRule independent of
  the other controllers. With approaches manipulating the child policy
  there is only a single mechanism (CR) being affected by multiple
  controllers and conflicts would be difficult to resolve.
+ Familiar semantics. This approach preserves the existing behavior of
  multiple PlacementRule and PlacementBinding CRs applied to one
  policy when the subFilter setting is disabled (which is the default
  behavior). The additional, opt-in, APIs allow variants of the use
  cases to be supported without corrupting the existing semantics.
+ Supporting selection of groups of clusters. By using the existing
  placement APIs the user/orchestrator has the full range of logic to
  use when selecting clusters for enforcement. In large fleets of
  clusters this is particularly useful.

For several reasons the proposal provides a feature that could
otherwise be implemented using multiple policies, placement rules
and/or labels (see alternatives below). Some of those considerations
are:
+ Scalability of the OCM cluster. In a large fleet of clusters
  duplication of a policy (eg one inform and one enforce) results in
  additional policies to be managed and replicated to cluster
  namespaces. That additional load reduces overall capacity.
+ Complexity for the user. Managing a large fleet of clusters (1000's)
  by manipulating labels on clusters can be complex to maintain. The
  user would need tools/visibility to determine what subsets of
  clusters have a label. With multiple configuration changes in flight
  the overlapping sets of cluster labels could be difficult to manage.

The proposal seeks to add a more granular "enable" capability to
policy enforcement. It is hoped and anticipated that this feature
would be more broadly applicable than the specific use cases listed
here.

### Risks and Mitigation

1. Additional complexity in policy binding computation. This proposal
   adds complexity to the computation of bound policies. The policy
   logic needs to account for this computation.
1. When debugging the state of their system users will need to be
   aware of the remediationActionOverride. This is mitigated by the
   opt-in default state of this option. This risk can be further
   mitigated by additional information in the binding status of the
   policy

## Design Details

### Open Questions [optional]

1. When a policy is set to enforce mode the child policy is slightly
   out of sync with the parent policy due to the updated
   remediationAction. The controller needs to be aware of this
   discrepancy and not overwrite the remediationAction when:
  1. Syncing future changes to the parent policy
  1. Periodic audits or triggered watches (due to the change)

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
+ Will there be e2e and integration tests, in addition to unit tests?
+ How will it be tested in isolation vs with other components?

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

The proposal is an opt-in extension of the PolicyBinding. On upgrade
from a release without this feature there will be no use of this
additional feature.

On downgrade the new remediationActionOverride section would be
invalid and should be rejected by CR validation against the
PolicyBinding CRD. Users should be aware that use of a feature which
does not exist in a prior release may cause loss of data during
downgrade.

### Version Skew Strategy


## Implementation History

+ 2022-05-17 Initial proposal
+ 2022-07-14 Update with remediationActionOverride mechanism for
  cluster selection.

## Drawbacks



## Alternatives

### Alternative 1
Support multiple (eg list) remediationActions on a policy. The
placement API would reference a specific entry in the list thus
allowing separate decisions to be made on the inform vs enforce
action. This approach supports the use case. The user would need
to manage the additional placement APIs.

### Alternative 2
Directly manipulating the child policy remediationAction. As noted
above there are two issues with this approach.
+ When multiple controllers are taking advantage of the Selective
  Policy Enforcement feature it is very difficult to manage a single
  shared "enforce" switch.
+ Multiple controllers own/share the child policy CR. This requires
  careful attention to overwrite scenarios.

### Alternative 3
New CR, similar to PlacementBinding to represent “enforcement
action”. Uses the same Placement APIs (PlacementRule or newer
Placement) and binds with policy. The key here is to NOT create
additional APIs but to leverage existing ones.

*The following alternatives describe drawbacks of using existing policy
features to achieve the stated goals.*

### Existing Solution 1: Single copy of enforce policy
If a single ConfigurationPolicy with remediationAction of enforce is
used to manage this configuration a user would make configuration
changes directly in the active policy. Changes made to this policy
would immediately apply to all clusters in the fleet.

### Existing Solution 2: Single copy of policy, user toggles inform/enforce
In this approach there is a single copy of the policy, as in Existing
Solution 1, but the user toggles the remediationAction between inform
and enforce as a safeguard. This scenario provides pre-application
visibility but all clusters would be simultaneously updated when the
remediationAction is set to enforce.

### Existing Solution 3: Two copies of policy, rollout managed using labels
In this approach the “active” policy is bound to the full set of
clusters. New clusters are bound to the active policy when they
join. Updates/changes to configuration are captured in a modified copy
of the active policy. Clusters are progressively unbound from the
active policy and bound to the new policy. When all clusters have been
moved labels are adjusted so the new policy becomes the active policy
and newly deployed clusters will be initially configured with the
updated policy. This procedure can be used to achieve the goals
however there are several drawbacks:
+ Determining compliance of the fleet of clusters against the desired
  end-state requires the user to evaluate what clusters have the "new"
  label and which do not. If multiple changes are being made this will
  multiply the number of combinations of new vs old labels.
+ Managing labels on clusters may have unintended side effects when
  labels are used in more than one placement rule.
+ The fleet is not protected from "errant" changes to policies. The
  policies remain in enforce mode and unintended changes to the policy
  would immediately propagate to the entire fleet.
+ The impact of the change cannot be viewed through the policy. The
  set of affected clusters is under user control which minimizes risk
  of applying the change to the wrong cluster(s), however there is not
  opportunity to review the change (diff of actual vs desired) prior
  to moving the cluster to the enforce policy.

If these drawbacks are not an issue for a given use case this approach
can be used and is not invalidated by this proposal.

### Existing Solution 4: Single copy of inform policy (current approach)
In this approach a single copy of the configuration policy with
remediationAction set to “inform” is used. When the configuration
needs to be remediated a transient copy of the policy with
remediationAction set to “enforce” is made and bound to the target
cluster(s). When the cluster(s) are compliant the enforce copy is
deleted. This has several drawbacks:
+ The copied policies can be out of sync with the parent
  policies. This is particularly an issue when a mistake is made in
  the policy which makes it impossible to enforce. The parent policy
  is edited but the transient copy must be deleted/re-created or
  updated by the user or controller which created it.
+ Complexity. The user or orchestrator/controller must manage the
  lifetime of the transient copies.
+ Scalability. The additional policies create load on the hub cluster
  reducing peak management capacity by the maximum number of
  concurrent copies created.

## Infrastructure Needed [optional]

N/A
