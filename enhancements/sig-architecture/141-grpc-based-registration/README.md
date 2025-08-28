# gRPC based registration

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal is an introduction to providing an optional GRPC based connection 
between klusterlet agent and hub cluster to replace the kubernetes based connection.

## Motivation

Today ocm agent connects to the hub using a kubeconfig with limited permission. To obtain
this kubeconfig, the agent sends a CSR resource using a bootstrap kubeconfig with limited
permission as a join request. The hub side component/actor needs to approve the request, and
creates related RBAC for the agent. The agent then is able to use the signed certificate from
the csr to construct the kubeconfig and connect to the hub.

This document is to design an alternative approach on hub/spoke connection setup to use GRPC
instead of directly connecting kube-apiserver.

There were security concerns about an agent using a kubeconfig to connect to the hub apiserver.
Even though the permission of this kubeconfig is carefully controlled, there are risk
the permission is misconfigured such that kubeconfig on the spoke is able to access other APIs
on the hub apiserver, and compromise the hub apiserver.

Using a GRPC connection between the hub and spoke can limit the exposure of the apiserver to
the spoke cluster, avoiding agents directly talking to hub apiserver. It also opens the
possibility in the future to:
- avoid setting cluster namespace for each cluster on the hub.
- use a non-kube controlplane to store data.

The similar mechanism
has been investigated in event-based-manifestwork that utilizes cloudEvent with message queue
or grpc as the connection mechanism between server and agent.



### Goals
- The user can configure the klusterlet to connect to the grpc endpoint of the hub with a bootstrap config.
  The bootstrap process is identical to the process of what we have today.
- The agent is able to connect to the hub cluster with a GRPC endpoint to sync cluster status and
  deliver manifestwork.
- The user or controller on the hub can still use the kubernetes style API to manage cluster and
  manifestwork as today.

### Non-Goals

## Proposal

### Scenarios

the kube-apiserver is used as the persistence layer. gRPC server translates the kube
resources (manifestwork/clusters/csr) to the grpc message and sends them to the klusterlet
agent. The agent also sends the status update to the gRPC server, and then the server
updates resource status in the kube-apiserver. The consumer/addons on the hub side
do not need to adjust any code. Its benefit is the agent no longer connects hub apiserver
directly, but it will not bring benefit in terms of scale.

```mermaid
graph TD;
    consumers<-->hub-apiserver;
    addons<-->hub-apiserver;
    hub-apiserver<-->grpc-server;
    grpc-server<-->klusterlet;
```

## Design Details

### gRPC server

gRPC server sits between the kube-apiserver and klusterlet agent. It receives request from
klusterlet agent in cloudevent format, translates to kube style API and call the kube-apiserver.
It also watches the kube-apiserver, translating the events from the informer to cloudevent and
sends to the related klusterlet agent. Requirements to the gRPC server are:

- Abstract the persistence layer: build an abstraction layer for persistence so that we can
  choose to use either k8s or db as the persistence layer.
- In addition to sync manifestwork, the grpc server also needs to support registration
  process and cluster status sync.
- How addon connects to the hub is out of scope of this design doc. The addon still uses the existing approach today.

### Abstract the persistence layer

We need several different interface abstraction to persistence layer:

Service interface: A common interface that sends resource spec to the agent and handles
status updates from the agent.

```go
type Service interface {
   // Get the cloudEvent based on resourceID from the service
   Get(ctx context.Context, resourceID string) (*cloudevents.Event, error)


   // List the cloudEvent from the service
   List(listOpts cetypes.ListOptions) ([]*cloudevents.Event, error)


   // HandleStatusUpdate processes the resource status update from the agent.
   HandleStatusUpdate(ctx context.Context, evt *cloudevents.Event) error


   // RegisterHandler register the handler to the service.
   RegisterHandler(handler EventHandler)
}
```

For each resources in kube-apiserver that klusterlet agent needs to interact with, e.g.
`ManagedCluster` and `ManagedClusterAddOn`, a service needs to be implemented.

### Registration process

The registration process should follow the same procedure as-is today. We will build a
grpc registration driver on the agent.

- A bootstrap config including grpc endpoint and token to connect to the grpc server
  on the hub is provided to the agent on the spoke. The token has limited permissions
  to only create certificate signing request and mamagedcluster with certain name.
