# Running deployments of cluster-manager and klusterlet outside of Hub and ManagedCluster

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary

The proposed work would support running deployments of cluster-manager and klusterlet outside of Hub and ManagedCluster. 

## Motivation

Currently, the cluster manager and klusterlet should be deployed on the hub/managed cluster, it is mandatory to expose the hub cluster to the managed cluster even if the managed cluster resides in an insecure environment. 

To address this, we propose to provide an option for running cluster-manager/klusterlet outside of the hub/managed cluster.

This ability will bring some additional possible benefits:
* We can leverage compute resources better by running several cluster-managers/klusterlets deployments on the same powerful cluster; it also enhances our management of all deployments.
* The footprints of the hub/managed cluster can be reduced.
* We can use this ability to run the klusterlet on the hub cluster, so we do not need to expose the hub cluster to the managed cluster, and accordingly, we expose the managed cluster to the hub.

### Goals

* Support deploy cluster-manager outside of the hub cluster.
* Support deploy klusterlet outside of the managed cluster.

### Non-Goals

* Support deploy addons agents outside of the managed cluster.
* Authenticate managed cluster without using CSR(CertificateSigningRequest).

## User Stories

### Story1

As a managed cluster administrator, I hope as few as possible workloads running in my cluster while the cluster can be managed by the hub.

### Story2

As an administrator of several Hubs, I want to run all deployments in one place for a better management experience.

### Story3

As a hub cluster administrator, on the premise that an untrusted cluster can be managed, I do not want the cluster to talk to the hub cluster directly. 

### The Comparison Between "Hosted mode" and "Push mode"

They both can solve the same "untrusted cluster" issue in the network. 

But the "push mode" will need listwatch on multiple spokes, which has been proven to be not scalable in kubefed since a single controller will consume a lot of memory.

However, in the "hosted mode", there is no such a "center controller" to "monitor" every spoke. Instead, it could even distribute the computation in multiple nodes. 

## Proposal

Introduce a mode of deploying, named "Hosted" mode, which means deploying cluster-manager/klusterlet outside of the hub/managed cluster.

### Architecture

#### Architecture of Hosted cluster-manager

```
        +---------------------------------------------------------+                                               
        |  hosting-cluster                                        |                                               
        |                                                         |                                               
        |  +----------------+  +-------------+ +------------+     |                                               
        |  |deployment:     |  | deployment: | |deployment: |     |                                               
        |  |cluster-manager |  | registration| |placement   | etc.|                                               
        |  |                |  |             | |            |     |                                               
        |  +------|---------+  +------|------+ +------|-----+     |                                               
        |         |                   |               |           |                                               
        +---------|-------------------|---------------|-----------+                                               
                  |                   |               |                                                           
                  |--------------|--------------------+                                                           
                                 |watch,create,update,delete...                                                   
                                 |                                                                                
        +------------------------v--------------------------------+                                               
        |  hub-cluster                                            |       registrater       +-----------------+   
        |                                                         <-------------------------> managed cluster |   
        |  +-------+ +-----------+ +---------------+              |                         +-----------------+   
        |  | CRDs  | |APIServices| |Configurations | etc.         |                                               
        |  |       | |           | |               |              |                                               
        |  +-------+ +-----------+ +---------------+              |                                               
        +---------------------------------------------------------+
```

The **hub-cluster** is where hub is installed and where the cluster agent should register.
 
The **hosting-cluster** is the cluster where deployments are actually running on. The deployments of several hub-clusters could run in one hosting-cluster and be divided by different namespaces.

Note that cluster-manager is deploy on **hosting-cluster** but configured with a kubeconfig secret named `external-hub-kubeconfig`. The secret is a cluster-admin kubeconfig of the **hub-cluster** and installed in the same namespace as other hub components.

#### Architecture of Hosted klusterlet

