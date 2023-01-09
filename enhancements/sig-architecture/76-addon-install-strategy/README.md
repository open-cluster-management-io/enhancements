# Addon Install Strategy

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal is to propose an enhancement of the `ClusterManagementAddon` API, so user can configure which clusters the related
`ManagedClusterAddon` should be enabled.

## Motivation

We are seeing requirements that user would like the addon to be automatically installed when
a new cluster is registered. We provided a [installStrategy interface](https://github.com/open-cluster-management-io/addon-framework/blob/main/pkg/agent/inteface.go#L56)
in `addon-framework`, so the addon developer can select to automatically enable the `ManagedClusterAddon` in all or selected clusters. 
However, this mechanism is not user friendly: 
- An addon user cannot easily change the install strategy. For example, the user may want to disable the addon in certain clusters by deleting `ManagedClusterAddon`, but the addon-manager would recreate it.
- It also confuses the addon developers, since we do not have an API standard to define the install strategy.

### Goals

- Enhance `ClusterManagementAddon` API to allow user to configure addon install strategy
- A centralized addon-install-controller on hub being responsible of creation/deletion of `ManagedClusterAddon`

### Non-Goals

N/A

## Proposal

### User Stories

#### Story 1

The admin can configure the `ClusterManagementAddon` API to automatically install a `ManagedClusterAddon` in all clusters
or selected clusters.

#### Story 2

The admin can select NOT to install a `ManagedClusterAddon` in one or several clusters.

#### Story 3

The addon developer should not care about which cluster the `ManagedClusterAddon` should be installed.

## Design Details

### ClusterManagementAddon API change

```go
type InstallStrategy struct {
	// Type is the type of the install strategy, it can be:
	// - Manual: no automatic install
	// - ClusterLabelSelector: install to clusters with certain labels
	// - Placements: install to clusters selected by placements.
	// +kubebuilder:validation:Enum=Manual;Placements
	// +kubebuilder:default:=Manual
	Type string `json:"type"`

	// ClusterLabelSelector is a label selector honored when install strategy type is
	// ClusterLabelSelector
	ClusterLabelSelector *metav1.LabelSelector `json:"labelSelector,omitempty"`

	// Placements is a list of placement references honored when install strategy type is
	// Placements. All clusters selected by these placements will install the addon
	Placements []PlacementRef `json:"placements,omitempty"`
}

type PlacementRef struct {
	// Name of the placement
	// +required
	Name string `json:"name"`

	// Namespace of the placement
	// +required
	Namespace string `json:"space"`

	// AddonTemplate is the template to generate ManagedClusterAddon. It takes effect only when creating the ManagedClusterAddon. If
  // the setting of a ManagedClusterAddon is updated by the user, it will not be reverted to the setting here.
	AddonTemplate ManagedClusterAddOnSpec `json:"addonTemplate,omitempty"`
}
```

### Install addon according to a placement

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
```

- when a cluster is added into the related `PlacementDecision`, the `ManagedClusterAddon` will be created on the cluster namespace.
- when a cluster is removed from the related `PlacementDecision`, the `ManagedClusterAddon` will be deleted on the cluster namespace.

### Install addons based on multiple placement with different configuration

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
      addonTemplate:
        installNamespace: aws-addon-ns
    - name: gcp-placement
      namespace: default
      addonTemplate:
        installNamespace: gcp-addon-ns
```

This will ensure that addon installed in aws uses aws-config while addon installed in gcp uses gcp config. If the user updates the config of a certain `ManagedClusterAddon`, it will NOT be reverted based on the configs in `ClusterManagementAddon`. The setting in `ClusterManagementAddon` only impact on creation stage.

### Disable addon in some clusters

At first change the install strategy to `Manual`

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: helloworld
spec:
  installStrategy:
    type: Manual
```

Then admin can delete `ManagedClusterAddon` in certain cluster namespaces.

When the user changes the strategy type from `Placement` to `Manual`, all the created are `ManagedClusterAddon` is kept. `ManagedClusterAddon`
will not be auto-created or deleted anymore.

When the user changes the strategy type from `Manual` to `Placement`, `ManagedClusterAddon` created manually by the the user will NOT be
overriden by the configuration on the `ClusterManagementAddon`.

### Addon Install Controller

We need to build and run a centralized addon install controller on the hub cluster. The addon
install controller will create/delete `ManagedClusterAddon` according to the install strategy
in related `ClusterManagementAddon`. The addon install controller will:
- create or delete but not modify the `ManagedClusterAddon` if it exists already.
- do not create when the `ManagedCluster` is deleting.
- delete all `ManagedClusterAddon` in the cluster namespace when the `ManagedCluster` is deleting.

### Test Plan

- e2e test on using cluster label selector type
- e2e test on using placements type
- e2e test on using manual type

### Graduation Criteria

TBD

### Upgrade / Downgrade Strategy

Addon-framework needs to be upgraded for the addons. The original `InstallStrategy` interface in addon-framework will be marked as deprecated.
If the old addon has not adopted the new API change and still use the `InstallStrategy` interface in addon-framework, the `installStrategy`
set in `ClusterManagementAddon` MUST be Manual.

If an addon upgrade to use the new addon-framework, the related `ClusterManagementAddon` should also be updated to use the appropriate
`installStrategy`.

## Alternatives

N/A
