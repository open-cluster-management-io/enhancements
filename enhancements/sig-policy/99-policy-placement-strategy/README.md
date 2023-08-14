# Policy placement strategy

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
      [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

Following from the [`DecisionStrategy` enhancement](../../sig-architecture/64-placementStrategy/README.md) in the
`Placement` API, policies can leverage this new logic to have a configurable and systematic way to roll out policy
updates to clusters.

## Motivation

Currently policies and subsequent updates are pushed out to clusters en masse based on the placement to which it has
been bound. The new `DecisionStrategy` field in the `Placement` API creates cluster groupings that controllers can
leverage to have segmented, configurable rollouts of resources. This will aid use cases where high availability may be a
priority or a set of test clusters should receive and verify the updates before the remaining clusters get updated.

### Goals

- Make `Placement` the primary API for placement (currently the governance propagator is somewhat
  `PlacementRule`-centric)
- Leverage the `Placement` helper library to retrieve placement decisions
- Reflect rollout status per cluster in the root policy for discoverability (whether a cluster is up-to-date or not)
- Implement the `RolloutStrategy` struct for policies, including:
  - `RolloutStrategy`
    - "All": all clusters at once
    - "Progressive": one cluster at a time
    - "ProgressivePerGroup": one group of clusters at a time
  - `ProgressDeadline`: Maximum amount of time to wait for status before timing out
  - `MinSuccessTime`: Minimum amount of time to wait before proceeding (i.e. "soak" time)
  - `MaxFailures`: Tolerance of percentage or number of clusters that can fail and still continue the rollout
  - `MandatoryDecisionGroups`: groups that should be handled before other groups
  - `MaxConcurrency`: Concurrency during a progressive rollout
- Add an aggregated rollout status for the root policy status.

### Non-Goals

Any specialized behaviors outside of those provided by the `Placement` library and, by extension, the `DecisionStrategy`
enhancement, that might require additional code other than that already provided/required by the `Placement` library.
This includes the ability to roll back a rollout. A rollback should be implemented by the user by applying the previous
version of the policy, and GitOps is the recommended method for this.

## Proposal

### Background

**Note:** This enhancement references the previous
[`DecisionStrategy` enhancement](../../sig-architecture/64-placementStrategy/README.md) in the `Placement` API, so it is
recommended to review it before proceeding.

In current policy flows, users create policies on the hub cluster (which are referred to as the root policy). These
policies are replicated by the `governance-policy-propagator` controller to managed cluster namespaces based on the
placement to which they have been bound by the associated `PlacementBinding` object. The `governance-policy-framework`
sync controllers on managed clusters monitor the associated managed cluster namespace on the hub for policy updates.

Additionally, policies have a `remediationAction` specified, which can either be `inform`, meaning the policy controller
monitors objects on the managed cluster reporting status without making changes, or `enforce`, meaning the policy
controller attempts to make changes on the managed cluster based on the policy definition and reports status accordingly
based on the success or failure of the update.

### Design

Upon policy creation or update, all clusters receive the same version, preventing version skew among different clusters.
However, the `remediationAction` for enforced policies is initially set to `inform` and the rollout occurs as each
cluster in turn has its policy updated to `enforce`. For each rollout strategy, this looks like:

| Rollout Strategy      | Remediation Inform                      | Remediation Enforce                                                                                                                                                            |
| --------------------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `All`                 | Policy rolled out to all clusters as-is | Policy rolled out to all clusters as-is                                                                                                                                        |
| `Progressive`         | (Same as `All`)                         | Policy applied initially to all clusters as `inform` and <br/>then serially enforces each cluster's policy and moves on to the next cluster once compliant (or timed out)      |
| `ProgressivePerGroup` | (Same as `All`)                         | Policy applied initially to all clusters as `inform` and <br/>rollout enforces a cluster group's policy and waits for compliant (or timed out) before moving to the next group |

Given this, policy propagation by the `governance-policy-propagator` controller on the hub takes place in a
multiple-pass approach:

1. First pass: Replicate policies to all clusters set with `remediationAction: inform`

   On the first pass following a root policy creation or update, the `governance-policy-propagator` controller on the
   hub cluster will replicate all policies to bound managed cluster namespaces and are created regardless of the rollout
   strategy specified, with the remediation action set to `inform` (the exception being if the rollout strategy is
   `All`). This way, all policies are up-to-date and reporting a compliance status based on the current version of the
   root policy. The aggregated rollout status on the root policy is set to `Progressing`. If the remediation action on
   the root policy is already `inform` or the strategy is `All`, the rollout status on each cluster's policy is set to
   `Progressing` and no second pass occurs. If the remediation action is `enforce`, the rollout status on each cluster's
   policy is set to `ToApply`.

2. Subsequent passes: Use the given rollout strategy to set `remediationAction: enforce`, stopping the rollout on
   non-compliance

   On the subsequent passes, if the `remediationAction` is `enforce` and has a progressive rollout strategy, the
   `governance-policy-propagator` fetches the managed clusters returned from the `Placement` library (based on the
   user-configured rollout strategy) and sets the rollout status of each returned cluster to `Progressing`, updating any
   `remediationAction` to `enforce` as defined in the root policy.

From here, work is picked up by the `governance-policy-framework` controllers on the managed clusters:

1. The template sync controller of the `governance-policy-framework` updates the applicable `ConfigurationPolicy`
   objects, which triggers reevaluation from the `config-policy-controller`.
2. Once the `ConfigurationPolicies` have a generation that matches the `lastEvaluatedGeneration` set on the replicated
   policy status, the status sync controller of the framework will set the rollout status on the replicated policy on
   the managed cluster and the replicated policy on the hub to `Succeeded` after all of the `ConfigurationPolicies` have
   a compliant status or `Failed` if there is any non-compliant status. (The `governance-policy-propagator` will also
   set the rollout status accordingly on the root policy for that managed cluster.)

Bringing this together, the rollout status of a managed cluster policy is defined as follows:

| Status        | Description                                                                                                                                                                                                                                          |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ToApply`     | A policy to be enforced in a progressive rollout is created/updated in the first pass of the propagator on the hub                                                                                                                                   |
| `Progressing` | Policy has no status or the `lastEvaluatedGeneration` does not match the generation on the managed cluster, and one of: Rollout is `All`, policy was defined as `inform`, or policy was selected by the `Placement` library and updated to `enforce` |
| `Succeeded`   | Policy is compliant and the `lastEvaluatedGeneration` matches the generation on the managed cluster                                                                                                                                                  |
| `Failed`      | Policy is non-compliant and the `lastEvaluatedGeneration` matches the generation on the managed cluster                                                                                                                                              |
| `TimeOut`     | Time has passed beyond the timeout specified in the `RolloutStrategy` without a returned status                                                                                                                                                      |
| `Skip`        | (unused)                                                                                                                                                                                                                                             |

And the aggregated root policy status would be: `Progressing` (this would include `ToApply`), `Succeeded`, or `Failed`.
(The `TimeOut` status would not be reflected.)

#### Handling managed cluster additions to the placement

When a new managed cluster is added to the placement, if the rollout is `Progressing`, it will be handled with the
current rollout where the `Placement` library will abstract this and return the cluster at the appropriate time
according to the rollout strategy. If the rollout is `Succeeded`, the `governance-policy-propagator` will update the
rollout status on the root policy back to `Progressing` and handle only the newly added managed clusters in a "miniature
rollout" as described in the [Design section](#design).

#### Handling complexities around the `PlacementBinding`

There is some additional complexity brought into the rollout because policies are bound to `Placement` using
`PlacementBindings`, so there could be multiple `Placements` bound to each policy. To remedy this, `All` and
`Progressive` (which is per-cluster) would be handled by pulling in all `PlacementDecisions` from all `Placements`. In
the case of multiple `Placements` where `ProgressivePerGroup` is specified, this would be handled by taking groups from
one placement and then moving on to the next placement.

Additionally, the `PlacementBinding` has specifications for selective policy enforcement to override the
`remediationAction` and selectively apply policies to managed clusters. (See the
[Selective Policy Enforcement enhancement](https://github.com/open-cluster-management-io/enhancements/tree/main/enhancements/sig-policy/28-selective-policy-enforcment).)
Since these may conflict with the rollout and has similar intentions, these specifications will be ignored if they are
provided with a policy specifying a rollout strategy other than `All`.

### User Stories

#### Story 1

As a system administrator, I want to know the status of a rollout for a policy and what clusters have been updated.

- **Summary**

  - Add a `RolloutStatus` to the `Policy` status in the CRD.

- **Details**

  - The `RolloutStatus` is added to the status on the replicated policy.
  - The `RolloutStatus` is added on the root policy per-cluster and as an aggregated status.

  (For technical notes see [Development Phase 1](#development-phase-1))

- **Snippet**

  **Root policy status**

  ```yaml
  status:
    compliant: Compliant
    rolloutStatus: Progressing # <-- New field
    placement:
      - placement: sample-placement
        placementBinding: sample-placementbinding
    status:
      - clustername: cluster1
        clusternamespace: cluster1
        compliant: Compliant
        rolloutStatus: Succeeded # <-- New field
      - clustername: cluster2
        clusternamespace: cluster2
        compliant: ""
        rolloutStatus: Progressing
      - clustername: cluster3
        clusternamespace: cluster3
        compliant: ""
        rolloutStatus: ToApply
  ```

  **Replicated policy status**

  ```yaml
  status:
    compliant: Compliant
    rolloutStatus: Succeeded # <-- New field
    details:
      - compliant: Compliant
        history:
          - ...
  ```

#### Story 2

As a system administrator, I want to specify a placement `DecisionStrategy` bound with policies, configure the order of
the updates, and continue enforcement rollout only when the policy on previous clusters show as "Compliant".

- **Summary**

  - Policies are initialized with `remediationAction: inform` and enforced policies are updated to
    `remediationAction: enforce` according to the `rolloutStrategy` defined on the policy.
  - Policy enforcement is continued only once the previously deployed clusters show as "Compliant".

  (For technical notes see [Development Phase 2](#development-phase-2) and [Development Phase 3](#development-phase-3))

- **Snippet**

  **Policy with `rolloutStrategy`**

  (See the
  [DecisionStrategy enhancement](https://github.com/open-cluster-management-io/enhancements/tree/main/enhancements/sig-architecture/64-placementStrategy#how-workload-applier-apis-will-benefits-from-using-placementstrategy)
  for details on the `RolloutStrategy` struct.)

  ```yaml
  apiVersion: policy.open-cluster-management.io/v1
  kind: Policy
  metadata:
    name: sample-policy
    namespace: policy-namespace
  spec:
    disabled: false
    remediationAction: enforce
    rolloutStrategy:               # <-- New field
      type: ProgressivePerGroup
    policy-templates:
      - objectDefinition:
          apiVersion: policy.open-cluster-management.io/v1
          kind: ConfigurationPolicy
          ...
  ---
  apiVersion: policy.open-cluster-management.io/v1
  kind: PlacementBinding
  metadata:
    name: sample-placementbinding
    namespace: policy-namespace
  placementRef:
    name: sample-placement
    apiGroup: cluster.open-cluster-management.io
    kind: Placement
  subjects:
    - name: sample-policy
      apiGroup: policy.open-cluster-management.io
      kind: Policy
  ```

  **Replicated policy in managed cluster namespace**

  ```yaml
  apiVersion: policy.open-cluster-management.io/v1
  kind: Policy
  metadata:
    name: policy-namespace.sample-policy
    namespace: cluster1
  spec:
    disabled: false
    remediationAction: inform # <-- Remediation action updated to prepare for rollout
    rolloutStrategy:
      type: ProgressivePerGroup
      progressivePerGroup:
        minSuccessTime: 5m
        progressDeadline: 10m
        maxFailures: 2%
        mandatoryDecisionGroups: []
    policy-templates:
      - objectDefinition:
          spec:
            apiVersion: policy.open-cluster-management.io/v1
            kind: ConfigurationPolicy
            ...
  status:
    compliant: ""
    rolloutStatus: ToApply # <-- Policy is waiting to be enforced
  ```

  **Placement**

  (provided for reference--this enhancement doesn't intend to update the Placement API. See the
  [DecisionStrategy enhancement](https://github.com/open-cluster-management-io/enhancements/tree/main/enhancements/sig-architecture/64-placementStrategy#placement-api)
  for details on the `DecisionStrategy` struct.)

  ```yaml
  apiVersion: cluster.open-cluster-management.io/v1beta1
  kind: Placement
  metadata:
    name: sample-placement
    namespace: policy-namespace
  spec:
    predicates:
      - requiredClusterSelector:
          labelSelector:
            matchExpressions:
              - key: environment
                operator: In
                values:
                  - dev
    decisionStrategy:
      groupStrategy:
        clustersPerDecisionGroup: 10
        DecisionGroups:
          - groupName: dev-emea
            clusterSelector:
              matchExpressions:
                - key: dev-emea
                  operator: Exist
          - groupName: dev-apac
            clusterSelector:
              matchExpressions:
                - key: dev-apac
                  operator: Exist
  ```

### Implementation Details

#### Development Phase 1

- **Summary**

  - Add a `RolloutStatus` to the `Policy` status in the CRD.

- **Details**

  - The `RolloutStatus` would be added to reflect: rollout status on the replicated policy, per-cluster on the root
    policy, and an aggregated status on the root policy. The aggregated root policy status would be: `Progressing` (this
    would include `ToApply`), `Succeeded`, or `Failed`. (The `TimeOut` status would not be reflected.)

- **Snippet**

  ```golang
  // CompliancePerClusterStatus defines compliance per cluster status
  type CompliancePerClusterStatus struct {
    ComplianceState  ComplianceState               `json:"compliant,omitempty"`
    RolloutStatus    clusterv1alpha1.RolloutStatus `json:"rolloutStatus,omitempty"` // <-- New field
    ClusterName      string                        `json:"clustername,omitempty"`
    ClusterNamespace string                        `json:"clusternamespace,omitempty"`
  }

  // PolicyStatus defines the observed state of Policy
  type PolicyStatus struct {
    Placement []*Placement                  `json:"placement,omitempty"`
    Status    []*CompliancePerClusterStatus `json:"status,omitempty"`

    // +kubebuilder:validation:Enum=Compliant;Pending;NonCompliant
    ComplianceState ComplianceState               `json:"compliant,omitempty"`
    RolloutStatus   clusterv1alpha1.RolloutStatus `json:"rolloutStatus,omitempty"` // <-- New field
    Details         []*DetailsPerTemplate         `json:"details,omitempty"`
  }

  // Policy is the Schema for the policies API
  // +kubebuilder:subresource:status
  // +kubebuilder:resource:path=policies,scope=Namespaced
  // +kubebuilder:resource:path=policies,shortName=plc
  // +kubebuilder:printcolumn:name="Remediation action",type="string",JSONPath=".spec.remediationAction"
  // +kubebuilder:printcolumn:name="Compliance state",type="string",JSONPath=".status.compliant"
  // +kubebuilder:printcolumn:name="Rollout status",type="string",JSONPath=".status.rolloutStatus" // <-- New printer column
  // +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"
  type Policy struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata"`

    Spec   PolicySpec   `json:"spec"`
    Status PolicyStatus `json:"status,omitempty"`
  }
  ```

#### Development Phase 2

- **Summary**

  - Use the `Placement` library to gather the list of clusters and iterate over them.

- **Details**

  - Make `Placement` the primary placement resource and parse `PlacementRule` decisions into the `Placement` struct
    instead of the `PlacementRule` struct.
  - Using the `Placement` library and implementing a `ClusterRolloutStatusFunc`, iterate over all clusters as usual
    using the `All` rollout strategy.

- **Snippet (untested)**

  ```golang
  // common.go
  type placementDecisionGetter struct {
    c client.Client
  }

  func (pd placementDecisionGetter) List(selector labels.Selector, namespace string) ([]*clusterv1beta1.PlacementDecision, error) {
    pdList := &clusterv1beta1.PlacementDecisionList{}
    lopts := &client.ListOptions{
      LabelSelector: selector,
      Namespace:     namespace,
    }

    err := pd.c.List(context.TODO(), pdList, lopts)
    if err != nil {
      return nil, err
    }

    pdPtrList := []*clusterv1beta1.PlacementDecision{}
    for _, pd := range pdList.Items {
      pdPtrList = append(pdPtrList, &pd)
    }

    return pdPtrList, err
  }

  func GetRolloutHandler(c client.Client, placement *clusterv1beta1.Placement) (*clusterv1alpha1.RolloutHandler, error) {
    pdTracker := clusterv1beta1.NewPlacementDecisionClustersTracker(placement, placementDecisionGetter{c}, nil, nil)

    _, _, err := pdTracker.Get()
    if err != nil {
      log.Error(err, "Error retrieving PlacementDecisions from tracker")
    }

    return clusterv1alpha1.NewRolloutHandler(pdTracker)
  }

  var GetClusterRolloutStatus clusterv1alpha1.ClusterRolloutStatusFunc = func(clusterName string) clusterv1alpha1.ClusterRolloutStatus {
    // Fetch and return the rollout status and the last transition time from the status in the replicated policy

    return clusterv1alpha1.ClusterRolloutStatus{
      Status: "<rollout-status>",
      LastTransitionTime: "<transition-time>",
    }
  }
  ```

  ```golang
  rolloutHandler := common.GetRolloutHandler(client, placement)
  strategy, rolloutResult, err := rolloutHandler.GetRolloutCluster(clusterv1alpha1.All, common.GetClusterRolloutStatus)

  // ... Use rolloutResults for propagation
  ```

#### Development Phase 3

- **Summary**

  - Only continue policy enforcement once the previously deployed clusters show as "Compliant".

- **Details**

  - Update the `Policy` CRD to contain the `RolloutStrategy` struct. (See the
    [`v1alpha1/RolloutStrategy`](https://github.com/open-cluster-management-io/api/blob/main/cluster/v1alpha1/types_rolloutstrategy.go))
    Defaults to `All` if a strategy is not provided or the remediation action is `inform`.
  - On create/update, replicate policy to all managed clusters with `remediationAction: inform` (if the rollout strategy
    isn't `All`).
  - When the `remediationAction` is set to `enforce`, policies not currently being rolled out will be set to `inform` to
    continue to report status without making changes on the managed cluster while waiting for the new version of the
    policy to be enforced.
  - Update the `ClusterRolloutStatusFunc`, (the `GetClusterRolloutStatus` function from
    [Development Phase 2](#development-phase-2)) to fetch the rollout status and last transition time from the
    replicated policy status. (See the [Design section](#design).)

    | Status        | Description                                                                                                                                           |
    | ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
    | `ToApply`     | A policy to be enforced in a progressive rollout is created/updated in the first pass of the propagator on the hub                                    |
    | `Progressing` | Rollout is `All`, policy was defined as `inform`, or policy was selected by the `Placement` library and updated to `enforce`                          |
    | `Succeeded`   | Policy was defined as `inform` and has status, or status is compliant and the `lastEvaluatedGeneration` matches the generation on the managed cluster |
    | `Failed`      | Policy is non-compliant and the `lastEvaluatedGeneration` matches the generation on the managed cluster                                               |
    | `TimeOut`     | Time has passed beyond the timeout specified in the `RolloutStrategy` without a returned status                                                       |
    | `Skip`        | (unused)                                                                                                                                              |

- **Snippet**

  **Note:** The `RolloutStrategy` is a partial struct from the `Placement` library. It's implemented manually here so
  that we can add features like `MandatoryDecisionGroups` and `Concurrency` at a later time.

  ```golang
  // PolicySpec defines the desired state of Policy
  type PolicySpec struct {
    // This provides the ability to enable and disable your policies.
    Disabled bool `json:"disabled"`
    // If set to true (default), all the policy's labels and annotations will be copied to the replicated policy.
    // If set to false, only the policy framework specific policy labels and annotations will be copied to the
    // replicated policy.
    // +kubebuilder:validation:Optional
    CopyPolicyMetadata *bool `json:"copyPolicyMetadata,omitempty"`
    // This value (Enforce or Inform) will override the remediationAction on each template
    RemediationAction RemediationAction `json:"remediationAction,omitempty"`
    // Used to create one or more policies to apply to a managed cluster
    PolicyTemplates []*PolicyTemplate `json:"policy-templates"`
    // PolicyDependencies that apply to each template in this Policy
    Dependencies []PolicyDependency `json:"dependencies,omitempty"`
    // +kubebuilder:validation:Optional
    RolloutStrategy clusterv1alpha1.RolloutStrategy `json:"rolloutStrategy,omitempty"` // <-- New field
  }
  ```

### Notes/Constraints

For the `Placement` library, this requires importing at least this package version (the `Placement` library is in the
`v1alpha1` version):

```
  open-cluster-management.io/api v0.11.1-0.20230828015110-b39eb9026c6e
