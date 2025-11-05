# ManifestWorkReplicaSet automatic rollback

https://github.com/kubernetes/design-proposals-archive/blob/acc25e14ca83dfda4f66d8cb1f1b491f26e78ffe/apps/controller_history.md
https://github.com/kubernetes/design-proposals-archive/blob/acc25e14ca83dfda4f66d8cb1f1b491f26e78ffe/apps/statefulset-update.md

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

To implement safe rollbacks for multi-cluster deployments, we propose utilizing Kubernetes ControllerRevision resources to maintain a historical record of MWRS changes. We will also introduce new status fields for tracking these revisions:

## Motivation

The ManifestWorkReplicaSet (MWRS) simplifies multi-cluster workload distribution by supporting several rollout strategies that enable gradual deployment of workloads across clusters.

Currently, when the progressive rollout fails, Work controller just stops rollout. In some critical application, we want to roll back quickly to the previous revision on failure which will minimize the impact.

### Goals

- Define manifestwork template revision management
- Define rollback state machine in ManifestWorkReplicaSet
- Define rollback status in ManifestWorkReplicaSet

### Non-Goals

TBD

## Proposal

### User Stories

#### Story 1 — automated rollback on failure

As a user, I want MWRS to automatically roll back workloads in the event of a failed rollout. When a deployment fails, MWRS should identify the previous revision and restore it consistently to the already rolled out clusters. The rollback should be able to perform any additional operations needed to safely revert to the stable version - for example, reverting manifests or skipping rollout steps in dependent systems in Argo Rollout use-case, which requires the mutation of old manifest resources.

### Risks and Mitigation

1. CRD Rollback - if manifestwork template includes CRD version update, rolling back the CRD resource cause the significant impact on the system and application. Using automatic rollback feature should be opt-in and use in caution.

## Design Details

### API Object

* spec.revisionHistoryLimit: This field will control the retention of the revision history.
* spec.placementRefs[*].rolloutStrategy.progressive.abortOnFailure:
  - enabled: auto rollback on failure

```yaml
apiVersion: work.open-cluster-management.io/v1alpha1
kind: ManifestWorkReplicaSet
metadata:
  name: hello-rollout
  namespace: default
spec:
  revisionHistoryLimit: 3
  placementRefs:
    - name: placement-rollout-progressive
      rolloutStrategy:
        type: Progressive
        progressive:
          ...
        abortOnFailure:
          enabled: true
  ...
```

