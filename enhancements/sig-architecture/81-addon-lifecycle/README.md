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

	// status represents the current status of the configuration for an add-on.
	// +optional
	Status AddOnHubConfigStatus `json:"status,omitempty"`
}

type AddOnHubConfigSpec struct {
	// version represents the desired addon version to install.
	// +optional
	DesiredVersion string `json:"desiredVersion,omitempty"`
}

// AddOnHubConfigStatus represents the current status of the configuration for an add-on.
type AddOnHubConfigStatus struct {
	// SupportedVersions lists all the valid addon versions. It's a hint for user to define desired version.
	// +optional
	SupportedVersions []string `json:"supportedVersions,omitempty"`
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
	// In scenario where all add-ons have the same configuration.
	// User can override the default configuration by defining the configs in the
	// install strategy for specific clusters.
	// +optional
	DefaultConfigs []AddOnConfig `json:"defaultConfigs,omitempty"`

	// InstallStrategy represents the install strategy of the add-on.
	// +optional
	InstallStrategy InstallStrategy `json:"installStrategy,omitempty"`
}

type InstallStrategy struct {
	// Type is the type of the install strategy, it can be:
	// - Manual: no automatic install
	// - Placements: install to clusters selected by placements.
	// +kubebuilder:validation:Enum=Manual;Placements
	// +kubebuilder:default:=Manual
	// +optional
	Type string `json:"type"`

	// Placements is a list of placement references honored when install strategy type is
	// Placements. All clusters selected by these placements will install the addon
	// If one cluster belongs to multiple placements, it will only apply the strategy defined
	// later in the order. That is to say, The latter strategy overrides the previous one.
	// +optional
	Placements []PlacementStrategy `json:"placements,omitempty"`
}

