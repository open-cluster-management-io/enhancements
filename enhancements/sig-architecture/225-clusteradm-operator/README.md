# Clusteradm Operator

## Release Signoff Checklist

- [] Enhancement is `provisional`
- [] Design details are appropriately documented from clear requirements
- [] Test plan is defined
- [] Graduation criteria for dev preview, tech preview, GA
- [] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal introduces the clusteradm operator, which will provide a declarative interface for managing multi-cluster environments. It wraps the imperative `clusteradm` CLI in a Kubernetes-native way, introducing the `MultiCluster` CRD to enable automated bootstrapping of Kubernetes clusters into a multicluster topology. Ongoing maintenance of the multi-cluster in the form of configuration patches and OCM version upgrades is also supported.

## Motivation

OCM's current installation procedure is imperative-only, requiring multiple manual operations, either via `clusteradm` or direct application of OCM custom resources to various Kubernetes clusters. The lack of a declarative interface for managing the lifecycle of an OCM multi-cluster has several limitations:

- Complexity around bootstrapping and maintenance
- Multi-cluster topology / configuration cannot be managed via GitOps
- Not aligned with Kubernetes best practices

### Goals

- Declarative lifecycle management of an OCM multi-cluster
- Simplify and streamline the OCM onboarding and maintenance experience

### Non-goals

- Extending or modifying the core OCM functionality in any way

## Proposal

Add a new component to the OCM ecosystem: the clusteradm operator. It will reconcile a new OCM `MultiCluster` CRD, which encapsulates all information required to bootstrap and maintain an OCM multi-cluster. The spec of a `MultiCluster` will contain all the same information traditionally provided to `clusteradm init` and `clusteradm join`. The `MultiCluster` API surface will feature 1:1 parity with the `clusteradm` CLI, including all optional flags. In some cases, e.g., kubeconfigs, the spec will contain references to secrets containing auxiliary data to avoid exposing sensitive information.

### User Stories

#### Story 1

As an infrastructure or platform engineer, I want to apply a custom resource to a Kubernetes cluster and have it bootstrap itself (plus zero or more additional Kubernetes clusters) into an OCM multi-cluster, so that I do not have to manually orchestrate a series of imperative CLI commands.

#### Story 2

As an infrastructure or platform engineer, I want to commit the definition of my OCM multi-cluster into version control, so that I can gate changes to it using code review and leverage a GitOps approach for managing its lifecycle and topology.

### Risks and Mitigation

1. People may not want to store kubeconfigs for their spoke clusters on the hub cluster. Especially in a non-EKS scenario where the kubeconfigs contain long lived tokens. This can be mitigated by continuing to expand OCM's workload identify capabilities to support native integration with Azure and GCP in alignment with security best practices. Spoke kubeconfigs _could_ also be deleted from the hub cluster following the initial bootstrapping of the multi-cluster, but that introduces many challeges regarding upgrades, reconfiguration, etc.

1. In certain environments, network connectivity from hub to spoke cannot be assumed. The clusteradm operator will **not** be able to bootstrap such spoke clusters automatically. However, additional metadata can be provided when these clusters are joined via `clusteradm join ... --klusterlet-annotation foo=bar`. The clusteradm operator on the hub can subsequently identify spoke clusters that were bootstrapped out-of-band and incorporate the health status of their corresponding `ManagedCluster` objects into the health of the overall `MultiCluster`.

## Design Details

### API change

No changes to the core OCM APIs are required.

### MultiCluster API

Current state of the v1alpha1 `MultiCluster` API, which is fully functional and meets all the capabilities outlined elsewhere in this enhancement proposal.

