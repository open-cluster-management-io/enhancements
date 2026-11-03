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
name and will be managed by a claim Work before any namespaced runtime resources are delivered.

For each claim, the work agent will create an absent resource, reconcile a resource owned by the
same addon instance, or report a conflict without changing a foreign resource. The addon will
control the fields declared in its manifest through server-side apply with `force: true`. Other
controllers may manage fields outside that declaration under Kubernetes' normal merge rules.
Deletion will use the same live ownership check and will never create a missing resource. This
requires additional Work API and work-agent support for ownership checks during apply and cleanup.

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
- **Claim Work**: a `ManifestWork` that manages hosting-bound cluster-scoped objects with an expected
  addon owner.
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
need unique identities and an ownership check before the work agent creates, updates, or removes
their resources.

### Goals

- Define the supported ownership model for both shared and per-instance cluster-scoped resources on
  a hosting cluster.
- Give a per-instance cluster-scoped resource a deterministic name derived from the complete
  `ManagedClusterAddOn` identity.
- Require an explicit addon-level opt-in before protected cluster-scoped handling is used.
- Preserve all existing defaults and behavior when the opt-in is absent.
- Detect an unannotated or differently owned pre-existing object before changing it.
- Report foreign owners and apply failures through a clear `ManagedClusterAddOn` condition.
- Prevent namespaced runtime resources from being created until their cluster-scoped prerequisites
  have been safely acquired.
- Preserve an already-running runtime while a later claim is pending or conflicting.
- Make reconciliation and deletion safe to repeat after a controller restart or failed request.
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
cluster-scoped claims from namespaced runtime manifests and wait until the work agent has safely
applied every claim before delivering the runtime.

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

**An identical pre-existing object could be rejected.** Comparing content does not prove ownership.
An unannotated resource or one annotated for another instance will be left unchanged, even if its
content matches the manifest.

**Claim reconciliation adds time before first deployment.** The runtime will wait for a round trip in
which the work agent applies each claim and reports its result. This delay is bounded by normal
`ManifestWork` reconciliation and keeps a runtime from starting before its prerequisites are ready.

**Another controller can edit a field declared by the addon.** The next reconcile will restore the
addon's desired value. Authors must omit fields they expect another controller to manage. Kubernetes
schema merge rules determine the ownership unit; an atomic list or map is managed as a whole.

**An object can change between a read and a write.** The work agent will check the live owner before
each mutation and use the UID and resourceVersion from that read to protect the write. A conflict
requires a fresh read and another ownership check. These values stay local to the agent.

**An incorrect resource identifier could make deletion unsafe.** The framework will use an explicit
authoritative mapping for the supported built-in kinds. Every other empty-namespace hosting manifest
will require one exact `ManifestConfig` declaration with a non-wildcard resource. Reconciliation will
fail before creating a Work when that declaration is absent or ambiguous.

**An addon can be deleted before its claims are applied.** The work agent's finalizer will inspect
the resources already tracked by the Work and check their live ownership. Cleanup will not require
the desired spec to apply successfully. Missing resources are already cleaned up, and foreign
resources will be preserved. A controller restart will repeat the same checks.

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

The instance identifier is a naming primitive. The work agent will check the live owner separately.

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

The framework will set the ownership declaration and update strategy for claim manifests. Compatible
feedback and an intentional orphan policy may be merged from the addon, but addon-supplied settings
will not bypass ownership checks.

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
       check owner and reconcile         held until every claim is applied
