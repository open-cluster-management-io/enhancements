# Syn status of applied resources in ManifestWork

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

It is becoming an important requirement that the the user or operand on the hub cluster should be able get the real time status of the resources in managed clusters after these resources have been applied to the managed cluster via `ManifestWorks`. This requirement comes from the use cases including:

 1. A user would like to know the status of the resources applied to the managed cluster, e.g. the user wants to know how many replicas are running in a deployment applied in the ManagedCluster.
 2. A higher level controller would require the status to be synced back and collected in ocm. It then can aggregate the collected statuses and return a status summary to the user of the platform.
 3. A workload orchestrator would dispatch multiple dependent jobs as a pipeline or DAG across multiple clusters. It needs to get the status of the deployed job on the ocm control plane to decide the next step.

## Motivation

Provide a common approach for users or controllers on the hub cluster to get the status of resources applied by
`ManifestWork`. A orchestration controller on the hub cluster can use this to design a control loop to distribute
workloads.

The straightforward approach is to return `status` of arbitrary resources to the hub using `RawExtension`. However, it won't be possible to manage such untyped structure. The size of the whole status field of a resource can be too big to
be put in the `ManifestWork`. 

Given the current use cases, the controllers or users on the hub only care about certain fields of the applied resources. It may make more sense to explicitly specify the fields of the status in `ManifestWork` spec, and work agent
will only return the value of these fields.

### Goals

## Proposal

We propose to update `ManifestWork` API so the user can specify the status fields of the resources applied by
`ManifestWork`. The work-agent will sync these status fields to the `ManifestWork` API.

### Design Details

#### API change  

In the spec of `ManifestWork`, add a new field 

```go

// ManifestWorkSpec represents a desired configuration of manifests to be deployed on the managed cluster.
type ManifestWorkSpec struct {
    ...
	// StatusFeedback defines the option to return status of applied resource
	StatusFeedbacks []StatusFeedbackOption `json:"statusFeedbacks,omitempty"`
}

type StatusFeedbackOption struct {
	// ResourceIdentifier represents the group, resource, name and namespace of a resoure.
	// +required
	ResourceIdentifier ResourceIdentifier `json:"resourceIdentifier"`

	// FeedBackRules defines what resource status field should be returned.
	// +required
	FeedbackRules []FeedbackRule `json:"feedbackRule"`
}

type FeedbackRule struct {
	// Type defines the option of how status can be returned.
	// It can be StatusJsonPaths only or WellKnownStatus.
	// If the type is StatusJsonPaths, user should specify the jsonPaths field
	// If the type is WellKnownStatus, certain commont fields of status will be reported,
	// including Replicas, ReadyReplicas, and AvailableReplicas. And these status fields
	// do not exist, no values will be reported.
	// +required
	Type string `json:"type"`

	// InterestedFields defines the status fields to be synced for a manifest
	// An interstedValues will be updated in certain manifest status of the manifestwork.status.
	JsonPaths []JsonPath `json:"jsonPaths,omitempty"`
}

type JsonPath struct {
	// name represents the aliase name for this field
	// +required
	Name string `json:"name"`
	// JsonPaths represents the json path of the field under status.
	// The path must point to a field with single value in the type of integer, bool or string.
	// If the path points to a structure, map or slice, no value will be returned in ManifestWork.
	// Ref to https://kubernetes.io/docs/reference/kubectl/jsonpath/ on how to write a jsonPath
	// +required
	Path string `json:"path"`
}

```

An example of `ManifestWork` to deploy a deployment and return `availableReplica` and `available` condition will be like

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
  statusFeedbackOptions:
  - resourceIdentifier:
      group: apps
      resource: deployments
      name: hello
      namespace: default
    feedbackRules:
    - type: WellKonwnStatus
    - type: StatusJsonPaths
      jsonPaths:
      - name: availableCondition
        path: .conditions[?(@.type=="Available")].status   
```

The status of `ManifestWork` will be updated to add

```go

