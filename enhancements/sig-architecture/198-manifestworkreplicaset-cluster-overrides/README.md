# Cluster-Specific Manifest Overrides for ManifestWorkReplicaSet

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

`ManifestWorkReplicaSet` (MWRS) distributes one `manifestWorkTemplate` to every cluster
selected by its `placementRefs`. The payload is copied verbatim: only the `ManifestWork`'s
namespace, labels and annotations differ per cluster. Users who need a small per-cluster
difference — a `nodeSelector` value that matches the local node pool, a pod label that lets
a service mesh distinguish copies — have to abandon MWRS and maintain one `ManifestWork`
per cluster.

This proposal introduces a new namespaced resource, `ManifestOverridePolicy`, which selects
manifests by resource identity and clusters by `ManagedCluster` labels, and applies ordered
patches (JSON Patch, JSON Merge Patch, or strategic merge patch) to the matched manifests as
they are rendered for each cluster. Policies apply implicitly to the `ManifestWorkReplicaSet`
resources in their namespace, so a platform team can define an override once and have it hold
for every workload distributed from that namespace, rather than restating it in each MWRS.

A companion cluster-scoped resource, `ResourceOverrideSchema`, supplies the list-merge metadata
(`x-kubernetes-patch-merge-key` and friends) that strategic merge patching needs for custom
resources, whose CRD schemas cannot declare it.

## Motivation