- The agent uses this config to connect to the grpc server, sends the join request, and 
  subscribes to the response message. Only csr response will be returned from server in
  this stage.
- Upon receiving the join request, the server creates csr with
  open-cluster-mangement.io/grpc as the signer and managedcluster on the hub apiserver. 
- If another actor approves the csr on the hub apiserver, the server signs the csr,
  and sends the csr response back to the agent. Upon receiving the csr response, the
  agent builds a new grpc configuration to connect to the server. The agent monitors
  the certificate, and sends a new csr message when the certificate is about to expire.

### Agent authorization

The server should authorize the agent’s identity on what message the agent is permitted to send.
- The server gets the user identity from the context.
- The server gets event data type and action type in the event.
- The server builds a SAR request and checks if the message is allowed.

### Post registration

After registration is finished, the registration driver will generate a gRPC config using
client certificate, and then build clients for klusterlet agent to connect to gRPC server.
These clients is to sync cluster status, manifestwork, and managedclusteraddon resources.

### High Availability

gRPC server should be able to horizontal scalable. Each agent will be registered to one of the
gRPC server, and when one of the gRPC server is down, agent should reconnect to another one.
The gRPC server instance maintain a list of agents that is connecting to itself, and only send
messages to the agent in the list.

When the agent reconnects to another gRPC server, agent will send a `resync` request to the
server asking for messages that might be lost during the reconnection.

### Risks and Mitigation

The following security principles should be considered between the broker and sources/agents

- The sources and agents should be authenticated by broker to prevent arbitrary clients can consume the event messages from the broker
- The sources should be authorized by broker to avoid one source can consume event messages from other sources
- The agent should be authorized by broker to avoid one agent can consume event messages from other clusters 

### Metrics

The gRPC server exposes Prometheus metrics to monitor the health and performance. They are grouped into two categories:

1. **General gRPC server metrics**
2. **CloudEvents-specific gRPC server metrics**

#### General gRPC server metrics

Common metrics for gRPC server health and performance, started with `grpc_server` as Prometheus subsystem name. Each metric comes with a operator guide on healthy vs. degraded values.

- **`grpc_server_active_connections`**

  **Type**: Gauge \
  **Description**: Current number of active connections. \
  **Healthy**: A stable or predictable number of grpc connections, based on expected client load. \
  **Degraded**: A sudden drop to zero (all clients disconnected) or a sharp surge above the baseline may indicate connection leaks, restarts, or faulty clients. \
  **Metrics sample**:
  ```
  grpc_server_active_connections{local_addr="10.244.0.18:8090",remote_addr="10.244.0.16:45128"} 1
  ```

- **`grpc_server_started_total`**

  **Type**: Counter \
  **Description**: Total number of RPCs started on the server. \
  **Healthy**: The number of started RPCs closely matches the number of handled RPCs (`grpc_server_handled_total`). \
  **Degraded**: A growing gap between started and handled RPCs (`grpc_server_handled_total`) suggests requests are failing before completion. \
  **Metrics sample**:
  ```
  grpc_server_started_total{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary"} 3
  grpc_server_started_total{grpc_method="Subscribe",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="server_stream"} 4
  ```

- **`grpc_server_msg_received_total`**

  **Type**: Counter \
  **Description**: Total number of RPC messages received on the server. \
  **Healthy**: Steady growth aligned with expected traffic. \
  **Degraded**: Abnormal spikes may indicate flooding or misbehaving clients. If `grpc_server_msg_received_total/grpc_server_msg_sent_total` also rises, the server or downstream may not be processing requests as fast as they are received. \
  **Metrics sample**:
  ```
  grpc_server_msg_received_total{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary"} 3
  grpc_server_msg_received_total{grpc_method="Subscribe",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="server_stream"} 4
  ```

- **`grpc_server_msg_sent_total`**

  **Type**: Counter \
  **Description**: Total number of gRPC messages sent by the server. \
  **Healthy**: Consistent growth that matches the gRPC server and downstream processing rate. \
  **Degraded**: A sudden drop may mean the server and downstream aren’t processing requests, If `grpc_server_msg_received_total/grpc_server_msg_sent_total` also rises, the server or downstream may be falling behind. \
  **Metrics sample**:
  ```
  grpc_server_msg_sent_total{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary"} 3
  grpc_server_msg_sent_total{grpc_method="Subscribe",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="server_stream"} 1
  ```