type PlacementStrategy struct {
	// Placement is the reference to a placement
	Placement PlacementRef `json:",inline"`

	// Configs is the configuration of managedClusterAddon during installation.
	// User can override the configuration by updating the managedClusterAddon directly.
	Configs []AddOnConfig `json:"configs,omitempty"`

	// The rollout strategy to apply addon configurations change.
	// The rollout strategy only watches the addon configurations defined in ClusterManagementAddOn.
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
	//   This is the default strategy.
	// - RollingUpdate: when configs change, apply the new configs to all the selected clusters with
	//   the concurrence rate defined in MaxConcurrentlyUpdating.
	// - RollingUpdateWithCanary: when configs change, wait and check if add-ons on the canary placement
	//   selected clusters have applied the new configs and are healthy, then apply the new configs to
	//   all the selected clusters with the concurrence rate defined in MaxConcurrentlyUpdating.
	//
	//   The field lastKnownGoodConfigSpecHash in the status record the last successfully applied
	//   spec hash of canary placement. If the config spec hash changes after the canary is passed and
	//   before the rollout is done, the current rollout will continue, then roll out to the latest change.
	//
	//   For example, the addon configs have spec hash A. The canary is passed and the lastKnownGoodConfigSpecHash
	//   would be A, and all the selected clusters are rolling out to A.
	//   Then the config spec hash changes to B. At this time, the clusters will continue rolling out to A.
	//   When the rollout is done and canary passed B, the lastKnownGoodConfigSpecHash would be B and
	//   all the clusters will start rolling out to B.
	//
	//   The canary placement does not have to be a subset of the install placement, and it is more like a
	//   reference for finding and checking canary clusters before upgrading all. To trigger the rollout
	//   on the canary clusters, you can define another rollout strategy with the type RollingUpdate, or even
	//   manually upgrade the addons on those clusters.
	//
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

`ClusterManagementAddOnStatus` adds `InstallProgression` to represent the current configuration references and conditions for each placement.

```golang
// ClusterManagementAddOnStatus represents the current status of cluster management add-on.
type ClusterManagementAddOnStatus struct {
	// installProgression is a list of current add-on configuration references per placement.
	// +optional
	InstallProgression []InstallProgression `json:"installProgression,omitempty"`
}

type InstallProgression struct {
	// Placement reference.
	// +optional
	Placement PlacementRef `json:"placement,omitempty"`

	// configReferences is a list of current add-on configuration references.
	// +optional
	ConfigReferences []InstallConfigReference `json:"configReferences,omitempty"`

	// conditions describe the state of the managed and monitored components for the operator.
	// +patchMergeKey=type
	// +patchStrategy=merge
	// +optional
	Conditions []metav1.Condition `json:"conditions,omitempty"  patchStrategy:"merge" patchMergeKey:"type"`
}

// InstallConfigReference is a reference to the current add-on configuration.
// This resource is used to record the configuration resource for the current add-on.
type InstallConfigReference struct {
	// This field is synced from ClusterManagementAddOn Configurations.
	ConfigGroupResource `json:",inline"`

	// This field is synced from ClusterManagementAddOn Configurations.
	ConfigReferent `json:",inline"`

	// desiredConfigSpecHash record the desired config spec hash.
	DesiredConfigSpecHash string `json:"desiredConfigSpecHash"`

	// lastKnownGoodConfigSpecHash record the last known good config spec hash.
	// For fresh install or rollout with type UpdateAll or RollingUpdate, the
	// lastKnownGoodConfigSpecHash is the same as lastAppliedConfigSpecHash.
	// For rollout with type RollingUpdateWithCanary, the lastKnownGoodConfigSpecHash
	// is the last successfully applied config spec hash of the canary placement.
	LastKnownGoodConfigSpecHash string `json:"lastKnownGoodConfigSpecHash"`

	// lastAppliedConfigSpecHash record the config spec hash when the all the corresponding
	// ManagedClusterAddOn are applied successfully.
	LastAppliedConfigSpecHash string `json:"lastAppliedConfigSpecHash"`
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

### ManifestWork new annotation `configsSpecHash`
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

### New condition type `Progressing`
A new condition type **Progressing** will be added to the `ManagedClusterAddOn` and it will replace the condition type **Config**.

This condition will also be used in `ClusterManagementAddOn` `installProgression` as a summary of each placement.

- The status could be `True` or `False`. 
  - `True` means the configurations are being applied (installing, upgrading).
  - `False` means the configurations are applied successfully or failed.
- The reason could be `Installing`, `InstallSucceed`, `InstallFailed`, `Upgrading`, `UpgradeSucceed`, `UpgradeFailed`,`WaitingForCanary`.
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


### Workflow
Below is an example of `ClusterManagementAddOn`, `ManagedClusterAddOn` and `ManifestWork`:

Kustomize could be used to produce the resource with a SHA based suffix.

The addon is upgrading from spec hash xxx to spec hash yyy, and the canary has passed the upgrade.


```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: AddOnHubConfig
metadata:
  name: hub-config-yyy
spec:
  desiredVersion: v0.11.0
status:
  supportedVersions:
  - v0.11.0
```

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  installStrategy:
    type: Placements
    placements:
    - name: aws-placement
      namespace: default
      configs:
      - group: addon.open-cluster-management.io
        resource: addonhubconfigs
        name: hub-config-yyy
      rolloutStrategy:
        type: RollingUpdateWithCanary
        # When type is RollingUpdateWithCanary, the rollout controller will check if clusters of canary-placement have 
        # applied new config hub-config-yyy and addons are healthy. If canary-placement clusters all upgrade successfully, 
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
    - name: canary-placement
      namespace: default
      configs:
      - group: addon.open-cluster-management.io
        resource: addonhubconfigs
        name: hub-config-yyy
      rolloutStrategy:
        type: RollingUpdate
        rollingUpdate:
          maxConcurrentlyUpdating: 25%
status:
  installProgression:
  - name: aws-placement
    namespace: default
    configReferences:
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-yyy
      # desiredConfigSpecHash is calculated from config spec hash and maintained by addon manager.
      desiredConfigSpecHash: <hub-config-yyy-spec-hash>
      # Record the config spec hash when the all the corresponding ManagedClusterAddOn are applied successfully. Maintained by the rollout controller.
      lastAppliedConfigSpecHash: <hub-config-xxx-spec-hash>
      # Record the last known good config spec hash. For rolling update with canary it's the spec hash of canary placement. Maintained by the rollout controller.
      lastKnownGoodConfigSpecHash: <hub-config-yyy-spec-hash>
    # A summary of all the ManagedClusterAddOn conditions.
    conditions:
    - lastTransitionTime: "2022-11-02T05:48:12Z"
      message: 200/400 clusters upgrading...
      reason: Upgrading
      status: "True"
      type: Progressing
  - name: canary-placement
    namespace: default
    configReferences:
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-yyy
      desiredConfigSpecHash: <hub-config-yyy-spec-hash>
      lastAppliedConfigSpecHash: <hub-config-yyy-spec-hash>
      lastKnownGoodConfigSpecHash: <hub-config-yyy-spec-hash>
    conditions:
    - lastTransitionTime: "2022-11-02T05:48:12Z"
      message: 100/100 upgrade completed with no errors.
      reason: UpgradeSucceed
      status: "Flase"
      type: Progressing
```

`ManagedClusterAddOn` is created by install controller and updated by rollout manager.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
status:
…
  # supportedConfigs is maintained by addon manager
  supportedConfigs:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
  # configReferences is maintained by rollout controller
  configReferences:
  - group: addon.open-cluster-management.io
    resource: addonhubconfigs
    name: hub-config-yyy
    desiredConfigSpecHash: <hub-config-yyy-spec-hash>
    lastAppliedConfigSpecHash: <hub-config-yyy-spec-hash>
  # conditions is maintained by rollout controller
  conditions:
  - lastTransitionTime: "2022-11-02T05:48:12Z"
    message: upgrade completed with no errors.
    reason: UpgradeSucceed
    status: "False"
    type: Progressing
```

`ManifestWork` is created by the addon manager.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  labels:
    open-cluster-management.io/addon-name: application-manager
  annotations:
    configSpecHash: | 
      {"addonhubconfigs.addon.open-cluster-management.io/hub-config-yyy":"<hub-config-yyy-spec-hash>"}
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

There will be 3 components and 2 kind of users involved in the workflow:

**Admin**
- The admin is responsible to install/upgrade the `ClusterManagementAddOn`, addon configurations `AddOnHubConfig` and addon manager deployment.
- Admin can modify the `spec.installStrategy` of `ClusterManagementAddOn`. For upgrade cases, needs to append 
`rolloutStrategy`.

**Normal User**
- A user can override the configuration by updating the `ManagedClusterAddon` directly. User defined configs won't be controlled by the rollout controller.

**AddOn Manager**
- Addon manager is responsible to update the `status.supportedConfigs` of `ManagedClusterAddOn`. 
- Addon manager has the permission to access all the configs. It watches the configs change and updates the `desiredConfigSpecHash` of `ClusterManagementAddOn`.
- Addon manager watches the `ManagedClusterAddOn` and generates `ManifestWork` with annotations `configsSpecHash`.

**Install Controller**
- Addon install controller is watching the `ClusterManagementAddOn` and generate `ManagedClusterAddOn` with `desiredConfigSpecHash` . 

**Rollout Controller**

A new component called rollout controller is introduced to rollout the configs.
- The rollout controller syncs the `desiredConfigSpecHash` from `ClusterManagementAddOn` to `ManagedClusterAddOn` .
  - For non-canary rollout, the rollout controller keeps the `desiredConfigSpecHash` in `ManagedClusterAddOn` 
    be aligned with the `desiredConfigSpecHash` in `ClusterManagementAddOn`. 
  - For canary rollout, the rollout controller keeps the `desiredConfigSpecHash` in `ManagedClusterAddOn` 
    be aligned with the `lastKnownGoodConfigSpecHash` in `ClusterManagementAddOn`. 
    The `lastKnownGoodConfigSpecHash` comes from the canary placment. 
- The rollout controller updates the `ManagedClusterAddOn` `lastAppliedConfigSpecHash` by checking the `ManifestWork`.
  - `ManifestWork` annotations `configsSpecHash` matches `desiredConfigSpecHash`.
  - `ManifestWork` Condition Available status is true.
  - `ManifestWork` Condition Available observedGeneration equals to generation.
  - If it is a fresh install since one addon can have multiple ManifestWorks, the `ManifestWork` condition ManifestApplied must also be true.
- The rollout controller updates the `ManagedClusterAddOn` conditions.
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
- The rollout controller updates the `ClusterManagementAddOn` `installProgression`.
  - For non-canary rollout, when all the addons rollout to `desiredConfigSpecHash`, it updates the `lastAppliedConfigSpecHash` and `lastKnownGoodConfigSpecHash` as `desiredConfigSpecHash`.
  - For canary rollout, when all the addons rollout to `lastKnownGoodConfigSpecHash`, it updates the `lastAppliedConfigSpecHash`as `desiredConfigSpecHash`.
  - The rollout controller also update the `conditions` of `installProgression`, which is a summary for all the placment addons.

### Examples

#### Fresh install addon to the default version.

[01-fresh-install](./kustomize-rollout-examples/01-fresh-install) shows a kustomize example which could be used to produce the `AddOnHubConfig` with a SHA based suffix and update the name reference in `ClusterManagementAddOn`, running `kubectl kustomize ./kustomize-rollout-examples/01-fresh-install` to show the output.

`AddOnHubConfig` has desired version v0.10.0. The resource name has a hash suffix xxx.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: AddOnHubConfig
metadata:
  name: hub-config-xxx
spec:
  desiredVersion: v0.10.0
status:
  supportedVersions:
  - v0.10.0
```

For fresh install, `ClusterManagementAddOn` has a list of configs and install strategy defined.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  installStrategy:
    type: Placements
    placements:
    - name: aws-placement
      namespace: default
      configs:
      - group: addon.open-cluster-management.io
        resource: addonhubconfigs
        name: hub-config-xxx
status:
  installProgression:
  - name: aws-placement
    namespace: default
    configReferences:
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-xxx
      # Addon manager calculates and updates the `desiredConfigSpecHash`.
      desiredConfigSpecHash: <hub-config-xxx-spec-hash>
      lastAppliedConfigSpecHash: ""
      lastKnownGoodConfigSpecHash: ""
    conditions:
    - lastTransitionTime: "2022-11-02T05:48:12Z"
      message: 400/400 installing...
      reason: Installing
      status: "True"
      type: Progressing
```

Since it's a fresh install, install controlller will generate `ManagedClusterAddOn` for all the clusters.

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
    resource: addonhubconfigs
    name: hub-config-xxx
    # The rollout controller sync the `desiredConfigSpecHash` from `ClusterManagementAddOn`.
    desiredConfigSpecHash: <hub-config-xxx-spec-hash>
    # The rollout controller watches the `ManifestWorks` generated by addon manager and updates the `lastAppliedConfigSpecHash`.
    lastAppliedConfigSpecHash: <hub-config-xxx-spec-hash>
  conditions:
  # The rollout controller watches the `ManifestWorks` generated by addon manager and updates the condition `Progressing`.
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
      {"addonhubconfigs.addon.open-cluster-management.io/hub-config-xxx":"<hub-config-xxx-spec-hash>"}
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

Once all the `ManagedClusterAddOns` are install successfullly, the rollout controller is also responsible to update the `installProgression` of `ClusterManagementAddOn`.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
...
status:
  installProgression:
  - name: aws-placement
    namespace: default
    configReferences:
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-xxx
      desiredConfigSpecHash: <hub-config-xxx-spec-hash>
      lastAppliedConfigSpecHash: <hub-config-xxx-spec-hash>
      lastKnownGoodConfigSpecHash: <hub-config-xxx-spec-hash>
    conditions:
    - lastTransitionTime: "2022-11-02T05:48:12Z"
      message: 400/400 install completed with no errors.
      reason: InstallSucceed
      status: "Flase"
      type: Progressing
```

#### Auto rolling upgrade addon to latest version.

[02-rolling-update](./kustomize-rollout-examples/02-rolling-update) shows a kustomize example, running `kubectl kustomize ./kustomize-rollout-examples/02-rolling-update` to show the output.

A new `AddOnHubConfig` with desired version v0.11.0 is created. The resource name has a hash suffix yyy.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: AddOnHubConfig
metadata:
  name: hub-config-yyy
spec:
  desiredVersion: v0.11.0
status:
  supportedVersions:
  - v0.11.0
  - v0.10.0
```

For rolling upgrade, `ClusterManagementAddOn` has rollout strategy with type `RollingUpdate` defined.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  installStrategy:
    type: Placements
    placements:
    - name: aws-placement
      namespace: default
      configs:
      - group: addon.open-cluster-management.io
        resource: addonhubconfigs
        name: hub-config-yyy
      rolloutStrategy:
        type: RollingUpdate
        rollingUpdate:
          maxConcurrentlyUpdating: 25%
status:
  installProgression:
  - name: aws-placement
    namespace: default
    configReferences:
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-yyy
      # When the config name changes, or config spec changes, the addon manager watches the changes
      # and update the `desiredConfigSpecHash`.
      desiredConfigSpecHash: <hub-config-yyy-spec-hash>
      lastAppliedConfigSpecHash: <hub-config-xxx-spec-hash>
      lastKnownGoodConfigSpecHash: <hub-config-xxx-spec-hash>
    conditons:
    - lastTransitionTime: "2022-11-02T05:48:12Z"
      message: 100/400 upgrading...
      reason: Upgrading
      status: "True"
      type: Progressing
```

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
    resource: addonhubconfigs
    name: hub-config-yyy
    # The rollout controller sync the `desiredConfigSpecHash` from `ClusterManagementAddOn`.
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
      {"addonhubconfigs.addon.open-cluster-management.io/hub-config-yyy":"<hub-config-yyy-spec-hash>"}
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
    resource: addonhubconfigs
    name: hub-config-yyy
    desiredConfigSpecHash: <hub-config-yyy-spec-hash>
    # The rollout controller watches the `ManifestWorks` generated by addon manager and updates the `lastAppliedConfigSpecHash`.
    lastAppliedConfigSpecHash: <hub-config-yyy-spec-hash>
  conditions:
  # The rollout controller watches the `ManifestWorks` generated by addon manager and updates the condition `Progressing`.
  - lastTransitionTime: "2022-11-02T05:48:12Z"
    message: upgrade completed with no errors.
    reason: UpgradeSucceed
    status: "False"
    type: Progressing
```

Once all the `ManagedClusterAddOns` are install successfullly, the rollout controller is also responsible to update the `installProgression` of `ClusterManagementAddOn`.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
...
status:
  installProgression:
  - name: aws-placement
    namespace: default
    configReferences:
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-yyy
      desiredConfigSpecHash: <hub-config-yyy-spec-hash>
      lastAppliedConfigSpecHash: <hub-config-yyy-spec-hash>
      lastKnownGoodConfigSpecHash: <hub-config-yyy-spec-hash>
    conditions:
    - lastTransitionTime: "2022-11-02T05:48:12Z"
      message: 400/400 upgrade completed with no errors.
      reason: UpgradeSucceed
      status: "False"
      type: Progressing
```

#### Auto canary upgrade addon to a specific version.

[03-rolling-update-with-canary](./kustomize-rollout-examples/03-rolling-update-with-canary) shows a kustomize example, running `kubectl kustomize ./kustomize-rollout-examples/03-rolling-update-with-canary` to show the output.

A new `AddOnHubConfig` with desired version v0.11.0 is created. The resource name has a hash suffix yyy.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: AddOnHubConfig
metadata:
  name: hub-config-yyy
spec:
  desiredVersion: v0.11.0
status:
  supportedVersions:
  - v0.11.0
  - v0.10.0
```

For rolling upgrade with canary, `ClusterManagementAddOn` has rollout strategy with type `RollingUpdateWithCanary` defined.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  installStrategy:
    type: Placements
    placements:
    - name: aws-placement
      namespace: default
      configs:
      - group: addon.open-cluster-management.io
        resource: addonhubconfigs
        name: hub-config-yyy
      rolloutStrategy:
        # The admin append rollout strategy type `RollingUpdateWithCanary` to `aws-placement` and refer to the `canary-placement` 
        # so that the `aws-placment` will auto-upgrade when `canary-placement` is upgrade successfully.
        type: RollingUpdateWithCanary
        rollingUpdateWithCanary:
          maxConcurrentlyUpdating: 25%
          placement:
            name: canary-placement
            namespace: default
    # The admin also needs to create a placement `canary-placement` with rollout strategy type `RollingUpdate`. The clusters 
    # inside `canary-placement` will apply the new config `hub-config-yyy` first. When it's done, `aws-placement` will continue 
    # to upgrade the remaining clusters.
    - name: canary-placement
      namespace: default
      configs:
      - group: addon.open-cluster-management.io
        resource: addonhubconfigs
        name: hub-config-yyy
      rolloutStrategy:
        type: RollingUpdate
        rollingUpdate:
          maxConcurrentlyUpdating: 25%
status:
  installProgression:
  - name: aws-placement
    namespace: default
    configReferences:
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-yyy
      # When the config name changes, or config spec changes, the addon manager watches the changes
      # and update the `desiredConfigSpecHash`.
      desiredConfigSpecHash: <hub-config-yyy-spec-hash>
      lastAppliedConfigSpecHash: <hub-config-xxx-spec-hash>
      lastKnownGoodConfigSpecHash: <hub-config-xxx-spec-hash>
    conditions:
    - lastTransitionTime: "2022-11-02T05:48:12Z"
      message: waitingForCanary...
      reason: WaitingForCanary
      status: "True"
      type: Progressing
  - name: canary-placement
    namespace: default
    configReferences:
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-yyy
      desiredConfigSpecHash: <hub-config-yyy-spec-hash>
      lastAppliedConfigSpecHash: <hub-config-xxx-spec-hash>
      lastKnownGoodConfigSpecHash: <hub-config-xxx-spec-hash>
    conditions:
    - lastTransitionTime: "2022-11-02T05:48:12Z"
      message: 25/100 upgrading...
      reason: Upgrading
      status: "True"
      type: Progressing
```

Once all the canary `ManagedClusterAddOns` are upgraded successfullly, the rollout controller will update the `installProgression` of `ClusterManagementAddOn` as below.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
...
status:
  installProgression:
  - name: aws-placement
    namespace: default
    configReferences:
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-yyy
      desiredConfigSpecHash: <hub-config-yyy-spec-hash>
      lastAppliedConfigSpecHash: <hub-config-xxx-spec-hash>
      # lastKnownGoodConfigSpecHash comes from canary-placement
      lastKnownGoodConfigSpecHash: <hub-config-yyy-spec-hash>
    conditions:
    - lastTransitionTime: "2022-11-02T05:48:12Z"
      message: 200/400 upgrading...
      reason: Upgrading
      status: "True"
      type: Progressing
  - name: canary-placement
    namespace: default
    configReferences:
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-yyy
      desiredConfigSpecHash: <hub-config-yyy-spec-hash>
      # lastKnownGoodConfigSpecHash and lastKnownGoodConfigSpecHash is yyy as canary upgrade successfully.
      lastAppliedConfigSpecHash: <hub-config-yyy-spec-hash>
      lastKnownGoodConfigSpecHash: <hub-config-yyy-spec-hash>
    conditions:
    - lastTransitionTime: "2022-11-02T05:48:12Z"
      message: 100/100 upgrade completed with no errors.
      reason: UpgradeSucceed
      status: "False"
      type: Progressing
```

When all the aws `ManagedClusterAddOns` are upgraded successfullly, the rollout controller will update the `installProgression` of `ClusterManagementAddOn` as below.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
...
status:
  installProgression:
  - name: aws-placement
    namespace: default
    configReferences:
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-yyy
      desiredConfigSpecHash: <hub-config-yyy-spec-hash>
      lastAppliedConfigSpecHash: <hub-config-yyy-spec-hash>
      lastKnownGoodConfigSpecHash: <hub-config-yyy-spec-hash>
    conditions:
    - lastTransitionTime: "2022-11-02T05:48:12Z"
      message: 400/400 upgrade completed with no errors.
      reason: UpgradeSucceed
      status: "False"
      type: Progressing
  - name: canary-placement
    namespace: default
    configReferences:
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-yyy
      desiredConfigSpecHash: <hub-config-yyy-spec-hash>
      lastAppliedConfigSpecHash: <hub-config-yyy-spec-hash>
      lastKnownGoodConfigSpecHash: <hub-config-yyy-spec-hash>
    conditions:
    - lastTransitionTime: "2022-11-02T05:48:12Z"
      message: 100/100 upgrade completed with no errors.
      reason: UpgradeSucceed
      status: "False"
      type: Progressing
```

#### Rollback addon to a specific version.

[04-rollback](./kustomize-rollout-examples/04-rollback) shows a kustomize example, running `kubectl kustomize ./kustomize-rollout-examples/04-rollback` to show the output.

If the canary upgrade doesn't work well, the admin can roll back the config name to the previous config name.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  installStrategy:
    type: Placements
    placements:
    - name: aws-placement
      namespace: default
      configs:
      - group: addon.open-cluster-management.io
        resource: addonhubconfigs
        name: hub-config-xxx
      rolloutStrategy:
        type: RollingUpdateWithCanary
        rollingUpdateWithCanary:
          maxConcurrentlyUpdating: 25%
          placement:
            name: canary-placement
            namespace: default
    - name: canary-placement
      namespace: default
      configs:
      - group: addon.open-cluster-management.io
        resource: addonhubconfigs
        name: hub-config-xxx
      rolloutStrategy:
        type: RollingUpdate
        rollingUpdate:
          maxConcurrentlyUpdating: 25%
status:
  installProgression:
  - name: aws-placement
    namespace: default
    configReferences:
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-xxx
      desiredConfigSpecHash: <hub-config-xxx-spec-hash>
      lastAppliedConfigSpecHash: <hub-config-xxx-spec-hash>
      lastKnownGoodConfigSpecHash: <hub-config-xxx-spec-hash>
    conditions:
    - lastTransitionTime: "2022-11-02T05:48:12Z"
      message: 100/400 upgrading...
      reason: Upgrading
      status: "True"
      type: Progressing
  - name: canary-placement
    namespace: default
    configReferences:
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-xxx
      desiredConfigSpecHash: <hub-config-xxx-spec-hash>
      lastAppliedConfigSpecHash: <hub-config-xxx-spec-hash>
      lastKnownGoodConfigSpecHash: <hub-config-xxx-spec-hash>
    conditions:
    - lastTransitionTime: "2022-11-02T05:48:12Z"
      message: 25/100 upgrading...
      reason: Upgrading
      status: "True"
      type: Progressing
```

#### Corner cases
1. A new cluster is added.

If the newly added cluster only belongs to canary-placement.
- if there is no `ManagedClusterAddOn`, it should follow the install strategy defined in the canary to install the addon.
- if there's already `ManagedClusterAddOn` on that cluster, it should follow the canary rollout strategy to rollout the 
addon to desired configs.

If the newly added cluster only belongs to aws-placement.
- if there is no `ManagedClusterAddOn`, it should follow the install strategy defined in aws to install the addon.
- if there's already `ManagedClusterAddOn` on that cluster, it should wait for the canary rollout finish, then follow the 
aws rollout strategy to rollout the addon to the desired status.

If the new added cluster belongs to both of them, it should first follow the strategy in canary-placement, and then follow 
the strategy in aws-placement.

2. Should the clusters changes in canary frequently retrigger the rollout?

The case is clusters change after the canary rollout is complete and before the aws rollout is done. Should this 
stops the aws rollout and wait for the canary to complete again and retrigger the rollout?

Our answer is this won't stop the ongoing rollout, since the `lastKnownGoodConfigSpecHash` did't change.

3. Should the desired configs changes frequently retrigger the rollout?

The case is the desired configs change after the canary rollout is complete and before the aws rollout is done. Should this 
stops the aws rollout and wait for the canary to complete again and retrigger the rollout?

Our answer is this won't stop the ongoing rollout.

For example, config with spec hash A is the `lastKnownGoodConfigSpecHash` before the modification. 
Then config changes to spec hash B, it passes the canary to become the new `lastKnownGoodConfigSpecHash`, and is rolling out.
Before the rollout is done, configs changes to spec hash C. The rollout to B should continue since it's the `lastKnownGoodConfigSpecHash`.
When the rollout is done, and canary passes C, the `lastKnownGoodConfigSpecHash` would become C and addons continue rollout to C.

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