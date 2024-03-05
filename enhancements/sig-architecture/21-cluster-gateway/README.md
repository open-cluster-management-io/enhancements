# Multi-Cluster API Gateway

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

After solving the issues of (1) hub -> spoke network connectivity [#19](https://github.com/open-cluster-management-io/enhancements/pull/19)
and (2) hub -> spoke authentication via token projection [#24](https://github.com/open-cluster-management-io/enhancements/pull/24)
the last missing puzzle for dynamic multi-cluster API invocation will be an
API gateway for helping switching contexts between clusters. This API gateway 
will be proxying the requests to any managed cluster similar to the mechanism 
of existing `serivce/proxy` or `pod/proxy`, and the gateway is supposed to be
plugging into the kubernetes cluster as an aggregated apiserver so that the
hub cluster can discover a new proxy subresource named `clustergateways/proxy`.

## Motivation

### Dynamically Switching between Managed Clusters

An old-fashion approach of reading resources from multiple managed clusters 
will be looping over the clusters and instantiate new client instances then
send the actual requests. While w/ the help of gateway, switching client context 
between the managed clusters will be as simple as replacing the target resource
name of the `clustergateway` resource, e.g. for accessing a cluster named `foo`,
all the client required to do is simply requesting the `proxy` subresource for
the new resource `clustergateway` named `foo`. Additionally, accessing arbitrary 
non-resource paths will also be supported, e.g. for probing the overall health
of the managed cluster, we wil be call an API path of `/apis/open-cluster-management.io/clustergateways/foo/proxy/healthz`
against the hub kube-apiserver. Note that the API gateway is supposed to use the 
projected service-account token [#24](https://github.com/open-cluster-management-io/enhancements/pull/24) 
to identify itself when proxying requests to the managed cluster's kube-apiserver.

### Keep KubeConfig/Tokens Invisible From Clients

In most cases, we hope to grant the accessibility of managed clusters' 
kube-apiserver to hub components w/o exposing the plain tokens. For the gateway,
it can be easily done by restricting the proxy clients RBAC rights to the 
`clustergateways/proxy` subresource, also the verbs can be tightened so that we
can for example ban some proxy clients from deleting or listing resources.
To secure the tokens from the both the gateway users and the other tenants
sharing the hub cluster, all the projected tokens are supposed to be maintained
under a single high-secured namespace named `managed-cluster-credentials`. 

### Network Connectivity

If the managed cluster is running in an isolated network, we will have to rely on
the work proposed by KEP [#19](https://github.com/open-cluster-management-io/enhancements/pull/19)
which establishes reverse proxy tunnels between the hub and spoke clusters so the
those requests proxied by the gateway can finally reach the managed cluster's
apiserver.

## Goals

- Having a new resource `clustergateway` and its subresource `proxy` based on 
  apiserver aggregation.
- Adding a new optional component named `cluster-gateway` for delegating requests
  to the spoke clusters.

## Proposal

An initial POC implementation now goes publicly available under the repo [cluster-gateway](https://github.com/oam-dev/cluster-gateway).


### API 

By installing the gateway to the hub cluster via applying an `APIService`, the new 
resource should be visible by the `kubectl api-resources` command. A sample will
be:

```yaml
apiVersion: "proxy.open-cluster-management.io/v1alpha1"
kind: "ClusterGateway"
metadata:
  name: <..>
spec:
  provider: ""
  access:
    endpoint: "https://127.0.0.1:9443"
    caBundle: "..."
    credential:
      type: [X509Certificate|ServiceAccountToken|ManagedServiceAccount]
      x509Certificate:
        certificate: "..."
        privateKey: "..."
      serviceAccountToken:
        token: "..." 
      managedServiceAccount:
        namespace: ".."
        name: "..."
status: { }      
```

Note that the credential being used by the gateway can be from 3 diffrent
sources:

- X509Certificate: PEM encoded certificates and keys.
- ServiceAccountToken: The JWT token signed and copied from the managed cluster.
- ManagedServiceAccount: Referencing an existing managed service-account, and read
  the corresponding projected tokens to authenticate the proxying requests.
  
### Client Usage


There can be two major kinds of clients using the api gateway:

- Kubectl Users: In some cases, the hub administrators need a one-time read/write
  to a managed cluster e.g. when trouble-shooting or operating certain cluster. The
  only thing we will need to do to make the kubectl works with the gatewa is just
  simply adding a suffix `/apis/open-cluster-management.io/clustergateways/<cluster name>/proxy/`
  to the original hub kubeconfig, this will magically turn the hub kubeconfig to a
  kubeconfig dedicated for any managed cluster.
  
- Hub System: Components in hub cluster can also do the similar things as appending
  a suffix to the request manually. Also we will be providing a set of client library
  by "client expansion" to make the integration easier:
  
A sample of the client library's interface will be:

```go
type ClusterGatewayExpansion interface {
	RESTClient(clusterName string) rest.Interface

	GetKubernetesClient(clusterName string) kubernetes.Interface
	GetControllerRuntimeClient(clusterName string, options client.Options) (client.Client, error)

	RoundTripperForCluster(clusterName string) http.RoundTripper
	RoundTripperForClusterFromContext() http.RoundTripper
	RoundTripperForClusterFromContextWrapper(http.RoundTripper) http.RoundTripper
}
// See also: https://github.com/oam-dev/cluster-gateway/blob/master/pkg/generated/clientset/versioned/typed/cluster/v1alpha1/clustergateway_expansion.go#L34-L35
```
  
For example to read a namespace from cluster "foo" via the library will be easy as:

```go
kubernetes.New(gatewayClient.
	ClusterV1alpha1().
	ClusterGateways().
	RESTClient("foo")).
	CoreV1().
	Namespaces().
	Get(....)
```


### Authorizing the Proxying Requests


Due to the fact that all the outbound requests from the gateway will be sharing a 
common identity when reaching a managed cluster, we are not able to control the 
granularity of authorization to discriminate different end users working behind
the proxy. However, practically it can be difficult to the control the RBAC scope 
of the outbound requests from the gateway dynamically b/c for now in OCM there's 
no such framework as "multi-cluster authorization". But one feasible solution is to 
have an additional authorization check before actually proxying the requests via the 
`rbacv1.SubjectAccessReview` (SAR) api. The subject of the SAR should keep the 
original client identity, but the verb can be virtual such as `<cluster-name>/<verb>`.
For example, when a client "user1" is attempting to read a namespace "default" from
cluster "foo", then the following SAR will be asserted before proxying:

```yaml
apiVersion: authorization.k8s.io/v1
kind: SubjectAccessReview
spec:
  resourceAttributes:
    group: ""
    resource: "namespaces"
    verb: "foo/get"
    name: "default"
```

Alternatively in the future if there's a native central authorization engine in OCM,
we can simply switch from manual SAR calls to the engine.


### Trimming Off Persisting Layer

Usually an aggregated apiserver requires ETCD backends, but this can be trimmed off if 
we can restrict the `ManagedServiceAccount` to be the only source of the authentication 
credentials. By customizing the REST storage implementation of the aggregated 
`ClusterGateway` resource, we can transform the secrets projected from managed cluster
to a virtual `ClusterGateway` instance. The virtual instance should be completely read-only
to avoid conflicts with the underlying secret resource.