- **`grpc_server_msg_received_bytes_total`**

  **Type**: Counter \
  **Description**: Total number of message bytes received on the gRPC server. \
  **Healthy**: Steady growth aligned with expected traffic. \
  **Degraded**: Abnormal spikes may indicate flooding or misbehaving clients. If `grpc_server_msg_received_bytes_total/grpc_server_msg_sent_bytes_total` also rises, the server or downstream may not be processing requests as fast as they are received. \
  **Metrics sample**:
  ```
  grpc_server_msg_received_bytes_total{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary"} 1729
  grpc_server_msg_received_bytes_total{grpc_method="Subscribe",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="server_stream"} 245
  ```

- **`grpc_server_msg_sent_bytes_total`**

  **Type**: Counter \
  **Description**: Total number of message bytes sent by the gRPC server. \
  **Healthy**: Consistent growth that matches the gRPC server and downstream processing rate. \
  **Degraded**: A sudden drop may mean the server and downstream aren’t processing requests, If `grpc_server_msg_received_total/grpc_server_msg_sent_total` also rises, the server or downstream may be falling behind. \
  **Metrics sample**:
  ```
  grpc_server_msg_sent_bytes_total{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary"} 0
  grpc_server_msg_sent_bytes_total{grpc_method="Subscribe",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="server_stream"} 1147
  ```

- **`grpc_server_handled_total`**

  **Type**: Counter \
  **Description**: Total number of RPCs completed on the server, regardless of success or failure. \
  **Healthy**: Most RPCs complete with `grpc_code="OK"`. \
  **Degraded**: An increasing number of non-OK codes (e.g., `Unavailable`, `DeadlineExceeded`, `Internal`) signals grpc server instability or downstream errors. \
  **Metrics sample**:
  ```
  grpc_server_handled_total{grpc_code="OK",grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary"} 3
  ```

- **`grpc_server_handling_seconds`**

  **Type**: Histogram \
  **Description**: Histogram of the duration of RPC handling by the gRPC server. \
  **Healthy**: Request latencies fall mostly into the lower buckets (e.g., <0.1s). \
  **Degraded**: Shifts into higher buckets (e.g., >1s or >5s) mean the server is slowing down, possibly due to load, resource starvation, or dependency issues. \
  **Metrics sample**:
  ```
  grpc_server_handling_seconds_bucket{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary",le="0.005"} 3
  grpc_server_handling_seconds_bucket{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary",le="0.01"} 3
  grpc_server_handling_seconds_bucket{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary",le="0.025"} 3
  grpc_server_handling_seconds_bucket{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary",le="0.05"} 3
  grpc_server_handling_seconds_bucket{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary",le="0.1"} 3
  grpc_server_handling_seconds_bucket{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary",le="0.25"} 3
  grpc_server_handling_seconds_bucket{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary",le="0.5"} 3
  grpc_server_handling_seconds_bucket{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary",le="1"} 3
  grpc_server_handling_seconds_bucket{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary",le="2.5"} 3
  grpc_server_handling_seconds_bucket{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary",le="5"} 3
  grpc_server_handling_seconds_bucket{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary",le="10"} 3
  grpc_server_handling_seconds_bucket{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary",le="+Inf"} 3
  grpc_server_handling_seconds_sum{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary"} 0.0055182140000000005
  grpc_server_handling_seconds_count{grpc_method="Publish",grpc_service="io.cloudevents.v1.CloudEventService",grpc_type="unary"} 3
  ```

#### CloudEvents-specific gRPC server metrics

Metrics specific to CloudEvents RPC calls, started with `grpc_server_ce` as Prometheus subsystem name. Each metric comes with a operator guide on healthy vs. degraded values.

- **`grpc_server_ce_called_total`**

  **Type**: Counter \
  **Description**: Total number of RPC requests called on the server. \
  **Healthy**: Regular increments matching agents traffic. \
  **Degraded**: Sudden stop (no calls received) or unexpected drops after a steady pattern may indicate communication issues with agents. \
  **Metrics sample**:
  ```
  grpc_server_ce_called_total{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",method="Publish"} 1
  grpc_server_ce_called_total{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",method="Subscribe"} 1
  ```

