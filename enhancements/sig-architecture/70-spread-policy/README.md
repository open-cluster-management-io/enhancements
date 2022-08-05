Spread Policy across Failure-domains in Placement APIs
---

## Summary
This document proposes relevant API and code changes to apply for the OCM project [Topology-aware scheduling in cloud-native multi-cluster/multi-cloud environments](https://summer-ospp.ac.cn/#/org/prodetail/221390086) in OSPP 2022. This work is based on [previous community discussions](https://github.com/open-cluster-management-io/community/issues/49#issuecomment-857743027) on spread policy with several modifications.

## Background
Modern cloud providers typically provide K8s clusters in different datacenters and cities. Besides, as a user, a company may want to use clusters from different providers plus some self-hosted clusters. Here, three failure domains appeared (i.e., zone, region, and provider). To achieve fault-tolerance and better resource utilization, it is natural that users want to distribute workloads among these failure domains using a spread policy.

K8s itself supports spread policy (by [PodTopologySpread](https://web.archive.org/web/20200507113854/https://kubernetes.io/blog/2020/05/introducing-podtopologyspread/)). The support on other multicluster management systems [like Karmada](https://github.com/karmada-io/karmada/pull/1646) is on the way. Hence, it seems necessary for OCM to support a spread policy.

## Motivation
In this section, some use cases will be presented to motivate potential features we need for spread policy. Generally, by using the spread policy, we want to spread the workload across different failure domains.

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
    <td rowspan="2">alibaba<br></td>
    <td>cn-hangzhou</td>
    <td>cn-hangzhou-e</td>
  </tr>
  <tr>
    <td>cn-shenzhen</td>
    <td>cn-shenzhen-a</td>
  </tr>
</tbody>
</table>

### Use Case 1: User-defined Topology of Failure Domains

Different companies may manage clusters with different topology structures of failure domains. Hence, we use labels or claims to denote the failure domain where a ManagedCluster belongs. The label key is named topology key. Notably, all examples will be given with labels in this document.

**Example.** A company with only self-hosted clusters may only have “region” and “zone” labels for the clusters, while a company that also uses the public cloud has an extra “provider”.

### Use Case 2: Even Spread

The most intuitional way to spread the workload within a topology key is to break the workload into several identical parts and assign each failure domain the same number of parts. In the model of OCM Placement, this means that we need to select the same numbers of clusters for use from the different failure domains within the same topology key.

**Example.** We have a backend service to deploy in the clusters located in the region us-east-1. We want to spread the workload across the zones to achieve disaster tolerance, and we need four clusters in total. The final placement decision should contain two from us-east-1a and other two from us-east-1b.

### Use Case 3: Max Skew

When the even spread is unsatisfiable to a certain degree, we may want the scheduler not to schedule the workload.

**Example.** We want a even spread across the zones in us-east-1 (i.e., us-east-1a and us-east-1b) and four clusters in total. If us-east-1a can provide three clusters and us-east-1b can only provide one cluster, then the even spread cannot be satisfied strictly.

Following K8s's PodTopologySpread, we define the skew of a topoloy to be selected_clusters_in_current_topology - **min** selected_clusters_in_a_topology. For the above example, max skew is 3 - 1 = 2. If we want the max skew to be smaller than 2, then the scheduler should report an error.

### Use Case 4: (Cluster) Affinity/Anti-affinity Spread

Except for even spread, we may want to put the workload first to a specified topology A, and then to another specified topology B if the clusters provided by A are insufficient. Besides, we want to try the best not to put our workload to topology C if other topologies can provide enough clusters.

**Example.** We have a backend service to deploy to the public cloud. We prefer aws than other vendors for some reasons. The scheduler will first try to select only aws clusters then clusters from other vendors only if aws cannot provide enough clusters.

### Use Case 5: Joint Spread

To express hierarchical topology structure, clusters are labeled with multiple topology keys (i.e., region, zone, etc.), we want them to work jointly to determine the spread decision.

**Example.** We want four clusters from aws to put our backend service. Further, we want an even spread across regions and an affinity spread across zones. Specifically, for the region us-east-1, we want to first put the workload to us-east-1a then us-east-1b. Assuming all zones here can provide enough clusters. Then, the scheduling result should contain two clusters from us-east-1a and other two clusters from us-west-1a.

### Use Case 6: Spread with Ratio (Out of Scope)

Sometimes, we may need to spread the workload with a specified ratio. Let us consider a scenario where most of the access to the workload is from the east coast of the US, even spread between us-east-1 and us-west-1 would be inappropriate. Instead, we want to place more workload on us-east-1 than us-west-1. Besides, we want to control such preferences by user-specified configurations.

**Example.** We want six clusters to deploy our backend service. Moreover, we want 2/3 of them to be located in us-east-1 and 1/3 to be located in us-west-1 since there are more clients from the east coast of the US.

## Goal
Generally, we have two goals for the spread policy:
1. Cover the full functionality in the Motivation section;
2. Implement a high-performance and scalable mechanism that works for thousands of clusters efficiently.

## Proposal

### API Spec
```go
type PlacementSpec struct {
	// ...

	// SpreadConstraints defines how placement decision should be distributed among a
	// set of clusters.
	SpreadConstraints []SpreadConstraintsTerm `json:"spreadConstraints,omitempty"`
}

// SpreadConstraintsTerm defines a terminology to spread placement decisions
type SpreadConstraintsTerm struct {
	// Type is the type of SpreadConstraintsTerm. It could be Even or Affinity.
	// +required
	Type string `json:"type"`

	// TopologyKey is either a label key or a cluster claim name of ManagedClusters
	// +required
	TopologyKey string `json:"topologyKey"`

	// TopologyKeyType indicates the type of TopologyKey. It could be Label or Claim.
	// +required
	TopologyKeyType string `json:"topologyKeyType"`

	// TopologyWeights indicates the optional weights of topologies, valid when Type
	// is Affinity.
	TopologyWeights []TopologyWeightTerm `json:"topologyWeights,omitempty"`

	// MaxSkew describes the degree to which the workload may be unevenly distributed,
	// valid when Type is Even. If omitted, the scheduler treat the even constraint a soft
	// constraint.
	MaxSkew *int32 `json:"maxSkew,omitempty"`

	// Order indicates the order of even constraints. It is valid when Type is Even and
	// required only when multiple even constraints are used. To spread the workload,
	// the scheduler takes the even constraints with a smaller order value into prior consideration.
	Order int32 `json:"order,omitempty"`
}

// TopologyWeightTerm defines a terminology for the weight of topologies
type TopologyWeightTerm struct {
	// Weight defines the relative number of clusters to schedule in this topology.
	// +required
	Weight int32 `json:"weight"`

	// Operator represents a key's relationship to a set of values.
	// Valid operators are In, NotIn, Exists and DoesNotExist.
	// +required
	Operator metav1.LabelSelectorOperator `json:"operator"`

	// Values is an array of string values. If the operator is In or NotIn,
	// the values array must be non-empty. If the operator is Exists or DoesNotExist,
	// the values array must be empty.
	Values []string `json:"values,omitempty"`
}
```

### API Example

#### Example 1: Joint use of a Even constraint and Affinity constraints
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: demo-placement
spec:
  # numberOfClusters is required if spreadConstraints contains
  # a constraint of Even type.
  numberOfClusters: 10
  # Prediates work before the spread constraints to filter
  # out some undesired topologies.
  predicates:
  - # ...
  # SpreadConstraints contain two constraints, which work
  # independently and jointly.
  spreadConstraints:
    # order is omitted for this Even constraint as there is only
    # one of it.
  - type: Even
    topologyKey: provider
    topologyKeyType: Label
    maxSkew: 2
    # Affininy constraints do not have a maxSkew field.
  - type: Affinity
    topologyKey: region
    topologyKeyType: Label
    # The cluster affinity api doesn't use LabelSelector
    # directly in order to force users to organize the
    # affinity rules of the same topologyKey together,
    # which may be more clear.
    topologyWeights:
    - weight: 20
      operator: In
      values:
      - us-east-1
      - cn-hangzhou
    - weight: 10
      operator: In
      values:
      - us-west-1
    - weight: 20
      # NotIn expresses anti-affinity.
      operator: NotIn
      values:
      - cn-shenzhen
  - type: Affinity
    topologyKey: zone
    topologyKeyType: Label
    topologyWeights:
    - # ...
```

#### Example 2: Joint use of multiple Even rules
```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: demo-placement
spec:
  numberOfClusters: 10
  predicates:
  - # ...
  spreadConstraints:
    # When multiple Even constraints are used, the user should
    # specify their orders. The scheduler will consider to first
    # do the spread according to the Even constraints with smaller
    # order value.
  - type: Even
    topologyKey: provider
    topologyKeyType: Label
    maxSkew: 2
    order: 1
    # maxSkew is omitted as this one is a soft Even constraint
  - type: Even
    topologyKey: region
    topologyKeyType: Label
    order: 2
```

### Implementation
Two types of spread constraints require different logics. Hence, they will be implemented separately. Affinity constraints are implemented by a `Prioritizer` to calculate the scores of the clusters using the information conveyed via `topologyWeights`. However, it seems that the scheduler's current design of the plugin framework makes it difficult to express the logic of even spreads. Therefore, an extra `Selector` plugin type is proposed.

**Implementation of (Cluster) Affinity/Anti-affinity.** An `AffinitySpreadPriorititizer` will be implemented as follows:

1. for each cluster, score it according to `topologyWeights` in the cluster affinity rules
2. normalize the cluster affinity scores to [-100, 100]

**Implementation of Even Spread.** The current scheduling framework contains two types of plugins, i.e., `Filter`s and `Prioritizer`s. `Filter`s are invoked *before* `Prioritizer`s to express hard constrains, filtering the clusters. `Prioritizer`s express soft constraints, scoring the clusters. The even spread constraints can be hard constraints (when `maxSkew` is not omitted) or soft constraints (when `maxSkew` is omitted). For a hard even spread constraint, the scheduler need to perform the hard decision logic (i.e., decide to schedule or not according to `maxSkew`) *after* scoring the clusters, which is unable to implement in a `Filter`.

Hence, a new plugin type, `Selector`, is proposed to express hard constrains *after* `Prioritizer`s. The `Selector` interface is showed below.

```go
// Selector defines a selector plugin that selects clusters after scoring them.
type Selector interface {
	Plugin

	// Select returns the selected clusters.
	Select(ctx context.Context, placement *clusterapiv1beta1.Placement, clusters []*clusterapiv1.ManagedCluster, scores map[string]int64) PluginSelectResult
}

// PluginSelectResult contains the details of a select plugin result.
type PluginSelectResult struct {
	// Selected contains the selected ManagedCluster.
	Selected []*clusterapiv1.ManagedCluster
	// Err contains the select plugin error message.
	Err error
}
```

Then, a `EvenSpreadSelector` will be implemented as follows to select n clusters.

1. The result cluster set = {}
2. Call `selectOne` n times, in each time insert the returned cluster into the result cluster set
3. Return the result cluster set

`selectOne`:

1. For each cluster, calculate a score (even_score) according the benefit of selection of it to balancing the skewness. If the selection of a cluster would cause the violation to `maxSkew`, exclude it from the later selecting phases. If all clusters are excluded, report a error.
2. For each cluster, calculate its final score = u*even_score + its_score_from_prioritizers, where u is a parameter.
3. Sort the clusters by final scores and return the one with the highest final score.
