# Rollback ManifestWorkReplicaSet

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

- Define manifestwork template revision management.
- Enable rollback feature with the revision.
- Enable automatic aborting rollout on the failure of rollout.

### Non-Goals

TBD

## Proposal

### User Stories

#### Story 1 — automatic rollback on failure

As a user, I want MWRS to automatically roll back workloads in the event of a failed rollout. When a deployment fails, MWRS should identify the previous revision and restore it consistently to the already rolled out clusters. The rollback should be able to perform any additional operations needed to safely revert to the stable version - for example, reverting manifests or skipping rollout steps in dependent systems in Argo Rollout use-case, which requires the mutation of old manifest resources.

### Risks and Mitigation

1. CRD Rollback - if manifestwork template includes CRD version update, rolling back the CRD resource cause the significant impact on the system and application. Using automatic rollback feature should be opt-in and use in caution.

## Design Details

The `abort` is the cancel operation to cancel the progressing rollout and roll it back to the the stable revision automatically without updating `.spec.ManifestWorkTemplate` whereas `rollback` requires the explicit manual action by updating `.spec.ManifestWorkTemplate`.

### ManifestWorkReplicaSet API Object

To support abort and rollback feature, we will have the following API changes:

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: <MWRS_Name>
spec: 
  # the number of revision history limit
  revisionHistoryLimit: 3
  placementRefs:
    - name: placement-rollout-progressive
      rolloutStrategy:
        type: Progressive
        progressive:
          ...
        # (NEW) abort starts if abortOnFailure is true. otherwise, it ends on the failure.
        abortOnFailure: true
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
status:
  # (NEW) hash collision count
  collisionCount: 0
  # (NEW) current revision.
  currentRevision: YYYYYYY
  # (NEW) desired update revision.
  updateRevision: XXXXXXX
  # (NEW) abort is true if abort starts
  abort: true | false
  # (NEW) abortedTime represents the time when abort starts.
  # When new revision rollout, abort needs to be set to false and abortedTime should be nil.
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

  # (NEW) Progressing is new Condition type to present the detail rollout progress.
  - lastTransitionTime: "2025-11-05T09:02:12Z"
    message: "New revision mwrs-XXXXXXX is successfully progressed"
    reason: NewManifestWorkAvailable
    status: "True"
    type: Progressing

  placementSummary:
  - availableDecisionGroups: 1 (2 / 2 clusters applied)
    name: placement1
    summary:
      # (NEW) update revision count
      updated: 2

      Applied: 2
      available: 2
      degraded: 0
      progressing: 0
      total: 2 # Can we use this for desired total cluster?
  summary:
    # (NEW) update revision count
    updated: 2

    Applied: 2
    available: 2
    total: 2 # Can we use this for desired total cluster?
```

#### API Status

This enhancement introduces `RolloutDegraded` reason to `PlacementRollout` condition type to make the current rollout status clear. We also added `Progressing` condition type for updating the current rollout progress.

| Condition Type | Reasons | Description |
| --- | --- | --- |
| PlacementVerified | AsExpected, PlacementDecisionNotFound, PlacementDecisionEmpty or NotAsExpected | indicates if Placement is valid |
| ManifestworkApplied | AsExpected, NotAsExpected or Processing | a ManifestWork has been created in each cluster defined by PlacementDecision |
| PlacementRollOut | Progressing or Complete or (NEW) RolloutDegraded | indicates if RollOut Strategy is complete. | 
| (NEW) Progressing | NewRevisionCreated, FoundNewRevision, NewManifestAvaialable, RolloutAborted, ProgressDeadlineExceeded | the rollout is progressing. Progress for a rollout is considered when a new manifest is created or adopted, when new manifestwork rolls out |


### ManifestWork API Object

To track the revision of manifestwork, we will introduce `work.open-cluster-management.io/controller-hash` label. the hash can be used for identify the ControllerRevision history and filter `Manifestwork` resources.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: <MWRS_Name>
  namespace: <cluster name>
  labels:
    # (NEW) controller-hash is to distinguish between old and new
    # Manifestworks during Manifestwork template updates.
    work.open-cluster-management.io/controller-hash: <hash>

    # existing labels.
    work.open-cluster-management.io/manifestworkreplicaset: default.mwrset-1
    work.open-cluster-management.io/placementname: placement1
spec: 
  ...
```

### `Abort` vs `Rollback`

Abort is used to `cancel` the current rollout. When aborting the current rollout, spec is not changed, but `status.abort` is set to `true`, which will set `status.abortedTime`. then loading the older revision and then apply `.spec.manifestWorkTemplate` of the old revision to all clusters at once.

On the other hand, rollback is the explicit action. If user wants to roll back to the older revision, work controller can find the revision from the list of ControllerRevision resources and apply the older manifestTemplate to update the `.spec.manifestWorkTemplate`.

### ManifestWork Revision History

[ControllerRevision](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/controller-revision-v1/) is a Kubernetes API resource used by controllers, such as the StatefulSet and DaemonSet controller, to track historical configuration changes. Work Controller will use the ControllerRevision to snapshot and version its desired ManifestWork Object state.

#### Snapshot Creation

In order to snapshot a version of its desired ManifestWork state, it will serialize and store the `.Spec.ManifestWorkTemplate` along with the `.Generation` in each snapshot. Each snapshot will be labeled with the ManifestWorkReplicaset name.

#### History Reconstruction

As proposed in Controller History, in order to reconstruct the revision history of a `ManifestWorkReplicaSet`, the Work controller will select all snapshots based on its `ManifestWorkReplicaSet` name and sort them by the contained `.Generation`. This will produce an ordered set of revisions to the `ManifestWorkReplicaSet`'s desired Object state.