- **`grpc_server_ce_msg_received_total`**

  **Type**: Counter \
  **Description**: Total number of messages received on the gRPC server. \
  **Healthy**: Regular increments matching agents traffic, most received events eventually lead to sent events. \
  **Degraded**: Large gap of `grpc_server_ce_msg_received_total/grpc_server_ce_msg_sent_total` (many received but few sent/processed) may mean server bottlenecks, or dropped events. \
  **Metrics sample**:
  ```
  grpc_server_ce_msg_received_total{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",method="Publish"} 1
  grpc_server_ce_msg_received_total{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",method="Subscribe"} 1
  ```

- **`grpc_server_ce_msg_sent_total`**

  **Type**: Counter \
  **Description**: Total number of messages sent by the gRPC server. \
  **Healthy**: Consistent growth that matches the gRPC server and downstream processing rate. \
  **Degraded**: Large gap of `grpc_server_ce_msg_received_total/grpc_server_ce_msg_sent_total` (many received but few sent/processed) may mean server bottlenecks, or dropped events. \
  **Metrics sample**:
  ```
  grpc_server_ce_msg_sent_total{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",method="Publish"} 1
  ```

- **`grpc_server_ce_processed_total`**

  **Type**: Counter \
  **Description**: Total number of messages processed by the gRPC server. \
  **Healthy**: Most of cloudevents are processed with `grpc_code="OK"`. \
  **Degraded**: Rising counts of non-OK codes show the server is failing during processing. \
  **Metrics sample**:
  ```
  grpc_server_ce_processed_total{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",grpc_code="OK",method="Publish"} 1
  ```

- **`grpc_server_ce_processed_duration_seconds_bucket`**

  **Type**: Histogram \
  **Description**: Histogram of the duration of RPC requests for cloudevents processed on the server. \
  **Healthy**: Processing durations mostly in small buckets (e.g., <0.1s). \
  **Degraded**: Shifts into higher buckets (>1s or >5s) signals slowdown in event handling. \
  **Metrics sample**:
  ```
  grpc_server_ce_processed_duration_seconds_bucket{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",grpc_code="OK",method="Publish",le="0.005"} 0
  grpc_server_ce_processed_duration_seconds_bucket{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",grpc_code="OK",method="Publish",le="0.01"} 1
  grpc_server_ce_processed_duration_seconds_bucket{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",grpc_code="OK",method="Publish",le="0.025"} 1
  grpc_server_ce_processed_duration_seconds_bucket{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",grpc_code="OK",method="Publish",le="0.05"} 1
  grpc_server_ce_processed_duration_seconds_bucket{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",grpc_code="OK",method="Publish",le="0.1"} 1
  grpc_server_ce_processed_duration_seconds_bucket{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",grpc_code="OK",method="Publish",le="0.25"} 1
  grpc_server_ce_processed_duration_seconds_bucket{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",grpc_code="OK",method="Publish",le="0.5"} 1
  grpc_server_ce_processed_duration_seconds_bucket{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",grpc_code="OK",method="Publish",le="1"} 1
  grpc_server_ce_processed_duration_seconds_bucket{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",grpc_code="OK",method="Publish",le="2.5"} 1
  grpc_server_ce_processed_duration_seconds_bucket{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",grpc_code="OK",method="Publish",le="5"} 1
  grpc_server_ce_processed_duration_seconds_bucket{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",grpc_code="OK",method="Publish",le="10"} 1
  grpc_server_ce_processed_duration_seconds_bucket{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",grpc_code="OK",method="Publish",le="+Inf"} 1
  grpc_server_ce_processed_duration_seconds_sum{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",grpc_code="OK",method="Publish"} 0.001053519
  grpc_server_ce_processed_duration_seconds_count{cluster="cluster1",data_type="io.open-cluster-management.works.v1alpha1.manifestbundles",grpc_code="OK",method="Publish"} 1
  ```

### Test Plan

**Note:** *Section not required until targeted at a release.*

- Unit tests
- Integration tests

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:

- [Maturity levels][maturity-levels]
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),
or by redefining what graduation means.

In general, we try to use the same stages (alpha, beta, stable), regardless how the functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

### Upgrade / Downgrade Strategy

N/A

### Version Skew Strategy

N/A

## Implementation History

N/A

## Drawbacks

N/A

## Alternatives

N/A
