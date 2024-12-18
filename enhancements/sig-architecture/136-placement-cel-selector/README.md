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
	// +optional
	CelExpressions []string `json:"celExpressions,omitempty"`
}
```

### ManagedCluster CEL Library

A ManagedClusterLib will be added.
- Variable `managedCluster` will be added.
- Function `scores` will be added. 
- Function `versionIsGreaterThan`, `versionIsLessThan` will be added. 
- Function `quantityIsGreaterThan`, `quantityIsLessThan` will be added. 

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
//     * number: for pure decimal values (e.g., 3)
//     * string: for values with units or decimal places (e.g., "300Mi", "1.5Gi")
//
// Examples:
//
//	managedCluster.scores("cpu-memory") // returns [{name: "cpu", value: 3, quantity: 3}, {name: "memory", value: 4, quantity: "300Mi"}]
//
// Version Comparisons:
//
// versionIsGreaterThan
//
// Returns true if the first version string is greater than the second version string.
// The version must follow Semantic Versioning specification (http://semver.org/).
// It can be with or without 'v' prefix (eg, "1.14.3" or "v1.14.3").
//
//	versionIsGreaterThan(<string>, <string>) <bool>
//
// Examples:
//
//	versionIsGreaterThan("1.25.0", "1.24.0") // returns true
//	versionIsGreaterThan("1.24.0", "1.25.0") // returns false
//
// versionIsLessThan
//
// Returns true if the first version string is less than the second version string.
// The version must follow Semantic Versioning specification (http://semver.org/).
// It can be with or without 'v' prefix (eg, "1.14.3" or "v1.14.3").
//
//	versionIsLessThan(<string>, <string>) <bool>
//
// Examples:
//
//	versionIsLessThan("1.24.0", "1.25.0") // returns true
//	versionIsLessThan("1.25.0", "1.24.0") // returns false
//
// Quantity Comparisons:
//
// quantityIsGreaterThan
//
// Returns true if the first quantity string is greater than the second quantity string.
//
//	quantityIsGreaterThan(<string>, <string>) <bool>
//
// Examples:
//
//	quantityIsGreaterThan("2Gi", "1Gi") // returns true
//	quantityIsGreaterThan("1Gi", "2Gi") // returns false
//	quantityIsGreaterThan("1000Mi", "1Gi") // returns false
//
// quantityIsLessThan
//
// Returns true if the first quantity string is less than the second quantity string.
//
//	quantityIsLessThan(<string>, <string>) <bool>
//
// Examples:
//
//	quantityIsLessThan("1Gi", "2Gi") // returns true
//	quantityIsLessThan("2Gi", "1Gi") // returns false
//	quantityIsLessThan("1000Mi", "1Gi") // returns true

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

		cel.Function("versionIsGreaterThan",
			cel.MemberOverload(
				"version_is_greater_than",
				[]*cel.Type{cel.StringType, cel.StringType},
				cel.BoolType,
				cel.FunctionBinding(versionIsGreaterThan)),
		),

		cel.Function("versionIsLessThan",
			cel.MemberOverload(
				"version_is_less_than",
				[]*cel.Type{cel.StringType, cel.StringType},
				cel.BoolType,
				cel.FunctionBinding(versionIsLessThan)),
		),

		cel.Function("quantityIsGreaterThan",
			cel.MemberOverload(
				"quantity_is_greater_than",
				[]*cel.Type{cel.StringType, cel.StringType},
				cel.BoolType,
				cel.FunctionBinding(quantityIsGreaterThan)),
		),

		cel.Function("quantityIsLessThan",
			cel.MemberOverload(
				"quantity_is_less_than",
				[]*cel.Type{cel.StringType, cel.StringType},
				cel.BoolType,
				cel.FunctionBinding(quantityIsLessThan)),
		),
	}
}
```

This `ManagedClusterLib` will be added when `cel.NewEnv()`.

```go
func NewEvaluator() (*Evaluator, error) {
	env, err := cel.NewEnv(
		cel.Lib(ManagedClusterLib{}), // Add the ManagedClusterLib to the CEL environment
	)
	if err != nil {
		return nil, fmt.Errorf("failed to create CEL environment: %w", err)
	}
	return &Evaluator{env: env}, nil
}
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
  numberOfClusters: 3
  clusterSets:
    - prod
  predicates:
    - requiredClusterSelector:
        celSelector:
          celExpressions:
            - managedCluster.Status.version.kubernetes == "v1.30.0"
```
2. The user can use CEL [Standard macros](https://github.com/google/cel-spec/blob/master/doc/langdef.md#macros) and [Standard functions](https://github.com/google/cel-spec/blob/master/doc/langdef.md#standard-definitions) on the `managedCluster` fields. For example, select clusters with version 1.30.* and 1.31.* by regex..

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: placement1
  namespace: default
spec:
  numberOfClusters: 3
  clusterSets:
    - prod
  predicates:
    - requiredClusterSelector:
        celSelector:
          celExpressions:
            - managedCluster.metadata.labels["version"].matches('^1\\.(30|31)\\.\\d+$')
```

3. The user can use CEL customized functions `versionIsGreaterThan` and `versionIsLessThan` to select clusters by version comparison. For example, select clusters whose kubernetes version > v1.13.0.


```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: placement1
  namespace: default
spec:
  numberOfClusters: 3
  clusterSets:
    - prod
  predicates:
    - requiredClusterSelector:
        celSelector:
          celExpressions:
            - managedCluster.Status.version.kubernetes.versionIsGreaterThan("v1.30.0")
```


4. The user can use CEL customized function `score` to select clusters by `AddonPlacementScore`. For example, select clusters whose cpuAvailable score < 2.


```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: placement1
  namespace: default
spec:
  numberOfClusters: 3
  clusterSets:
    - prod
  predicates:
    - requiredClusterSelector:
        celSelector:
          celExpressions:
            - managedCluster.scores("default").filter(s, s.name == 'cpuAvailable').all(e, e.quantity > 4)
            - managedCluster.scores("default").filter(s, s.name == 'memAvailable').all(e, e.quantity.quantityIsGreaterThan("100Mi"))
```


### Test Plan

- TBD

### Graduation Criteria
N/A

### Upgrade Strategy
N/A

## Alternatives
N/A