```

Claim Works will contain only hosting-bound cluster-scoped resources. Runtime Works will contain only
hosting-bound namespaced resources. Work names and labels will continue to identify the target
`ManagedClusterAddOn`, and claim Work names will use a distinct deterministic prefix so the two
phases cannot overwrite each other.

The addon manager will persist the hosted manifest cleanup finalizer before creating the first claim
Work. It will not remove that finalizer merely because a Work delete request has been issued; every
claim and runtime Work protected by the finalizer must actually disappear first. This closes the
deletion race between the first hosting Work and lifecycle protection.

If the addon uses an instance-specific hosting namespace, the generated `Namespace` will itself be a
cluster-scoped claim. Namespaced Deployments, ServiceAccounts, Roles, RoleBindings, Secrets, and
hooks will use that hosting namespace and will remain in runtime Works.
The generated Namespace will use the same ownership checks and reconciliation rules whether or not
`HostedClusterScopedResourcesEnabled` is also selected.

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

### Required Work support

The Work API needs an additive per-manifest ownership declaration carrying the expected
`hosting-addon` annotation value. The framework will supply it from the addon instance identity.
The work agent will enforce that owner before applying a manifest, adding Work ownership, or
deleting a resource. It will retain the expected owner in AppliedManifestWork so finalization can
continue after the hub-side Work disappears.

Before enabling writes in a claim Work, the framework will submit it as `ReadOnly` with
`SelectivelyOrphan`. A supporting agent will report `HostingResourceClaimSupported=True` for that
Work generation without changing any resource. The framework will then select the steady-state SSA
strategy and intended deletion policy. If support is absent, the Work will remain read-only and no
runtime will be delivered. This check establishes agent support before any resource is changed.

The existing Work settings alone do not provide these ownership checks. The new declaration and
supporting agent are required before the framework can send a writable claim. Existing
AppliedManifestWork resource tracking remains in use.

### Reconciliation

Each reconcile will read the live resource and make one decision:

| Live resource | Action |
|---|---|
| Absent | Create it with the expected owner annotation and the owner reference required by the Work's deletion policy. |
| Owner matches this addon instance | Ensure Work ownership and apply the addon's desired fields. |
| Owner is missing or belongs to another instance | Leave it unchanged and report `HostingResourceConflict`. |

Creation will use a create request so an object appearing at the same name cannot be overwritten.
An AlreadyExists response requires another read and the same ownership check. A foreign resource
will not gain a Work owner reference or be recorded as owned by this Work.

For an existing owned resource, the work agent will check the live owner before each write and use
the UID and resourceVersion from that read as concurrency checks. A conflict requires a fresh read;
it will not authorize an unconditional retry. These checks remain local to the agent.

Once ownership is established, the Work will use:

```yaml
updateStrategy:
  type: ServerSideApply
  serverSideApply:
    force: true
    fieldManager: work-agent-<deterministic full-digest suffix for this addon instance>
