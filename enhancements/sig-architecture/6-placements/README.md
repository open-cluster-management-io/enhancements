# New Placement APIs

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary

The proposed work would define new placement APIs to select a set of ManagedClusters from multiple ManagedClusterSets.

## Motivation

Since placementrule is used in multiple components (GRC/App/Observability). It is more appropriate to define it in API repo and have a generic component to consume it based on the current placementrule APIs in https://github.com/open-cluster-management/multicloud-operators-placementrule. 

The placement APIs will provide a definition on how to select a set of ManagedClusters in multiple ManagedClusterSets. And placement decisions will be generated according to the spec defined in the placement APIs.

Placement APIs can be used by a higher primitive and controller to decide which ManagedClusters the manifests should be placed.

### User Stories

#### Story 1: A cluster-admin on hub can place a policy or workload to all ManagedClusters in all ManagedClusterSets; The cluster-admin on hub can also place a policy or workload to any selected ManagedClusters in ManagedClusterSets.

#### Story 2: A normal user on hub can place a policy or a workload to a slice of ManagedClusters in one or multiple ManagedClusterSets that the user is authorized to access.

#### Story 3: A user can use predicate to select ManagedClusters
  - Select ManagedClusters based on a specified flavor - e.g. OpenShift
  - Select ManagedClusters in aws and gcp
  - Select ManagedClusters not in vsphere
  - Select ManagedClusters based on a specified flavor, like OpenShift, with a specific version
  - Select ManagedClusters labeled as prod (which is set by users).

#### Story 4: A user can define the number of ManagedClusters to be selected
  - Select 3 ManagedClusters based on a specified flavor
  - Select 5 ManagedClusters on aws

#### Story 5: The placement API should be scalable to support selecting hundreds of ManagedClusters from thousands of ManagedClusters.

#### Story 6: A user can select ManagedClusters across different regions.

#### Story 7: A controller is able to consume placements to select desired ManagedClusters.

#### Story 8: A user can specify tolerations and select ManagedClusters with particular taints.

#### Story 9: A user can fetch metrics data and figure out how the decision is made for a placement.

#### Story 10: A user can specify a time window for a placement rule.

### Goals

- Define a new placement APIs in API repo to select a set of ManagedClusters in multiple ManagedClusterSets;
- Have a component created with controllers to consume placements and generate placement decisions;
- Limit the access to ManagedClusters by leveraging ManagedClusterSets;
- Replace and deprecate the current placementrule APIs in https://github.com/open-cluster-management/multicloud-operators-placementrule
- The user stories 8-10 need further investigation and will be covered in separate enhancements when API version moves to beta;

## Proposal

### API Spec

