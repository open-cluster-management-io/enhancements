# Investigating SQL-backed storage for the OCM hub

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

> **Status note:** Technology demonstration. This proposal records the results
> of two storage investigations for community discussion; it does not select
> an architecture or commit to implementation.

## Summary

This proposal records two investigations into storing Open Cluster Management
(OCM) hub state in PostgreSQL:

1. An **aggregated API server** serving selected OCM APIs through Kubernetes
   API aggregation, persisting objects in PostgreSQL.
2. A **separate hosted Kubernetes API** persisting its state through
   [Kine](https://github.com/k3s-io/kine), an etcd-compatible storage adapter,
   backed by PostgreSQL.

Both prototypes demonstrated limited OCM functionality in disposable local
environments. Neither establishes production readiness, and neither measured a
performance or storage-load improvement over an etcd-backed hub.

## Motivation

An OCM hub stores both OCM resources and Kubernetes-native resources in its
etcd cluster by default. As fleet size and workload volume grow, it is worth
investigating alternative storage arrangements and understanding their
compatibility requirements, operational costs, and potential effect on hub
storage. The two investigations explore different boundaries: selected OCM
APIs behind an aggregated server, versus an entire separate Kubernetes API on
SQL-backed storage.

### Goals

- Record the architecture, results, and limitations of both investigations.
- Identify the open questions further work must answer.

### Non-Goals

- Selecting a production architecture or committing to a database.
- Defining a supported deployment or migration path.
- Claiming reduced etcd load or improved performance; nothing was measured.

## Investigation 1: Aggregated API server

```text
OCM clients → Kubernetes API aggregation → aggregated OCM API server → PostgreSQL
```

The prototype serves selected OCM group/versions through API aggregation.
Kubernetes remains the authentication authority; authorization is delegated
through `SubjectAccessReview`.

A local demo registered a spoke and the hub's local cluster, and delivered a
ManifestWork that was stored in PostgreSQL, watched by the spoke work agent,
and applied on the spoke — exercising the watch-list protocol and
field-selector authorization semantics that OCM controllers and agents depend
on.

### Findings and limitations

- An `APIService` owns a **whole group/version**, not an individual resource.
  The demo shadowed existing CRD-backed APIs on a disposable cluster;
  pre-takeover objects remained stranded in etcd. This is not a migration
  strategy.
- Kubernetes-native resources — Leases, CSRs, Secrets, ConfigMaps — remain in
  hub etcd.
- CRD behavior is not reproduced: no schema validation, defaulting, CEL,
  admission webhooks, OpenAPI, or version conversion. OCM admission webhooks
  were bypassed.
- The PostgreSQL watch journal and revision allocation are prototype quality
  (single-row write serialization, no compaction) and unvalidated for scale or
  recovery.

## Investigation 2: Separate hosted Kubernetes API with Kine

```text
OCM controllers and spokes → separate kube-apiserver → Kine → PostgreSQL
```

The prototype runs OCM hub controllers in Hosted mode (see the [hosted
deployment mode](../33-hosted-deploy-mode/)) against a standalone
kube-apiserver. The hosting cluster keeps its native etcd-backed API for
infrastructure; all OCM hub state — CRDs, registration CSRs, hub RBAC, Leases,
ManifestWorks, placements, add-on resources — is stored through Kine in
PostgreSQL.

A clean automated local setup connected two spokes; both joined, became
available, and received ManifestWork and add-on workloads. The ManagedCluster
CRD was present only in the standalone OCM API, and PostgreSQL contained Kine
records.

### Findings and limitations

- Kine reported slow SQL queries and duplicate-key conflicts on
  `kine_name_prev_revision_uindex` during concurrent add-on status updates.
  Flows completed, but the cause and correctness implications are unresolved.
- No throughput, latency, durability, high-availability, backup/restore,
  production-security, or compatibility claims beyond the tested versions.
- TLS was disabled and components were single-instance — acceptable only for
  a disposable demo. Kine implements the etcd API subset Kubernetes requires,
  not a general etcd replacement.

## Comparison

| | Aggregated API server | Hosted API with Kine |
|---|---|---|
| API surface | Selected OCM APIs | Full Kubernetes API, separately deployed |
| SQL storage scope | Selected OCM custom resources | All state of the separate API, including Kubernetes-native resources |
| Compatibility effort | Reproducing CRD and Kubernetes API semantics | Validating Kubernetes-on-Kine behavior and reliability |
| Migration | Unresolved; shadows whole group/versions | Unresolved |
| Production readiness | Not established | Not established |

## Open questions

- Which hub resources and workloads account for enough etcd activity for a
  change to be worthwhile?
- Can either approach preserve the API, authorization, watch, and lifecycle
  behavior OCM depends on?
- Can the Kine conflicts be reproduced and explained without lost or
  inconsistent updates?
- How would deployment, migration, rollback, backup, and disaster recovery
  work?
- What do controlled benchmarks show against an equivalent etcd-backed hub?

A ManifestWork load simulator accompanies the investigations for comparable
fleet workloads; its direct-hub projections are modeled estimates, not
measured savings.

## Artifacts

The proof-of-technology artifacts were posted as pull requests against the OCM
repository and closed as complete once recorded here; the closed references
below remain accessible:

- Aggregated API server: POC server, hub API compatibility inventory,
  three-cluster demo with recorded results —
  [ocm#1820](https://github.com/open-cluster-management-io/ocm/pull/1820)
  (closed)
- Hosted API with Kine: hosted hub demo with test results, ManifestWork load
  simulator, Hosted-mode chart configuration —
  [ocm#1821](https://github.com/open-cluster-management-io/ocm/pull/1821)
  (closed)

## Related work

- [Hosted deployment mode](../33-hosted-deploy-mode/)
- [Event-based ManifestWork](../224-event-based-manifestwork/) — an
  alternative approach to reducing hub storage.

## Implementation History

- 2026-11-09: Initial provisional proposal recording the investigation
  results and artifact links.
