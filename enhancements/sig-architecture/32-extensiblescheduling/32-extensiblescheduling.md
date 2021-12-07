# placement extensible scheduling

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/website/)

## Summary

The proposed work provides an API to represent the score of the managed cluster, to support placement extensible scheduling.

## Motivation

When implementing placement resource based scheduling, we find in some cases, the prioritizer needs extra data (more than the default value provided by `ManagedCluster`) to calculate the score of the managed cluster. For example, there is a requirement to schedule based on resource monitoring data from the cluster.

So we want a more extensible way to support scheduling based on customized scores.

### Goals
- Design a new API(CRD) to contain the customized scores for each managed cluster.
- Let placement prioritizer support rating clusters with the customized scores provided by the new API(CRD).

### Non-Goals
- How to maintain the lifecycle (create/update/delete) of the CRs.
- Placement filters will not support the new API(CRD).

  Putting this into non-goals is because most of the cases we know are that users want to use customized values to prioritize clusters. And for filters, adding labels to filter is easier than supporting customized values to filter.

## Proposal

### User Stories

#### Story 1: Users could use the data pushed from each managed cluster to select clusters.
  - On each managed cluster, there is a customized agent monitoring the resource usage (eg. CPU ratio) of the cluster. It will calculate a score and push the result to the hub.
  - As an end user, I can configure placement yaml to use this score to select clusters.
  
#### Story 2: Users could use the metrics collected on the hub to select clusters.
  - On the hub, there is a customized agent to get metrics (eg. cluster allocatable memory) from Thanos. It will generate a score for each cluster.
  - As an end user, I can configure placement yaml to use this score to select clusters.

#### Story 3: Disaster recovery workload could be automatically switched to an available cluster.
  - A user has two clusters, a primary and backup cluster on which workload storage is configured so storage is synced from primary cluster to backup.
  - As an end user, I want to deploy workloads on the primary cluster first. And when the primary cluster is down, the workload should be automatically switched to the backup cluster.

#### Story 4: Users could use the data provided by third party controllers to select clusters.
  - A third party controller can comprehensively evaluate cluster on latency, region, iops etc. and rate cluster according to the SLA the cluster can provide.
  - As an end user, I can configure placement yaml to use this score to select clusters.

### Risks and Mitigation
N/A

## Design Details

### AddOnPlacementScore API
AddOnPlacementScore is the new API to add, to represent a list of scores of one managed cluster.
```go
// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +kubebuilder:resource:scope="Namespaced"
// +kubebuilder:subresource:status

// AddOnPlacementScore represents a bundle of scores of one managed cluster, which could be used by placement.
// AddOnPlacementScore is a namespace scoped resource. The namespace of the resource is the cluster namespace.
type AddOnPlacementScore struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// Status represents the status of the AddOnPlacementScore.
	// +optional
	Status AddOnPlacementScoreStatus `json:"status,omitempty"`
}

//AddOnPlacementScoreStatus represents the current status of AddOnPlacementScore.
type AddOnPlacementScoreStatus struct {
	// Conditions contain the different condition statuses for this AddOnPlacementScore.
	// +optional
	Conditions []metav1.Condition `json:"conditions"`

	// Scores contains a list of score name and value of this managed cluster.
	// +optional
	Scores []AddOnPlacementScoreItem `json:"scores,omitempty"`

	// ValidUntil defines the valid time of the scores.
	// After this time, the scores are considered to be invalid by placement. nil means never expire.
	// The controller owning this resource should keep the scores up-to-date.
	// +optional
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Type=string
	// +kubebuilder:validation:Format=date-time
	ValidUntil *metav1.Time `json:"validUntil" protobuf:"bytes,4,opt,name=lastTransitionTime"`
}

//AddOnPlacementScoreItem represents the score name and value.
type AddOnPlacementScoreItem struct {
	// Name is the name of the score
	// +kubebuilder:validation:Required
	// +required
	Name string `json:"name"`

	// Value is the value of the score. The score range is from -100 to 100.
	// +kubebuilder:validation:Minimum:=-100
	// +kubebuilder:validation:Maximum:=100
	// +required
	Value int32 `json:"value,omitempty"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// AddOnPlacementScoreList is a collection of AddOnPlacementScore.
