# Manifest Update Strategy in ManifestWork

## Release Signoff Checklist

- [] Enhancement is `implemented`
- [] Design details are appropriately documented from clear requirements
- [] Test plan is defined
- [] Graduation criteria for dev preview, tech preview, GA
- [] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

Today work agent do a fully update of each manifest defined in the `ManifestWork`. There is some special treatments for different resources and fields:

### `ApplyDirectly` for known types

`ApplyDirectly` in `openshift/library-go` is used for resources with known types, including those resources in core API group, CRDs,
APIServices etc. In general, `ApplyDirectly` will get the existing resources; compare the spec of existing and required, and update
the existing if there is diff. `Labels` and `Annotations` will be merged from required to the existing resource. 
In addition, there are special treatments for certain types. For example, if `Secret.type` is changed in the required resource, `ApplyDirectly` will delete the existing and create a new `Secret` as required, since `Secret.type` is an immutable field.

### Apply unstructured

If a resource is not in the known type for `ApplyDirectly`, the resource will be regarded as an unstrcuture object. `Labels` and `Annotations` will also be merged similar as `ApplyDirectly`. For fields not in `metadata` and `status`, work agent will do a
deepequal and fully update the resource if there is diff. Apply unstructured covers much more resource types including `apps` API
group and all the custom resources.

However apply unstructure method brings several issues for work agent:

#### Too many update calls

Kube APIServer will commonly set default values for resources, and there will always be diff when work agent compare existing with
required resource. It will cause many unnecessary update calls. For example, when a `deployment` is created, kube-apiserver will set
default service account for the deployment spec. Since the manifest defined in `ManifestWork` does not have this field, work-agent
will update deployment constantly.

#### Apply failure

Some field of the resources has both defaulting and immutable set, with which work agent is unable to update the resource. `Job` is an example. If user does not set `spec.Selector` of job, the kube-apiserver will set a default selector and label on pod. 
These fields are immutable, so work agent will fail later when it compares the job manifest and tries to update the job.

#### Fight with other actors

A resource might not only managed by work agent, but also other actors. Other actors would want to mutate some field in the resource
which will later be rolled back by the work agent. Some use cases may need these actors to control some fields of the resources other than work agent.

A possible solution to resolve the first two problems is that work-agent records the hash of required spec and the generation of the last applied resource, and instead of comparing required to the existing resources, work-agent compares hash (if required changes) and generation (if existing changes). However, it cannot resolve the third problem involving multiple actors.

## Motivation

Provide an interface for user to control how resource defined in `ManifestWork` should be updated.

## Proposal

We propose to update `ManifestWork` API so the user can specify upstrategy for each resource manifest in the `ManifestWork`. Possible strategy includes:

1. by default, work-agent update resource as what is implemented today.
2. user can choose not to update the resource, work-agent will only check the existence of the resource, but not keep its spec update.
3. user can choose to use server side apply when work-agent update the resource.

### Design Details

#### API change

We could add an updateStrategy for each manifest as below:

```go
type UpdateStrategy struct {
	// type defines the strategy to update this manifest, default value is Update.
	// Update type means to update resource by an update call.
	// CreateOnly type means do not update resource based on current manifest.
	// ServerSideApply type means to update resource using server side apply as fieldManager of work-controller.
	// If there is conflict, the related Applied confition of manifest will in the status of False with the
	// reason of ApplyConflict.
	// +kubebuilder:default=Update
	// +kubebuilder:validation:Enum=Update;CreateOnly;ServerSideApply
	// +kubebuilder:validation:Required
	// +required
	Type string `json:"type"`

	// serverSideApply defines the configuration for server side apply. It is honored only when
	// type of updateStrategy is ServerSideApply
	// +optional
	ServerSideApply *ServerSideApplyConfig `json:"serverSideApply,omitempty"`
}

type ServerSideApplyConfig struct {
	// Force represents to force apply the manifest.
	// +optional
	Force bool `json:"force"`
}
```

And example of the API would be:

```yaml
kind: ManifestWork
metadata:
  name: demo-work1
spec:
  workload:
    manifests:
    - apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: hello
        namespace: default
      spec:
        replica: 1
        selector:
          matchLabels:
            app: hello
        template:
          metadata:
            labels:
              app: hello
          spec:
            containers:
            - name: hello
              image: quay.io/asmacdo/busybox
              command: ['sh', '-c', 'echo "Hello, World!" && sleep 3600']
  manifestConfigs:
  - resourceIdentifier:
      group: apps
      resource: deployments
      name: hello
      namespace: default
    updateStrategy:
      type: ServerSideApply   
```

It means that the to create deployment `hello` on the managed cluster if it does not exist, and using server side apply to update
the resource if it exists on the managed cluster already.

#### Server side apply

With service side apply, it is possible to make work agent coordinate with other actors on the spoke. Take the above `ManifestWork` as an
example, if there is an HPA controller on the spoke that take the ownership of the `spec.replicas` field. The condition of `ManifestWork`
should show a status condition of applied failed as below:

```yaml
status:
  resourceStatus:
    manifests:
      - conditions:
        - lastTransitionTime: "2021-10-14T14:59:09Z"
          message: Manifest ownership conflict
          reason: ApplyManifestConflict
          status: "false"
          type: Applied
        - lastTransitionTime: "2021-10-14T14:59:09Z"
          message: Resource is available
          reason: ResourceAvailable
          status: "True"
          type: Available
        resourceMeta:
          group: apps
          kind: Deployment
          name: hello
          namespace: default
          ordinal: 0
          resource: deployments
          version: v1
```

When the user see this status condition, the user can update the `ManifestWork` by removing the replica field, which will turn the applied status condition to true again. The message of of `Applied` condition should have the details to tell user that the update of which field is conflict with which field manager. It should also show specific message when the spoke does not support server side apply.

```yaml
kind: ManifestWork
metadata:
  name: demo-work1
spec:
  workload:
    manifests:
    - apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: hello
        namespace: default
      spec:
        selector:
          matchLabels:
            app: hello
        template:
          metadata:
            labels:
              app: hello
          spec:
            containers:
            - name: hello
              image: quay.io/asmacdo/busybox
              command: ['sh', '-c', 'echo "Hello, World!" && sleep 3600']
  manifestConfigs:
  - resourceIdentifier:
      group: apps
      resource: deployments
      name: hello
      namespace: default
    updateStrategy:
      type: ServerSideApply   
```

Service side apply has some limitations comparing with legacy update strategy. User cannot remove a field in the resources with
server side apply in some scenarios. For example, if the merge stratefy defined for a certain field in api schema is `merge`,
user will not be able to remove an item in the list of this field.

#### Create Only

Create only strategy defines that work agent will only ensure the existence of the resource on the managed
cluster, and ignore the update of resource upon any change on the manifestwork. Compared with server side apply
strategy, create only strategy gives ownership of all the fields in the resource to the admin or controllers
on the managedcluster, while server side apply strategy has finer grained control on resource fiels ownership.
Create only strategy is useful in the older version cluster on which server side apply is not supported,
but the user on hub still want the controllers on the managed cluster to be able to managed resources
created by the manifestwork.


### Test Plan
- test different update strategy
- test update strategy in the manifestwork
- test resource apply conflict case and ensure that the message returned is correct

### Graduation Criteria
N/A

### Upgrade Strategy
It will need upgrade on CRD of ManifestWork on hub cluster, and upgrade of work agent on managed cluster.

### Version Skew Strategy
- The UpdateStrategy field is optional, and if it is not set, the manifestwork will be updated as is by the work agent with elder version
- The elder version work agent will ignore the UpdateStrategy field.

## Alternatives
