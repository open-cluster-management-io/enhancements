# placement extensible scheduling

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/website/)

## Summary

The proposed work provides an API to represent a scalar value of the managed cluster, to support placement extensible scheduling.

## Motivation

When implementing placement resource based scheduling, we find in some cases, prioritizer needs extra data (more than the defualt value provided by `ManagedCluster`) to calculate the score of the managed cluster. For example, there is requirement to schedule based on resource monitoring data from the cluster.

So we want a more extensible way to support scheduling based on customized value.

### Goals
- Design a new API(CRD) to contain a scalar value for each managed cluster.
- Let placement prioritizer maintain the lifecycle (creating/deleting) of the CR.
- Let placement prioritizer support rating clusters with the scalar value of the CR.

### Non-Goals
- How a customized agent update the scalar value into the CR.
- Placement filter will not support the new API(CRD).

## Proposal

### User Stories

#### Story 1: Users could use the data pushed from each managed cluster to select clusters.
  - On each managed cluster, there is a customized agent monitoring the resource usage (eg. CPU ratio) of the cluster. It will calculate a scalar value and push the result to hub.
  - As an end user, I can configure placement yaml to use this scalar value to select clusters.
  
#### Story 2: Users could use the metrics collected on hub to select clusters.
  - On the hub, there is a customized agent to get metrics (eg. cluster allocatable memory) from Thanos. It will generate a scalar value for each cluster.
  - As an end user, I can configure placement yaml to use this scalar value to select clusters.

#### Story 3: Disaster recovery workload could be automatically switched to an avaliable cluster.
  - A user has two clusters, a primary and backup cluster on which workload storage is configured so storage are synced from primary cluster to backup.
  - As an end user, I want to deploy workloads on the primary cluster first. And when primary cluster is down, the workload should be automatically switched to the backup cluster.

#### Story 4: Users could use the data provided by third party controllers to select clusters.
  - A third party controller can comprehensively evaluate cluster on latency, region, iops etc. and rate cluster according to the SLA the cluster can provide.
  - As an end user, I can configure placement yaml to use this scalar value to select clusters.

### Risks and Mitigation
N/A

## Design Details

### ManagedClusterScalar API
ManagedClusterScalar is the new API to add, to represents a scalable value (aka score) of one managed cluster.
```go
// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +kubebuilder:resource:scope="Namespaced"
// +kubebuilder:subresource:status

// ManagedClusterScalar represents a scalable value (aka score) of one managed cluster.
// Each ManagedClusterScalar only represents the scalar value of one prioritizer.
// ManagedClusterScalar is a namesapce scoped resource. The namespace of the resource is the cluster namespace.
type ManagedClusterScalar struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// Spec defines the attributes of the ManagedClusterScalar.
	// +kubebuilder:validation:Required
	// +required
	Spec ManagedClusterScalarSpec `json:"spec"`

	// Status represents the status of the ManagedClusterScalar.
	// +optional
	Status ManagedClusterScalarStatus `json:"status,omitempty"`
}

// ManagedClusterScalarSpec defines the attributes of the ManagedClusterScalar.
type ManagedClusterScalarSpec struct {
	// PrioritizerName will be the prioritizer name used in placement.
	// +kubebuilder:validation:Required
	// +required
	PrioritizerName string `json:"prioritizerName,omitempty"`
}

//ManagedClusterScalarStatus represents the current status of ManagedClusterScalar.
type ManagedClusterScalarStatus struct {
	// Conditions contain the different condition statuses for this managed cluster scalar.
	// +optional
	Conditions []metav1.Condition `json:"conditions"`

	// Scalar contains a scalable value of this managed cluster.
	// +optional
	Scalar int64 `json:"scalar,omitempty"`

	// ValidUntil defines the time this scalar is valid.
	// After this time, the scalar is considered to be invalid by placement. nil means never expire.
	// The controller ownning this resource should keep the scalar up-to-date.
	// +optional
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Type=string
	// +kubebuilder:validation:Format=date-time
	ValidUntil *metav1.Time `json:"validUntil" protobuf:"bytes,4,opt,name=lastTransitionTime"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// ManagedClusterScalarList is a collection of managed cluster scalar.
