# Investigating SQL-backed storage for the OCM hub

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

> **Status note:** This is a **technology demonstration** proposal. It records
> the results of two storage investigations for community discussion. It does
> not select an architecture, define a migration path, or commit to
> implementation.

## Summary

This proposal records the results of two investigations into storing Open
Cluster Management (OCM) hub state in PostgreSQL:

1. An **aggregated API server** that serves selected OCM APIs through
   Kubernetes API aggregation and persists their objects in PostgreSQL.
2. A **separate hosted Kubernetes API** whose state is persisted through
   [Kine](https://github.com/k3s-io/kine), an etcd-compatible storage adapter,
   with PostgreSQL as the backend.

Both prototypes demonstrated limited OCM functionality in disposable local
environments. Neither establishes production readiness, and neither measured a
performance or storage-load improvement over an etcd-backed hub. This proposal
shares the findings, limitations, and open questions to support community
discussion of whether and how further work should proceed.

## Motivation

An OCM hub stores OCM resources and Kubernetes-native resources through the
Kubernetes API and, by default, its etcd cluster. As fleet size and workload
volume grow, it is worth investigating alternative storage arrangements and
understanding their compatibility requirements, operational costs, and
potential effect on hub storage.

The two investigations explore different boundaries for moving hub state onto a
SQL database: serving selected OCM APIs from an aggregated API server, or
placing an entire separate Kubernetes API for hub state on a SQL-backed storage
layer.

### Goals

- Record the architecture, observed results, and limitations of both
  investigations.
- Identify the compatibility and operational questions that further work must
  answer.
- Establish what evidence would be required before proposing any
  implementation.

### Non-Goals

- Selecting a production architecture or committing to PostgreSQL or any other
  database.
- Defining a supported deployment, migration, upgrade, or rollback path.
- Claiming reduced etcd load or improved performance; no comparable
  measurements were taken.

## Investigation 1: Aggregated API server

```text
OCM clients → Kubernetes API aggregation → aggregated OCM API server → PostgreSQL
```

The prototype serves selected OCM group/versions through Kubernetes API
aggregation. Kubernetes remains the authentication authority; the prototype
verifies the aggregation proxy client certificate and delegates authorization
through `SubjectAccessReview`.

A disposable local demo registered a spoke cluster and the hub's local cluster,
and delivered a ManifestWork that was stored in PostgreSQL, watched by the
spoke work agent through the aggregated server, and applied on the spoke. The
implementation exercised list/watch behavior that OCM controllers and agents
depend on, including the watch-list protocol with initial events, and
field-selector authorization semantics required by the RBAC granted to spoke
agents.

### Findings and limitations

- An `APIService` owns a **whole group/version**, not an individual resource.
  The demo explicitly shadowed existing CRD-backed APIs on a disposable
  cluster; objects created before the takeover remained stranded in the
  original storage. This is not a migration strategy.
- Kubernetes-native resources — including Leases, CSRs, Secrets, and
  ConfigMaps — remain in the hub's etcd. Moving them would require separately
  authenticated internal APIs and coordinated controller and agent changes.
- The prototype does not implement full CRD behavior: structural schema
  validation, defaulting, CEL validation, admission webhooks, OpenAPI schemas,
  and API version conversion are missing. OCM admission webhooks were bypassed
  for proxied requests in the demo.
- The PostgreSQL watch journal and revision allocation are prototype quality:
  writes serialize through a single revision row, and the journal is not
  compacted. They are not validated for scale or recovery.

## Investigation 2: Separate hosted Kubernetes API with Kine

```text
OCM controllers and spokes → separate kube-apiserver → Kine → PostgreSQL
```

The prototype runs OCM hub controllers in Hosted mode against a separate
standalone Kubernetes API server, building on the [hosted deployment
mode](../33-hosted-deploy-mode/). The hosting cluster keeps its native
etcd-backed API for infrastructure; the separate API server stores OCM hub
state — including OCM CRDs, registration CSRs, hub RBAC, Leases,
ManifestWorks, placements, and add-on resources — through Kine and PostgreSQL.

A clean automated local setup deployed the topology and connected two spokes.
Both joined, became available, and received ManifestWork and add-on workloads.
API checks confirmed the separation: the ManagedCluster CRD was present in the
standalone OCM API and absent from the hosting API, and PostgreSQL contained
Kine records.

### Findings and limitations

- During testing, Kine reported slow SQL queries and duplicate-key conflicts
  on `kine_name_prev_revision_uindex` during concurrent add-on status updates.
  The tested flows eventually completed, but the cause and correctness
  implications are unresolved. This run is not evidence of acceptable write
  reliability.
- The demo did not establish throughput, latency, durability after failure,
  high availability, backup and restore, production security, or compatibility
  beyond the tested Kubernetes and Kine versions.
- Database connections disabled TLS and components were single-instance;
  these choices are acceptable only for a disposable demonstration.
- Kine implements the etcd API subset Kubernetes requires; it is not a general
  etcd replacement, and the tested version combination needs validation before
  broader conclusions.

## Comparison

| | Aggregated API server | Hosted API with Kine |
|---|---|---|
| API surface | Prototype serves selected OCM APIs | Full Kubernetes API, separately deployed |
| SQL storage scope | Selected OCM custom resources | All state of the separate API, including Kubernetes-native hub resources |
| Compatibility effort | Reproducing CRD and Kubernetes API semantics (validation, admission, watch, conversion) | Validating Kubernetes-on-Kine behavior and storage reliability |
| Migration | Unresolved; the demo shadows whole group/versions and strands existing objects | Unresolved |
| Production readiness | Not established | Not established |

## Open questions and further validation

- Which hub resources and workloads account for enough etcd activity for a
  change to be worthwhile? No attribution has been measured.
- Can either approach preserve the required API, authorization, watch, and
  lifecycle behavior that OCM controllers and agents depend on?
- Can the Kine conflicts observed under concurrent writes be reproduced and
  explained without lost or inconsistent updates?
- How would deployment, migration, rollback, upgrade, backup, and disaster
  recovery work for each topology?
- What do controlled benchmarks show against an equivalent etcd-backed hub
  under representative load?
- What security and operational requirements (TLS, credential rotation, HA,
  multi-replica operation) would each topology impose?

A ManifestWork load simulator was developed alongside these investigations to
generate comparable fleet workloads and collect API-server, etcd, and database
metrics. Its modeled direct-hub projections are estimates and must not be
presented as measured savings.

## Artifacts

The prototype implementations and detailed test results for both
investigations are published as proof-of-technology pull requests against the
OCM repository. They are records of the investigations, not proposals to
merge:

- **Aggregated API server investigation** — the POC server with its
  architecture and implementation specs, the hub API compatibility inventory,
  and the three-cluster demo with recorded results and object counts:
  [open-cluster-management-io/ocm#1820](https://github.com/open-cluster-management-io/ocm/pull/1820)
- **Hosted Kubernetes API with Kine investigation** — the Kine/PostgreSQL
  hosted hub demo with its implementation and test results, the ManifestWork
  load simulator, and the Hosted-mode chart configuration it required:
  [open-cluster-management-io/ocm#1821](https://github.com/open-cluster-management-io/ocm/pull/1821)

## Related work

- [Hosted deployment mode](../33-hosted-deploy-mode/)
- [Event-based ManifestWork](../224-event-based-manifestwork/) — an
  alternative approach to reducing hub storage for ManifestWorks.

## Implementation History

- 2026-11-09: Initial provisional proposal recording the investigation
  results.
