# Managed cluster addons management

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary

Using [open-cluster-management registration](https://github.com/open-cluster-management/registration)
to uniformly manage the lease update and registration of all managed cluster addons.

## Motivation

Currently each managed cluster addon maintains its lease by itself on the hub cluster. Although
each addon lease update request is small, once the scale of clusters becomes large, it will result
in a performance problem, for example, if we have 7 addons and there are 1000 clusters, there will
be 7000 lease update requests from managed clusters to the hub cluster every minutes.

And another problem is that each managed cluster addon uses its hub service account token to
communicate with the hub cluster. Once the scale of clusters becomes large, this result in that a
large number of service accounts are created on the hub cluster, for example, if we have 7 addons
and there are 1000 clusters, there will be 7000 service accounts on the hub cluster.

To resolve these two performance issues, we propose to introduce a uniform way to manage the lease
update and registration of all managed cluster addons.

### Goals

- Using registration to manage managed cluster addons lease update to reduce addon lease update
  requests
- Using registration to provide a common way for managed cluster addons registration instead of
  current service account mechanism

## Proposal

We want to follow up the [addon framework proposal](https://github.com/open-cluster-management/enhancements/issues/8)
to use open-cluster-management registration to manage the lease update and registration of all
managed cluster addons.

## Design Details

### Lease update

We use registration agent to maintain the status of addons, each addon maintains its lease on
managed cluster, the registration agent watches the lease of addons on managed cluster, if addons
update their leases constantly, the registration agent will maintain the status of the 
`managedclusteraddons` corresponding to the addons to `available` on the hub cluster, once one
addon stops to update its lease, the registration agent will update the status of the
`managedclusteraddons` corresponding to the addon to `unavailable` on the hub cluster.

The registration agent maintains the status of managed cluster on the hub cluster, so the registration
hub controller watches the managed cluster status, if the status of managed cluster is `unknown`,
the registration hub controller will update the status of all addons to `unknown` on the hub cluster.

By this way, we can eliminate the addon lease update requests from managed clusters to hub cluster.

### Addon registration

We use registration to support managed cluster addons registration, after one addon is registered,
the addon can access to kube-apiserver with kube style API or other endpoints on hub cluster with
client certificate authentication.

The `ManagedClusterAddOns` API will be enhanced to record managed cluster addon configuration
information

- Add addon installation namespace to `ManagedClusterAddOns` spec
- Add addon registration information to `ManagedClusterAddOns` status

```go
// ManagedClusterAddOn is the Custom Resource object which holds the current state
// of an add-on. This object is used by add-on operators to convey their state.
// This resource should be created in the ManagedCluster namespace.
type ManagedClusterAddOn struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata"`

	// spec holds configuration that could apply to any operator.
	// +kubebuilder:validation:Required
	// +required
	Spec ManagedClusterAddOnSpec `json:"spec"`

	// status holds the information about the state of an operator.  It is consistent with status information across
	// the Kubernetes ecosystem.
	// +optional
	Status ManagedClusterAddOnStatus `json:"status"`
}

// ManagedClusterAddOnSpec defines the install configuration of
// an addon agent on managed cluster.
type ManagedClusterAddOnSpec struct {
	// installNamespace is the namespace on the managed cluster to install the addon agent.
	// If it is not set, open-cluster-management-agent-addon namespace is used to install the addon agent.
	// +optional
	// +kubebuilder:validation:MaxLength=63
	// +kubebuilder:validation:Pattern=^[a-z0-9]([-a-z0-9]*[a-z0-9])?$
	InstallNamespace string `json:"installNamespace,omitempty"`
}

// RegistrationConfig defines the configuration of the addon agent to register to hub. The Klusterlet agent will
// create a csr for the addon agent with the registrationConfig.
type RegistrationConfig struct {
	// signerName is the name of signer that addon agent will use to create csr.
	// +required
	// +kubebuilder:validation:MaxLength=571
	// +kubebuilder:validation:MinLength=5
	SignerName string `json:"signerName"`

	// subject is the user subject of the addon agent to be registered to the hub.
	// If it is not set, the addon agent will have the default subject
	// "subject": {
	//	"user": "system:open-cluster-management:addon:{addonName}:{clusterName}:{agentName}",
	//	"groups: ["system:open-cluster-management:addon", "system:open-cluster-management:addon:{addonName}", "system:authenticated"]
	// }
	//
	// +optional
	Subject Subject `json:"subject,omitempty"`
}

// Subject is the user subject of the addon agent to be registered to the hub.
type Subject struct {
	// user is the user name of the addon agent.
	User string `json:"user"`

	// groups is the user group of the addon agent.
	// +optional
	Groups []string `json:"groups,omitempty"`

	// organizationUnit is the ou of the addon agent
	// +optional
	OrganizationUnits []string `json:"organizationUnit,omitempty"`
}

// ManagedClusterAddOnStatus provides information about the status of the operator.
// +k8s:deepcopy-gen=true
type ManagedClusterAddOnStatus struct {
	// conditions describe the state of the managed and monitored components for the operator.
	// +patchMergeKey=type
	// +patchStrategy=merge
	// +optional
	Conditions []metav1.Condition `json:"conditions,omitempty"  patchStrategy:"merge" patchMergeKey:"type"`

	// relatedObjects is a list of objects that are "interesting" or related to this operator. Common uses are:
	// 1. the detailed resource driving the operator
	// 2. operator namespaces
	// 3. operand namespaces
	// 4. related ClusterManagementAddon resource
	// +optional
	RelatedObjects []ObjectReference `json:"relatedObjects,omitempty"`

	// addOnMeta is a reference to the metadata information for the add-on.
	// This should be same as the addOnMeta for the corresponding ClusterManagementAddOn resource.
	// +optional
	AddOnMeta AddOnMeta `json:"addOnMeta"`

	// addOnConfiguration is a reference to configuration information for the add-on.
	// This resource is use to locate the configuration resource for the add-on.
	// +optional
	AddOnConfiguration ConfigCoordinates `json:"addOnConfiguration"`

	// registrations is the conifigurations for the addon agent to register to hub. It should be set by each addon controller
	// on hub to define how the addon agent on managedcluster is registered. With the registration defined,
	// The addon agent can access to kube apiserver with kube style API or other endpoints on hub cluster with client
	// certificate authentication. A csr will be created per registration configuration. If more than one
	// registrationConfig is defined, a csr will be created for each registration configuration. It is not allowed that
	// multiple registrationConfigs have the same signer name. After the csr is approved on the hub cluster, the klusterlet
	// agent will create a secret in the installNamespace for the registrationConfig. If the signerName is
	// "kubernetes.io/kube-apiserver-client", the secret name will be "{addon name}-hub-kubeconfig" whose contents includes
	// key/cert and kubeconfig. Otherwise, the secret name will be "{addon name}-{signer name}-client-cert" whose contents includes key/cert.
	// +optional
	Registrations []RegistrationConfig `json:"registrations,omitempty"`
}

const (
	// ManagedClusterAddOnConditionAvailable represents that the addon agent is running on the managed cluster
	ManagedClusterAddOnConditionAvailable string = "Available"

	// ManagedClusterAddOnConditionDegraded represents that the addon agent is providing degraded service on
	// the managed cluster.
	ManagedClusterAddOnConditionDegraded string = "Degraded"
)

// ObjectReference contains enough information to let you inspect or modify the referred object.
type ObjectReference struct {
	// group of the referent.
	// +kubebuilder:validation:Required
	// +required
	Group string `json:"group"`
	// resource of the referent.
	// +kubebuilder:validation:Required
	// +required
	Resource string `json:"resource"`
	// name of the referent.
	// +kubebuilder:validation:Required
	// +required
	Name string `json:"name"`
}
```

If one managed cluster addon needs to register it to the hub cluster, the addon developers need to
implement an addon hub controller based on addon-framework library for their addon.

The addon-framework library provides an interface to help addon hub controller to save the
addon configuration information to its corresponding `managedclusteraddons`.

On the managed cluster, the registration agent watches the hub cluster `managedclusteraddons`.
The registration agent follows next steps to register an addon

1. The registration agent creates a CSR request with its own hub kubeconfig to register the addon
   to the hub cluster
2. On the hub cluster, the registration hub controller approves the CSR request automatically (the
   addon-framework library provides the interface, the addon hub controller can implement it to
   approve its CSR automatically)
3. After the CSR request was approved on the hub cluster, the registration agent gets the certificate
   from the CSR request to establish the hub kubeconfig and save the hub kubeconfig to a secret in
   the managed cluster addon namespace. The addon will mount the secret to get the hub kubeconfig
   to connect with the hub cluster

When the certificate of managed cluster addon is about to expire, the registration agent will send
a request to rotate the certificate on the hub cluster, the registration hub controller will auto
approve the certificate rotation request.

Taking observability addon as an example, its `managedclusteraddons` will be like

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: observability
  namespace: cluster1
spec:
  installNamespace: open-cluster-management-addon-observability
status:
  registrations:
  - signerName: "kubernetes.io/kube-apiserver-client"
  - signerName: "open-cluster-management.io/observability-signer"
    subject:
      user: open-cluster-management-observability-cluster1
      groups:
      - open-cluster-management-observability
      extras:
        organizationalUnit: acm
```

In this example, the observability addon requires two CSRs. One to access hub kube-api (with singer
name `kubernetes.io/kube-apiserver-client`), and another to access its own endpoint (with customized
singer name `open-cluster-management.io/observability-signer`).

After the two CSRs are created on the hub cluster, the observability hub addon controller will check
the signer, group and subject of the CSRs to verify whether the CSR is valid, if all fields are valid,
the controller will approve the CSR.

### Test Plan

#### Unit Tests

All controller implementations will have thorough coverage of unit tests.

#### Integration Tests
Considering certificate operation, we will use integration tests to coverage addon registration
test cases, includes:

- Register an addon with kube singer
- Register an addon with custom signer
- Update an addon registration configuration
- Addon client certification rotation

#### E2E Tests

Using e2e tests to coverage addon lease update test cases, includes:

- Maintain addon status to available if addon update its lease constantly
- Update addon status to unavailable if addon stops to update its lease
- Update addon status to unknown if there is no lease for this addon
- Update addon status to unknown if managed cluster becomes to unknown

### Graduation Criteria

The registration feature gate (`AddonManagement`) will initially be in a default-disabled alpha state.
After graduating to GA, we will make the feature enabled by default.

#### GA Graduation

- All design are implemented
- Address known bugs and add tests to prevent regressions
- Scalability requirement is met, when the cluster size is expanded to 1000 clusters, the
  performance will not degrade significantly
- This design is widely adopted by managed cluster addons
- Docs are up-to-date with latest implemented

### Upgrade Strategy

- The `ManagedClusterAddOn` API should be updated on the hub cluster
- For lease update, each addon needs to create and update its lease on its installation namespace
  on the managed cluster
- For each addon, if it needs to register to hub cluster, it needs to leverage addon framework to
  recode its registration configuration information to the its `ManagedClusterAddOn` CR

### Version Skew Strategy

For addon registration part, it is a new feature, it will not affect the addons of the old version.

For lease update part, the registration will keep compatibility with 2.2, and after 2.3, the
compatibility of 2.2 will not be maintained.