#### History Maintenance

In order to prevent the revision history of the `ManifestWorkReplicaSet` from exceeding storage limits, the Work controller will periodically prune its revision history so that no more that `.Spec.RevisionHistoryLimit` non-live versions of desired Object state are preserved.

### Enhanced Rollout with revision history

1. Reconstruct ManifestWorkReplicaSet history:
    1. List existing `ControllerRevision`s controlled by this `ManifestWorkReplicaSet`.
    1. Find the history of the current `ManifestWorkReplicaSet`'s `.spec.ManifestWorkTemplate` desired state, and create one if not found:
    1. The `.name` of the history will be unique, generated from `.spec.ManifestWorkTemplate` hash with hash collision resolution (`.status.collisionCount`) 
    1. If history creation failed:
        1. If it is because of name collision:
            1. If the collided history is same as `ManifestWorkReplicaSet`'s `.spec.ManifestWorkTemplate` desired state, there is the already created the history
            1. Otherwise, bump ManifestWorkReplicaSet `.status.collisionCount` by 1, 
        1. Else, exit and reconcile again
    1. New ControllerRevision will be labeled with `work.open-cluster-management.io/controller-hash`
    1. Work controller will add a ControllerRef in the history `.ownerReferences`.
    1. The current history should be the largest `.revision` number amongst the others. Update `.revision` to the largest if it is not. (for example, after rollback, we will need to update it to the latest.)
2. Update `.spec.updateRevision` to the newly calculated revision.
3. Iterate each placement references (`.spec.placementRefs`)
    1. Get all manifestworks associated with the current placement
    1. Get the progressing manifestwork hash
        1. Use `.spec.updateRevision` if `.status.abort` is true, otherwise, use `.spec.currentRevision`.
    1. Find manifestworks which has the same hash of the progressing manifestwork hash.
        1. Determine cluster rollout status (Failed, Success)
    1. Create Rollout handler
    1. Calculate RolloutResult (rollout/timeout/removed candidate clusters)
    1. If rolloutResult includes timeout clusters and `.spec.placementRefs[*].rolloutStrategy.abortOnFailure` is true
        1. Set `.status.abort` to true and set `.status.abortedTime` to the current time.
        1. Fetch `ControllerRevision` corresponding to `.status.currentRevision` and extract `.spec.manifestWorkTemplate`
        1. Apply `.spec.manifestWorkTemplate` to all clusters in `rolloutResult.clusterToRollout`
    1. Iterate each cluster of `rolloutResult.clusterToRollout`
        1. Create ManifestWork and add `work.open-cluster-management.io/controller-hash` label with new hash
        2. Apply ManifestWork to each cluster namespace
    1. Clean up manifestwork resources from each cluster namespace
    1. Status update
        1. Calculate `.summary.updated` by counting the number of `Succeeded` status of rollout cluster
        1. if `.summary.updated` == `the desired total number of clusters`,
            1. Set `.status.updateRevision` to `.status.currentRevision`

### Status transition

This section describes the detail status transition for each rollout condition.

#### New manifestwork update transition

| PlacementVerified | ManifestworkApplied | Progressing | PlacementRollOut |
| --- | --- | --- | --- |
| AsExpected (True) | | NewRevisionCreated (True) | |
| AsExpected (True)| Progressing (False) | NewRevisionCreated (True) | Progressing (False) | 
| AsExpected (True)| Progressing (False) | NewRevisionCreated (True) | Progressing (False) | 
| AsExpected (True)| AsExpected (True) | NewRevisionAvailable (True) | Complete (True) |

#### Rollback (found the existing revision transition)

| PlacementVerified | ManifestworkApplied | Progressing | PlacementRollOut |
| --- | --- | --- | --- |
| AsExpected (True) | | FoundNewRevision (True) | |
| AsExpected (True)| Progressing (False) | FoundNewRevision (True) | Progressing (False) |
| AsExpected (True)| Progressing (False) | FoundNewRevision (True) | Progressing (False) |
| AsExpected (True)| AsExpected (True) | NewRevisionAvailable (True) | Progressing (False) |
| AsExpected (True)| AsExpected (True) | NewRevisionAvailable (True) | Complete (True) |

#### ProgressDeadlineExceeded transition

| PlacementVerified | ManifestworkApplied | Progressing | PlacementRollOut |
| --- | --- | --- | --- |
| AsExpected (True) | | NewRevisionCreated (True) | |
| AsExpected (True)| Progressing (False) | NewRevisionCreated (True) | Progressing (False) | 
| AsExpected (True)| Progressing (False) | NewRevisionCreated (True) | Progressing (False) | 
| AsExpected (True)| NotAsExpected (False) | ProgressDeadlineExceeded (False) | RolloutDegraded (False) |

#### Abort transition

| PlacementVerified | ManifestworkApplied | Progressing | PlacementRollOut |
| --- | --- | --- | --- |
| AsExpected (True) | | NewRevisionCreated (True) | |
| AsExpected (True)| Progressing (False) | NewRevisionCreated (True) | Progressing (False) | 
| AsExpected (True)| Progressing (False) | NewRevisionCreated (True) | Progressing (False) | 
| AsExpected (True)| NotAsExpected (False) | ProgressDeadlineExceeded (False) | RolloutDegraded (False) |
| AsExpected (False)| NotAsExpected (False) | RolloutAborted (False) | RolloutDegraded (False) |

### Open Questions

1. Should we have the delay before starting aborting a rollout ?

### Test Plan

- Unit-test
- Integration test

### Graduation Criteria

TBD

### Upgrade / Downgrade Strategy


### Version Skew Strategy

TBD

## Implementation History

TBD