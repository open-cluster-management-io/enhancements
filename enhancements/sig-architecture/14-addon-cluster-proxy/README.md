# Addon: Apiserver Network Proxy

## Release Signoff Checklist

- [ ] Enhancement is `implemented`
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

Originally in OCM, a minimal requirement for the underlying network 
infrastructure is to support one-way network connectivity (spoke->hub) so 
that the basic cluster registration and management can work. The SIG 
subproject apiserver-network-proxy will provide a reverse proxy tunnel over 
a long connection and multiplexes packets through the tunnel so that hub 
and spoke clusters are mutually accessible w/o raising more requirements to 
the infrastructure. Note that the reverse tunnel is only supposed to transmit 
"mouse flows" to avoid congestion.

#### Direct Multi-Cluster Resource Access

In some cases where we need to directly read the presence/state of specific 
resources in the managed clusters, however for now we have no idea of 
those resources unless they were created via the manifests api. Moreover,
we don't have the fine-grained information about their fields and status 
even if they are natively delivered by OCM. These problems will be eased
in a degree if we can establish a proxy tunnel between hub and spoke 
clusters.


### Goals

Alpha: 

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
- An addon controller automates the installation of proxy servers and 
  proxy agents by consuming the corresponding configuration custom resource.
  Optionally the proxy-agents can aggregate and report the status upwards
  so that admins can checkout their healthiness easily w/o even touching
  the managed-clusters.
- An addon agent that oversees the proxy-agents and manages the rotation
  of either certificates or tokens.


#### Components

##### Cluster Proxy Operator