```
                      +--------------+                                                                            
                      |  hub cluster |                                                                            
                      +-------^------+                                                                            
                              |                                                                                   
                              |                                                                                   
                              |                                                                                   
+-----------------------------|------------------------------------+                                              
|  hosting-cluster                                                 |                                              
|                                                                  |                                              
| +--------------+  +--------------------+  +---------------+      |                                              
| | deployment:  |  | deployment:        |  | deployment:   |  etc.|                                              
| | klusterlet   |  | registration-agent |  | work-agent    |      |                                              
| +------|-------+  +--------------------+  +-------|-------+      |                                              
|        |                    |                     |              |                                              
+--------|--------------------|---------------------|--------------+                                              
         |                    |                     |                                                             
         |                    |                     |                                                             
         ---------------------|----------------------                                                             
                              | watch,create,update,delete...                                                     
                              |                                                                                   
+-----------------------------v------------------------------------+                                              
|  managed-cluster                                                 |                                              
|                                                                  |                                              
|  +----------------------+   +----------------------+             |                                              
|  | CRDs:                |   | Configurations:      |             |                                              
|  | clusterClaim         |   | role,rolebinding,sa  |   etc.      |                                              
|  | appliedmanifestwork  |   |                      |             |                                              
|  +----------------------+   +----------------------+             |                                              
+------------------------------------------------------------------+  
```

The **managed-cluster** is the cluster managed by the hub cluster, manifestwork will be deployed onto this cluster.
The **hosting-cluster** is the cluster where Klusterlet related deployments are actually running.

The managed cluster will not connect to the hub cluster, all things done by the hosting cluster. The hosting cluster could be the hub cluster, it could be another third-party cluster as well.

In Hosted mode, the "klusterlet.spec.namespace" will only be used on the managed cluster, all agent components will be installed to the namespace with the same name as klusterlet on the hosting cluster. The other thing to note is that users are responsible for creating an `external-managed-kubeconfig` in the agent installation namespace, Klusterlet-operator will use this kubeconfig to get a new minimum permission kubeconfig for the managed cluster, then render it to registration-agent and work-agent.

In addition, the addons agents are not considered to be deployed outside of the managed cluster currently. Here are some reasons:
* In Hosted mode, the deploying of addon-agent will be more complex: the addon-agents deployment runs on the hosting cluster; the role, service account, rolebindings runs on the managed cluster; maybe there will be some other things that need to be deployed and we don’t know where they should be deployed.
* We are not sure that all addons agents pods can run on the hosting cluster, maybe some addons agents running on the hosting cluster will cause the function to fail.

#### Architecture of a combined usage of both side

```
                    +-----------------------------------------------------------------+                                 
                    | hosting-cluster                                                 |                                 
                    |                                                                 |                                 
                    | cluster-manager                         klusterlet              |                                 
                    | +-------------------+                   +-------------------+   |                                 
                    | | +-------------------+                 | +-------------------+ |                                 
                    | | |                 | |<--------------->| |                 | | |                                 
                    | +-|-----------------+ |                 +-|-----------------+ | |                                 
                    |   +-------------------+                   +-------------------+ |                                 
                    +-------|----------------------------------------------|----------+                                 
                            |                                              |                                            
                            |     <private cloud>  |-----------------------|----------------------
                            |                      |  <public cloud>       |                                            
                   watch,create,update,delete...   |         watch,create,update,delete...                               
                            |                      |                       |                                            
                    +-------v-----------+          |           +-----------v-----------+                                 
                    |hub                |          |           |managed cluster        |                                 
                    |                   |     logic|ally       |                       |                                 
                    | +--------+        |<- - - - -|- - - - - >| +--------+            |                                 
                    | | CRDs...|        |          |           | |CRDs... |            |                                 
                    | +--------+        |          |           | +--------+            |                                 
                    +-------------------+          |           +-----------------------+     
```

A real use case would be: The hub cluster is installed in a **private network** while the managed cluster is in a public cloud. right now, the managed cluster cannot connect to the hub cluster. **It also means deployments on managed cluster can not apply any resources to the hub cluster directly**.

But with our proposal, as you can see in the above diagram: The components of cluster-manager and the components of klusterlet are installed in the same cluster -- the hosting cluster which is also in the private cloud.

Therefore, instead of applying resources such as `ManagedCluster`  from the managed cluster in the private cluster, the klusterlet is applying the resources from the hosting cluster, and the hub cluster is accessible for the hosting cluster.

#### Minimize the permission of the external kubeconfig

In Hosted mode, users are required to provide an `external-hub/managed-kubeconfig`, klusterlet-operator will use this kubeconfig to create roles, service accounts, rolebindings with the minimum permission that the registration/work needs in the hub/managed cluster, and retrieve a token from the secret that the created service account relates in the hub/managed cluster to construct the external hub/managed cluster kubeconfig for registration and work.


