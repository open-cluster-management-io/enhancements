# Addon Multiple Configs with Same GVK

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal proposes an enhancement to the addon configurations, enabling users to configure multiple configs of the same GVK.

## Motivation

Users have three locations to set [addon configurations](https://open-cluster-management.io/concepts/addon/#add-on-configurations) today: 
1. In `ClusterManagementAddon` `spec.supportedConfigs`, the user can set the `defaultConfig` to declare a default configuration for all the addons.
2. In `ClusterManagementAddOn`, `spec.installStrategy.placements[].configs`, the user can declare configuration for addons on a group of clusters selected by placement.
3. In `ManagedClusterAddOn` `spec.configs`, the user can declare the configuration for the addon on a specific cluster.

Currently, configs with the same GVK only support a single config. The config with the same GVK in place `3` will override the config in place `2`, 
which will override the config in place `1`. The `mca.status` lists the final effective addon configuration. Using `AddonDeploymentConfig` resource as an example, 
a user can declare, in place `1`, a default `AddonDeploymentConfig`.  Additionally, the user can also declare, in place `2`, an `AddonDeploymentConfig` for a group of clusters, which will
override the default config. Finally, the user can declare, in place `3`, for a specific cluster another `AddonDeploymentConfig`, which will override the ones in `1` and `2`.

Now we are seeing the requirement to set multiple configs for the same GVK. For example, for `OpenTelemetryCollector`, the user may want to define 2 `OpenTelemetryCollector`
in place `2`, one CR is configured to collect traces and another to collect logs on the same cluster. This proposal aims to satisfy that requirement.


### Goals

- Enhance `ClusterManagementAddon` and `ManagementClusterAddon` API to allow users to configure multiple addon configs with the same GVK.

### Non-Goals

- `ClusterManagementAddon` `spec.supportedConfigs` does not allow the user to configure multiple addon configs with the same GVK in the `defaultConfig`.

## Proposal

### User Stories

1. The admin can configure the `ClusterManagementAddon` API to set multiple configs with the same GVK per install strategy.
2. The admin can configure the `ManagementClusterAddon` API to set multiple configs with the same GVK for a specific cluster.

## Design Details

### API changes

No API changes are needed, `cma.spec.installStrategy.placements[].configs` and `mca.spec.configs` already support multiple configs. 

```go
	// Configs is the configuration of managedClusterAddon during installation.
	// User can override the configuration by updating the managedClusterAddon directly.
	// +optional
	Configs []AddOnConfig `json:"configs,omitempty"`
```

### Effective addon configs

As the motivation mentioned, today there are 3 places to set addon configurations. 

Currently, since each GVK supports only one config, the GVK is used as the identifier for configs. When configs with the same GVK configured in 
multiple places, it follows the override sequence and only one config is listed as the effective config in the `mca.status`. 

With the support for multiple configs with the same GVK, `GVK + namespace + name` becomes the identifier. 
We need to reconsider the logic for setting `mca.status` when configuring multiple configs in multiple places.

In the initial discussion we listed three options, finally choose the below "Override by GVK" option, as it better satisfies a real use case. 
The other two are listed in the **Alternatives** for future needs.

The below example omits the group and namespace for simplicity. 

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: example
spec:
  supportedConfigs:
    - resource: r1
      defaultConfig:
        name: n1
    - resource: r2
  installStrategy:
    type: Placements
    placements:
      - name: example
        namespace: example
        configs:
          - resource: r2
            name: n1
          - resource: r2
            name: n2


apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: example
  namespace: cluster1
spec:
  installNamespace: open-cluster-management-agent-addon
  configs:
    - resource: r2
      name: n3
```

#### Override by GVK

"Override by GVK" is a way try to keep consistent with current behavior. GVK is used as the identifier for a group of configs.

In the above example, config `r2` `n3` in `ManagedClusterAddOn` (place `3`) will override `r2` `n1` and `n2` in `installStrategy` (place `2`). The default config `r1` `n1` will remain, since no other config for resource `r1` is defined in either place `2` or `3`.
The `mca.status` will look like the following:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: example
  namespace: cluster1
status:
...
  configReferences:
  - desiredConfig:
      name: n3
      specHash: <spec-hash>
    name: n3
    resource: r2
  - desiredConfig:
      name: n1
      specHash: <spec-hash>
    name: n1
    resource: r1
```

The limitation is if the `r2n1` and `r2n2` resources are some global configs, the user may not want it to be overridden by `r2n3`. In this case, the user need to define
both `r2n1`,`r2n2` and `r2n3` in `ManagedClusterAddOn`.

### Effective config values

`WithGetValuesFuncs()` depends on how some functions handle multiple configs, for example: 
[GetAddOnDeploymentConfigValues()](https://github.com/open-cluster-management-io/addon-framework/blob/main/pkg/addonfactory/addondeploymentconfig.go#L144) will only read the last object list in `mca.status`. 

```go
// GetAddOnDeploymentConfigValues uses AddOnDeploymentConfigGetter to get the AddOnDeploymentConfig object, then
// uses AddOnDeploymentConfigToValuesFunc to transform the AddOnDeploymentConfig object to Values object
// If there are multiple AddOnDeploymentConfig objects in the AddOn ConfigReferences, the big index object will
// override the one from small index
func GetAddOnDeploymentConfigValues(
		getter utils.AddOnDeploymentConfigGetter, toValuesFuncs ...AddOnDeploymentConfigToValuesFunc) GetValuesFunc {
	...
	}
```

For other addons, could write their own `WithGetValuesFuncs()` to determine how to use the listed configs in `mca.status`. 

### How to trigger rollout

Below is the current rollout logic and step 2 needs to be enhanced since one GVK will have multiple configs.

1. The [buildConfigurationGraph](https://github.com/open-cluster-management-io/ocm/blob/release-0.13/pkg/addon/controllers/addonconfiguration/controller.go#L166) 
will build and list the desired configs for each mca. 
2. The [setRolloutStatus](https://github.com/open-cluster-management-io/ocm/blob/release-0.13/pkg/addon/controllers/addonconfiguration/graph.go#L54) 
will compare the actual configs in `mca.Status.ConfigReferences` with desired configs, using GVK as key, compare the actual and desired name+namespace+hash, if not match, set status to ToApply.
3. The [generateRolloutResult](https://github.com/open-cluster-management-io/ocm/blob/release-0.13/pkg/addon/controllers/addonconfiguration/controller.go#L132) 
will get clusters that need to rollout (with ToApply status) based on the rollout strategy. 

### Test Plan

- e2e test on setting configs in `ClusterManagementAddon` `spec.supportedConfigs`.
- e2e test on setting multiple configs in `ClusterManagementAddOn` `spec.installStrategy.placements[].configs`.
- e2e test on setting multiple configs in `ManagedClusterAddOn` `spec.configs`.

### Graduation Criteria

TBD

### Upgrade / Downgrade Strategy

TBD

## Alternatives

### Effective addon configs
#### Merge

With `GVK + namespace + name` as the identifier for each config, there is no overriding concept now actually, all the configured configs will be listed in the `mca.status`. 

Configs will be ordered, with the last configured on top. For example, configs in mca will be on top of cma configs, and mca configs with higher indices 
will be on top of those with lower indices.

The addon's own controller will determine how to use the configs (e.g., choose one, read all, etc).

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: example
  namespace: cluster1
status:
...
  configReferences:
  - desiredConfig:
      name: n3
      specHash: <spec-hash>
    name: n3
    resource: r2
  - desiredConfig:
      name: n2
      specHash: <spec-hash>
    name: n2
    resource: r2
  - desiredConfig:
      name: n1
      specHash: <spec-hash>
    name: n1
    resource: r2
  - desiredConfig:
      name: n1
      specHash: <spec-hash>
    name: n1
    resource: r1
```

#### Override

"Override" is straightforward, any configs in the mca will override the configs in cma install strategy and override the configs in cma default configs.

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: example
  namespace: cluster1
status:
...
  configReferences:
  - desiredConfig:
      name: n3
      specHash: <spec-hash>
    name: n3
    resource: r2
```