```go
package v1alpha1

import (
	"fmt"
	"sort"
	"time"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// MultiClusterSpec defines the desired state of MultiCluster.
type MultiClusterSpec struct {
	Hub              Hub               `json:"hub"`
	Spokes           []Spoke           `json:"spokes"`
	RegistrationAuth *RegistrationAuth `json:"registrationAuth,omitempty"`
}

// MultiClusterStatus defines the observed state of MultiCluster.
type MultiClusterStatus struct {
	Phase        string        `json:"phase,omitempty"`
	Conditions   []Condition   `json:"conditions,omitempty"`
	JoinedSpokes []JoinedSpoke `json:"joinedSpokes,omitempty"`
}

// ToComparable returns a deep copy of the MultiClusterStatus that's suitable for semantic comparison.
func (s *MultiClusterStatus) ToComparable(_ ...Condition) *MultiClusterStatus {
	comparable := s.DeepCopy()
	for i := range comparable.Conditions {
		comparable.Conditions[i].LastTransitionTime = metav1.Time{}
	}
	return comparable
}

// GetCondition returns the condition with the supplied type, if it exists.
func (s *MultiClusterStatus) GetCondition(cType string) *Condition {
	for _, c := range s.Conditions {
		if c.Type == cType {
			return &c
		}
	}
	return nil
}

// SetConditions sets the supplied conditions, adding net-new conditions and
// replacing any existing conditions of the same type. This is a no-op if all
// supplied conditions are identical (ignoring the last transition time) to
// those already set. If cover is false, existing conditions are not replaced.
func (s *MultiClusterStatus) SetConditions(cover bool, c ...Condition) {
	for _, new := range c {
		exists := false
		for i, existing := range s.Conditions {
			if existing.Type != new.Type {
				continue
			}
			if existing.Equal(new) {
				exists = true
				continue
			}
			exists = true
			if cover {
				s.Conditions[i] = new
			}
		}
		if !exists {
			s.Conditions = append(s.Conditions, new)
		}
	}
}

// Equal returns true if the status is identical to the supplied status,
// ignoring the LastTransitionTimes and order of statuses.
func (s *MultiClusterStatus) Equal(other *MultiClusterStatus) bool {
	if s == nil || other == nil {
		return s == nil && other == nil
	}

	if len(other.Conditions) != len(s.Conditions) {
		return false
	}

	sc := make([]Condition, len(s.Conditions))
	copy(sc, s.Conditions)

	oc := make([]Condition, len(other.Conditions))
	copy(oc, other.Conditions)

	// We should not have more than one condition of each type
	sort.Slice(sc, func(i, j int) bool { return sc[i].Type < sc[j].Type })
	sort.Slice(oc, func(i, j int) bool { return oc[i].Type < oc[j].Type })

	for i := range sc {
		if !sc[i].Equal(oc[i]) {
			return false
		}
	}

	return true
}

// Condition describes the state of a MultiCluster.
type Condition struct {
	metav1.Condition `json:",inline"`
	WantStatus       metav1.ConditionStatus `json:"wantStatus"`
}

// Equal returns true if the condition is identical to the supplied condition, ignoring the LastTransitionTime.
func (c Condition) Equal(other Condition) bool {
	return c.Type == other.Type && c.Status == other.Status && c.WantStatus == other.WantStatus &&
		c.Reason == other.Reason && c.Message == other.Message
}

// Hub provides specifications for an OCM hub cluster.
type Hub struct {
	// ClusterManager configuration.
	// +kubebuilder:default:={}
	ClusterManager *ClusterManager `json:"clusterManager,omitempty"`

	// If true, create open-cluster-management namespace, otherwise use existing one.
	// +kubebuilder:default:=true
	CreateNamespace bool `json:"createNamespace"`

	// If set, the hub will be reinitialized.
	Force bool `json:"force,omitempty"`

	// Kubeconfig details for the Hub cluster.
	Kubeconfig *Kubeconfig `json:"kubeconfig"`

	// Singleton control plane configuration. If provided, deploy a singleton control plane instead of clustermanager.
	// This is an alpha stage flag.
	SingletonControlPlane *SingletonControlPlane `json:"singleton,omitempty"`

	// APIServer is the API server URL for the Hub cluster. If provided, the hub will be joined
	// using this API server instead of the one in the obtained kubeconfig. This is useful when
	// using in-cluster kubeconfig when that kubeconfig would return an incorrect API server URL.
	APIServer *string `json:"apiServer,omitempty"`
}

// SingletonControlPlane is the configuration for a singleton control plane
type SingletonControlPlane struct {
	// The name of the singleton control plane.
	// +kubebuilder:default:="singleton-controlplane"
	Name string `json:"name"`

	// Helm configuration for the multicluster-controlplane Helm chart.
	// For now https://open-cluster-management.io/helm-charts/ocm/multicluster-controlplane is always used - no private registry support.
	// See: https://github.com/open-cluster-management-io/multicluster-controlplane/blob/main/charts/multicluster-controlplane/values.yaml
	Helm Helm `json:"helm"`
}

// Helm is the configuration for helm.
type Helm struct {
	// Raw, YAML-formatted Helm values.
	Values string `json:"values,omitempty"`

	// Comma-separated Helm values, e.g., key1=val1,key2=val2.
	Set []string `json:"set,omitempty"`

	// Comma-separated Helm JSON values, e.g., key1=jsonval1,key2=jsonval2.
	SetJSON []string `json:"setJson,omitempty"`

	// Comma-separated Helm literal STRING values.
	SetLiteral []string `json:"setLiteral,omitempty"`

	// Comma-separated Helm STRING values, e.g., key1=val1,key2=val2.
	SetString []string `json:"setString,omitempty"`
}

// ClusterManager is the configuration for a cluster manager.
type ClusterManager struct {
	// A set of comma-separated pairs of the form 'key1=value1,key2=value2' that describe feature gates for alpha/experimental features.
	// Options are:
	//  - AddonManagement (ALPHA - default=true)
	//  - AllAlpha (ALPHA - default=false)
	//  - AllBeta (BETA - default=false)
	//  - CloudEventsDrivers (ALPHA - default=false)
	//  - DefaultClusterSet (ALPHA - default=false)
	//  - ManagedClusterAutoApproval (ALPHA - default=false)
	//  - ManifestWorkReplicaSet (ALPHA - default=false)
	//  - NilExecutorValidating (ALPHA - default=false)
	//  - ResourceCleanup (BETA - default=true)
	//  - V1beta1CSRAPICompatibility (ALPHA - default=false)
	// +kubebuilder:default:="AddonManagement=true"
	FeatureGates string `json:"featureGates,omitempty"`

	// If set, the cluster manager operator will be purged and the open-cluster-management namespace deleted
	// when the MultiCluster CR is deleted.
	// +kubebuilder:default:=true
	PurgeOperator bool `json:"purgeOperator,omitempty"`

	// Resource specifications for all clustermanager-managed containers.
	Resources *ResourceSpec `json:"resources,omitempty"`

	// Version and image registry details for the cluster manager.
	// +kubebuilder:default:={}
	Source *OCMSource `json:"source,omitempty"`

	// If set, the bootstrap token will used instead of a service account token.
	UseBootstrapToken bool `json:"useBootstrapToken,omitempty"`
}

// OCMSource is the configuration for an OCM source.
type OCMSource struct {
	// The version of predefined compatible image versions (e.g. v0.6.0). Defaults to the latest released version.
	// You can also set "latest" to install the latest development version.
	// +kubebuilder:default:="default"
	BundleVersion string `json:"bundleVersion,omitempty"`

	// The name of the image registry serving OCM images, which will be used for all OCM components."
	// +kubebuilder:default:="quay.io/open-cluster-management"
	Registry string `json:"registry,omitempty"`
}

// Kubeconfig is the configuration for a kubeconfig.
type Kubeconfig struct {
	// A reference to an existing secret containing a kubeconfig.
	// Must be provided for remote clusters.
	// For same-cluster, must be provided unless InCluster is set to true.
	// +optional
	SecretReference *SecretReference `json:"secretReference,omitempty"`

	// If set, the kubeconfig will be read from the cluster.
	// Only applicable for same-cluster operations.
	// Defaults to false.
	// +optional
	InCluster bool `json:"inCluster,omitempty"`

	// The context to use in the kubeconfig file.
	Context string `json:"context,omitempty"`
}

// SecretReference describes how to retrieve a kubeconfig stored as a secret
type SecretReference struct {
	// The name of the secret.
	Name string `json:"name"`

	// The namespace the secret is in.
	Namespace string `json:"namespace"`

	// The map key to access the kubeconfig.
	// Leave empty to use 'kubeconfig'.
	// +optional
	KubeconfigKey *string `json:"kubeconfigKey,omitempty"`
}

// Spoke provides specifications for joining and potentially upgrading spokes.
type Spoke struct {
	// The name of the spoke cluster.
	// +kubebuilder:validation:MaxLength=63
	// +kubebuilder:validation:Pattern=^[a-z0-9]([-a-z0-9]*[a-z0-9])?$
	Name string `json:"name"`

	// If true, create open-cluster-management namespace and agent namespace (open-cluster-management-agent for Default mode,
	// <klusterlet-name> for Hosted mode), otherwise use existing one.
	// +kubebuilder:default:=true
	CreateNamespace bool `json:"createNamespace"`

	// If true, sync the labels from klusterlet to all agent resources.
	SyncLabels bool `json:"syncLabels,omitempty"`

	// Kubeconfig details for the Spoke cluster.
	Kubeconfig *Kubeconfig `json:"kubeconfig"`

	// Hub cluster CA certificate, optional
	Ca string `json:"ca,omitempty"`

	// Proxy CA certificate, optional
	ProxyCa string `json:"proxyCa,omitempty"`

	// URL of a forward proxy server used by agents to connect to the Hub cluster.
	ProxyURL string `json:"proxyUrl,omitempty"`

	// Klusterlet configuration.
	// +kubebuilder:default:={}
	Klusterlet Klusterlet `json:"klusterlet,omitempty"`

	// ClusterARN is the ARN of the spoke cluster.
	// This field is optionally used for AWS IRSA registration authentication.
	ClusterARN string `json:"clusterARN,omitempty"`
}

// JoinType returns a status condition type indicating that a particular Spoke cluster has joined the Hub.
func (s *Spoke) JoinType() string {
	return fmt.Sprintf("spoke-cluster-%s-joined", s.conditionName())
}

func (s *Spoke) conditionName() string {
	name := s.Name
	if len(name) > 42 {
		name = name[:42] // account for extra 21 chars in the condition type (max. total of 63)
	}
	return name
}

// JoinedSpoke represents a spoke that has been joined to a hub.
type JoinedSpoke struct {
	// The name of the spoke cluster.
	Name string `json:"name"`

	// Kubeconfig details for the Spoke cluster.
	Kubeconfig *Kubeconfig `json:"kubeconfig"`

	// If set, the klusterlet operator will be purged and all open-cluster-management namespaces deleted
	// when the klusterlet is unjoined from its Hub cluster.
	// +kubebuilder:default:=true
	PurgeKlusterletOperator bool `json:"purgeKlusterletOperator,omitempty"`
}

// UnjoinType returns a status condition type indicating that a particular Spoke cluster has been removed from the Hub.
func (j *JoinedSpoke) UnjoinType() string {
	return fmt.Sprintf("spoke-cluster-%s-unjoined", j.conditionName())
}

func (j *JoinedSpoke) conditionName() string {
	name := j.Name
	if len(name) > 40 {
		name = name[:40] // account for extra 23 chars in the condition type (max. total of 63)
	}
	return name
}

// Klusterlet is the configuration for a klusterlet.
type Klusterlet struct {
	// A set of comma-separated pairs of the form 'key1=value1,key2=value2' that describe feature gates for alpha/experimental features.
	// Options are:
	//  - AddonManagement (ALPHA - default=true)
	//  - AllAlpha (ALPHA - default=false)
	//  - AllBeta (BETA - default=false)
	//  - ClusterClaim (ALPHA - default=true)
	//  - ExecutorValidatingCaches (ALPHA - default=false)
	//  - RawFeedbackJsonString (ALPHA - default=false)
	//  - V1beta1CSRAPICompatibility (ALPHA - default=false)
	// +kubebuilder:default:="AddonManagement=true,ClusterClaim=true"
	FeatureGates string `json:"featureGates,omitempty"`

	// Deployent mode for klusterlet
	// +kubebuilder:validation:Enum=Default;Hosted
	// +kubebuilder:default:="Default"
	Mode string `json:"mode,omitempty"`

	// If set, the klusterlet operator will be purged and all open-cluster-management namespaces deleted
	// when the klusterlet is unjoined from its Hub cluster.
	// +kubebuilder:default:=true
	PurgeOperator bool `json:"purgeOperator,omitempty"`

	// If true, the installed klusterlet agent will start the cluster registration process by looking for the
	// internal endpoint from the public cluster-info in the Hub cluster instead of using hubApiServer.
	ForceInternalEndpointLookup bool `json:"forceInternalEndpointLookup,omitempty"`

	// External managed cluster kubeconfig, required if using hosted mode.
	ManagedClusterKubeconfig *Kubeconfig `json:"managedClusterKubeconfig,omitempty"`

	// If true, the klusterlet accesses the managed cluster using the internal endpoint from the public
	// cluster-info in the managed cluster instead of using managedClusterKubeconfig.
	ForceInternalEndpointLookupManaged bool `json:"forceInternalEndpointLookupManaged,omitempty"`

	// Resource specifications for all klusterlet-managed containers.
	Resources *ResourceSpec `json:"resources,omitempty"`

	// If true, deploy klusterlet in singleton mode, with registration and work agents running in a single pod.
	// This is an alpha stage flag.
	Singleton bool `json:"singleton,omitempty"`

	// Version and image registry details for the klusterlet.
	// +kubebuilder:default:={}
	Source *OCMSource `json:"source,omitempty"`
}

// ResourceSpec defines resource limits and requests for all managed clusters.
type ResourceSpec struct {
	// The resource limits of all the containers managed by the Cluster Manager or Klusterlet operators.
	Limits ResourceValues `json:"limits,omitempty"`

	// The resource requests of all the containers managed by the Cluster Manager or Klusterlet operators.
	Requests ResourceValues `json:"requests,omitempty"`

	// The resource QoS class of all the containers managed by the Cluster Manager or Klusterlet operators.
	// One of Default, BestEffort or ResourceRequirement.
	// +kubebuilder:validation:Enum=Default;BestEffort;ResourceRequirement
	// +kubebuilder:default:="Default"
	QosClass string `json:"qosClass"`
}

// ResourceValues detail container resource constraints.
type ResourceValues struct {
	// The number of CPU units to request, e.g., '800m'.
	CPU string `json:"cpu,omitempty"`

	// The amount of memory to request, e.g., '8Gi'.
	Memory string `json:"memory,omitempty"`
}

// String returns a string representation of the resource values.
func (r *ResourceValues) String() string {
	if r.CPU != "" && r.Memory != "" {
		return fmt.Sprintf("cpu=%s,memory=%s", r.CPU, r.Memory)
	} else if r.CPU != "" {
		return fmt.Sprintf("cpu=%s", r.CPU)
	} else if r.Memory != "" {
		return fmt.Sprintf("memory=%s", r.Memory)
	}
	return ""
}

// RegistrationAuth provides specifications for registration authentication.
type RegistrationAuth struct {
	// The registration authentication driver to use.
	// Options are:
	//  - csr: Use the default CSR-based registration authentication.
	//  - awsirsa: Use AWS IAM Role for Service Accounts (IRSA) registration authentication.
	// The set of valid options is open for extension.
	// +kubebuilder:validation:Enum=csr;awsirsa
	// +kubebuilder:default:="csr"
	Driver string `json:"driver"`

	// The Hub cluster ARN for awsirsa registration authentication. Required when Type is awsirsa, otherwise ignored.
	HubClusterARN string `json:"hubClusterARN,omitempty"`

	// List of AWS EKS ARN patterns so any EKS clusters with these patterns will be auto accepted to join with hub cluster.
	// Example pattern: "arn:aws:eks:us-west-2:123456789013:cluster/.*"
	AutoApprovedARNPatterns []string `json:"autoApprovedARNPatterns,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="PHASE",type=string,JSONPath=`.status.phase`
