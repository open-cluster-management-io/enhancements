# PlacementDecision RollingUpdate Strategy

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This enhancement introduces a rolling update strategy for `PlacementDecisions` to prevent clusters from temporarily disappearing during `PlacementDecisions` updates, which causes consumers like addon controllers to unnecessarily delete and recreate addons. Similar to Kubernetes Deployment's RollingUpdate strategy, this uses a surge-then-finalize approach to ensure zero-downtime updates.

## Motivation

When clusters are selected by `Placements` and distributed across multiple `PlacementDecisions` (each limited to 100 clusters), adding or removing clusters can cause them to shift between decisions. The current sequential update mechanism creates a time window where a cluster may temporarily not exist in any PlacementDecision, leading to incorrect resource deletion and recreation.

### Problem Statement

**Current Behavior:**
```
Initial State:
  decision-1: [cluster1...cluster100]
  decision-2: [cluster101...cluster150]

Action: Add cluster0 (triggers re-sorting)

Expected Final State:
  decision-1: [cluster0...cluster99]
  decision-2: [cluster100...cluster150]

Problem: During current decision update strategy
  T1: decision-1 updated → [cluster0...cluster99] (cluster100 removed)
  T2: decision-2 updated → [cluster100...cluster150] (cluster100 added)

  Time Window [T1, T2]: cluster100 exists in NO decision ❌
```

**Impact:**
Controllers watching `PlacementDecisions` interpret the temporary disappearance as cluster removal. For example, the addon-management-controller would delete and recreate `ManagedClusterAddOns` with different UIDs, this causes unnecessary workload disruption and resource churn across all PlacementDecision consumers

### Goals

- Update the `Placement` API to control how the `PlacementDecisions` being updated
- Eliminate the time window where clusters temporarily disappear from all `PlacementDecisions` during updates
- Preserve backward compatibility with existing PlacementDecision consumers

### Non-Goals

- How consumers watch or process `PlacementDecisions`

## Proposal

### User Stories

#### Story 1: Addon Stability During Cluster Scaling
As a cluster admin, when I add new/delete clusters to my fleet, I expect existing addon deployments to remain stable and not be unnecessarily recreated just because cluster indices shift in `PlacementDecisions`.

#### Story 2: Zero-Downtime Cluster Management
As an operations engineer, when managing a large fleet of clusters (>100 clusters requiring multiple `PlacementDecisions`), I need cluster reassignments between decisions to be transparent to controllers watching those decisions.

### Design Overview

Implement a **RollingUpdate strategy** controlled by a new field in the `Placement` API's `decisionStrategy`:

**Phase 1 (Surge):** Update each `PlacementDecision` with the union of old and new clusters

**Phase 2 (Finalize):** Update each `PlacementDecision` to contain only the new clusters and delete obsolete `PlacementDecisions`

This ensures that throughout the update process, every cluster is visible in at least one `PlacementDecision`, similar to how Deployment's RollingUpdate ensures pods are always available.

### Terminology

