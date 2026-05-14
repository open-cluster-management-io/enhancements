# Make ManifestWorkReplicaSet a default feature of OCM (enable by default)

> **Note:** This document was drafted with the assistance of AI to accelerate structuring, formatting, and clarity. AI was also used to verify that the proposal is compliant with the enhancement template and that the technical concepts described are accurate and workable. All content is based on human research, discussions, and community context.

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)
- [ ] Changes are shared across community channels such as Slack, meetings, and mailing lists

## Summary

ManifestWorkReplicaSet (MWRS) is currently behind a feature gate
(`Default: false`) and has been since its introduction.  Since release it has become a popular mechanism in OCM
for deploying ManifestWorks across groups of clusters selected by a Placement.
This enhancement proposes changing its default from
`false` to `true` so it is enabled out of the box.


## Motivation

MWRS is the natural complement to Placement: Placement selects clusters, MWRS
deploys workloads to them. Today, users who discover MWRS through documentation
or community examples cannot use it without first modifying their ClusterManager
CR to enable the feature gate. This creates an unnecessary barrier for a
capability that is fundamental to multi-cluster workload distribution.

Additionally, several active enhancements build on MWRS (rollback, rollout
plugins, ignore-field selectors) which signals the community considers it a
core capability, not an experimental one. Its current status does not reflect the
level of investment and dependency it has accumulated.

Finally, MWRS is a key component of several active projects in the OCM ecosystem, including the flower-addon, federated-learning-controller,
the dynamic-scoring-framework, and clusteradm. Keeping it behind an opt-in gate limits adoption and risks discouraging investment in new features.

### Goals

1. Change the MWRS feature gate default from `false` to `true` so it is
   enabled out of the box.
2. Resolve known bugs that would be unacceptable for a default-enabled feature,
   specifically issues where MWRS masks errors or reports misleading status.
3. Ensure adequate test coverage for the MWRS controller and webhook, including
   e2e tests covering error propagation and status reporting.
4. Update documentation to reflect MWRS as a default-enabled feature.

### Non-Goals

1. Graduating MWRS to GA (stable). That is a future step once the
   default-enabled feature has been validated in production.
2. API changes to the ManifestWorkReplicaSet CRD. This proposal only changes
   the feature gate default. Any API evolution is covered by separate
   enhancements.
3. Removing the feature gate entirely. Users can still disable MWRS
   if needed.

## Proposal

### User Stories

#### Story 1

As a platform engineer, I want to deploy ManifestWorks across clusters selected
by a Placement without having to first discover and enable a feature gate in my
ClusterManager CR. MWRS should work out of the box.

#### Story 2

As a contributor building an OCM add-on or feature using MWRS I want MWRS to be a default capability so that my work does not
require users to enable an additional prerequisite gate.


### Implementation Details/Notes/Constraints

The core change is a one-line update in
`open-cluster-management-io/api/feature/feature.go`:

```go
// Before
ManifestWorkReplicaSet: {Default: false, PreRelease: featuregate.Alpha},

// After
ManifestWorkReplicaSet: {Default: true, PreRelease: featuregate.Beta},
```

This causes the work controller deployment and MWRS webhook to be started by
default when the ClusterManager is reconciled, without requiring explicit
feature gate configuration.

**Prerequisites that must be resolved before this change lands:**

- [#1522](https://github.com/open-cluster-management-io/ocm/issues/1522):
  MWRS reports `PlacementDecisionEmpty` when ManifestWork apply fails, masking
  the real error. A default-enabled feature must not hide errors from users.
- [#1380](https://github.com/open-cluster-management-io/ocm/issues/1380):
  Should use `generateName` to create ManifestWork from MWRS.
- [#1198](https://github.com/open-cluster-management-io/ocm/issues/1198):
  Advanced MWRS use-cases need to be validated or documented as known
  limitations.
  
  NOTE: There may be more. See Open Questions below.

### Risks and Mitigation

**Risk:** Enabling MWRS by default starts additional controllers and webhooks
that consume resources on the hub, even for users who don't use MWRS.

**Mitigation:** The MWRS controller and webhook are lightweight and they are
idle when no ManifestWorkReplicaSet resources exist. The resource overhead is
comparable to other default-enabled controllers. Users who need to minimize
hub footprint can disable the gate.

**Risk:** Existing clusters upgrading to a version with MWRS defaulted to true
will have the controller start automatically.

**Mitigation:** The controller is no-op without MWRS resources. No existing
workloads are affected. The webhook only validates MWRS resources, not
ManifestWorks.

## Design Details

### Open Questions

1. Are there any other open issues beyond #1522, #1380, and #1198 that should
   block graduation?
2. Should the `v1alpha1` API version of MWRS be promoted to `v1beta1` as part
   of this graduation, or can the API version change follow separately?

### Test Plan

- Existing unit and integration tests for the MWRS controller must pass.
- E2e tests must cover:
  - Basic MWRS creation, update, and deletion across multiple clusters.
  - Error propagation: when a ManifestWork fails to apply, the error must be
    visible in MWRS status (validates #1522 fix).
  - Placement changes: adding/removing clusters from a Placement must
    correctly update the set of ManifestWorks.
  - Status aggregation: MWRS summary must accurately reflect the state of
    all child ManifestWorks.
- Verify that enabling MWRS by default does not regress existing ManifestWork
  or Placement functionality.

### Graduation Criteria

**Enable by default (this proposal):**

- All blocking bugs resolved (#1522, #1380).
- E2e test coverage for error propagation and status reporting.
- Documentation updated to describe MWRS as a default feature.
- At least one release with the fix for #1522 shipped and validated.

### Upgrade / Downgrade Strategy

**Upgrade:** Clusters upgrading to the version where MWRS defaults to true
will have the MWRS controller and webhook start automatically. This is safe
because the controller is idle without MWRS resources. No user action is
required to maintain previous behavior.

**Downgrade:** If a user needs to disable MWRS after upgrade, they can set
the feature gate to false in their ClusterManager CR:

```yaml
spec:
  workConfiguration:
    featureGates:
      - feature: ManifestWorkReplicaSet
        mode: Disable
```

Existing ManifestWorkReplicaSet resources will stop being reconciled but will
not be deleted. Re-enabling the gate resumes reconciliation.

### Documentation and Awareness
* Documentation needs to be updated as this represents a change in behavior.
* The change should be well publicized and shared across Slack channels, email lists, and community meetings.


### Version Skew Strategy

MWRS is a hub-only feature. There is no version
skew concern between hub and spoke, as spokes do not need to be aware of MWRS.

If a hub is upgraded and MWRS enabled while spokes are on an older version, the
spokes continue to process ManifestWorks as before. MWRS is transparent to
spoke agents.

## Implementation History

- 2024: MWRS introduced behind a feature gate (default disabled).
- Enhancement 229: Rollback support for MWRS (building on MWRS as foundation).
- Enhancement 231: Rollout plugin extensibility for MWRS.
- 2026-05-13: This proposal filed to enable MWRS by default.

## Drawbacks

Some users may prefer a minimal hub footprint and not want additional
controllers running by default. However, the MWRS controller is effectively
no-op without MWRS resources, and the gate remains available to disable it.

## Alternatives

**Keep MWRS disabled by default and rely on documentation to guide users to enable it.**
This is the status quo. It works but creates unnecessary friction for a
capability that the community is actively building on. Features that other
enhancements depend on should be easily accessible.


## Infrastructure Needed

No new infrastructure needed. Existing CI and e2e test infrastructure is
sufficient.
