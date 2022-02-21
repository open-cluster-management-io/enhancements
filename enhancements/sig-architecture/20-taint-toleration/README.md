# Taint-Toleration in Placement APIs

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary

The proposed work would define Taints in MnagedCluster and Tolerations in placement APIs to allow users to control the selection of managed clusters more flexibly, and evict selected unhealthy/not-reporting clusters after a certain period.

## Motivation

For the new placement enhancement need of supporting filtering unhealthy/not-reporting clusters and keep workloads from being placed in unhealthy or unreachable clusters, we may consider the similar logic of taint/toleration in Kubernetes. Basically, there are two things we could do:

1) Implement taint/toleration in placement apis.
2) Add a new controller to map states of clusters to taints automatically.

Taints and tolerations would work together to allow users to control the selection of managed clusters more flexibly, and evict selected clusters after a certain period. Specifically, Taints are properties of managed clusters, they allow a placement to repel a set of managed clusters. Tolerations are applied to placements, and allow the managed clusters with matching taints to be scheduled onto placements.

### User Stories

Story 1: Users schedule some workloads to certain managed clusters.

Story 2: When a managed cluster gets offline, the system can make applications deployed on this cluster to be transferred to another available managed cluster immediately. 

Story 3: User can set a cluster to maintaining status, so workloads will be evicted and no workloads will be scheduled to the cluster. User can set security configuration still in these clusters even the cluster is in maintaining mode.

Story 4: When a managed cluster gets offline, users can make applications deployed on this cluster to be transferred to another available managed cluster after a tolerated time.

## Proposal

### API repo

#### ManagedClusterSpec
In `ManagedClusterSpec`, we can add `taints` which has the similar structure with taints in Kubernetes. 

