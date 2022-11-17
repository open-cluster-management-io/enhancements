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
How to enhance the existing addon framework and addon manager to support multiple version addon installtion is not this 
proposal's scope.

## Proposal

### Use cases

#### Story 1
- As a user, I can configure a desired version of my addon to install for a given managed cluster. 
- I can know the installation status, including the addon version(desired version, current version), 
progress (installing, failed, succeeded) and completed time..

#### Story 2
- As a user, I can configure a desired version of my addon to upgrade for a given managed cluster. 
- I can know the upgrade status, including the addon version(desired version, current version), 
progress (upgrading, failed, succeeded) and completed time..

#### Story 3
- As a user, I can configure a desired version of my addon to rollback for a given managed cluster. 
- I can know the rollback status, including the addon version(desired version, current version), 
progress (rolling back, failed, succeeded) and completed time.

### Risks and Mitigation
N/A

## Design Details
The design is based on [Addon install strategy #77](https://github.com/open-cluster-management-io/enhancements/pull/77/files). 

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
	
  // installVersion represents the desired addon version to install.
  // Valid installVersion are listed in ClusterManagementAddOn status.supportedVersion.
  // If installVersion is not specified, the ClusterManagementAddOn status.defaultVersion 
  // wil be used.
  // +optional
  InstallVersion string `json:"installVersion,omitempty"`

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
  // If the addon installation is in progressing, the value is unknown.
  // +optional
  CurrentVersion string `json:"currentVersion,omitempty"`

  // lastVersion record the previous defined installVersion.
  // If the previous addon installVersion is invalid, the value is unknown.
  // If no previous addon installVersion, the value is empty.
  // +optional
  LastVersion string `json:"lastVersion,omitempty"`
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
        installVersion: v1
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: global-config
    - name: canary-placement
      namespace: default
      addonTemplate:
        installVersion: v2
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: canary-config
status:
  supportedVersions:
  - v1
  - v2
  defaultVersion:
  - v2
```

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: helloworld
  namespace: cluster1
spec:
  installNamespace: open-cluster-management-agent-addon
  installVersion: v2
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
has label `open-cluster-management.io/addon-version` to indicate the addon version. 
- Addon manager copies the last `ManagedClusterAddOn` status.installVersion as status.lastVersion.
- Addon manager update the `ManagedClusterAddOn` status.currentVersion based on `ManifestWork` status.
- Addon manager updates the `ManagedClusterAddOn` status.condition to tell the user whether installation/upgrade/rollback 
is succeeded or failed.  

**The installVersion, lastVersion and currentVersion**

When `ManagedClusterAddOn` installVersion changes, the addon manager will update the `ManagedClusterAddOn` status.lastVersion 
and status.currentVersion:
- Addon manager copies previous spec.installVersion as the status.lastVersion. 
  (Fresh install doesn’t have lastVersion, invalid version is treated as unknown.)
- Addon manager updates the status.currentVersion to unknown when `ManifestWork` installation/upgrade/rollback is not finished.
- Addon manager updates status.currentVersion as spec.installVersion when `ManifestWork` installation/upgrade/rollback is finished.
  - `ManifestWork` has correct “addon-name” and “addon-version” labels.
  - `ManifestWork` Condition Available status is true.
  - `ManifestWork` Condition Available observedGeneration equals to generation.
  - If it is a fresh install, since one addon can have multiple ManifestWorks, the `ManifestWork` condition ManifestApplied 
  must also be true.

**Condition**

Addon manager also updates the `ManagedClusterAddOn` status.condition to tell user 
whether installation/upgrade/rollback is succeeded or failed, lastTransitionTime record the finish time.   
- If currentVersion==installVersion, update condition Progressing status to false with reason Succeed.
- If currentVersion==unknown
  - lastVersion < installVersion, update condition Progressing status to true with reason Upgrading.
  - lastVersion > installVersion, update condition Progressing status to true with reason Rollingback.
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
  defaultVersion:
  - v1
```

If installVersion is not defined in `ManagedClusterAddOn`, use the defaultVersion v1.
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
  defaultVersion:
  -v2
```

Since the defaultVersion is upgraded to v2, the `ManagedClusterAddOn` default installVersion is updated to v2.
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
        installVersion: v1
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: global-config
status:
  supportedVersions:
  - v1
  defaultVersion:
  - v1
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
        installVersion: v1
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: global-config
status:
  supportedVersions:
  - v1
  - v2
  defaultVersion:
  - v2
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
        installVersion: v1
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: global-config
    - name: canary-placement
      namespace: default
      addonTemplate:
        installVersion: v2
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: canary-config
status:
  supportedVersions:
  - v1
  - v2
  defaultVersion:
  - v2
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
  installVersion: v2
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
  installVersion: v2
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
  installVersion: v3
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
        installVersion: v1
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: global-config
    - name: canary-placement
      namespace: default
      addonTemplate:
        installVersion: v1
        configs:
        - group: addon.open-cluster-management.io
          resource: addondeploymentconfigs
          name: canary-config
status:
  supportedVersions:
  - v1
  - v2
  defaultVersion:
  - v2
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
  installVersion: v1
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
  installVersion: v1
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
  installVersion: v0
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
N/A
