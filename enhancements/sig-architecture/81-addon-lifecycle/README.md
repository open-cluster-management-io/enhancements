# Add-on Lifecycle

## Release Signoff Checklist

- [] Enhancement is `implemented`
- [] Design details are appropriately documented from clear requirements
- [] Test plan is defined
- [] Graduation criteria for dev preview, tech preview, GA
- [] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal is to propose an enhancement of the `ClusterManagementAddon` and `ManagedClusterAddOn` API so that users can do
canary upgrade on a specific addon.

## Motivation

Today, the addon agent is controlled by the addon manager, and when the addon manager upgrades all the agents will upgrade automatically.
In some cases, we are seeing requirements that the user would like to rolling upgrade an addon to the new version. That is, 
upgrade the addon on a group of clusters first, if everything works fine, then upgrade all the clusters to the new version.

### Goals

- Enhance the existing `ClusterManagementAddon` and `ManagedClusterAddOn` API, to support version-based addon installation 
on a group of clusters. 
- Enhance the existing `ClusterManagementAddon` and `ManagedClusterAddOn` API, to reflect the addon installation, upgrade 
and rollback progress, including the addon version, progress, and completed time.

### Non-Goals
- Enhance the existing addon framework and addon manager to support rolling upgrade.
- Enhance the `ClusterManagementAddon` status to reflect the overall upgrade progress, for example, how 
many clusters are upgraded with new configs.

## Proposal

### Use cases

#### Story 1
- As a user, I can configure a desired version of my addon to install for a given managed cluster. 
- I can know the installation status, including the addon version(desired version, current version), 
progress (installing, failed, succeeded) and completed time.

#### Story 2
- As a user, I can configure a desired version of my addon to upgrade for a given managed cluster. 
- I can know the upgrade status, including the addon version(desired version, current version), 
progress (upgrading, failed, succeeded) and completed time.

#### Story 3
- As a user, I can configure a desired version of my addon to roll back for a given managed cluster. 
- I can know the rollback status, including the addon version(desired version, current version), 
progress (rolling back, failed, succeeded) and completed time.

### Risks and Mitigation
N/A