The request originates in [ocm#1175][ocm-1175] and was carried into
[enhancements#198][issue-198]. Two distinct needs appear in those threads, and separating
them explains the shape of this proposal.

**Values that are facts about a cluster.** The application node pool is tagged
`nodepool: apps-a` on `cluster-a` and `nodepool: apps-b` on `cluster-b`. A Deployment's
`nodeSelector` must match whichever one it lands on. This is true of every workload on that
cluster, not just this one. Region-scoped settings — a higher `priorityClassName` in
`region-a` — have the same shape. These are naturally expressed as "for clusters matching
this label, patch this field", which is what this proposal provides.

**Values derived from cluster identity.** Each cluster's copy of a Deployment needs a
distinguishing pod label so Envoy or Istio can route to the copies individually. Nothing
about `cluster-a` says the label should be any particular string, only that it must differ
from `cluster-b`'s. Expressing this with static patches means one override entry per cluster,
which does not survive fleet growth. This need is **out of scope here** and is deferred to a
separate templating enhancement (see [Non-Goals](#non-goals)); the API below is shaped so
that templated patch values can be added without a breaking change.

### Current Limitations

Verified against `open-cluster-management-io/api` and `open-cluster-management-io/ocm` at
`main`:

- `ManifestWorkReplicaSetSpec` (`work/v1alpha1/types_manifestworkreplicaset.go`) has three
  fields — `manifestWorkTemplate`, `placementRefs`, `cascadeDeletionPolicy`. There is no
  customization surface.
- Delivery is a literal struct copy: `Spec: mwrSet.Spec.ManifestWorkTemplate` in
  `manifestworkreplicaset_deploy_reconcile.go`.
- The MWRS controller is wired with a `PlacementDecisionInformer` only
  (`manifestworkreplicaset_controller.go`), and `ClusterDecision` carries just `clusterName`,
  `reason` and `score`. No `ManagedCluster` attribute is reachable from the controller today.

### Goals

- Let users keep one shared `manifestWorkTemplate` in MWRS and declare per-cluster
  differences separately.
- Select target clusters by `ManagedCluster` label, and target manifests by resource identity.
- Support JSON Patch (RFC 6902), JSON Merge Patch (RFC 7386) and Kubernetes strategic merge
  patch, including for custom resources.
- Make the applied result deterministic: the same inputs always produce the same rendered
  `ManifestWork`.
- Make override failures visible per cluster, and surface them to the MWRS rollout machinery
  rather than silently stalling.
- Make it possible to tell which policy changed a given `ManifestWork`.

### Non-Goals

- **Templating or CEL expressions in patch values.** Deferred to a separate enhancement
  covering both MWRS and `ManifestOverridePolicy`. The consensus on [#198][issue-198] is that
  it should use CEL with the full `ManagedCluster` object as context rather than fixed
  placeholders such as `{{CLUSTER_NAME}}`.
- **`forEach`-style iteration** over a list to generate repeated patches. Keeping `overrides`
  a list leaves syntactic room, but iteration source, variable scope and execution order are
  genuinely undecided; see [Open Questions](#open-questions).
- **Overriding anything outside the manifest payload.** `manifestConfigs`, `deleteOption` and
  the other `ManifestWorkSpec` fields are not patchable by this proposal.
- **Changing a manifest's identity.** A patch may not change `apiVersion`, `kind`, `name` or
  `namespace`; see [Identity invariant](#identity-invariant).
- **Selecting clusters by `Placement` or `ManagedClusterSet`.** Only a label selector in the
  first iteration; see [Alternatives](#alternatives).

## Proposal

### User Stories

#### Story 1: node pool differs per cluster

As a platform engineer, I distribute one Deployment to twenty clusters. Each cluster tags its
application node pool with a `nodepool` label on its `ManagedCluster`. I write one
`ManifestOverridePolicy` per distinct node pool value and the right `nodeSelector` is applied
to every workload distributed from my namespace, including ones added later.

#### Story 2: more replicas in production

As an application owner, I want `replicas: 5` on clusters labelled `env: prod` and the
template's `replicas: 2` everywhere else, without maintaining two copies of the Deployment.

#### Story 3: a shared override reused by many workloads

As a platform engineer, every workload my team distributes must carry a compliance label and
tolerate the `dedicated` taint on clusters in `region: eu`. I define that once, and it holds
for every MWRS in the namespace rather than being pasted into each one.

#### Story 4: patching a custom resource

As a platform engineer, I distribute a Knative `Service` and need to add one environment
variable to its container on clusters in `region: east`, leaving the image and the sidecar
container intact. I register a `ResourceOverrideSchema` for `serving.knative.dev/v1 Service`
once, and strategic merge patches on that type then merge the container list by name instead
of replacing it.

### API Design

#### ManifestOverridePolicy

Namespaced. Applies to every `ManifestWorkReplicaSet` in the same namespace.

```go
// ManifestOverridePolicy defines patches applied to manifests distributed by the
// ManifestWorkReplicaSets in the same namespace, for the clusters it selects.
type ManifestOverridePolicy struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   ManifestOverridePolicySpec   `json:"spec"`
    Status ManifestOverridePolicyStatus `json:"status,omitempty"`
}

type ManifestOverridePolicySpec struct {
    // Priority orders policies against each other. Policies are applied in ascending
    // priority order, so a higher priority is applied later and wins a conflict.
    // Policies with equal priority are ordered by name, so the result is always
    // deterministic.
    // +kubebuilder:default=0
    // +optional
    Priority int32 `json:"priority,omitempty"`

    // ResourceSelectors selects the manifests this policy patches. A manifest is selected
    // if it matches any selector. An empty list selects no manifest.
    // Name and namespace accept "*" to match any value, as elsewhere in ManifestWork.
    // +kubebuilder:validation:MinItems=1
    ResourceSelectors []workv1.ResourceIdentifier `json:"resourceSelectors"`

    // Overrides are evaluated in list order. Every override whose clusterSelector matches
    // the cluster contributes its patches, in order.
    // +kubebuilder:validation:MinItems=1
    Overrides []Override `json:"overrides"`
}

type Override struct {
    // ClusterSelector selects clusters by ManagedCluster labels.
    // An empty selector matches every cluster selected by the MWRS placement.
    // +optional
    ClusterSelector metav1.LabelSelector `json:"clusterSelector,omitempty"`

    // Patches are applied in list order to each selected manifest.
    // +kubebuilder:validation:MinItems=1
    Patches []Patch `json:"patches"`
}

type Patch struct {
    // Type is the patch format.
    // +kubebuilder:validation:Enum=JSONPatch;JSONMergePatch;StrategicMergePatch
    Type PatchType `json:"type"`

    // Patch is the patch body, as an embedded object or array rather than a string,
    // so that it is schema-checked and readable in the resource.
    // +kubebuilder:pruning:PreserveUnknownFields
    Patch runtime.RawExtension `json:"patch"`

    // AtomicLists controls what happens when a StrategicMergePatch would replace a list
    // declared atomic, or a list whose merge semantics cannot be determined from the
    // available schema.
    // "Reject" (default) fails the patch rather than silently discarding entries.
    // "Replace" permits the replacement, for users who intend it.
    // Ignored for JSONPatch and JSONMergePatch.
    // +kubebuilder:validation:Enum=Reject;Replace
    // +kubebuilder:default=Reject
    // +optional
    AtomicLists AtomicListPolicy `json:"atomicLists,omitempty"`
}
```

`status` records the policy's observed generation and a condition summarising whether it
rendered successfully everywhere it applied.

#### ResourceOverrideSchema

Cluster-scoped, administrator-managed, at most one per target GVK so that schema selection is
unambiguous.

```go
// ResourceOverrideSchema supplies list-merge metadata for a resource type, so that
// strategic merge patches can merge lists instead of replacing them. It affects override
// rendering only: it does not modify any CRD and does not affect API server validation.
type ResourceOverrideSchemaSpec struct {
    // Target is the resource type this schema describes.
    Target ResourceTypeRef `json:"target"`

    // OpenAPIV3Schema is a partial schema overlay. Merge metadata given here takes
    // precedence; anything not mentioned is inherited from the type's own schema.
    // +kubebuilder:pruning:PreserveUnknownFields
    OpenAPIV3Schema runtime.RawExtension `json:"openAPIV3Schema"`
}

type ResourceTypeRef struct {
    APIVersion string `json:"apiVersion"`
    Kind       string `json:"kind"`
}
```

Example, declaring that a Knative `Service`'s containers and their environment variables
merge by `name`:

```yaml
apiVersion: work.open-cluster-management.io/v1alpha1
kind: ResourceOverrideSchema
metadata:
  name: knative-service
spec:
  target:
    apiVersion: serving.knative.dev/v1
    kind: Service
  openAPIV3Schema:
    properties:
      spec:
        properties:
          template:
            properties:
              spec:
                properties:
                  containers:
                    type: array
                    x-kubernetes-patch-strategy: merge
                    x-kubernetes-patch-merge-key: name
                    items:
                      properties:
                        env:
                          type: array
                          x-kubernetes-patch-strategy: merge
                          x-kubernetes-patch-merge-key: name
```

This follows the pattern Kustomize documents for custom resources in its
[custom OpenAPI schema example][kustomize-openapi].

### Example

```yaml
apiVersion: work.open-cluster-management.io/v1alpha1
kind: ManifestOverridePolicy
metadata:
  name: nodepool-per-cluster
  namespace: team-a
spec:
  priority: 10
  resourceSelectors:
    - group: apps
      resource: deployments
      namespace: "*"
      name: "*"
  overrides:
    - clusterSelector:
        matchLabels:
          nodepool: apps-a
      patches:
        - type: JSONMergePatch
          patch:
            spec:
              template:
                spec:
                  nodeSelector:
                    nodepool: apps-a
    - clusterSelector:
        matchLabels:
          nodepool: apps-b
      patches:
        - type: JSONMergePatch
          patch:
            spec:
              template:
                spec:
                  nodeSelector:
                    nodepool: apps-b
```

The `ManifestWorkReplicaSet` is unchanged — no new field, no reference to the policy:

```yaml
apiVersion: work.open-cluster-management.io/v1alpha1
kind: ManifestWorkReplicaSet
metadata:
  name: web-app
  namespace: team-a
spec:
  manifestWorkTemplate:
    workload:
      manifests:
        - apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: web-app
            namespace: default
          spec:
            replicas: 2
            # ...
  placementRefs:
    - name: web-app-placement
```

### Why policies apply implicitly

An alternative is an `overridePolicies` list on the MWRS, mirroring `placementRefs`. The
thread settled against it: a governance override that application teams can forget to list is
not an override. Implicit application also matches how OCM's `Policy` framework and Kyverno
behave, which is the mental model users already have for "a rule the platform team imposes".

The cost is that an MWRS author cannot see, from the MWRS alone, what will be applied. That is
mitigated by recording provenance on the rendered `ManifestWork` and on MWRS status; see
[Observability](#observability).

### Implementation Details

#### A single renderer, used by all three call sites

This is a correctness constraint, not a preference. The template is copied in three places in
the MWRS controller, and two of them are drift detection:

| Site | Purpose |
|---|---|
| `manifestworkreplicaset_deploy_reconcile.go` (candidate build) | builds a candidate and calls `workapplier.ManifestWorkEqual` |
| `manifestworkreplicaset_status_reconcile.go` | the same comparison, for status accounting |
| `manifestworkreplicaset_deploy_reconcile.go` (create) | creates the `ManifestWork` |

If overrides were applied only at creation, both comparison sites would compare an
un-rendered template against a rendered live object. They would never agree. The cluster
would stay `ToApply`, `Progressive` rollout would keep selecting the same clusters,
`PlacementRolledOut` would never reach `Complete`, and the status counts would be wrong.

All three sites must therefore route through one function:

```go
func Render(template *workapiv1.ManifestWorkSpec, clusterLabels map[string]string,
    policies []Policy, mapper meta.RESTMapper) (*Result, error)
```

`Render` never mutates `template` and is deterministic. A prototype with unit tests for the
scenarios in [#198][issue-198] is in [ocm#1792][ocm-1792].

#### Ordering

1. Policies in the MWRS namespace are sorted by `(priority, name)`, ascending.
2. Within a policy, `overrides` are evaluated in list order; an override contributes only if
   its `clusterSelector` matches the cluster.
3. Within an override, `patches` are applied in list order.

Later patches see the result of earlier ones. Because the sort is total and every list is
ordered, two renders of the same inputs always produce the same bytes.

#### Resource selection

`resourceSelectors` reuses `ManifestWork`'s existing `ResourceIdentifier`
(group / resource / namespace / name), the same type `manifestConfigs` uses, with the same
`"*"` wildcard support for name and namespace. This keeps one resource-addressing concept in
the work API and lets the implementation reuse `helper.ResourceMatch`.

This requires mapping each manifest's `apiVersion`/`kind` to its resource, which the work hub
does not do today — it needs a `RESTMapper`. See [Open Questions](#open-questions); it is also
the reason `ResourceOverrideSchema` currently targets by `apiVersion`/`kind` instead, which is
an inconsistency the proposal should settle before it goes `implementable`.

#### Patch application

- **JSONPatch** — RFC 6902, via `github.com/evanphx/json-patch/v5`. v5 is used deliberately:
  v4 lets `replace` on a missing field succeed silently, v5 returns an error. `sdk-go`
  already uses v5.
- **JSONMergePatch** — RFC 7386.
- **StrategicMergePatch** — for built-in types the merge keys come from the Go types in
  `client-go`'s scheme. For custom resources they come from the type's CRD schema, overlaid
  with any `ResourceOverrideSchema` registered for that GVK.

For custom resources the implementation uses typed conversion plus merge from
`sigs.k8s.io/structured-merge-diff`, which is already vendored in OCM, rather than
`strategicpatch`. This is the approach Kubernetes' own `MutatingAdmissionPolicy`
`ApplyConfiguration` takes. It is necessary because a raw CRD schema declares list semantics
with `x-kubernetes-list-type` and `x-kubernetes-list-map-keys`, which `strategicpatch` does
not read; handing it a CRD schema is not sufficient on its own. A composite
`x-kubernetes-list-map-keys` must be honoured in full and must never be silently reduced to
its first field.

When the available schema cannot support the requested merge — an atomic list, or a map list
whose keys cannot be honoured — the patch fails rather than silently replacing the list,
unless the patch sets `atomicLists: Replace`.

#### Identity invariant

After all patches are applied, the rendered manifest's `apiVersion`, `kind`, `name` and
`namespace` must equal the original's. A patch that changes any of them is an error. Without
this, a patch could retarget a manifest at a different object, orphan the previously applied
resource, and make `ManifestWork` status accounting meaningless.

#### Controller wiring

The MWRS controller gains two informers:

- `ManifestOverridePolicy`, to re-render when a policy changes.
- `ManagedCluster`, because label selectors need cluster labels, which `PlacementDecision`
  does not carry.

The `ManagedCluster` informer is the significant cost. It caches every `ManagedCluster` on the
hub and must re-render affected MWRS when labels change. Mitigations to measure before
committing: an informer `transform` that drops `managedFields` and `status`, and event
filtering so that only label changes enqueue work. See
[Risks and Mitigation](#risks-and-mitigation).

#### Failure handling

A render failure affects one cluster. The implementation:

- does not create or update that cluster's `ManifestWork`;
- reports the cluster to the rollout machinery as `Failed`, not as `ToApply`. This matters:
  a cluster stuck in `ToApply` never counts toward `maxFailures` and a `Progressive` rollout
  with a `progressDeadline` would stall instead of failing;
- sets a condition on the MWRS so a failure anywhere is visible without inspecting each
  cluster;
- includes the offending policy, manifest index, override index and patch index in the
  message. The prototype's `RenderError` already carries exactly these.

Clusters whose render succeeded are unaffected.

#### Observability

Two mechanisms, following the precedent of `ManagedClusterAddOn.status.configReferences`:

- Each rendered `ManifestWork` records the policies that patched it, so an operator looking at
  a `ManifestWork` can tell why it differs from the template.
- MWRS status summarises which policies applied and surfaces any per-cluster failures.

### Risks and Mitigation

- **`ManagedCluster` informer cost.** Memory grows with fleet size and label churn triggers
  re-render. Needs measurement at realistic fleet scale before this graduates. Mitigated by an
  informer `transform` and label-only event filtering.
- **Implicit application is surprising.** A workload can change because someone edited a
  policy the MWRS does not name. Mitigated by provenance on the `ManifestWork` and MWRS
  status, and by policies being namespaced so the blast radius is one namespace.
- **Silent data loss through list replacement.** The reason `atomicLists` defaults to
  `Reject`. Replacing a container list and losing a sidecar is the kind of failure that is
  discovered in production.
- **Two overlapping customisation mechanisms.** If templating lands later, users will have
  patches and templates to choose between, with a permanent documentation cost. Keeping
  templating out of this proposal limits the combined surface for now, but the interaction
  must be designed when templating is proposed.
- **A new CRD is a permanent commitment.** Install, RBAC, bundle packaging downstream,
  upgrade handling, QE matrix and support burden. This is the main argument for the
  MWRS-embedded alternative below.
- **Divergence from the `ManagedCluster` as source of truth.** A fleet-wide value such as a
  node pool name is a fact about the cluster and is already recorded on the `ManagedCluster`.
  Restating the cluster-to-value mapping inside policies duplicates it, and the duplicate can
  go stale. Templating would remove this duplication; until then, label selectors group
  clusters so that the mapping is written per distinct value, not per cluster.

## Design Details

### Open Questions

1. **Where do CRD schemas come from?** Reading a CRD schema requires the CRD on the hub, but
   a type such as Knative may only be installed on managed clusters. Options discussed on
   [#198][issue-198]:
   a. require a `ResourceOverrideSchema` and error otherwise — no new machinery, but the
      administrator must supply the schema;
   b. add discovery on the hub — only works for types installed on the hub;
   c. collect schemas from managed clusters and keep a content-hash-deduplicated cache on the
      hub, with small per-cluster references, as proposed by @kahirokunn. This handles clusters
      whose schemas genuinely differ for the same `apiVersion`/`kind`, but requires an
      agent-to-hub schema reporting channel that does not exist in `work` today, plus a refresh
      and garbage-collection story. That is arguably its own enhancement.
   This proposal currently assumes (a) with (c) as a follow-up, and this is the main
   unresolved question.

2. **One identifier scheme or two?** `resourceSelectors` uses `ResourceIdentifier`
   (group/resource) for consistency with `manifestConfigs` and per review feedback on
   [#155][pr-155]; `ResourceOverrideSchema.spec.target` uses `apiVersion`/`kind` because it
   describes a type rather than selecting instances, and because manifests carry
   `apiVersion`/`kind`. Using group/resource everywhere requires a `RESTMapper` on the work
   hub; using `apiVersion`/`kind` everywhere diverges from `manifestConfigs` inside the same
   API. Both are defensible; the proposal should not ship with the inconsistency unexplained.

3. **How is `forEach` added later, if at all?** Keeping `overrides` a list leaves room, but
   the loop boundary determines variable scope and execution order. Given
   `forEach: [{region: [east, west]}]` with two patches appending `--start=$region` and
   `--end=$region`, an override-level loop yields
   `["--start=east", "--end=east", "--start=west", "--end=west"]` while a patch-level loop
   yields `["--start=east", "--start=west", "--end=east", "--end=west"]`. Also undecided:
   what may drive iteration (literal lists, `ManagedCluster` data, fields in the matched
   manifest), and whether a loop variable may be used in `clusterSelector` — which in turn
   decides whether expansion happens before or after cluster matching. Recorded here so the
   templating enhancement inherits the questions rather than rediscovering them.

4. **Scope of the first implementation.** Built-in types with a clear error for custom
   resources, with `ResourceOverrideSchema` as a follow-up; or both together. Reviewers have
   expressed both preferences. The answer depends partly on question 1.

5. **Should policies be namespaced or cluster-scoped?** Namespaced is proposed, so that
   authorisation follows the namespace that already governs the MWRS. A platform team wanting
   a fleet-wide override must then create one policy per namespace.

### Test Plan

**Unit tests** on the renderer, which is pure and the natural place for most coverage:
each patch type; ordering across priority, overrides and patches; wildcard and non-matching
resource selectors; cluster selectors that match, do not match, and are empty; the identity
invariant; atomic-list reject and the explicit `Replace` opt-in; composite
`x-kubernetes-list-map-keys`; `ResourceOverrideSchema` overlay precedence over the CRD schema;
determinism, by rendering twice and comparing bytes. [ocm#1792][ocm-1792] already carries this
suite for the agreed subset.

**Integration tests** on the controller: that a policy change re-renders existing
`ManifestWork`s; that the two drift-detection sites agree with the rendered object, so no
churn occurs — a regression test for the failure mode described above; that a render failure
marks one cluster `Failed` and leaves others untouched; that a `Progressive` rollout with
`maxFailures` and `progressDeadline` reacts to a failing cluster instead of stalling; that
`ManagedCluster` label changes re-render.

**E2E tests**: a `ManifestWorkReplicaSet` plus a `ManifestOverridePolicy` producing different
payloads on two managed clusters, including a custom resource patched with a registered
`ResourceOverrideSchema`.

**Scale**: measure hub memory and reconcile latency with the `ManagedCluster` informer at
realistic fleet size, with and without the informer `transform`, before graduation.

### Graduation Criteria

#### Alpha

- `ManifestOverridePolicy` served at `v1alpha1` behind a feature gate, off by default.
- Renderer used by all three MWRS call sites; unit and integration tests above passing.
- Per-cluster failure reporting and provenance implemented.

#### Beta

- `ResourceOverrideSchema` and custom-resource strategic merge implemented, with open
  question 1 resolved.
- E2E coverage on a real multi-cluster setup.
- Scale measurements published for the `ManagedCluster` informer.
- Feature gate on by default.

#### GA

- No API changes for one release.
- User-facing documentation on the OCM website.
- Field experience with the implicit-application model, in particular whether provenance is
  sufficient in practice.

### Upgrade / Downgrade Strategy

Both resources are new and optional. An existing cluster needs no change on upgrade and
behaviour is unchanged until a `ManifestOverridePolicy` is created. On downgrade, or if the
feature gate is turned off, rendering reverts to the plain template; the next reconcile
rewrites each `ManifestWork` back to the un-overridden payload, which is a real workload
change and must be documented. Deleting the policies before downgrading produces the same
result through the normal reconcile path and is the recommended procedure.

### Version Skew Strategy

Rendering happens entirely on the hub, and the agent receives an ordinary `ManifestWork`. An
`n-2` klusterlet that knows nothing about this feature is unaffected. If option (c) in open
question 1 is adopted later, that will introduce a genuine hub/agent contract and will need
its own skew analysis.

## Implementation History

- 2026-10-07: [enhancements#198][issue-198] opened by @kahirokunn, from
  [ocm#1175][ocm-1175].
- 2026-10-13 to 2026-10-15: discussion converges on a separate override resource rather than
  fields on MWRS; @mikeshng rejects attaching overrides to `Placement` on the grounds that
  `Placement` must stay workload-agnostic.
- 2026-10-22 to 2026-10-23: implicit application, priority ordering, per-cluster failure
  isolation and provenance agreed; templating and `forEach` moved out of scope.
- 2026-10-30: renderer prototype opened as [ocm#1792][ocm-1792].
- 2026-11-02: atomic-list rejection agreed as the default; `ResourceOverrideSchema` and the
  cross-cluster schema cache proposed; this proposal opened.

## Drawbacks

The strongest argument against this proposal is cost against benefit. It adds two CRDs, a
second cluster-selection mechanism alongside `Placement`, and a `ManagedCluster` informer in
the work hub — and it still does not solve the per-copy-uniqueness need that started
[ocm#1175][ocm-1175], which needs the templating enhancement. A reviewer could reasonably
hold that the narrower MWRS-embedded alternative below delivers most of the value for a
fraction of the permanent surface, and that users who need more should use a GitOps tool.

The counter-argument is that overrides are genuinely shared across workloads — the point
@everpeace raised — and that a field inside MWRS cannot express sharing without being
restated in every MWRS, which is both laborious and a governance hole.

## Alternatives

### Overrides embedded in MWRS, keyed by clusterSet or placement

This is the direction of [#155][pr-155] and of review feedback on it, which suggested an
extensible `overrides` field with an `overrideStrategy` selecting between `clustersets` and
`placements`:

```yaml
overrides:
  overrideStrategy: placements
  placements:
    - name: aws-placement
      patch: ...
    - name: gke-placement
      patch: ...
```

*For:* near-zero cost — the reconciler already iterates `placementRefs` and holds the
`Placement` per iteration. No new CRD, no second selection mechanism, no new informer if
keyed by placement. Fields on an API that is still `v1alpha1`.

*Against:* overrides cannot be shared across `ManifestWorkReplicaSet`s, and nothing stops an
MWRS author from omitting one.

This alternative was not chosen because of the sharing and governance arguments, but it is
materially cheaper and the choice deserves an explicit decision rather than being settled by
the order in which the two threads happened to progress.

### Overrides referenced from Placement

@everpeace's original sketch added an `overridePolicies` list to `Placement`, with MWRS
continuing to reference only the `Placement`. Rejected because `Placement` is used for things
other than workload distribution — selecting clusters to audit or observe, for example — and
coupling it to workload concerns narrows it.

### Templating instead of patches

Make patch values reference cluster data: `{{CLUSTER_NAME}}`, or CEL over the `ManagedCluster`
object. This addresses both needs in the motivation and removes the need to enumerate
clusters or distinct values anywhere, which is the property that fails at fleet scale. OCM has
precedent: `pkg/addon/templateagent` substitutes `{{CLUSTER_NAME}}` with `fasttemplate`, and
the policy framework offers Go templating with `lookup` and `fromClusterClaim`.

Not an alternative so much as a complement, and the thread agreed to sequence it separately.
It is listed here because it weakens the case for sharing: much of what teams would want to
share is precisely a cluster-to-value mapping, and the architecturally correct home for that
is the `ManagedCluster` object rather than a list that restates it.

### Do nothing; direct users to GitOps

`ApplicationSet` over `PlacementDecision`s already solves per-cluster customisation for users
on Argo CD. Zero cost, and it genuinely serves a large population. It does not help anyone
using MWRS directly, which is the population MWRS exists for — more relevant now that the MWRS
feature gate is enabled by default.

## References

- [enhancements#198 — Feature request: Support cluster-specific resource overrides][issue-198]
- [enhancements#155 — Add proposal for cluster set level override in MWRS][pr-155]
- [ocm#1175 — the originating request][ocm-1175]
- [ocm#1792 — renderer prototype][ocm-1792]
- [Karmada OverridePolicy](https://karmada.io/docs/userguide/scheduling/override-policy/)
- [Kustomize custom OpenAPI schema][kustomize-openapi]

[issue-198]: https://github.com/open-cluster-management-io/enhancements/issues/198
[pr-155]: https://github.com/open-cluster-management-io/enhancements/pull/155
[ocm-1175]: https://github.com/open-cluster-management-io/ocm/issues/1175
[ocm-1792]: https://github.com/open-cluster-management-io/ocm/pull/1792
[kustomize-openapi]: https://github.com/kubernetes-sigs/kustomize/blob/master/examples/customOpenAPIschema.md
