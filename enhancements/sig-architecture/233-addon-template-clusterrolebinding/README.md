# Addon Template ClusterRoleBinding

## Release Signoff Checklist

- [ ] Enhancement is `provisional`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary
Extend AddonTemplate kubeClient registration to support ClusterRoleBindings for addon agent service accounts. A new hubPermission type called "AllNamespaces" will be added along with an associated AllNamespacesBindingConfig which references a ClusterRole in the hub. The addon manager will create a ClusterRoleBinding for each ManagedClusterAddon that binds the ClusterRole to the subject associated with the spoke addon agent.

## Motivation
Currently hubPermissions only supports creating a RoleBinding for CurrentCluster or SingleNamespace. This means that spoke addon agents can't be granted permissions on non-namespaced resources/custom-resources without a hub addon controller that manages the ClusterRoleBinding for ManagedClusterAddons.

My use-case is for a non-namespaced CRD that describes shared state for the fleet which needs to be distributed out to spoke clusters. The only way for me to do this right now is to add a custom controller in the addon's hub manager that watches ManagedClusterAddons and creates the ClusterRoleBindings itself.

## Proposal
- Add "AllNamespaces" hub permission type, which references a single ClusterRole in the hub
- Addon Manager will manage ClusterRoleBindings for spoke agent addons with AllNamespaces permissions
  - Each ManagedClusterAddon has its own ClusterRoleBinding named `open-cluster-management:<addon-name>:clusterrole:<cluster-name>:agent`

### Design Details

### Permission change

Add `clusterrolebindings` to the `open-cluster-management:{{ .ClusterManagerName }}-addon-manager:controller` ClusterRole

#### API change

Add new HubPermissionsBindingType

```go

const (
    ...

    // HubPermissionsBindingAllNamespaces means the addon agent will have access to resources in any namespace,
    // or to non-namespaced resources on the hub cluster.
    HubPermissionsBindingAllNamespaces HubPermissionsBindingType = "AllNamespaces"
)
```

Add config for AllNamespaces permissions

```go
type HubPermissionConfig struct {
    // Type of the permissions setting. It defines how to bind the roleRef on the hub cluster. It can be:
    // - CurrentCluster: Bind the roleRef to the namespace with the same name as the managedCluster.
    // - SingleNamespace: Bind the roleRef to the namespace specified by SingleNamespaceBindingConfig.
    // - AllNamespaces: Bind the clusterRoleRef to the subject for the managedCluster.
    //
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:Enum:=CurrentCluster;SingleNamespace;AllNamespaces
    Type HubPermissionsBindingType `json:"type"`

    ...

    // AllNamespaces contains the configuration of AllNamespaces type binding.
    // It is required when the type is AllNamespaces
    AllNamespaces *AllNamespacesBindingConfig `json:"allNamespaces,omitempty"`
}

type AllNamespacesBindingConfig struct {
    // ClusterRoleName is the name of the clusterrole the addon agent is bound. A clusterrolebinding
    // will be created referring to this cluster role with subjects for each cluster namespace.
    // The user must make sure the clusterrole exists on the hub cluster.
    // +kubebuilder:validation:Required
    ClusterRoleName string `json:"clusterRoleName"`
}
```

#### hub implementation
Update `createKubeClientPermissions` in `pkg/addon/templateagent/registration.go` to handle the additional hub permission case `addonapiv1alpha1.HubPermissionsBindingAllNamespaces`. This will call a new method `createPermissionClusterRoleBinding` which is nearly the same as the existing `createPermissionRoleBinding` except that it creates a ClusterRoleBinding instead of a RoleBinding.

#### examples

```yaml
apiVersion: addon.open-cluster-management.io/v1beta1
kind: AddOnTemplate
metadata:
  name: my-agent
spec:
  addonName: my-agent
  agentSpec:
    workload:
      manifests:
      - apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: my-agent-addon
          namespace: open-cluster-management-agent-addon
        spec:
          replicas: 1
          selector:
            matchLabels:
              addon-agent: my-agent
          template:
            metadata:
              labels:
                addon-agent: my-agent
            spec:
              containers:
              - name: addon-agent
                image: registry.example.com/my-agent:latest
  registration:
  - type: KubeClient
    kubeClient:
      hubPermissions:
      - type: AllNamespaces
        allNamespaces:
          clusterRoleName: my-agent-hub-permissions
```

### Test Plan
- Verify existing AddonTemplates with hub permissions continue to work as before
- Add AllNamespaces binding for a hub ClusterRole, verify ClusterRoleBinding is created for each ManagedClusterAddon

### Graduation Criteria
N/A

### Upgrade Strategy
Upgrade AddonTemplate CRD. Existing AddonTemplates will not need to be changed.

### Version Skew Strategy
The AllNamespaces hubPermissions type is added on top of existing types. No migration required.