type AddOnPlacementScoreList struct {
	metav1.TypeMeta `json:",inline"`
	// Standard list metadata.
	// More info: https://git.k8s.io/community/contributors/devel/api-conventions.md#types-kinds
	// +optional
	metav1.ListMeta `json:"metadata,omitempty"`

	// Items is a list of AddOnPlacementScore
	Items []AddOnPlacementScore `json:"items"`
}
```
### Placement API
The changes in Placement API includes:

- Add `PrioritizerScoreCoordinate` inside `PrioritizerConfig`.

  `PrioritizerScoreCoordinate` represents the configuration of the prioritizer and customized scores. `PrioritizerScoreCoordinate` will replace `Name` in the future.

- Support configuring negative prioritizer weight. 

  A negative weight indicates "not suggest to select', or "reverse selection". For example, a user owns 5 clusters, he wants to send the workload to the top one, and also wants to get the last one to check if there is something wrong with that cluster or do something else. With negative weight, he can achieve it by creating 2 placements, one with weight 1 to select the top one, and another one with weight -1 to select the last one.

```golang
type Placement struct {
  ...
	Spec PlacementSpec `json:"spec"`
}

type PlacementSpec struct {
  ...

	// PrioritizerPolicy defines the policy of the prioritizers.
	// If this field is unset, then default prioritizer mode and configurations are used.
	// Referring to PrioritizerPolicy to see more description about Mode and Configurations.
	// +optional
	PrioritizerPolicy PrioritizerPolicy `json:"prioritizerPolicy"`
}

// PrioritizerPolicy represents the policy of prioritizer
type PrioritizerPolicy struct {
  ...

	// +optional
	Configurations []PrioritizerConfig `json:"configurations,omitempty"`
}

// PrioritizerConfig represents the configuration of prioritizer
type PrioritizerConfig struct {
	// Name will be deprecated in v1beta1 and replaced by PrioritizerScoreCoordinate.BuildIn.
	// If both Name and PrioritizerScoreCoordinate.BuildIn are defined, will use the value
	// in PrioritizerScoreCoordinate.BuildIn.
	// Name is the name of a prioritizer. Below are the valid names:
	// 1) Balance: balance the decisions among the clusters.
	// 2) Steady: ensure the existing decision is stabilized.
	// 3) ResourceAllocatableCPU & ResourceAllocatableMemory: sort clusters based on the allocatable.
	// +kubebuilder:validation:Required
	// +required
	Name string `json:"name"`

	// PrioritizerScoreCoordinate represents the configuration of the prioritizer and score source.
	// +optional
	PrioritizerScoreCoordinate PrioritizerScoreCoordinate `json:"scoreCoordinate"`

	// Weight defines the weight of the prioritizer score. The value must be ranged in [-10,10].
	// Each prioritizer will calculate an integer score of a cluster in the range of [-100, 100].
	// The final score of a cluster will be sum(weight * prioritizer_score).
	// A higher weight indicates that the prioritizer weights more in the cluster selection,
	// while 0 weight indicates that the prioritizer is disabled. A negative weight indicates
	// wants to select the last ones.
	// +kubebuilder:validation:Minimum:=-10
	// +kubebuilder:validation:Maximum:=10
	// +kubebuilder:default:=1
	// +optional
	Weight int32 `json:"weight,omitempty"`
}

// PrioritizerScoreCoordinate represents the configuration of the prioritizer and score source
type PrioritizerScoreCoordinate struct {
	// Type defines the type of the prioritizer.
	// Type is either "BuildIn", "AddOn" or "", where "" is "BuildIn" by default.
	// When the type is "BuildIn", need to specify a buildin prioritizer name in BuildIn.
	// When the type is "AddOn", need to configure the score source in AddOn.
	// +kubebuilder:default:=BuildIn
	// +optional
	Type string `json:"type,omitempty"`

	// BuildIn defines the name of a buildin prioritizer. Below are the valid buildin prioritizer names.
	// 1) Balance: balance the decisions among the clusters.
	// 2) Steady: ensure the existing decision is stabilized.
	// 3) ResourceAllocatableCPU & ResourceAllocatableMemory: sort clusters based on the allocatable.
	// +optional
	BuildIn string `json:"buildIn,omitempty"`

	// When type is "AddOn", AddOn defines the resource name and score name.
	// +optional
	AddOn PrioritizerAddOnScore `json:"addOn"`
}