#### Placement API
`Placement` is a namespace scoped API and represents a selection of ManagedClusters in one or multiple ManagedClusterSets. When a placement is created, a controller consuming the placement should generate one or multiple placement decisions based on the spec defined in placement.
```go
// Placement defines a rule to select a set of ManagedClusters from the ManagedClusterSets bound
// to the placement namespace.
//
// Here is how the placement policy combines with other selection methods to determine a matching
// list of ManagedClusters:
// 1) Kubernetes clusters are registered with hub as cluster-scoped ManagedClusters;
// 2) ManagedClusters are organized into cluster-scoped ManagedClusterSets;
// 3) ManagedClusterSets are bound to workload namespaces;
// 4) Namespace-scoped Placements specify a slice of ManagedClusterSets which select a working set
//    of potential ManagedClusters;
// 5) Then Placements subselect from that working set using label/claim selection.
//
// No ManagedCluster will be selected if there is no ManagedClusterSet bound to the placement
// namespace. User is able to bind a ManagedClusterSet to a namespace by creating a 
// ManagedClusterSetBinding in the namespace if they have an RBAC rule to CREATE on the virtual
// subresource of `managedclustersets/bind`.
//
// A slice of PlacementDecisions with label placement.open-cluster-management.io={placement name}
// will be created to represent the ManagedCluster selected by this placement.
// 
// If a ManagedCluster is selected and added into the PlacementDecisions, other components may apply
// workload on it; once it is removed from the PlacementDecisions, the workload applied on this
// ManagedCluster should be evicted accordingly.
type Placement struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// Spec defines the attributes of Placement.
	Spec PlacementSpec `json:"spec"`

	// Status represents the current status of the Placement
	// +optional
	Status PlacementStatus `json:"status,omitempty"`
}

// PlacementSpec defines the attributes of Placement.
// An empty PlacementSpec selects all ManagedClusters from the bound ManagedClusterSets to placement namespace.
type PlacementSpec struct {
	// Predicates represent a slice of predicates to select ManagedClusters. The predicates are ORed.
	// +optional
	Predicates []ClusterPredicate `json:"predicates,omitempty"`

	// Affinity represents the scheduling constraints
	// +optional
	Affinity *Affinity `json:"affinity,omitempty"`
}

// ClusterPredicate represents a predicate to select ManagedClusters.
type ClusterPredicate struct {
	// ClusterSets represent the ManagedClusterSets from which the ManagedClusters are selected.
	// If the slice is empty, ManagedClusters will be selected from the ManagedClusterSets bound to the placement
	// namespace, otherwise ManagedClusters will be selected from the intersection of this slice and the
	// ManagedClusterSets bound to the placement namespace.
	// +optional
	ClusterSets []string `json:"clusterSets,omitempty"`

	// NumberOfClusters represents the number of ManagedClusters to be selected which meet the
	// requirements in this predicate.
	// 1) If not specified, it indicates all matched ManagedClusters will be selected;
	// 2) Otherwise if the nubmer of matched ManagedClusters is larger than NumberOfClusters, a random subset
	//    with expected number of ManagedClusters will be selected;
	// 3) If the nubmer of matched ManagedClusters is equal to NumberOfClusters, all matched ManagedClusters
	//    will be selected;
	// 4) If the nubmer of matched ManagedClusters is less than NumberOfClusters, all matched ManagedClusters
	//    will be selected, and the status of condition `PlacementConditionSatisfied` will be set to false;
	// +optional
	NumberOfClusters *int32 `json:"numberOfClusters,omitempty"`

	// RequiredDuringSchedulingRequiredDuringExecution represents a scheduling strategy for a
	// cluster selector. If specified, the cluster selector should be matched at both scheduling
	// time (when the placement decisions are made for the first time) and execution time (After
	// one or multiple placement decisions are made). It means,
	// 1) Any ManagedCluster, which does not match the selector, should not be selected;
	// 2) If a selected ManagedCluster ceases to match the selector (e.g. due to an update) after the
	//    placement decisions are made, it will be eventually removed from the placement decisions;
	// 3) If a ManagedCluster starts to match the selector after the placement decisions are made, it
	//    will either be selected or at least has chance to be selected (when NumberOfClusters is
	//    specified);
	//
	// Other scheduling strategies not yet implemented includes
	// RequiredDuringSchedulingIgnoredDuringExecution and PreferredDuringSchedulingIgnoredDuringExecution
	// +optional
	RequiredDuringSchedulingRequiredDuringExecution *ClusterSelector `json:"requiredDuringSchedulingRequiredDuringExecution,omitempty"`
}

// ClusterSelector represents the AND of the containing selectors
type ClusterSelector struct {
	// LabelSelector represents a selector of ManagedClusters by label
	// +optional
	LabelSelector metav1.LabelSelector `json:"labelSelector,omitempty"`

	// ClaimSelector represents a selector of ManagedClusters by clusterClaims in status
	// +optional
	ClaimSelector ClusterClaimSelector `json:"claimSelector,omitempty"`
}

// A clusterclaim selector is a claim query over a set of ManagedClusters. An empty cluster claim
// selector matches all objects. A null cluster claim selector matches no objects.
type ClusterClaimSelector struct {
	// matchExpressions is a list of cluster claim selector requirements. The requirements are ANDed.
	// +optional
	MatchExpressions []metav1.LabelSelectorRequirement `json:"matchExpressions,omitempty"`
}

// Affinity is a group of affinity scheduling rules.
type Affinity struct {
	// ClusterAntiAffinity describes cluster affinity scheduling rules for the placement.
	// +optional
	ClusterAntiAffinity *ClusterAntiAffinity `json:"clusterAntiAffinity,omitempty"`
}

// ClusterAntiAffinity is a group of cluster affinity scheduling rules.
type ClusterAntiAffinity struct {
	// RequiredDuringSchedulingRequiredDuringExecution is a scheduling strategies for
	// ClusterAntiAffinity. The anti-affinity requirements specified by this field should be met
	// at both scheduling time (when the placement decisions are made for the first time) and
	// execution time (After one or multiple placement decisions are made). It means,
	// 1) Any ManagedCluster, which does not meet the requirements, should not be selected;
	// 2) If a selected ManagedCluster ceases to meet the requirements (e.g. due to an update) after the
	//    placement decisions are made, it will be eventually removed from the placement decisions;
	// 3) If a ManagedCluster starts to meet the requirements after the placement decisions are made, it
	//    will either be selected or at least has chance to be selected;
	// When there are multiple elements, the lists of ManagedClusters corresponding to each
	// ClusterAffinityTerm are intersected, i.e. all terms must be satisfied.
	//
	// Other scheduling strategies not yet implemented includes
  // RequiredDuringSchedulingIgnoredDuringExecution and PreferredDuringSchedulingIgnoredDuringExecution
	// +optional
	RequiredDuringSchedulingRequiredDuringExecution []ClusterAffinityTerm `json:"requiredDuringSchedulingRequiredDuringExecution,omitempty"`
}

// ClusterAffinityTerm defines a constraint on a set of ManagedClusters. Any ManagedCluster in this set
// should be co-located (affinity) or not co-located (anti-affinity) with each other, where co-located
// is defined as the value of the label/claim with name <topologyKey> on ManagedCluster is the same.
type ClusterAffinityTerm struct {
	// TopologyKey is either a label key or a cluster claim name of ManagedClusters
	// +required
	TopologyKey string `json:"topologyKey"`

	// TopologyKeyType indicates the type of TopologyKey. It could be Label or Claim.
	// +required
	TopologyKeyType string `json:"topologyKeyType"`
}

const (
	TopologyKeyTypeLabel string = "Label"
	TopologyKeyTypeClaim string = "Claim"
)

type PlacementStatus struct {
	// NumberOfSelectedClusters represents the number of selected ManagedClusters
	NumberOfSelectedClusters int32 `json:"numberOfSelectedClusters"`

	// Conditions contains the different condition statuses for this Placement.
	Conditions []metav1.Condition `json:"conditions"`
}

const (
	// PlacementConditionSatisfied means Placement is satisfied.
	// A placement is satisfied only if all predicates are satisfied. And a predicate
	// is not satisfied if
	// 1) NumberOfClusters is specified;
	// 2) And the number of matched ManagedClusters for this predicate is less than NumberOfClusters;
	PlacementConditionSatisfied string = "PlacementSatisfied"
)
```

