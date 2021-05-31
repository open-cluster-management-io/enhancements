# Addon Framework

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary

This proposal is for a start to standardize the addon framework. The framework consists of two parts:
- A lightweight part that only provide cluster registration and manifest resource distribution functions (registration-operator)
- A bunch of addons that can easily enabled/disabled on the lightweight part.

Commonly an addon will have an agent component that runs on managed cluster, which should have at least one of the following characteristic:

1. It needs to talk to hub with certain ACLs
2. Its configuration diffs in different managed cluster.

We have `ClusterManagementAddon` and `ManagedClusterAddon` APIs defined already, so that an addon controller on the hub can be represented as a ClusterManagerAddon resource and an addon agent can be represented as a ManagedClusterAddon resource in  the cluster namespace on the hub.

## Motivation

With a standardize addon framework, it allows users to easily managed each function as a separate addon. In addition, it will be possible to provide an addon framework library so developers could be able to develop their own addon functions more easily.

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

As an open-cluster-management admin on hub, I can enable the managedcluster side addon installation on all managedclusters.

#### Story 3

As an admin on a managed cluster, I can enable and config the addon agent on a certain managed cluster. I was able to enable the addon agent on hub, which triggers the agent installation on the managed cluster.

#### Story 4

As an addon developer, I can easily develop the addon using the library without deep understanding on how agent manifests are distributed and registered to hub.


### Standardize addon procedure

#### Register an addon on hub

A common process for an addon to be installed on the hub cluster follows the procedure below:

1. Cluster admin on hub deploy a certain addon controller
2. Addon controller registers itself to the hub by creating a ClusterManagementAddon resource on hub
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
   crd: submariner.addon.open-cluster-management.io
   name: submariner-config
```

#### Enable addon agent on a managed cluster

1. A user creates a ManagedClusterAddon resource of which the name is the same as the name of ClusterManagerAddon resource in the cluster namespace.
It ensures that an addon can only be added to a managedcluster one time.

```yaml
 apiVersion: addon.open-cluster-management.io/v1alpha1
 kind: ManagedClusterAddon
 metadata:
   name: submariner
   namespace: cluster1
```
2. Addon controller on hub deploys the addon agent by creating manifestworks in cluster namespace.
3. The agent is started on a managed cluster, and registers back using credential.

#### Automatic enable addon agent to multiple clusters
The addon hub controller can automatically enable the addon agent to multiple managedclusters by creating ManagedClusterAddon resource in each of the cluster namespace. 
The addon hub controller can createManagedClusterAddon resource to every cluster namespace, or choose some cluster namespace via placement apis.

#### Communication pattern

An addon agent in a managed cluster will communicate with the hub:
- The addon agent talks to the kube apiserver of the hub. For reading desired state and for pushing small quantities of data that are stored in kubernetes API objects, this is the recommended path.
- The addon agent talks to another endpoint on the hub.  For writing large amounts of data to the hub or sending data that doesn’t fit cleanly into a kubernetes style API, this is the recommended approach.
- The two communication pattern can conexist for a single addon agent.

In both ways, the communication should be properly authenticated/authorized.

##### mTLS for authentication

The addon agent should be able to register to the hub by issuing a csr request and choose the signer. 
If the addon agent talks to the kube-apiserver of the hub, the agent should use the `kubernetes.io/kube-apiserver-client` signerName to create the csr.
Otherwise, the addon agent should use a customized signer for different endpoints. The endpoint, which belongs to 
the hub component of the addon should be able to accept the cert signed by the customized signer for this addon.

##### Permission for the addon agent

The addon should have a controller on hub to correctly set the permission for an addon agent to talk to the hub. 
1. An addon agent MUST only be able to access certain resources on the cluster namespace on the hub.
2. An addon agent MUST only be able to access certain resources on a specified namespace (not cluster namespace) on the hub.
3. An addon agent MUST be able to access a certain url on a non kube-apiserver endpoint.

The addon controller on the hub should have a reconcile loop that when an addon is enabled on a cluster, the permission of the addon agent can be set correctly. 

### Addon framework interfaces

To ease the development of addon controllers, it is possible that we provide a framework library to encapsulate the common part, 
so that developers of addon controllers only need to care about the manifests of addon agents that should be deployed on a managed cluster. 

#### Addon Manager

```go
// AddonManager is the interface to initialize a manager on hub to manage the addon
// agents on all managedcluster
type AddonManager interface {
	// AddAgent register an addon agent to the manager.
	AddAgent(addon agent.AgentAddon) error

	// AddAgentWithRegistration register an addon agent with registration to the manager.
	AddAgentWithRegistration(addon agent.AgentAddonWithRegistration) error

	// Start starts all registered addon agent.
	Start(ctx context.Context) error
}
```

#### Implement agent addon interface

```go
// AgentAddon defines manifests of agent deployed on managed cluster
type AgentAddon interface {
	// Manifests returns a list of manifest resources to be deployed on the managed cluster for this addon
	Manifests(cluster *clusterv1.ManagedCluster, config runtime.Object) ([]runtime.Object, error)

	// GetAgentAddonOptions returns the agent options.
	GetAgentAddonOptions() AgentAddonOptions
}