// +kubebuilder:printcolumn:name="AGE",type=date,JSONPath=".metadata.creationTimestamp"

// MultiCluster is the Schema for the multiclusters API.
type MultiCluster struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   MultiClusterSpec   `json:"spec,omitempty"`
	Status MultiClusterStatus `json:"status,omitempty"`
}

// GetDriver returns the registration auth type, defaults to csr.
func (ra *RegistrationAuth) GetDriver() string {
	if ra == nil {
		// default registration auth type
		return CSRRegistrationDriver
	}
	return ra.Driver
}

// GetCondition gets the condition with the supplied type, if it exists.
func (m *MultiCluster) GetCondition(cType string) *Condition {
	return m.Status.GetCondition(cType)
}

// SetConditions sets the supplied conditions on a MultiCluster, replacing any existing conditions.
func (m *MultiCluster) SetConditions(cover bool, c ...Condition) {
	m.Status.SetConditions(cover, c...)
}

// +kubebuilder:object:root=true

// MultiClusterList contains a list of MultiCluster.
type MultiClusterList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []MultiCluster `json:"items"`
}

func init() {
	SchemeBuilder.Register(&MultiCluster{}, &MultiClusterList{})
}
```

### Test Plan

The following scenarios will be covered via E2E tests:

- Installation
  - Hub-as-spoke
  - Dedicated hub + spoke (IaaS)
  - Dedicated hub + spoke (EKS)
- Patches/Upgrades
  - Feature gates (not actually supported at this time, requires feature implementation in `clusteradm` first)
  - Helm values (singleton)
  - Upgrade OCM bundle version
- Uninstallation

### Graduation Criteria

TBD

### Upgrade Strategy

Upgrades for the clusteradm operator itself can be conducted using Helm. The `MultiCluster` CRD on hub cluster and the operator image will be upgraded, along with any additional configuration options added or modified between releases.

### Version Skew Strategy

- As the operator matures, the [webhook conversion](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/#webhook-conversion) strategy will be utilized to handle transformations between stored and served versions of the `MultiCluster` CRD.
- Not applicable for core OCM APIs. Any version skew will be inherited from OCM bundle versions.
