# Rollback ManifestWorkReplicaSet

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This enhancement proposes adding rollback capabilities to ManifestWorkReplicaSet (MWRS) to enable safe recovery from failed multi-cluster deployments. The solution leverages Kubernetes ControllerRevision resources to maintain a historical record of MWRS template changes, similar to how StatefulSet and DaemonSet controllers track revision history. 

The enhancement introduces two key rollback mechanisms: automatic rollback on failure (via `abortOnFailure` flag) and manual rollback (via spec updates). When a progressive rollout fails, the controller can automatically revert to the previous stable revision across all affected clusters, minimizing downtime and impact. For manual rollback scenarios, users can update the MWRS spec to reference a previous revision, and the controller will restore that revision.

New API fields include `revisionHistoryLimit` to control history retention, `abortOnFailure` to enable automatic rollback, and status fields (`currentRevision`, `updateRevision`, `abort`, `abortedTime`, `collisionCount`) to track rollout state. The enhancement also introduces a new `Progressing` condition type and `RolloutDegraded` reason to provide better visibility into rollout and rollback operations.

## Motivation

The ManifestWorkReplicaSet (MWRS) simplifies multi-cluster workload distribution by supporting several rollout strategies that enable gradual deployment of workloads across clusters. This allows operators to safely deploy applications to multiple managed clusters with controlled rollout patterns.

However, the current implementation has limitation: when a progressive rollout fails, the Work controller simply stops the rollout, leaving clusters in an inconsistent state. Some clusters may have the new version deployed while others remain on the old version, and the failed clusters are stuck in a failed state. This creates several problems:

1. **Inconsistent Cluster State**: Partial rollouts leave the system in an undefined state where different clusters run different versions
2. **Extended Downtime**: Critical applications may experience extended downtime while waiting for manual intervention

For critical applications, especially those requiring high availability, the ability to automatically roll back to a previous stable revision on failure is essential to minimize impact and maintain service continuity. This enhancement addresses these limitations by providing both automatic and manual rollback capabilities with proper revision history management.

### Goals

- Define manifestwork template revision management.
- Enable rollback feature with the revision.
- Enable automatic rollout on the failure of rollout.

### Non-Goals

- Manual rollback via kubectl plugin commands (rollback will be performed by updating `.spec.manifestWorkTemplate`)
- Rollback of CRD versions (automatic rollback should be used with caution when CRDs are involved)
- Rollback across different API versions of ManifestWorkReplicaSet

## Proposal

### User Stories

#### Story 1 — Automatic rollback on failure

As a user, I want MWRS to automatically roll back workloads in the event of a failed rollout. When a deployment fails, MWRS should identify the previous revision and restore it consistently to the already rolled out clusters. The rollback should be able to perform any additional operations needed to safely revert to the stable version.

#### Story 2 — Manual rollback capability

As a user, I want to manually roll back to a previous stable revision when needed. I should be able to update the ManifestWorkReplicaSet spec to reference a previous revision, and the controller should restore that revision across all clusters.

### Implementation Details/Notes/Constraints

The rollback mechanism leverages Kubernetes ControllerRevision resources to maintain a historical record of ManifestWorkReplicaSet changes. This approach is similar to how StatefulSet and DaemonSet controllers track revision history.

Key implementation constraints:
- Revision history is limited by `revisionHistoryLimit` to prevent unbounded storage growth
- Automatic rollback (`abortOnFailure`) is opt-in and should be used with caution when CRDs are involved
- Rollback operations require the previous revision to be available in ControllerRevision history
- The controller uses hash-based collision detection to ensure unique revision naming

### Risks and Mitigation

1. **CRD Rollback Risk** - If the manifestwork template includes a CRD version update, rolling back the CRD resource can cause significant impact on the system and application. 
   - **Mitigation**: Using the automatic rollback feature (`abortOnFailure`) should be opt-in and used with caution. Users should be aware of the implications when rolling back CRD changes.