```go
type ManagedClusterSpec struct {
	ManagedClusterClientConfigs []ClientConfig `json:"managedClusterClientConfigs,omitempty"`
	HubAcceptsClient bool `json:"hubAcceptsClient"`
	LeaseDurationSeconds int32 `json:"leaseDurationSeconds,omitempty"`

	// Taints is a property of managed cluster that allow the cluster to be repelled when scheduling.
	// Taints, including 'ManagedClusterUnavailable' and 'ManagedClusterUnreachable', can not be added/removed by agent
	// running on the managed cluster; while it's fine to add/remove other taints from either hub cluser or managed cluster.
	// +optional
	Taints []Taint `json:"taints,omitempty"`
}
 
type Taint struct {
	// Key is the taint key applied to a cluster. e.g. bar or foo.example.com/bar.
	// The regex it matches is (dns1123SubdomainFmt/)?(qualifiedNameFmt)
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Pattern=`^([a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*/)?(([A-Za-z0-9][-A-Za-z0-9_.]*)?[A-Za-z0-9])$`
	// +kubebuilder:validation:MaxLength=316
	// +required
	Key string `json:"key"`
	// Value is the taint value corresponding to the taint key.
	// +kubebuilder:validation:MaxLength=1024
	// +optional
	Value string `json:"value,omitempty"`
	// Effect indicates the effect of the taint on placements that do not tolerate the taint.
	// Valid effects are NoSelect, PreferNoSelect and NoSelectIfNew.
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Enum:=NoSelect;PreferNoSelect;NoSelectIfNew
	// +required
	Effect TaintEffect `json:"effect"`
	// TimeAdded represents the time at which the taint was added.
	// +nullable
	// +required
	TimeAdded metav1.Time `json:"timeAdded"`
}

type TaintEffect string
const (
	// TaintEffectNoSelect means placements are not allowed to select the cluster unless they tolerate the taint.
	// The cluster will be removed from the placement cluster decisions if a placement has already selected
	// this cluster.
	TaintEffectNoSelect TaintEffect = "NoSelect"
	// TaintEffectPreferNoSelect means the scheduler tries not to select the cluster, rather than prohibiting
	// placements from selecting the cluster entirely.
	TaintEffectPreferNoSelect TaintEffect = "PreferNoSelect"
	// TaintEffectNoSelectIfNew means placements are not allowed to select the cluster unless
	// 1) they tolerate the taint;
	// 2) they have already had the cluster in their cluster decisions;
	TaintEffectNoSelectIfNew TaintEffect = "NoSelectIfNew"
)

const (
	// ManagedClusterTaintUnavailable is the key of the taint added to a managed cluster when it is not available.
	// To be specific, the cluster has a condtion 'ManagedClusterConditionAvailable' with status of 'False';
	ManagedClusterTaintUnavailable string = "cluster.open-cluster-management.io/unavailable"
	// ManagedClusterTaintUnreachable is the key of the taint added to a managed cluster when it is not reachable.
	// To be specific,
	// 1) The cluster has no condition 'ManagedClusterConditionAvailable';
	// 2) Or the status of condtion 'ManagedClusterConditionAvailable' is 'Unknown';
	ManagedClusterTaintUnreachable string = "cluster.open-cluster-management.io/unreachable"
)
```

#### PlacementSpec
Similarly, we add "Kubernetes like" `tolerations` to the `PlacementSpec`, which has `key`, `value`, `operator`, and `tolerationSeconds`, while skip `effect` now.

```go
type PlacementSpec struct {
	ClusterSets []string `json:"clusterSets,omitempty"`
	NumberOfClusters *int32 `json:"numberOfClusters,omitempty"`
	Predicates []ClusterPredicate `json:"predicates,omitempty"`
	// Tolerations are applied to placements, and allow (but do not require) the managed clusters with
	// certain taints to be selected by placements with matching tolerations.
	// +optional
	Tolerations []Toleration `json:"tolerations,omitempty"`
}
 
// Toleration represents the toleration object that can be attached to a placement.
// The placement this Toleration is attached to tolerates any taint that matches
// the triple <key,value,effect> using the matching operator <operator>.
type Toleration struct {
	// Key is the taint key that the toleration applies to. Empty means match all taint keys.
	// If the key is empty, operator must be Exists; this combination means to match all values and all keys.
	// +kubebuilder:validation:Pattern=`^([a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*/)?(([A-Za-z0-9][-A-Za-z0-9_.]*)?[A-Za-z0-9])$`
	// +kubebuilder:validation:MaxLength=316
	// +optional
	Key string `json:"key,omitempty"`
	// Operator represents a key's relationship to the value.
	// Valid operators are Exists and Equal. Defaults to Equal.
	// Exists is equivalent to wildcard for value, so that a placement can
	// tolerate all taints of a particular category.
	// +kubebuilder:default:="Equal"
	// +optional
	Operator TolerationOperator `json:"operator,omitempty"`
	// Value is the taint value the toleration matches to.
	// If the operator is Exists, the value should be empty, otherwise just a regular string.
	// +kubebuilder:validation:MaxLength=1024
	// +optional
	Value string `json:"value,omitempty"`
	// Effect indicates the taint effect to match. Empty means match all taint effects.
	// When specified, allowed values are NoSelect, PreferNoSelect and NoSelectIfNew.
	// +kubebuilder:validation:Enum:=NoSelect;PreferNoSelect;NoSelectIfNew
	// +optional
	Effect v1.TaintEffect `json:"effect,omitempty"`
	// TolerationSeconds represents the period of time the toleration (which must be of effect
	// NoSelect/PreferNoSelect, otherwise this field is ignored) tolerates the taint.
	// The default value is nil, which indicates it tolerates the taint forever.
	// The start time of counting the TolerationSeconds should be the TimeAdded in Taint, not the cluster
	// scheduled time or TolerationSeconds added time.
	// +optional
	TolerationSeconds *int64 `json:"tolerationSeconds,omitempty"`
}
```

### Placement repo

#### Four steps of current schedule:

* Filter: Get clusters satisfying all the requirements from all the clusters. 

* Score: Sort by certain criteria.

* Select: Only select the number of clusters we need, not all feasible clusters.

* Bind: merge the cluster decisions into placementdecisions

#### Changes:

* Current placement process is implemented by plugins. To add taint-toleration, we need to add a new plugin which check taint-toleration matching after the filter pipline in the schedule function. Specifically, we do following matching checking in this new plugin:
  * for every taint of all the taints on a managed cluster
  * go through every toleration on the placement
  * check if there is a match
* By the end of the schedule process, we check all tolerated taints to get the first expiration time of the cluster.
* `scheduleResult` should be updated by adding `requeue` and `delay`, which represent whether requeue need to be triggered in the controller and after how many seconds it would be triggerd. 
* We also add a requeue process in the controller to evict the cluster after this `delay` time. `delay` time is the first expiration time of tolerated taints, and it would be computed as: toleration.TolerationSeconds - (current time - taint.TimeAdded), and the first expiration time is the minimum among all expiration times.
* Update the plugin interface, which can return the `requeueAfter` time for each plugin.

### Register repo

#### Add new controller:

Based on what we have in current Placement APIs, we add a new controller which can map the state of unhealthy and unreachable managed clusters to corresponding taints `cluster.open-cluster-management.io/unavailable` and `cluster.open-cluster-management.io/unreachable`, so we can leverage the Taint-Toleration to support[ filtering unhealthy/not-reporting clusters (issue #48)](https://github.com/open-cluster-management-io/community/issues/48).

### Examples

#### 1. Cluster Maintaining

Users (admins) can set a cluster to maintaining status, so workloads will be evicted and no workloads will be scheduled to the cluster. There's no need to set the toleration in placement.yaml under this senario. If toleration has to be set, make sure there won't be a matching key-value with the `maintaining` key-value in taint.

cluster.yaml:

```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: cluster1
  labels:
    cluster.open-cluster-management.io/clusterset: clusterset1
spec:
  hubAcceptsClient: true
  taints:
    - key: maintaining
      value: "true"
      effect: "NoSelect"
      timeAdded: 2021-07-06T15:00:00+08:00
```

placement.yaml:

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement1
  namespace: default
spec: ~
```

#### 2. Schedule on Certain Clusters

Developers want to schedule some workloads to certain managed clusters.

cluster.yaml:

```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: cluster1
  labels:
    cluster.open-cluster-management.io/clusterset: clusterset1
spec:
  hubAcceptsClient: true
  taints:
    - key: gpu
      value: "true"
      effect: "NoSelectIfNew"
      timeAdded: 2021-07-06T15:00:00+08:00
```
placement.yaml:

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement1
  namespace: default
spec:
  tolerations:
    - key: gpu
      value: "true"
      operator: Equal
```

#### 3. Support Filtering Unhealthy/Unreachable Clusters

When managed clusters' availability becomes `false`, the taint `cluster.open-cluster-management.io/unavailable` would be automatically added to clusters. If managed clusters' availability becomes `unknown`, the taint `cluster.open-cluster-management.io/unreachable` would be added. Those unhealthy or unreachable clusters could be evicted immediatly or after `TolerationSeconds`.

##### 3.1 Immediate Eviction

Admins or developers want applications deployed on this cluster to be transferred to another available managed cluster immediately.

cluster.yaml:

```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: cluster1
  labels:
    cluster.open-cluster-management.io/clusterset: clusterset1
spec:
  hubAcceptsClient: true
  taints:
    - key: gpu
      value: "true"
      effect: "NoSelectIfNew"
      timeAdded: 2021-07-06T15:00:00+08:00
    - key: unhealthy
      effect: "NoSelect"
      timeAdded: 2021-07-06T15:00:00+08:00
```

placement.yaml:

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement1
  namespace: default
spec:
  tolerations:
    - key: gpu
      value: "true"
      operator: Equal
```

##### 3.2 Eviction After TolerationSeconds

When a managed cluster gets offline, developers or admins want applications deployed on this cluster to be transferred to another available managed cluster after a tolerated time.

cluster.yaml:

```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: cluster1
  labels:
    cluster.open-cluster-management.io/clusterset: clusterset1
spec:
  hubAcceptsClient: true
  taints:
    - key: gpu
      value: "true"
      effect: "NoSelectIfNew"
      timeAdded: 2021-07-06T15:00:00+08:00
    - key: unreachable
      effect: "NoSelect"
      timeAdded: 2021-07-06T15:00:00+08:00
```

placement.yaml:

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement1
  namespace: default
spec:
  tolerations:
    - key: gpu
      value: "true"
      operator: Equal
    - key: unreachable
      operator: Exists
      tolerationSeconds: 90
```

### Test Plan

- Unit tests will cover the functionality of the controller.
- Unit tests will cover the new plugin.
- Integration tests will cover all user stories.
- e2e tests will cover the following cases:
  - Choose clusters by creating a taint with matched toleration;
  - Prevent choosing clusters by creating a taint without matched toleration;
  - `unhealthy`/`unreachable` taints are added automaticaly when availabiliies of clusters become `false`/`unknown`;
  - Clusters with `unhealthy`/`unreachable` taints are evicted immediately when tolerationSeconds <= 0 or was not set;
  - Clusters with `unhealthy`/`unreachable` taints are evicted after tolerationSeconds when tolerationSeconds > 0;

### Graduation Criteria

#### Alpha
1. The new plugin is reviewed and accepted;
2. Implementation is completed to support the functionalities of taint-toleration;
3. Develop test cases to demonstrate that the above user stories work correctly;

#### Beta
1. Support mapping unhealthy/unreachable clusters to taints automatically by adding controllers in Register repo;

#### GA
1. Pass the performance/scalability testing;

### Upgrade Strategy

### Version Skew Strategy

## Alternatives

### Use affinity by adding timeAdded to claims, instead of taint-toleration to ensure placement can repel inappropriate clusters.

1. Pros

- Less code, less implementation.

2. Cons

- As for user story 1, if users want to make certain clusters unselectable by using affinity, they also need to know affinity settings on placements. While by using taint-toleration, users can be awareless to tolerations. Similarly, user story 4 is also not supported.
- Only claims can be modified to add TolerationSeconds, while labels can’t.
