# New Update Strategy only when workload changes in ManifestWork

## Release Signoff Checklist

- [] Enhancement is `provisional`
- [] Design details are appropriately documented from clear requirements
- [] Test plan is defined
- [] Graduation criteria for dev preview, tech preview, GA
- [] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary
This proposal is to introduce new options in update stratgey of manifestwork to resolve issues
on how to coordinate other actors and work agent when they apply the same resource on the spoke
cluster.

## Motivation
We have introduced several update strategy in `ManifestWork` to resolve resource conflict before:
- `CreateOnly` let work-agent to create the resource only and ignore any further change on the
resource
- `ServerSideApply` utilize the server side apply mechanism in kube-apiserver, so if work-agent
and another actor try to operator the same field in a resource, a conflict can be identified.

However, some issues are raised indicating the above strategy can still not meet all the requirements
- [issue 631](https://github.com/open-cluster-management-io/ocm/issues/631) reports a case that user
uses manifestwork to create a configmap, and then another actor on the spoke change the configmap. The
user does not want the configmap to be changed back by the work-agent. However, when the configmap resource
in the manifestwork is updated, the user wants the configmap to be updated accordingly.
- [issue 670](https://github.com/open-cluster-management-io/ocm/issues/670) is a case when the user
uses manifestwork to create argocd's `Application` resource. The `Application` resource has an `operation`
field which is used to trigger the sync, and the field will be removed by the argocd when the sync is done.
User in the manifestwork sets the `operation` field and when argocd removes the field, the user does not want
the field to be updated back by the work-agent.
- [issue 690](https://github.com/open-cluster-management-io/ocm/issues/690) is the case that user wants
to create a deployment using manifestwork, but what HPA on the spoke the control the replicas of the deployment.
Hence the user wants the work-agent to ignore the replicas field of the deployment in the manfiestwor if it is
set.


## Proposal

We would like to introduce new options in `Update` and `ServerSideApply` update strategy to resolve the above issue.
So user can set the option in the manifestwork, so when another actor on the spoke cluster update the resource. The
work-agent will ignore the change and not try to change the resource back. On the other hand, when the resource spec
defined in the manifestwork is updated by the user, the work-agent will still update the resource accordingly. In
summary, the option is to ignore the change triggered from spoke cluster only.

In addition, in `ServerSideApplly` strategy, we would also introduce a new `ignoreDifferences` options similar as what
is defined in argoCD (https://argo-cd.readthedocs.io/en/stable/user-guide/diffing/). Such that user can choose to not
let work-agent patch certain resource fields.

### Design Details

#### API change

The change will be added into the `updateStratey` field. For `Update` strategy, we will add an option

```go
type UpdateConfig struct {
    // OnSpokeChange defines whether the resource should be overriden by the manifestwork it is changed
    // on the spoke by another actor.
    // +kubebuilder:default=Override
    // +kubebuilder:validation:Enum=Override;NoOverride
    // +kubebuilder:validation:Required
    // +required
	OnSpokeChange string `json:"onSpokeChange,omitempty"`
}
```

For existing `ServerSideApply` strategy, we will add:

```go
type ServerSideApplyConfig struct {
	...
	// IgnoreFields defines a list of json paths in the resource that will not be updated on the spoke.
	IgnoreFields []string `json:"ignoreFields,omitempty"`

	// OnSpokeChange defines whether the resource should be overriden by the manifestwork it is changed
	// on the spoke by another actor.
	// +kubebuilder:default=Override
	// +kubebuilder:validation:Enum=Override;NoOverride
	// +kubebuilder:validation:Required
	// +required
	OnSpokeChange string `json:"onSpokeChange,omitempty"`
}
```

#### agent implemntation

When work-agent identity that the `OnSpokeChange` is set `NoOverride` for a certain resource in the `ManifestWork`, the agent
when apply the resource to the spoke cluster will add an annotation with the key `open-cluster-management.io/object-hash`.
The value of the annotation is the computed hash of the resource spec in the `ManifestWork`. Later when another actor
updates the resource, the work-agent will at first check if the object-hash mismatches with the current resource spec
in the `ManifestWork`. If it is the same, the resource will not be updated so the change from spoke is ignored. When
the resource in the manifestwork is updated, the annotation will not match which then trigger the work-agent to update.

To handle the `IgnoreFields` for `ServerSideApply`, we will remove the fields defined in the `IgnoreFields` and then
generates the apply patch. The objec-hash will also be computed considering the `IgnoreFields`. 

#### examples

To resolve issue 642, user can set the manifestwork with `OnSpokeChange` set to `NoOverride`

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  namespace: <target managed cluster>
  name: hello-work-demo
spec:
  workload: ...
  manifestConfigs:
    - resourceIdentifier:
        resource: secrets
        namespace: default
        name: some-secret
      updateStrategy:
        type: Update
        update:
          onSpokeChange: NoOverride
```

To resolve issue 670, user also can do the same for argocd application.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  namespace: <target managed cluster>
  name: hello-work-demo
spec:
  workload: ...
  manifestConfigs:
    - resourceIdentifier:
        group: argoproj.io/v1alpha1
        resource: application
        namespace: default
        name: application1
      updateStrategy:
        type: Update
        update:
          onSpokeChange: NoOverride
```

To resolve issue 690, user can set like:

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  namespace: <target managed cluster>
  name: hello-work-demo
spec:
  workload: ...
  manifestConfigs:
    - resourceIdentifier:
        group: apps/v1
        resource: deployment
        namespace: default
        name: deploy1
      updateStrategy:
        type: ServerSideApply
        serverSideApply:
          ignoreFields:
          - ".spec.replicas"
```


### Test Plan

- test on `OnSpokeChange` with `Overrid` and `NonOverride` option
- test on `IgnoreFields` with a single field, a full strcutrue and an item in the list.

### Graduation Criteria
N/A

### Upgrade Strategy
It will need upgrade on CRD of ManifestWork on hub cluster, and upgrade of work agent on managed cluster.

### Version Skew Strategy
- The field is optional, and if it is not set, the manifestwork will be updated as is by the work agent with elder version
- The elder version work agent will ignore the newly added field field.

## Alternatives