2. **Storage Growth Risk** - Maintaining revision history could lead to unbounded storage growth.
   - **Mitigation**: The `revisionHistoryLimit` field allows users to control the maximum number of revisions retained. The controller will automatically prune old revisions.

3. **Rollback Failure Risk** - If a rollback operation itself fails, the system could be left in an inconsistent state.
   - **Mitigation**: The controller tracks rollback status through conditions and status fields, allowing for monitoring and manual intervention if needed.

## Design Details

The `abort` operation cancels the progressing rollout and rolls it back to the stable revision automatically without updating `.spec.manifestWorkTemplate`, whereas `rollback` requires explicit manual action by updating `.spec.manifestWorkTemplate`.

### ManifestWorkReplicaSet API Object

To support abort and rollback feature, we will have the following API changes:

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWorkReplicaSet
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
        # (NEW) abortOnFailure: if true, automatically aborts and rolls back on failure; otherwise, rollout ends on failure
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
  # (NEW) collisionCount: hash collision count for revision naming
  collisionCount: 0
  # (NEW) currentRevision: the current revision hash
  currentRevision: YYYYYYY
  # (NEW) updateRevision: the desired update revision hash
  updateRevision: XXXXXXX
  # (NEW) abort: true if abort operation has started
  abort: true | false
  # (NEW) abortedTime: represents the time when abort starts
  # When a new revision rollout begins, abort should be set to false and abortedTime should be nil
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

  # (NEW) Progressing: new Condition type to present the detailed rollout progress
  - lastTransitionTime: "2025-11-05T09:02:12Z"
    message: "New revision mwrs-XXXXXXX is successfully progressed"
    reason: NewManifestWorkAvailable
    status: "True"
    type: Progressing

  placementSummary:
  - availableDecisionGroups: 1 (2 / 2 clusters applied)
    name: placement1
    summary:
      # (NEW) updated: count of clusters with the update revision applied
      updated: 2
      Applied: 2
      available: 2
      degraded: 0
      progressing: 0
      total: 2
  summary:
    # (NEW) updated: count of clusters with the update revision applied
    updated: 2
    Applied: 2
    available: 2
    total: 2