### API Changes

At API level, we add a field "DeployOption" to the "ClusterManagerSpec" and "KlusterletSpec". The detail comes with the following:

```golang
// DeployOption describes the deploy options for cluster-manager or klusterlet
type DeployOption struct {
	// Mode can be Default or Hosted.
	// For cluster-manager:
	//   - In Default mode, the Hub is installed as a whole and all parts of Hub are deployed in the same cluster.
	//   - In Hosted mode, only crd and configurations are installed on one cluster(defined as hub-cluster). Controllers run in another cluster (defined as hosting-cluster) and connect to the hub with the kubeconfig in secret of "external-hub-kubeconfig"(a kubeconfig of hub-cluster with cluster-admin permission).
	// For klusterlet:
	//   - In Default mode, all klusterlet related resources are deployed on the managed cluster.
	//   - In Hosted mode, only crd and configurations are installed on the spoke/managed cluster. Controllers run in another cluster (defined as hosting-cluster) and connect to the mangaged cluster with the kubeconfig in secret of "external-managed-kubeconfig"(a kubeconfig of managed-cluster with cluster-admin permission).
	// The purpose of Hosted mode is to give it more flexibility, for example we can install a hub on a cluster with no worker nodes, meanwhile running all deployments on another more powerful cluster.
	// And we can also register a managed cluster to the hub that has some firewall rules preventing access from the managed cluster.
	//
	// Note: Do not modify the Mode field once it's applied.
	//
	// +required
	// +default=Default
	// +kubebuilder:validation:Required
	// +kubebuilder:default=Default
	// +kubebuilder:validation:Enum=Default;Hosted
	Mode InstallMode `json:"mode"`

	// Hosted includes configurations we needs for clustermanager in the hosted mode.
	// +optional
	Hosted *HostedClusterManagerConfiguration `json:"hosted,omitempty"`
}

// HostedClusterManagerConfiguration represents customized configurations we need to set for clustermanager in the hosted mode.
type HostedClusterManagerConfiguration struct {
	// RegistrationWebhookConfiguration represents the customized webhook-server configuration of registration.
	// +optional
	RegistrationWebhookConfiguration WebhookConfiguration `json:"registrationWebhookConfiguration,omitempty"`

	// WorkWebhookConfiguration represents the customized webhook-server configuration of work.
	// +optional
	WorkWebhookConfiguration WebhookConfiguration `json:"workWebhookConfiguration,omitempty"`
}

// WebhookConfiguration has two properties: Address and Port.
type WebhookConfiguration struct {
	// Address represents the address of a webhook-server.
	// It could be in IP format or fqdn format.
	// The Address must be reachable by apiserver of the hub cluster.
	// +required
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Pattern=^(([a-zA-Z0-9]|[a-zA-Z0-9][a-zA-Z0-9\-]*[a-zA-Z0-9])\.)*([A-Za-z0-9]|[A-Za-z0-9][A-Za-z0-9\-]*[A-Za-z0-9])$
	Address string `json:"address"`

	// Port represents the port of a webhook-server. The default value of Port is 443.
	// +optional
	// +default=443
	// +kubebuilder:default=443
	// +kubebuilder:validation:Maximum=65535
	Port int32 `json:"port,omitempty"`
}

// InstallMode represents the mode of deploy cluster-manager or klusterlet
type InstallMode string

const (
	// InstallModeDefault is the default deploy mode.
	// The cluster-manager will be deployed in the hub-cluster, the klusterlet will be deployed in the managed-cluster.
	InstallModeDefault InstallMode = "Default"

	// InstallModeHosted means deploying components outside.
	// The cluster-manager will be deployed outside of the hub-cluster, the klusterlet will be deployed outside of the managed-cluster.
	InstallModeHosted InstallMode = "Hosted"
)
```

We also need to make the `Mode` field immutable by setting up a webhook server.

### Graduation Criteria

Alpha:
* Hosted mode is implement and works well on KinD cluster for both cluster-manager and klusterlet

### Test Plan

Alpha:
* Unit tests that makes sure the project's fundamental quality.
* Integration tests against real KinD clusters to exercise the installation.