type ManagedClusterScalarList struct {
	metav1.TypeMeta `json:",inline"`
	// Standard list metadata.
	// More info: https://git.k8s.io/community/contributors/devel/api-conventions.md#types-kinds
	// +optional
	metav1.ListMeta `json:"metadata,omitempty"`

	// Items is a list of managed clusters
	Items []ManagedClusterScalar `json:"items"`
}
```

### Let placement prioritizer maintain the lifecycle of the CR.
- Create

The creation of ManagedClusterScalar CR is handled by each prioritizer plugin on demand. As every update in ManagedClusterScalar CR will introduce an api call, that will bring potential performance presure to api server. So we only want to create ManagedClusterScalar CRs for enabled prioritizer in filterd cluster namespace.

The placement schedule framework will filter avaliable clusters as first step.
Then the framework will call each prioritizer one by one. If the prioritizer is enable in placement yaml, it will create a corresponding ManagedClusterScalar CR in filtered cluster namespace as below. 
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterScalar
metadata:
  name: {PrioritizerName}
  namespace: cluster1
spec:
  prioritizerName: {PrioritizerNameInPlacement}
```
The name is the prioritizer name.
The namespace is the filtered cluster namespace.
The spec.prioritizerName is the prioritizer name used in placement yaml. 

The code change includes add PreScore() in Prioritizer interface, in which to create ManagedClusterScalar CR before Score().
```go
// Prioritizer defines a prioritizer plugin that score each cluster. The score is normalized
// as a floating betwween 0 and 1.
type Prioritizer interface {
	Plugin

	// PreScore() do some prepare work before Score().
	// For example, creating ManagedClusterScalar CR before Score().
	PreScore(ctx context.Context, placement *clusterapiv1alpha1.Placement, clusters []*clusterapiv1.ManagedCluster) error

	// Score gives the score to a list of the clusters, it returns a map with the key as
	// the cluster name.
	Score(ctx context.Context, placement *clusterapiv1alpha1.Placement, clusters []*clusterapiv1.ManagedCluster) (map[string]int64, error)
}
```

- Delete

The deletion of ManagedClusterScalar CR is handled by the framework (out of all the prioritizer plugins). The ManagedClusterScalar CR is created by each prioritizer plugin on demand, and the CR should be shared by some placements cross namespaces. When the ManagedClusterScalar CR is not used by any placement, need to clean it to avoid unnecessary api call.

There will be a GC() funcion in the framework, being called every 10 minutes. Go through all the Placment CRs and ManagedClusterScalar CRs, and clean not used ManagedClusterScalar CRs.

- Update

ManagedClusterScalar CR's status is updated by customized agent or third party controllers. Out of scope and won't be covered in this proposal.

### Let placement prioritizer support rating clusters with the scalar value of the CR.
Once ManagedClusterScalar CR is created in PreScore(), a customized agent or third party controllers should watch the change and update the CR status as below.
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterScalar
metadata:
  name: {PrioritizerName}
  namespace: cluster1
spec:
  prioritizerName: {PrioritizerNameInPlacement}
status:
  conditions:
  - lastTransitionTime: "2021-10-28T08:31:39Z"
    message: ManagedClusterScalar updated successfully
    reason: ManagedClusterScalarUpdated
    status: "True"
    type: ManagedClusterScalarUpdated
  validUntil: "2021-10-29T18:31:39Z"
  score: 74
```
Usually each ManagedClusterScalar CR will be created in more than one cluster namespace, different clusters update the status separately. So the prioritizer plugin which depends on ManagedClusterScalar should follow below logic to get avaliable score and sort all the clusters.
- Prescore

In prescore stage, prioritizer plugin should create the CR before use it (have talked about this in previous section).
The plugin should also check if 80% of the CRs have a valid score (80% is a reasonable threshold hardcode so far, might make it configurable in the future). If yes, the plugin will continue to score the clusters. If not, terminate this schduling and let requeue to trigger a reschedule.

- Score

In score stage, prioritizer plugin get valid score from ManagedClusterScalar CR. If no score updated in ManagedClusterScalar CR status, treat it as 0.
After getting all the cluster score, the plugin should formalized them from -100 to 100. The maximal will be 100 and minimal is -100, other score distribute in proportion.

- Re-Schedule

Rather than watch ManagedClusterScalar and trigger Score(), that will cause frequently reschedule. The scheduler framework should periodically (eg. every 5 minutes) reschedule the placement.

Future work：there might be a cluster-scoped CRD to configure the threshold of each prioritizer. For example, the frequency to trigger a re-scheudle, the threshold in prescore to treat score as valid.

### Examples

#### An agent update ResourceRatioCPU score to API, and placement prioritizer plugin score based on it. (Use Story 1&2)
1. User create a new placement as below
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
      - name: CustomizeResourceRatioCPU
```
2. Placement controller watch the placement changes and prioritizer plugin generate a new ManagedClusterScalar CR `resourceratiocpu` to trigger customized data collection.
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterScalar
metadata:
  name: resourceratiocpu
  namespace: cluster1
