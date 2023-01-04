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

`ClusterManagementAddOn` removes `SupportedConfigs` and adds `Configuration` section to list the add-on configurations 
and the rollout strategy when configurations change.

```golang
// ClusterManagementAddOnSpec provides information for the add-on.
type ClusterManagementAddOnSpec struct {
  // addOnMeta is a reference to the metadata information for the add-on.
  // +optional
  AddOnMeta AddOnMeta `json:"addOnMeta,omitempty"`

  // configuration list the add-on configurations and the rollout strategy when the configurations change.
  // +optional
  Configuration Configuration `json:"configuration,omitempty"`
}

// Configuration represents a list of the add-on configurations and their rollout strategy.
type Configuration struct {
  // configs is a list of add-on configurations.
  // The add-on configurations for each cluster can be overridden by the configs of the ManagedClusterAddon spec.
  // +optional
  Configs []AddOnConfig `json:"configs,omitempty"`

  // The rollout strategy to apply new configs. 
  // The rollout strategy only watches the listed configs change. 
  // If the rollout strategy is not defined, the default strategy UpdateAll is used.
  // If there are configs change during the rollout process, the rollout will start over. For example, configs list 
  // configA and configB. The change in configA triggers the rollout. If the configB is also
  // changed before the rollout is complete, the current rollout stops and the rollout starts over.
  // +optional
  RolloutStrategy RolloutStrategy `json:"rolloutStrategy,omitempty"`
}

// RolloutStrategy represents the rollout strategy of the add-on configuration.
type RolloutStrategy struct {
  // Type is the type of the rollout strategy, it supports UpdateAll and RollingUpdateWithPlacement:
  // - UpdateAll: when configs change, apply the new configs to all the clusters.
  // - RollingUpdateWithPlacement: when configs change, rolling update new configs on the clusters 
  //   selected by placements. 
  //   If any of the configs are overridden by ManagedClusterAddOn on the specific cluster, the new configs 
  //   won't take effect on that cluster.
  //   This rollout strategy is only responsible for applying new configs. When the strategy is modified or 
  //   removed, the applied configs won't be deleted from the cluster.
  // +kubebuilder:validation:Enum=RollingUpdateWithPlacement;UpdateAll
  // +kubebuilder:default:=UpdateAll
  // +optional
  Type string `json:"type"`

  // Rolling update with placement config params. Present only if the type is RollingUpdateWithPlacement.
  // +optional
  RollingUpdateWithPlacement *RollingUpdateWithPlacement `json:"rollingUpdateWithPlacement,omitempty"`
}

// RollingUpdateWithPlacement represents the placement and behavior to rolling update add-on configurations
// on the selected clusters.
type RollingUpdateWithPlacement struct {
  // name of the placement
  // +kubebuilder:validation:Required
  // +required
  Name string `json:"name"`

  // namespace of the placement.
  // +kubebuilder:validation:Required
  // +required
  Namespace string `json:"namespace"`

  // The maximum concurrently updating number of addons.
  // Value can be an absolute number (ex: 5) or a percentage of desired addons (ex: 10%).
  // Absolute number is calculated from percentage by rounding up.
  // Defaults to 25%.
  // Example: when this is set to 30%, once the addon configs change, the addon on 30% of the selected clusters
  // will adopt the new configs. When the new configs are ready, the addon on the remaining clusters
  // will be further updated.
  // +optional
  MaxConcurrentlyUpdating *intstr.IntOrString `json:"maxConcurrentlyUpdating,omitempty"`
}
```

`ManagedClusterAddOnStatus` adds `SupportedConfigs` to list the configuration types which are allowed to override the 
the configuration defined in `ClusterManagementAddOn`.

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

When a new config is updated in the `ManagedClusterAddOn`, the addon manager will first calculate the spec hash and 
update the `DesiredConfigSpecHash`.
Then the addon manager will generate `ManifestWork` with annotations `configsSpecHash`.

Next, the addon manager updates the `LastAppliedConfigSpecHash` by checking the `ManifestWork`.
  - `ManifestWork` annotations `configsSpecHash` matches `DesiredConfigSpecHash`.
  - `ManifestWork` Condition Available status is true.
  - `ManifestWork` Condition Available observedGeneration equals to generation.
  - If it is a fresh install since one addon can have multiple ManifestWorks, the `ManifestWork` condition 
  ManifestApplied must also be true.

The addon manager then compares the `LastAppliedConfigSpecHash` with `DesiredConfigSpecHash` to update the condition.
- If `LastAppliedConfigSpecHash` == `DesiredConfigSpecHash`
  - If `LastAppliedConfigSpecHash` is empty before it equals to `DesiredConfigSpecHash`, update condition Progressing 
  status to false with reason `InstallSucceed`.
  - If `LastAppliedConfigSpecHash` is not empty before it equals to `DesiredConfigSpecHash`, update condition Progressing 
  status to false with reason `UpgradeSucceed`.
