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

In the OCM placement predicate stage, the user can currently use a label selector or claim selector to selector clusters. For detailed usage see [Label/Claim selection](https://open-cluster-management.io/docs/concepts/placement/#predicates). It can meet most of the use cases. However, there are some more advanced needs recently that the current functionality cannot fulfill. For example, the user wants to filter clusters with label "version" > 1.30.0. It's not smart to list every supported version in the label selector. Another example is bin packing, the user may want to filter clusters with specific resource quantities before prioritizing, the resource quantity will come from another resource, eg `AddonPlacementScore` but not labels or claims. So we have this proposal to support selecting clusters with CEL expressions.

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
The input types may either be instances of `proto.Message` or `ref.Type`. Since OCM API does not implement `proto.Message`, we will add a function called ConvertManagedCluster() to convert ManagedCluster to a Map as an easier way to get started.

- Function `score` will be added. 
`score` is used to get value from given `AddonPlacementScore` resource name and score name.
The input is `managedCluster` (`cel.DynType`), score resource name (`cel.StringType`) and scrore name (`cel.StringType`). 
Output is the score value (`cel.IntType`).

- Function `versionIsGreaterThan`, `versionIsLessThan` will be added. 
`versionIsGreaterThan` checks if a version is greater than a given version. `versionIsLessThan` checks if a version is less than a given version.
The input are 2 versions with type `cel.StringType`. The output is bool.

```go
// CompileOptions implements cel.Library interface to provide compile-time options.
func (ManagedClusterLib) CompileOptions() []cel.EnvOption {
	return []cel.EnvOption{
		// The input types may either be instances of `proto.Message` or `ref.Type`.
		// Here we use func ConvertManagedCluster() to convert ManagedCluster to a Map.
		cel.Variable("managedCluster", cel.MapType(cel.StringType, cel.AnyType)),

		// score is used to get value from given addonplacementscore resource name and score name
		// The input is `managedCluster` (`cel.DynType`), score resource name (`cel.StringType`) and scrore name (`cel.StringType`).
		// Output is the score value (`cel.IntType`).
		cel.Function("score",
			cel.MemberOverload(
				"cluster_contains_score",
				[]*cel.Type{cel.DynType, cel.StringType, cel.StringType},
				cel.IntType,
				cel.FunctionBinding(clusterContainsScore)),
		),

		// versionIsGreaterThan checks if a version is greater than a given version.
		// The input are 2 versions with type `cel.StringType`.
		// The output is bool.
		cel.Function("versionIsGreaterThan",
			cel.MemberOverload(
				"version_is_greater_than",
				[]*cel.Type{cel.StringType, cel.StringType},
				cel.BoolType,
				cel.FunctionBinding(versionIsGreaterThan)),
		),

		// versionIsLessThan checks if a version is less than a given version.
		// The input are 2 versions with type `cel.StringType`.
		// The output is bool.
		cel.Function("versionIsLessThan",
			cel.MemberOverload(
				"version_is_less_than",
				[]*cel.Type{cel.StringType, cel.StringType},
				cel.BoolType,
				cel.FunctionBinding(versionIsLessThan)),
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

3. The user can use CEL customized functions `versionIsGreaterThan` and `versionIsLessThan` to select clusters by version comparison. For example, select clusters whose version > 1.13.0.


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
            - managedCluster.metadata.labels["version"].versionIsGreaterThan("1.30.0")
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
            - managedCluster.score("default, "cpuAvailable") < 2
```


### Test Plan

- TBD

### Graduation Criteria
N/A

### Upgrade Strategy
N/A

## Alternatives
N/A


