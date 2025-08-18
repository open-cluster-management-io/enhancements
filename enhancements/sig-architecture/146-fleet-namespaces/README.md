## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This enhancement introduces a global mechanism for defining and managing namespaces on Managed Clusters directly from
the hub. Namespaces can be associated with a ManagedClusterSet, enabling centralized application of Kubernetes access
controls through ClusterPermissions. Both the namespaces and their access controls are bound and managed by the
ManagedClusterSet, and applied consistently to all Managed Clusters in the ManagedClusterSet. This provides
multi-cluster governance and improves security consistency across fleets.

The solution extends the existing ManagedClusterSet API to include namespace configuration and leverages the
registration agent on each managed cluster to ensure namespace consistency. Each managed namespace is labeled and
annotated to track its management state and support multiple hub scenarios.

## Motivation

Currently, creating consistent namespaces across multiple clusters in a ManagedClusterSet requires manual coordination
using ManifestWork and ClusterPermission resources. This approach is error-prone, lacks automation, and makes it
difficult to maintain consistency as the fleet scales.

This enhancement addresses these challenges by introducing a declarative approach to namespace management at the
ManagedClusterSet level. The global mechanism enables:

- **Centralized Configuration**: Define namespaces once at the ManagedClusterSet level rather than per cluster
- **Automatic Propagation**: Namespaces are automatically created and maintained across all clusters in the set
- **Consistent Security**: ClusterPermissions can be applied uniformly across the managed namespaces
- **Simplified Operations**: Reduce operational overhead for managing namespaces across large fleets
- **Multi-tenancy Support**: Enable better isolation and resource organization for workloads like Virtual Machines

This enhancement focuses specifically on creating uniform namespace management across Managed Clusters in a
ManagedClusterSet, not managing all namespaces on the Managed Cluster.

### Goals

1. Update `ManagedClusterSet` API to support namespaces that span all clusters in a ManagedClusterSet (Global Namespaces)
2. Associate ClusterPermissions to the ManagedClusterSet in terms of the Global Namespace
3. Work with the existing RBAC tooling

### Non-Goals

1. Managing all namespaces on the Managed Cluster
2. Enforcing workload (manifestwork) to live in the managed namespace.

## Proposal

This section provides detailed implementation specifics for the namespace management enhancement.

### User Stories

#### Story 1
Create a Global Namespace. This namespace will be created on all Managed Clusters in a ManagedClusterSet, and the
appropriate ClusterPermissions will be applied.

#### Story 2
Delete a Global Namespace. This will either remove the namespace on all Managed Clusters or leave it in place but stop
managing it from the hub.

### Implementation Details/Notes/Constraints [optional]

Global Namespaces address a clear need, particularly for resources like virtual machines. They help guarantee
consistency in the architecture and assist when moving VMs and displaying VMs. While this could be accomplished using
ClusterPermissions and ManifestWork, it would not be automatic and the integration with ManagedClusterSet would be less
clear.

#### Controller Implementation

The implementation involves two main components:

1. **Hub Controller**: Monitors ManagedClusterSet resources for namespace configuration changes and propagates these
   changes to the status field of ManagedCluster resources that belong to the clusterset.

2. **Registration Agent**: Runs on each managed cluster and monitors the ManagedCluster status for namespace
   configuration updates. When changes are detected, the agent creates, updates, or removes namespaces according to the
   specified configuration.

#### Namespace Lifecycle Management

The namespace lifecycle follows these stages:

1. **Creation**: When a namespace is added to a ManagedClusterSet configuration, the hub controller updates all
   associated ManagedCluster statuses. The registration agent on each cluster detects this change and creates the
   namespace with appropriate labels and annotations.

2. **Update**: Changes to namespace configuration (such as deletion strategy) are propagated through the same mechanism.

3. **Removal**: When a namespace is removed from the ManagedClusterSet or when a cluster is removed from the set, the
   registration agent follows the configured deletion strategy (Keep or Delete).

#### Label and Annotation Strategy

