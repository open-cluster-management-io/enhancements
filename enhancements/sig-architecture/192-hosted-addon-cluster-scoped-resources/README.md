# Safely manage cluster-scoped resources for hosted addons

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
[website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

A hosting cluster can run one addon instance for each of many managed clusters. Namespaces can
isolate the namespaced resources of those instances, but they cannot isolate a `ClusterRole`,
`ClusterRoleBinding`, `CustomResourceDefinition`, or any other cluster-scoped object. Two hosted
instances that render the same cluster-scoped name currently address the same object. One instance
can consequently change or delete an object needed by another instance without either addon
declaring that relationship.

This proposal will define two explicit ownership models. A resource that is identical for every
instance will be installed once by a prerequisite addon running in Default mode on the hosting
cluster. A resource that belongs to one hosted instance will have a deterministic, instance-specific
name and will be acquired through a protected claim phase before any namespaced runtime resources
are delivered.

The claim phase will never adopt or change a pre-existing resource unless its live ownership
annotation already identifies the same `ManagedClusterAddOn`. A new resource will first be created
with `CreateOnly` and protected with `SelectivelyOrphan`. After a live CEL condition verifies its
owner, it will be promoted to non-forcing server-side apply under a field manager unique to the addon
instance. A conflict will be reported on the `ManagedClusterAddOn` before a foreign resource is
mutated.

The complete behavior will be optional. Existing addons and existing hosted deployments will keep
their current behavior. An addon type will select the feature before its addon instances are
created, and that selection and the related instance configuration will be immutable for the life of
those instances. No migration will be required.

### Terms

- **Target managed cluster**: the managed cluster represented by the namespace containing a
  `ManagedClusterAddOn`.
- **Hosting cluster**: the managed cluster on which a Hosted-mode addon agent runs.
- **Hosted addon instance**: one `ManagedClusterAddOn`, identified by its namespace and name.
- **Shared prerequisite resource**: a cluster-scoped object whose desired content is identical for
  every hosted instance on a hosting cluster.
- **Per-instance resource**: a cluster-scoped object whose lifecycle or content belongs to one
  hosted addon instance.
- **Claim Work**: a `ManifestWork` that contains hosting-bound cluster-scoped objects and performs
  ownership acquisition and verification.
- **Runtime Work**: a `ManifestWork` that contains the namespaced resources that run the hosted addon
  agent.

## Motivation

[Issue #192](https://github.com/open-cluster-management-io/enhancements/issues/192) asks for a
defined way for multiple hosted addon instances to install, update, and remove cluster-scoped
resources safely. It also requires incompatible requirements to be detected before an existing
resource is changed and reported on the affected `ManagedClusterAddOn`.

Kubernetes identifies a namespaced object by group, resource, namespace, and name. It identifies a
cluster-scoped object by group, resource, and name. A per-instance hosting namespace therefore does
not help when two instances both render a fixed object such as:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: example-addon-agent
subjects:
- kind: ServiceAccount
  name: example-addon-agent
  namespace: one-instance-runtime-namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: example-addon-agent
```

The second instance would address the same binding name but require a different subject namespace.
Normal update behavior could replace the first subject, and normal Work deletion could remove the
binding while another instance still needs it.

The opposite case also exists. A `ClusterRole` or CRD can be byte-for-byte identical for every
hosted instance. Duplicating it under instance-specific names is unnecessary or, for a CRD, not
possible. Letting every instance manage the same fixed name creates multiple owners with independent
upgrade and deletion lifecycles.

These two cases need different solutions. Shared objects need one stable owner. Per-instance objects
need unique identities, exclusive ownership, and an acquisition protocol that does not mutate a
foreign object while deciding whether it can be managed.

### Goals

- Define the supported ownership model for both shared and per-instance cluster-scoped resources on
  a hosting cluster.
- Give a per-instance cluster-scoped resource a deterministic name derived from the complete
  `ManagedClusterAddOn` identity.
- Require an explicit addon-level opt-in before protected cluster-scoped handling is used.
- Preserve all existing defaults and behavior when the opt-in is absent.
- Detect an unannotated or differently owned pre-existing object before changing it.
- Report acquisition and server-side apply conflicts through a clear
  `ManagedClusterAddOn` condition.
- Prevent namespaced runtime resources from being created until their cluster-scoped prerequisites
  have been safely acquired.
- Preserve an already-running runtime while a later claim is pending or conflicting.
- Define safe deletion and controller-recovery behavior for every acquisition state.
- Identify cluster-scoped manifests without guessing a custom resource plural.

### Non-Goals

- Automatically determining whether two independently rendered shared resources are semantically
  compatible.
- Automatically creating, placing, or ordering prerequisite addons.
- Migrating an existing addon instance from legacy behavior to the protected behavior.
- Changing the selected behavior or hosting configuration after an addon instance is created.
- Sharing a per-instance resource between two `ManagedClusterAddOn` objects.
- Providing a general locking protocol for `ManifestWork` resources outside hosted addons.
- Replacing Kubernetes RBAC, server-side apply ownership, or `ManifestWork` deletion semantics.
- Solving namespaced resource isolation. That is a separate concern; this proposal only relies on a
  hosting runtime namespace when one has been selected for the addon type.

## Proposal

An addon author will classify each cluster-scoped dependency as either shared or per-instance.

Shared resources will be omitted from every hosted instance's manifests. They will instead be
delivered once to the hosting cluster by an ordinary addon using Default mode. The hosted instances
will only refer to those objects. For example, a prerequisite addon will own one stable
`ClusterRole`, and each hosted instance will own a uniquely named `ClusterRoleBinding` that refers to
that role.

Per-instance resources will use a deterministic instance identifier in their names. The addon will
explicitly enable protected cluster-scoped handling. The addon framework will separate
cluster-scoped claims from namespaced runtime manifests, acquire every claim without changing any
pre-existing object, verify live ownership, and only then deliver the runtime.

### User Stories

#### Addon author with shared RBAC or APIs

I want to install one shared `ClusterRole` or CRD on a hosting cluster and let every hosted instance
refer to it, so instances cannot race to update or delete the shared object.

#### Addon author with instance-specific RBAC

I want each hosted instance's `ClusterRoleBinding` to have a stable unique name and exclusive owner,
so adding or removing one managed cluster cannot change another instance's access.

#### Hub administrator encountering an existing object

I want an addon to report a conflict before it changes an unannotated or differently owned
cluster-scoped object, even when that object's content happens to look compatible.

#### Operator not selecting the feature

I want addon behavior to remain unchanged after an upgrade unless the addon type explicitly selects
the protected cluster-scoped resource contract for newly created instances.

### Risks and Mitigations

**A deterministic name could collide.** The instance identifier will be derived from a domain-
separated SHA-256 digest of the `ManagedClusterAddOn` namespace and name. A shortened digest suitable
for Kubernetes names will be exposed to templates. The complete digest will be used in the
server-side apply field manager. The ownership annotation will still contain the readable, complete
`<target-managed-cluster>/<addon-name>` identity, so ownership will not be inferred from the suffix
alone.

**An identical pre-existing object could be rejected.** This is deliberate. Comparing content does
not prove ownership and leaves a race between comparison and mutation. `CreateOnly` plus live
annotation verification will reject both an unannotated resource and a resource annotated for
another instance. The framework will not automatically adopt either one.

**Claim reconciliation adds time before first deployment.** The runtime will wait for a round trip in
which the work agent creates or observes each claim and reports the live ownership condition. This
delay is bounded by normal `ManifestWork` reconciliation and is preferable to deploying a runtime
whose RBAC or other cluster dependency is unsafe.

**A claim can become conflicting after the runtime has started.** Non-forcing server-side apply will
surface field ownership conflicts. The controller will leave existing runtime Works in place while
reporting the conflict. It will not roll back or delete a running instance merely because a later
claim revision cannot be applied.

**An incorrect resource identifier could make deletion unsafe.** The framework will use an explicit
authoritative mapping for the supported built-in kinds. Every other empty-namespace hosting manifest
will require one exact `ManifestConfig` declaration with a non-wildcard resource. Reconciliation will
fail before creating a Work when that declaration is absent or ambiguous.

**A deletion can begin between acquisition and promotion.** A claim created under `CreateOnly` is
temporarily orphaned, so deleting its Work immediately could leak it. Cleanup will wait for a
current-generation live ownership result. A claim verified as owned will be promoted to normal Work
ownership before Work deletion; a foreign claim will remain orphaned. A terminating Work that can no
longer be updated will use a bounded recovery Work to obtain the same live observation.

**The ownership annotation can be forged by a cluster administrator.** The annotation is an
ownership coordination signal, not a security boundary against users with cluster-wide write
permission. Access to create or modify claim resources, `ManifestWork`, and
`AppliedManifestWork` must remain restricted to trusted controllers and administrators.

## Design Details

### Resource ownership model

#### Shared resources

A cluster-scoped resource will be treated as shared when every hosted instance needs the same name
and desired content. Common examples include:

- `CustomResourceDefinition`
- a `ClusterRole` containing common rules
- admission webhook configurations
- storage or scheduling policy that is intentionally common to the hosting cluster

The resource will be packaged in a prerequisite addon that runs in Default mode on the hosting
cluster. Placement and rollout of that prerequisite will use the existing
`ClusterManagementAddOn` installation model. Hosted instances will not include the shared manifest
in either their claim or runtime Works.

No new dependency scheduler will be introduced. The addon operator will ensure that the prerequisite
addon is available before creating dependent hosted instances. The prerequisite addon will retain
one owner and one upgrade lifecycle regardless of how many hosted instances use it.

#### Per-instance resources

A resource will be per-instance when its content or lifecycle belongs to one target managed cluster.
A `ClusterRoleBinding` whose subject is the runtime ServiceAccount is the primary example.

On the protected path, the framework will expose `HostedInstanceID` as a built-in template value.
Helm charts will receive the value as `.Values.hostedInstanceID`, and Go templates will receive it as
`.HostedInstanceID`. The value will not be injected when the option is false, preserving any existing
user-supplied value with the same name. It will be stable for the pair:

```text
ManagedClusterAddOn namespace + ManagedClusterAddOn name
```

The value will be a 20-character hexadecimal prefix of a domain-separated SHA-256 digest. Addon
authors will append it to a human-readable prefix while respecting the target resource's name limit:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: example-addon-agent-{{ .HostedInstanceID }}
```

The instance identifier is a naming primitive, not an ownership proof. The claim protocol below will
provide ownership proof.

### Explicit opt-in and immutability

The addon framework agent options will gain an additive field:

```go
type AgentAddonOptions struct {
    HostedModeEnabled                         bool
    HostedClusterScopedResourcesEnabled       bool
}
```

An addon factory will expose the corresponding
`WithHostedClusterScopedResourcesEnabledOption()` builder method. The protected path will be used
only when all of the following are true:

1. the addon type enables Hosted mode;
2. the addon type enables `HostedClusterScopedResourcesEnabled`; and
3. the `ManagedClusterAddOn` is created in Hosted mode.

The option will default to `false`. With the option unset, manifest grouping, update strategy,
status, and deletion behavior will remain exactly as they are today.

The option is a contract selected by the addon type before instances are created. The install mode,
hosting cluster selection, hosting namespace selection when used, and cluster-scoped resource option
will be treated as immutable for the lifetime of an opted-in instance. Converting an existing
instance between legacy and protected behavior, or changing any of those selections after creation,
will be outside the scope of this proposal.

### Identifying cluster-scoped manifests

The addon manager does not have discovery for every hosting cluster at render time. It must not
derive a resource plural by lowercasing a Kind because an incorrect plural would cause the
`ManifestConfig` and deletion rule to miss the actual object.

A hosting-bound manifest with an empty `metadata.namespace` will enter cluster-scoped validation.
The initial authoritative built-in mapping will contain exactly these GVK to GVR entries:

| Group/version/kind | Authoritative group/version/resource |
|---|---|
| `v1, Kind=Namespace` | core group, `v1`, `namespaces` |
| `rbac.authorization.k8s.io/v1, Kind=ClusterRole` | `rbac.authorization.k8s.io`, `v1`, `clusterroles` |
| `rbac.authorization.k8s.io/v1, Kind=ClusterRoleBinding` | `rbac.authorization.k8s.io`, `v1`, `clusterrolebindings` |

`ManifestWork.spec.manifestConfigs.resourceIdentifier` records group and resource but not version.
The framework will resolve the exact GVR first, then write its group and resource into that
identifier.

Every other GVK, including a custom cluster-scoped kind, will require exactly one addon-supplied
`ManifestConfigOption`. Its `ResourceIdentifier` must have:

- `group` equal to the rendered object's API group;
- `resource` set to the authoritative plural and neither empty nor `*`;
- `name` equal to the rendered object's exact name; and
- `namespace` equal to the rendered object's empty namespace.

For example:

```go
ManifestConfigs: []workv1.ManifestConfigOption{
    {
        ResourceIdentifier: workv1.ResourceIdentifier{
            Group:     "policy.example.io",
            Resource:  "clusterpolicies",
            Name:      "example-addon-policy-" + hostedInstanceID,
            Namespace: "",
        },
    },
}
```

An addon that renders a custom per-instance name will construct this exact declaration from the same
deterministic instance identity used by the manifest renderer. Wildcards will not be accepted for
claim or deletion decisions. Zero matches or multiple matches will stop reconciliation and report a
configuration error before any claim Work is created.

The framework will control the update strategy, ownership condition, and temporary deletion rule for
claim manifests. Other compatible feedback requested by the addon may be merged, but an addon-
supplied update strategy will not weaken the claim protocol.

### Claim and runtime Works

Hosted manifests will be partitioned by destination and scope:

```text
                         rendered hosted manifests
                                  |
                 +----------------+----------------+
                 |                                 |
       empty metadata.namespace          non-empty metadata.namespace
                 |                                 |
            claim Works                        runtime Works
                 |                                 |
     acquire -> verify -> promote        held until every claim is ready
```

Claim Works will contain only hosting-bound cluster-scoped resources. Runtime Works will contain only
hosting-bound namespaced resources. Work names and labels will continue to identify the target
`ManagedClusterAddOn`, and claim Work names will use a distinct deterministic prefix so the two
phases cannot overwrite each other.

The addon manager will persist the hosted manifest cleanup finalizer before creating the first claim
Work. It will not remove that finalizer merely because a Work delete request has been issued; every
claim and runtime Work protected by the finalizer must actually disappear first. This closes the
deletion race between initial acquisition and lifecycle protection.

If the addon uses an instance-specific hosting namespace, the generated `Namespace` will itself be a
cluster-scoped claim. Namespaced Deployments, ServiceAccounts, Roles, RoleBindings, Secrets, and
hooks will use that hosting namespace and will remain in runtime Works.

### Ownership identity

Before a claim manifest is sent to a hosting cluster, the framework will set:

```yaml
metadata:
  annotations:
    addon.open-cluster-management.io/hosting-addon: "<target-managed-cluster>/<addon-name>"
```

The namespace of a `ManagedClusterAddOn` is the target managed cluster name, so the annotation value
identifies the complete addon instance. The framework will set this key when it is absent or already
matches. A nonempty chart-supplied value for another instance will be rejected before a Work is
created. An annotation matching another instance, or no annotation on the live object, will mean
that the live object is foreign.

### Two-phase claim acquisition

Each claim will move through the following states. State decisions will be based on the live object
feedback for the current Work generation, not on the desired manifest alone.

#### 1. Acquire without adoption

Until ownership is proven, the claim's `ManifestConfigOption` will use:

```yaml
updateStrategy:
  type: CreateOnly
```

The Work will also use `SelectivelyOrphan` for every acquiring resource. Together these settings
will provide these outcomes:

- If the object is absent, the work agent may create the desired object with the expected ownership
  annotation.
- If the object already exists, `CreateOnly` will not update its annotation, fields, or spec.
- A pre-existing object will not be attached to the claim Work merely because its GVR and name
  match.
- Deleting the Work while ownership remains foreign will not delete that object.

The temporary orphan rule is part of acquisition safety, not the addon's requested steady-state
deletion policy.

#### 2. Verify the live owner

The claim config will add a CEL condition named `HostingResourceOwned`. Its expression will require
the live object's annotation to equal the expected instance identity. A JSON path feedback rule will
also return the live annotation value for diagnostics.

The controller will only accept an ownership result when all of these are current:

- the claim Work has `Applied=True`;
- the Work condition observes the Work's current generation;
- the manifest has a `HostingResourceOwned` condition for the current generation; and
- that condition is `True`.

An ownership condition of `False` will be treated as a conflict even if the Work's aggregate
`Applied` condition is `True`. The controller will set:

```yaml
type: HostingManifestApplied
status: "False"
reason: HostingResourceConflict
```

The message will identify the claim Work, resource kind and name, expected owner, and observed owner
or missing annotation. No runtime Work will be created for a new instance in this state.

#### 3. Promote verified ownership

After the live ownership condition is `True`, the claim will be promoted to:

```yaml
updateStrategy:
  type: ServerSideApply
  serverSideApply:
    force: false
    fieldManager: <deterministic full-digest manager for this addon instance>
```

The field manager will be stable and unique to the `ManagedClusterAddOn` namespace and name. `force`
will always be `false`; the framework will never take fields from another manager to make a claim
succeed.

The temporary acquisition orphan rule will be removed for the verified object so normal Work
ownership can manage its lifecycle. An intentional addon deletion policy that already requires the
resource to be orphaned will remain in place.

The controller will wait until the current Work generation reports `Applied=True` and the live Work
contains the exact non-forcing strategy, expected field manager, and expected orphan state. It will
compare only these controlled claim fields rather than the entire desired and stored Work, because
API defaulting may add unrelated fields.

### Runtime gating and steady-state conflicts

Runtime Works for a new instance will be created only after every desired claim Work has completed
promotion for its current generation. A pending ownership observation, an unrecognized custom GVR,
a foreign owner, an apply failure, or an SSA conflict will keep the initial runtime gated.

If an instance already has runtime Works and a later desired claim revision becomes pending or
conflicting, those existing runtime Works will remain unchanged. The controller will not create the
new runtime revision, but it will also not delete a working runtime as a side effect. The
`HostingManifestApplied` condition will expose the blocked claim until it is resolved.

Non-forcing SSA conflicts will use the same `HostingResourceConflict` reason and will preserve the
manifest-level Work error in the condition message. Other claim application failures will retain the
general manifest-apply failure reason.

### Deletion and recovery

Claim cleanup must distinguish an object created for this instance from a pre-existing foreign
object. A Work spec alone is insufficient because deletion can begin before acquisition has been
promoted.

Before deleting a claim Work, the controller will prepare and observe a safe deletion shape for each
manifest:

1. If current-generation live feedback says `HostingResourceOwned=True`, the controller will ensure
   that the object has been promoted to non-forcing SSA and normal Work ownership. Normal Work
   cleanup may then delete it, unless the addon's intentional deletion policy says to orphan it.
2. If current-generation live feedback says `HostingResourceOwned=False`, the controller will keep
   `CreateOnly` and `SelectivelyOrphan`. Deleting the Work will leave the foreign object untouched.
3. If no current ownership result exists, the controller will keep the Work and wait. It will not
   guess based on the desired annotation or an older generation.

This ordering will cover the race in which the work agent creates an absent object during
acquisition, then the `ManagedClusterAddOn` is deleted before the next promotion reconcile. The live
condition will identify the newly created object as owned, cleanup will promote it, and ordinary Work
deletion will remove it rather than leak it.

A Work that already has a deletion timestamp cannot be updated into the safe shape. In that case the
controller will create one deterministic recovery Work containing the same claims under
`CreateOnly` and `SelectivelyOrphan`. The recovery Work will re-observe live ownership and drive the
same three-way decision. A recovery Work will never create another recovery Work, so controller
restarts or repeated reconciles cannot form an unbounded chain.

Stale claim Works, addon deletion, and hosting-cluster cleanup will all use this same deletion state
machine. Runtime Works and claim Works will both be included in cleanup, but a foreign claim will
never be deleted with the Work.

If the hosting `ManagedCluster` is missing, no new live feedback can arrive. In that terminal cleanup
case, every claim whose ownership is not already proven will be persisted as `CreateOnly` with an
exact `SelectivelyOrphan` rule. The controller will verify that safe shape on the hub-side Work before
requesting deletion. The remote object may remain if the hosting cluster returns, but it will not be
deleted based on an ownership guess. Generic ManifestWork cleanup finalizers will not be
force-removed; a permanently unavailable work agent can therefore leave a terminating Work that
requires administrator inspection.

### Status

The existing `HostingManifestApplied` condition on `ManagedClusterAddOn` will summarize both phases.
The proposed reasons will be:

| Reason | Meaning |
|---|---|
| `HostingResourceClaimsPending` | A claim Work, current-generation ownership result, or promotion is still pending. |
| `HostingResourceConflict` | A live object is unannotated, belongs to another instance, or non-forcing SSA reported a field conflict. |
| existing manifest apply failure reason | A claim or runtime Work failed for a reason other than ownership conflict. |

The conflict message will include the Work and manifest identity and will preserve the underlying
work-agent message. Detailed per-manifest conditions and feedback will remain available on the claim
`ManifestWork` for diagnosis.

`HostingManifestApplied=True` will require the claim phase to be ready and the current runtime Works
to be applied. A condition from an old Work generation will never satisfy either phase.

### Compatibility and migration

This proposal will be fully additive and optional:

- `HostedClusterScopedResourcesEnabled` will default to `false`.
- Existing addon types will not select it automatically.
- Existing `ManagedClusterAddOn` objects will not be reclassified, renamed, split into claim Works,
  annotated, or otherwise migrated.
- Existing manifest rendering, Work names, update strategies, deletion behavior, and conditions will
  remain unchanged when the option is false.
- No stored object conversion or data migration will be required.
- The option and the Hosted-mode placement configuration selected for a new opted-in instance will
  be immutable for that instance.

The protected behavior will therefore apply only to addon types and newly created instances that
select the complete contract from the beginning. Changing an existing instance between modes will
not be supported by this proposal.

### Security

Cluster-scoped resources can grant broad access or change cluster-wide behavior. The following
properties will limit the authority introduced by this proposal:

- A hosted instance will not manage a shared resource. A separately placed prerequisite addon will
  have the sole lifecycle for that resource.
- Acquisition will use `CreateOnly`, so checking ownership cannot mutate a pre-existing object.
- Ownership will be verified from the live object for the current generation.
- The framework will stamp the expected owner instead of trusting a chart-supplied value.
- Steady-state updates will use non-forcing SSA with one field manager per addon instance.
- Exact GVR and name declarations will be required when no authoritative built-in mapping exists.
- A foreign object will remain selectively orphaned during cleanup.
- Runtime ServiceAccounts will receive only the shared or per-instance RBAC explicitly bound to
  them; selecting this feature will not grant an agent general cluster-scoped write access.

The ownership annotation will not protect against a principal that can directly edit the claimed
cluster-scoped object. Such a principal is already inside the cluster-administrator trust boundary.
Audit policy should cover writes to the protected cluster-scoped kinds and to `ManifestWork` and
`AppliedManifestWork` resources.

### Test Plan

Unit coverage will include:

- default-off behavior that produces the same Works and status as the legacy path;
- stable and distinct instance identifiers for different namespace/name pairs;
- the exact built-in GVK to GVR mapping;
- rejection of missing, wildcard, mismatched, and ambiguous custom `ManifestConfig` declarations;
- framework stamping of the expected ownership annotation;
- `CreateOnly` plus temporary `SelectivelyOrphan` for unverified claims;
- current-generation CEL ownership results for missing, matching, and foreign annotations;
- a custom ownership condition of `False` mapping to `HostingResourceConflict` even when the Work is
  otherwise applied;
- promotion to non-forcing SSA with the exact per-instance field manager;
- gating on explicit claim fields without whole-Work semantic equality;
- initial runtime gating and preservation of already-running runtime Works;
- cleanup of an owned claim, orphaning of a foreign claim, waiting for an unknown claim, and
  preservation of an intentional orphan policy; and
- bounded recovery when deletion starts before ownership is observed.

Integration coverage will use two target managed clusters that share one hosting cluster. It will
verify that one prerequisite addon owns a shared `ClusterRole`, while two hosted instances create
different `ClusterRoleBinding` names and subjects. It will also cover an unannotated pre-existing
binding, a binding owned by another instance, an SSA field conflict, a controller restart during
acquisition, and deletion immediately after a new claim is created.

End-to-end coverage will create two new opted-in hosted addon instances on one hosting cluster. Both
agents will reach availability with independent RBAC. Deleting either instance will remove only its
per-instance claim and runtime while leaving the other instance and the shared prerequisite
resource available. A separate conflict case will prove that a foreign fixed-name object remains
byte-for-byte unchanged and that the affected `ManagedClusterAddOn` reports
`HostingResourceConflict` before any runtime is delivered.

### Graduation Criteria

#### Alpha

- The opt-in, deterministic identity, resource classification, claim protocol, runtime gate, status,
  and deletion recovery contracts are available together.
- Unit and integration coverage exercises every ownership and deletion state.
- One sample hosted addon demonstrates a shared prerequisite `ClusterRole` and per-instance
  `ClusterRoleBinding`.
- Default-off compatibility is covered explicitly.

#### Beta

- At least one production hosted addon adopts the opt-in for newly created instances.
- The addon runs multiple instances on one hosting cluster across a full release without a
  cluster-scoped resource collision.
- Operational documentation covers prerequisite placement, conflict diagnosis, and supported
  version skew.
- No migration is required for addon types that do not select the feature.

#### GA

- Multiple hosted addon types use the contract in production across at least one release.
- The ownership annotation, identity algorithm, status reason, and deletion protocol require no
  incompatible changes based on Beta experience.
- Upgrade and rollback documentation is complete for both opted-in and default-off deployments.

### Upgrade / Downgrade Strategy

**Upgrade:** The new option will default to off. Upgrading the API and addon framework will not
change any existing addon instance and will not require a migration. Existing resources will not be
renamed or annotated, and existing Works will not be split. New addon instances will use the
protected path only when their addon type has selected it before creation and all required
components support the contract.

The work agent on a hosting cluster must support `CreateOnly`, `SelectivelyOrphan`, CEL condition
feedback, non-forcing server-side apply, and explicit field managers before a new opted-in instance
is created there. If those capabilities are unavailable, the claim phase will remain not ready and
the runtime will not be delivered.

**Downgrade:** Deployments that have not selected the option will be safe to downgrade because their
behavior and stored objects are unchanged. An active opted-in instance will require controller and
work-agent versions that understand the claim protocol for its complete lifetime. Downgrading those
components below that capability will not be a supported configuration. The KEP will not define an
automatic conversion of an opted-in instance to legacy behavior.

## Implementation History

- 2026-09-14: Initial proposal.

## Alternatives

**Give every cluster-scoped resource a unique name.** This works for per-instance bindings and some
policies, but it unnecessarily duplicates identical RBAC and cannot work for resources such as a CRD
whose name is fixed by API identity. It also gives every instance an independent copy to upgrade.

**Let every hosted instance manage shared fixed names.** This retains multiple independent owners
and deletion lifecycles for one object. Non-forcing SSA can expose field conflicts, but it cannot make
deleting any one owner's Work safe for all other owners.

**Adopt a pre-existing object when its content is identical.** Equality is not ownership. Defaulting,
admission, and mutable fields make complete comparison difficult, and another actor can change the
object after comparison. This proposal will require a live matching owner annotation instead.

**Use forced server-side apply.** Forced apply would resolve a conflict by taking fields from another
manager, which violates the requirement to detect incompatibility before changing the existing
resource.

**Use one Work and rely on manifest ordering.** `ManifestWork` does not provide the ownership barrier
needed between cluster-scoped claims and namespaced runtime objects. Splitting the phases gives the
controller an explicit point at which all live ownership results must be current.

**Use only Kubernetes owner references.** An owner reference written while observing an existing
object would itself be an adoption and mutation. It also would not distinguish an object created
during acquisition from a foreign object when deletion races with reconciliation. The proposed
annotation, CreateOnly strategy, live condition, and promotion step make that decision explicit.

**Discover every custom GVR from the hosting cluster.** The hub-side addon manager does not have a
discovery client for every hosting cluster at render time, and discovery would not remove the need
for an exact identifier in update and deletion rules. Requiring an exact declaration is deterministic
and fails closed.

**Reject every cluster-scoped manifest from hosted addons.** This is safe but prevents legitimate
per-instance resources such as a uniquely named `ClusterRoleBinding`. The opt-in claim protocol
allows that use case without making protected behavior a default.

## Infrastructure Needed

- No new repository or CI infrastructure will be needed.
- The addon API will define the ownership annotation and conflict reason used in status.
- The shared addon SDK will define the deterministic instance identifier and field manager helpers.
- The addon framework will provide the opt-in, built-in template value, claim/runtime orchestration,
  validation, status, and cleanup behavior.
- The existing `ManifestWork` API capabilities will be reused without a new Work API version.

## References

- Issue: https://github.com/open-cluster-management-io/enhancements/issues/192
- [33-hosted-deploy-mode](../33-hosted-deploy-mode/README.md) - klusterlet Hosted mode
- [47-manifestwork-updatestrategy](../47-manifestwork-updatestrategy/README.md) - CreateOnly and server-side apply
- [63-hosted-addon](../63-hosted-addon/README.md) - addon Hosted mode
- [29-manifestwork-status-feedback](../29-manifestwork-status-feedback/README.md) - manifest feedback