```

#### API Status

This enhancement introduces `RolloutDegraded` reason to `PlacementRollout` condition type to make the current rollout status clear. This enhancement also added `Progressing` condition type for updating the current rollout progress.

| Condition Type | Reasons | Description |
| --- | --- | --- |
| PlacementVerified | AsExpected, PlacementDecisionNotFound, PlacementDecisionEmpty, NotAsExpected | Indicates if Placement is valid |
| ManifestworkApplied | AsExpected, NotAsExpected, Processing | A ManifestWork has been created in each cluster defined by PlacementDecision |
| PlacementRollOut | Progressing, Complete, `(NEW)` RolloutDegraded | Indicates if RollOut Strategy is complete |
| `(NEW)` Progressing | NewRevisionCreated, FoundNewRevision, NewManifestWorkAvailable, RolloutAborted, ProgressDeadlineExceeded | The rollout is progressing. Progress for a rollout is considered when a new manifest is created or adopted, when new manifestwork rolls out |


### ManifestWork API Object

To track the revision of manifestwork, this enhancement will introduce the `work.open-cluster-management.io/controller-hash` label. The hash can be used to identify the ControllerRevision history and filter `ManifestWork` resources.

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

`Abort` is used to **cancel** the current rollout. When aborting the current rollout, spec is not changed, but `status.abort` is set to `true`, which will set `status.abortedTime`. then loading the older revision and then apply `.spec.manifestWorkTemplate` of the old revision to all clusters at once.

On the other hand, `rollback` is the explicit action. If user wants to roll back to the older revision, the user (or cli) can find the revision from the list of ControllerRevision resources and apply the older manifestTemplate to update the `.spec.manifestWorkTemplate`.

### ManifestWork Revision History

[ControllerRevision](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/controller-revision-v1/) is a Kubernetes API resource used by controllers, such as the StatefulSet and DaemonSet controller, to track historical configuration changes. Work Controller will use the ControllerRevision to snapshot and version its desired ManifestWork Object state.

#### Snapshot Creation

In order to snapshot a version of its desired ManifestWork state, it will serialize and store the `.Spec.ManifestWorkTemplate` along with the `.Generation` in each snapshot. Each snapshot will be labeled with the ManifestWorkReplicaset name.

#### History Reconstruction

As proposed in Controller History, in order to reconstruct the revision history of a `ManifestWorkReplicaSet`, the Work controller will select all snapshots based on its `ManifestWorkReplicaSet` name and sort them by the contained `.Generation`. This will produce an ordered set of revisions to the `ManifestWorkReplicaSet`'s desired Object state.

#### History Maintenance

In order to prevent the revision history of the `ManifestWorkReplicaSet` from exceeding storage limits, the Work controller will periodically prune its revision history so that no more that `.Spec.RevisionHistoryLimit` non-live versions of desired Object state are preserved.

### Enhanced Rollout with revision history

1. `(NEW)` Reconstruct ManifestWorkReplicaSet history:
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
2. `(NEW)` Update `.spec.updateRevision` to the newly calculated revision.
3. Iterate each placement references (`.spec.placementRefs`)
    1. Get all manifestworks associated with the current placement
    1. `(NEW)` Find the progressing manifestwork hash
        1. Use `.spec.updateRevision` if `.status.abort` is true, otherwise, use `.spec.currentRevision`.
    1. Find manifestworks which has the same hash of the progressing manifestwork hash.
        1. [Determine cluster rollout status (Failed, Success)](https://github.com/open-cluster-management-io/ocm/blob/main/pkg/work/hub/controllers/manifestworkreplicasetcontroller/manifestworkreplicaset_deploy_reconcile.go#L96)
    1. Create Rollout handler
    1. Calculate RolloutResult (rollout/timeout/removed candidate clusters)
    1. `(NEW)` If rolloutResult includes timeout clusters and `.spec.placementRefs[*].rolloutStrategy.abortOnFailure` is true
        1. Set `.status.abort` to true and set `.status.abortedTime` to the current time.
        1. Fetch `ControllerRevision` corresponding to `.status.currentRevision` and extract `.spec.manifestWorkTemplate`
        1. Apply `.spec.manifestWorkTemplate` to all clusters in `rolloutResult.clusterToRollout`
    1. Iterate each cluster of `rolloutResult.clusterToRollout`
        1. `(NEW)` Create ManifestWork and add `work.open-cluster-management.io/controller-hash` label with new hash
        2. Apply ManifestWork to each cluster namespace
    1. Clean up manifestwork resources from each cluster namespace
    1. `(NEW)` Status update
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

The test plan will include unit tests, integration tests, and end-to-end tests to ensure the rollback functionality works correctly across different scenarios.

#### Unit Tests

- ControllerRevision creation and management
  - Test revision history creation when ManifestWorkTemplate changes
  - Test revision history pruning based on `revisionHistoryLimit`
  - Test hash collision detection and resolution
  - Test revision number assignment and updates

- Rollout logic with revisions
  - Test automatic abort on failure when `abortOnFailure` is enabled
  - Test manual rollback by updating spec
  - Test status field updates (currentRevision, updateRevision, abort, abortedTime)
  - Test condition transitions (Progressing, PlacementRolledOut with RolloutDegraded)

- ManifestWork labeling
  - Test `controller-hash` label assignment
  - Test ManifestWork filtering by revision hash

#### Integration Tests

- End-to-end rollback scenarios
  - Test automatic rollback when progressive rollout fails
  - Test manual rollback to previous revision
  - Test rollback with multiple placement references
  - Test rollback across different rollout strategies (Progressive, All)

- Revision history management
  - Test ControllerRevision lifecycle (create, update, delete)
  - Test revision history reconstruction after controller restart
  - Test revision history pruning with various limits

- Status tracking
  - Test status condition transitions during rollback
  - Test `updated` count calculation in summary
  - Test abort status tracking and timing

### Upgrade / Downgrade Strategy

#### Upgrade Strategy

When upgrading to a version that includes rollback support:

- **Existing ManifestWorkReplicaSets**: Existing MWRS resources will continue to work without modification. The controller will:
  - Initialize revision history on first reconciliation after upgrade
  - Create ControllerRevision for the current state if no history exists
  - Set default `revisionHistoryLimit` if not specified (default: 10)
  - Initialize status fields (currentRevision, updateRevision, collisionCount) based on current state

- **No Breaking Changes**: All new fields are optional and additive. Existing MWRS resources will automatically gain rollback capabilities without requiring spec changes.

- **Gradual Adoption**: Users can opt-in to automatic rollback by setting `abortOnFailure: true` in rollout strategies when ready.

#### Downgrade Strategy

When downgrading to a version without rollback support:

- **ControllerRevision Resources**: ControllerRevision resources created by the new version will remain in the cluster but will be ignored by the older controller. They can be manually cleaned up if desired.

- **Status Fields**: Older controllers will ignore unknown status fields (currentRevision, updateRevision, abort, abortedTime, collisionCount). The MWRS will continue to function, but rollback capabilities will be unavailable.

- **Spec Fields**: The `revisionHistoryLimit` and `abortOnFailure` fields will be ignored by older controllers. Rollout behavior will revert to the previous implementation (no automatic rollback).

- **ManifestWork Labels**: The `controller-hash` label will be ignored by older controllers but will not cause issues.

**Note**: During downgrade, any ongoing rollback operations should be completed before downgrading to avoid inconsistent state.

### Version Skew Strategy

During upgrades, there will be periods where different components are running different versions. The rollback feature is designed to handle version skew gracefully.

#### Hub Controller Version Skew

- **New Hub, Old Agents**: The new hub controller can create ControllerRevision resources and manage rollback, but managed clusters with older agents will continue to work normally. The `controller-hash` label on ManifestWork resources will be ignored by older agents but won't cause issues.

- **Old Hub, New Agents**: Older hub controllers will not create ControllerRevision resources or use rollback features. New agents will work normally but won't benefit from rollback capabilities until the hub is upgraded.

#### Guarantees

- **Backward Compatibility**: All new API fields are optional and additive. Older controllers will ignore unknown fields, ensuring backward compatibility.

- **Forward Compatibility**: New controllers can handle MWRS resources created by older versions by initializing revision history on first reconciliation.

- **No Data Loss**: During version skew, no data will be lost. ControllerRevision resources created by new versions will persist even if the controller is downgraded.

- **Graceful Degradation**: If rollback features are unavailable (due to version skew or missing ControllerRevision), the MWRS will continue to function with standard rollout behavior (no automatic rollback).

## Drawbacks

1. **Complexity**: The rollback mechanism adds complexity to the controller logic, particularly around revision management, status tracking, and abort handling. This increases the maintenance burden and potential for bugs.

2. **CRD Rollback Risks**: Automatic rollback of CRD changes can be dangerous and may cause data loss or system instability. Users must be educated about these risks.

## Alternatives

### Alternative 1: No Automatic Abort

Only provide manual rollback capabilities, requiring users to explicitly trigger rollbacks.

**Pros**: Simpler implementation, less risk of unintended rollbacks
**Cons**: Slower response to failures, requires manual intervention, may not meet user needs for critical applications

**Decision**: We chose ControllerRevision because it's a standard Kubernetes pattern, provides good balance of features and complexity, and integrates well with existing Kubernetes tooling.