spec:
  prioritizerName: CustomizeResourceRatioCPU
```
3. The agent on managed cluster watch the new ManagedClusterScalar CR and update score into it. The agent refresh the score periodically before expire time.
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterScalar
metadata:
  name: resourceratiocpu
  namespace: cluster1
spec:
  prioritizerName: CustomizeResourceRatioCPU
status:
  conditions:
  - lastTransitionTime: "2021-10-28T08:31:39Z"
    message: ManagedClusterScalar updated successfully
    reason: ManagedClusterScalarUpdated
    status: "True"
    type: ManagedClusterScalarUpdated
  validUntil: "2021-10-29T18:31:39Z"
  score: 74
```
4. Placement get score from ManagedClusterScalar CRs and sort clusters.
Placement prioritizer plugin will wait for all the ManagedClusterScalar CRs have score, then sort clusters and put the result in PlacementDecision. Placement will reschedule every 5 minutes to ensure the decision is up-to-date.

#### Disaster recovery workload could be automatically switched to an avaliable cluster. (Use Story 3)
1. A user has 2 clusters, one is primary and one is backup cluster. To deploy workloads on the primary cluster, define the placement as below.  
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
      - name: Customized-DisasterRecovery 
```
2. Placement controller watch the placement changes and prioritizer plugin generate a new ManagedClusterScalar CR `disasterrecovery` to trigger customized data collection.
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterScalar
metadata:
  name: disasterrecovery
  namespace: primary
spec:
  prioritizerName: Customized-DisasterRecovery
```
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterScalar
metadata:
  name: disasterrecovery
  namespace: backup
spec:
  prioritizerName: CustomizeDisasterRecovery
```
3. The customized controller watch the new ManagedClusterScalar CR and update score into it. The primary has score 100 and backup has score 0, and validUntil is nil (the score never expire).
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterScalar
metadata:
  name: disasterrecovery
  namespace: primary
spec:
  prioritizerName: CustomizeDisasterRecovery
status:
  conditions:
  - lastTransitionTime: "2021-10-28T08:31:39Z"
    message: ManagedClusterScalar updated successfully
    reason: ManagedClusterScalarUpdated
    status: "True"
    type: ManagedClusterScalarUpdated
  score: 100
```
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ManagedClusterScalar
metadata:
  name: disasterrecovery
  namespace: backup
spec:
  prioritizerName: CustomizeDisasterRecovery
status:
  conditions:
  - lastTransitionTime: "2021-10-28T08:31:39Z"
    message: ManagedClusterScalar updated successfully
    reason: ManagedClusterScalarUpdated
    status: "True"
    type: ManagedClusterScalarUpdated
  score: 0
```
4. Placement get score from ManagedClusterScalar CRs and sort clusters.
As cluster primary has a higher score, the Placement decision will chose and workload will be running on the primary. 
5. When the primary cluster is unavailable (a taint is added). The placement will filter out the primary because of taint, and backup cluster will be chosen. Then disaster recovery workload will be running on the backup cluster automatically.

### Test Plan

- Unit tests cover placement creating/reading/cleaning ManagedClusterScalar.
- Unit tests cover placement desicion when ManagedClusterScalar status changes/expire/empty.
- Integration tests cover user story 1-4;

### Graduation Criteria
#### Alpha
1. The new APIs is reviewed and accepted;
2. Implementation is completed to support the functionalities;
3. Develop test cases to demonstrate that the above user stories work correctly;

#### Beta
1. Need to revisit the API shape before upgrade to beta based on user’s feedback.

### Upgrade / Downgrade Strategy
N/A

### Version Skew Strategy
N/A

## Drawbacks
1. Is one score enough? any other missed use cases? For example, spreading depends on previous choice.

## Alternative
### Support user-defined scheduler
As and end user, can specify the user-defined scheduler name in placement, and let a customized scheduler to generate placement decision. In this way, extensible scheduling can also be achieved.

Pros: 
  - More flexible, the customized scheduler can schedule based on resource usage, SLA, metrics, etc.

Cons:
  - More effort to write a customized scheduler.
  - The customized scheduler needs to understand all the spec of placement api, the changes of the api.