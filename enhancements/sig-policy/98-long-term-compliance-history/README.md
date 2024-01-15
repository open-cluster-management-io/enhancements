# Long-term Compliance History of Policies

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
      [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

To address the gaps in policy compliance history, it is proposed to store compliance events in a central relational
database so that they can be stored long-term and are queryable with rich metadata. Each managed cluster would be
responsible for recording their compliance events through an HTTP API on the hub that abstracts the database.

## Motivation

The current policy compliance history is stored in the policy template's parent policy and is limited to just 10
compliance events. When a policy is deleted, compliance history for that policy is lost for all clusters. If a cluster
is deleted, all the compliance history for that cluster is lost. Additionally, the current compliance history is not
easily queryable and does not provide rich context of the violation. One exception to this is that there are some
metrics that expose whether a policy is compliant or noncompliant but those must be scraped by Prometheus and changes
between scrapes will be lost, therefore, the data cannot be relied upon.

When a user wants to pass an audit or determine when an incident occurred, it is required to have accurrate, historical,
and queryable compliance data.

### Goals

1. Provide an accurate compliance history on the hub or other centralized location.
1. The compliance history is stored in a queryable format.
1. The complaince history can contain data for at least 2 years for a hub that manages 3000 managed clusters with 30
   policies that mostly stay compliant.
1. The compliance history retains data of deleted managed clusters and deleted policies.
1. The outcome is that policy compliance events can be queried by a date range and at least the cluster name, parent
   policy name, policy name, compliance state, and compliance message can be shown. The solution should allow extending
   with more data.
1. This feature should be opt-in.

### Non-Goals

1. Open Cluster Management will not manage the Postgresql database. The user must bring their own.

## Proposal

### Design

#### Database Design

The compliance history needs to be stored efficiently in a queryable format. Since the data format is consistent, using
a relational database is a good fit. A proposed database schema is shown below. Note that this schema allows the same
database to be used for multiple Open Cluster Management hubs, though if managed cluster names overlap, it'd be less
confusing if a separate database on the same Postgresql database be used.

##### clusters Table

| Column     | Description                                                                                |
| ---------- | ------------------------------------------------------------------------------------------ |
| id         | an internal primary key                                                                    |
| name       | a string column for the cluster name in the ManagedCluster object; this will have an index |
| cluster_id | the ACM generated unique cluster ID; this will have a unique constraint                    |

##### parent_policies Table

| Column     | Description                                                         |
| ---------- | ------------------------------------------------------------------- |
| id         | an internal primary key                                             |
| name       | a string column for the name of the parent root policy              |
| namespace  | a string column for the namespace of the parent root policy         |
| categories | an array column of the sorted categories associated with the policy |
| controls   | an array column of the sorted controls associated with the policy   |
| standards  | an array column of the sorted standards associated with the policy  |

There would be a combined unique constraint on the `name`, `categories`, `controls`, and `standards` columns. Note that
the array column is a Postgresql feature and not supported in databases such as SQLite. If we do need to support other
database types in the future, these can be string/text columns using a JSON array format (e.g. ["one","two"]) and
filters on theses columns would check for the substring surrounded by quotes. For example,
`SELECT * FROM parent_policies WHERE categories LIKE '%"my category"%'`. If the database supports JSON functions on a
string/text column, then those could be used instead of a substring query.

##### policies Table

| Column    | Description                                                                                                                        |
| --------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| id        | an internal primary key                                                                                                            |
| kind      | a string column of the policy Kind (e.g. ConfigurationPolicy)                                                                      |
| api_group | a string column of the policy group (e.g. policy.open-cluster-management.io)                                                       |
| name      | a string column for the name of the policy                                                                                         |
| namespace | an optional string column for the namespace of the policy. This will not be set for native OCM policies to avoid data duplication. |
| spec      | a JSONB column of the spec field of the replicated policy as JSON (e.g. ConfigurationPolicy after Hub templates)                   |

There would be a combined unique constraint on the `kind`, `api_group`, `name`, `parent_policy_id`, and `spec` columns.
`spec` would have an index as well.

The `spec` of the policy (e.g. `ConfigurationPolicy`) is stored for a customer to audit the policy contents that
generated a compliance event. This would be stored as JSONB for uniqueness constraints capabilities in Postgres and
efficient search.

##### compliance_events Table

| Column           | Description                                                                               |
| ---------------- | ----------------------------------------------------------------------------------------- |
| id               | an internal primary key                                                                   |
| cluster_id       | a foreign key to the `id` column in the `clusters` table                                  |
| policy_id        | a foreign key to the `id` column in the `policies` table                                  |
| parent_policy_id | an optional foreign key to the parent policy on the hub in the parent_policies table      |
| compliance       | a string column of a compliance state name (i.e. Compliant); this will have an index      |
| message          | a string column with the compliance message                                               |
| timestamp        | a UTC date time column for the timestamp of the compliance event; this will have an index |
| metadata         | a JSONB column with additional metadata                                                   |

The `metadata` field would be empty for now but could be eventually used to store information such as the object diff of
an inform `ConfigurationPolicy` without altering the database schema. This field is not designed for filtering when
generating reports. In the event this feature is expanded to allow non-OCM policy results, the `metadata` field would
likely be used to include `standards`, `categories`, and `controls` fields since they would not inherit those from an
OCM parent policy.

#### How a Compliance Event is Recorded

##### Current State

A policy controller, such as the Configuration Policy controller, will evaluate a policy (e.g. a `ConfigurationPolicy`)
and produce a compliance event. This is currently recorded as a Kubernetes event. The Status Sync controller will
process the Kubernetes event and add the status entry in the local and hub replicated policy that owns the evaluated
policy (i.e. in the `policy-templates` array). A policy's history is limited to 10 events.

##### Recording a Compliance Event

As previously mentioned, the compliance events will be stored in a relational database. This database will not be
managed by Open Cluster Management and is limited to just Postgresql at the momement. The connection information for
this database will be in a secret called `governance-policy-database` in the `open-cluster-management` namespace and
will require either a key of `connectionURL` in the format of
`postgres://username:password@localhost:5432/database_name` or the separate keys of `user`, `password`, `host`, `port`,
`dbname`, and `sslmode`. Additionally, a `ca` key can be set with the PEM encoded certificate authority certificate to
trust for SSL/TLS connections to the Postgres server.

For security reasons, each managed cluster cannot directly write to the database. Instead, an HTTP endpoint is to be
added to the Policy Propagator to perform authorization checks and translating compliance events to the appropriate SQL
commands. This is preferred so that the database design doesn't require a separate database (not server) per cluster for
least privilege authorization. At this time, the HTTP endpoint will only support Postgresql, however, it should be
written with a plugin system so that it can support sending an event on a message in the future. Note that this HTTP
endpoint will only start if the `governance-policy-database` secret exists. Technically speaking, a separate goroutine
would watch for the creation of this secret. Addtionally, the HTTP endpoint will be exposed using an `Ingress` object.
On OpenShift clusters,
[special annotations](https://docs.openshift.com/container-platform/4.13/networking/routes/route-configuration.html#nw-ingress-creating-a-route-via-an-ingress_route-configuration)
will create a route for it that is protected by TLS. On other Kubernetes distributions, a certificate must be provided
for the `Ingress` object.

The authorization code would leverage the Go APIs in
[github.com/stolostron/rbac-api-utils](https://github.com/stolostron/rbac-api-utils/blob/159deac7d398c3470ae730c3fbb8ffbbcca1ac5a/pkg/rbac/rbac.go#L187)
or [kube-rbac-proxy](https://pkg.go.dev/github.com/brancz/kube-rbac-proxy@v0.14.2/pkg/authz#NewSarAuthorizer), and
require the Kubernetes service account recording the compliance event to have write access to the `status` subresource
of the `policies.policy.open-cluster-management.io` resource in the applicable managed cluster namespace on the hub. The
Status Sync controller already has this access. Below is an example of a role on the hub that could record compliance
events for a managed cluster called `cluster1`.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: policy-status-writer
  namespace: cluster1
rules:
  - apiGroups:
      - policy.open-cluster-management.io
    resources:
      - policies/status
    verbs:
      - patch
```

The HTTP endpoint would be at `/api/v1/compliance-events` and would accept a `POST` request with the following schema:

```json
{
  "cluster": { "name": "cluster1" },
  "parent_policy": {
    "name": "etcd-encryption",
    "namespace": "policies",
    // Optional
    "catageories": [],
    // Optional
    "controls": [],
    // Optional
    "standards": []
  },
  "policy": {
    "apiGroup": "policy.open-cluster-management.io",
    "kind": "ConfigurationPolicy",
    "name": "etcd-encryption",
    "spec": {
      "remediationAction": "enforce"
    }
  },
  "event": {
    "compliance": "NonCompliant",
    "message": "configmaps [app-data] not found in namespace default",
    "timestamp": "2023-07-19T18:25:43.511Z",
    // Optional
    "metadata": {}
  }
}
```

To save on network bandwidth and resource utilization, the alternative format is accepted if the database IDs are known:

```json
{
  "cluster": { "name": "cluster1" },
  "parent_policy": {
    "id": "1"
  },
  "policy": {
    "id": "2"
  },
  "event": {
    "compliance": "NonCompliant",
    "message": "configmaps [app-data] not found in namespace default",
    "timestamp": "2023-07-19T18:25:43.511Z",
    // Optional
    "metadata": {}
  }
}
```

If the compliance event is recorded correctly, an HTTP 201 status code is returned.

##### Viewing Compliance Events

Another HTTP endpoint would be added at `/api/v1/compliance-events` that would accept a `GET` request. This would be
responsible for exposing the compliance events and performing the correct authorization. The authorization would require
`get` permissions on the clusters' associated `ManagedCluster` object to be able to list the compliance events for the
cluster. For performance reasons, this is preferred to performing a `SubjectAccessReview` on every cluster namespace for
a user per request. We can utilize the
[github.com/stolostron/rbac-api-utils](https://github.com/stolostron/rbac-api-utils/blob/159deac7d398c3470ae730c3fbb8ffbbcca1ac5a/pkg/rbac/rbac.go#L187)
library to do this.

The following optional query parameters would be accepted on this HTTP endpoint. Note that those without descriptions
just filter on the field it references. Additionally, multiple values can be specified with commas (e.g.
`?cluster.name=cluster1,cluster2`) for "or" filtering. Commas can be escaped with `\,` if necessary.

- cluster.cluster_id
- cluster.name
- direction - the direction to sort by. This defaults to `desc`. It can be `asc` or `desc`.
- event.compliance
- event.message
- event.message_includes - a filter for compliance messages that include the following substring. Only a single value is
  supported.
- event.message_like - a SQL "LIKE" filter for compliance messages. The percent sign `%` represents a wildcard of zero
  or more characters. The underscore sign `*` represents a wildcard of a single character. For example
  `%configmaps [%my-configmap%]%` would match any configuration policy compliance message that refers to the configmap
  `my-configmap`.
- event.reported_by
- event.timestamp
- event.timestamp_after - an RFC 3339 timestamp to indicate only compliance events after this time should be shown.
- event.timestamp_before - an RFC 3339 timestamp to indicate only compliance events before this time should be shown.
- id
- include_spec - a flag to include the policy's spec in the return value. This is not set by default.
- page - the page number in the query. This defaults to 1.
- parent_policy.categories
- parent_policy.controls
- parent_policy.id
- parent_policy.name
- parent_policy.namespace
- parent_policy.standards
- per_page - the number of compliance events returned per page. This defaults to 20 and maxes out at 100.
- policy.apiGroup
- policy.id
- policy.kind
- policy.name
- policy.namespace
- policy.severity
- sort - the field to sort by. This defaults to `event.timestamp`. All fields except `policy.spec` and `event.metadata`
  are sortable by using dot notation. To specify multiple sort options, use commas such as
  `?sort=policy.name,policy.namespace`.

The output would look as follows and be sorted by timestamps in descending order. The content would be filtered based on
the user's access as described above:

```json
{
  "data": [
    {
      "id": 2, // Internally, this is the ID of the compliance_events table to allow a GET query of a particular compliance event.
      "cluster": {
        "name": "cluster1"
      },
      "parent_policy": {
        "id": 3,
        "name": "etcd-encryption",
        "namespace": "policies",
        "catageories": ["CM Configuration Management"],
        "controls": ["CM-2 Baseline Configuration"],
        "standards": ["NIST SP 800-53"]
      },
      "policy": {
        "apiGroup": "policy.open-cluster-management.io",
        "id": 2,
        "kind": "ConfigurationPolicy",
        "name": "etcd-encryption",
        "namespace": "",
        // Only shown with `?include_spec`
        "spec": {}
      },
      "event": {
        "compliance": "NonCompliant",
        "message": "configmaps [app-data] not found in namespace default",
        "timestamp": "2023-07-19T18:25:43.511Z",
        "metadata": {}
      }
    },
    {
      "id": 1,
      "cluster": {
        "name": "cluster2"
      },
      "parent_policy": {
        "id": 2,
        "name": "etcd-encryption",
        "namespace": "policies",
        "catageories": [],
        "controls": [],
        "standards": []
      },
      "policy": {
        "apiGroup": "policy.open-cluster-management.io",
        "id": 4,
        "kind": "ConfigurationPolicy",
        "name": "etcd-encryption",
        "namespace": "",
        // Only shown with `?include_spec`
        "spec": {}
      },
      "event": {
        "compliance": "Compliant",
        "message": "configmaps [app-data] found as specified in namespace default",
        "timestamp": "2023-07-19T18:25:41.523Z",
        "metadata": {}
      }
    }
  ],
  "metadata": {
    "page": 1,
    "pages": 7,
    "per_page": 20,
    "total": 123
  }
}
```

##### Generating a Report

Another HTTP endpoint would be added at `/api/v1/reports/compliance-events` that would accept a `GET` request with the
same query parameters as the `/api/v1/compliance-events` HTTP endpoint (excluding pagination), but would generate a CSV
file in the same format as the `/api/v1/compliance-events` HTTP endpoint, but flattened. For example,
`{"parent_policy": {"name": "policies.etcd-encryption"}}` would become `parent_policy_name` as the column name. Note
that commas and other special characters must be escaped.

If performance testing shows that this is too slow, we could change this a `POST` request that returns a report ID to be
downloaded at a later time. The generated report would be deleted after a certain amount of time.

##### Managed Cluster Phase 1

![Phase 1 Diagram](phase-1.jpeg)

The minimal approach is to have the Status Sync controller also record the compliance event in the database on the hub
when it's recording the compliance event on the parent replicated policy. This phase 1 approach would not be resilient
to lost connectivity to the hub since there would not be a compliance event queue on the managed cluster. This would be
addressed in phase 2.

The Policy Propagator will add entries to the `policies` and `parent_policies` tables as it creates replicated policies.
It will also set the annotations of `policy.open-cluster-management.io/policy-compliance-db-id` and
`policy.open-cluster-management.io/parent-policy-compliance-db-id` on each replicated policy template (e.g.
`ConfigurationPolicy`) entry within the replicated policy to match what was inserted in the database. Additionally,
it'll set the `policy.open-cluster-management.io/policy-spec-sha1` set to the hex format of the SHA1 of the `spec` of
the policy (e.g. `ConfigurationPolicy`). This way, the Status Sync controller won't need to provide as much data when
recording a compliance event. Reducing bandwidth utilization to the hub is helpful in edge scenarios. With this in mind,
the current compliance events sent by the controllers will set the database related annotations from the policy being
evaluated. With this additional information, the Status Sync controller can avoid sending the whole spec, policy, and
parent policy in the`/api/v1/compliance-events` `POST` request. This also reduces the chance of race conditions where a
compliance event is sent and the policy is updated before the compliance event is recorded in the database. The rest of
the data required for the `POST` request can be found in the Kuberenetes `Event` object.

To record a compliance event when a policy is deleted, the Spec Sync controller will record these compliance events
before it deletes the policy template (e.g. ConfigurationPolicy). A finalizer on the policy template objects was
considered but it adds complexity with little additional value since anyone with access to delete a policy can remove
the finalizer.

There is an existing compliance history in the replicated `Policy` objects on the hub, though it is potentially
incomplete due to the shortcomings previously mentioned. If a user wants to import that data in the new compliance event
store, a script will be provided that will add entries from the replicated `Policy` objects' `status.details` field on
the hub. In this case, the `spec` would be set to `unknown`.

##### Managed Cluster Phase 2 - Add Resiliency

![Phase 2 Diagram](phase-2.jpeg)

Phase 1 has the limitation of not being able to withstand disconnects from the hub. This would prevent a user from
relying on the data where the following events can occur: network connectivity is flaky, the managed cluster is expected
to be disconnected from the hub (e.g. managed cluster on a ship), or the hub is offline for maintenance.

To account for this, we need a queueing mechanism on each managed cluster that stores compliance events when the hub is
not reachable. A queuing mechanism in memory on the Status Sync controller would be the simplest approach, however, this
would not survive container restarts and could consume a large amount of memory for a small cluster. Another approach
would be to have a message broker such as RabbitMQ that supports queues on disk, however, the resource utilization on
small clusters would be too large and it brings with it a maintenance cost of Open Cluster Management (OCM) needing to
maintain this deployment for the user. Another option is using SQLite as a queue, however, it would need to be on a
persistent volume and thus, the newer [write-ahead-log](https://www.sqlite.org/wal.html) concurrency option is not an
option due to it potentially being on network storage. The older concurrency options involve an
[exclusive lock](https://www.sqlite.org/lockingv3.html). This would mean that future evolutions that involve concurrency
such as that described in phase 3 would not be supported.

The proposed approach is to use [embedded etcd](https://pkg.go.dev/go.etcd.io/etcd/server/v3/embed) as this is very
light weight as shown in the implementation details section below. This supports concurrent writes and has an HTTP
endpoint for phase 3. This might seem unconventional since etcd is a key-value store and not a queue per se, but we can
use it in this manner.

The Governance Policy Framework addon pod would run a separate goroutine that runs embedded etcd. In this phase, this
would have the etcd GRPC endpoint limited to localhost to avoid needing authentication enabled. The Status Sync
controller would record the compliance event in etcd with the key format of
`compliance-events/<timestamp from Kubernetes event>-<event-uid>`. For example,
`compliance-events/2023-07-25T13:15:08Z-4b400c5f-3828-4138-ad4f-03de4d69655b`. The value would be in the format of the
`POST` request for the `/api/v1/compliance-events` HTTP endpoint on the hub. The Governance Policy Framework addon pod
would run a separate goroutine that would periodically poll etcd for new compliance events and try to record them on the
hub. It would do this in order by sorting by the timestamps in the keys. When successful, it would delete the entry in
etcd. This goroutine would prevent the pod from shutting down until all messages are synced unless the embedded etcd is
backed by a persistent volume as detailed below.

By default, the embedded etcd would use an [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
volume which requires no persistent volume configuration on the managed cluster and would survive container restarts. If
the pod is removed from the node, the data is lost however. For this reason, OCM could be configured to request a
persistent volume for better accuracy. This would be configured using an
[AddOnDeploymentConfig](https://github.com/open-cluster-management-io/enhancements/blob/main/enhancements/sig-architecture/58-addon-configuration/README.md)
`customizedVariables` entry. The variable would be `{"name": "UsePVForComplianceEventCache", "value": true}`. Having
this variable set would trigger the Governance Policy Addon controller to create a `PersistentVolumeClaim` instead of
using `emptyDir`. We should also consider an "out of the box" policy that checks for instances where a persistent volume
is not used to warn the user of this.

##### Managed Cluster Phase 3 - Additional Metadata

![Phase 3 Diagram](phase-3.jpeg)

In my opinion, using Kubernetes events as a message queueing system is abusing what it is meant to be, which is
information for the cluster user. This generates a lot of noise that is not relevant to the user. Additionally, the new
[Event](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/event-v1/#Event) API only allows a `note`
of 1kB. This is very limiting for when the policy components would need to migrate to this new `Event` API. That being
said, the old `Event` API is not expected to be removed in the near future though.

It is proposed to expose the embedded etcd GRPC endpoint and have policy controllers directly record compliance events
using it instead of recording Kubernetes events when the embedded etcd is backed by a persistent volume. The GRPC
endpoint would be protected with authentication and the credentials would be stored in a secret called
`governance-policy-framework-cache` in the managed cluster namespace. Each policy controller would read this secret on
start up. Recording the compliance event directly from the controller would allow recording rich metadata about the
compliance event. One example is the diff between what is in the policy and what is on the cluster, similar to what
ArgoCD does.

The Governance Policy Framework would still have a goroutine that periodically polls etcd for compliance events while
disconnected from the hub, however, when there is a connection to the hub, it can transition to watching for new keys
with the prefix `compliance-events/`.

### User Stories

#### Story 1 - Phase 1

As a policy user, I require the compliance history of my fleet of clusters so that I can perform an audit over the last
two years.

#### Story 2 - Phase 2

As a policy user, I require that compliance history events still be recorded when the cluster is disconnected from the
hub.

#### Story 3

As a policy user, I'd like an easy way to generate a spreadsheet report of policy compliance events within a date range
to make auditing easier.

#### Story 4 - Phase 3 (Optional)

As a policy user, I require that compliance history events contain additional metadata about the violation such as the
diff between what is declared in the policy and what is on the cluster to know what the exact violation was. This will
help in forensic investigations after a breach has occurred of a managed cluster.

#### Story 5

As a policy user, I'd like to import my existing compliance history to the new compliance events database.

### Implementation Details/Notes/Constraints [optional]

#### Embedded etcd Resource Utilization

The following minimal embedded etcd Go program uses around 3MB of memory for the etcd portion under light load:

```go
package main

import (
	"log"
	"net/http"
	"net/url"
	"time"

	"go.etcd.io/etcd/server/v3/embed"
)

func main() {
	cfg := embed.NewConfig()
	cfg.AdvertiseClientUrls = []url.URL{}
	cfg.AdvertisePeerUrls = []url.URL{}
	cfg.ListenPeerUrls = []url.URL{}
	cfg.ListenMetricsUrls = []url.URL{}
	cfg.EnableV2 = false
	cfg.Dir = "default.etcd"
	e, err := embed.StartEtcd(cfg)
	if err != nil {
		log.Fatal(err)
	}
	defer e.Close()
	select {
	case <-e.Server.ReadyNotify():
		log.Printf("Server is ready!")
	case <-time.After(60 * time.Second):
		e.Server.Stop() // trigger a shutdown
		log.Printf("Server took too long to start!")
	}
	log.Fatal(<-e.Err())
}
```

### Risks and Mitigation

### Open Questions [optional]

1. How to enable it on the managed cluster? Should it be per managed cluster?
1. Is the RBAC sufficient? Instead of "get" access on a managed cluster, should we have a custom verb of
   `get-compliance-events`?

### Test Plan

**Note:** _Section not required until targeted at a release._

### Graduation Criteria

It would be technical preview in the release after implementation. After initial usage, the following release may have
updates or make this feature GA.

### Upgrade / Downgrade Strategy

There are no concerns with upgrades/downgrads since this is not a breaking change.

### Version Skew Strategy

## Implementation History

N/A

## Drawbacks

- For the most accurate results, it requires a persistent volume on each managed cluster.

## Alternatives

N/A

## Infrastructure Needed [optional]

N/A