Each managed namespace includes:
- **Labels**: `clusterset.open-cluster-management.io/<hub-hash>: "true"` for quick filtering
- **Annotations**: Detailed JSON configuration including clusterset name and deletion strategy for proper cleanup coordination

This approach ensures that multiple hubs can manage different namespaces on the same cluster without conflicts.

### Risks and Mitigation

1. **Deleting a Global Namespace**: Mitigated by implementing a configurable deletion strategy (Keep/Delete) that allows
   administrators to choose whether namespaces should be preserved or removed when no longer managed. The
   annotation-based tracking ensures proper cleanup coordination.

2. **Adopting existing namespaces**: When a namespace already exists on a cluster, the registration agent will adopt it
   by adding the appropriate labels and annotations without disrupting existing resources. A validation webhook can
   prevent conflicts with system namespaces.

3. **System namespace protection**: Implementation includes safeguards to prevent management of critical system
   namespaces (kube-system, kube-public, etc.) through validation at both the API level and agent level.

4. **Multi-hub conflicts**: The hub-hash based labeling system prevents conflicts when multiple OCM hubs attempt to
   manage the same cluster, ensuring each hub only manages its own namespace configurations.

5. **Agent failures**: If the registration agent fails or becomes unavailable, namespace configurations remain in place.
   When the agent recovers, it reconciles the current state with the desired configuration, ensuring eventual consistency.


## Design Details

### API Changes

#### ManagedClusterSet API Enhancement

The `ManagedClusterSet` API is extended with a new field to specify namespaces that should be managed across all
clusters in the set:

```go
// ManagedNamespaceConfig describes the configuration to manage namespaces on the clusters
type ManagedNamespaceConfig struct {
    // namespaces is a list of namespaces that will be managed across the clusterset.
    // Each namespace will be created on all clusters belonging to this ManagedClusterSet.
    // +optional
    Namespaces []string `json:"namespaces,omitempty"`
	
	// lifecycleStrategy defines the strategy when a namespace is removed from management,
	// the cluster is removed from the clusterset, or the clusterset is removed.
	// Valid values are:
	// - "Keep": Preserve the namespace and its contents on the managed cluster
	// - "Delete": Remove the namespace and all its contents from the managed cluster
	// +kubebuilder:validation:Enum=Keep;Delete
	// +optional
	LifecycleStrategy string `json:"lifecycleStrategy,omitempty"`
}
```

#### ManagedCluster API Status Enhancement

The `ManagedCluster` status is enhanced to reflect the namespace configuration inherited from associated clustersets:

```go
type ClusterSet struct {
	// name is the name of the clusterset
	Name string `json:"name,omitempty"`

    // namespaces is a list of namespaces that will be managed by the cluster,
	// inherited from the related clusterset. These namespaces will be created
	// and maintained by the registration agent on the managed cluster.
	Namespaces []string `json:"namespaces,omitempty"`

    // lifecycleStrategy defines the strategy when a namespace is removed from management,
    // the cluster is removed from the clusterset, or the clusterset is removed.
    // Valid values are "Keep" or "Delete".
	LifecycleStrategy string `json:"lifecycleStrategy,omitempty"`

	// conditions are the status conditions of the managed namespace
	Conditions []metav1.Condition `json:"conditions,omitempty"`
}
```

Since a `ManagedCluster` might belong to multiple clustersets, this will be a list in the status of the `ManagedCluster`.

#### Controller Workflow

1. **Hub Controller**: When a user updates the namespace configuration on a `ManagedClusterSet`, the hub controller
   identifies all `ManagedCluster` resources that belong to this clusterset and updates their status to reflect the new namespace configuration.

2. **Registration Agent**: The registration agent on each managed cluster watches for changes in the `ManagedCluster`
   status and reconciles the actual namespace state with the desired configuration.

3. **Conflict Resolution**: When a cluster belongs to multiple clustersets with overlapping namespace configurations,
   the registration agent merges the configurations and tracks the source of each namespace through annotations. 

Each namespace will have a label with key "clusterset.open-cluster-management.io/<hub hash>", indicating that the
namespace is currently managed by an OCM hub.

