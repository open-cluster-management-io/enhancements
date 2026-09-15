# Give each hosted addon instance a safe runtime namespace

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
[website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

A hosting cluster can run the same addon for many managed clusters. Hosted addon manifests currently
use the managed-cluster installation namespace as their hosting-cluster runtime namespace. That value
is not unique to a `ManagedClusterAddOn`, so two instances of the same addon can render the same
Deployment, ServiceAccount, Secret, and Lease names into the same namespace.

This proposal gives an addon type an opt-in that assigns every new hosted `ManagedClusterAddOn` a
deterministic, DNS-safe runtime namespace derived from the addon's identity. The namespace will be
claimed and its ownership verified before any runtime manifest or registration credential is placed
there. The managed-cluster installation namespace will retain its existing meaning and behavior.

The feature will be off by default. Existing Default-mode and Hosted-mode addons will not be moved or
changed by an upgrade. An addon type will select the opt-in before creating its
`ManagedClusterAddOn` objects, and that choice and each object's install mode will remain unchanged
for the lifetime of those objects.

### Terms

- **Target cluster**: the managed cluster represented by the namespace containing a
  `ManagedClusterAddOn`.
- **Hosting cluster**: the managed cluster on which a Hosted-mode addon agent runs.
- **Managed installation namespace**: the namespace on the target cluster represented by
  `ManagedClusterAddOn.status.namespace` and the existing addon installation configuration.
- **Hosting namespace**: the instance-specific namespace on the hosting cluster in which the hosted
  addon agent and its namespaced supporting resources run.
- **Addon instance identity**: the ordered pair `(managed cluster name, addon name)`, which is the
  identity of one `ManagedClusterAddOn`.
- **Claim Work**: a separate `ManifestWork` that creates or verifies ownership of the hosting
  namespace before a runtime Work can be created.
- **Runtime Work**: the `ManifestWork` containing the namespaced resources that run the addon agent on
  the hosting cluster.

## Motivation

[Issue #191](https://github.com/open-cluster-management-io/enhancements/issues/191) describes the
collision that occurs when several instances of one hosted addon run on one hosting cluster. An addon
normally renders stable names for objects such as its Deployment, ServiceAccount, Role, Secret, and
Lease. If every instance renders those names into the same namespace, a later instance can update the
objects used by an earlier instance.

The existing addon installation namespace is documented as a managed-cluster destination. Reusing it
as the hosting destination overloads one value with two independent meanings. Asking an administrator
to configure a different value for every managed cluster would preserve the possibility of human
error and would also change the managed-side destination.

The framework already has the identity needed to make the hosting destination unique. A
`ManagedClusterAddOn` named `addonName` in namespace `managedClusterName` represents exactly one addon
instance. A deterministic namespace derived from both values can separate namespaced resources
without requiring an administrator to allocate names.

### Goals

- Give every opted-in Hosted-mode `ManagedClusterAddOn` a deterministic, instance-specific runtime
  namespace on its hosting cluster.
- Keep the managed installation namespace and `status.namespace` semantics unchanged.
- Expose the resolved hosting namespace through a dedicated status field and `relatedObjects`.
- Detect a pre-existing or colliding namespace before any addon runtime object or registration
  credential can overwrite an object in that namespace.
- Make namespace acquisition safe when the candidate namespace already belongs to another owner.
- Keep Default-mode behavior and all existing Hosted-mode behavior unchanged unless an addon type
  explicitly opts in before its addon objects are created.
- Define deletion ordering that does not abandon cross-namespace Works or remove lifecycle finalizers
  before their resources have been handled.

### Non-Goals

- Migrating an existing addon instance from a shared namespace to an instance-specific namespace.
- Changing an existing `ManagedClusterAddOn` between Default and Hosted mode, or changing the
  namespace option while addon instances exist. Those configurations are immutable and such changes
  are outside this proposal.
- Protecting arbitrary cluster-scoped resources rendered by hosted addons. Namespace ownership is
  required by this proposal, but protection for ClusterRoles, ClusterRoleBindings, CRDs, and other
  cluster-scoped resources belongs to [issue #192](https://github.com/open-cluster-management-io/enhancements/issues/192).
- Supplying a managed-cluster ServiceAccount credential to a hosted addon. That is covered by
  [issue #190](https://github.com/open-cluster-management-io/enhancements/issues/190).
- Treating a deterministic name or an ownership annotation as a security boundary.
- Providing zero-downtime deletion and recreation.
- Allowing an administrator to choose the generated hosting namespace.

## Proposal

### User Stories

#### Administrator hosting one addon for many managed clusters

I want every instance of the addon to run in a separate namespace on the hosting cluster so that one
instance cannot overwrite another instance's namespaced resources.

#### Addon author

I want a framework-provided hosting namespace and instance identifier in my templates so that I do
not have to invent a naming or coordination scheme.

#### Administrator investigating an addon

I want the resolved hosting namespace recorded on `ManagedClusterAddOn` so that I can locate the
runtime resources without reproducing a hash algorithm.

#### Operator who does not opt in

I want an upgrade to preserve every existing namespace, manifest, registration destination, and
default without a migration.

### Risks and Mitigations

**A truncated digest can collide.** The generated name will include 80 bits of a domain-separated
SHA-256 digest, so an accidental collision is unlikely but not impossible. The framework will not
rely on probability for correctness. It will verify an ownership annotation through Work feedback
before creating runtime resources. A different or missing owner will produce a conflict condition
instead of adoption.

**A pre-existing namespace can have the generated name.** A namespace with no ownership annotation
will be treated as foreign. Initial acquisition will use `CreateOnly`, so the framework will not add
an annotation to or otherwise change that namespace. Runtime and registration will remain blocked.

**An annotation is not proof against a privileged actor.** An actor that can create or modify
namespaces can copy an annotation. Kubernetes RBAC remains the authorization boundary. The annotation
is a coordination and collision-detection mechanism, not authentication.

**A new namespace can affect cluster policy.** NetworkPolicy, ResourceQuota, LimitRange, admission
policy, and namespace-scoped RBAC do not automatically follow an addon to a new namespace. This is
why the feature will be opt-in and immutable for the lifetime of each addon object. An addon author
must document the policies required by the generated namespace before enabling the option.

**A hosting cluster can disappear during deletion.** The hub must prefer preserving a possibly
foreign resource over completing unsafe deletion. Unverified claims will be changed to `CreateOnly`
and selectively orphaned before their Works are deleted. Generic Work cleanup finalizers will not be
force-removed. If the hosting work agent cannot complete cleanup, an administrator may need to restore
the hosting cluster or resolve the Work finalizer after inspecting the remote resources.

**Status remains useful during deletion.** The hosting namespace is needed while
cross-namespace resources are being removed. `status.hostingNamespace` and its `relatedObjects` entry
will remain until all hosted Works and their addon lifecycle finalizers are gone, then they will be
cleared.

## Design Details

The design has seven parts:

1. An addon type opts in before any of its `ManagedClusterAddOn` objects are created.
2. The framework derives a stable namespace from the addon instance identity.
3. The resolved value is published separately from the managed installation namespace.
4. A claim Work verifies namespace ownership before a runtime Work can exist.
5. templates render all hosted namespaced resources into the verified namespace.
6. hosted registration uses the published namespace only after ownership is verified.
7. deletion retains finalizers and status until cross-namespace cleanup is complete.

### 1. Opt-in and immutable configuration

`AgentAddonOptions` will gain an option that is false by default:

```go
type AgentAddonOptions struct {
    // Existing fields are omitted.

    // HostingNamespaceEnabled places hosting-cluster resources for each
    // ManagedClusterAddOn in a deterministic, instance-specific namespace.
    // It does not change the managed-cluster installation namespace.
    HostingNamespaceEnabled bool
}
```

The addon factory will provide a corresponding `WithHostingNamespaceEnabledOption()` convenience
method. The option will require hosted mode support. It will affect only Hosted-mode instances of an
addon type configured with the option from the start.

The following are part of the API contract:

- The option will remain false unless an addon author explicitly enables it.
- An addon author will set it before creating any `ManagedClusterAddOn` for that addon type.
- The option will not be toggled while any such object exists.
- The install mode chosen for a `ManagedClusterAddOn` will not change during that object's lifetime.

The framework will not define behavior for changing those choices. This keeps an upgrade inert:
installing versions that understand this proposal will not relocate or rewrite any existing addon.

When the option is false:

- no generated hosting namespace or namespace claim Work will be created;
- `status.hostingNamespace` will remain empty;
- no new hosting namespace or instance identity template value will be injected;
- registration will continue using its existing destination rules; and
- Default mode and legacy Hosted mode will behave as they did before this proposal.

### 2. Deterministic instance identity and namespace

The framework and hosted components will share one helper for the addon instance identity. Inputs
will be the full, ordered `(managedClusterName, addonName)` pair. The digest will be SHA-256 over a
versioned domain ending in a NUL byte and length-delimited inputs. Each length will be encoded as an
unsigned 64-bit big-endian integer:

```text
domain bytes: open-cluster-management.io/addon-instance/v1\x00
input bytes: uint64be(length(managedClusterName)), managedClusterName,
             uint64be(length(addonName)), addonName
instanceID: first 20 lowercase hexadecimal digest characters
```

Length delimiting prevents ambiguous pairs from producing the same byte sequence. The versioned
domain prevents another use of SHA-256 from accidentally sharing this identity space.

The namespace will have this form:

```text
open-cluster-management-<addon-token>-<instanceID>
```

`open-cluster-management-` is 24 characters and `instanceID` is 20 characters. The separating dash
leaves at most 18 characters for `addon-token`, keeping the complete name within the 63-character
DNS-1123 label limit.

The addon token will be formed as follows:

1. Convert the addon name to lowercase.
2. Preserve ASCII letters and digits.
3. Replace each run of other characters with one dash.
4. Trim leading and trailing dashes.
5. Truncate to 18 characters and trim a trailing dash again.
6. Use `addon` if the result is empty.

The digest will always use the complete original inputs, not the shortened token. The human-readable
token is diagnostic only. The 20-character instance ID will also be exposed independently for addon
authors that need collision-resistant names for resources outside a namespace.

Because every controller will use the shared helper, the namespace will be recoverable after a hub
restart without persisting random state. The algorithm and version will be part of the compatibility
contract and will not change for existing API versions.

### 3. Status and API compatibility

`ManagedClusterAddOnStatus` will gain a dedicated field in both `v1alpha1` and `v1beta1`:

```go
type ManagedClusterAddOnStatus struct {
    // namespace is the namespace on the managed cluster used for the addon
    // registration Secret or Lease.
    Namespace string `json:"namespace,omitempty"`

    // hostingNamespace is the namespace on the hosting cluster where the
    // addon agent runs in Hosted mode.
    HostingNamespace string `json:"hostingNamespace,omitempty"`
}
```

The two fields will have deliberately separate meanings:

| Field | Cluster | Meaning |
|---|---|---|
| `status.namespace` | target managed cluster | Existing managed-side installation, registration Secret, and Lease namespace |
| `status.hostingNamespace` | hosting cluster | Opted-in Hosted-mode runtime, registration Secret, and Lease namespace |

`status.namespace` will not be repurposed. Existing readers and writers will continue to see its
current value.

Both served API versions will use the same JSON field name and schema. Generated conversions will
copy `hostingNamespace` in both directions so a `v1alpha1` to `v1beta1` round trip preserves it. The
field will be optional and empty for every non-opted-in object. Older clients that do not know the
field can ignore it without affecting spec compatibility.

The addon manager will also merge this object reference into `status.relatedObjects`:

```yaml
- group: ""
  resource: namespaces
  name: <status.hostingNamespace>
```

The typed field will be the authoritative machine-readable routing value. `relatedObjects` will be a
discovery aid for users and generic tooling; consumers will not parse it to decide where credentials
or manifests belong.

The status field and RelatedObject will be set when the opted-in Hosted-mode addon is reconciled. They
will remain present while any hosting ManifestWork or hosting lifecycle finalizer remains. They will
be removed only after hosted cleanup is complete. This makes status truthful during asynchronous
cross-namespace deletion.

### 4. Claim before runtime

A Kubernetes Namespace is cluster-scoped, so placing a Namespace manifest and a Deployment in one
ordinary Work would not prevent the Deployment from reaching a pre-existing foreign namespace. The
framework will split reconciliation into a claim phase and a runtime phase.

The claim Work will contain the generated Namespace with this annotation:

```yaml
metadata:
  annotations:
    addon.open-cluster-management.io/hosting-addon: <managedClusterName>/<addonName>
```

The initial claim will use the ManifestWork `CreateOnly` update strategy. It will also selectively
orphan that Namespace if the Work is deleted before ownership has been observed. These settings give
the initial claim two safe outcomes:

- If the Namespace does not exist, the work agent creates it with the expected annotation.
- If it already exists, the work agent does not mutate it and the existing annotation can be
  inspected through feedback.

The claim Work will request both annotation feedback and a custom CEL condition named
`HostingResourceOwned`. The condition will be true only when the live Namespace contains the exact
expected owner value. The addon manager will require all of the following before proceeding:

1. The claim Work exists.
2. Its `Applied` condition reports the current Work generation.
3. The owner feedback is present.
4. `HostingResourceOwned` is true for the expected instance identity.
5. The verified claim has been promoted to non-forcing server-side apply with a stable,
   instance-specific field manager.
6. The promoted Work generation has been observed.

Promotion will remove temporary orphaning for a Namespace that has been proven to belong to this
addon. `force` will remain false so the claim cannot silently take fields from another field manager.

If the Namespace is unannotated or has another owner, the addon manager will set
`HostingManifestApplied=False` with reason `HostingResourceConflict`. It will not create the runtime
Work and registration will not write a Secret or Lease into that Namespace. A transient absence of
current-generation feedback will remain pending rather than being interpreted as ownership.

The claim Work will be separate from the runtime Work. That boundary prevents Work ordering or a
single Work's parallel manifest application from placing namespaced resources before the ownership
decision.

### 5. Runtime template contract

When the option is selected, both the Helm addon factory and the Go template addon factory will
expose these built-in values:

```text
.AddonInstallNamespace      existing managed installation namespace
.HostingInstallNamespace    hosting runtime namespace
.HostedInstanceID           stable ID for this ManagedClusterAddOn
```

For Helm value serialization, the new values will be named `hostingInstallNamespace` and
`hostedInstanceID`.

When this proposal is enabled and the install mode is Hosted,
`.HostingInstallNamespace` will come only from `status.hostingNamespace`. Addon templates must use it
for every namespaced object labeled for the hosting cluster. The runtime Work builder will reject a
hosting manifest whose namespace does not exactly equal the authoritative status value.

When the proposal is not enabled, the factories will not inject `HostingInstallNamespace` or
`HostedInstanceID`. Existing user-supplied values with those names and all current template inputs
will therefore remain unchanged. An addon author will add the new template references as part of the
same creation-time opt-in.

The framework will generate the Namespace manifest. Addon authors will not need to add a duplicate
Namespace template. Arbitrary cluster-scoped templates will remain outside this proposal; the
generated Namespace is the one required exception.

The hosted lifecycle finalizers required for runtime and pre-delete hook cleanup will be persisted
before the first runtime Work can be created. In particular, when an addon defines a hosting
pre-delete hook, its hosting pre-delete finalizer will be established before runtime creation. A
deletion request therefore cannot arrive in a gap where runtime resources exist but the hook is not
protected by a finalizer.

### 6. Registration Secret and Lease routing

The hosted registration agent must use `status.hostingNamespace` as the authoritative namespace for
its registration Secret and Lease. It will not independently reproduce the namespace name from the
cluster and addon names. This avoids coupling registration behavior to an algorithm that is owned by
the addon framework and makes the status value the single routing contract.

Publishing the destination is not sufficient to make it safe. Before writing a registration Secret,
the hosting-cluster registration agent will read that Namespace directly and require its
`addon.open-cluster-management.io/hosting-addon` annotation to equal the exact
`<managedClusterName>/<addonName>` identity. It will not infer namespace ownership from the aggregate
`HostingManifestApplied` condition, because a later failure in an unrelated runtime manifest must not
churn a valid registration credential.

If `status.hostingNamespace` is nonempty but the Namespace is missing, unannotated, or differently
owned, registration will stop any registration controller for that destination and remove any
well-known credential Secret that might have been created before the unsafe state was observed. It
will wait for a later exact ownership result instead of falling back to `status.namespace`.

For a legacy addon with an empty `status.hostingNamespace`, the registration agent will continue to
use the existing installation namespace and hosted-mode detection. Once the new field is nonempty,
it will take precedence over legacy inference.

### 7. Creation and deletion lifecycle

The lifecycle below applies to an addon created with Hosted mode and the namespace option already
selected, with those choices unchanged until deletion.

#### Creation

Creation will proceed in this order:

1. Resolve and publish `status.hostingNamespace` and its RelatedObject.
2. Create the Namespace claim Work with safe acquisition settings.
3. Wait for current-generation ownership feedback and safe promotion.
4. Persist the hosted manifest finalizer and any configured hosting pre-delete hook finalizer.
5. Create the namespaced runtime Work.
6. Mark hosting manifests applied only after the Work reports success.
7. Allow hosted registration to create its Secret and Lease after its direct Namespace ownership
   check succeeds.

There can be an availability delay while ownership is checked. There cannot be an interval in which
the hosted agent runs in an unverified namespace.

#### Deletion

Deletion will proceed in this order:

1. Stop creating or updating runtime resources.
2. If a hosting pre-delete hook is configured, create its Work only after its finalizer is present and
   wait for the hook's terminal result according to the existing hook contract.
3. If a hook builder no longer returns a hook, delete any stale hook Work and wait for its actual
   disappearance before removing the hook finalizer.
4. Delete runtime Works and wait for the Works and their remote AppliedManifestWorks to be cleaned up.
5. Delete the claim Work. An owned Namespace can be removed; an unverified or foreign Namespace must
   be orphaned.
6. Remove the corresponding addon finalizer only after the Works protected by it are gone.
7. Clear `status.hostingNamespace` and the namespace RelatedObject only after no hosted Work or hosted
   lifecycle finalizer remains.

Issuing a Work delete request will not count as cleanup completion. Waiting for disappearance avoids
removing the `ManagedClusterAddOn` while a cross-namespace Work still has remote resources to delete.

#### Missing target ManagedCluster

The absence of the target `ManagedCluster` will not short-circuit addon cleanup. The controller can
still identify hosting Works from their addon name and addon namespace labels. It will run hook,
runtime, and claim cleanup, retain status while those Works exist, and remove finalizers only after
the cross-namespace Works are gone.

#### Missing hosting ManagedCluster

If the hosting `ManagedCluster` disappears before claim ownership has been observed, the controller
cannot know whether each claimed object was created for this addon or was pre-existing. Before
deleting the claim Work, it will persist `CreateOnly` for every uncertain resource and add an exact
selective orphan rule. It will verify that safe Work shape before issuing deletion.
This prevents deletion of a foreign object if stale feedback arrives later.

A resource previously verified as owned can retain normal deletion behavior. A resource known to be
foreign will be orphaned. Generic ManifestWork cleanup finalizers will not be force-removed because
doing so could abandon resources on a cluster that later returns. If the work agent is permanently
unavailable, deletion may require administrator intervention. Safety takes precedence over automatic
finalizer removal in this failure mode.

### Architecture

```text
                    hub cluster

  ManagedClusterAddOn <target>/<addon>
      status.namespace ----------------------> managed-side namespace
      status.hostingNamespace ----+
      RelatedObject: Namespace     |
                                   |
                    hosting cluster namespace on hub
                                   |
                       claim ManifestWork
                         CreateOnly first
                         owner feedback + CEL
                                   |
                         ownership verified?
                            /            \
                          no              yes
                          |                |
             HostingResourceConflict      | promote to non-force SSA
             no runtime or credential      |
                                           v
                                  runtime ManifestWork
                                           |
                                           v
                                  hosting cluster
                         Namespace <status.hostingNamespace>
                           Deployment, ServiceAccount,
                           registration Secret, Lease
```

### Test Plan

Unit tests will cover:

- deterministic identity output and separation of ambiguous input pairs;
- DNS-1123 validity and the 63-character bound for maximum-length and unusual addon names;
- use of the complete managed cluster and addon names in the digest;
- conversion of `hostingNamespace` between `v1alpha1` and `v1beta1`;
- template values when the option is enabled and disabled;
- rejection of a hosted namespaced manifest outside `status.hostingNamespace`;
- claim acquisition, owner feedback parsing, non-forcing promotion, and conflict conditions;
- temporary selective orphan rules for claims whose ownership is uncertain;
- RelatedObject retention and removal; and
- finalizer ordering when a hook is present, absent, or removed from the builder output.

Integration tests will cover:

- two instances of one addon on the same hosting cluster receiving different namespaces while using
  identical names for their namespaced runtime objects;
- an unannotated generated-name collision remaining unchanged and blocking runtime creation;
- a namespace owned by another addon producing `HostingResourceConflict`;
- registration Secret and Lease creation only after ownership has been verified;
- deletion waiting for hook, runtime, and claim Works to disappear;
- deletion when the target ManagedCluster is already absent;
- conservative orphaning when the hosting ManagedCluster disappears before ownership feedback; and
- an upgrade with the option unset producing no status, Work, namespace, registration, or template
  behavior change for existing Default-mode and Hosted-mode addons.

Tests that mutate install mode or the namespace option after creation are outside this proposal
because those choices are immutable for a `ManagedClusterAddOn` lifetime.

### Graduation Criteria

#### Alpha

- The option is available to addon authors and defaults to false.
- Both API versions expose the optional status field and preserve it through conversion.
- Namespace naming, claiming, status, template values, registration gating, and cleanup behavior are
  covered by unit and integration tests.
- At least one example addon documents the immutable opt-in contract.

#### Beta

- At least two hosted addon implementations use instance-specific namespaces.
- Upgrade and deletion behavior has been exercised at fleet scale.
- Conflict conditions and administrator recovery steps are documented.
- Metrics or events make claim-pending and claim-conflict states observable.

#### GA

- The namespace identity and status contracts have remained stable through at least two minor
  releases.
- No unresolved data-loss or cross-instance collision issue is attributed to this feature.
- User-facing documentation describes opt-in, immutability, security boundaries, and cleanup.

### Upgrade / Downgrade Strategy

An upgrade will not enable this proposal. `HostingNamespaceEnabled` defaults to false, the new status
field is optional, and template values preserve their previous destinations when the option is
unset. Existing Default-mode and Hosted-mode `ManagedClusterAddOn` objects require no data migration
and will not be relocated.

The API and CRD containing `status.hostingNamespace` must be upgraded before addon managers write the
field. Hosted registration components that understand the field and ownership gate must be upgraded
before an addon type enables the option. During version skew, older clients may ignore the optional
status field, but an addon author must not opt in until every participating hosted registration agent
supports it.

Opt-in is supported only when selected before any `ManagedClusterAddOn` for that addon type is
created. It is not an upgrade step for existing objects, and no existing object requires migration.
Changing that selection later is outside this proposal.

Default-off deployments can downgrade without object changes. An opted-in instance requires
components that understand the namespace ownership and registration routing contracts throughout
its lifetime; downgrading those components below that capability while such an instance exists is
unsupported.

The naming algorithm will not change on upgrade. A future incompatible algorithm would require a new
explicit opt-in and identity version, not silent recomputation for existing objects.

## Implementation History

- 2026-09-14: Initial proposal.

## Alternatives

### Reuse `agentInstallNamespace`

An administrator could assign a different installation namespace for every target cluster. This
would keep one value responsible for both managed-side and hosting-side resources, change where
managed resources install, and depend on administrators never reusing a value. It does not solve the
missing framework concept.

### Concatenate the managed cluster and addon names

Kubernetes namespace names are DNS-1123 labels with a 63-character maximum. Legal object names can
exceed that limit when concatenated, and punctuation can be invalid in a label. Truncation without a
digest would introduce deterministic collisions. A bounded human token plus digest handles both
constraints.

### Generate a random namespace

A random suffix could avoid ordinary collisions, but the chosen value would need durable
coordination, recovery, and ownership rules. A deterministic identity is easier to reconstruct and
inspect while retaining an explicit collision check.

### Publish only a RelatedObject

`status.relatedObjects` is useful for discovery but is not a strong routing contract. Registration
and template rendering need one authoritative value with clear API ownership and conversion
semantics. The typed status field provides that value, while RelatedObjects remains useful to users.

### Adopt an existing unannotated namespace

Adding the expected annotation to an existing namespace would allow the addon to take over resources
that belong to a user or another controller. `CreateOnly` acquisition and exact owner feedback avoid
that destructive behavior.

### Force server-side apply during acquisition

Force apply could seize annotation or policy fields from another manager. Initial `CreateOnly`
followed by verified, non-forcing apply preserves ownership boundaries.

### Add an instance suffix to every namespaced resource

Requiring each addon author to suffix every Deployment, Role, ServiceAccount, Secret, Lease, and
reference would be easy to apply inconsistently. Namespace isolation lets ordinary names remain
stable and also provides an operational boundary for each addon instance.

## Infrastructure Needed

No new external infrastructure is required. The design uses existing ManagedClusterAddOn status,
ManifestWork feedback, CEL conditions, server-side apply, and Kubernetes namespaces.

## References

- [Issue #191](https://github.com/open-cluster-management-io/enhancements/issues/191)
- [Hosted mode for addon agents](../63-hosted-addon/README.md)
- [Hosted addon hosting-cluster discovery](../188-hosted-addon-follow-klusterlet/README.md)
- [Issue #190: managed-cluster credentials for hosted addons](https://github.com/open-cluster-management-io/enhancements/issues/190)
- [Issue #192: protect hosted addon cluster-scoped resources](https://github.com/open-cluster-management-io/enhancements/issues/192)