```

The field manager will be stable for the addon instance and use ManifestWork's required
`work-agent` prefix. The addon controls the fields declared in its manifest. If another controller
changes one of those fields, the next reconcile will restore the desired value. Fields outside
that declaration will follow normal SSA ownership and merge rules. For example, another controller
may maintain an annotation key that the addon does not declare. An atomic list or map is one
management unit, so its contents cannot be divided between controllers.

The agent will establish Work ownership independently of whether the desired spec can be applied,
and track the owned resource in the existing AppliedManifestWork status. A later validation failure
will therefore not prevent cleanup. Intentional orphan policies will continue to omit Work ownership.

After a restart the agent will read the resource and repeat the same decision. While the Work is
active, a missing resource can be created again.

### Runtime gating and failures

The framework will create the initial runtime Works only when every claim Work is in its writable
configuration and reports `Applied=True` and `HostingResourceClaimSupported=True` for its current
generation. The supporting agent's apply result includes the ownership check.

If an instance already has runtime Works and a later claim revision is pending or fails, those
runtime Works will remain unchanged. The framework will withhold the new runtime revision and
report the blocked claim through `HostingManifestApplied`.

A missing or different owner will use `HostingResourceConflict`, with the resource identity and
expected and observed owners in the message. Invalid desired manifests and other API failures will
retain the general apply-failure reason. Changes by another manager to the addon's declared fields
will be reconciled, rather than reported as ownership conflicts.

### Deletion

The hub will delete runtime Works first, then claim Works, and wait for their actual disappearance.
The work-agent finalizer will use the resource identities and UIDs already tracked by
AppliedManifestWork and read each live resource:

| Live resource | Action |
|---|---|
| Absent | Cleanup is complete. |
| Tracked UID and addon owner match | Remove it according to the Work's deletion policy, unless it is intentionally orphaned. |
| UID changed, or owner is missing or different | Preserve it. Remove only this Work's owner reference if one remains. |

Owner-reference changes will use the live object's concurrency checks. Resource deletion will use
the tracked UID as a precondition, so a replacement at the same name cannot be removed after a
stale read. A conflict requires another read and the same checks.

Finalization will only clean up tracked resources. It can do so even when the desired update was
invalid or never reached `Applied=True`, and will never create a missing object. A restart repeats
the same checks. The finalizer will remain until every tracked resource has been removed, preserved
under the orphan policy, or found absent.

The addon manager will keep its cleanup finalizer until the protected Works are gone. The absence
of the target ManagedCluster will not bypass cross-namespace cleanup.

If the hosting ManagedCluster is missing, the hub cannot obtain a fresh observation. Before
requesting deletion it will keep any uncertain claim read-only with selective orphaning, preserving
the earlier safeguard against recreating resources if the agent returns. Agent-side finalization
will still use the live ownership checks above. Generic Work cleanup finalizers will not be
force-removed; a permanently unavailable agent may require administrator inspection.

### Status

The existing `HostingManifestApplied` condition on `ManagedClusterAddOn` will summarize both phases.
The proposed reasons will be:

| Reason | Meaning |
|---|---|
| `HostingResourceClaimsPending` | Agent support or a current-generation claim apply result is still pending. |
| `HostingResourceConflict` | A live object is unannotated or belongs to another instance. |
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
- The agent will check live ownership before every apply and cleanup operation.
- Creation will not overwrite an object that appears after an absence check, and writes will use
  the current read's UID and resourceVersion checks.
- The framework will stamp the expected owner instead of trusting a chart-supplied value.
- SSA will restore the fields declared by this addon while preserving other controllers' fields
  according to Kubernetes merge rules.
- Cleanup will use existing Work resource tracking and will never apply a rejected spec or recreate
  a missing object.
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
- read-only support checks that cannot mutate foreign resources on an older agent;
- the absent, owned, and foreign resource decisions, including AlreadyExists after an absence check;
- restoration of a declared field changed by another manager and preservation of an undeclared
  annotation, using normal SSA merge rules;
- local ownership checks and concurrent deletion, recreation, or update before each write;
- initial runtime gating and preservation of already-running runtime Works;
- finalization of owned, foreign, absent, and replaced resources, including intentional orphaning;
- repeated cleanup after an invalid desired update or agent restart, without applying the rejected
  spec or creating a missing object; and
- conservative cleanup when the hosting cluster disappears.

Integration coverage will use two target managed clusters that share one hosting cluster. It will
verify that one prerequisite addon owns a shared `ClusterRole`, while two hosted instances create
different `ClusterRoleBinding` names and subjects. It will also cover an unannotated pre-existing
binding, a binding owned by another instance, changes to declared and undeclared fields, and a
controller restart during apply or deletion. A delete-and-recreate case will replace an object
between its ownership check and write; a foreign replacement must remain unchanged. An older work
agent must leave the support-check Work read-only. The generated Namespace will follow the same
rules whether the cluster-scoped resource option is enabled or disabled.

Deletion coverage will start with an immutable-field update rejected by the API server. Deleting
the addon must still complete without applying that spec. It will also cover mixed owned, foreign,
absent, and replaced resources, intentional orphaning, and a restart before cleanup acknowledgement.

End-to-end coverage will create two new opted-in hosted addon instances on one hosting cluster. Both
agents will reach availability with independent RBAC. Deleting either instance will remove only its
per-instance claim and runtime while leaving the other instance and the shared prerequisite
resource available. A separate conflict case will prove that a foreign fixed-name object remains
byte-for-byte unchanged and that the affected `ManagedClusterAddOn` reports
`HostingResourceConflict` before any runtime is delivered.

### Graduation Criteria

#### Alpha

- The opt-in, deterministic identity, resource classification, ownership checks, runtime gate,
  status, and finalization are available together.
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

The work agent must support the ownership declaration during both apply and finalization. The
framework will require `HostingResourceClaimSupported` from the initial read-only support check
before selecting a writable strategy. Without it, claims will remain read-only and runtime will
not be delivered. The support check, owned-field SSA, orphan policy, and finalization must remain
supported for the instance's complete lifetime.

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

**Apply without checking the owner.** SSA field management cannot distinguish a foreign resource
from one this addon owns. The live owner check must happen before either applying fields or adding
a Work owner reference. Force is appropriate for the declared fields of an owned resource, but it
does not authorize adoption of another instance's object.

**Use non-forcing SSA and transfer create-time fields.** This adds a handoff and cleanup protocol
to preserve third-party edits to fields the addon declares. The addon owns those fields and should
restore their desired values, so neither the handoff nor direct `managedFields` edits are needed.

**Use one Work and rely on manifest ordering.** `ManifestWork` does not provide the ownership barrier
needed between cluster-scoped claims and namespaced runtime objects. Splitting the phases gives the
controller an explicit point at which all live ownership results must be current.

**Use only Kubernetes owner references.** An owner reference written while observing an existing
object would itself be an adoption and mutation. It also would not distinguish an object created
during creation from a foreign object when deletion races with reconciliation. The expected owner
and a fresh live check let the agent make that decision before every operation.

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
- The Work API needs an additive per-manifest ownership declaration. The work agent needs support
  reporting and ownership checks during apply and finalization. Namespace claims in KEP 191 use
  the same behavior. Existing SSA and AppliedManifestWork tracking will be reused without a new
  Work API version, a field ownership handoff, or a recovery Work.

## References

- Issue: https://github.com/open-cluster-management-io/enhancements/issues/192
- [33-hosted-deploy-mode](../33-hosted-deploy-mode/README.md) - klusterlet Hosted mode
- [47-manifestwork-updatestrategy](../47-manifestwork-updatestrategy/README.md) - CreateOnly and server-side apply
- [63-hosted-addon](../63-hosted-addon/README.md) - addon Hosted mode
- [29-manifestwork-status-feedback](../29-manifestwork-status-feedback/README.md) - manifest feedback
