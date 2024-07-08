# Addon Multiple Configs with Same GVK

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal is to propose an enhancement to the addon configurations, enables users to configure multiple configs of the same GVK.

## Motivation

Users have three locations to set [addon configurations](https://open-cluster-management.io/concepts/addon/#add-on-configurations): 
1. In `ClusterManagementAddon` `spec.supportedConfigs`, user can set the `defaultConfig` where all the addons to have same configurations.
2. In `ClusterManagementAddOn`, `spec.installStrategy.placements[].configs`, user can set configs for addons on a group of clusters selected by placement.
3. In `ManagedClusterAddOn` `spec.configs`, user can set configs for addon on a specific cluster.

Currently, configs with the same GVK only support a single config. The config with the same GVK in place `3` will override the config in place `2`, 
which will override the config in place `1`. The final effective addon configs are listed in the mca.status.

Now we are seeing the requirement to set multiple configs for the same GVK. For example, for `OpenTelemetryCollector`, one CR might be configured to 
collect traces and another to collect logs on the same cluster. This proposal aims to satisfy that requirement.

### Goals

- Enhance `ClusterManagementAddon` and `ManagementClusterAddon` API to allow user to configure mutiple addon configs with same GVK.

### Non-Goals

- `ClusterManagementAddon` `spec.supportedConfigs` does not allow user to configure mutiple addon configs with same GVK in the `defaultConfig`.

## Proposal

### User Stories

1. The admin can configure the `ClusterManagementAddon` API to set mutilple configs with same gvk per install strategy.
2. The admin can configure the `ManagementClusterAddon` API to set mutilple configs with same gvk for specific cluster.

## Design Details

### API changes

`cma.spec.installStrategy.placements[].configs` and `mca.spec.configs` already support mutiple configs. 

```go
	// Configs is the configuration of managedClusterAddon during installation.
	// User can override the configuration by updating the managedClusterAddon directly.
	// +optional
	Configs []AddOnConfig `json:"configs,omitempty"`
```

### Effective addon configs

As the motivation mentioned, today there are 3 places to set addon configurations. 

Currently, since each GVK supports only one config, the GVK is used as the identifier for configs. When configs with same GVK configured in multiple places, it follows the override sequence and 
only one config is listed as the effective config in the mca.status.

With the support for multiple configs with the same GVK, `GVK + namespace + name` becomes the identifier. 
We need to reconsider the logic for setting mca.status when configuring multiple configs in multiple places.

Using the example below (omitting the group and namespace for simplicity), we present three options. 
We do not expect to support all, but want to involve discussion and finally only choose one.

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

#### Merge

With `GVK + namespace + name` as the identifier for each config, there is no override concept now actually, all the configured configs will be listed in the mca.status. 

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

#### Override by GVK

"Override by GVK" is a way try to keep consistent with current behavior. 

The gaps is if the r2n1 and r2n2 resource are actually some global configs, you may not want it to be overriden by r2n3. Compared with "Merge" options, 
this options can not cover such case and is lack of flexibility.

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

### Effective config values

`WithGetValuesFuncs()` depends on how the funcs handle multiple configs, for example: 
[GetAddOnDeploymentConfigValues()](https://github.com/open-cluster-management-io/addon-framework/blob/main/pkg/addonfactory/addondeploymentconfig.go#L144) will only read the last object list in mca.status.

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

For other addons, could write their own `WithGetValuesFuncs()` to determine how to use the listed configs in mca.status. 

### How to trigger rollout

TBD

### Test Plan

- e2e test on setting configs in `ClusterManagementAddon` `spec.supportedConfigs`.
- e2e test on setting multiple configs in `ClusterManagementAddOn` `spec.installStrategy.placements[].configs`.
- e2e test on setting multiple configs in `ManagedClusterAddOn` `spec.configs`.

### Graduation Criteria

TBD

### Upgrade / Downgrade Strategy

TBD

## Alternatives

N/A