// AgentAddonWithRegistration manifests of agent deployed on managed cluster, rbac rules and bootstrap secret for agent to register to hub.
type AgentAddonWithRegistration interface {
	AgentAddon
	// HubRBAC returns the role and rolebinding on the hub for an agent on the managed cluster.
	HubRBAC(cluster *clusterv1.ManagedCluster, group string) (*rbacv1.Role, *rbacv1.RoleBinding)

	// BootstrapKubeConfig returns the bootstrap kubeconfig used by the agent to register to the hub.
	BootstrapKubeConfig(cluster *clusterv1.ManagedCluster) ([]byte, error)

	// RegistrationConfig returns how addon agent wants to register to the hub.
	RegistrationConfig(cluster *clusterv1.ManagedCluster) []RegistrationConfig

	// CSRApproveCheck checks whether the addon agent registration should be approved by the hub.
    // Commonly, we need to check whether the requester is valid and whether CN/ORG field in the request is valid.
	CSRApproveCheck(cluster *clusterv1.ManagedCluster, csr *certificatesv1.CertificateSigningRequest) bool
}

// AgentAddonOptions are the argumet for creating an addon agent.
type AgentAddonOptions struct {
	// AddonName is the name of the addon
	AddonName string

	// AddonNamespace is the namespace to deploy addon agent on spoke
	AddonInstallNamespace string

	// ConfigurationGVR is the gvr of the resource for configuring addon
	ConfigurationGVR *schema.GroupVersionResource
}

type RegistrationConfig struct {
	SignerName string
	Subject    *pkix.Name
}
```

#### Example

The example is to deploy a `helloworld` and a `helloworldwithregistration` addon to the managedcluster.

```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"os"
	"time"

	goflag "flag"

	"github.com/open-cluster-management/addon-framework/pkg/version"
	"github.com/openshift/library-go/pkg/controller/controllercmd"
	"github.com/spf13/cobra"
	"github.com/spf13/pflag"
	"k8s.io/apimachinery/pkg/runtime"
	utilflag "k8s.io/component-base/cli/flag"
	"k8s.io/component-base/logs"

	"github.com/open-cluster-management/addon-framework/pkg/addonmanager"
	"github.com/open-cluster-management/addon-framework/pkg/agent"
	clusterv1 "github.com/open-cluster-management/api/cluster/v1"
	certificatesv1 "k8s.io/api/certificates/v1"
	corev1 "k8s.io/api/core/v1"
	rbacv1 "k8s.io/api/rbac/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func main() {
	rand.Seed(time.Now().UTC().UnixNano())

	pflag.CommandLine.SetNormalizeFunc(utilflag.WordSepNormalizeFunc)
	pflag.CommandLine.AddGoFlagSet(goflag.CommandLine)

	logs.InitLogs()
	defer logs.FlushLogs()

	command := newCommand()
	if err := command.Execute(); err != nil {
		fmt.Fprintf(os.Stderr, "%v\n", err)
		os.Exit(1)
	}
}

func newCommand() *cobra.Command {
	cmd := controllercmd.
		NewControllerCommandConfig("addon-controller", version.Get(), runController).
		NewCommand()
	cmd.Use = "controller"
	cmd.Short = "Start the addon controller"

	return cmd
}

