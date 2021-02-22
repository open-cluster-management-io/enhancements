# Addon Framework

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary

The is proposal is for a start to standardize the addon framework. The framework consists of two parts:
- A lightweight part that only provide cluster registration and manifest resource distribution functions (registration-operator)
- A bunch of addons that can easily enabled/disabled on the lightweight part.

Commonly an addon will have an agent component that runs on managed cluster, which should have at least one of the following characteristic:

1. It needs to talk to hub with certain ACLs
2. Its configuration diffs in different managed cluster.

we have `ClusterManagerAddon` and `ManagedClusterAddon` APIs defined already, so that an addon controller on the hub can be represented as a ClusterManagerAddon resource and an addon agent can be represented as a ManagedClusterAddon resource in  the cluster namespace on the hub.

## Motivation

With a standardize addon framework, it allows users to easily managed each function a separate addon. In addition, it will be possible to provide a an addon framework library so developers could be able to develop their own addon functions more easily.

### Goals

- Standardize the addon registration/enablement procedure
    - Enable/disable addon
    - Communication pattern
    - Authn/Authz
- Provide an addon framework interface to ease the addon development


### Non-Goals

- Implementation details of addon

## Proposal

### User Stories

#### Story 1

As an open-cluster-management admin on hub, I can add an addon into the hub. For example, I want to install the submariner addon into the hub.

#### Story 2

As an open-cluster-management admin on hub, I can enable the addon installation on all managedclusters.

#### Story 3

As an admin on a managed cluster, I can enable and config the addon agent on a certain managed cluster. I was able to enable the addon agent on hub, which triggers the agent installation on the managed cluster.

#### Story 4

As an addon developer, I can easily develop the addon using the library without deep understanding on how agent manifests are distributed and registerd to hub.


### Standardize addon procedure

#### Register an addon on hub

A common process for an addon to be installed on the hub cluster follows the procedure below:

1. Cluster admin on hub deploy a certain addon controller
2. Addon controller registers itself to the hub by creating a ClusterManagerAddon resource on hub
```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddon
metadata:
 name: submariner
spec:
 addonMeta:
   displayName: submariner
   description: submariner-addon
 addonConfiguration:
   crd: submarineraddonconfig.addon.open-cluster-management.io
   name: submariner-config
```

#### Enable addon agent on a managed cluster

1. A user creates a ManagedClusterAddon resource of which the name is the same as the name of ClusterManagerAddon resource in the cluster namespace.
```yaml
 apiVersion: addon.open-cluster-management.io/v1alpha1
 kind: ManagedClusterAddon
 metadata:
   name: submariner
   namespace: cluster1
```
2. Addon controller on hub deploys the agent by creating manifestworks in cluster namespace.
3. The agent is started on a managed cluster, and registers back using credential.

#### Communication pattern

An addon agent in a managed cluster will communicate with the hub:
- The addon agent talks to the kube apiserver of the hub.  For reading desired state and for pushing small quantities of data that are stored in kubernetes API objects, this is the recommended path.
- The addon agent talks to another endpoint on the hub.  For writing large amounts of data to the hub or sending data that doesn’t fit cleanly into a kubernetes style API, this is the recommended approach.

In both ways, the communication should be properly authenticated/authorized.

##### mTLS for authentication

The addon agent should be able to register to the hub by issuing a csr request and choose the signer. If the addon agent talks to the kube-apiserver of the hub, the agent should use the `kubernetes.io/kube-apiserver-client` to create the csr. Otherwise, the addon agent should use a customized signer for different endpoints. The endpoint, which supposedly belongs to the hub component of the addon should be able to accept the cert signed by the customized signer for this addon.

##### Permission for the addon agent

The addon should have a controller on hub to correctly set the permission for an addon agent to talk to the hub. 
1. An addon agent may only be able to access certain resources on the cluster namespace on the hub.
2. An addon agent may only be able to access certain resources on a specified namespace (not cluster namespace) on the hub.
3. An addon agent may be able to access a certain url on a non kube-apiserver endpoint.

The addon controller on the hub should have a reconcile loop that when an addon is enabled on a cluster, the permission of the addon agent can be set correctly. 

### Addon framework interfaces

To ease the development of addon controllers, it is possible that we provide a framework library to encapsulate the common part, so that developers of addon controllers only need to care about the manifests of addon agents that should be deployed on a managed cluster. 

#### Addon Manager

```go
// AddonManagerContext defines the context of addon manager.
type AddonManagerContext struct {
   // KubeConfig provides the REST config with no content type (it will default to JSON).
   KubeConfig *rest.Config
 
   // AddonName is the name of the addon
   AddonName string
 
   // AddonNamespace is the namespace to deploy addon agent on spoke
   AddonNamespace string
 
   // ConfigurationGVR is the gvr of the resource for configuring addon
   ConfigurationGVR *schema.GroupVersionResource
}

// AddonManager defines an addon manager on hub to trigger the installation and configuration
// of addon agent on managed cluster.
type AddonManager struct

// NewAddonManager returns an addon manager
func NewAddonManager(managerContext *AddonManagerContext, agentAddon addoninterface.AgentAddon) *AddonManager

// Enable addon to be registered onto the hub using csr
func (a *AddonManager) WithRegistrationEnabled(bootstrapKubeConfig []byte) *AddonManager {


func (a *AddonManager) WithSigner(signer string) *AddonManager

 // A list of function to check whether the csr created by the addon agent should be approved by the manager
// Commonly, we need to check whether the requester is valid and whether CN/ORG field in the request is valid.
// If it is not set, only renewed csr is approved.
func (a *AddonManager) WithCSRApproveFunc(approveFuncs ...addoninterface.CSRApproveCheckFunc)

// Run starts the addon manager
func (a *AddonManager) Run(ctx context.Context) error
```

#### Implement agent addon interface

```go
// AgentAddon defines an interface on manifests of agent deployed on managed cluster and rbac rules of agent on hub
type AgentAddon interface {
   // AgentManifests returns a list of manifest resources to be deployed on the managed cluster for this addon
   AgentManifests(cluster *clusterv1.ManagedCluster, config runtime.Object) ([]runtime.Object, error)
 
   // AgentHubRBAC returns the role and rolebinding on the hub for an agent on the managed cluster.
   AgentHubRBAC(cluster *clusterv1.ManagedCluster, group string) (*rbacv1.Role, *rbacv1.RoleBinding)
}
```

### Test Plan

E2E tests can use kind to create multiple clusters to test. An example should be created using addon framework for the purpose of testing

- An addon agent can be successfully intalled on Cluster A
- An addon agent have correct permission to connect to hub after registration

### Graduation Criteria

#### Alpha -> Beta Graduation

- E2E tests exists
- Beta -> GA Graduation criteria defined.
- At least one implementation using addon framework library.

#### Beta -> GA Graduation

- Scalability/performance testing, under large number of clusters.

### Upgrade / Downgrade Strategy

registration-operator must be updated to install `ManagedClusterAddon` and `ClusterManagerAddon` APIs.

### Version Skew Strategy

registration-operator must be upgraded before running any addons.