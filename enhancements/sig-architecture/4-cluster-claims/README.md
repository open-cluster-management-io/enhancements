# Update ManagedCluster status to include ClusterInfo

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary

The proposed work would integrate the ClusterInfoStatus into ManagedCluster:

https://github.com/open-cluster-management/multicloud-operators-foundation/blob/master/pkg/apis/internal.open-cluster-management.io/v1beta1/clusterinfo_types.go#L22-L62

## Motivation

Today, several label properties, including clusterID, cloud and vendor, are updated on ManagedCluster objects that can be modified by users, creating invalid data for matching behavior. It would be a good idea to include cluster info provided by the Klusterlet in the status of ManagedCluster.

It is required from other components that more cluster specific info should be added into ManagedCluster API. This information includes the clusterID, the ocp distribution version, vendor and cloud provider etc. On another hand, ManagedCluster API should be designed to have a generic and extensible structure to include this info that also can work with different implementations.

### Goals

- Allow extensible info in managed cluster to be added in ManagedCluster API
- Remove cluster info labels from managed cluster eventually and use managed cluster status instead;
- Align wth upstream design;

### Non-Goals

- Replace and deprecate ManagedClusterInfo API

## Proposal

To align with the upstream proposal for ClusterID (https://docs.google.com/document/d/1S0u6xzP2gcJKPipA6tBNDNuid76nVKeGhTk7PrCIuQY/edit), leverage ClusterClaims created on managed cluster to store and manage cluster information.

### Risks and Mitigations

- The upstream proposal for ClusterID is not finalized yet and may change in the future.  
The mitigation is to create ClusterClaim CRD in group cluster.open-cluster-management.io with version v1alph1 for now. Once the proposal is finalized, support both of them for one or two releases and then deprecate the one in group cluster.open-cluster-management.io.
- There exists a debate on whether the `signature` field of ClusterClaim is necessary or not from upstream discussion. In this proposal, `signature` will not be included until there is a final decision.

## Design Details

As shown in the table below, ManagedClusterInfo includes 10 fields in its status. Some of them will NOT be merged into ManagedCluster.

| Name    | Description    | Comments   |
| ---- | ---- | ---- |
| Conditions | condition information for a managed cluster | **Not merged**. ManagedCluster has its own Conditions. |
| Version | the kube version of managed cluster | **Not merged**. ManagedCluster has a field with same name in its status, which also points to the kube version of managed cluster |
| KubeVendor | the kubernetes provider of the managed cluster| |
| CloudVendor | the cloud provider for the managed cluster | |
| ClusterID | the identifier of managed cluster | |
| DistributionInfo | the information about distribution of managed cluster | **Not merged**. It's product specific. |
| ConsoleURL | the url of console in managed cluster | |
| NodeList | a list of the status of nodes | **Not merged**. There is a scale issue for a cluster with a large number of nodes. |
| LoggingEndpoint | the endpoint to connect to logging server of managed cluster | **Not merged**. It's product specific. |
| LoggingPort | the port to connect to logging server of managed cluster | **Not merged**. It's product specific. |

ClusterClaim is a CRD which comes from the upstream proposal for ClusterID. It’s a cluster-scoped resource created on managed clusters and provides a way to store cluster scoped information. Its name is a well known or customized name to identify the claim. The following shows the definition of ClusterClaim.

```go
type ClusterClaim struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    // Spec defines the attributes of the ClusterClaim.
    Spec ClusterClaimSpec `json:"spec,omitempty"`
}

type ClusterClaimSpec struct {
    // Value is a claim-dependent string
    Value string `json:"value,omitempty"`

    // Signature may be used to validate a specific claim
    // +optional
    Signature string `json:"signature,omitempty"`
}
```

An example claim looks like,

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ClusterClaim
metadata:
  name: id.openshift.io
spec:
  value: 95f91f25-d7a2-4fc3-9237-2ef633d8451c
```

The following ClusterClaims might be created on a managed cluster. Some of them are immutable.

| Claim Name | Reserved | Mutable | Description |
| ---- | ---- | ---- | ---- |
| id.k8s.io | true | false | ClusterID defined in upstream proposal |
| kubeversion.open-cluster-management.io | true | true | Kubernetes version|
| platform.open-cluster-management.io | true | false | Platform the managed cluster is running on, like AWS, GCE, and Equinix Metal |
| product.open-cluster-management.io | true | false |  Product name, like OpenShift, Anthos, EKS and GKE |
| id.openshift.io | false | false | OpenShift external ID. It's available for OCP cluster only |
| consoleurl.openshift.io | false | true | URL of management console. It's available for OCP cluster only |
| version.openshift.io | false | true | OCP version. It's available for OCP cluster only |


And on hub side, add a list of ClusterClaims in ManagedClusterStatus.

```go
// ManagedClusterStatus represents the current status of joined managed cluster.
type ManagedClusterStatus struct {
    // Conditions contains the different condition statuses for this managed cluster.
    Conditions []metav1.Condition `json:"conditions"`

    // Capacity represents the total resource capacity from all nodeStatuses
    // on the managed cluster.
    Capacity ResourceList `json:"capacity,omitempty"`

    // Allocatable represents the total allocatable resources on the managed cluster.
    Allocatable ResourceList `json:"allocatable,omitempty"`

    // Version represents the kubernetes version of the managed cluster.
    Version ManagedClusterVersion `json:"version,omitempty"`

    // ClusterClaims represents cluster information that a managed cluster claims
    // +optional
    ClusterClaims []ManagedClusterClaim `json:"clusterClaims,omitempty"`
}

// ManagedClusterClaim represents a ClusterClaim collected from managed cluster.
type ManagedClusterClaim struct {
    // Name is a well known or custom name to identify the claim.
    Name string `json:"name,omitempty"`

    // Value is a claim-dependent string
    Value string `json:"value,omitempty"`
}
```

The registration agent running on managed cluster will sync ClusterClaims created on managed cluster to the status of ManagedCluster on hub once the cluster joins the hub. (The status of condition type ManagedClusterJoined becomes true.) To prevent from including too many ClusterClaims in a ManagedCluster, add a new parameter `cluster-claims-max` for registration agent with default value `20`. The registration agent starts sync from the reserved claims and then the customized ones. Some of the customized claims might not be synced once the total number of the ClusterClaims exceeds the value of `cluster-claims-max`.

A managed cluster with ClusterClaims looks like,

```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  labels:
    cloud: Amazon
    clusterID: 95f91f25-d7a2-4fc3-9237-2ef633d8451c
    installer.name: multiclusterhub
    installer.namespace: open-cluster-management
    name: cluster1
    vendor: OpenShift
  name: cluster1
spec:
  hubAcceptsClient: true
  leaseDurationSeconds: 60
status:
  allocatable:
    cpu: '15'
    memory: 65257Mi
  capacity:
    cpu: '18'
    memory: 72001Mi
  clusterClaims:
    - name: id.k8s.io
      value: cluster1
    - name: kubeversion.open-cluster-management.io
      value: v1.18.3+6c42de8
    - name: platform.open-cluster-management.io
      value: AWS
    - name: product.open-cluster-management.io
      value: OpenShift
    - name: id.openshift.io
      value: 95f91f25-d7a2-4fc3-9237-2ef633d8451c
    - name: consoleurl.openshift.io
      value: 'https://console-openshift-console.apps.xxxx.dev04.red-chesterfield.com'
    - name: version.openshift.io
      value: '4.5'
  conditions:
    - lastTransitionTime: '2020-10-26T07:08:49Z'
      message: Accepted by hub cluster admin
      reason: HubClusterAdminAccepted
      status: 'True'
      type: HubAcceptedManagedCluster
    - lastTransitionTime: '2020-10-26T07:09:18Z'
      message: Managed cluster joined
      reason: ManagedClusterJoined
      status: 'True'
      type: ManagedClusterJoined
    - lastTransitionTime: '2020-10-30T07:20:20Z'
      message: Managed cluster is available
      reason: ManagedClusterAvailable
      status: 'True'
      type: ManagedClusterConditionAvailable
  version:
    kubernetes: v1.18.3+6c42de8
```

With this design, ClusterClaim, like `id.openshift.io` and `version.openshift.io`, could be created by vendor-specific agent running on a managed cluster. And the registration agent can focus on the vendor-neutral logic only. It's flexible and easy to extend. For instance, create a ClusterClaim with name `clusterset.k8s.io` which describes its current ClusterSet membership.

## Alternatives

Merge the fields of ClusterInfoStatus into ManagedClusterStatus directly except
- Conditions
- Version
- DistributionInfo
- NodeList
- LoggingEndpoint
- LoggingPort 

```go
type ManagedClusterStatus struct {
    // Conditions contains the different condition statuses for this managed cluster.
    Conditions []metav1.Condition `json:"conditions"`

    // Capacity represents the total resource capacity from all nodeStatuses
    // on the managed cluster.
    Capacity ResourceList `json:"capacity,omitempty"`

    // Allocatable represents the total allocatable resources on the managed cluster.
    Allocatable ResourceList `json:"allocatable,omitempty"`

    // Version represents the kubernetes version of the managed cluster.
    Version ManagedClusterVersion `json:"version,omitempty"`

    // KubeVendor represents the kubernetes provider of the managed cluster.
    // +optional
    KubeVendor KubeVendorType `json:"kubeVendor,omitempty"`

    // CloudVendor represents the cloud provider for the managed cluster.
    // +optional
    CloudVendor CloudVendorType `json:"cloudVendor,omitempty"`

    // ClusterID is the identifier of managed cluster
    // +optional
    ClusterID string `json:"clusterID,omitempty"`

    // ConsoleURL shows the url of console in managed cluster
    // +optional
    ConsoleURL string `json:"consoleURL,omitempty"`
}
```
The values of those fields will be populated by the registration agent running on the managed cluster when it joins the hub.

With this design, registration agent have to deal with vendor-specific logic to collect fields, like CloudVendor and ClusterID. It's not extendable.