func runController(ctx context.Context, controllerContext *controllercmd.ControllerContext) error {
	mgr, err := addonmanager.New(controllerContext.KubeConfig)
	if err != nil {
		return err
	}
	agent := &helloWorldAgent{}
	mgr.AddAgent(agent)
	agentRegistration := &helloWorldAgentWithRegistration{}
	mgr.AddAgentWithRegistration(agentRegistration)
	mgr.Start(ctx)

	<-ctx.Done()

	return nil
}

type helloWorldAgent struct{}

func (h *helloWorldAgent) Manifests(cluster *clusterv1.ManagedCluster, config runtime.Object) ([]runtime.Object, error) {
	return []runtime.Object{
		&corev1.ConfigMap{
			TypeMeta: metav1.TypeMeta{
				Kind:       "ConfigMap",
				APIVersion: "v1",
			},
			ObjectMeta: metav1.ObjectMeta{
				Name:      "hello-world",
				Namespace: "default",
			},
		},
	}, nil
}

func (h *helloWorldAgent) GetAgentAddonOptions() agent.AgentAddonOptions {
	return agent.AgentAddonOptions{
		AddonName:             "helloworld",
		AddonInstallNamespace: "default",
	}
}

// Another agent with registration enabled.
type helloWorldAgentWithRegistration struct{}

func (h *helloWorldAgentWithRegistration) Manifests(cluster *clusterv1.ManagedCluster, config runtime.Object) ([]runtime.Object, error) {
	return []runtime.Object{
		&corev1.ConfigMap{
			TypeMeta: metav1.TypeMeta{
				Kind:       "ConfigMap",
				APIVersion: "v1",
			},
			ObjectMeta: metav1.ObjectMeta{
				Name:      "hello-world",
				Namespace: "default",
			},
		},
	}, nil
}

func (h *helloWorldAgentWithRegistration) GetAgentAddonOptions() agent.AgentAddonOptions {
	return agent.AgentAddonOptions{
		AddonName:             "helloworldwithregistration",
		AddonInstallNamespace: "default",
	}
}

// Specify what RBAC is needed on hub for the addon agent on managed cluster.
func (h *helloWorldAgentWithRegistration) HubRBAC(cluster *clusterv1.ManagedCluster, group string) (*rbacv1.Role, *rbacv1.RoleBinding) {
	return nil, nil
}

// The kubeconfig will be passed to the managedcluster to for registration bootstrap
func (h *helloWorldAgentWithRegistration) BootstrapKubeConfig(cluster *clusterv1.ManagedCluster) ([]byte, error) {
	return nil, nil
}

// This is to specify what signer is to used for agent registration
func (h *helloWorldAgentWithRegistration) RegistrationConfig(cluster *clusterv1.ManagedCluster) []agent.RegistrationConfig {
	return []agent.RegistrationConfig{
		{
			SignerName: "helloworldsigner",
		},
	}
}

// Specify whether the csr issued by the addon agent can be approved. The controller can check the validtiy of requester, and content of csr.
// It is recommended to return false if controller cannot guarantee that the csr is issue by a malicious agent.
func (h *helloWorldAgentWithRegistration) CSRApproveCheck(cluster *clusterv1.ManagedCluster, csr *certificatesv1.CertificateSigningRequest) bool {
	return true
}
```

### Test Plan

E2E tests can use kind to create multiple clusters to test. It can also use one cluster with self-join, that the cluster works as both hub and maanagedcluster.
An example should be created using addon framework for the purpose of testing.

- An addon agent can be successfully installed on Cluster A.
- An addon agent have correct permission to connect to hub after registration.

### Graduation Criteria

#### Alpha -> Beta Graduation

- E2E tests exists
- Beta -> GA Graduation criteria defined.
- At least one implementation using addon framework library.
- Fulfill the use case that controller auto generating `ManagedClusterAddon`. 

#### Beta -> GA Graduation

- Scalability/performance testing, under 1000 clusters.

### Upgrade / Downgrade Strategy

registration-operator must be updated to install `ManagedClusterAddon` and `ClusterManagerAddon` APIs.

### Version Skew Strategy

registration-operator must be upgraded before running any addons.