// PrioritizerAddOnScore represents the configuration of the addon score source.
type PrioritizerAddOnScore struct {
	// ResourceName defines the resource name of the AddOnPlacementScore.
	// The placement prioritizer selects AddOnPlacementScore CR by this name.
	// +kubebuilder:validation:Required
	// +required
	ResourceName string `json:"resourceName"`

	// ScoreName defines the score name inside AddOnPlacementScore.
	// AddOnPlacementScore contains a list of score name and score value, ScoreName specify the score to be used by
	// the prioritizer.
	// +kubebuilder:validation:Required
	// +required
	ScoreName string `json:"scoreName"`
}
```

### Let placement prioritizer support rating clusters with the customized score of the CR.
How to maintain the lifecycle of AddOnPlacementScore CRs is out of scope in this proposal. Here we suppose there already are AddOnPlacementScore CRs created in each managed cluster namespace before placement attempts to use it.
A valid AddOnPlacementScore CR resembles as below:
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: AddOnPlacementScore
metadata:
  name: default
  namespace: cluster1
status:
  conditions:
  - lastTransitionTime: "2021-10-28T08:31:39Z"
    message: AddOnPlacementScore updated successfully
    reason: AddOnPlacementScoreUpdated
    status: "True"
    type: AddOnPlacementScoreUpdated
  validUntil: "2021-10-29T18:31:39Z"
  scores:
  - name: "cpuratio"
    value: 88
  - name: "memratio"
    value: 77
  - name: "cpu"
    value: 66
  - name: "mem"
    value: 55
```
The `AddOnPlacementScore` resource name is "default" in this example, the namespace "cluster1" is the managed cluster namespace.
The `status.scores` contains a list of score items, for example, score "cpuratio" has value "88".

As an end user, can define a placement as below to use the scores in `AddOnPlacementScore` to sort clusters.
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement
  namespace: ns1
spec:
  numberOfClusters: 3
  prioritizerPolicy:
    mode: Exact
    configurations:
      - scoreCoordinate:
          type: BuildIn
          buildIn: Steady
        weight: 3
      - scoreCoordinate:
          type: AddOn
          addOn:
            resourceName: default
            scoreName: cpuratio
        weight: 1
```

In the above example, the placement defines buildin prioritizer "Steady" with weight 3, and as well as using score "cpuratio" in "default" AddOnPlacementScore to sort clusters.

When this placement.yaml is created, the placement controller sorts the clusters with "BuildIn" prioritizer "Steady" and "AddOn" score "cpuratio". The "BuildIn" prioritizer scheduling behavior keeps the same before, the "AddOn" type prioritizer scheduling process is as below:

- First schedule

The prioritizer will get customized score "cpuratio" from "default" `AddOnPlacementScore`, and calculate a final score for each managed clusters.

- Reschedule

The scores inside `AddOnPlacementScore` are supposed to be updated frequently. Rather than watch the changes in `AddOnPlacementScore` and trigger reschedule, the scheduler framework would periodically (eg. every 5 minutes) reschedule the placement.

**What if no valid score is generated?**

For the invalid score cases, for example, the AddOnPlacementScore CR is not created for some managed clusters, the score is missing in some CR, or the score is expired. The prioritizer will give those clusters with score 0, and the final placement decision is still made by the total score of each prioritizers.

### How to maintain the lifecycle of the AddOnPlacementScore CRs? 
The details of how to maintain the AddOnPlacementScore CRs lifecycle is out of scope, however, this proposal would like to give some suggestions about how to implement a 3rd party controller for it.
- Where should the 3rd party controller run? 

  The 3rd part controller could run on either hub or managed cluster. Just like what is described in user story 1 & 2.

- When should the score be created?

  It can be created with the existence of a ManagedCluster, or on demand for the purpose of reducing objects on hub.

- When should the score be updated?

  We recommend that you set `ValidUntil` when updating the score, so that the placement controller can know if the score is still valid in case it failed to update for a long time. An expired score will be treated as score 0 in placement controller.

  The score could be updated when your monitoring data changes, or at least you need to update it before it expires.

- How to calculate the score.

  The score must be in the range -100 to 100, you need to normalize the scores before update it into `AddOnPlacementScore`.

  For instance if it is a controller on hub to maintain the score, it can normalize based on max/min value it collects. While if the controller is running on the managed cluster, it can use a reasonable max/min value to normalize the score, eg, if the cluster memory allocatable is larger than 100GB, score is 100, if less than 1GB give score -100, and other scores distributed among them.

Besides the above suggestions, we would consider a common library to make it easier.

### Examples

#### A customized controller updates scores to API for managed clusters, and users could use these scores to select clusters.(Use Story 1&2)
1. A customized controller creates "default" `AddOnPlacementScore` for each managed cluster, and update scores "cpuratio", "memratio" into it.
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: AddOnPlacementScore
metadata:
  name: default
  namespace: {managed cluster namespace}
status:
  conditions:
  - lastTransitionTime: "2021-10-28T08:31:39Z"
    message: AddOnPlacementScore updated successfully
    reason: AddOnPlacementScoreUpdated
    status: "True"
    type: AddOnPlacementScoreUpdated
  validUntil: "2021-10-29T18:31:39Z"
  scores:
  - name: "cpuratio"
    value: 88
  - prioritizer: "memratio"
    value: 77
```
2. User creates a new placement as below to select clusters with most "cpuratio" score provided by AddOnPlacementScore "default", so that he can deploy the workload to those clusters..
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement
  namespace: ns1
