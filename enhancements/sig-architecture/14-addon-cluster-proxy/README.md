# Addon: Apiserver Network Proxy

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary

Automating the installation of [apiserver-network-proxy](https://github.com/kubernetes-sigs/apiserver-network-proxy)
so that admins or services can access the managed-clusters' kube-apiserver
in an imperative approach but without retaining any per-cluster credentials
in the hub cluster.

## Motivation

#### Mutual Cluster Connectivity

Originally in OCM a minimal requirement for the underlying network 
infrastructure is to support one-way network connectivity (spoke->hub) so 
that the basic cluster registration and management can work. The
apiserver-network-proxy will provide a reverse proxy tunnel over a long 
connection and multiplexes packets through the tunnel so that hub and spoke 
clusters are mutually accessible w/o raising more requirements to the 
infrastructure. Note that the reverse tunnel is only supposed to transmit 
"mouse flows" to avoid congestion.

#### Direct Multi-Cluster Resource Access

In some cases where we need to directly read the presence/state of specific 
resources in the managed clusters, however for now we have no idea of 
those resources unless they were created via the manifests api. Moreover,
we don't have the fine-grained information about their fields and values 
even if they are natively delivered by OCM. These problems will be eased
in a degree if we can establish a proxy tunnel between hub and spoke 
clusters.

#### Avoid Potential Credential Leakage

Normally we're supposed to obtain the credential (w/ sufficient permission)
from the spoke cluster before raising api calls against the spoke cluster, 
but that requires us to copy and dump the credential out of the spoke 
cluster which exposes the cluster under potential danger of being accessed 
by unexpected users. Also in some cases where the hub/spoke clusters are
mutually untrusted, copying the credentials will be strictly impossible.


### Goals

P1: 

- Automate the installation and operation and apiserver-network-proxy via 
  the addon-framework.
  - Structurelize the configuration of the proxy components from command-line
    flags to a custom resource configuration object.
- Automate the certificate rotation of both proxy-servers and proxy-agents.

### Non-Goals

- Tweak the original implementation of apiserver-network-proxy.

### User Story

#### 1. Calling spoke cluster's kubernetes api 

## Proposal

#### Architecture


The following figure shows the interaction between the components:

```
   
                     -----------------------------------------
                     | Hub-Cluster                           |
                     |                                       |
  ----------------   | ----------------     ---------------  |
  | Proxy Client |> > >| Proxy Server | <===| Proxy Addon |  |
  ----------------   | ----------------     |  Controller |  |
                     |      V               ---------------  |
                     |      V                    ‖           |
                     |    / V \                  ‖           |
                     ----/| V |\-----------------‖------------
                          | V |                  ‖
                   Reverse| V |Tunnel            ‖ Install/Operate
                          | V |                  ‖ 
 -------------------------| V |-------------     ‖
 | Managed-Cluster        | V |            |     ‖
 |                        | v |            |     ‖
 | -----------------   ---------------     |     ‖
 | | KubeApiserver |< <| Proxy Agent | <==========
 | -----------------   ---------------     |
 -------------------------------------------

```

- A proxy-agent that establishes a long connection towards the proxy-server
  and register itself w/ the cluster's identity.
- A proxy-server that receives handshakes from proxy-agent and maintains 
  tunnel over the long connection.
- A proxy-client requesting the target kube-apiserver through the tunnel
  over either websocket HTTP-CONNECT or grpc protocol.
- An addon controller automates the installation of proxy servers and proxy 
  agents by consuming the corresponding configuration custom resource. And
  optionally the proxy-agents can aggregate and report the status upwards
  so that admins can checkout their healthiness easily w/o even touching
  the managed-clusters.


#### Components

##### Cluster Proxy Operator

According to the latest design and implementation of [addon-framework](https://github.com/open-cluster-management-io/enhancements/tree/main/enhancements/sig-architecture/8-addon-framework),
we will bring a new agent controller named `cluster-proxy-operator` that
manages the lifecycle of the proxy components by watching the configuration
under a new group of api `proxy.addon.open-cluster-management.io/v1alpha1`:

- __ManagedProxyServer__: the configuration and status of proxy-servers.

- __MangedProxyAgent__: the configuration and status of proxy-agents.

Additionally, the operator will be list-watching the standard `ManagedClusterAddon`
resource and automatically installs proxy-agents w/ the expected options
upon discovering the addon in each cluster's namespace.


##### ManagedProxyServer

The `ManagedProxyServer` is supposed to be a cluster-scoped resource and holds 
information for provisioning an active proxy-server instance. An example will 
be:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedProxyServer
metadata:
  name: default # Or any unique name, will be the name of the workload
spec:
  replicas: 1
  certificates:
    signingSANs: 
      - "1.1.1.1"
      - "example.com"
    rotation:
      expiringDays: 365
      forceReloadDays: 200
  proxy: 
    mode: [http-connect|grpc|uds] 
  deploy:
    namespace: open-cluster-management
    agentAuthentication:
      namespace: <validating namespace>
      serviceAccount: <validating service-account>
    listeningPorts:
      proxyServer: 8090
      agentServer: 8091 
      healthServer: 8092
      adminServer: 8095 
status:
  lastObserverdGeneration: 2
  conditions:
  - type: "AllHealthy" 
    status: "True"
  - type: "AllUpToDate" 
    status: "True"
  components:
  - name: default-1
    namespace: open-cluster-management
    healthy: true
    restartCount: 0
  - name: default-2
    namespace: open-cluster-management
    healthy: true
    restartCount: 0
```

The operator will be actively syncing up the deployment of the proxy-servers
to the configuration in the following steps:

1. Signing all the required X509 certificates and keys then stores them into
   a secret object:
   
    i.   a CSR requesting server certificate which will be shared across the 
         replicas of the proxy servers.
   
    ii. a CSR requesting client certificate for logging onto in the tunnel 
        (skipped if the proxy server under "uds" mode).

2. Rolling/scaling out the proxy servers upon spec changes.
   
3. Actively probing/reporting the status of the proxy server instances.

##### ManagedProxyAgent

The `ManagedProxyAgent` resource is also a cluster-scoped resource and this
is the template configuration for installing agents to specific clusters:

An example is shown below:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedProxyAgent
metadata:
  name: default # Or any unique name, will be the name of the workload
spec:
  serverReference: 
    # A pointer to the server configuration so that auto-signed certificates
    # and other related config can be imported.
    apiVersion: "addon.open-cluster-management.io/v1alpha1"
    resource: managedproxyservers
    name: default
  replicas: 2
  proxyServerAddress: 1.1.1.1
  clusterServiceAccount: open-cluster-management:proxy-agent:egress
  certificates:
    bootstrapSecret: 
      # A reference to a secret resource that stores bootstrap kubeconfig
      namespace: default
      name: proxy-bootstrap-kubeconfig
```


##### ManagedClusterAddon

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddon
metadata:
 name: apiserver-network-proxy
spec:
 addonMeta:
   displayName: apiserver-network-proxy
   description: "https://github.com/kubernetes-sigs/apiserver-network-proxy"
 addonConfiguration:
   # Importing templates for creating real agent instances.
   crd: managedproxyagents.proxy.addon.open-cluster-management.io
   name: default
```

#### Expected Usage


##### Direct HTTP calls

Basically all the client library supporting HTTP proxy are expected to use 
the proxy addons w/o additional work. As a matter of fact, most of those 
popular HTTP client libraries only supports unsecured HTTP proxy instead of 
mTLS-secured HTTPS proxy, and neither golang SDK nor JDK has builtin support 
for mTLS proxy. So at the cost of securing the proxy server further, we will 
need to adapt HTTPS proxy by our own effort. The good news is that the
apiserver-network-proxy provided packages to talk over mTLS proxy. And also 
there are a few https client binaries that natively supports mTLS proxy e.g.
`curl` in the latest versions as is shown below:

```bash
curl -v -p \
     --proxy-key certs/frontend/private/proxy-client.key \
     --proxy-cert certs/frontend/issued/proxy-client.crt \
     --proxy-cacert certs/frontend/issued/ca.crt \
     --proxy-cert-type PEM \
     -x https://127.0.0.1:8090 \   # the proxy-server's address
     http://localhost:8000/success # any target L4 address
````


##### SSH over HTTPS tunnels

The original ssh command supports customizing tunnelling techniques by passing
`-o "ProxyCommand=..."` option, e.g. ssh over an plain HTTP proxy will be:

```shell
ssh user@host \
-o "ProxyCommand=nc --proxy-type http --proxy 127.0.0.1:8090 %h %p"
```

However, mutual TLS is not yet supported for configuring proxy options, so an
alternative will be replacing `nc` w/ a simple binary named `konnectivity-nc`
that supports both mTLS https or grpc proxy as an extension.

An very simple revision of `konnectiviy-nc` will be:

[https://gist.github.com/yue9944882/3266bb3a73fde5538e861ae174f6f177](https://gist.github.com/yue9944882/3266bb3a73fde5538e861ae174f6f177)

And the ssh command will be:

```shell
ssh user@host \
-o "ProxyCommand=konnectivity-nc \
--proxy-ca-cert <filepath to ca cert> \
--proxy-cert <filepath to x509 cert> \
--proxy-key <filepath to private key> \
%h %p"
```

#### Framework Integration

Based on the extensibility provided by the addon-framework, the new proxy addon
will be implementing the required interfaces in the following behavior:


```go
var _ AgentAddon = &ApiserverNetworkProxyAddon{}

func (a *ApiserverNetworkProxyAddon) Manifests(cluster *clusterv1.ManagedCluster, config runtime.Object) ([]runtime.Object, error) {
    // 1. a service-account with fundamental spoke cluster permissions.
    // 2. a deployment installing the proxy-agents
}

func (a *ApiserverNetworkProxyAddon) GetAgentAddonOptions() AgentAddonOptions {
	// using the `ProxyConfiguration` above as options
}

var _ AgentAddonWithRegistration = &ApiserverNetworkProxyAddon{}

func (a *ApiserverNetworkProxyAddon) HubRBAC(cluster *clusterv1.ManagedCluster, group string) (*rbacv1.Role, *rbacv1.RoleBinding) {
	// none
}

func (a *ApiserverNetworkProxyAddon) BootstrapKubeConfig(cluster *clusterv1.ManagedCluster) ([]byte, error) {
	// a bootstrap kubeconfig for raising initial CSR requests
}


func (a *ApiserverNetworkProxyAddon) RegistrationConfig(cluster *clusterv1.ManagedCluster) []RegistrationConfig {
	// groups:  ["open-cluster-management:proxy-agents"]
}

func (a *ApiserverNetworkProxyAddon) CSRApproveCheck(cluster *clusterv1.ManagedCluster, csr *certificatesv1.CertificateSigningRequest) bool {
	// matching all those CSR with the expected bootstrap kubeconfig
}
```

## Future Work

#### An Optional Aggregated-Apiserver Gateway

In order to simplify the direct multi-cluster api calls further, we plan to
make the addon controller to opt-in to install an aggregated-apiserver which
provides a `/proxy` subresource that allows users to access the managedclusters
via plain HTTPS calls w/o proxy. An example will be:

```bash
# probing managed cluster's /healthz
kubectl get --raw="/apis/<the addon api-group>/<the addon api-version>/<addon-resource-name>/<managed cluster name>/proxy/healthz"
# listing managed cluster's namespaces
kubectl get --raw="/apis/<the addon api-group>/<the addon api-version>/<addon-resource-name>/<managed cluster name>/proxy/api/v1/namespaces"
```

And the official golang and java SDKs are supposed to talk to the managed cluster
by simply adding an api prefix to the requests.

#### Proxy Server Authorization 

Even if the proxy-servers are mTLS-secured, all the client talking to the servers
will be actually sharing the same identity for each cluster. We can consider passing
the real client identity via impersonation headers or something else that keep that
information.