- If `LastAppliedConfigSpecHash` != `DesiredConfigSpecHash`
  - If `LastAppliedConfigSpecHash` is empty, update condition Progressing status to true with reason `Installing`.
    - If meets some error during the installing, for example, the `ManifestWork` reports an error, update condition Progressing 
    status to false with reason `InstallFailed`.
  - If `LastAppliedConfigSpecHash` is not empty, update condition Progressing status to true with reason `Upgrading`.
    - If meets some error during the installing, for example, the `ManifestWork` reports an error, update condition Progressing 
    status to false with reason `UpgradeFailed`.

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
  configuration:
    configs:
    - group: addon.open-cluster-management.io
      resource: addondeploymentconfigs
      name: deploy-config-xxx
      namespace: open-cluster-management
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-yyy
    rolloutStrategy:
      type: RollingUpdateWithPlacement
      rollingUpdateWithPlacement:
        name: canary-placement
        namespace: default
        maxConcurrentlyUpdating: 25%
  installStrategy:
    type: Placements
    placements:
    - name: aws-placement
      namespace: default
      addonSpec:
        installNamespace: open-cluster-management-agent-addon
        configs:
          - group: addon.open-cluster-management.io
            resource: addondeploymentconfigs
            name: aws-config
    - name: gcp-placement
      namespace: default
      addonSpec:
        installNamespace: open-cluster-management-agent-addon
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: gcp-config
```

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
spec:
  installNamespace: open-cluster-management-agent-addon
  configs:
    - group: addon.open-cluster-management.io
      resource: addondeploymentconfigs
      name: aws-config
Status:
…
  supportedConfigs:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
  configReferences:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: aws-config
    namespace: open-cluster-management
    desiredConfigSpecHash: <aws-config-spec-hash>
    lastAppliedConfigSpecHash: <aws-config-spec-hash>
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
      "addondeploymentconfigs.addon.open-cluster-management.io/open-cluster-management/aws-config":"<aws-config-spec-hash>"}
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
1. `ClusterManagementAddOn`, default addon configurations `AddOnHubConfig` `AddOnDeploymentConfig` and addon manager 
deployment is installed/upgraded by admin.
2. Admin can modify the `spec.installStrategy` of `ClusterManagementAddOn`. This is not a necessary step, the admin can 
choose to use the installStrategy defined by the installer. For upgrade case, the admin also needs to append 
`rolloutStrategy` to `ClusterManagementAddOn`.
3. `ManagedClusterAddOn` is created for each cluster with the addon configurations. The resource could be created by 
addon-install-controller which is watching the `ClusterManagementAddOn`, or be manually created by a user. 
4. Addon manager updates the `status.supportedConfigs` of `ManagedClusterAddOn`. The `status.supportedConfigs` list the 
config type which is allowed to be overridden by a normal user.
5. A user can update the `configs` of `ManagedClusterAddOn` based on `status.supportedConfigs`.
6. Addon manager updates the `status.configReferences` of `ManagedClusterAddOn`, including the `desiredConfigSpecHash`.
7. Addon manager watches the `ManagedClusterAddOn`, gets addon configurations, and generates `ManifestWorks` for each 
cluster. The `ManifestWorks` has the annotation `configSpecHash` to indicate the addon configurations. 
8. Addon manager watches the `ManifestWorks` and updates the `lastAppliedConfigSpecHash` in `ManagedClusterAddOn` status.
9. Addon manager updates the `ManagedClusterAddOn` `status.condition` type `Progressing` to report the configuration 
install/upgrade result.

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

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  configuration:
    configs:
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
    - name: aws-placement
      namespace: default
      addonSpec:
        installNamespace: open-cluster-management-agent-addon
        configs:
          - group: addon.open-cluster-management.io
            resource: addondeploymentconfigs
            name: aws-config
    - name: gcp-placement
      namespace: default
      addonSpec:
        installNamespace: open-cluster-management-agent-addon
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: gcp-config
```

The user can modify `spec.configs` based on `status.supportedConfigs` to override the configs defined in `ClusterManagementAddOn`.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
spec:
  installNamespace: open-cluster-management-agent-addon
  configs:
    - group: addon.open-cluster-management.io
      resource: addondeploymentconfigs
      name: aws-config
Status:
…
  supportedConfigs:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
  configReferences:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: aws-config
    namespace: open-cluster-management
    desiredConfigSpecHash: <aws-config-spec-hash>
    lastAppliedConfigSpecHash: <aws-config-spec-hash>
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