#### PlacementDecision API
`PlacementDecision` represents a slice of clusters selected by a placement. A `Placement` can link to multiple `PlacementDecisions`, and uses a label selector for reference.
```go
// PlacementDecision indicates a decision from a placement
// PlacementDecision should has a label placement.open-cluster-management.io={placement name}
// to reference a certain placement.
type PlacementDecision struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// Decisions is a slice of decisions according to a placement
	// The number of decisions should not be larger than 100
	Decisions []ClusterDecision `json:"decisions"`
}

// ClusterDecision represents a decision from a placement
type ClusterDecision struct {
	// ClusterName is the name of the ManagedCluster
	// +required
	ClusterName string `json:"clusterName"`

	// Reason represents the reason why the ManagedCluster is selected.
	// +required
	Reason string `json:"reason"`
}
```

### Examples

#### 1. Select ManagedClusters with cluster names

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: openshift-clusters-with-specific-names
  namespace: ns1
spec:
  predicates:
    - requiredDuringSchedulingRequiredDuringExecution:
        claimSelector:
          matchExpressions:
            - key: id.k8s.io
              operator: In
              values:
                - cluster1
                - cluster2
```

#### 2. Select 3 ManagedClusters from a specific ManagedClusterSet.

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: clusters-from-specific-clusterset
  namespace: ns1
spec:
  predicates:
    - clusterSets:
      - clusterset1
      numberOfClusters: 3
```
**Note:** No ManagedCluster will be selected if the ManagedClusterSet is not bound to the placement namespace.

#### 3. Select ManagedClusters based on a specific flavor with specific versions

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: openshift-clusters-with-specific-version
  namespace: ns1
spec:
  predicates:
    - requiredDuringSchedulingRequiredDuringExecution:
        labelSelector:
          matchLabels:
            vendor: OpenShift
        claimSelector:
          matchExpressions:
            - key: version.openshift.io
              operator: In
              values:
                - 4.5.8
                - 4.5.9
