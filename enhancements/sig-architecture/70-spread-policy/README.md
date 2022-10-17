Spread Policy across Failure-domains in Placement APIs
---

## Summary
This document proposes relevant API and code changes to apply for the OCM project [Topology-aware scheduling in cloud-native multi-cluster/multi-cloud environments](https://summer-ospp.ac.cn/#/org/prodetail/221390086) in OSPP 2022.
This work is based on [previous community discussions](https://github.com/open-cluster-management-io/community/issues/49#issuecomment-857743027) on spread policy with several modifications.

## Background
Modern cloud providers typically provide K8s clusters in different datacenters and cities.
Besides, as a user, a company may want to use clusters from different providers plus some self-hosted clusters.
Here, three failure domains appeared (i.e., zone, region, and provider).
To achieve fault-tolerance and better resource utilization, it is natural that users want to distribute workloads among these failure domains using a spread policy.

K8s itself supports spread policy (by [PodTopologySpread](https://web.archive.org/web/20200507113854/https://kubernetes.io/blog/2020/05/introducing-podtopologyspread/)).
The support on other multicluster management systems [like Karmada](https://github.com/karmada-io/karmada/pull/1646) is on the way. Hence, it seems necessary for OCM to support a spread policy.

## Motivation
In this section, some use cases will be presented to motivate potential features we need for spread policy.
Generally, by using the spread policy, we want to spread the workload across different failure domains.

Through this document, the following failure domain topology is used for example.

<table align="center">
<thead>
  <tr>
    <th>Provider</th>
    <th>Region</th>
    <th>Zone</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td rowspan="3">aws</td>
    <td rowspan="2">us-east</td>
    <td>us-east-1a</td>
  </tr>
  <tr>
    <td>us-east-1b</td>
  </tr>
  <tr>
    <td>us-west</td>
    <td>us-west-1a</td>
  </tr>
  <tr>
    <td>alibaba<br></td>
    <td>cn-hangzhou</td>
    <td>cn-hangzhou-e</td>
  </tr>
</tbody>
</table>

### Use Case 1: User-defined Topology of Failure Domains

Different companies may manage clusters with different topology structures of failure domains.
Hence, we use labels or claims to denote the failure domain where a ManagedCluster belongs.
The label key is named topology key.
Notably, all examples will be given with labels in this document.

**Example.** A company with only self-hosted clusters may only have “region” and “zone” labels for the clusters, while a company that also uses the public cloud has an extra “provider”.

### Use Case 2: Even Spread

The most intuitional way to spread the workload within a topology key is to break the workload into several identical parts and assign each failure domain the same number of parts.
In the model of OCM Placement, this means that we need to select the same numbers of clusters for use from the different failure domains within the same topology key.

**Example.** We have a backend service to deploy in the clusters located in the region us-east-1.
We want to spread the workload across the zones to achieve disaster tolerance, and we need four clusters in total.
The final placement decision should contain two from us-east-1a and other two from us-east-1b.

### Use Case 3: Max Skew

When the even spread is unsatisfiable to a certain degree, we may want the scheduler not to schedule the workload.

**Example.** We want a even spread across the zones in us-east-1 (i.e., us-east-1a and us-east-1b) and four clusters in total.
If us-east-1a can provide three clusters and us-east-1b can only provide one cluster, then the even spread cannot be satisfied strictly.

Following K8s's PodTopologySpread, we define the skew of a topoloy to be selected_clusters_in_current_topology - **min** selected_clusters_in_a_topology.
For the above example, max skew is 3 - 1 = 2.
If we want the max skew to be smaller than 2, then the scheduler should not schedule the workload.
The max skew constraint is a hard constraint if specified. 

### Use Case 5: Joint Spread

To express hierarchical topology structure, clusters are labeled with multiple topology keys (i.e., region, zone, etc.), we want them to work jointly to determine the spread decision.

**Example.** We want four clusters from aws to put our backend service.
Further, we want to spread the workload first across regions and then across zones.
Assuming all zones here can provide enough clusters.
Then, the scheduling result should contain one cluster from us-east-1a, one cluster from us-east-1b, and other two clusters from us-west-1a.

### Use Case 6: Spread with Ratio (Out of Scope)

Sometimes, we may need to spread the workload with a specified ratio.
Let us consider a scenario where most of the access to the workload is from the east coast of the US, even spread between us-east-1 and us-west-1 would be inappropriate.
Instead, we want to place more workload on us-east-1 than us-west-1.
Besides, we want to control such preferences by user-specified configurations.

**Example.** We want six clusters to deploy our backend service.
Moreover, we want 2/3 of them to be located in us-east-1 and 1/3 to be located in us-west-1 since there are more clients from the east coast of the US.

## Goal
Generally, we have two goals for the spread policy:
1. Cover the full functionality in the Motivation section;
2. Implement a high-performance and scalable mechanism that works for thousands of clusters efficiently.

## Proposal

### API Spec
```go
type PlacementSpec struct {
	// ...

	// SpreadPolicy defines how placement decisions should be distributed among a
	// set of clusters.
	// +optional
	SpreadPolicy SpreadPolicy `json:"spreadPolicy,omitempty"`
}

// SpreadPolicy defines how placement decisions should be spreaded among the
// clusters.
type SpreadPolicy struct {
	// SpreadConstraints defines how placement decision should be distributed among a
	// set of clusters.
	// +optional
	// +kubebuilder:validation:MaxItems=8
	SpreadConstraints []SpreadConstraintsTerm `json:"spreadConstraints,omitempty"`
}

// SpreadConstraintsTerm defines a terminology to spread placement decisions
type SpreadConstraintsTerm struct {
	// TopologyKey is either a label key or a cluster claim name of ManagedClusters
	// +required
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Pattern=`^([a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*/)?([A-Za-z0-9][-A-Za-z0-9_.]*)?[A-Za-z0-9]$`
	// +kubebuilder:validation:MaxLength=316
	TopologyKey string `json:"topologyKey"`

	// TopologyKeyType indicates the type of TopologyKey. It could be Label or Claim.
	// +required
	// +kubebuilder:validation:Required
	TopologyKeyType TopologyKeyType `json:"topologyKeyType"`

	// MaxSkew describes the degree to which the workload may be unevenly distributed.
	// If present, the scheduler will not schedule if the MaxSkew cannot be satisfied.
	// The minimum possible value is 1.
	// +optional
	// +kubebuilder:validation:Minimum=1
	MaxSkew *int32 `json:"maxSkew,omitempty"`

	// Weight indicates the weight of the SpreadPolicyTerm. It is required only when multiple
	// terms are used. To spread the workload, the scheduler takes the even constraints with
	// higher weight into prior consideration.
	// +optional
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:default=1
	Weight int32 `json:"weight,omitempty"`
}

// +kubebuilder:validation:Enum=Label;Claim
type TopologyKeyType string

const (
	// Valid TopologyKeyType value is Claim, Label.
	TopologyKeyTypeClaim TopologyKeyType = "Claim"
	TopologyKeyTypeLabel TopologyKeyType = "Label"
)
```

### API Example

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: demo-placement
spec:
  # Optional. If numberOfClusters is not specified, it means to
  # select all clusters which meet the placement requirements.
  numberOfClusters: 4
  # Prediates work before the spread constraints to filter
  # out some undesired topologies.
  predicates:
  - # ...
  spreadPolicy:
    # spreadConstraints contain two terms, which work
    # independently and jointly.
    spreadConstraints:
    # When multiple terms are used, the user should specify their weights.
    # The scheduler will consider to first do the spread according to the
    # terms with higher weight.
    # In this example, the scheduler will first spread the workload
    # by providers.
    # Within each provider, the scheduler then spread the workload by
    # regions. 
    - topologyKey: provider
      topologyKeyType: Label
      maxSkew: 2
      weight: 20
      # maxSkew is omitted as this term is a soft constraint
    - topologyKey: region
      topologyKeyType: Label
      weight: 10
  # The spread policy has a scoreCoordinate in prioritizerPolicy.
  # In Additive mode, the weight of SpreadPolicy is 0 by default.
  # In Exact mode, the weight of SpreadPolicy should be set by users.
  prioritizerPolicy:
    mode: Additive
    configurations:
      - scoreCoordinate:
          type: BuiltIn
          builtIn: Spread
        weight: 2

```

### Implementation
The current scheduling framework contains two types of plugins, i.e., `Filter`s and `Prioritizer`s. 
`Filter`s are invoked *before* `Prioritizer`s to handle hard constrains, filtering the clusters. 
Then, the `Prioritizer`s score the clusters.
To implement `maxSkew`, we need to first score the clusters (i.e., do the spread), then ensure the `maxSkew` is not exceeded by some filter-like operations for hard constraints.
This cannot be implemented in the current framework.
Hence, a new plugin type, `Selector`, is proposed to implement hard constrains *after* `Prioritizer`s.
The `Selector` interface is showed below.

```go
// Selector defines a selector plugin that selects clusters after scoring them.
type Selector interface {
	Plugin

	// Select returns the selected clusters.
	Select(ctx context.Context, placement *clusterapiv1beta1.Placement, scores map[string]int64, clusters []*clusterapiv1.ManagedCluster) (PluginSelectResult, *framework.Status)
}

// PluginSelectResult contains the details of a select plugin result.
type PluginSelectResult struct {
	// Selected contains the selected ManagedCluster.
	Selected []*clusterapiv1.ManagedCluster
}
```

Then, a `EvenSpreadSelector` will run the following steps n times to select n clusters:

1. Exclude the clusters whose selection would cause the violation to `maxSkew`.  

2. Calculate the spread score of each cluster according to the current skewness.  

3. sum the spread scores and the scores from Prioritizers, and select the cluster with highest final score.  

### Graduation Criteria

#### Alpha
1. Demonstrate proposed features at the community meeting (last week in Sept 2022);
2. The new APIs are reviewed and accepted.

#### Beta
1. Implementation is completed to support all the functionalities; 
2. Develop test cases to cover all the cases including corner cases;
3. Build a simulator for deterministic output and integration test.

#### GA
1. Pass the performance/scalability testing.
