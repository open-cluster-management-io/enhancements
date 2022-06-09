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

The Selective Policy Enforcement feature adds an API to the policy
framework to allow enforcement of inform policies to selected
clusters. The effect of this would be the equivalent of setting the
complianceType of child (cluster namespace copy) policies to enforce.

The proposed API would be the addition of a remediationAction of
“controlled” to Policy. In this mode the policy would act as an inform
policy until the user, controller, or higher level orchestration
enables remediation directly on the child policy.

The mechanism to enable enforcement of a child policy would be to add
an annotation to the child policy which enables enforcement

```
metadata:
  annotations:
    versioned-policy-enable: [any|<generation>]
```

The values of the annotation govern how the controller will handle
modifications to the parent policy specification while the child
policy is in enforcing mode.

The values of the annotation can take either of these forms:
+ The value of `any` indicates that changes to the parent policy are
immediately passed through to the enforcement of the child policy. As
long as this annotation has the value of `any` remediation is enabled
on the child policy and changes to the parent policy are immediately
enforced to the cluster.
+ An integer `generation` value indicates that the child policy will
continue enforcing as long as the child policy `metadata/generation`
value is equal to the specified value. In this way the user of this
feature has control over whether changes to the parent policy happen
immediately or require additional intervention by the user. At the
expense of this additional complexity the behavior closes race
conditions on a user/controller enabling/disabling enforcement and
near-concurrent changes to the parent policy specification.

The overall pattern follows a similar one in OLM operator updates
where a manual installPlanApproval is explicitly enabled by the user
when an update is desired.

The implementation is relatively simple and defers the “selection”
flexibility to the user/orchestrator/controller

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
1. My controller/operator selects, based on its criteria, a set of
   clusters which should receive the configuration update.
1. My controller/orchestrator annotates the child policy for the
   selected clusters with `versioned-policy-enable: <value>`
1. The GRC policy controller remediates the policy on only the
   selected clusters according to existing "enforce" mode rules. The
   other bound clusters continue evaluating the policy under "inform"
   mode rules.
1. My controller recognizes the child policy moving to the compliant
   state and removes the `versioned-policy-enable` annotation.
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

The proposal seeks to add this functionality without creating a
separate/parallel cluster-to-policy placement/decision. This would add
complexity and be error prone. The existing Policy placement logic
governs the association of policies with clusters. This proposal seeks
to sub-select from the existing placement a set of clusters for
modified remediationAction.

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

1. Additional complexity in policy remediation. This proposal adds a
   3rd state to policies ("controlled") which users would need to be
   aware of when investigating why clusters aren't in compliance. This
   risk is mitigated by the new state being "opt-in" and immediately
   visible in the Policy object remediationAction field.

## Design Details

### Open Questions [optional]

1. The mechanism of enabling a enforcement of the policy to a cluster
   involves directly manipulating the child policy. The child policy
   is, however, a direct copy of the parent policy. Manipulating the
   child brings it out of sync with the parent. The controller would
   need to be aware of this and not overwrite the annotation when:
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

The proposal is an opt-in extension of the remediationAction for a
policy. On upgrade from a release without this feature there will be
no use of this additional feature.

On downgrade the new remediationAction would be invalid and should be
rejected by CR validation against the Policy CRD. Users should be
aware that use of a feature which does not exist in a prior release
may cause loss of data during downgrade.

### Version Skew Strategy

If the policy controller on spoke clusters is out of sync with the hub
it is possible a controlled policy would be handled by a spoke which
does not support controlled policies. The spoke policy controller
would ignore the policy as malformed with an unknown
remediationAction. When the spoke policy controller is updated to the
current version the policy enforcement would occur as expected/defined
by the policy.

## Implementation History

2022-05-17 Initial proposal

## Drawbacks



## Alternatives

### Alternative 1
Support multiple (eg list) remediationActions on a policy. The
placement API would reference a specific entry in the list thus
allowing separate decisions to be made on the inform vs enforce
action. This approach supports the use case. The user would need
to manage the additional placement APIs.

### Alternative 2
Extending the placement API with a remediationAction override. As the
placement is evaluated the child policy remediationAction would be set
to the override value. To support a progressive rollout without
detaching most clusters from the policy there would need to be
multiple placement APIs (one for inform and one for enforce). This
approach would also require users to be aware that the
remediationAction in the Policy could be overridden when
debugging/evaluating when/why policies are applied. The controller
would also need to be aware of the intended difference between child
and parent policy.

### Alternative 3
New CR, similar to PlacementBinding to represent “enforcement
action”. Uses the same Placement APIs (PlacementRule or newer
Placement) and binds with policy. The key here is to NOT create a
parallel binding decision tree which could get out of sync with the
desired bindings. This would be a sub-selection from the already made
decision.  This preserves flexibility and re-uses existing placement
APIs, however it adds complexity to the remediation decision. A user
would have to ensure a specific cluster is included in both the
“binding” placement rule as well as the “enforcement” placement
rule. This is likely to be error prone and adds a level of indirection
that the user would need to be aware of.

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