```

For testing, the `governance-policy-propagator` doesn't currently account for multiple managed clusters. As part of this
enhancement, the test flows would need to be enhanced (and/or a separate workflow created) that deploys multiple managed
clusters.

The `PlacementDecisionGetter` returns an array of pointers (`[]*clusterv1beta1.PlacementDecision`) because it was
intended to retrieve from a cache, so the implementation could consider setting up a `PlacementDecision` cache instead
of using the Kubernetes client directly.

### Risks and Mitigation

- The `Placement` library is relatively new and untested outside of its repo and this implementation leans heavily on
  its logic. While it works in theory, there could be some tweaks/adjustments as development proceeds, lengthening time
  for implementation. The phased approach intends to address this to make partial implementation feasible.

## Design Details

### Open Questions

1. Should the per-cluster status on the root policy be grouped similar to how they're grouped in the
   `PlacementDecisions`?

### Test Plan

- Unit tests in the repo
- E2E tests in the repo (would need to add additional managed clusters for this, potentially in a separate workflow,
  though ideally alongside the existing tests)
- Policy integration tests in the framework (would need to add additional managed clusters for this, potentially in a
  separate workflow, though ideally alongside the existing tests)

## Drawbacks / Alternatives

The maturity of the `Placement` library could be brought into question. This enhancement could hold after migrating to
the `Placement` library in Story 2 and only support the `All` strategy until we ensure that we have a robust test
environment that can test the various configurations before moving forward.

An alternative to having the `RolloutStrategy` on the `Policy` is to have it on the `PlacementBinding`. Having the
rollout strategy defined in `PlacementBinding` would create unnecessary complexity because a policy could be bound to
different placements with different strategies.
