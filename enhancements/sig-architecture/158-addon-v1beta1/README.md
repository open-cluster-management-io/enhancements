# Addon API v1beta1

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal outlines the evolution of the Open Cluster Management addon APIs from `v1alpha1` to `v1beta1`. The v1beta1 API design addresses lessons learned from v1alpha1, removes deprecated fields, improves the addon configuration, and provides a cleaner, more maintainable API surface for addon registration.

## Motivation

The current `v1alpha1` addon APIs (`ClusterManagementAddOn` and `ManagedClusterAddOn`) have accumulated deprecated fields and design decisions that need refinement based on real-world usage and community feedback. All of the change should be addressed in `v1beta1`.

### Goals

- **Clean up deprecated fields**: Remove fields that have been deprecated by better alternatives
- **Addon configuration**: Rename `spec.supportedConfigs` to `spec.defaultConfigs` to better reflect its purpose
- **Addon registration**: Redesign `status.registrations` to be more flexible for future addon registration patterns
- **backward compatibility**: The v1beta1 API will maintain backward compatibility through conversion webhooks

### Non-Goals

- The migration plan from v1alpha1 to v1beta1, like when to introduce v1beta1, when to remove v1alpha1 and when to change the CRD served and stored version.

## Proposal

### User Stories

#### Story 1: Addon Developer Migration
As an addon developer, I can develop the addon using either `v1alpha1` or `v1beta1` APIs. And I can migrate my addon from v1alpha1 to v1beta1 APIs without breaking existing deployments.

#### Story 2: OCM Hub Cluster Operator
As a hub cluster operator, I can use v1beta1 APIs for new addon deployments while continuing to support existing v1alpha1 addons.

#### Story 3: Cluster Administrator
As a cluster administrator, I can configure an addon using either `v1alpha1` or `v1beta1` API.

## Design Details

### ClusterManagementAddOn defaultConfigs

#### Key Change

`ClusterManagementAddOn` `spec.supportedConfigs` -> `spec.defaultConfigs`

#### API Definition

The `spec.defaultConfigs` does **NOT** support defining multiple configs with the Same GVK.

```golang
// ClusterManagementAddOnSpec provides information for the add-on.
type ClusterManagementAddOnSpec struct {
    ...
    // defaultConfigs is a list of default add-on configurations.
    // In scenarios where all add-ons have the same configuration, it 
    // can be overridden by install strategy configs or ManagedClusterAddOn configs.
  	// +optional
  	// +listType=map
  	// +listMapKey=group
  	// +listMapKey=resource
  	DefaultConfigs []AddOnConfig `json:"defaultConfigs,omitempty"`
    ...
}

// AddOnConfig represents a configuration for the add-on.
type AddOnConfig struct {
	// group and resource of add-on configuration.
	ConfigGroupResource `json:",inline"`

	// name and namespace of add-on configuration.
	ConfigReferent `json:",inline"`
}

// ConfigGroupResource represents the GroupResource of the add-on configuration
type ConfigGroupResource struct {
	// group of the add-on configuration.
	// +optional
	// +kubebuilder:default=""
	Group string `json:"group"`

	// resource of the add-on configuration.
	// +required
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:MinLength:=1
	Resource string `json:"resource"`
}

// ConfigReferent represents the namespace and name for an add-on configuration.
type ConfigReferent struct {
	// namespace of the add-on configuration.
	// If this field is not set, the configuration is in the cluster scope.
	// +optional
	Namespace string `json:"namespace,omitempty"`

	// name of the add-on configuration.
	// +required
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:MinLength:=1
	Name string `json:"name"`
}
```

#### Conversion Webhook Design
- The conversion webhook will convert v1alpha1 `spec.supportedConfigs` to v1beta1 `spec.defaultConfigs`.
- `__reserved_no_default__` as a **reserved name** will be used in `spec.defaultConfigs.name` to store the converted v1alpha1 API without default configs.

For example:

**Case 1: `ClusterManagementAddOn` `spec.supportedConfigs` is defined in v1alpha1 without defaultConfig**

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: application-manager
spec:
  supportedConfigs:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
---
apiVersion: addon.open-cluster-management.io/v1beta1
kind: ClusterManagementAddOn
metadata:
  name: application-manager
spec:
  defaultConfigs:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: __reserved_no_default__
```

**Case 2: `spec.supportedConfigs.defaultConfig` is defined in v1alpha1**

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: application-manager
spec:
  supportedConfigs:
  - defaultConfig:
      name: default-deploy-config
      namespace: open-cluster-management
    group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
---
apiVersion: addon.open-cluster-management.io/v1beta1
kind: ClusterManagementAddOn
metadata:
  name: application-manager
spec:
  defaultConfigs:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: default-deploy-config
    namespace: open-cluster-management
```

### ManagedClusterAddon registrations

#### Key Change
Redesign `status.registrations` to be more flexible for future addon registration patterns in v1beta1.

The v1alpha1 `RegistrationConfig` uses a flat structure with `signerName`, `subject`, and `driver` fields. In v1beta1, this is redesigned into a typed structure with two registration types: `kubeClient` and `customSigner`.

#### API Definition

**v1beta1 Structure:**

```golang
// RegistrationType represents the type of registration configuration for the addon agent.
type RegistrationType string

const (
	// kubeClient represents the registration type for addon agents that need to access
	// the hub kube-apiserver using kubeClient.
	KubeClient RegistrationType = "kubeClient"

	// customSigner represents the registration type for addon agents that need to access non-kube endpoints
	// on the hub cluster with client certificate authentication.
	CustomSigner RegistrationType = "customSigner"
)

// RegistrationConfig defines the configuration for the addon agent to register to the hub cluster.
type RegistrationConfig struct {
	// type specifies the type of registration configuration.
	// +kubebuilder:validation:Enum=kubeClient;customSigner
	// +required
	Type RegistrationType `json:"type"`

	// kubeClient holds the configuration for kubeClient type registration.
	// It should be set when type is "kubeClient".
	// +optional
	KubeClient *KubeClientConfig `json:"kubeClient,omitempty"`

	// customSigner holds the configuration for customSigner type registration.
	// It should be set when type is "customSigner".
	// +optional
	CustomSigner *CustomSignerConfig `json:"customSigner,omitempty"`
}

type KubeClientConfig struct {
	// subject is the user subject of the addon agent to be registered to the hub.
	// +optional
	Subject KubeClientSubject `json:"subject,omitempty"`

	// driver is the authentication driver used by managedclusteraddon for kubeClient registration. Possible values are csr and token.
	// This field is set by the agent to declare which driver it is using.
	// +optional
	// +kubebuilder:validation:Enum=csr;token
	Driver string `json:"driver,omitempty"`
}

type CustomSignerConfig struct {
	// signerName is the name of signer that addon agent will use to create csr.
	// +required
	// +kubebuilder:validation:MaxLength=571
	// +kubebuilder:validation:MinLength=5
	// +kubebuilder:validation:Pattern=^([a-z0-9][a-z0-9-]*[a-z0-9]\.)+[a-z]+\/[a-z0-9-\.]+$
	SignerName string `json:"signerName"`

	// subject is the user subject of the addon agent to be registered to the hub.
	// +optional
	Subject Subject `json:"subject,omitempty"`
}

// BaseSubject contains the common fields for addon agent subjects.
type BaseSubject struct {
	// user is the user name of the addon agent.
	User string `json:"user"`

	// groups is the user group of the addon agent.
	// +optional
	Groups []string `json:"groups,omitempty"`
}

type KubeClientSubject struct {
	BaseSubject `json:",inline"`
}

// subject is the user subject of the addon agent to be registered to the hub.
// If it is not set, the addon agent will have the default subject
//
//	"subject": {
//	  "user": "system:open-cluster-management:cluster:{clusterName}:addon:{addonName}:agent:{agentName}",
//	  "groups: ["system:open-cluster-management:cluster:{clusterName}:addon:{addonName}"]
//	}
type Subject struct {
	BaseSubject `json:",inline"`

	// organizationUnits is the ou of the addon agent
	// +optional
	OrganizationUnits []string `json:"organizationUnits,omitempty"`
}
```