// ManifestCondition represents the conditions of the resources deployed on a
// managed cluster.
type ManifestCondition struct {
	// ResourceMeta represents the group, version, kind, name and namespace of a resoure.
	// +required
	ResourceMeta ManifestResourceMeta `json:"resourceMeta"`

	// StatusFeedback represents the values of the status feedback
	StatusFeedback StatusFeedback `json:"statusFeeback,omitempty"`

	// Conditions represents the conditions of this resource on a managed cluster.
	// +required
	Conditions []metav1.Condition `json:"conditions"`
}

type StatusFeedback struct {
    // InterestedValue represents the value of the interested field
	// If the relates interestedPath returns a empty, nil or non-single value,
	// The InterestedValue will not be added.
    InterestedValues []InterestedValue `json:"interestedValues"`
}

type InterestedValue struct {
    // name represents the aliase name for this field
	// +required
	Name string `json:"name"`

    // value represents the value of the field
	Value FieldValue `json:"fieldValue"`
}

type FieldValue struct {
  // Type is the type of the value, it can by integer, string or boolean
	Type ValueType `json:"type"`

	// +optional
	Integer int32 `json:"integer,omitempty"`

	// +optional
	String string `json:"string,omitempty"`

	// +optional
	Boolean bool `json:"boolean,omitempty"`
}

```

So the work agent will return a status with the above example as

```yaml
status:
  resourceStatus:
    manifests:
      - conditions:
        - lastTransitionTime: "2021-10-14T14:59:09Z"
          message: Apply manifest complete
          reason: AppliedManifestComplete
          status: "True"
          type: Applied
        - lastTransitionTime: "2021-10-14T14:59:09Z"
          message: Resource is available
          reason: ResourceAvailable
          status: "True"
          type: Available
        statusFeeback:
          interestedValues:
          - fieldValue:
              integer: 1
              type: integer
            name: availableReplica
          - fieldValue:
              integer: 1
              type: integer
            name: readyReplica
          - fieldValue:
              integer: 1
              type: integer
            name: replica
          - fieldValue:
              string: "True"
              type: string
            name: availableCondition
        resourceMeta:
          group: apps
          kind: Deployment
          name: hello
          namespace: default
          ordinal: 0
          resource: deployments
          version: v1
```

The returned value must be a scalar value, and the work agent should check the type of the returned value. The work agent should
treat the status that a value "ISNotFound" or "TypeMismatch" separately. If the path of the interestedValue is not found in the
status of the resource, this values should be ignored. If the path of the interestedValue is valid, but the type of the value is
not scalar, e.g a list or map, the condition of "StatusFeedbackAvailable" should be set false and a message shoud be added to
indicate that which intrestedValue cannot be obtained

#### Status update frequency

Ideally the work agent should have a informer for each applied resource to update interestedValue. It will need to
manage too many informers in the agent and also will result in frequent update on status of `ManifestWork`.
Instead, we will start with a periodic  update. User can specify the update frequency of interestedValues by setting a
`--status-update-frequency` flag on the work agent. we should make it configurable for different resource in the
api spec as the future work.

### Test Plan

E2E tests will be added to cover cases including:
- Invalid status fields.
- Status field updates.
- Add, update or remove an interested field

### Graduation Criteria
N/A

### Upgrade Strategy
It will need upgrade on CRD of ManifestWork on hub cluster, and upgrade of work agent on managed cluster.

### Version Skew Strategy
- The StatusFeedback field is optional, and if it is not set, the manifestwork can be correctly treated by work agent with elder version
- The elder version work agent will ignore the StatusFeedback field.

## Alternatives

### Return raw status data of the applied resource.

Return the whole status of a the applied resource as `runtime.RawExtension` in the status of `ManifestWork`. It might be
more user friendly since the status data will be the same as status of the applie resource. On the other hand, it will generate many data payload on hub cluster and many update call. It may also result in many unnecessary control loop in the controllers on the hub when watching the `ManifestWork`. 

### Directly access apiserver of managed cluster.

User or operand can directly access the apiserverr of managed cluster or access via proxy (e.g. [clusternet](https://github.com/clusternet/clusternet)) to get the resource status. However, it will requires the controllers on the hub to
watch multiple apiservers on the managed clusters. It needs the hub cluster to maintain credentials for each  managed cluster, and watch resources across multiple cluster is high resource consuming.