## Design Details
The design is based on [Addon install strategy #77](https://github.com/open-cluster-management-io/enhancements/pull/77/files) 
and [Add-on Configuration](https://github.com/open-cluster-management-io/enhancements/tree/main/enhancements/sig-architecture/58-addon-configuration).

### API changes

Introduce a new API `AddOnHubConfig`. This is a cluster-scoped API that can be used to do some global configurations, 
eg. the desired addon version.

```golang
// +genclient
// +genclient:nonNamespaced

// AddOnHubConfig represents a hub-scoped configuration for an add-on.
type AddOnHubConfig struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// spec represents a desired configuration for an add-on.
	// +required
	Spec AddOnHubConfigSpec `json:"spec"`
}

type AddOnHubConfigSpec struct {
  // version represents the desired addon version to install.
  // +optional
  DesiredVersion string `json:"desiredVersion,omitempty"`
}
```

`ClusterManagementAddOn` removes `SupportedConfigs`, adds `DefaultConfigs` section to list the default add-on configurations. 
`InstallStrategy` adds a nested `RolloutStrategy` to control the add-ons upgrade behavior when configurations change.

```golang
// ClusterManagementAddOnSpec provides information for the add-on.
type ClusterManagementAddOnSpec struct {
  // addOnMeta is a reference to the metadata information for the add-on.
  // +optional
  AddOnMeta AddOnMeta `json:"addOnMeta,omitempty"`

	// defaultConfigs represents a list of default add-on configurations.
	// In scenario where all add-ons have a same configuration.
	// The default configs can be overridden by the configs defined in install strategy for
	// specific clusters. Any add-on configs defined in ManagedClusterAddon spec explicitly can
	// override the configs defined in ClusterManagedAddOn.
	// +optional
	DefaultConfigs []AddOnConfig `json:"defaultConfigs,omitempty"`

	// InstallStrategy represents the install strategy of the add-on.
	// +optional
	InstallStrategy InstallStrategy `json:"installStrategy,omitempty"`
}

type InstallStrategy struct {
	// Type is the type of the install strategy, it can be:
	// - Manual: no automatic install
	// - ClusterLabelSelector: install to clusters with certain labels
	// - Placements: install to clusters selected by placements.
	// +kubebuilder:validation:Enum=Manual;Placements
	// +kubebuilder:default:=Manual
	// +optional
	Type string `json:"type"`

	// ClusterLabelSelector is a label selector honored when install strategy type is
	// ClusterLabelSelector
	// +optional
	ClusterLabelSelector *metav1.LabelSelector `json:"labelSelector,omitempty"`

	// Placements is a list of placement references, addon settings and rollout strategy 
	// honored when installing strategy type is Placements.
	// All clusters selected by these placements will install the addon.
	// If one cluster belongs to multiple placements, it will only apply the strategy defined 
	// later in the order. That is to say, the strategy defined later in the order overrides the 
	// previous.
	// +optional
	Placements []PlacementStrategies `json:"placements,omitempty"`
}

type PlacementStrategies struct {
	// Name of the placement strategy
	// +kubebuilder:validation:Required
	// +required
	Name string `json:"name"`

	// Placement reference.
	// +kubebuilder:validation:Required
	// +required
	Placement PlacementRef `json:"placement,omitempty"`

	// AddonTemplate is the template to generate ManagedClusterAddon.
	// +optional
	AddonTemplate ManagedClusterAddOnSpec `json:"addonTemplate,omitempty"`

	// The rollout strategy to apply new addon configs.
	// The rollout strategy watches the default configs and addon template configs change.
	// If the rollout strategy is not defined, the default strategy UpdateAll is used.
	// If there are configs change during the rollout process, the rollout will start over. 
	// For example, addon template configs list configA and configB. The change in configA triggers 
	// the rollout. If the configB is also changed before the rollout is complete, the current 
	// rollout stops and the rollout starts over.
	// +optional
	RolloutStrategy RolloutStrategy `json:"rolloutStrategy,omitempty"`
}

type PlacementRef struct {
	// Name of the placement
	// +kubebuilder:validation:Required
	// +required
	Name string `json:"name"`

	// Namespace of the placement
	// +kubebuilder:validation:Required
	// +required
	Namespace string `json:"namespace"`
}

// RolloutStrategy represents the rollout strategy of the add-on configuration.
type RolloutStrategy struct {
	// Type is the type of the rollout strategy, it supports UpdateAll, RollingUpdate and RollingUpdateWithCanary:
	// - UpdateAll: when configs change, apply the new configs to all the selected clusters at once.
	// - RollingUpdate: when configs change, apply the new configs to all the selected clusters with the concurrence
	//   rate defined in MaxConcurrentlyUpdating.
	// - RollingUpdateWithCanary: when configs change, wait and check if add-ons on the canary placement selected clusters
	//   have applied the new configs and are healthy, then apply the new configs to all the selected clusters with the
	//   concurrence rate defined in MaxConcurrentlyUpdating. 
	//   The canary placement does not have to be a subset of the install placement, and it is more like a reference for finding 
	//   and checking canary clusters before upgrading all. To trigger the rollout on the canary clusters, you can define another 
	//   rollout strategy with the type RollingUpdate, or even manually upgrade the addons on those clusters.
	// +kubebuilder:validation:Enum=UpdateAll;RollingUpdate;RollingUpdateWithCanary
	// +kubebuilder:default:=UpdateAll
	// +optional
	Type string `json:"type"`

	// Rolling update with placement config params. Present only if the type is RollingUpdate.
	// +optional
	RollingUpdate *RollingUpdate `json:"rollingUpdate,omitempty"`

	// Rolling update with placement config params. Present only if the type is RollingUpdateWithCanary.
	// +optional
	RollingUpdateWithCanary *RollingUpdateWithCanary `json:"rollingUpdateWithCanary,omitempty"`
}

// RollingUpdate represents the behavior to rolling update add-on configurations
// on the selected clusters.
type RollingUpdate struct {
	// The maximum concurrently updating number of addons.
	// Value can be an absolute number (ex: 5) or a percentage of desired addons (ex: 10%).
	// Absolute number is calculated from percentage by rounding up.
	// Defaults to 25%.
	// Example: when this is set to 30%, once the addon configs change, the addon on 30% of the selected clusters
	// will adopt the new configs. When the addons with new configs are healthy, the addon on the remaining clusters
	// will be further updated.
	// +kubebuilder:default:="25%"
	// +optional
	MaxConcurrentlyUpdating *intstr.IntOrString `json:"maxConcurrentlyUpdating,omitempty"`
}

// RollingUpdateWithCanary represents the canary placement and behavior to rolling update add-on configurations
// on the selected clusters.
type RollingUpdateWithCanary struct {
	// Canary placement reference.
	// +kubebuilder:validation:Required
	// +required
	Placement PlacementRef `json:"placement,omitempty"`

	// the behavior to rolling update add-on configurations.
	RollingUpdate `json:",inline"`
}
```

`ClusterManagementAddOnStatus` adds `ConfigReferences` to represent the current configuration references and `DesiredConfigSpecHash`.

```golang
// ClusterManagementAddOnStatus represents the current status of the cluster management add-on.
type ClusterManagementAddOnStatus struct {
	// configReferences is a list of current add-on configuration references.
	// The configs defined in ClusterManagementAddOn defaultConfigs and installStrategy configs fields will be listed here.
	// +optional
	ConfigReferences []DesiredConfigReference `json:"configReferences,omitempty"`
}

// DesiredConfigReference is a reference to the current add-on configuration.
// This resource is used to record the configuration resource for the current add-on.
type DesiredConfigReference struct {
	// This field is synced from ClusterManagementAddOn defaultConfig and installStrategy configs fields.
	ConfigGroupResource `json:",inline"`

	// This field is synced from ClusterManagementAddOn defaultConfig and installStrategy configs fields.
	ConfigReferent `json:",inline"`

	// desiredConfigSpecHash record the desired config spec hash.
	DesiredConfigSpecHash string `json:"desiredConfigSpecHash"`
}
```

`ManagedClusterAddOnStatus` adds `SupportedConfigs` to list the configuration types which are allowed to override the 
configuration defined in `ClusterManagementAddOn`.

`ManagedClusterAddOnStatus` remove `LastObservedGeneration` and use `DesiredConfigSpecHash` and `LastAppliedConfigSpecHash` 
to record the configuration spec hash.

```golang
// ManagedClusterAddOnStatus provides information about the status of the operator.
// +k8s:deepcopy-gen=true
type ManagedClusterAddOnStatus struct {
  ...
  // supportedConfigs is a list of configuration types that are allowed to override the add-on configurations defined 
  // in ClusterManagementAddOn spec.
  // The default is an empty list, which means the add-on configurations can not be overridden.
  // +optional
  // +listType=map
  // +listMapKey=group
  // +listMapKey=resource
  SupportedConfigs []ConfigGroupResource `json:"supportedConfigs,omitempty"`

  // configReferences is a list of current add-on configuration references.
  // This will be overridden by the clustermanagementaddon configuration references.
  // +optional
  ConfigReferences []ConfigReference `json:"configReferences,omitempty"`
  ...
}

// ConfigReference is a reference to the current add-on configuration.
// This resource is used to locate the configuration resource for the current add-on.
type ConfigReference struct {
  // This field is synced from ClusterManagementAddOn configGroupResource field.
  ConfigGroupResource `json:",inline"`

  // This field is synced from ClusterManagementAddOn Configuration and ManagedClusterAddOn config fields.
  // If both are defined, the ManagedClusterAddOn configs will overwrite the ClusterManagementAddOn
  // Configuration.
  ConfigReferent `json:",inline"`

  // lastObservedGeneration is the observed generation of the add-on configuration.
  // LastObservedGeneration int64 `json:"lastObservedGeneration"`

  // desiredConfigSpecHash record the desired config spec hash.
  DesiredConfigSpecHash string `json:"desiredConfigSpecHash"`

  // lastAppliedConfigSpecHash record the config spec hash when the corresponding ManifestWork is applied successfully.
  LastAppliedConfigSpecHash string `json:"lastAppliedConfigSpecHash"`
}
```

### New annotation
A new annotation `configsSpecHash` is added to the `ManifestWork` to record the corresponding config and spec hash.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  labels:
    open-cluster-management.io/addon-name: application-manager
  annotations:
    configsSpecHash: {"<resource>.<group>/<namespace>/<name>":"<config-spec-hash>",...}
  generation: 2
  name: addon-application-manager-deploy
  namespace: cluster2
```

### New condition
A new condition type **Progressing** will be added to the `ManagedClusterAddOn` and it will replace the condition type **Config**.
- The status could be `True` or `False`.
  - `True` means the configurations are being applied (installing, upgrading).
  - `False` means the configurations are applied successfully or failed.
- The reason could be `Installing`, `InstallSucceed`, `InstallFailed`, `Upgrading`, `UpgradeSucceed`, `UpgradeFailed`.
- `lastTransitionTime` record the transition time.   

```yaml
  - lastTransitionTime: "2022-11-02T05:48:12Z"
    message: install completed with no errors.
    reason: Succeed
    status: "False"
    type: Progressing
#  remove condition type Config.
#  - lastTransitionTime: "2022-11-02T05:48:12Z"
#    message: install completed with no errors.
#    reason: Succeed
#    status: "False"
#    type: Config
```

Each addon manager is responsible to watch its `ClusterManagementAddOn` and its own configurations. Once the configurations 
name or spec hash changes, the addon manager updates the `desiredConfigSpecHash` in `ClusterManagementAddOnStatus`.

A new component called rollout controller is responsible to compare the `desiredConfigSpecHash` of `ClusterManagementAddOnStatus` 
and `ManagedClusterAddOnStatus` and updates the `ManagedClusterAddOnStatus` `desiredConfigSpecHash`.

The addon manager watches the `ManagedClusterAddOn` and generates `ManifestWork` with annotations `configsSpecHash`.

Next, the rollout controller updates the `ManagedClusterAddOnStatus` `lastAppliedConfigSpecHash` by checking the `ManifestWork`.
  - `ManifestWork` annotations `configsSpecHash` matches `desiredConfigSpecHash`.
  - `ManifestWork` Condition Available status is true.
  - `ManifestWork` Condition Available observedGeneration equals to generation.
  - If it is a fresh install since one addon can have multiple ManifestWorks, the `ManifestWork` condition 
  ManifestApplied must also be true.

The rollout controller then compares the `lastAppliedConfigSpecHash` with `desiredConfigSpecHash` to update the condition.
- If `lastAppliedConfigSpecHash` == `desiredConfigSpecHash`
  - If `LastAppliedConfigSpecHash` is empty before it equals to `desiredConfigSpecHash`, update condition Progressing 
  status to false with reason `InstallSucceed`.
  - If `lastAppliedConfigSpecHash` is not empty before it equals to `desiredConfigSpecHash`, update condition Progressing 
  status to false with reason `UpgradeSucceed`.
- If `lastAppliedConfigSpecHash` != `desiredConfigSpecHash`
  - If `lastAppliedConfigSpecHash` is empty, update condition Progressing status to true with reason `Installing`.
    - If meets some error during the installation, for example, the `ManifestWork` reports an error, update the condition Progressing 
    status to false with the reason `InstallFailed`.
  - If `lastAppliedConfigSpecHash` is not empty, update condition Progressing status to true with reason `Upgrading`.
    - If meets some error during the installation, for example, the `ManifestWork` reports an error, update the condition Progressing 
    status to false with the reason `UpgradeFailed`.

### Workflow
Below is an example of `ClusterManagementAddOn`, `ManagedClusterAddOn` and `ManifestWork`:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: AddOnHubConfig
metadata:
  name: hub-config-yyy
spec:
  desiredVersion: v0.10.0
```

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  defaultConfigs:
  - group: addon.open-cluster-management.io
    resource: addonhubconfigs
    name: hub-config-yyy
  installStrategy:
    type: Placements
    placements:
    - name: aws
      placement:
        name: aws-placement
        namespace: default
      addonTemplate:
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: aws-config-yyy
      rolloutStrategy:
        type: RollingUpdateWithCanary
        # When type is RollingUpdateWithCanary, the rollout controller will check if clusters of canary-placement have 
        # applied new config aws-config-yyy and addons are healthy. If canary-placement clusters all upgrade successfully, 
        # rollout controller will continue to upgrade clusters in aws-placement.
        #
        # The placment defined in RollingUpdateWithCanary is just a reference for finding and checking canary clusters before 
        # upgrading all. 
        # To trigger the rollout of canary-placement, you can define another rolloutStrategy with type RollingUpdate 
        # as below, or even manually upgrade the addons on canary-placement clusters.
        rollingUpdateWithCanary:
          placement:
            name: canary-placement
            namespace: default
          maxConcurrentlyUpdating: 25%
    - name: canary
      placement:
        name: canary-placement
        namespace: default
      addonTemplate:
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: aws-config-yyy
      rolloutStrategy:
        type: RollingUpdate
        rollingUpdate:
          maxConcurrentlyUpdating: 25%
status:
  configReferences:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: aws-config-yyy
    namespace: open-cluster-management
    desiredConfigSpecHash: <aws-config-yyy-spec-hash>
  - group: addon.open-cluster-management.io
    resource: addonhubconfigs
    name: hub-config-yyy
    desiredConfigSpecHash: <hub-config-yyy-spec-hash>
```

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
status:
…
  supportedConfigs:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
  configReferences:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: aws-config-yyy
    namespace: open-cluster-management
    desiredConfigSpecHash: <aws-config-yyy-spec-hash>
    lastAppliedConfigSpecHash: <aws-config-yyy-spec-hash>
  - group: addon.open-cluster-management.io
    resource: addonhubconfigs
    name: hub-config-yyy
    desiredConfigSpecHash: <hub-config-yyy-spec-hash>
    lastAppliedConfigSpecHash: <hub-config-yyy-spec-hash>
  conditions:
  - lastTransitionTime: "2022-11-02T05:48:12Z"
    message: upgrade completed with no errors.
    reason: UpgradeSucceed
    status: "False"
    type: Progressing
```

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  labels:
    open-cluster-management.io/addon-name: application-manager
  annotations:
    configSpecHash: | 
      {"addonhubconfigs.addon.open-cluster-management.io/hub-config-yyy":"<hub-config-yyy-spec-hash>", 
      "addondeploymentconfigs.addon.open-cluster-management.io/open-cluster-management/aws-config-yyy":"<aws-config-yyy-spec-hash>"}
  generation: 2
  name: addon-application-manager-deploy
  namespace: cluster2
…
status:
  conditions:
  - lastTransitionTime: "2022-11-02T05:47:30Z"
    message: All resources are available
    observedGeneration: 2
    reason: ResourcesAvailable
    status: "True"
    type: Available
```

The workflow would be:
1. `ClusterManagementAddOn`, addon configurations `AddOnHubConfig` `AddOnDeploymentConfig` and addon manager 
deployment is installed/upgraded by admin.
2. Admin can modify the `spec.installStrategy` of `ClusterManagementAddOn`. This is not a necessary step, the admin can 
choose to use the installStrategy defined by the installer. For upgrade cases, the admin also needs to append 
`rolloutStrategy` to `ClusterManagementAddOn`.
3. `ManagedClusterAddOn` is created for each cluster with the addon configurations. The resource could be created by 
addon-install-controller which is watching the `ClusterManagementAddOn`, or be manually created by a user. 
4. Addon manager updates the `status.supportedConfigs` of `ManagedClusterAddOn`. The `status.supportedConfigs` list the 
config type which is allowed to be overridden by a normal user.
5. Addon manager updates the `status.configReferences` of `ClusterManagementAddOn`. The `status.configReferences` list the 
config name and `desiredConfigSpecHash`.
6. A user can update the `configs` of `ManagedClusterAddOn` based on `status.supportedConfigs`.
7. Addon-rollout-controller watches the `ClusterManagementAddOn` and updates the `desiredConfigSpecHash` in `ManagedClusterAddOn` status.
8. Addon manager watches the `ManagedClusterAddOn`, gets addon configurations, and generates `ManifestWorks` for each 
cluster. The `ManifestWorks` has the annotation `configSpecHash` to indicate the addon configurations. 
9. Addon-rollout-controller watches the `ManifestWorks` and updates the `lastAppliedConfigSpecHash` in `ManagedClusterAddOn` status.
10. Addon-rollout-controller updates the `ManagedClusterAddOn` `status.condition` type `Progressing` to report the configuration install/upgrade result.

### Examples

#### Fresh install addon to the default version.
`AddOnHubConfig` has desired version v0.10.0. The resource name has a hash suffix which will change when the content is updated.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: AddOnHubConfig
metadata:
  name: hub-config-xxx
spec:
  desiredVersion: v0.10.0
```

For fresh install, `ClusterManagementAddOn` has a list of configs and install strategy defined.

Addon manager updates the `status.configReferences` of `ClusterManagementAddOn`. The `status.configReferences` list the 
config name and `desiredConfigSpecHash`.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  defaultConfigs:
    - group: addon.open-cluster-management.io
      resource: addondeploymentconfigs
      name: deploy-config-xxx
      namespace: open-cluster-management
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-xxx
  installStrategy:
    type: Placements
    placements:
    - name: aws
      placement:
        name: aws-placement
        namespace: default
      addonTemplate:
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: aws-config-xxx
    - name: gcp
      placement:
        name: gcp-placement
        namespace: default
      addonTemplate:
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: gcp-config-xxx
status:
  configReferences:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: deploy-config-xxx
    namespace: open-cluster-management
    desiredConfigSpecHash: <deploy-config-xxx-spec-hash>
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: aws-config-xxx
    namespace: open-cluster-management
    desiredConfigSpecHash: <aws-config-xxx-spec-hash>
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: gcp-config-xxx
    namespace: open-cluster-management
    desiredConfigSpecHash: <gcp-config-xxx-spec-hash>
  - group: addon.open-cluster-management.io
    resource: addonhubconfigs
    name: hub-config-xxx
    desiredConfigSpecHash: <hub-config-xxx-spec-hash>
```

Since it's a fresh install, install controlller will generate `ManagedClusterAddOn` for all the clusters.

The rollout controller will watch the `ClusterManagementAddOn`, update `ManagedClusterAddOn` with `desiredConfigSpecHash` 
for all of the selected clusters.

The rollout controller is also responsible to update the `lastAppliedConfigSpecHash` and `conditions` by watching the addon 
`ManifestWorks` generated by addon manager.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
status:
…
  supportedConfigs:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
  configReferences:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: aws-config-xxx
    namespace: open-cluster-management
    desiredConfigSpecHash: <aws-config-xxx-spec-hash>
    lastAppliedConfigSpecHash: <aws-config-xxx-spec-hash>
  - group: addon.open-cluster-management.io
    resource: addonhubconfigs
    name: hub-config-xxx
    desiredConfigSpecHash: <hub-config-xxx-spec-hash>
    lastAppliedConfigSpecHash: <hub-config-xxx-spec-hash>
  conditions:
  - lastTransitionTime: "2022-11-02T05:48:12Z"
    message: install completed with no errors.
    reason: InstallSucceed
    status: "False"
    type: Progressing
```

Add manager watches `ManagedClusterAddOn` and generates `ManifestWork` with annotation `configSpecHash`.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  annotations:
    configSpecHash: | 
      {"addonhubconfigs.addon.open-cluster-management.io/hub-config-xxx":"<hub-config-xxx-spec-hash>", 
      "addondeploymentconfigs.addon.open-cluster-management.io/open-cluster-management/aws-config-xxx":"<aws-config-xxx-spec-hash>"}
  labels:
    open-cluster-management.io/addon-name: application-manager
  generation: 2
  name: addon-application-manager-deploy
  namespace: cluster2
…
status:
  conditions:
  - lastTransitionTime: "2022-11-02T05:47:30Z"
    message: All resources are available
    observedGeneration: 2
    reason: ResourcesAvailable
    status: "True"
    type: Available
```

#### Auto rolling upgrade addon to latest version.
Create a new `AddOnHubConfig` with the latest version,

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: AddOnHubConfig
metadata:
  name: hub-config-yyy
spec:
  desiredVersion: v0.11.0
```

When the config name changes, or config spec changes, the addon manager watches the change in `ClusterManagementAddOn`, 
`AddOnHubConfig` and update the `desiredConfigSpecHash` of `ClusterManagementAddOn`.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  defaultConfigs:
    - group: addon.open-cluster-management.io
      resource: addondeploymentconfigs
      name: deploy-config-yyy
      namespace: open-cluster-management
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-yyy
  installStrategy:
    type: Placements
    placements:
    - name: aws
      placement:
        name: aws-placement
        namespace: default
      addonTemplate:
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: aws-config-yyy
      rolloutStrategy:
        type: RollingUpdate
        rollingUpdate:
          maxConcurrentlyUpdating: 25%
    - name: gcp
      placement:
        name: gcp-placement
        namespace: default
      addonTemplate:
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: gcp-config-yyy
      rolloutStrategy:
        type: RollingUpdate
        rollingUpdate:
          maxConcurrentlyUpdating: 25%
status:
  configReferences:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: deploy-config-yyy
    namespace: open-cluster-management
    desiredConfigSpecHash: <deploy-config-yyy-spec-hash>
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: aws-config-yyy
    namespace: open-cluster-management
    desiredConfigSpecHash: <aws-config-yyy-spec-hash>
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: gcp-config-yyy
    namespace: open-cluster-management
    desiredConfigSpecHash: <gcp-config-yyy-spec-hash>
  - group: addon.open-cluster-management.io
    resource: addonhubconfigs
    name: hub-config-yyy
    desiredConfigSpecHash: <hub-config-yyy-spec-hash>
```

The rollout manager compare the `desiredConfigSpecHash` of `ClusterManagementAddOn` and `ManagedClusterAddOn`, update the 
`desiredConfigSpecHash` of `ManagedClusterAddOnStatus`.

Since the rollout strategy type is `RollingUpdate`, the rollout manager will first pick 25% of the clusters to do the update.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
status:
…
  supportedConfigs:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
  configReferences:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: aws-config-yyy
    namespace: open-cluster-management
    desiredConfigSpecHash: <aws-config-yyy-spec-hash>
    lastAppliedConfigSpecHash: <aws-config-xxx-spec-hash>
  - group: addon.open-cluster-management.io
    resource: addonhubconfigs
    name: hub-config-yyy
    desiredConfigSpecHash: <hub-config-yyy-spec-hash>
    lastAppliedConfigSpecHash: <hub-config-xxx-spec-hash>
  conditions:
  - lastTransitionTime: "2022-11-02T05:48:12Z"
    message: upgrading...
    reason: Upgrading
    status: "True"
    type: Progressing
```

The addon manager watches the change in `ManagedClusterAddOn` and update the `ManifestWork`.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  annotations:
    configSpecHash: | 
      {"addonhubconfigs.addon.open-cluster-management.io/hub-config-yyy":"<hub-config-yyy-spec-hash>", 
      "addondeploymentconfigs.addon.open-cluster-management.io/open-cluster-management/aws-config-yyy":"<aws-config-yyy-spec-hash>"}
  labels:
    open-cluster-management.io/addon-name: application-manager
  generation: 2
  name: addon-application-manager-deploy
  namespace: cluster2
…
status:
  conditions:
  - lastTransitionTime: "2022-11-02T05:47:30Z"
    message: All resources are available
    observedGeneration: 2
    reason: ResourcesAvailable
    status: "True"
    type: Available
```

The rollout controller then watches the `ManifestWork` and update the `lastAppliedConfigSpecHash` and `conditions`. 
When the 25% `ManagedClusterAddOn` is healthy, rollout manager will continue to update the remaining clusters.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
status:
…
  supportedConfigs:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
  configReferences:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: aws-config-yyy
    namespace: open-cluster-management
    desiredConfigSpecHash: <aws-config-yyy-spec-hash>
    lastAppliedConfigSpecHash: <aws-config-yyy-spec-hash>
  - group: addon.open-cluster-management.io
    resource: addonhubconfigs
    name: hub-config-yyy
    desiredConfigSpecHash: <hub-config-yyy-spec-hash>
    lastAppliedConfigSpecHash: <hub-config-yyy-spec-hash>
  conditions:
  - lastTransitionTime: "2022-11-02T05:48:12Z"
    message: upgrade completed with no errors.
    reason: UpgradeSucceed
    status: "False"
    type: Progressing
```

#### Auto canary upgrade addon to a specific version.
Create a new `AddOnHubConfig` with the latest version,

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: AddOnHubConfig
metadata:
  name: hub-config-yyy
spec:
  desiredVersion: v0.11.0
```

The admin append rollout strategy type `RollingUpdateWithCanary` to `aws-placement` and refer to the `canary-placement` 
so that the `aws-placment` will auto-upgrade when `canary-placement` is upgrade successfully.

The admin also needs to create a placement `canary-placement` with rollout strategy type `RollingUpdate`. The clusters 
inside `canary-placement` will apply the new config `aws-config-yyy` first. When it's done, `aws-placement` will continue 
to upgrade the remaining clusters.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  defaultConfigs:
    - group: addon.open-cluster-management.io
      resource: addondeploymentconfigs
      name: deploy-config-yyy
      namespace: open-cluster-management
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-yyy
  installStrategy:
    type: Placements
    placements:
    - name: aws
      placement:
        name: aws-placement
        namespace: default
      addonTemplate:
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: aws-config-yyy
      rolloutStrategy:
        type: RollingUpdateWithCanary
        rollingUpdateWithCanary:
          maxConcurrentlyUpdating: 25%
          placement:
            name: canary-placement
            namespace: default
    - name: canary
      placement:
        name: canary-placement
        namespace: default
      addonTemplate:
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: aws-config-yyy
      rolloutStrategy:
        type: RollingUpdate
        rollingUpdate:
          maxConcurrentlyUpdating: 25%
    - name: gcp
      placement:
        name: gcp-placement
        namespace: default
      addonTemplate:
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: gcp-config-yyy
      rolloutStrategy:
        type: RollingUpdate
        rollingUpdate:
          maxConcurrentlyUpdating: 25%
status:
  configReferences:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: deploy-config-yyy
    namespace: open-cluster-management
    desiredConfigSpecHash: <deploy-config-yyy-spec-hash>
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: aws-config-yyy
    namespace: open-cluster-management
    desiredConfigSpecHash: <aws-config-yyy-spec-hash>
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: gcp-config-yyy
    namespace: open-cluster-management
    desiredConfigSpecHash: <gcp-config-yyy-spec-hash>
  - group: addon.open-cluster-management.io
    resource: addonhubconfigs
    name: hub-config-yyy
    desiredConfigSpecHash: <hub-config-yyy-spec-hash>
```

#### Rollback addon to a specific version.

If the canary upgrade doesn't work well, the admin can roll back the config name to the previous config name.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  defaultConfigs:
    - group: addon.open-cluster-management.io
      resource: addondeploymentconfigs
      name: deploy-config-xxx
      namespace: open-cluster-management
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-xxx
  installStrategy:
    type: Placements
    placements:
    - name: aws
      placement:
        name: aws-placement
        namespace: default
      addonTemplate:
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: aws-config-xxx
      rolloutStrategy:
        type: RollingUpdateWithCanary
        rollingUpdateWithCanary:
          maxConcurrentlyUpdating: 25%
          placement:
            name: canary-placement
            namespace: default
    - name: canary
      placement:
        name: canary-placement
        namespace: default
      addonTemplate:
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: aws-config-xxx
      rolloutStrategy:
        type: RollingUpdate
        rollingUpdate:
          maxConcurrentlyUpdating: 25%
    - name: gcp
      placement:
        name: gcp-placement
        namespace: default
      addonTemplate:
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: gcp-config-xxx
      rolloutStrategy:
        type: RollingUpdate
        rollingUpdate:
          maxConcurrentlyUpdating: 25%
```

#### Corner cases
1. A new cluster is added.

If the newly added cluster only belongs to canary-placement.
- if there is no `ManagedClusterAddOn`, it should follow the install strategy defined in the canary to install the addon.
- if there's already `ManagedClusterAddOn` on that cluster, it should follow the canary rollout strategy to upgrade the 
addon to desired configs.

If the newly added cluster only belongs to aws-placement.
- if there is no `ManagedClusterAddOn`, it should follow the install strategy defined in aws to install the addon.
- if there's already `ManagedClusterAddOn` on that cluster, it should wait for the canary upgrade finish, then follow the 
aws rollout strategy to upgrade the addon to the desired status.

If the new added cluster belongs to both of them, it should first follow the strategy in canary-placement, and then follow 
the strategy in aws-placement.

2. Should the changes in canary frequently trigger the aws rollout?

The case is configs or clusters change after the canary upgrade is complete and before the aws upgrade finish. Should this 
stops the aws rollout and wait for the canary to complete again and retrigger the rollout?

The current consideration is to add a `installProgression` in the `ClusterManagementAddOn` status to record the result. 
So the rollout controller only needs to check the `installProgression` to know if the canary has once finished, if yes, 
it could start rolling upgrade and won't be frequently retriggered.

```yaml
installProgression:
- name: canary
  placement:
    name: canary-placement
    namespace: default
  configReferences:
  - group: addon.open-cluster-management.io
    resource: addonhubconfigs
    name: hub-config-yyy
    desiredConfigSpecHash: <hub-config-yyy-spec-hash>
  - group: addon.open-cluster-management.io
    resource: addonhubconfigs
    name: hub-config-yyy
    desiredConfigSpecHash: <hub-config-yyy-spec-hash>
  lastTransitionTime: "2022-11-02T05:48:12Z"
  message: install completed with no errors.
  reason: Succeed
  status: "False"
  type: Progressing
```

### Test Plan
- All implementations will have thorough coverage of unit tests.
- Integration tests will cover the user cases.

### Graduation Criteria
#### Alpha
At first, This proposal will be in the alpha stage and needs to meet
1. The new APIs are reviewed and accepted;
2. Implementation is completed to support the functionalities;
3. Develop test cases to demonstrate this proposal works correctly;

#### Beta
1. Need to revisit the API shape before upgrading to beta based on user feedback.

### Upgrade / Downgrade Strategy
TBD

### Version Skew Strategy
N/A

## Alternatives
### API changes

```golang
// ClusterManagementAddOnStatus represents the current status of cluster management add-on.
type ClusterManagementAddOnStatus struct {
  // SupportedVersions lists all the valid addon versions. 
  // +optional
  SupportedVersions []string `json:"supportedVersions,omitempty"`

  // DefaultVersion lists the default addon versions to be installed. 
  // +optional
  DefaultVersion string `json:"defaultVersion,omitempty"`
}
```

```golang
// ManagedClusterAddOnSpec defines the install configuration of
// an addon agent on managed cluster.
type ManagedClusterAddOnSpec struct {
  // …
  InstallNamespace string `json:"installNamespace,omitempty"`
	
  // desiredVersion represents the desired addon version to install.
  // Setting this value will trigger an upgrade or rollback (if the current version
  // listed in the status does not match the desired version).
  // 
  // Valid addon versions are listed in supportedVersion in the status. 
  // If invalid desiredVersion is specified, that may cause the upgrade or rollback 
  // to fail.
  // If desiredVersion is not specified, the defaultVersion in the status will be used.
  //
  // The progress of an update or rollback will be reported in the conditions.
  // +optional
  DesiredVersion string `json:"desiredVersion,omitempty"`

  // …
  Configs []AddOnConfig `json:"configs,omitempty"`
}
```

```golang
// ManagedClusterAddOnStatus provides information about the status of the operator.
// +k8s:deepcopy-gen=true
type ManagedClusterAddOnStatus struct {
  // …

  // …
  Conditions []metav1.Condition `json:"conditions,omitempty"  patchStrategy:"merge" patchMergeKey:"type"`

  // currentVersion represents the currently successfully installed addon version.
  // If the addon installation, upgrade or rollback is in progressing, the value is unknown.
  // +optional
  CurrentVersion string `json:"currentVersion,omitempty"`

  // lastVersion record the previous defined desiredVersion.
  // If the previous addon desiredVersion is invalid, the value is unknown.
  // If no previous addon desiredVersion, the value is empty.
  // +optional
  LastVersion string `json:"lastVersion,omitempty"`
}
```

### New condition
A new condition type **Progressing** will be added to the `ManagedClusterAddOn`. 
- When the addon in installing/upgrading/rollingback, the status will be true, 
  reason will be Installing/Upgrading/Rollingback.
- When the addon is installed successfully/failed, the status will be false, 
  reason will be Succeed/Failed.

```yaml
  - lastTransitionTime: "2022-11-02T05:48:12Z"
    message: install completed with no errors.
    reason: Succeed
    status: "False"
    type: Progressing
```

### Workflow
Below is an example of `ClusterManagementAddOn`, `ManagedClusterAddOn` and `ManifestWork`:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  installStrategy:
    type: Placements
    placements:
    - name: global-placement
      namespace: default
      addonTemplate:
        desiredVersion: v1
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: global-config
    - name: canary-placement
      namespace: default
      addonTemplate:
        desiredVersion: v2
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: canary-config
status:
  supportedVersions:
  - v1
  - v2
  defaultVersion: v2
```

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
spec:
  installNamespace: open-cluster-management-agent-addon
  desiredVersion: v2
Status:
…
  lastVersion: v1
  currentVersion: v2
  conditions:
  - lastTransitionTime: "2022-11-02T05:48:12Z"
    message: install completed with no errors.
    reason: Succeed
    status: "False"
    type: Progressing
```

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  labels:
    open-cluster-management.io/addon-name: application-manager
    open-cluster-management.io/addon-version: v2
  generation: 2
  name: addon-application-manager-deploy
  namespace: cluster2
…
status:
  conditions:
  - lastTransitionTime: "2022-11-02T05:47:30Z"
    message: All resources are available
    observedGeneration: 2
    reason: ResourcesAvailable
    status: "True"
    type: Available
```

The workflow would be:
- `ClusterManagementAddOn` and addon manager deployment is installed or upgraded.
- Addon manager updates the status.supportedVersion and status.defaultVersion in `ClusterManagementAddOn`.
- User updates the spec.installStrategy. This is not a necessary step, the user can choose to use the installStrategy 
defined by the installer.  
- `ManagedClusterAddOn` is created for each cluster with the desired addon version. The resource could be created by 
addon-install-controller which is watching the installStrategy, or be manually created by a user. 
- Addon manager watches the `ManagedClusterAddOn` and generates `ManifestWorks` for each cluster. The `ManifestWorks` 
has the label `open-cluster-management.io/addon-version` to indicate the addon version. 
- Addon manager copies the last `ManagedClusterAddOn` status.desiredVersion as status.lastVersion.
- Addon manager updates the `ManagedClusterAddOn` status.currentVersion based on `ManifestWork` status.
- Addon manager updates the `ManagedClusterAddOn` status.condition to tell the user whether installation/upgrade/rollback 
is succeeded or failed.  

**The desiredVersion, lastVersion and currentVersion**

If the `ManagedClusterAddOn` `status.currentVersion` does not match the `spec.desiredVersion`, the addon manager will an update:
- Addon manager copies previous spec.desiredVersion as the status.lastVersion. 
  (Fresh install doesn’t have lastVersion, invalid version is treated as unknown.)
- Addon manager updates the status.currentVersion to unknown when `ManifestWork` installation/upgrade/rollback is not finished.
- Addon manager updates status.currentVersion as spec.desiredVersion when `ManifestWork` installation/upgrade/rollback is finished.
  - `ManifestWork` has correct “addon-name” and “addon-version” labels.
  - `ManifestWork` Condition Available status is true.
  - `ManifestWork` Condition Available observedGeneration equals to generation.
  - If it is a fresh install, since one addon can have multiple ManifestWorks, the `ManifestWork` condition ManifestApplied 
  must also be true.

**Condition**

Addon manager also updates the `ManagedClusterAddOn` status.condition to tell the user 
whether installation/upgrade/rollback is succeeded or failed, lastTransitionTime records the finish time.   
- If currentVersion==desiredVersion, update condition Progressing status to false with reason Succeed.
- If currentVersion==unknown
  - lastVersion < desiredVersion, update condition Progressing status to true with reason Upgrading.
  - lastVersion > desiredVersion, update condition Progressing status to true with reason Rollingback.
  - lastVersion=="" || lastVersion="unknown", update condition Progressing status to true with reason Installing.
- Any other reasons, update condition Progressing status to false with reason Failed.