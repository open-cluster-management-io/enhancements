# Moving cluster-proxy addon into klusterlet

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal outlines an architectural change to move cluster-proxy capabilities from an addon into the core
klusterlet agent, enhancing Open Cluster Management's core functionality.

## Motivation

The `cluster-proxy addon` has been widely used in various scenarios that complement the core functionality of
Open Cluster Management. Use cases include:
- Users fetch pod logs via `cluster-proxy`
- Users access VM consoles when `kubevirt` is deployed in a `ManagedCluster`
- `MultiKueue` integrates with `cluster-proxy` to submit jobs to `ManagedClusters`
- `MulticlusterGateway` in `KubeVela` uses `cluster-proxy` to push requests to targeted clusters

Currently, `cluster-proxy` is implemented based on the ANP (APIServer Network Proxy) server and deployed as an addon.
However, we have identified several limitations when using `cluster-proxy`:

1. `cluster-proxy` provides a `kconnectivity` interface which is gRPC-based. It is difficult to call directly
   with Kubernetes clients, requiring an HTTP proxy frontend to handle requests.
2. `cluster-proxy` leverages the ANP server which transports TCP packets over gRPC tunnels. The agent directly
   proxies TCP packets to the target location, making it difficult for the agent to act as a proxy or delegator
   to mutate and proxy HTTP requests.
3. End users must always enable the cluster-proxy addon, introducing additional operational overhead when
   cluster-proxy functionality is commonly needed.

We are introducing gRPC as a registration mechanism in OCM, which makes it easier to move cluster-proxy
capabilities into the klusterlet.

### Goals

- Add a feature flag `ClusterProxy` in both klusterlet and clustermanager. When enabled, a gRPC server will
  start on the hub, and the klusterlet agent will connect to the gRPC server for proxy functionality.
- When the feature is enabled, ClusterManager starts an HTTP server to proxy requests to the target cluster
  via the klusterlet agent.
- All existing capabilities of `cluster-proxy` will be preserved.

### Non-Goals

- How to deprecate the `cluster-proxy addon` is not in the scope of this proposal, given that `cluster-proxy`
  is still consumed by `MulticlusterGateway` and other consumers.

## Proposal

### User Stories

#### Story 1

Users can enable the `ClusterProxy` feature gate in both clustermanager and klusterlet. Once enabled, users can
use the HTTP server on the hub cluster to proxy requests to the API server of the targeted cluster.

#### Story 2

Users can authenticate via proxy using either the token for the targeted cluster, or the token for the hub
cluster if impersonation permissions are granted to the klusterlet agent.


### Risks and Mitigation

N/A

## Design Details

#### Registration
The connection between klusterlet and hub for proxy functionality will remain gRPC-based. This is independent
of the gRPC registration driver, requiring a separate gRPC configuration file for the proxy. To obtain the gRPC
configuration for proxy, the klusterlet agent will send a Certificate Signing Request (CSR) to the hub with the
signer name `open-cluster-management.io/klusterlet-proxy`. The gRPC server on the hub will approve and sign the
CSR. The gRPC configuration will be saved in the `hub-kubeconfig-secret` with the key `proxy-grpc.yaml`. The
klusterlet will only start the proxy when it detects the existence of this configuration file. The proxy
registration process begins after the cluster registration process and will not impact normal cluster
registration.

#### Proxy API

The hub cluster will start an externally accessible HTTP endpoint to proxy requests to the API server of each managed
cluster. The API path follows the format: `https://<server address>:<server port>/<cluster name>`.

Consumers can configure their kubeconfig using this API endpoint and the related CA bundle to access the API server
of the targeted cluster.

`clusteradm` will be enhanced with a subcommand `clusteradm proxy --cluster <cluster name>`, which will configure
the kubeconfig with the proxy endpoint in a cluster-specific context and switch the kubeconfig context to the target
cluster. Users can then use kubectl to connect to the target cluster.

#### Integration with ClusterProfile API

#### Installation

To enable the feature, the user needs to enable the feature gate in `ClusterManager` and `Klusterlet` API.
In addition, A new proxyConfig field should be added in `ClusterManager` API:

```go
type ProxyConfig struct {
    // EndpointExposure represents the configuration for endpoint exposure.
    // +optional
    EndpointExposure *GRPCEndpointExposure `json:"endpointExposure,omitempty"`
}
```

The proxyConfig field will also be added onto `Klusterlet` API:

```go
type ProxyConfig struct {
    Endpoint string `json:"endpoint,omitempty"`
    CABundle []byte `json:"caBundle,omitempty"`
    
    // Authentications defines how the agent authenticates with the cluster.
    // By default it is userToken, but it could also be impersonation or both.
    Authentications []string `json:"authentications,omitempty"`
}
```

### Test Plan

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- When the feature is enabled, the proxy is installed correctly
- Users can access the target cluster via proxy
- Users can authenticate using either token or impersonation methods

### Graduation Criteria

**Alpha:**
- Proxy can be installed and function correctly
- Proxy works with each registration driver

**Beta:**
- At least two consumers are using or migrating from cluster-proxy addon to this feature
- clusteradm is updated to adopt this feature
- End-to-end tests ensure all use cases of existing cluster-proxy addon are covered

**GA (Graduate):**
- All consumers have migrated from cluster-proxy addon to this feature
- Metrics are defined and exposed for the proxy server
- Scalability testing is completed

### Upgrade / Downgrade Strategy

TBD

### Version Skew Strategy

There should be no version compatibility issues.

## Implementation History

N/A

## Drawbacks

The proxy agent may consume excessive bandwidth, potentially impacting the existing registration process.

## Alternatives

Enhance the cluster-proxy addon instead of moving it to core. Since the proxy is becoming a fundamental function
in OCM, maintaining it as a core function will ensure better stability and quality.