According to the latest design and implementation of [addon-framework](https://github.com/open-cluster-management-io/enhancements/tree/main/enhancements/sig-architecture/8-addon-framework),
we will bring a new add manager named `cluster-proxy-operator` that
manages the lifecycle of the proxy components by watching the configuration
under a new group of api `proxy.open-cluster-management.io/v1alpha1` along
with a few builtins resources provided by OCM's addon-framework:

- __ManagedProxyConfiguration__: the configuration template for installing 
    proxy-servers and proxy-agents.

- __ClusterManagementAddon__: if present, triggering the operator to install
    the proxy-servers by reading the configuration template in the hub cluster.
  
- __ManagedClusterAddon__: if present in any cluster's namespace, prescribe
    the operator to install proxy-agents according to the template.

##### ManagedProxyServer

The `ManagedProxyServer` is supposed to be a cluster-scoped resource and is a 
profile storing configuration template for prescribing the installation of 
proxy-servers and proxy-agents.

```yaml
apiVersion: proxy.open-cluster-management.io/v1alpha1
kind: ManagedProxyConfiguration
metadata:
  name: default # Or any unique name, will be the name of the workload
spec:
  certificates:
    signing:
      additionalSANs:
        - "1.1.1.1"
        - "example.com"
      rotation:
        expiringDays: 365
        forceReloadDays: 200
    bootstrap:
      secret:
        namespace: open-cluster-management
        name: proxy-server-bootstrap
  deploy:
    serviceAccount:
      namespace: open-cluster-management
      name: proxy-agents
    ports:
      proxyServer: 8090
      agentServer: 8091
      healthServer: 8092
      adminServer: 8095
  proxyServer:
    image: <...>
    replicas: 2
    inClusterService:
      namespace: open-cluster-management
      name: proxy-server-default
  proxyAgent:
    image: <...>
    replicas: 2
status:
  lastObserverdGeneration: 2
  certificates:
    signedCertificate: <...> # PEM-encoded
    privateKey: <...>  # PKCS#8 RSA generated key
    bootstrapSecret: <...>
  conditions:
  - type: "AllHealthy" 
    status: "True"
  - type: "AllUpToDate" 
    status: "True"
  components:
    proxyServer:
     - name: default-1
       namespace: open-cluster-management
       healthy: true
       restartCount: 0
     - name: default-2
       namespace: open-cluster-management
       healthy: true
       restartCount: 0
    proxyAgent:
      - cluster: <deployed cluster name> 
        lastUpdateTimestamp: <...>
```


##### ClusterManagementAddon

`ClusterManagementAddon` is the original addon custom resource provided by the
addon framework. Upon creating the addon resource and connecting it w/ an
existing `ManagedProxyConfiguration` resource, the operator will be actively 
syncing up the deployment of the proxy-servers to the configuration in the 
following steps:

1. Signing all the required X509 certificates and keys then stores them into
   a secret object:

   i.   a CSR requesting server certificate which will be shared across the
   replicas of the proxy servers.

   ii.  a CSR requesting client certificate for logging onto in the tunnel
   (skipped if the proxy server under "uds" mode).
   
   iii. a bootstrap secret that allows the future proxy-agents to issue
   CSR requests.

2. Rolling/scaling out the proxy servers upon spec changes.

3. Actively probing/reporting the status of the proxy server instances.

And an example of `ClusterManagementAddon` is shown below:

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
   crd: managedproxyconfigurations.proxy.addon.open-cluster-management.io
   name: default
```

##### ManagedClusterAddon

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddon
metadata:
  name: apiserver-network-proxy
  namespace: <deploying cluster name>
spec:
  relatedObjects:
    - apiVersion: addon.open-cluster-management.io/v1alpha1
      resource: clustermanagementaddons
      name: apiserver-network-proxy
  addonMeta:
    displayName: apiserver-network-proxy
    description: "https://github.com/kubernetes-sigs/apiserver-network-proxy"
  addonConfiguration:
    # Importing templates for creating real agent instances.
    crd: managedproxyconfigurations.proxy.addon.open-cluster-management.io
    name: default
```
  
#### Proxy Security

The following figure shows the three sets of mTLS certificates/keys
involved in the proxy framework:

1. The mTLS key-pair for establishing the tunnel.
2. The mTLS key-pair for initiating the proxy service.
3. The mTLS key-pair for accessing the endpoint k8s apiserver, 
   alternatively it can be a service-account token.

```

                        hub   |  public   |  spoke
                      network | internet  | network
client ======> server ========|===========|=========>  agent 
-------------------------- Proxy Tunnel ----------------------
L4                 <----------- mTLS(1) ---------------       T0: Setting up the tunnel
L5                 <------ grpc proxy protocol --------       T0: Setting up the tunnel
------------------- Proxied Multiplexed Traffic --------------
client -mTLS(2)->                                             T1: Connects proxy server   
       --------->                                             T2: Requests a proxied stream
       --------------------- mTLS(3) ----------------->       T3: Unwinds kube-apiserver mTLS
       ----------------- k8s api requests ------------>       T4: Sends request payload 
```

##### Server <-> Proxy Authentication

The tunnel from agent to server is secured by mTLS technique, the same way
we're doing for securing kubernetes' apiserver. So it's almost safe to expose
the proxy-servers' entry point even in the public networks unless it is 
flooded by DDoS attacks.


##### Server <-> Agent Authentication

Based on the authentication above, in the original design of ANP, the proxy
server can opt-in to verify the agents' identity by asking the agents to pass a 
service-account-token in the handshake attempts so the proxy-servers can 
validate the token by the token's signed namespace, name, or api-audience.
Also in our proxy addon, the authorization will be mandatory in order to
make the proxy-server to accept only those trusted agents.

#### Examples:

##### Installs the proxy framework

Manual Preparation:

-   Installing the proxy framework:
    -   Create a `ManagedProxyConfiguration` to configure the servers and agents.
    -   Create a `ClusterManagementAddon` to trigger proxy-server installation.
    -   Create a `ManagedClusterAddon` to trigger proxy-agent installation for
        each managed cluster.
        
##### Example: A multi-cluster controller read/writes multiple clusters

Here's an example what we expected to do for writing a multi-cluster controller
that dynamically raises requests against multiple managed clusters:

Manual Preparation:

-   Setup the controller:        
    -   Create a `CredentialProjection` to requests token for accessing the managed
        cluster, and the credentials will automatically provisioned under each
        managed cluster's namespace w/ prescribed permission. The controller should
        know the name of the `CredentialProjection` to use.
    -   Modify the [net.DialFunc](https://github.com/kubernetes/kubernetes/blob/edb0a72cff0e43bab72a02cada8486d562ee1cd5/staging/src/k8s.io/apimachinery/pkg/util/net/http.go#L213)
        under [rest.Config#Dial](https://github.com/kubernetes/kubernetes/blob/edb0a72cff0e43bab72a02cada8486d562ee1cd5/staging/src/k8s.io/client-go/rest/config.go#L135)
        to make the controller multiplexes its actual k8s api requests into the 
        tunnel.
        
The Process of an Actual API Calls:

1.  Read the key-pairs from the status of the belonging `ManagedProxyConfiguration`.
2.  Get the credential (i.e. mTLS(3) in the figure above) for the target cluster
    to talk to.
3.  Establish a proxy stream against the proxy-servers using mTLS(2) from step 1. 
4.  Send real k8s api requests using mTLS(3) from step 2.


##### Example: A kubectl plugin that runs local http proxy for the managed cluster


Here's an example of a new kubectl plugin named `kubectl cluster-proxy` that runs
a http proxy server at localhost.

Manual Preparation:

-   Prepare a credential for actually accessing the managed cluster:
    -   Create a `CredentialProjection` w/ e.g. list pod permission.
    
The Process of an Actual API Calls:

1.  Run `kubectl cluster-proxy <cluster-name> --credential=<..> get pod`  
2.  Read the key-pairs from the status of the belonging `ManagedProxyConfiguration`.
3.  Read the secrets according to the name from `--credential`
4.  Establishes a proxy stream using mTLS cert/key from step 1.
5.  Send real k8s api requests using mTLS cert/key from step 2.
        
    

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


## Graduation Criteria

Alpha:

- Proxy servers and agents can be automatically installed/upgrade by the 
  new addon-manager.
- Addon can be uninstalled flexibly w/o leaving any garbage.
- Automatic certificate rotation.
- A brief status of the proxy servers and agents can be aggregated and 
  reflected on the
  
Beta:

- Fine-grained status can be aggregated on the addon resource so that admin 
  can observe the traffic going through the tunnel and the healthiness of 
  servers and agents.
- Proxy servers can scale horizontally by editing the number of replicas in 
  the addon.

#### Test Plan

Alpha:

- Unit tests that makes sure the project's fundamental quality.
- Integration tests against real KinD clusters to execercise the installation.


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

And the official golang and java SDKs are supposed to talk to the managed 
cluster by simply adding an api prefix to the requests. Note that all 
requests from the aggregated-apiserver will be using the cert/key prescribed 
by `CredentialProjection`.