Add manager generates `ManifestWork` with annotation `configSpecHash`.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  annotations:
    configSpecHash: | 
      {"addonhubconfigs.addon.open-cluster-management.io/hub-config-xxx":"<hub-config-xxx-spec-hash>", 
      "addondeploymentconfigs.addon.open-cluster-management.io/open-cluster-management/aws-config":"<aws-config-spec-hash>"}
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

#### Auto upgrade addon to latest version.
Create a new `AddOnHubConfig` with the latest version,

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: AddOnHubConfig
metadata:
  name: hub-config-yyy
spec:
  desiredVersion: v0.11.0
```

The config name inside `ClusterManagementAddOn` will change at the same time. 
The addon manager watches the change and applies the new config to all the clusters.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  configuration:
    configs:
    - group: addon.open-cluster-management.io
      resource: addondeploymentconfigs
      name: deploy-config-xxx
      namespace: open-cluster-management
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-yyy
  installStrategy:
    type: Placements
    placements:
    - name: aws-placement
      namespace: default
      addonSpec:
        installNamespace: open-cluster-management-agent-addon
        configs:
          - group: addon.open-cluster-management.io
            resource: addondeploymentconfigs
            name: aws-config
    - name: gcp-placement
      namespace: default
      addonSpec:
        installNamespace: open-cluster-management-agent-addon
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: gcp-config
```

#### Canary upgrade addon to specific version.
Create a new `AddOnHubConfig` with the latest version,

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: AddOnHubConfig
metadata:
  name: hub-config-yyy
spec:
  desiredVersion: v0.11.0
```

The config name inside `ClusterManagementAddOn` will change at the same time. 

The admin can create a placement `canary-placement` to select the canary upgrade clusters.
Then append `rolloutStrategy` and refer to the `canary-placement` so that the new config will only apply to the canary clusters.

The addon manager watches the change and applies the new config to the selected clusters.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  configuration:
    configs:
    - group: addon.open-cluster-management.io
      resource: addondeploymentconfigs
      name: deploy-config-xxx
      namespace: open-cluster-management
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-yyy
    rolloutStrategy:
      type: RollingUpdateWithPlacement
      rollingUpdateWithPlacement:
        name: canary-placement
        namespace: default
        maxConcurrentlyUpdating: 25%
  installStrategy:
    type: Placements
    placements:
    - name: aws-placement
      namespace: default
      addonSpec:
        installNamespace: open-cluster-management-agent-addon
        configs:
          - group: addon.open-cluster-management.io
            resource: addondeploymentconfigs
            name: aws-config
    - name: gcp-placement
      namespace: default
      addonSpec:
        installNamespace: open-cluster-management-agent-addon
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: gcp-config
```

The `ManagedClusterAddOn` `status.configReferences` updates to new config.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
spec:
  installNamespace: open-cluster-management-agent-addon
  configs:
    - group: addon.open-cluster-management.io
      resource: addondeploymentconfigs
      name: aws-config
Status:
…
  supportedConfigs:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
  configReferences:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: aws-config
    namespace: open-cluster-management
    desiredConfigSpecHash: <aws-config-spec-hash>
    lastAppliedConfigSpecHash: <aws-config-spec-hash>
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

The new config spec hash is applied to `ManifestWork`.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  annotations:
    configSpecHash: | 
      {"addonhubconfigs.addon.open-cluster-management.io/hub-config-yyy":"<hub-config-yyy-spec-hash>", 
      "addondeploymentconfigs.addon.open-cluster-management.io/open-cluster-management/aws-config":"<aws-config-spec-hash>"}
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

Once the ManifestWork is applied successfully, the `lastAppliedConfigSpecHash` of `ManagedClusterAddOn` and condition will be updated.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
spec:
  installNamespace: open-cluster-management-agent-addon
  configs:
    - group: addon.open-cluster-management.io
      resource: addondeploymentconfigs
      name: aws-config
Status:
…
  supportedConfigs:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
  configReferences:
  - group: addon.open-cluster-management.io
    resource: addondeploymentconfigs
    name: aws-config
    namespace: open-cluster-management
    desiredConfigSpecHash: <aws-config-spec-hash>
    lastAppliedConfigSpecHash: <aws-config-spec-hash>
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

If the canary upgrade works fine, the admin can remove the `rolloutStrategy` or create another placement `global-placement` 
to select all the clusters.
Modify the `rolloutStrategy` to refer to `global-placement` and apply the new config to all the clusters.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  configuration:
    configs:
    - group: addon.open-cluster-management.io
      resource: addondeploymentconfigs
      name: deploy-config-xxx
      namespace: open-cluster-management
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-yyy
    rolloutStrategy:
      type: RollingUpdateWithPlacement
      rollingUpdateWithPlacement:
        name: global-placement
        namespace: default
        maxConcurrentlyUpdating: 25%
  installStrategy:
  ...