#### Conversion Webhook Design

**v1alpha1 → v1beta1 Conversion:**
- If `signerName == "kubernetes.io/kube-apiserver-client"`:
  - Set `type = "kubeClient"`
  - Create `KubeClientConfig` with `subject` (user, groups) and `driver`
- Otherwise:
  - Set `type = "customSigner"`
  - Create `CustomSignerConfig` with original `signerName` and full `subject` (user, groups, organizationUnits)

**v1beta1 → v1alpha1 Conversion:**
- If `type == "kubeClient"`:
  - Set `signerName = "kubernetes.io/kube-apiserver-client"`
  - Copy `subject.user` and `subject.groups` from `kubeClient.subject`
  - Copy `driver` from `kubeClient.driver`
- If `type == "customSigner"`:
  - Copy `signerName` from `customSigner.signerName`
  - Copy full `subject` including `organizationUnits`

**Examples:**

**Case 1: kubeClient type registration**

```yaml
# v1alpha1
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
status:
  registrations:
  - signerName: kubernetes.io/kube-apiserver-client
    subject:
      user: foo
      groups:
      - group1
    driver: csr
---
# v1beta1
apiVersion: addon.open-cluster-management.io/v1beta1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
status:
  registrations:
  - type: kubeClient
    kubeClient:
      subject:
        user: foo
        groups:
        - group1
      driver: csr
```

**Case 2: customSigner type registration**

```yaml
# v1alpha1
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: custom-addon
  namespace: cluster1
status:
  registrations:
  - signerName: signer1
    subject:
      user: foo
      groups:
      - "group1"
      organizationUnits:
      - "group1"
---
# v1beta1
apiVersion: addon.open-cluster-management.io/v1beta1
kind: ManagedClusterAddOn
metadata:
  name: custom-addon
  namespace: cluster1
status:
  registrations:
  - type: customSigner
    customSigner:
      signerName: signer1
      subject:
        user: foo
        groups:
        - "group1"
        organizationUnits:
        - "group1"
```

### ManagedClusterAddon installNamespace

#### Key Change
`ManagedClusterAddon` `spec.installNamespace` is removed in v1beta1.