- **`clustersPerDecisionGroup`**: User-configurable parameter (default `100%`) that divides selected clusters into decision groups. See [Placement API](https://github.com/open-cluster-management-io/api/blob/main/cluster/v1beta1/types_placement.go) for details.
- **`maxNumOfClusterDecisions`**: Hard-coded constant (`100`) that limits each `PlacementDecision` object to max 100 clusters. If a decision group exceeds this, it splits into multiple `PlacementDecision` objects. See [scheduling controller](https://github.com/open-cluster-management-io/ocm/blob/main/pkg/placement/controllers/scheduling/scheduling_controller.go) for implementation.

### API Changes

Add a new optional field `updateStrategy` to the `decisionStrategy` in the Placement API:

**Note:** This is conceptually different from the existing `rolloutStrategy` in [`cluster/v1alpha1/types_rolloutstrategy.go`](https://github.com/open-cluster-management-io/api/blob/main/cluster/v1alpha1/types_rolloutstrategy.go). That API controls **how workloads are applied to clusters** (gradual rollout with health checks), while this new API controls **how PlacementDecision objects themselves are updated**.

```go
// DecisionStrategy divides the created placement decision to groups and defines number of clusters per decision group.
type DecisionStrategy struct {
    // groupStrategy defines strategies to divide selected clusters into decision groups.
    // +optional
    GroupStrategy GroupStrategy `json:"groupStrategy,omitempty"`

    // updateStrategy defines how PlacementDecision objects should be updated when
    // the set of selected clusters changes. This controls the update mechanism for the decision
    // objects themselves, not how workloads are rolled out to clusters.
    // +optional
    UpdateStrategy UpdateStrategy `json:"updateStrategy,omitempty"`
}

// UpdateStrategy defines how PlacementDecision objects should be updated.
type UpdateStrategy struct {
    // type indicates the type of the update strategy. Default is "All".
    // See UpdateStrategyType for detailed behavior of each type.
    //
    // +kubebuilder:validation:Enum=All;RollingUpdate
    // +kubebuilder:default=All
    // +optional
    Type UpdateStrategyType `json:"type,omitempty"`
}

// UpdateStrategyType is a string representation of the update strategy type
type UpdateStrategyType string

const (
    // All means update all PlacementDecisions immediately (current behavior).
    // Clusters may temporarily disappear from all decisions when moving between decisions,
    // which can cause consumers to incorrectly delete and recreate resources.
    All UpdateStrategyType = "All"

    // RollingUpdate means use a rolling update strategy similar to Deployment's RollingUpdate.
    // Phase 1 (Surge): Merge old and new clusters (temporary over-limit allowed)
    // Phase 2 (Finalize): Update to final state and delete obsolete PlacementDecisions
    // This ensures clusters remain visible in at least one decision throughout the update.
    RollingUpdate UpdateStrategyType = "RollingUpdate"
)
```

**Example Placement with RollingUpdate Enabled:**

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: my-placement
  namespace: default
spec:
  clusterSets:
    - global
  decisionStrategy:
    updateStrategy:
      type: RollingUpdate  # Enable rolling update
```

### Detailed Example

#### Scenario 1: Adding cluster0 causes cluster100 to move

```
Initial: decision-1: [cluster1...cluster100], decision-2: [cluster101...cluster150]
Add cluster0 → cluster100 moves from decision-1 to decision-2

RollingUpdate Process:
  Phase 1 (Surge): decision-1: [cluster0...cluster100] (101 clusters)
                   decision-2: [cluster100...cluster150] (cluster100 in both)
  Phase 2 (Finalize): decision-1: [cluster0...cluster99]
                      decision-2: [cluster100...cluster150]

Result: Only cluster0's addon created; cluster100's addon stable
```

#### Scenario 2: Deleting cluster0 causes cluster100 to move back

```
Initial: decision-1: [cluster0...cluster99], decision-2: [cluster100...cluster150]
Delete cluster0 → cluster100 moves from decision-2 to decision-1

RollingUpdate Process:
  Phase 1 (Surge): decision-1: [cluster0...cluster100] (101 clusters)
                   decision-2: [cluster100...cluster150] (cluster100 in both)
  Phase 2 (Finalize): decision-1: [cluster1...cluster100]
                      decision-2: [cluster101...cluster150]

Result: Only cluster0's addon deleted; all other addons stable (no recreation)
```

#### Scenario 3: clustersPerDecisionGroup changes (100% → 50)

```
Initial: 250 clusters in 3 decisions (100+100+50)
Change clustersPerDecisionGroup from 100% to 50 → redistribute to 5 decisions

RollingUpdate Process:
  Phase 1 (Surge): decision-0: [cluster1...cluster100] (unchanged)
                   decision-1: [cluster51...cluster200] (150 clusters)
                   decision-2: [cluster101...cluster250] (150 clusters)
                   decision-3: [cluster151...cluster200] (new, 50 clusters)
                   decision-4: [cluster201...cluster250] (new, 50 clusters)

  Phase 2 (Finalize): decision-0: [cluster1...cluster50]
                      decision-1: [cluster51...cluster100]
                      decision-2: [cluster101...cluster150]
                      decision-3: [cluster151...cluster200]
                      decision-4: [cluster201...cluster250]

Result: All 250 clusters visible throughout; zero addon churn; temporary surge to 150 per decision
```

### Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Phase 1 exceeds 100 clusters in a single `PlacementDecision` object | Low | Logged as info; expected during surge phase |
| Performance degradation | Low | 2x API calls (one for Phase 1 merge, one for Phase 2 finalize) |
| Backward compatibility | None | No API changes; transparent to consumers |

## Test Plan

## Graduation Criteria

### Alpha (Current Implementation)

- [ ] Add `updateStrategy` field to Placement CRD
- [ ] Default `updateStrategy.type` to "All" (current behavior)
- [ ] Controller supports both "All" and "RollingUpdate" types
- [ ] Unit tests and integration tests 

### Beta

- [ ] Recommend `updateStrategy.type` to "RollingUpdate"

### GA

- [ ] Default `updateStrategy.type` to "RollingUpdate"

## Upgrade / Downgrade Strategy

### Upgrade

- Existing `Placements` continue working without changes (backward compatible)
- Users can opt-in by setting `updateStrategy.type: RollingUpdate`

### Downgrade

- `Placements` with `updateStrategy.type: RollingUpdate` will fall back to "All" behavior
- Cluster movements may cause temporary disappearance again

## Alternatives Considered

### Alternative 1: Client-Side Delay with AddAfter

**Approach:** Add 100ms delay to all informer events in PlacementDecision consumers (e.g., addon-management-controller) to batch rapid sequential updates before reconciliation. See PRs: [ocm#1344](https://github.com/open-cluster-management-io/ocm/pull/1344), [sdk-go#191](https://github.com/open-cluster-management-io/sdk-go/pull/191)

**Implementation:**
- Modified sdk-go to support `WithInformersQueueKeysFuncAndDelay()` method
- Use WorkQueue's `AddAfter()` for automatic event deduplication
- If multiple PlacementDecision updates occur within 100ms, only one reconcile happens
- Delete events are still processed immediately for timely cleanup

**Code Example:**
```go
factory.New().
    WithInformersQueueKeysFuncAndDelay(
        queue.QueueKeyByMetaName,
        100*time.Millisecond,  // Batch events within 100ms
        addonInformers.Informer(),
        placementDecisionInformer.Informer()).
    WithSync(c.sync).ToController("addon-management-controller")
```

**Pros:**
- Solves the immediate problem for addon-management-controller
- Relatively simple implementation (~100 lines across sdk-go and OCM)
- No changes to Placement API or scheduling controller
- Backward compatible (consumers can opt-in via delay parameter)

**Cons:**
- **Consumer-side workaround**: Each consumer (addon-manager, manifestwork, policy controller) must implement the delay independently
- **Fixed delay heuristic**: 100ms may be insufficient for large clusters or excessive for small ones
- **Does not eliminate the race**: Only reduces probability; if updates take >100ms, the window still exists
- **Incomplete solution**: Does not address the root cause (sequential PlacementDecision updates)
- **Maintenance burden**: All PlacementDecision consumers need to be updated and tested

### Alternative 2: Annotation-Based Control

**Approach:** Add annotations to PlacementDecision objects to indicate when they are being updated. Consumers must check these annotations before processing PlacementDecision content to avoid acting on intermediate states.

**Implementation:**
- Add annotation `cluster.open-cluster-management.io/placement-update-status` to PlacementDecision
  - Value: `updating` - Decision is being updated, consumers should skip
  - Value: `ready` - Decision is stable, safe to process
- Placement controller sets annotation to `updating` before modifying `PlacementDecisions`
- After all decisions are updated, controller sets annotation to `ready`

**Example Annotation:**
```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: PlacementDecision
metadata:
  name: placement-1-decision-1
  annotations:
    cluster.open-cluster-management.io/placement-update-status: "updating"
spec:
  decisions:
    - clusterName: cluster1
    - clusterName: cluster2
```

**Pros:**
- Simple to implement (no API changes)
- Consumers can easily check annotation before processing
- Clear signal of update state
- No timing assumptions required

**Cons:**
- **Requires all consumers to be updated**: Each consumer must add annotation checking logic
- **Race window still exists**: Between annotation check and reading decision content
- **Incomplete protection**: Consumers that ignore annotations will still see intermediate states
- **Maintenance burden**: All PlacementDecision consumers need code changes