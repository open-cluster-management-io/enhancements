# Let hosted addons discover their hosting cluster automatically

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
[website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

OCM has two separate notions of "hosting cluster": where a klusterlet's own controllers run
([33-hosted-deploy-mode](../33-hosted-deploy-mode/README.md)), and where a specific addon's agent
runs ([63-hosted-addon](../63-hosted-addon/README.md)). Nothing today checks that these two agree.
When they disagree, the addon gets stuck at `Available: Unknown` with no clue why.

This proposal lets a klusterlet optionally report where it actually runs, lets the addon framework
check a declared hosting cluster against that report, and lets an addon deployer skip naming a
hosting cluster at all and let the framework resolve it. All of it is opt-in and off by default.

### Terms

- **K-hosting-cluster**: the cluster a klusterlet's own controllers run on (KEP-33).
- **A-hosting-cluster**: the cluster an addon's `hosting-cluster-name` annotation names (KEP-63).
- **Self-report**: a klusterlet publishing its own K-hosting-cluster so the hub can see it.
- **Auto-discovery**: the hub resolving and writing an addon's A-hosting-cluster automatically from
  a self-report, instead of a human setting it.

## Motivation

[Issue #188](https://github.com/open-cluster-management-io/enhancements/issues/188) describes a
hosted-mode addon that ends up permanently unavailable because its declared hosting cluster doesn't
match the hosting cluster its target's klusterlet is actually using. Nothing in the framework checks
the two values against each other today, so the addon can look fine when it's created and only get
stuck later, once a klusterlet migration or a typo causes the two values to drift apart.

### Two "hosting cluster" concepts that never talk to each other

| | klusterlet's hosting cluster | addon's hosting cluster |
|---|---|---|
| Where it's declared | `Klusterlet.spec.ClusterName`, or `Hosted.ManagementClusterName` in Hosted mode, on the spoke | `addon.open-cluster-management.io/hosting-cluster-name` annotation, on the hub |
| Visible to the hub today | No | Yes |

This is already a hard requirement, not something this proposal introduces:
[63-hosted-addon](../63-hosted-addon/README.md#constraints) states outright that "the hosting cluster
of the addon must be the same hosting cluster of the klusterlet." An addon on a given target can still
choose Default or Hosted mode independently of that target's own klusterlet mode - nothing here changes
that choice - but once an addon *does* run Hosted, KEP-63 already says its hosting cluster has to be
the same one its target's klusterlet uses. Nothing today ever checks that the two values actually agree.

The hub has no way to see where a klusterlet's controllers are actually running. There's no field on
`ManagedCluster`, no `ClusterClaim`, nothing. So even if the framework wanted to compare the two
values, it couldn't. That's the gap this proposal closes.

An addon's Lease is checked from whichever cluster the framework believes is hosting it. If that
belief is wrong, the check runs against the wrong API server and finds nothing, forever. It's not a
flaky check that eventually recovers, and there's no error message today pointing at this as the
cause.

[63-hosted-addon](../63-hosted-addon/README.md#graduation-criteria) already lists, as an unfulfilled
Beta criterion, an addon deployer not having to name a hosting cluster at all - reasoning that its own
[constraints](../63-hosted-addon/README.md#constraints) make that cluster unique once it's known. What
was missing was ever being able to know it: the hub had no way to see where a klusterlet's controllers
actually run. The auto-discovery piece of this proposal, described in Design Details below, is what
closes that gap.

### Goals

- Let a klusterlet optionally report the cluster its own controllers run on.
- Let the framework check an addon's declared hosting cluster against that report, and surface a
  clear condition when they don't match.
- Let an addon deployer skip naming a hosting cluster and have the framework resolve it once,
  automatically, the first time.
- Keep every part of this opt-in. An operator who upgrades and changes nothing should see no
  difference in behavior.

### Non-Goals

- Changing how hosted-mode addons get deployed once a valid hosting cluster is known. That's already
  defined by [63-hosted-addon](../63-hosted-addon/README.md).
- Destroying anything that already exists because of a mismatch. A mismatch is only ever a condition.
- Cryptographic proof that a klusterlet's self-report is accurate. Same trust model as any other
  `ClusterClaim`.
- Making any of this a default, ever.
- Re-resolving an addon's hosting cluster once it's already been resolved, for any reason - including
  following a later klusterlet migration. Auto-discovery fills in an unset value once and never
  revisits it.
- Moving an already-deployed addon to a different hosting cluster. Out of scope; delete and recreate
  it instead.

## Proposal

### User Stories

#### Hub administrator running hosted-mode addons

I want to be told when an addon's declared hosting cluster doesn't match where its target's
klusterlet actually runs, instead of debugging a silent `Available: Unknown`.

#### Hub administrator managing many hosted-mode addons

I don't want to type the hosting cluster name by hand for every addon I stand up. I want to say
"this addon is hosted, figure out where" and have it get filled in correctly the first time.

#### Operator who never uses hosted mode

I want zero behavior change on my klusterlets and addons unless I explicitly ask for it.

### Risks and Mitigations

**A self-reported claim can be wrong.** A `ClusterClaim` has the same trust level as any other
self-reported fact about a spoke today, with no attestation beyond what the spoke chooses to report -
the same trust model the existing `4-cluster-claims` mechanism already has. Auto-discovery adds a
bound on top of that trust: it only ever routes an addon to a cluster sharing a `ManagedClusterSet`
with the target (see the Guardrail below), so a compromised spoke can at worst redirect its own addon
within a grouping an administrator already made, never to an arbitrary cluster in the fleet. A
compromised spoke could also use its unrestricted `create` permission (see Self-reported hosting
cluster below) to create `ClusterClaim` objects under names other than the reserved one; nothing in
this proposal reads or trusts any claim but the reserved name, so those are inert noise, not a path to
a different outcome.

**Turning on self-report can reveal a pre-existing mismatch.** The moment a target's klusterlet
starts self-reporting, every Hosted-mode addon on that target gets compared against it, including
ones that were already deployed and happen to be mismatched. That's by design. It's always just a
condition change - nothing described anywhere in this proposal ever tears an addon down, so turning
on self-report can't take one down as a side effect.

**A known gap is accepted rather than solved here.** Turning `ReportHostingCluster` back off leaves
the last-published claim in place rather than deleting it, so validation and auto-discovery keep
treating it as live until something else overwrites or removes it.

## Design Details

This is three pieces that build on each other. Only the first has its own opt-in; the rest activate as
a consequence of it, or carry a separate opt-in of their own:

1. A klusterlet can **self-report** its own K-hosting-cluster - the one opt-in a klusterlet needs.
2. **Validation** has no opt-in of its own: it activates automatically, for any addon, the moment its
   target has a self-report to compare against.
3. Instead of a human declaring A-hosting-cluster, the hub can **auto-discover** it from the
   self-report - a separate opt-in on the addon type, on top of self-report existing.

A hub can turn on self-reporting alone and get a real improvement - validation activates with it,
giving a clear condition instead of a silent hang - without ever touching auto-discovery.

### 1. Self-reported hosting cluster

A klusterlet gets a new opt-in field. When set, the klusterlet operator publishes its own
K-hosting-cluster as a `ClusterClaim`, the same mechanism [4-cluster-claims](../4-cluster-claims/README.md)
already uses for a spoke to expose facts about itself to the hub.

```golang
// KlusterletDeployOption describes the deployment options for klusterlet
type KlusterletDeployOption struct {
	// Mode can be Default, Hosted, Singleton, or SingletonHosted. (pre-existing field, shown here
	// only for context - self-report reads it but doesn't add a new value to it)
	// +optional
	Mode InstallMode `json:"mode"`

	// ... existing fields ...

	// ReportHostingCluster, when set to Enable, makes this klusterlet self-report where its own
	// controllers actually run - its own cluster name when they run locally, or
	// Hosted.ManagementClusterName when they run on a separate management cluster - via a
	// reserved ClusterClaim named "hosting-cluster.open-cluster-management.io". Disabled by
	// default: the klusterlet-operator emits nothing and behaves exactly as it does today unless
	// this is explicitly set to Enable.
	// +optional
	ReportHostingCluster ReportHostingClusterMode `json:"reportHostingCluster,omitempty"`

	// Hosted holds configuration for a mode whose controllers run on a separate management
	// cluster rather than locally. (pre-existing field, shown here only for context)
	// +optional
	Hosted *KlusterletHostedConfiguration `json:"hosted,omitempty"`
}

// ReportHostingClusterMode is the type for KlusterletDeployOption.ReportHostingCluster. A typed
// enum, not a bool, so a later mode can be added without a breaking change to the field's shape.
// +kubebuilder:validation:Enum=Enable;Disable
type ReportHostingClusterMode string

const (
	ReportHostingClusterModeEnable  ReportHostingClusterMode = "Enable"
	ReportHostingClusterModeDisable ReportHostingClusterMode = "Disable" // the default
)

// KlusterletHostedConfiguration holds configuration for a mode whose controllers run on a
// separate management cluster rather than locally. Pre-existing type, shown here only for
// context - self-report reads its ManagementClusterName field but doesn't change its shape.
type KlusterletHostedConfiguration struct {
	// ManagementClusterName is the name (as known to the hub) of the cluster where this
	// klusterlet's controllers actually run, for a mode where that's a separate management
	// cluster rather than the klusterlet running locally.
	// +optional
	ManagementClusterName string `json:"managementClusterName,omitempty"`
}
```

`ReportHostingCluster` is the only field this proposal adds; `Hosted` and `ManagementClusterName`
already exist to support that mode's normal operation independent of self-report.

Self-report's own logic only ever branches on whether `Hosted` is set, the same signal the klusterlet
already uses to decide where its own controllers run. When unset, the controllers run locally, and
self-report publishes the klusterlet's own pre-existing `Klusterlet.spec.ClusterName` (a sibling of
`deployOption`, naming the cluster the klusterlet resource itself is for). When set, the controllers
run on the named separate management cluster, and self-report publishes
`Hosted.ManagementClusterName` instead. This proposal adds no new `Klusterlet.spec.deployOption.mode`
value, and self-report keeps working unchanged if more modes of either kind are added later, since it
never branches on `Mode` directly.
`hosting-cluster.open-cluster-management.io` joins
[4-cluster-claims](../4-cluster-claims/README.md)'s existing list of reserved claim names, which are
synced to the hub ahead of ordinary ones and aren't subject to the cap (20 by default) that mechanism
places on how many ordinary claims per cluster it will surface.

With `ReportHostingCluster` unset, a klusterlet makes zero calls of any kind against the `ClusterClaim`
API - the opt-in gates the call itself, not just what it writes. Once it's on, the klusterlet's
`ClusterRole` needs only `get`, `create`, `update`, and `delete` on `ClusterClaim`, and only the
reserved claim by name. `resourceNames` scopes `get`/`update`/`delete`; `create` can't be scoped by
name under Kubernetes RBAC, since the object doesn't exist yet at authorization time. No `list` or
`watch` is needed at all. Because RBAC can't be conditioned on a spec field, this grant lands on
every klusterlet's `ClusterRole` the moment its operator upgrades, whether or not any individual
klusterlet ever turns the field on - the opt-in only governs whether the permission is ever used.

Once on, the klusterlet keeps the claim in sync with whichever value is currently active - its own
`ClusterName` in Default mode, `ManagementClusterName` in Hosted mode - on every reconcile, and clears
it if that value is cleared. `Mode` itself isn't something this covers: it's already documented as set
once and never changed, a pre-existing constraint this proposal doesn't alter. Turning the opt-in back
off deliberately leaves the last-published claim in place rather than deleting it, keeping "off" a
strict no-op rather than "no calls except one last cleanup" - see Risks and Mitigations above.

### 2. Validation

Once a target's klusterlet self-reports, the framework can compare it against whatever
A-hosting-cluster an addon declares.

- No self-report available (feature off, older klusterlet, or the value hasn't propagated yet): the
  addon behaves exactly as it does today. Absence of the signal is never treated as an error.
- Self-report matches the declared value: same outcome as today, with a condition message that now
  says so explicitly.
- Self-report disagrees with the declared value: a new `HostingClusterMismatch` reason gets set on
  the addon's `HostingClusterValidity` condition. Whether the addon is held back or left running
  depends on whether its hosting manifests have already been created. This is tracked the same way an
  invalid hosting cluster already is today: the addon carries a hosting-manifest cleanup finalizer,
  added when addon-framework first creates the hosting manifests and removed once they're fully torn
  down. A never-deployed addon (finalizer not yet added) is held back, since there's nothing running
  yet to protect. An already-deployed addon (finalizer present) keeps running, always - nothing
  described anywhere in this proposal ever tears it down. None of this blocks an explicit delete
  request either - an addon already being deleted is never held back by a mismatch.

```golang
const (
	// HostingClusterValidityReasonMismatch is the reason of condition HostingClusterValidity indicating the
	// declared hosting cluster of the addon does not match the actual hosting cluster self-reported by the
	// target managed cluster's klusterlet.
	HostingClusterValidityReasonMismatch = "HostingClusterMismatch"

	// HostingClusterValidityReasonAutoDiscoveryPending is the reason of condition HostingClusterValidity
	// indicating install-mode is Hosted but no hosting cluster has been resolved yet.
	HostingClusterValidityReasonAutoDiscoveryPending = "HostingClusterAutoDiscoveryPending"
)
```

The second reason belongs to auto-discovery below, but it's the same condition type, so it's listed
here with the first.

### 3. Auto-discovery

An addon deployer can ask for a hosting cluster to be resolved automatically instead of naming one:

```golang
const (
	// InstallModeAnnotationKey signals that this addon should be installed in Hosted mode, with its
	// hosting cluster resolved automatically instead of set via HostingClusterNameAnnotationKey.
	// It should be set on ManagedClusterAddon resource only.
	InstallModeAnnotationKey = "addon.open-cluster-management.io/install-mode"
)
```

An annotation, not a spec field, for the same reason `hosting-cluster-name` itself already is one:
`ManagedClusterAddOnSpec` today holds only `configs`, and every other piece of per-instance Hosted-mode
configuration - the hosting cluster name, the install namespace - already lives in an annotation rather
than growing that spec. This keeps the new signal consistent with the mechanism it's extending instead
of introducing a second way to configure the same kind of thing.

Setting this to `Hosted` on a `ManagedClusterAddOn`, without also setting `hosting-cluster-name`,
means "deploy this in Hosted mode, wherever the target's klusterlet actually runs." This only takes
effect for an addon type whose `ClusterManagementAddOn` has separately opted in:

```golang
type ClusterManagementAddOnSpec struct {
	// ... existing fields ...

	// hostedModeAutoDiscovery, when its mode is set to Enable, lets ManagedClusterAddOns of this
	// addon type resolve their hosting cluster automatically instead of requiring
	// hosting-cluster-name to be set. Disabled by default.
	// +optional
	HostedModeAutoDiscovery *HostedModeAutoDiscoveryConfig `json:"hostedModeAutoDiscovery,omitempty"`
}

// HostedModeAutoDiscoveryConfig configures automatic hosting-cluster resolution for addon Hosted mode.
type HostedModeAutoDiscoveryConfig struct {
	// mode turns automatic hosting-cluster resolution on or off for ManagedClusterAddOns of this
	// type. Disabled by default.
	// +optional
	Mode HostedModeAutoDiscoveryModeType `json:"mode,omitempty"`
}

// HostedModeAutoDiscoveryModeType is the type for HostedModeAutoDiscoveryConfig.Mode. A typed
// enum, not a bool, so a later mode can be added without a breaking change to the field's shape.
// +kubebuilder:validation:Enum=Enable;Disable
type HostedModeAutoDiscoveryModeType string

const (
	HostedModeAutoDiscoveryModeEnable  HostedModeAutoDiscoveryModeType = "Enable"
	HostedModeAutoDiscoveryModeDisable HostedModeAutoDiscoveryModeType = "Disable" // the default
)
```

This is not a backward-compatibility artifact; it's a deliberate authority boundary. This is a real
spec field on `ClusterManagementAddOn`, separate from the per-instance annotation above: turning it on
is a hub-administrator action, gated by whatever RBAC already governs that type's
`ClusterManagementAddOn`, not something an individual addon deployer can do by setting an annotation
on their own `ManagedClusterAddOn`. The two keys are required together on purpose - one addon deployer
asking for auto-discovery on their own instance can't make it happen for an addon type the type's
owner hasn't separately vetted and opted in.

It's shaped as a struct with a single `mode` field today, rather than a bare enum value on the parent
spec, to leave room for future auto-discovery tuning without needing another top-level field on
`ClusterManagementAddOnSpec`.

When both are enabled - the addon type has `HostedModeAutoDiscovery.Mode: Enable` and the instance has
`install-mode: Hosted` - a resolver checks the target's self-reported claim and, if it finds one,
writes it as `hosting-cluster-name`, along with a marker so it can tell its own writes apart from a
human-set value:

```golang
const (
	// HostingClusterNameManagedByAnnotationKey marks a HostingClusterNameAnnotationKey value as
	// having been written by the auto-discovery resolver, distinguishing it from a human-set value.
	// It should be set on ManagedClusterAddon resource only.
	HostingClusterNameManagedByAnnotationKey = "addon.open-cluster-management.io/hosting-cluster-name-managed-by"

	// HostingClusterNameManagedByAutoDiscoveryValue is the value of HostingClusterNameManagedByAnnotationKey
	// set by the auto-discovery resolver.
	HostingClusterNameManagedByAutoDiscoveryValue = "auto-discovery"
)
```

The resolver only ever fills in `hosting-cluster-name` when it has no value at all. Once a value is
present - whether the resolver wrote it or a human did - that value itself is never revisited: the
marker plays no part in deciding whether or how `hosting-cluster-name` gets written. A self-report
changing after the fact, including one that reflects a genuine klusterlet migration, doesn't move an
already-resolved addon; validation (above) still compares the two and surfaces a mismatch if they now
disagree, but nothing here rewrites the declared value automatically (see Non-Goals for what to do
instead). The marker does drive one piece of cleanup: if `install-mode: Hosted` is later removed from
an addon that still carries the marker, both annotations are removed together, rather than leaving an
orphaned `hosting-cluster-name` with no explanation of how it got there.

`HostingClusterAutoDiscoveryPending` applies only while `hosting-cluster-name` has no value at all -
whether because the target has no self-report yet, or because one exists but fails the guardrail
below. It's reported instead of falling through to whatever an empty declared value means today, and
takes precedence over mismatch handling for as long as it applies, since there's no declared value
yet to compare or act on. It's specific to `install-mode: Hosted` on an addon whose type has
`HostedModeAutoDiscovery.Mode: Enable`: an addon whose type never enabled auto-discovery falls
through to that same pre-existing empty-value handling instead, since nothing will ever resolve it. A
`hosting-cluster-name` set directly - by a human, or left over from before auto-discovery was
enabled - is never `HostingClusterAutoDiscoveryPending`, regardless of auto-discovery being enabled
for the type; validation (above) governs it like any other declared value.

#### Guardrail: co-membership required

Auto-discovery only ever routes an addon to a cluster that shares a real `ManagedClusterSet` with the
addon's target - bounding how far a wrong or malicious self-report can reach to a grouping an
administrator already made. Mandatory, no per-addon-type override.

`global` and `default` don't count as a deliberate grouping: `global` matches every cluster
unconditionally, and `default` catches every cluster nobody's explicitly grouped yet. Sharing only one
of those (or both) is treated the same as sharing no set at all.

The check runs each time the resolver tries to fill in an unset `hosting-cluster-name`; a failing
check just leaves the addon pending for a later reconcile. Once a value is written, it's never checked
again for that addon (see Non-Goals).

### Architecture

```text
                              +--------------------------------------------+
                              |  hub                                       |
                              |                                            |
                              |  ManagedCluster (target)                   |
                              |    status.clusterClaims:                   |
                              |      hosting-cluster....: cluster-B <------|-----.
                              |                                            |     |
                              |  ClusterManagementAddOn (addon type)       |     | self-report
                              |    spec.hostedModeAutoDiscovery.mode       |     | syncs up
                              |                    |                       |     |
                              |                    v                       |     |
                              |             resolver controller            |     |
                              |                    |                       |     |
                              |                    v                       |     |
                              |  ManagedClusterAddOn (target's addon)      |     |
                              |    hosting-cluster-name: cluster-B         |     |
                              |    hosting-cluster-name-managed-by: ...    |     |
                              |                    |                       |     |
                              |                    v                       |     |
                              |              validation controller         |     |
                              |          compares annotation vs claim      |     |
                              |                    |                       |     |
                              |            HostingClusterValidity          |     |
                              +--------------------------------------------+     |
                                                                                 |
                              +---------------------------+   +------------------------+
                              |  cluster-B (hosting)      |   |  target cluster        |
                              |                           |   |                        |
                              |  klusterlet (Hosted mode) |   |  ClusterClaim:         |
                              |    reportHostingCluster:  |   |    hosting-cluster...  |
                              |      Enable          -----------> = cluster-B          |
                              |  addon agent deployment   |   |                        |
                              +---------------------------+   +------------------------+
```

The target cluster is where the klusterlet's CRDs and configuration live, and where its self-reported
`ClusterClaim` is visible. Cluster-B is where the klusterlet's controllers and the addon's agent
actually run. The self-report flows from cluster-B's klusterlet onto the target cluster as a
`ClusterClaim`, syncs up to the hub through the existing `ManagedCluster` status pipeline, and from
there both the resolver and validation read it.

### Persisting the new `ClusterManagementAddOn` field

`ClusterManagementAddOn`'s stored API version is older than the version this field is added to, the
same situation `ManagedClusterAddOn`'s existing `InstallNamespace` field already has (in the opposite
direction: present on its stored version, absent from a newer served one). The same preservation
approach applies here: the conversion path between the served and stored versions stashes the field
into a hub-internal annotation on the way down and restores it on the way back up, invisible to any
client that only ever asks for the served version. `Klusterlet`'s own new fields need none of this,
since `Klusterlet` has only ever had one API version.

### Test Plan

- Unit tests covering every branch of validation and the opt-in gate: no self-report, confirmed
  match, confirmed mismatch on both a never-deployed and an already-deployed addon,
  `HostingClusterAutoDiscoveryPending` both with and without a self-report already present and only
  when the addon type has actually opted in, and confirming zero `ClusterClaim` API calls of any kind
  while the opt-in is off.
- A test proving the klusterlet's `ClusterClaim` RBAC is scoped as intended, not just declared: the
  reserved claim name succeeds for `get`/`update`/`delete`, any other claim name fails, `create`
  succeeds unrestricted, and `list`/`watch` are refused outright.
- Unit and integration tests for the resolver: successful resolution, the `ManagedClusterSet`
  guardrail (`global` alone, `default` alone, and the two together, all leave it unsatisfied; a real
  set still resolves it even when both clusters also happen to share `global`), a self-report that
  later changes - including one that reflects a klusterlet migration - never moving an addon's
  already-resolved `hosting-cluster-name`, a human-set value never being overridden by a subsequent
  self-report either, and `install-mode: Hosted` being removed cleaning up `hosting-cluster-name`
  together with the marker for a resolver-written value, while leaving an explicit human value with
  no marker untouched.
- A conversion round-trip test proving `HostedModeAutoDiscovery` survives being written and read back
  through the older stored API version.
- End-to-end tests extending an existing hosted-mode addon example: a brand-new mismatched addon is
  blocked, a valid auto-discovery case resolves and reaches `Available: True`, a guardrail-failing
  case stays pending, and a mismatch on an already-deployed addon stays loud but running regardless
  of how long it persists.

### Graduation Criteria

#### Alpha

- Self-report, validation, and auto-discovery all implemented, each independently toggleable except
  validation, which has no toggle of its own and activates automatically for any addon the moment its
  target has a self-report to compare against.
- Unit and integration test coverage for every opt-in combination above.
- End-to-end coverage for at least one hosted-mode addon exercising validation and auto-discovery.

#### Beta

- At least one real hosted-mode addon, not one of the sample addons used for the Alpha end-to-end
  coverage above, adopts self-report and validation in production use.
- No changes needed to the shape of the opt-ins based on alpha feedback.

#### GA

- Multiple hosted-mode addon types run auto-discovery in production across at least one release, with
  no incidents attributable to the one-shot resolution design.

### Upgrade / Downgrade Strategy

**Upgrade:** every opt-in defaults to off. Upgrading `api`, `addon-framework`, and `ocm` changes
nothing in behavior for a klusterlet, addon, or `ClusterManagementAddOn` that doesn't explicitly set
one of the new fields or annotations. The one exception is RBAC: a klusterlet's `ClusterRole` gains
the `ClusterClaim` permissions described under Self-reported hosting cluster above unconditionally,
since RBAC can't be conditioned on a spec field - but that permission sits unused unless
`ReportHostingCluster` is actually turned on.

**Downgrade:** a klusterlet or addon with an opt-in already on, downgraded to a version before this
proposal, stops emitting or checking the new signal. The `ClusterClaim`, condition reasons, and
annotations are additive and don't change how anything else already behaves. Downgrading only part
of the stack - just the klusterlet, or just the hub side - doesn't introduce a new failure mode: the
downgraded side simply stops participating, which extends the same stale-claim gap already accepted
above for as long as that side stays behind, rather than introducing anything new.

## Implementation History

- 2026-08-11: Initial proposal.

## Alternatives

**A typed `ManagedCluster` status field instead of a `ClusterClaim`.** There's real precedent for
this kind of self-reported fact living on a typed field. A reserved `ClusterClaim` is a smaller
commitment on an already-alpha, already-mutable mechanism than a new field on a stable, GA-level
type.

**Hub-admin-attested binding at accept time, instead of any spoke self-report.** More secure: the hub
decides the hosting-cluster relationship itself instead of trusting a claim from the spoke. It needs
new accept-time UX and doesn't reuse anything that exists today. Worth a second look if the
`ManagedClusterSet` guardrail is ever found insufficient.

**Admission-time validation instead of a reconciler.** A validating admission policy can't express
"look up the value self-reported by the resource named in this object's own field," and can't react
to the target's self-report changing after the addon was validly created - which is exactly how an
addon that looked fine at creation time later surfaces a real mismatch. This is a reconcile-shaped
problem.

**Watching for the addon's Lease to appear, instead of trusting a self-report.** Fails for a reason
specific to the shape of the bug: the mismatch that breaks the Lease check is the same mismatch that
would need detecting. There's no hub-observable signal here that isn't itself downstream of the same
broken assumption.

**Validation only, no auto-discovery.** An earlier, narrower framing of this proposal. Leaves an
addon deployer responsible for typing the correct hosting cluster by hand for every hosted-mode addon
they stand up. Doesn't answer the actual ask in the issue.

**Promoting `ClusterManagementAddOn`'s stored API version instead of the field-preservation approach
above.** Worth pursuing on its own, since it would let every new field on the type persist without
preservation logic going forward. It's a bigger, separate undertaking than it looks: a real version
promotion needs existing objects migrated to the new stored encoding, not just future writes using
it. Out of scope here; a candidate for its own proposal covering the addon API group as a whole.

**Destructively cleaning up a mismatch, and automatically relocating an addon when
`hosting-cluster-name` changes.** Both considered and left out: they only matter once an
already-resolved addon needs to move, which this proposal doesn't take on. Worth revisiting if a
future proposal wants to handle post-resolution changes.

## Infrastructure Needed

- No new repositories or CI infrastructure. Changes land in `api`, `addon-framework`, and `ocm`, in
  that order, since each of the latter two vendors the one(s) before it.

## References

- Issue: https://github.com/open-cluster-management-io/enhancements/issues/188
- [33-hosted-deploy-mode](../33-hosted-deploy-mode/README.md) — klusterlet Hosted mode
- [63-hosted-addon](../63-hosted-addon/README.md) — addon Hosted mode
- [4-cluster-claims](../4-cluster-claims/README.md) — the `ClusterClaim` mechanism self-report builds on
