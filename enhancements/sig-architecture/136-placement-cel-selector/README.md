# Select clusters with CEL expressions

## Release Signoff Checklist

- [] Enhancement is `provisional`
- [] Design details are appropriately documented from clear requirements
- [] Test plan is defined
- [] Graduation criteria for dev preview, tech preview, GA
- [] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal proposes an enhancement to the placement cluster selector, enabling users to select clusters with CEL expressions.

## Motivation

In the OCM placement predicate stage, the user can currently use a label selector or claim selector to selector clusters. For detailed usage see [Label/Claim selection](https://open-cluster-management.io/docs/concepts/placement/#predicates). It can meet most of the use cases. However, there are some more advanced needs recently that the current functionality cannot fulfill. For example, the user wants to filter clusters with label "version" > 1.30.0. It's not smart to list every supported version in the label selector. Another example is bin packing, the user may want to filter clusters with specific resource quantities before prioritizing, the resource quantity will come from another resource, eg `AddonPlacementScore` but not labels or claims. So we have this proposal to support selecting clusters with [CEL expressions](https://cel.dev/).

### Goals

- Enhance `Placement` API to allow users to select clusters with CEL expressions.

### Non-Goals

- Enhance the `AddonPlacementScore` to include the original resource value for selecting clusters that is not in the scope.

## Proposal

In this proposal, we will introduce the `celSelector` in the predicate section. It will be a list of `celExpressions`. The result of the expressions are ANDed. The result of `labelSelector`, `claimSelector` and `celSelector` is also ANDed.

We will also introduce a CEL library, `managedCluster` is declared as a variable and can be used in CEL expressions. Custom functions like comparing versions and getting cluster resources will also be added.

### User Stories

1. The user can select clusters by fields other than labels and claims. For example, select clusters by field `managedCluster.Status.version.kubernetes`.
2. The user can use CEL [Standard macros](https://github.com/google/cel-spec/blob/master/doc/langdef.md#macros) and [Standard functions](https://github.com/google/cel-spec/blob/master/doc/langdef.md#standard-definitions) on the `managedCluster` fields. For example, select clusters with version 1.30.* and 1.31.* by regex `'^1\\.(30|31)\\.\\d+$'`.
3. The user can use CEL customized function to select clusters by version comparison. For example, select clusters whose version > 1.13.0.
4. The user can use CEL customized function to select clusters by `AddonPlacementScore`. For example, select clusters whose cpuAvailable < 2.

## Design Details

### API Changes

Add the new field `celSelector` to the `clusterSelector`. `celSelector` will be list of `celExpressions` with string type.

```go
type ClusterSelector struct {
	// LabelSelector represents a selector of ManagedClusters by label
	// +optional
	LabelSelector metav1.LabelSelector `json:"labelSelector,omitempty"`

	// ClaimSelector represents a selector of ManagedClusters by clusterClaims in status
	// +optional
	ClaimSelector ClusterClaimSelector `json:"claimSelector,omitempty"`

	// CelSelector represents a selector of ManagedClusters by CEL expressions on ManagedCluster fields
	// +optional
	CelSelector ClusterCelSelector `json:"celSelector,omitempty"`
}

// ClusterCelSelector is a list of CEL expressions. The expressions are ANDed.
type ClusterCelSelector struct {
	// CelExpressions describes a list of validation rules written in the CEL expression language.
	// ref: https://github.com/google/cel-spec
	// The CelExpressions is scoped to the Managed Cluster, the `managedCluster` variable in the CEL expression is bound to the scoped value.
	// For example, select clusters with label version < v1.31.0:
	//     semver(managedCluster.metadata.labels["version"], true).isLessThan(semver("v1.31.0", true))
	// +optional
	CelExpressions []string `json:"celExpressions,omitempty"`
}
```

### ManagedCluster CEL Library

A ManagedClusterLib will be added.
- Variable `managedCluster` will be added.
- Function `scores` will be added. 
- Function `parseJSON` will be added. 

```go
// ManagedClusterLib defines the CEL library for ManagedCluster evaluation.
// It provides functions and variables for evaluating ManagedCluster properties
// and their associated resources.
//
// Variables:
//
// managedCluster
//
// Provides access to ManagedCluster properties.
//
// Functions:
//
// scores
//
// Returns a list of AddOnPlacementScoreItem for a given cluster and AddOnPlacementScore resource name.
//
//	scores(<ManagedCluster>, <string>) <list>
//
// The returned list contains maps with the following structure:
//   - name: string - The name of the score
//   - value: int - The numeric score value
//   - quantity: number|string - The quantity value, represented as:
//   - number: for pure decimal values (e.g., 3)
//   - string: for values with units or decimal places (e.g., "300Mi", "1.5Gi")
//
// Examples:
//
//	managedCluster.scores("cpu-memory") // returns [{name: "cpu", value: 3, quantity: 3"}, {name: "memory", value: 4, quantity: "300Mi"}]
//
// parseJSON
//
// Parses a JSON string into a CEL-compatible map or list.
//
//	parseJSON(<string>) <dyn>
//
// Takes a single string argument, attempts to parse it as JSON, and returns the resulting
// data structure as a CEL-compatible value. If the input is not a valid JSON string, it returns an error.
//
// Examples:
//
//	parseJSON("{\"key\": \"value\"}") // returns a map with key-value pairs
type ManagedClusterLib struct{}

// CompileOptions implements cel.Library interface to provide compile-time options.
func (ManagedClusterLib) CompileOptions() []cel.EnvOption {
	return []cel.EnvOption{
		// The input types may either be instances of `proto.Message` or `ref.Type`.
		// Here we use func ConvertManagedCluster() to convert ManagedCluster to a Map.
		cel.Variable("managedCluster", cel.MapType(cel.StringType, cel.DynType)),

		cel.Function("scores",
			cel.MemberOverload(
				"cluster_scores",
				[]*cel.Type{cel.DynType, cel.StringType},
				cel.ListType(cel.DynType),
				cel.FunctionBinding(clusterScores)),
		),

		cel.Function("parseJSON",
			cel.MemberOverload("parse_json_string",
				[]*cel.Type{cel.StringType},
				cel.DynType,
				cel.FunctionBinding(l.parseJSON),
			),
		),
	}
}
```

The below library will be added when creating the CEL program environment.
- ManagedClusterLib library 
- Extend strings library
- Kubernetes known libraries: list, regex, url, quantity, IP, cidr, format, semver.

```go
func NewEvaluator() (*Evaluator, error) {
	// Add the ManagedClusterLib and Kubernetes libs to the CEL environment
	env, err := cel.NewEnv(
		ManagedClusterLib(scoreLister),
		cel.OptionalTypes(),
		ext.Strings(),
		library.Lists(),
		library.Regex(),
		library.URLs(),
		library.Quantity(),
		library.IP(),
		library.CIDR(),
		library.Format(),
		library.SemverLib(),
	)
	if err != nil {
		return nil, fmt.Errorf("failed to create CEL environment: %w", err)
	}
	return &Evaluator{env: env}, nil
}
```

### Metrics for time spent evaluating CEL at runtime.
Add histograms metrics for time spent evaluating CEL at runtime.

```
# HELP scheduling_cel_duration_seconds [ALPHA] How long in seconds it takes to schedule a placement.
# TYPE scheduling_cel_duration_seconds histogram
scheduling_cel_duration_seconds_bucket{name="placement_scheduling",le="1e-06"} 0
scheduling_cel_duration_seconds_bucket{name="placement_scheduling",le="9.999999999999999e-06"} 0
scheduling_cel_duration_seconds_bucket{name="placement_scheduling",le="9.999999999999999e-05"} 13
scheduling_cel_duration_seconds_bucket{name="placement_scheduling",le="0.001"} 100
scheduling_cel_duration_seconds_bucket{name="placement_scheduling",le="0.01"} 100
scheduling_cel_duration_seconds_bucket{name="placement_scheduling",le="0.1"} 100
scheduling_cel_duration_seconds_bucket{name="placement_scheduling",le="1"} 100
scheduling_cel_duration_seconds_bucket{name="placement_scheduling",le="10"} 100
scheduling_cel_duration_seconds_bucket{name="placement_scheduling",le="100"} 100
scheduling_cel_duration_seconds_bucket{name="placement_scheduling",le="1000"} 100
scheduling_cel_duration_seconds_bucket{name="placement_scheduling",le="+Inf"} 100
scheduling_cel_duration_seconds_sum{name="placement_scheduling"} 0.013472362999999998
scheduling_cel_duration_seconds_count{name="placement_scheduling"} 100
```

### Runtime Constraints
Estimated Cost Limit
- Don't allow Placement to contain CEL expressions that would, in the worst case, take too long to run.

Runtime Cost Limit
- Halt CEL expressions that take too long to run.

### CEL validation error message
When the CEL expressions faield to run due to meet cost limit or expressions error, the error message will return and shown in the Placement status.
```
    - lastTransitionTime: "2025-03-20T15:33:00Z"
      message: failed to compile CEL expression 'managedCluster.metadata.labels["version"].matchess('^1\\.(14|15)\\.\\d+$')': ERROR: <input>:1:51: undeclared reference to 'matchess' (in container '')
      reason: Misconfigured
      status: "True"
      type: PlacementMisconfigured
```

### Examples

1. The user can select clusters by Kubernetes version listed in `managedCluster.Status.version.kubernetes`.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: placement1
  namespace: default
spec:
  predicates:
    - requiredClusterSelector:
        celSelector:
          celExpressions:
            - managedCluster.status.version.kubernetes == "v1.30.0"
```
2. The user can use CEL [Standard macros](https://github.com/google/cel-spec/blob/master/doc/langdef.md#macros) and [Standard functions](https://github.com/google/cel-spec/blob/master/doc/langdef.md#standard-definitions) on the `managedCluster` fields. 

For example, select clusters with version 1.30.* and 1.31.* by regex.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: placement1
  namespace: default
spec:
  predicates:
    - requiredClusterSelector:
        celSelector:
          celExpressions:
            - managedCluster.metadata.labels["version"].matches('^1\\.(30|31)\\.\\d+$')
```

If the version info is stored in `clusterClaims`, the user can use the following expression.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: placement1
  namespace: default
spec:
  predicates:
    - requiredClusterSelector:
        celSelector:
          celExpressions:
            - managedCluster.status.clusterClaims.exists(c, c.name == "kubeversion.open-cluster-management.io" && c.value.matches('^1\\.(30|31)\\.\\d+$'))
```


3. The user can use Kubernetes semver library functions `isLessThan` and `isGreaterThan` to select clusters by version comparison. For example, select clusters whose kubernetes version > v1.13.0.


```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: placement1
  namespace: default
spec:
  predicates:
    - requiredClusterSelector:
        celSelector:
          celExpressions:
            # Upstream enhancement to support v is a prefix https://github.com/kubernetes/kubernetes/pull/130648.
			- semver(managedCluster.status.version.kubernetes, true).isGreaterThan(semver("v1.30.0", true))
```


4. The user can use CEL customized function `scores` to select clusters by `AddonPlacementScore`. And use Kubernetes quantity library function `isLessThan` and `isGreaterThan` to compare quantities.
For example, select clusters whose cpuAvailable quantity > 4 and memAvailable quantity > 100Mi.


```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: placement1
  namespace: default
spec:
  predicates:
    - requiredClusterSelector:
        celSelector:
          celExpressions:
            - managedCluster.scores("default").filter(s, s.name == 'cpuAvailable').all(e, e.quantity > 4)
            - managedCluster.scores("default").filter(s, s.name == 'memAvailable').all(e, quantity(e.quantity).isGreaterThan(quantity("100Mi")))
```

5. The user can use CEL to prase properties with format like, a comma separated string or a json format string.

```yaml
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ClusterProfile
metadata:
 name: bravelion
 namespace: ...
...
status:
   properties:
   - name: sku.node.k8s.io
     value: g6.xlarge,Standard_NC48ads_H100,m3-ultramem-32
   - name: sku.gpu.kubernetes-fleet.io
     value: |
		{
		    "H100": [
		        {
		            "Standard_NC48ads_H100_v4": 10
		        }
		        {
		            "Standard_NC96ads_H100_v4": 2
		        }
		    ]
		  "A100": [
		        {
		            "Standard_NC48ads_A100_v4": 50
		        }
		        {
		            "Standard_NC96ads_A100_v4": 20
		        }
		    ]
		}
...
```

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
…
spec:
  predicates:
    - requiredClusterSelector:
        celSelector:
          celExpressions:
            - managedCluster.status.properties.exists(c, c.name == "sku.node.k8s.io" && c.value.split(",").exists(e, e == "g6.xlarge"))
            - managedCluster.status.properties.exists(c, c.name == "sku.gpu.kubernetes-fleet.io" && c.value.parseJSON().H100.exists(e, e.Standard_NC96ads_H100_v4 == 2))
```

### Test Plan

- testing selecting clusters with different ManagedCluster fields.
- testing selecting clusters with scores, parseJSON function.
- testing selecting clusters with kubernetes library.
- testing selecting clusters with cost limit.
- check scheduling_cel_duration_seconds metrics.

### Graduation Criteria
N/A

### Upgrade Strategy
N/A

## Alternatives
N/A


