# Progressive Policy Rollout

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
      [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

Open Cluster Management policies will support the
[rollout strategy API](https://open-cluster-management.io/concepts/placement/#rollout-strategy) with the additional
strategy of `manualPerGroup`. This enhancement allows control over the conditions for when a new policy or update to an
existing policy is deployed to the Kubernetes fleet in customizable increments.

## Motivation

Currently, new policies and updates to existing policies instantly deploy to the clusters selected by the bound
placement. To do this progressively requires copies of policies or complex templating, and manual management of the
placements. In complex environments, this can be impractical. The
[decisionStrategy](https://open-cluster-management.io/concepts/placement/#decision-strategy) placement API allows for
defining groups of the selected clusters. The
[rollout strategy API](https://open-cluster-management.io/concepts/placement/#rollout-strategy) defines the conditions
for how a policy change deploys to the environment.

### Goals

- Progressive rollout of new policies and updates to policies based on the
  [decisionStrategy](https://open-cluster-management.io/concepts/placement/#decision-strategy) field in the `Placement`
  API
- Support clusters added to the placement during a rollout
- Reflect the rollout status in the root and replicated policies for discoverability
- Allow for manual approval of each decision group
- Leverage the `Placement` helper library to retrieve placement decisions

### Non-Goals

- **Rollbacks** - Reverting a policy does not necessarily rollback the change. It's up to the user to determine how to
  rollback in the event of a rollout failure.
- **Group Rollouts** - Rollouts of a group of policies is not a goal at this moment due to technical and user experience
  complexity. The policy templates in the `policy-templates` array within the policy (e.g. multiple
  `ConfigurationPolicy`) does rollout at as group.

## Proposal

### Background

**Note:** This enhancement references the previous
[`DecisionStrategy` enhancement](../../sig-architecture/64-placementStrategy/README.md) in the `Placement` API, so it is
recommended to review it before proceeding.

In current policy flows, users create policies on the hub cluster (which are referred to as the root policy). These
policies are replicated by the `governance-policy-propagator` controller to managed cluster namespaces based on the
placement to which they have been bound by the associated `PlacementBinding` object. The `governance-policy-framework`
sync controllers on managed clusters watch the associated managed cluster namespace on the hub for policy updates.

### API Design

#### Rollout Strategy API Integration

Open Cluster Management policies will now have a new field of `spec.rolloutStrategy` that is based on the
[rollout strategy API](https://open-cluster-management.io/concepts/placement/#rollout-strategy). An example is shown
below.

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
    progressivePerGroup:
      minSuccessTime: 5m
      progressDeadline: 10m
      maxFailures: 2%
      ...
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        ...
```

The default `spec.rolloutStrategy.type` will be `All` which is the same as the current behavior of deploying to all
clusters.

#### manualPerGroup Rollout Strategy

In addition to the standard options in the
[rollout strategy API](https://open-cluster-management.io/concepts/placement/#rollout-strategy), there will be an
additional `spec.rolloutStrategy.type` option of `manualPerGroup` and will be contributed to the upstream rollout
strategy API. This acts like `progressivePerGroup` except the user has manual control for when a group is available to
receive the new policy version. The order of the groups in the `Placement` is not seen as a dependency chain but will
act as a tiebreaker for the order of rollout when multiple groups are approved.

This is to account for complex environment requirements that may not be conducive to automated rules. For example, a
group of clusters that handle credit card machines are in a change freeze between the United States Thanksgiving holiday
and Christmas due to peak shopping, however, other clusters may have low usage during that time and is an ideal period
for deployments.

To control these groups, the user will interact with a new Kubernetes resource of `Rollout` as shown below.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Rollout
metadata:
  name: policy-<same as root policy>
  namespace: <same as root policy>
spec:
  approvalsForVersion: "2.0" <-- optional if the user wants to tie the approval to a specific policy "version".
  decisionGroups:
    - groupName: dev
      rolloutApproved: true <-- user sets this to true when ready
    - groupName: uat
      rolloutApproved: false
    - groupName: prod
      rolloutApproved: false
  ungrouped:
    rolloutApproved: false
```

This control is separate from the policy's `spec.rolloutStrategy` because the `status` field is required for the
technical implementation as described in another section. Having it separate also allows control of the rollout in a
console or CLI even if the policy is managed by GitOps. Additionally, if the policy applies to multiple hubs, the groups
may be different on those hubs.

The name, with a `policy-` prefix, and the namespace of the `Rollout` needs to be the same as the root policy it refers
to. This object will be automatically created by the Governance Policy Propagator with only the `status` and
`metadata.ownerReferences` fields set for storing state.

The `spec.decisionGroups` field is set by the user to indicate which groups are approved, and the group names are
discoverable in the `Rollout` status. These groups directly match those in the
[decisionStrategy](https://open-cluster-management.io/concepts/placement/#decision-strategy) field in the `Placement`
API. If the user wants a subset of the groups to be automatically approved, they would add the groups to the policy's
`spec.rolloutStrategy.manualPerGroup.mandatoryDecisionGroups` array.

If the user wants the approval to be conditional on a policy version, the user must set `spec.approvalsForVersion` to a
value that matches the `policy.open-cluster-management.io/version` annotation on the root policy. By default, the
approvals are not tied to a version and the `policy.open-cluster-management.io/version` annotation is ignored.

If the decision groups do not capture all clusters in the placement predicate, the user can set
`spec.ungrouped.rolloutApproved` to `true` to approve those clusters.

### Status

#### Replicated Policy Status

A replicated policy is the policy copy in the managed cluster namespace. Each replicated policy will have a rollout
status defined as follows:

| Status        | Description                                                                                           |
| ------------- | ----------------------------------------------------------------------------------------------------- |
| `ToApply`     | The cluster is waiting for the new version of the policy but the old version is still applied.        |
| `Progressing` | The new version of the policy has been deployed but the rollout status is not yet known.              |
| `Succeeded`   | The policy has confirmed it is compliant.                                                             |
| `Failed`      | The policy is still noncompliant after `progressDeadline` has elapsed.                                |
| `TimeOut`     | Time has passed beyond the timeout specified in `spec.rolloutStrategy` without a returned compliance. |
| `NewCluster`  | The policy was applied to the cluster after the rollout completed.                                    |
| `Skip`        | (unused)                                                                                              |

Below is an example of the replicated policy rollout status:

```yaml
status:
  compliant: Compliant
  rolloutStatus: Succeeded # <-- New field
  details:
    - compliant: Compliant
      history:
        - ...
```

#### Root Policy Status

The root policy is the `Policy` object defined by the user. It will contain the rollout status for each cluster and an
overall rollout status for the policy. View the following example:

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

### Rollout Behavior

The majority of the rollout behavior is configurable and described in the
[rollout strategy API](https://open-cluster-management.io/concepts/placement/#rollout-strategy). This sections focuses
on the policy specific behavior.

The following sections will describe scenarios and assume the following policy and placement. Notice the
`spec.rolloutStrategy.type` field is set to `ProgressivePerGroup`.

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: sample-policy
  namespace: policy-namespace
spec:
  disabled: false
  remediationAction: enforce
  rolloutStrategy:
    type: ProgressivePerGroup
    progressivePerGroup:
      progressDeadline: 10m
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        ...
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: sample-placement
  namespace: policy-namespace
spec:
  decisionStrategy:
    groupStrategy:
      decisionGroups:
        - groupName: dev
          clusterSelector:
            matchExpressions:
              - key: env
                operator: In
                values:
                  - dev
        - groupName: stage
          clusterSelector:
            matchExpressions:
              - key: env
                operator: In
                values:
                  - stage
        - groupName: prod
          clusterSelector:
            matchExpressions:
              - key: env
                operator: In
                values:
                  - prod
```

#### New Policy Succeeded

1. The policy applies to all clusters in the `dev` group. These clusters have the `rolloutStatus` of `InProgress`. The
   remaining clusters have the `rolloutStatus` of `ToApply`. The root policy `status.rolloutStatus` value is
   `InProgress`.
1. All the clusters in the `dev` group become compliant. Their `rolloutStatus` is set to `Succeeded`.
1. The policy applies to all clusters in the `stage` group. These clusters have the `rolloutStatus` of `InProgress`.
1. All the clusters in the `stage` group become compliant. Their `rolloutStatus` is set to `Succeeded`.
1. The policy applies to all clusters in the `prod` group. These clusters have the `rolloutStatus` of `InProgress`.
1. All the clusters in the `prod` group become compliant. Their `rolloutStatus` is set to `Succeeded`. The root policy
   `status.rolloutStatus` value is `Succeeded`.

#### New Policy Failed

##### First Rollout

1. The policy applies to all clusters in the `dev` group. These clusters have the `rolloutStatus` of `InProgress`. The
   remaining clusters have the `rolloutStatus` of `ToApply`. The root policy `status.rolloutStatus` value is
   `InProgress`.
1. All the clusters in the `dev` group become compliant. Their `rolloutStatus` is set to `Succeeded`.
1. The policy applies to all clusters in the `stage` group. These clusters have the `rolloutStatus` of `InProgress`.
1. One cluster in the `stage` group stays noncompliant for the duration of the `spec.rolloutStrategy.progressDeadline`
   of 10 minutes. Its `rolloutStatus` is set to `Failed`. The root policy `status.rolloutStatus` value is `Failed`. If
   `spec.rolloutStrategy.progressDeadline` was not set, it would have waited indefinitely and never failed.

##### Second Rollout

1. The root policy is updated.
1. The updated policy applies to all clusters in the `dev` group. These clusters have the `rolloutStatus` of
   `InProgress`. The remaining clusters have the `rolloutStatus` of `ToApply` and keep whatever policy version, if any,
   from the failed rollout. The root policy `status.rolloutStatus` value is `InProgress`.
1. All the clusters in the `dev` group become compliant. Their `rolloutStatus` is set to `Succeeded`.
1. The updated policy applies to all clusters in the `stage` group. These clusters have the `rolloutStatus` of
   `InProgress`.
1. All the clusters in the `stage` group become compliant. Their `rolloutStatus` is set to `Succeeded`.
1. The updated policy applies to all clusters in the `prod` group. These clusters have the `rolloutStatus` of
   `InProgress`.
1. All the clusters in the `prod` group become compliant. Their `rolloutStatus` is set to `Succeeded`. The root policy
   `status.rolloutStatus` value is `Succeeded`.

#### Updated Policy Succeeded

1. The root policy is updated.
1. The updated policy applies to all clusters in the `dev` group. These clusters have the `rolloutStatus` of
   `InProgress`. The remaining clusters have the `rolloutStatus` of `ToApply` and keep the previous policy version. The
   root policy `status.rolloutStatus` value is `InProgress`.
1. All the clusters in the `dev` group become compliant. Their `rolloutStatus` is set to `Succeeded`.
1. The updated policy applies to all clusters in the `stage` group. These clusters have the `rolloutStatus` of
   `InProgress`.
1. All the clusters in the `stage` group become compliant. Their `rolloutStatus` is set to `Succeeded`.
1. The updated policy applies to all clusters in the `prod` group. These clusters have the `rolloutStatus` of
   `InProgress`.
1. All the clusters in the `prod` group become compliant. Their `rolloutStatus` is set to `Succeeded`. The root policy
   `status.rolloutStatus` value is `Succeeded`.

#### Updated Policy Failed

1. The root policy is updated.
1. The updated policy applies to all clusters in the `dev` group. These clusters have the `rolloutStatus` of
   `InProgress`. The remaining clusters have the `rolloutStatus` of `ToApply` and keep the previous policy version. The
   root policy `status.rolloutStatus` value is `InProgress`.
1. One cluster in the `dev` group stays noncompliant for the duration of the `spec.rolloutStrategy.progressDeadline` of
   10 minutes. The root policy `status.rolloutStatus` value is `Failed`. The `dev` group still keeps the policy version
   of the failed rollout. All remaining groups stay with the last successful version.

#### Updated Policy With manualPerGroup in Order

1. The root policy is updated.
1. All clusters have the `rolloutStatus` of `ToApply` and keep the previous policy version. The root policy
   `status.rolloutStatus` value is `ToApply`.
1. The user sets the `spec.decisionGroups[0].rolloutApproved` (`dev` decision group) value to `true`.
1. The updated policy applies to all clusters in the `dev` group. These clusters have the `rolloutStatus` of
   `InProgress`. The root policy `status.rolloutStatus` value is `InProgress`.
1. All the clusters in the `dev` group become compliant. Their `rolloutStatus` is set to `Succeeded`.
1. The user sets the `spec.decisionGroups[1].rolloutApproved` (`stage` decision group) value to `true`.
1. The updated policy applies to all clusters in the `stage` group. These clusters have the `rolloutStatus` of
   `InProgress`.
1. All the clusters in the `stage` group become compliant. Their `rolloutStatus` is set to `Succeeded`.
1. The user sets the `spec.decisionGroups[2].rolloutApproved` (`prod` decision group) value to `true`.
1. The updated policy applies to all clusters in the `prod` group. These clusters have the `rolloutStatus` of
   `InProgress`.
1. All the clusters in the `prod` group become compliant. Their `rolloutStatus` is set to `Succeeded`. The root policy
   `status.rolloutStatus` value is `Succeeded`.

#### Updated Policy With manualPerGroup Out of Order

The below example is not realistic, but illustrates that if you have several production groups, you can choose whichever
you'd like to be deployed to.

1. The root policy is updated.
1. All clusters have the `rolloutStatus` of `ToApply` and keep the previous policy version. The root policy
   `status.rolloutStatus` value is `ToApply`.
1. The user sets the `spec.decisionGroups[1].rolloutApproved` (`stage` decision group) value to `true`.
1. The updated policy applies to all clusters in the `stage` group. These clusters have the `rolloutStatus` of
   `InProgress`. The root policy `status.rolloutStatus` value is `InProgress`.
1. All the clusters in the `stage` group become compliant. Their `rolloutStatus` is set to `Succeeded`.
1. The user sets the `spec.decisionGroups[2].rolloutApproved` (`prod` decision group) value to `true`.
1. The updated policy applies to all clusters in the `prod` group. These clusters have the `rolloutStatus` of
   `InProgress`.
1. All the clusters in the `prod` group become compliant. Their `rolloutStatus` is set to `Succeeded`.
1. The user sets the `spec.decisionGroups[0].rolloutApproved` (`stage` decision group) value to `true`.
1. The updated policy applies to all clusters in the `stage` group. These clusters have the `rolloutStatus` of
   `InProgress`.
1. All the clusters in the `stage` group become compliant. Their `rolloutStatus` is set to `Succeeded`. The root policy
   `status.rolloutStatus` value is `Succeeded`.

#### Common Situations

##### A New Cluster Is Added During a Rollout

When a new cluster is added, it receives the same policy as the others in the same group currently have. For example, if
there is a policy update and a cluster is added to the `stage` group while the `dev` group is still `InProgress`, the
new cluster would receive the last successful policy version, if any, until the rollout progressed to the `stage` group.

If the cluster is added to an earlier group than the group that is `InProgress`, the rollout switches back to that group
and waits for the cluster to rollout before resuming progress at the point previous to the new cluster being added.

##### A New Cluster Is Added After a Rollout

When a managed cluster is added to the placement after the rollout completed, the rollout does not restart. The cluster
directly receives the last successful policy definition and has a rollout status of `NewCluster`.

##### A Cluster In The Group Is Offline

If a cluster does not provide any compliance after the `progressDeadline` value has elapsed, the cluster is set to
`TimeOut` and the rollout fails. Assuming no additional rollout is started, when the cluster comes back online, it will
receieve the last successful policy version, if any.

##### Retrying a Failed Rollout

This is only necessary if `progressDeadline` is used since this is the only way a rollout can fail. Otherwise, the
rollout indefinitely waits for the clusters to become compliant.

In this situation, the first option is to update the policy spec in some way which would completly restart the rollout.

The second option is to use the `spec.retryRollout.rolloutUID` field on the `Rollout` object. When set to a UID that
matches `status.rolloutUID`, the Governance Policy Propagator will restart the rollout and create a new rollout UID.
This is used instead of a boolean that gets automatically flipped back to `false` to support a GitOps workflow.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Rollout
metadata:
  name: policy-<same as root policy>
  namespace: <same as root policy>
spec:
  retryRollout:
    rolloutUID: <UID from status.rolloutUID>
```

To support this, the `Rollout` will have UID in its status that gets generated on every new rollout.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Rollout
metadata:
  name: policy-<same as root policy>
  namespace: <same as root policy>
status:
  rolloutUID: 3ac24863-c829-4a0d-9ea8-050c9c4debf6
```

##### Ignoring Clusters During a Rollout

If there are clusters that are acting up and shouldn't impact the rollout from proceeding, the user can set a label
selector on the `spec.rolloutStrategy.ignoreClusterRolloutStatus` field on the `Policy` to match such clusters. They
will still receive the updated policy but their rollout status does not prevent the rollout from proceeding. If
`progressDeadline` is not set, then the rollout does not wait for a rollout status from this cluster.

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: sample-policy
  namespace: policy-namespace
spec:
  disabled: false
  remediationAction: enforce
  rolloutStrategy:
    ignoreClusterRolloutStatus:
      matchExpressions:
        - key: naughty-or-nice
          operator: In
          values:
            - naughty
  ...
```

##### Updating the Policy During a Rollout

The existing rollout halts at its current state. A new policy rollout starts with the new policy version.

##### Removing an Approval in manualPerGroup

Any clusters in the group that that had its approval revoked will not receive the new policy version if the rollout
hadn't started on them.

##### The `Rollout` Object is Deleted During a Rollout

The last successfully rolled out policy definition is no longer available, so the rollout will behave like a new policy
is being rolled out but had one or more failed previous rollouts. In other words, new clusters added to the placement
will not receive a policy if they aren't in a group being rolled out to yet.

##### Multiple Placements Are Bound to the Policy

With the rollout type of `ProgressivePerGroup`, the placements are sorted by name and then processed in order. For
example, if the first placement has groups `dev` and `stage` and the second placement has the group `prod`, then the
order is: `dev` -> `stage` -> ungrouped clusters from the first placement -> `prod` -> ungrouped clusters from the
second placement.

##### Automatically Approving Groups in manualPerGroup

Add the groups in the policy's `spec.rolloutStrategy.manualPerGroup.mandatoryDecisionGroups` array and they will be
treated as approved regardless of what is in the `Rollout` object.

### Implementation Details

#### Storing State

The main purpose of the `Rollout` object is to store rollout state. This is separate from the root policy to mitigate
size limitations on etcd objects. It's also generic for other Open Cluster Management components to adopt.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Rollout
metadata:
  name: policy-<same as root policy>
  namespace: <same as root policy>
status:
  decisionGroups:
    - groupName: dev
    - groupName: stage
    - groupName: prod
  lastSuccessful:
    apiVersion: v1
    kind: Policy
    ....
  currentRolloutGeneration: <generation of root policy>
  rolloutStatus: <same as root policy>
  rolloutUID: 3ac24863-c829-4a0d-9ea8-050c9c4debf6
```

The `status.decisionGroups` field is for convenience when using the `manualPerGroup` rollout strategy since those values
can be copied directly into the `spec` of the `Rollout`. This also helps the console integration.

The `status.lastSuccessful` value includes the policy definition after a successful rollout. This is used when there is
a new cluster during a rollout or after a failed rollout.

The `status.currentRolloutGeneration` value indicates the root policy generation of the rollout in progress. If this
generation does not match the root policy generation, then the Policy Propagator can know that a new rollout must be
started.

The `status.rolloutUID` is a UID that gets generated every time a new rollout is started.

#### Development Phase 1

- **Summary**

  - Add a `RolloutStatus` to the `Policy` status in the CRD.
  - Add the `Rollout` CRD and populate the `status.lastSuccessful` and `status.currentRolloutGeneration` field. This
    would ideally be included in the core of Open Cluster Management for any addon that supports rolloutStrategy to
    utilize.

- **Details**

  - The `RolloutStatus` would be added to reflect: rollout status on the replicated policy, per-cluster on the root
    policy, and an aggregated status on the root policy.

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

  - Add support for the `progressive` and `progressivePerGroup` rollout types.
  - This includes the custom `ignoreClusterRolloutStatus`.

- **Details**

  - Update the `Policy` CRD to contain the `RolloutStrategy` struct. (See the
    [`v1alpha1/RolloutStrategy`](https://github.com/open-cluster-management-io/api/blob/main/cluster/v1alpha1/types_rolloutstrategy.go))
    Defaults to `All` if a strategy is not provided.
  - Add a `status.observedGeneration` field to all policy types to know when an updated policy has been evaluated.
  - A succesful rollout for a cluster is the replicated policy being compliant and the `status.observedGeneration` field
    of all policies in the `policy-templates` array match their generation on the cluster. The Status Sync controller
    will handle this.

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

#### Development Phase 4

- **Summary**

  - Add support for the `manualPerGroup` rollout type.

- **Details**

  - Add the `manualPerGroup` rollout strategy as an option in the `spec.rolloutStrategy.type` field. It accepts all the
    same arguments as `progressivePerGroup`.
  - Add the `spec` portion to `Rollout` CRD.

### Notes/Constraints

The `PlacementDecisionGetter` returns an array of pointers (`[]*clusterv1beta1.PlacementDecision`) because it was
intended to retrieve from a cache, so the implementation could consider setting up a `PlacementDecision` cache instead
of using the Kubernetes client directly.

### Risks and Mitigation

- The `Placement` library does not support the `manualPerGroup` rollout strategy.

## Design Details

### Open Questions

1. Should the `spec.rolloutStrategy.ignoreClusterRolloutStatus` field be contributed to the rollout strategy API?

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