#### Conversion Webhook Design
- Annotation `addon.open-cluster-management.io/v1alpha1-install-namespace:<install-namespace>` will be introduced in v1beta1 to store the `spec.installNamespace` value from v1alpha1.
- The conversion webhook will convert [v1alpha1](https://github.com/open-cluster-management-io/api/blob/main/addon/v1alpha1/types_managedclusteraddon.go#L44) `spec.installNamespace` to v1beta1 annotation.

For example:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
spec:
  installNamespace: helloworld
...
---
apiVersion: addon.open-cluster-management.io/v1beta1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
  annotations:
    addon.open-cluster-management.io/v1alpha1-install-namespace: helloworld
spec:
...
```

### ManagedClusterAddon status

#### Key Change
`ManagedClusterAddon` `status.configReferences.name` `status.configReferences.namespace` is removed in v1beta1.

#### Conversion Webhook Design
- The conversion webhook will convert v1alpha1 `status.configReferences.name` `status.configReferences.namespace` to v1beta1 `status.configReferences.desiredConfig.name` `status.configReferences.desiredConfig.namespace`.

For example:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
...
status:
  configReferences:
  - desiredConfig:
      name: managed-serviceaccount-2.10
      specHash: 649d929d65e3cb801d3a627ba6de3103f951f8fb225b7e0915885b7bc6b84129
    group: addon.open-cluster-management.io
    lastAppliedConfig:
      name: managed-serviceaccount-2.10
      specHash: 649d929d65e3cb801d3a627ba6de3103f951f8fb225b7e0915885b7bc6b84129
    lastObservedGeneration: 1
    # Deprecated fields (duplicated from desiredConfig)
    name: managed-serviceaccount-2.10
    resource: addontemplates
---
apiVersion: addon.open-cluster-management.io/v1beta1
kind: ManagedClusterAddOn
metadata:
...
status:
  configReferences:
  - desiredConfig:
      name: managed-serviceaccount-2.10
      specHash: 649d929d65e3cb801d3a627ba6de3103f951f8fb225b7e0915885b7bc6b84129
    group: addon.open-cluster-management.io
    lastAppliedConfig:
      name: managed-serviceaccount-2.10
      specHash: 649d929d65e3cb801d3a627ba6de3103f951f8fb225b7e0915885b7bc6b84129
    resource: addontemplates
    lastObservedGeneration: 1
    # Deprecated name/namespace fields removed
```

### Other removed deprecated fields

Below fields are removed in v1beta1 and will not provide conversion.

Fields to removed:
- `ClusterManagementAddOn` `spec.addOnConfiguration`
- `ManagedClusterAddon` `status.addOnConfiguration`

Annotation on `ManagedClusterAddon` to removed:
- `open-cluster-management.io/addon-pre-delete` (use annotation `addon.open-cluster-management.io/addon-pre-delete` instead)

Labels on `ManagedClusterAddon` to removed:
- `addon.open-cluster-management.io/hosted-manifest-location` (use annotation `addon.open-cluster-management.io/hosted-manifest-location` instead)

Finalizers on `ManagedClusterAddon` to removed:
- `cluster.open-cluster-management.io/addon-pre-delete` (use finalizer `addon.open-cluster-management.io/addon-pre-delete` instead)
- `cluster.open-cluster-management.io/hosting-addon-pre-delete`  (use finalizer `addon.open-cluster-management.io/hosting-addon-pre-delete` instead)
- `cluster.open-cluster-management.io/hosting-manifests-cleanup`  (use finalizer `addon.open-cluster-management.io/hosting-manifests-cleanup` instead)

### Test Plan

#### Unit Tests
- Validation of v1beta1 API constraints
- Round-trip conversion (v1alpha1 → v1beta1 → v1alpha1) with data preservation
- Edge cases: empty configs, nil values, deprecated fields
- Conversion webhook logic for all field transformations

#### Integration Tests
- Create v1alpha1 resources, read as v1beta1
- Create v1beta1 resources, read as v1alpha1
- Mixed version environments (some using v1alpha1, others v1beta1)
- Webhook availability and fallback scenarios

#### E2E Tests
- Full addon lifecycle with v1beta1 APIs
- Existing v1alpha1 addons and controller continue to work after v1beta1 introduction
- Multiple addon instances across API versions

#### Backward Compatibility Tests
- Existing v1alpha1 addon continue to work
- Existing v1alpha1 controllers can operate against v1beta1 stored resources
- No data loss by conversion webhook

### Graduation Criteria

#### Alpha
- [ ] v1beta1 API definitions merged and available
- [ ] Conversion webhooks implemented and tested
- [ ] Basic unit and integration test coverage
- [ ] Documentation for v1beta1 API structure
- [ ] Known limitations documented

#### Beta
- [ ] v1beta1 API served alongside v1alpha1
- [ ] Conversion webhooks proven stable in production environments
- [ ] At least 3 addons successfully migrated to v1beta1
- [ ] Comprehensive test coverage (unit, integration, e2e)
- [ ] Migration guide and best practices documented

#### GA
- [ ] v1beta1 API used in production environments
- [ ] No major API changes needed based on user feedback
- [ ] All core OCM addons migrated to v1beta1
- [ ] Complete user-facing documentation
- [ ] API considered stable and ready for long-term support

### Upgrade/Downgrade Strategy

Follows https://github.com/open-cluster-management-io/api/blob/main/docs/development.md#api-upgrade-flow 

### Version Skew Strategy

- Conversion webhooks handle API version differences
- v1alpha1 and v1beta1 can coexist for extended transition period

## Alternatives

N/A