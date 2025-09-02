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

Currently, `cluster-proxy` is implemented based on the [ANP (APIServer Network Proxy)](https://github.com/kubernetes-sigs/apiserver-network-proxy)
server and deployed as an addon. However, we have identified several limitations when using `cluster-proxy`:

1. `cluster-proxy` provides a `kconnectivity` interface which is gRPC-based. It is difficult to call directly
   with Kubernetes clients, requiring an HTTP proxy frontend to handle requests.
2. `cluster-proxy` leverages the ANP server which transports TCP packets over gRPC tunnels. The agent directly
   proxies TCP packets to the target location, making it difficult for the agent to act as a proxy or delegator
   to mutate and proxy HTTP requests.
3. End users must always enable the cluster-proxy addon, introducing additional operational overhead when
   cluster-proxy functionality is commonly needed.
4. ANP runs as a binary runtime, offering low flexibility, and users face a "black box" problem.

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
- Enhance `clusteradm` to configure kubeconfig file with proxy's endpoint.

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

#### Integration with ClusterProfile API

When this feature enabled, we should be able to provide a tool to build kubeconfig and connect to the targeted cluster
based on clusterInventory API using https://github.com/kubernetes-sigs/cluster-inventory-api/blob/main/pkg/credentials/config.go:

- Build a binary or script that could provide credential for the cluster via ManagedServiceAccount or impersonation
- Update clusterProfile status in the registration controller using proxy endpoint.

#### Installation

To enable the feature, the user needs to enable the feature gate in `ClusterManager` and `Klusterlet` API.
In addition, A new `GRPCConfiguration` field should be added in `ClusterManager` API:

```go
type GRPCConfiguration struct {
	// ImagePullSpec is the image for grpc server
    ImagePullSpec string `json:"imagePullSpec,omitempty"`
	
	// featureGates represents the features enabled for grpc server
	FeatureGates []FeatureGate `json:"featureGates,omitempty"`

	// endpointsExposure represents the configuration for grpc endpoint exposure.
	// +optional
	EndpointsExposure []EndpointExposure `json:"endpointsExposure"`
}

type EndpointExposure struct {
	// type is the type of the endpoint, could be agentToServer or user.
	Type string `json:"type"`
	
	// protocol is the protocol used for the endpoint, could be http or grpc.
	Protocol string `json:"protocol"`
	
	GRPC *EndpointExposure `json:"grpcEndpointExposure"`
	
	HTTP *EndpointExposure `json:"httpEndpointExposure"`
}

type EndpointExposure struct {
    Endpoint string `json:"endpoint,omitempty"`
    CABundle []byte `json:"caBundle,omitempty"`
}
```

The following are examples of `ClusterManager` on how to install grpc proxy.
Example of using proxy with csr registration and proxy would look like:

```yaml
spec:
  grpcConfiguration:
    imagePullSpec: <grpc image>
    featureGates:
    - feature: ClusterProxy
      mode: Enabled
    endpointsExposure:
    - type: user
      potocol: HTTP
      httpEndpointExposure:
        endpoint: https://<external http server>
    - type: agentToServer
      protocol: GRPC
      grpcEndpointExposure:
        endpoint: grpc://<external grpc address>
```

Example of grpc registration with proxy enabled:

```yaml
spec:
  registrationConfiguration:
    registrationDrivers:
    - authType: csr
    - authType: grpc
  grpcConfiguration:
    imagePullSpec: <grpc image>
    featureGates:
      - feature: ClusterProxy
        mode: Enabled
    endpointsExposure:
      - type: user
        potocol: HTTP
        httpEndpointExposure:
          endpoint: https://<external http server>
      - type: agentToServer
        protocol: GRPC
        grpcEndpointExposure:
          endpoint: grpc://<external grpc address>
```

Example of grpc registraion with proxy disabled:

```yaml
spec:
  registrationConfiguration:
    registrationDrivers:
    - authType: csr
    - authType: grpc
  grpcConfiguration:
    imagePullSpec: <grpc image>
    endpointsExposure:
    - type: agentToServer
      protocol: GRPC
      grpcEndpointExposure:
        endpoint: grpc://<external grpc address>
```

The proxyConfig field will also be added onto `Klusterlet` API:

```go
type ProxyConfig struct {
     GRPC *EndpointExposure `json:"grpcEndpoint"`
    
    // Authentications defines how the agent authenticates with the cluster.
    // By default it is userToken, but it could also be impersonation or both.
    Authentications []string `json:"authentications,omitempty"`
}
```

An example of enabling proxy on klusterlet will be:

```yaml
spec:
  proxyConfig:
    grpcEndpoint:
      endpoint: <grpc://server address>
      caBundle: <base64 encoded ca>
    authentications:
      - userToken
      - impersonation
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
- Integrate with cluster-inventory API.
- End-to-end tests ensure all use cases of existing cluster-proxy addon are covered

**GA (Graduate):**
- All consumers have migrated from cluster-proxy addon to this feature.
- Metrics are defined and exposed for the proxy server.
- Scalability testing and performance analysis on throughput, resource consumption, and concurrent requests are completed.

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
