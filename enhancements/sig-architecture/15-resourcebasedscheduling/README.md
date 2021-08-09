# Resource Based Scheduling

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/website/)

## Summary

The proposed work would enhance the Placement APIs to support scheduling based on the resource usage/capacity of managed clusters.

## Motivation

During placement scheduling, managed clusters will be selected based on resource(s) (like memory and cpu etc.) usage/capacity to avoid scheduling workloads on clusters with insufficient resources.

Link to the feature request: https://github.com/open-cluster-management-io/community/issues/52

### User Stories

#### Story 1: User is able to select clusters with specific resource(s), whose allocatable to capacity ratio is larger.
  - User wants to deploy an application on a cluster whose CPU usage is less than others;
  - User wants to deploy an application on 2 clusters whose allocatable to capacity ratio of both CPU and memory is larger than others;

#### Story 2: User is able to select clusters with more allocatable specific resource(s) by specifying the resource(s) in placement.
  - User wants to deploy an application on a cluster with largest memory allocatable;

#### Story 3: User is able to select clusters based on some resource(s) usage and keep the decisions adjusted according to the usage changes.
  - A CPU-intensive application would like to be deployed on the cluster with least CPU utilization rate.

#### Story 4: User is able to select clusters based on some resource(s) usage once, without considering the usage changes afterwards, keeping the decisions pinned.
  - Some critical service should run on a steady set of clusters, even if the resource usage of those clusters changes;

#### Story 5: Support scheduling based on usage/allocatable of resources other than CPU and Memory. For example,
  - AI applications are expected to run on clusters with least gpu usage;
  - An application can be scheduled to cluster with largest available network bandwidth;

### Goals

- Story 1 and 2, allow user to select managed clusters based on resource usage/capability with the Placement APIs;
- Story 3 and 4, user is able to make the placement sensitive to resource usage or pin the placementdecisions and ignore the resource changes afterwards;

### Non-Goals
- Balance workload across the fleet based on cluster resource usage;
- Story 5 will not be supported until there is a clear requirement;

## Proposal

### 1. Changes on Placement APIs
Add a field `PrioritizerConfigs` into the Placement spec to allow users to weight prioritizers.
```go
type PlacementSpec struct {
	// ... ...

	// PrioritizerConfigs override the default configuration of the prioritizers
	// +optional
	PrioritizerConfigs []PrioritizerConfig `json:"prioritizerConfigs,omitempty"`
}

// PrioritizerConfig represents the configuration of prioritizer
type PrioritizerConfig struct {
	// Name is the name of prioritizer
	// +required
	Name string `json:"name"`

	// Weight defines the weight of prioritizer. The value must be ranged in [0,10].
	// +kubebuilder:validation:Minimum:=0
	// +kubebuilder:validation:Maximum:=10
	// +kubebuilder:default=1
	// +required
	Weight int32 `json:"weight"`
}
```

### 2. Creating a plugin, `resource`, to score managed clusters based on cluster resource usage.
The plugin is registered as the below 4 prioritizers with different names, which are disabled by default (weight is 0). 
1. `ResourceRatioCPU`, it scores managed clusters according to allocatable to capacity ratio of CPU.
2. `ResourceAllocatableCPU`, it scores managed clusters according to allocatable CPU.
3. `ResourceRatioMemory`, it scores managed clusters according to allocatable to capacity ratio of Memory.
4. `ResourceAllocatableMemory`, it scores managed clusters according to allocatable Memory.

According to the name it registered, the `resource` plugin uses different formulas to calculate the score of a managed cluster, the value falls in the range between -100 and 100.
| Prioritizer | Formula |
| - | - |
| `ResourceRatioCPU` | score = 2 * 100 * (0.5 - cpu_allocatable / cpu_capacity) |
| `ResourceAllocatableCPU` | score = 2 * 100 * (0.5 - (cpu_allocatable - min(cpu_allocatable)) / (max(cpu_allocatable) - min(cpu_allocatable))) |
| `ResourceRatioMemory` | score = 2 * 100 * (0.5 - memory_allocatable / memory_capacity) |
| `ResourceAllocatableMemory` | score = 2 * 100 * (0.5 - memory_allocatable - min(memory_allocatable)) / (max(memory_allocatable) - min(memory_allocatable))) |

### 3. Resource data required during scheduling
Currently the following resource data is available for both CPU and Memory in the status of ManagedCluster, which is updated by registration agent.
- `allocatable`, sum of the `allocatable` on each node in a managed cluster;
- `capacity`, sum of the `capacity` on each node in a managed cluster;

```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: local-cluster
spec:
  hubAcceptsClient: true
  leaseDurationSeconds: 60
status:
  allocatable:
    cpu: '15'
    memory: 63194Mi
  capacity:
    core_worker: '18'
    cpu: '18'
    memory: 69938Mi
    socket_worker: '6'
```
In order to support other resources, like GPU, the `allocatable` and `capacity` should be included in the status of managed cluster either.

### 4. How it works together with the existing Pritoritizers
Besides the new prioritizers, there are two exising prioritizers with weight 1 in the plugin framework of the scheduler.
- `balance`, it balances the number of decisions among the clusters. The cluster with the highest number of decison is given the lowest score, while the empty cluster is given the highest score. The score ranges from -100 to 100.
- `steady`, it ensures the existing decision is stabilized. The clusters that existing decisions choose are given the highest score while the clusters with no existing decisions are given the lowest score. The score is either 100 or 0.

When making cluster decisions, managed clusters are sorted by the final score of managed cluster, which is the sum of scores from all prioritizers with weights. User is able to adjust the weights of prioritizers to impact the final score, for example
- Set the weight of `ResourceRatioCPU`/`ResourceAllocatableCPU`/`ResourceRatioMemory`/`ResourceAllocatableMemory` to schedule placement based on resource usage;
- Make placement sensitive to resource usage by setting a higer weight for `ResourceRatioCPU`/`ResourceAllocatableCPU`/`ResourceRatioMemory`/`ResourceAllocatableMemory`;
- Ignore resource usage change and pin the placementdecisions by increasing the weight of `steady`;

## Examples

### 1. Select 3 clusters whose CPU/memory usage is less than others. (Use Story 1)

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement1
  namespace: ns1
spec:
  numberOfClusters: 3
  prioritizerConfigs:
    - name: ResourceRatioCPU
    - name: ResourceRatioMemory
```

### 2. Select a cluster with the largest memory available. (Use Story 2)

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement2
  namespace: ns1
spec:
  numberOfClusters: 1
  prioritizerConfigs:
    - name: ResourceAllocatableMemory
```

### 3. Make placement sensitive to resource changes. (Use Story 3)

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement3
  namespace: ns1
spec:
  numberOfClusters: 2
  prioritizerConfigs:
    - name: ResourceRatioCPU
      weight: 2
    - name: ResourceAllocatableMemory
      weight: 2
```

### 4. Pin the placementdecisions. (Use Story 4)
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement4
  namespace: ns1
spec:
  numberOfClusters: 4
  prioritizerConfigs:
    - name: steady
      weight: 3
    - name: ResourceAllocatableCPU
```

## Test Plan

- Unit tests cover the new plugin;
- Unit tests cover the 4 new prioritizers;
- Integration tests cover user story 1-4;

## Graduation Criteria
N/A

## Upgrade / Downgrade Strategy
N/A

## Version Skew Strategy
N/A