spec:
  numberOfClusters: 3
  prioritizerPolicy:
    mode: Exact
    configurations:
      - scoreCoordinate:
          type: AddOn
          addOn:
            resourceName: default
            scoreName: cpuratio
        weight: 1
```
3. User creates a new placement as below to select clusters with least "cpuratio" score provided by AddOnPlacementScore "default", so that he can check if there is something wrong with that cluster.
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement
  namespace: ns1
spec:
  numberOfClusters: 1
  prioritizerPolicy:
    mode: Exact
    configurations:
      - scoreCoordinate:
          type: AddOn
          addOn:
            resourceName: default
            scoreName: cpuratio
        weight: -1
```

#### Disaster recovery workload could be automatically switched to an available cluster. (Use Story 3)
1. A user has 2 clusters, one is primary and one is backup cluster. A customized controller update score into it. The primary has score 100 and backup has score 0, and validUntil is nil (the score never expire).
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: AddOnPlacementScore
metadata:
  name: disasterrecovery
  namespace: primary
status:
  conditions:
  - lastTransitionTime: "2021-10-28T08:31:39Z"
    message: AddOnPlacementScore updated successfully
    reason: AddOnPlacementScoreUpdated
    status: "True"
    type: AddOnPlacementScoreUpdated
  scores:
  - name: "workload"
    value: 100
```
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: AddOnPlacementScore
metadata:
  name: disasterrecovery
  namespace: backup
status:
  conditions:
  - lastTransitionTime: "2021-10-28T08:31:39Z"
    message: AddOnPlacementScore updated successfully
    reason: AddOnPlacementScoreUpdated
    status: "True"
    type: AddOnPlacementScoreUpdated
  scores:
  - name: "workload"
    value: 0
```
2. To deploy workloads on the primary cluster, the user define the placement as below.  
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement
  namespace: ns1
spec:
  numberOfClusters: 1
  prioritizerPolicy:
    mode: Exact
    configurations:
      - scoreCoordinate:
          type: AddOn
          addOn:
            resourceName: disasterrecovery
            scoreName: workload
        weight: 1
```
As cluster primary has a higher score, the Placement decision will chose it and workload will be running on the primary. 

3. When the primary cluster is unavailable (a taint is added). The placement will filter out the primary because of taint, and the backup cluster will be chosen. Then disaster recovery workload will be running on the backup cluster automatically.

### Test Plan

- Unit tests cover placement decisions when AddOnPlacementScore status changes/expire/empty.
- Integration tests cover user story 1-4;

### Graduation Criteria
#### Alpha
1. The new APIs is reviewed and accepted;
2. Implementation is completed to support the functionalities;
3. Develop test cases to demonstrate that the above user stories work correctly;

#### Beta
1. Need to revisit the API shape before upgrading to beta based on user’s feedback.

### Upgrade / Downgrade Strategy
N/A

### Version Skew Strategy
N/A

## Alternative
### Support user-defined scheduler
As an end user, can specify the user-defined scheduler name in placement, and let a customized scheduler generate placement decisions. In this way, extensible scheduling can also be achieved.

Pros: 
  - More flexible, the customized scheduler can schedule based on resource usage, SLA, metrics, etc.

Cons:
  - More effort to write a customized scheduler.
  - The customized scheduler needs to understand all the spec of placement api, the changes of the api.