```

#### Rollback addon to a specific version.

If the canary upgrade doesn't work well, the admin can roll back the config name to the previous config name.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  configuration:
    configs:
    - group: addon.open-cluster-management.io
      resource: addondeploymentconfigs
      name: deploy-config-xxx
      namespace: open-cluster-management
    - group: addon.open-cluster-management.io
      resource: addonhubconfigs
      name: hub-config-xxx
    rolloutStrategy:
      type: RollingUpdateWithPlacement
      rollingUpdateWithPlacement:
        name: canary-placement
        namespace: default
        maxConcurrentlyUpdating: 25%
  installStrategy:
  ...
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

### Examples

#### Fresh install addon to default version.
The supportedVersions and defaultVersion is updated by addon manager.

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
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: global-config
status:
  supportedVersions:
  - v1
  defaultVersion: v1
```

If desiredVersion is not defined in `ManagedClusterAddOn`, use the defaultVersion v1.
When the installation is finished, the currentVersion is updated to v1.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
spec:
  installNamespace: open-cluster-management-agent-addon
Status:
…
  currentVersion: v1
  conditions:
  - lastTransitionTime: "2022-11-02T05:48:12Z"
    message: install completed with no errors.
    reason: Succeed
    status: "False"
    type: Progressing
```

Add manager generates `ManifestWork` with addon-name and addon-version label.
```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  labels:
    open-cluster-management.io/addon-name: application-manager
    open-cluster-management.io/addon-version: v1
  generation: 1
  name: addon-application-manager-deploy
  namespace: cluster1
…
```

#### Auto upgrade addon to latest version.
After addon manager is upgraded, the supportedVersions and defaultVersion is also updated by addon manager.

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
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: global-config
status:
  supportedVersions:
  - v1
  - v2
  defaultVersion: v2
```

Since the defaultVersion is upgraded to v2, the `ManagedClusterAddOn` default desiredVersion is updated to v2.
The addon manager will update `ManifestWork` with addon-version v2.
When the upgrade is finished, the currentVersion is updated to v2.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
spec:
  installNamespace: open-cluster-management-agent-addon
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

#### Canary upgrade addon to specific version.
Users can modify the installStrategy to achieve canary upgrade.

1. Define the installedVerison explicitly in global-placement before upgrade.
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
status:
  supportedVersions:
  - v1
  defaultVersion: v1
```

2. Upgrade the ClusterManagementAddOn and addon manager.

After the addon manager is upgraded, the supportedVersions and defaultVersion is also updated by addon manager.

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
status:
  supportedVersions:
  - v1
  - v2
  defaultVersion: v2
```

3. Add new placement to select a group of clusters to upgrade to the new version addon.

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

4. The upgraded `ManagedClusterAddOn` would be like:

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
  currentVersion: unknown
  conditions:
  - lastTransitionTime: "2022-11-02T05:48:12Z"
    message: Upgrading addon to version v2.
    reason: Upgrading
    status: "True"
    type: Progressing
```

Then becomes 

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

If the upgrade version is invalid, or any other error occurs during upgrade, the `ManagedClusterAddOn` would be like:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
spec:
  installNamespace: open-cluster-management-agent-addon
  desiredVersion: v3
Status:
…
  lastVersion: v1
  currentVersion: unknown
  conditions:
  - lastTransitionTime: "2022-11-02T05:48:12Z"
    message: could not upgrade to invalid version v3
    reason: Failed
    status: "False"
    type: Progressing
```

#### Rollback addon to a specific version.
If the addon is not working well after upgrade, or the condition show error message during upgrade. User can modify the 
installStrategy to rollback the addons.

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
        desiredVersion: v1
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

The rollbacked `ManagedClusterAddOn` would be like:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
spec:
  installNamespace: open-cluster-management-agent-addon
  desiredVersion: v1
Status:
…
  lastVersion: v2
  currentVersion: unknown
  conditions:
  - lastTransitionTime: "2022-11-02T05:48:12Z"
    message: Rollingback addon to version v1.
    reason: Rollingback
    status: "True"
    type: Progressing
```

Then becomes 

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
spec:
  installNamespace: open-cluster-management-agent-addon
  desiredVersion: v1
Status:
…
  lastVersion: v2
  currentVersion: v1
  conditions:
  - lastTransitionTime: "2022-11-02T05:48:12Z"
    message: install completed with no errors.
    reason: Succeed
    status: "False"
    type: Progressing
```

If the rollbacked version is invalid, or any other error occurs during rollback, the `ManagedClusterAddOn` would be like:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
spec:
  installNamespace: open-cluster-management-agent-addon
  desiredVersion: v0
Status:
…
  lastVersion: v1
  currentVersion: unknown
  conditions:
  - lastTransitionTime: "2022-11-02T05:48:12Z"
    message: could not rollback to invalid version v0
    reason: Failed
    status: "False"
    type: Progressing
```