[Spec](https://github.com/open-cluster-management-io/api/blob/main/work/v1alpha1/types_manifestworkreplicaset.go#L57)

```golang
// ManifestWorkReplicaSetSpec defines the desired state of ManifestWorkReplicaSet
type ManifestWorkReplicaSetSpec struct {
  // RevisionHistoryLimit is the maximum number of revisions that will 
  // be maintained in the manifestWorkTemplate's revision history. The revision history
  // consists of all revisions not represented by a currently applied 
  // manifestWorkTemplate version. The default value is 2.
  RevisionHistoryLimit *int32 `json:revisionHistoryLimit,omitempty`

  // ...
}
```

[rolloutstrategy](https://github.com/open-cluster-management-io/api/blob/main/cluster/v1alpha1/types_rolloutstrategy.go#L126)

```golang
// RollbackConfig is a rollback configuration on failure.
type RollbackConfig struct {
  // +optional
  Enabled bool `json:",inline"`

  // +optional
  SkipValidation bool `json:",inline"`
}

// localPlacementReference is the name of a Placement resource in current namespace
type LocalPlacementReference struct {
  // Name of the Placement resource in the current namespace
  // +required
  // +kubebuilder:validation:Required
  // +kubebuilder:validation:MinLength=1
  Name string `json:"name"`

  // +optional
  // +kubebuilder:default={type: All, all: {progressDeadline: None}}
  RolloutStrategy cluster.RolloutStrategy `json:"rolloutStrategy"`

  // RollbackConfig is the configuration to define rollback behavior on failure
  // +optional
  RollbackConfig *rollback `json:"rollback,omitempty"`
}
```

```golang
// ManifestWorkStatus represents the current status of managed cluster ManifestWork.
type ManifestWorkStatus struct {
  // collisionCount is the count of hash collisions for the Manifestwork. The StatefulSet controller
	// uses this field as a collision avoidance mechanism when it needs to create the name for the
	// newest ControllerRevision.
	// +optional
	CollisionCount *int32 `json:"collisionCount,omitempty"`

  // CurrentRevision, if not empty, indicates the version of ManifestworkTemplate used to generate Manifestworks.
  CurrentRevision string `json:"currentRevision,omitempty"`
  
  // UpdateRevision, if not empty, indicates the version of PodSpecTemplate, 
  // VolumeClaimsTemplate tuple used to generate Pods in the sequence
  // [Replicas-UpdatedReplicas,Replicas)
  UpdateRevision string `json:"updateRevision,omitempty"`
  
	// Conditions contains the different condition statuses for this work.
	// Valid condition types are:
	// 1. Applied represents workload in ManifestWork is applied successfully on managed cluster.
	// 2. Progressing represents workload in ManifestWork is being applied on managed cluster.
	// 3. Available represents workload in ManifestWork exists on the managed cluster.
	// 4. Degraded represents the current state of workload does not match the desired
	// state for a certain period.
	Conditions []metav1.Condition `json:"conditions,omitempty"`

	// ResourceStatus represents the status of each resource in manifestwork deployed on a
	// managed cluster. The Klusterlet agent on managed cluster syncs the condition from the managed cluster to the hub.
	// +optional
	ResourceStatus ManifestResourceStatus `json:"resourceStatus,omitempty"`
}
```


```yaml
status:
  collisionCount: 0
  currentRevision: YYYYYYY
  updateRevision: XXXXXXX
```


```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: <MWRS_Name>
  labels:
    # controller-hash is to distinguish between old and new
    # Manifestworks during Manifestwork template updates.
    work.open-cluster-management.io/controller-hash: <hash>
    work.open-cluster-management.io/manifestworkreplicaset: default.mwrset-1
    work.open-cluster-management.io/placementname: placement1
spec: 
  placementRefs:
    # ...
  manifestWorkTemplate:
    manifestConfigs:
    - feedbackRules:
        - jsonPaths:
          - name: isHealthy
            path: '.status.conditions[?(@.type=="Healthy")].status'
          - name: updatedReplicas
            path: '.status.updatedReplicas'
          - name: replicas
            path: '.status.replicas'
          - name: replicas
            path: '.status.updatePodHash'
          - name: replicas
            path: '.status.currentPodHash'
          type: JSONPaths
      resourceIdentifier:
        group: argoproj.io
        name: hello-rollout
        namespace: sam
        resource: rollouts
    workload:
      manifests:
        - kind: Rollout
          apiVersion: argoproj.io/v1alpha1
          metadata:
            name: hello-rollout
            namespace: default
```

### ManifestWork Revision History

[ControllerRevision]() is a Kubernetes API resource used by controllers, such as the StatefulSet and DaemonSet controller, to track historical configuration changes. Work Controller will use the ControllerRevision to snapshot and version its target ManifestWork Object state.

#### Snapshot Creation

In order to snapshot a version of its target ManifestWork state, it will serialize and store the .Spec.ManifestWorkTemplate along with the .Generation in each snapshot. Each snapshot will be labeled with the ManifestWorkReplicaset name.

#### History Reconstruction

As proposed in Controller History, in order to reconstruct the revision history of a ManifestWorkReplicaSet, the Work controller will select all snapshots based on its ManifestWorkReplicaSet name and sort them by the contained .Generation. This will produce an ordered set of revisions to the ManifestWorkReplicaSet's target Object state.

#### History Maintenance

In order to prevent the revision history of the ManifestWorkReplicaSet from exceeding memory or storage limits, the Work controller will periodically prune its revision history so that no more that .Spec.RevisionHistoryLimit non-live versions of target Object state are preserved.

### Revised Work Controller

1. Reconstruct ManifestWorkReplicaSet history:
    1. List existing ControllerRevision controlled by this ManifestWorkReplicaSet.
    1. Find the history of the current ManifestWorkReplicaSet's ManifestWorkTemplate target state, and create one if not found:
    1. The .name of the history will be unique, generated from .spec.ManifestWorkTemplate hash with hash collision resolution (`.status.collisionCount`) 
    1. If history creation failed:
        1. If it is because of name collision:
            1. If the collided history is same as ManifestWorkReplicaSet's ManifestWorkTemplate target state, there is the already created the history
            1. Otherwise, bump ManifestWorkReplicaSet .status.collisionCount by 1, 
        1. Else, exit and reconcile again
    1. New ControllerRevision will be labeled with `work.open-cluster-management.io/controller-hash`
    1. Work controller will add a ControllerRef in the history `.ownerReferences`.
    1. The current history should be the largest `.revision` number amongst the others. Update `.revision` to the largest if it is not. (for example, after rollback, we will need to update it to the latest.)



### Status

`Progressing` condition will be added to track the rollout update status.

| Condition Type | Reasons | Description |
| --- | --- | --- |
| PlacementVerified | AsExpected, PlacementDecisionNotFound, PlacementDecisionEmpty or NotAsExpected | indicates if Placement is valid |
| ManifestworkApplied | AsExpected, NotAsExpected or Processing | a ManifestWork has been created in each cluster defined by PlacementDecision |
| PlacementRollOut | Progressing or Complete | indicates if RollOut Strategy is complete. | 
| Progressing | NewRevisionCreated, FoundNewRevision, NewManifestAvaialable, RolloutAborted, ProgressDeadlineExceeded | the rollout is progressing. Progress for a rollout is considered when a new manifest is created or adopted, when new manifestwork rolls out |

The following status shows the example of conditions:

```yaml
apiVersion: work.open-cluster-management.io/v1alpha1
kind: ManifestWorkReplicaSet
metadata:
  name: mwrset-1
  namespace: default
spec: ...
status:
  collisionCount: 0
  currentRevision: YYYYYYY
  updateRevision: XXXXXXX
  abort: true | false
  abortedTime: "2025-11-04T15:41:05Z"

  conditions:
  - lastTransitionTime: "2025-11-04T15:41:05Z"
    message: ""
    reason: AsExpected
    status: "True"
    type: PlacementVerified
  - lastTransitionTime: "2025-11-05T09:02:12Z"
    message: ""
    reason: Complete
    status: "True"
    type: PlacementRolledOut
  - lastTransitionTime: "2025-11-05T09:02:12Z"
    message: ""
    reason: AsExpected
    status: "True"
    type: ManifestworkApplied

  - lastTransitionTime: "2025-11-05T09:02:12Z"
    message: "New revision mwrs-XXXXXXX is successfully progressed"
    reason: NewManifestWorkAvailable
    status: "True"
    type: Progressing
  placementSummary:
  - availableDecisionGroups: 1 (2 / 2 clusters applied)
    name: placement1
    summary:
      Updated: 2 # New update revision count
      Applied: 2
      available: 2
      degraded: 0
      progressing: 0
      total: 2 # This represents the update revision count. shouldn't we repurpose total to count all existing manifestwork?
  summary:
    Updated: 2 # New update revision count
    Applied: 2
    available: 2
    total: 2 # This represents the update revision count. shouldn't we repurpose total to count all existing manifestwork?
```

### New manifestwork update transition

| PlacementVerified | ManifestworkApplied | Progressing | PlacementRollOut |
| --- | --- | --- | --- |
| AsExpected (True) | | NewRevisionCreated (True) | |
| AsExpected (True)| Progressing (False) | NewRevisionCreated (True) | Progressing (False) | 
| AsExpected (True)| Progressing (False) | NewRevisionCreated (True) | Progressing (False) | 
| AsExpected (True)| AsExpected (True) | NewRevisionAvailable (True) | Complete (True) |


### Found the existing revision transition (rollback)

| PlacementVerified | ManifestworkApplied | Progressing | PlacementRollOut |
| --- | --- | --- | --- |
| AsExpected (True) | | FoundNewRevision (True) | |
| AsExpected (True)| Progressing (False) | FoundNewRevision (True) | Progressing (False) |
| AsExpected (True)| Progressing (False) | FoundNewRevision (True) | Progressing (False) |
| AsExpected (True)| AsExpected (True) | NewRevisionAvailable (True) | Progressing (False) |

### ProgressDeadlineExceeded transition

| PlacementVerified | ManifestworkApplied | Progressing | PlacementRollOut |
| --- | --- | --- | --- |
| AsExpected (True) | | NewRevisionCreated (True) | |
| AsExpected (True)| Progressing (False) | NewRevisionCreated (True) | Progressing (False) | 
| AsExpected (True)| Progressing (False) | NewRevisionCreated (True) | Progressing (False) | 
| AsExpected (True)| NotAsExpected (False) | ProgressDeadlineExceeded (True) | Progressing (False) |

### Aborting transition

| PlacementVerified | ManifestworkApplied | Progressing | PlacementRollOut |
| --- | --- | --- | --- |
| AsExpected (True) | | NewRevisionCreated (True) | |
| AsExpected (True)| Progressing (False) | NewRevisionCreated (True) | Progressing (False) | 
| AsExpected (True)| Progressing (False) | NewRevisionCreated (True) | Progressing (False) | 
| AsExpected (True)| NotAsExpected (False) | ProgressDeadlineExceeded (True) | Progressing (False) |
| AsExpected (False)| NotAsExpected (False) | RolloutAborted (False) | Progressing (False) |


### Aborting vs Rollback

Abort is used to cancel the current rollout. When aborting the current rollout, spec is not changed, but `status.abort` is set to `true`, which will set `status.abortedTime`. then loading the older revision and then apply to all manifestworks. 

On the other hand, rollback is the explicit action. If user wants to roll back to the older revision, user can apply the older manifestTemplate to update the Spec. work controller will look up the history and roll it out as normal. 

### Open Questions [optional]

> TBD

### Test Plan

- Unit-test
- Integration test

### Graduation Criteria

TBD

### Upgrade / Downgrade Strategy


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