Each namespace will also have a set of annotations.

An example of the namespace will be like:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: some-namespace
  annotations:
    clusterset.open-cluster-management.io/hub1: "[{clustesetName: global, deletionStrategy: keep}, {clustesetName: set1, deletionStrategy: delete}]"
    clusterset.open-cluster-management.io/hub2: "[{clustesetName: global, deletionStrategy: keep}]"
  labels:
    clusterset.open-cluster-management.io/hub1: true
    clusterset.open-cluster-management.io/hub2: true
```

This ensures that information is correctly recorded when there are multiple agents from different hubs and the cluster
belongs to multiple clustersets. Labels help to quickly filter namespaces that are currently being managed by the hub,
while annotations contain more detailed information to handle removal.

### Return status of the managed namespace

When the registration agent applies the namespace to the managed cluster, it also should show the result in the status
of the managed cluster using the status condition to indicate:

1. a namespace is successfully applied to the cluster.
2. whether the namespace is in the deleting state.

### Management Removal

The management of the namespace is removed when:
1. The cluster is removed from the clusterset.
2. The clusterset is deleted.
3. The cluster is deleted.
4. The namespace is explicitly removed from the ManagedClusterSet configuration.

The registration agent needs to check if the related labels or annotations should be removed or updated when the above
cases occur.

#### Detailed Removal Process

1. **Configuration Change Detection**: The registration agent continuously monitors the ManagedCluster status for
   changes in namespace configuration.

2. **Cleanup Decision**: When a namespace is no longer managed, the agent checks the deletion strategy:
   - **Keep**: Removes OCM-specific labels and annotations but preserves the namespace and its contents
   - **Delete**: Removes the entire namespace and all its contents

3. **Multi-hub Coordination**: If multiple hubs manage the same cluster, the agent only removes labels and annotations
   specific to the hub that is no longer managing the namespace, preserving management by other hubs.

4. **Graceful Degradation**: If the hub becomes unavailable, the agent continues to maintain existing namespaces until
   connectivity is restored and new instructions are received.

### Open Questions [optional]

1. **Cross-cluster resource dependencies**: How should the system handle cases where resources in managed namespaces
   have dependencies across clusters in the same ManagedClusterSet?

2. **Namespace quotas and limits**: Should the global namespace management include propagation of ResourceQuotas and
   LimitRanges to ensure consistent resource constraints across all clusters?

3. **Integration with GitOps workflows**: How should this feature integrate with existing GitOps deployments that may
   already manage some of these namespaces?

4. **Monitoring and observability**: What metrics and events should be exposed to help administrators monitor the health
   and status of global namespace management across their fleet?

### Test Plan

Consider the following in developing a test plan for this enhancement:
- integration tests should cover the case:
  - add multiple namespace into clustersets
  - remove namespace from clustersets
  - add/remove cluster from clusterset
  - delete clusterset
  - cluster is added to multiple clusterset

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:

- [alpha]
  - namespace managed in clusterset is synced to all clusters in the clusterset
  - status of clusterset shows the status
- [beta]
  - allowing to specify lifecycle strategy of each namespace.
  - enforcing workload namespace
- [graduate]
  - performance tests are preformed
  - at least 3 consumers.

Clearly define what graduation means by either linking to the [API doc definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),
or by redefining what graduation means.

In general, we try to use the same stages (alpha, beta, stable), regardless how the functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

### Upgrade / Downgrade Strategy

Upgrade/Downgrade will not be impacted.

### Version Skew Strategy

The agent with old version will ignore the field, which should be fine

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

The idea is to find the best form of an argument why this enhancement should _not_ be implemented.

## Alternatives

### Using ManifestWork

We could build a hub controller to create a manifestwork to apply namespaces to the managed cluster
based on configurations in the clusterset. It has benefit that we do not need to implement
label/annotation management for the namespace. The issue with this approach is:
1. it introduces depedency that a feature in registration component depends on work component
2. it introduces an additional manifestwork for each cluster.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.