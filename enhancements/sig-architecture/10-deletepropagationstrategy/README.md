# Introduce DeletePropagation in ManifestWork

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary

Today, when a manifestwork is deleted or a manifest resource in the manifestwork is removed, the work agent
deletes the related resources on the managedcluster in the `ForegroundDeletion` strategy. It means the manifestwork
will be kept in deleting status until the related resource owned by the manifestwork is deleted on the managedcluster.
There is use cases that asks for othere deletion strategies. This proposal is to introduce a property in manifestwork
that the user can define the orphan deletion propagation for resources in the manifestwork.

## Motivation

### Goals

- Define the DeletePropagationStrategy property in manifestwork
- Indicate the behavior of work agent upon the existence of DeletePropagationStrategy property

### Non-Goals

- Deletion strategy other than orphan.

## Proposal

We will introduce a `DeletePropagations` property in the manifestwork API.

### User Stories

#### Story 1

As a manifestwork user, I create a manifeswork with a list of resources. When I delete the manifestwork on the hub,
I do not want the resources defined in the manifestwork to be garbage collected on the managedcluster.

#### Story 2

As a manifestwork user, I create a manifeswork with a list of resources. When I remove one or several resources from
the manifestwork, I do not what these resources to be garbage collected on the managedcluster.


## Design Details

### Manifestwork API update

We will introduce a `DeleteOption` property as following:

```go
// ManifestWorkSpec represents a desired configuration of manifests to be deployed on the managed cluster.
type ManifestWorkSpec struct {
	// Workload represents the manifest workload to be deployed on a managed cluster.
	Workload ManifestsTemplate `json:"workload,omitempty"`

	// DeleteOption represents deletion strategy when the manifestwork is deleted.
	// Foreground deletion strategy is applied to all the resource in this manifestwork if it is not set.
	// +optional
	DeleteOption DeleteOption `json:"deleteOption,omitempty"`
}

type DeleteOption struct {
	// propagationPolicy can be Orphan or SelectivelyOrphan
	// SelectivelyOrphan should be rarely used.  It is provided for cases where particular resources is transfering
	// ownership from one ManifestWork to another or another management unit.
	// Setting this value will allow a flow like
	// 1. create manifestwork/2 to manage foo
	// 2. update manifestwork/1 to selectively orphan foo
	// 3. remove foo from manifestwork/1 without impacting continuity because manifestwork/2 adopts it.
	// +optional
	PropagationPolicy DeletePropagationPolicyType `json:"propagationPolicy,omitempty"`

	// selectivelyOrphan represents a list of resources following orphan deletion stratecy
	SelectivelyOrphan *SelectivelyOrphan `json:"selectivelyOrphans,omitempty"`
}

// SelectivelyOrphan represents a list of resources following orphan deletion stratecy
type SelectivelyOrphan struct {
	// orphaningRules defines a slice of orphaningrule.
	// Each orphaningrule identifies a single resource included in this manifestwork
	// +optional
	OrphaningRules []OrphaningRule `json:"orphaningRules,omitempty"`
}

// OrphaningRule identifies a single resource included in this manifestwork
type OrphaningRule struct {
	// Group is the api group of the resources in the workload that the strategy is applied
	// +required
	Group string `json:"group"`
	// Resource is the resources in the workload that the strategy is applied
	// +required
	Resource string `json:"resource"`
	// ResourceNamespace is the namespaces of the resources in the workload that the strategy is applied
	// +optional
	ResourceNamespace string `json:"resourceNamespace"`
	// ResourceName is the names of the resources in the workload that the strategy is applied
	// +required
	ResourceName string `json:"resourceName"`
}

type DeletePropagationPolicyType string

const (
	// DeletePropagationPolicyTypeOrphan represents that all the resources in the manifestwork is orphaned
	// when the manifestwork is deleted.
	DeletePropagationPolicyTypeOrphan DeletePropagationPolicyType = "Orphan"
	// DeletePropagationPolicyTypeSelectivelyOrphan represents that only selected resources in the manifestwork
	// is orphaned when the manifestwork is deleted.
	DeletePropagationPolicyTypeSelectivelyOrphan DeletePropagationPolicyType = "SelectivelyOrphan"
)
```

### Examples

#### All the resources in manifestwork is orphaned

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: work
  namespace: cluster1
spec:
  workload:
    manifests:
    - apiVersion: v1
      kind: ConfigMap
      metadata:
        name: cm1
        namespace: default
  deleteOption:
    propagationPolicy: Orphan
```

The manifestwork defition indicates that all resources will be orphaned when the manifestwork is deleted

#### Only certain resource type is orphaned

The following example indicates only customresourcedefitions foo is orphaned. When customresourcedefition foo is 
removed from manifestwork, it is also orphaned.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: work
  namespace: cluster1
spec:
  workload:
    manifests:
    - apiVersion: v1
      kind: ConfigMap
      metadata:
        name: cm1
        namespace: default
    - apiVersion: apiextensions.k8s.io/v1
      kind: CustomResourceDefinition
      name: foo
      ...
  deleteOption:
    propagationPolicy: SelectivelyOrphan
    selectivelyOrphan:
    - group: apiextensions.k8s.io
      resource: customresourcedefinitions
      resourceName: foo
```

Another example as below indicates that only configmap with name of cm2 will be orphan deleted. When cm2 is removed from
manifeswork, the cm1 on managedcluster is orphaned. While when cm1 is removed from the manifestwork, cm1 will be deleted
on the managedcluster.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: work
  namespace: cluster1
spec:
  workload:
    manifests:
    - apiVersion: v1
      kind: ConfigMap
      metadata:
        name: cm1
        namespace: default
    - apiVersion: v1
      kind: ConfigMap
      metadata:
        name: cm2
        namespace: default
  deleteOption:
    propagationPolicy: SelectivelyOrphan
    selectivelyOrphan:
    - group: apiextensions.k8s.io
      resource: customresourcedefinitions
      resourceName: cm2
      resourceNamespace: default
```

### Implementation Details
1. When work agent apply a resource, it should evaluate the DeletePropagationStrategy field. If a resource is included in the field,
   the ownerref of the appliemanifestwork will not be added onto the resource.
2. When the resource is removed from manifestwork, the work agent should check if the resource has the ownerref of the related
   appliedmanifestwork. The work agent skip the deletion if the ownerref does not exist.
3. When the manifestwork is deleted, the work agent should check if each resource has the ownerref of the related
   appliedmanifestwork. The work agent skip the deletion if the ownerref does not exist.
4. If the manifestwork is already in deleting state, mutating DeletePropagationStrategy orphan field should be disallowed.

### Test Plan

e2e test will be implement to cover the following cases:
- orphan all resources in a mannifestwork
- orphan certain resources in a manifestwork
- disable orphan deletion to ensure backward compatible
- mutate orphan configuration.

### Graduation Criteria

### Upgrade / Downgrade Strategy

- The registration-operator should be upgraded to ensure that manifestwork API is upgraded
- If the DeletePropagationStrategy property is not set in the manifestwork, we need to ensure that manifestwork
  is treated following foreground deletion processs

### Version Skew Strategy

- The DeletePropagationStrategy is optional, and if it is not set, the manifestwork can be correctly treated by work agent with elder version
- The elder version work agent will ignore the DeletePropagationStrategy field.

## Implementation History

## Drawbacks

## Alternatives

- We should only allow orphan delete the whole manifestwork. Specifying certain resource to be orphaned might be complicated and confusing

