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
- How to maintain the life cycle (create/update/delete) of the CRs.
- Placement filters will not support the new API(CRD).

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
Add field `Type`, `ScoreResourceName` and `ScoreName` to `PrioritizerConfig`, to support specifying the customized score to sort clusters.

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
	// Type defines the type of the prioritizer.
	// Type is either "Build-in", "AddOnScore" or "", where "" is "Build-in" by default.
	// When type is "Build-in", you can use the build-in prioritizers.
	// When type is "AddOnScore", the prioritizers use score provided by AddOnPlacementScore to prioritize clusters, the
	// score source should be defined in ScoreResourceName and ScoreName.
	// +kubebuilder:default:=Build-in
	// +optional
	Type string `json:"type,omitempty"`

	// Name is the name of a prioritizer. Below are the valid build-in prioritizer names.
	// 1) Balance: balance the decisions among the clusters.
	// 2) Steady: ensure the existing decision is stabilized.
	// 3) ResourceAllocatableCPU & ResourceAllocatableMemory: sort clusters based on the allocatable.
	// +kubebuilder:validation:Required
	// +required
	Name string `json:"name"`

	// ScoreResourceName defines the resource name of the AddOnPlacementScore.
	// AddOnPlacementScore resource contains a bundle of scores. When type is defined as "AddOnScore",
	// you need to define the AddOnPlacementScore resource name which
	// could be used by the prioritizer to sort clusters.
	// +optional
	ScoreResourceName string `json:"scoreResourceName,omitempty"`

	// ScoreName defines the score name of the AddOnPlacementScore.
	// AddOnPlacementScore contains a list of score name and score value, ScoreName specify the score to be used by
	// the prioritizer.
	// +optional
	ScoreName string `json:"scoreName,omitempty"`

	// Weight defines the weight of the prioritizer. The value must be ranged in [0,10].
	// Each prioritizer will calculate an integer score of a cluster in the range of [-100, 100].
	// The final score of a cluster will be sum(weight * prioritizer_score).
	// A higher weight indicates that the prioritizer weights more in the cluster selection,
	// while 0 weight indicates that the prioritizer is disabled.
	// +kubebuilder:validation:Minimum:=-10
	// +kubebuilder:validation:Maximum:=10
	// +kubebuilder:default:=1
	// +optional
	Weight int32 `json:"weight,omitempty"`
}
```

### Let placement prioritizer support rating clusters with the customized score of the CR.
How to maintain the lifecycle of AddOnPlacementScore CRs is out of scope in this proposal. Here we suppose there already are AddOnPlacementScore CRs created in each managed cluster namespace.
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
      - name: Steady
        weight: 3
      - type: AddOnScore
        name: ResourceRatioCPU
        weight: 1
        scoreResourceName: default
        scoreName: cpuratio
```

- For "Build-in" type prioritizers, you don't need to specify the type explicitly. In the above example, the placement defines prioritizer "Steady" with weight 3.
- For "AddOnScore" type prioritizers, also need define scoreResourceName and scoreName. In above example, the placement defines an "AddOnScore" type prioritizer "ResourceRatioCPU" which will use score "cpuratio" provided by AddOnPlacementScore "default" to sort clusters, the weight is 1.

When this placement.yaml is created, the placement controller sorts the clusters with "Build-in" prioritizer "Steady" and "AddOnScore" prioritizer "ResourceRatioCPU". The build-in prioritizer scheduling behavior keeps the same before, the "AddOnScore" type prioritizer scheduling process is as below:

- First schedule

The prioritizer "ResourceRatioCPU" will get customized score "cpuratio" from "default" `AddOnPlacementScore`, and calculate a final score for each managed clusters.

- Reschedule

The scores inside `AddOnPlacementScore` are supposed to be updated frequently. Rather than watch the changes in `AddOnPlacementScore` and trigger reschedule, the scheduler framework would periodically (eg. every 5 minutes) reschedule the placement.

**What if no valid score is generated?**

What if some of the scores are invalid? For example, the AddOnPlacementScore CR is not created for some managed clusters, the score is missing in some CR, the score is expired. To handle the invalid cases, we add a PreScore() in the Prioritizer interface, to do some pre-check before Score().
```golang
// Prioritizer defines a prioritizer plugin that scores each cluster. The score is normalized
// as floating between 0 and 1.
type Prioritizer interface {
	Plugin

	// PreScore() do some preparation work before Score().
	PreScore(ctx context.Context, placement *clusterapiv1alpha1.Placement, clusters []*clusterapiv1.ManagedCluster) error

	// Score gives the score to a list of the clusters, it returns a map with the key as
	// the cluster name.
	Score(ctx context.Context, placement *clusterapiv1alpha1.Placement, clusters []*clusterapiv1.ManagedCluster) (map[string]int64, error)
}
```

With the PreSocre() added, the "AddOnScore" type prioritizer scheduling process is as below:

- First schedule
  - In PreScore(), the prioritizer checks if 80% of the CRs have a valid score. (80% is a reasonable threshold hardcode so far, might make it configurable in the future). 
    - If yes, the prioritizer will continue to score the clusters. 
    - If no, terminate this scheduling and wait for the next reschedule.
  - In Score(), the prioritizer will get customized score from `AddOnPlacementScore`, and calculate a final score for each managed cluster.

- Reschedule
  - In PreScore(), the prioritizer checks if 80% of the CRs have a valid score.
    - If yes, the prioritizer will continue to score the clusters. 
    - If no, remove the managed cluster which has invalid score from the placement decision.
  - In Score(), the prioritizer will get customized score from `AddOnPlacementScore`, and calculate a final score for each managed cluster.

Future work：there might be a cluster-scoped CRD to configure the threshold of each prioritizer. For example, the frequency to trigger a re-schedule, the threshold to treat the cluster's score as valid.

### How to maintain the lifecycle of the AddOnPlacementScore CRs? 
Even this is out of scope in this proposal, 2 options to maintain the resources:
- Can write a customized controller running on each managed cluster, to maintain the lifecycle (create/update/delete) of the CRs.
- Let the placement controller create/delete the CRs for each managed cluster, and customized score provider only needs to write an agent to update the scores.

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
2. User creates a new placement as below to sort clusters with "cpuratio" score provided by AddOnPlacementScore "default".
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
      - type: AddOnScore
        name: ResourceRatioCPU
        weight: 1
        scoreResourceName: default
        scoreName: cpuratio
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
      - type: AddOnScore
        name: DisasterRecovery
        weight: 1
        scoreResourceName: disasterrecovery
        scoreName: workload
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