```

#### 4. Select 2 AWS ManagedClusters and 1 GCE ManagedCluster

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: clusters-from-aws-and-gce
  namespace: ns1
spec:
  predicates:
    - numberOfClusters: 2
      requiredDuringSchedulingRequiredDuringExecution:
        claimSelector:
          matchExpressions:
            - key: platform.open-cluster-management.io
              operator: In
              values:
                - AWS
    - numberOfClusters: 1
      requiredDuringSchedulingRequiredDuringExecution:
        claimSelector:
          matchExpressions:
            - key: platform.open-cluster-management.io
              operator: In
              values:
                - GCE
```

#### 5. Select ManagedClusters across different regions

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: clusters-across-regions
  namespace: ns1
spec:
  affinity:
    clusterAntiAffinity:
      requiredDuringSchedulingRequiredDuringExecution:
        - topologyKey: region.open-cluster-management.io
          topologyKeyType: Claim
```

## Design Details

### User Scenario
1. Cluster admin imports some Kubernetes clusters to hub as ManagedClusters;
2. Cluster admin organizes those ManagedClusters with ManagedClusterSets;
3. Cluster admin grant a normal user the access to some ManagedClusterSets;
4. The user binds some authorized ManagedClusterSets to their working namespace on hub cluster by creating ManagedClusterSetBindings in this namespace. An example looks like,
    ```yaml
    apiVersion: cluster.open-cluster-management.io/v1alpha1
    kind: ManagedClusterSetBinding
    metadata:
      name: clusterset1
      namespace: ns1
    spec:
      clusterSet: clusterset1
    ```

5. The user creates a placement in the working namespace;
    ```yaml
    apiVersion: cluster.open-cluster-management.io/v1alpha1
    kind: Placement
    metadata:
      name: placement1
      namespace: ns1
    spec:
      predicates:
        - requiredDuringSchedulingRequiredDuringExecution:
            labelSelector:
              matchLabels:
                vendor: OpenShift
    ```

6. Placement Controller generates placementdecisions in the placement namespace according to the ManagedClusterSetBindings in this namespace and the spec of the placement.
    ```yaml
    apiVersion: cluster.open-cluster-management.io/v1alpha1
    kind: PlacementDecision
    metadata:
      labels:
        placement.open-cluster-management.io: placement1
      name: placement1-decision-1
      namespace: ns1
    spec:
      decisions:
        - clusterName: cluster1
        - clusterName: cluster2
        - clusterName: cluster3
    ```

Limitations:
- A ManagedClusterSetBinding binds only one ManagedClusterSet to a namespace. It becomes worse if a user, like admin, wants to bind all ManagedClusterSets to a namespace, because,
  - The number of ManagedClusterSets might be very large;
  - The new added ManagedClusterSets cannot be bound to the namespace automatically;

### Workflows

#### Workflow 1: Placement Reconciliation
1. If the placement has old placementdecisions, fetch them all and get a slice of old matched ManagedClusters;
2. Evaluate each cluster predicates (see Workflow 2 below for more details) and get the union of all predicates result;
3. Enforce the affinity constrains on the union and remove ManagedClusters if necessary;
4. If the placement has no old placementdicisions yet, create placementdicisions;
5. Otherwise, update the existing placementdicisions, and add/remove placementdicisions if necessary.

#### Workflow 2: Evaluate a cluster predicate
1. If `ClusterSets` is not empty, get ManagedClusters belongs to intersection of `ClusterSets` and the ManagedClusterSets bound to the placement namespace;
2. Otherwise get ManagedClusters from the ManagedClusterSets bound to the placement namespace;
3. If `RequiredDuringSchedulingRequiredDuringExecution` is specified, filter ManagedClusters with label/claim selectors;
4. If `NumberOfClusters` is not specified, return all matched ManagedClusters;
5. Otherwise, get the intersection of old matched ManagedClusters and new matched ManagedClusters;
6. If `NumberOfClusters` is equal to the size of the intersection, return ManagedClusters in the intersection;
7. If `NumberOfClusters` is less than the size of the intersection, select random ManagedClusters from the intersection and return;
8. Otherwise, return ManagedClusters in the intersection with some random additional ManagedClusters from the remaining of the new matched ManagedClusters;

**Note:** The old matched ManagedClusters are preferred during the predicate evaluation, which makes the placement decisions as stable as possible.

### An example of consuming controller

Assuming a controller, `WorkloadController`, applies workload to ManagedClusters selected with a placement. The code snipet below shows the logic of reconciliation.

```go
func (c *WorkloadController) sync(workload []runtime.RawExtension, placement *Placement) error {
	//get the current cluster selection from placement decisions
	newClusterSelection, err := c.getClustersInPlacementDecisions(placement)
	if err != nil {
		return err
	}

	//get the old cluster selection from the workload which have been applied
	oldClusterSelection, err := c.getClustersWithWorkloadApplied(placement.Namespace, placement.Name)
	if err != nil {
		return err
	}

	// find out the clusters which are not selected before, and apply workload on them
	toBeApplied := newClusterSelection.Difference(oldClusterSelection)
	err = c.applyWorkload(workload, toBeApplied.List())
	if err != nil {
		return err
	}

	// find out the clusters which are no longer selected and remove workload from them
	toBeRemoved := oldClusterSelection.Difference(newClusterSelection)
	return c.removeWorkload(placement.Namespace, placement.Name, toBeRemoved.List())
}
```

### Open Questions

1. The API group of the new placement APIs could be `cluster.open-cluster-management.io` or `scheduling.open-cluster-management.io`. Which one is better?

### Test Plan

- Unit tests will cover the functionality of the controllers.
- Unit tests will cover the new APIs.
- Integration tests will cover all user stories.
- And e2e tests will cover the following cases.
  - Create/update/delete a placement;
  - ManagedCluster labels/claims changes;
  - A ManagedClusterSet is bound/unbound to/from the placement namespace;
  - A ManagedCluster is added/removed to/from a bound ManagedClusterSet;

### Graduation Criteria

#### Alpha
1. The new APIs is reviewed and accepted;
2. Implementation is completed to support the functionalities;
3. Develop test cases to demonstrate that the above user stories work correctly;

#### Beta
1. The new APIs is able to cover all the capabilities of the current placementrule APIs;
2. A tool is provided to convert the objects of the current placementrule APIs to the objects of the new placement APIs.
3. At lease one component uses the new APIs;
4. Support selecting ManagedClusters belong to no ManagedClusterSet;
5. Support tolerations/taints (user story 8);
6. Create metrics data when creating/updating placement decisions (user story 9);
7. Support time window (user story 10);

#### GA
1. Pass the performance/scalability testing;
2. Migrate all components which has dependency on the current placementrule APIs to the new placement APIs;
3. Deprecate the current placementrule APIs, and it will still exist in the next 3 releases.

### Upgrade Strategy

#### Placement Manager
Placement Manager is a new component and contains controllers to implement the new placement APIs. 
1. A new repo is required for `placement-manager`;
2. Change on `ClusterManager` CRD is required to include information for `placement-manager`;
3. Change on `cluster-manager` is required to
    - Install the new placement CRD on hub cluster;
    - Deploy `placement-manager`;

#### API Migration
1. An offline migration is required for those components which use the current placementrule API to
    - Convert the objects of the current placementrule APIs to objects of the new placement APIs by a tool (which will be in place once the new placement APIs is in beta);
    - Bind necessary ManagedClusterSets to the placement namespaces.
2. The new placement APIs will co-exists with the current placementrule APIs in at least 3 releases;

### Version Skew Strategy

## Alternatives

### 1. Setting ManagedClusterSets explicitly in a `Placement` instead of binding ManagedClusterSets to the placement namespace

1. A normal user creates a placement with explicit references to ManagedClusterSets;
    ```yaml
    apiVersion: cluster.open-cluster-management.io/v1alpha1
    kind: Placement
    metadata:
      name: placement1
      namespace: ns1
    spec:
      clusterSets:
        - clusterset1
        - clusterset2
      predicates:
        - requiredDuringSchedulingRequiredDuringExecution:
            labelSelector:
              matchLabels:
                vendor: OpenShift
    ```
2. A validating webhook rejects the create/update/patch requests of placements if the user has no permission on the referenced ManagedClusterSets;
3. Placement Controller generates placementdecisions in the same namespace according to the spec of the placement.

#### Pro
- No need to bind ManagedClusterSets to the placement namespace;

#### Cons
- The webhook can only handle create/update/patch requests of placements. If we want to use ManagedClusterSets to limit the access scope of ManagedClusters for other resource, we have to either update this webhook or add a new one;
- It will not be easy to get the ManagedClusterSets for the working namespace;
