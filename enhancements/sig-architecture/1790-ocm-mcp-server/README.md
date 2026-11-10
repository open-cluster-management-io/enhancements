# OCM MCP Server

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal adds an MCP (Model Context Protocol) server for Open Cluster
Management. MCP is an open standard that lets AI assistants (such as Claude,
Copilot or Cursor) call tools to read data and take actions.

The OCM MCP server runs against a hub cluster and exposes OCM concepts as
simple tools: list clusters, explain a Placement, find where a workload is
deployed, check add-on health, and so on. A user can then ask questions in
plain language, for example:

- "Which clusters are not healthy right now?"
- "Why was cluster-3 not selected by my placement?"
- "Where is my nginx deployment running, and is it available everywhere?"

The first version is **read-only**. Write actions come later, behind an
explicit opt-in flag.

## Motivation

OCM spreads information across many resources. To answer one question, a user
often has to read several objects and connect them by hand:

- To know why a cluster was not selected, you need the Placement, the
  ManagedClusterSetBindings, the cluster labels, claims and taints, and the
  prioritizer scores.
- To know why a workload is not running, you need to follow
  ManifestWorkReplicaSet → Placement → PlacementDecision → ManifestWork →
  status feedback on each cluster.

This is hard for new users and slow for experienced ones. AI assistants are
now a common way to work with Kubernetes, and generic Kubernetes MCP servers
already exist. But they do not understand OCM relationships, so they return
raw YAML and leave the user to connect the pieces.

An OCM-aware MCP server does that work once, in one place, using the same
logic OCM itself uses.

### Goals

- Provide an MCP server that understands OCM resources and how they connect.
- Start with read-only tools that answer the most common questions:
  cluster health, placement decisions, and workload status.
- Reuse existing OCM code where possible (for example the placement
  debugger endpoint) instead of re-implementing logic.
- Use only the permissions of the user's own kubeconfig. No extra privileges.
- Work with any MCP client, over stdio (local) and streamable HTTP
  (in-cluster).

### Non-Goals

- Building a chat UI or shipping an AI model. The server only exposes tools;
  the user brings their own MCP client.
- Changing any OCM API.
- Replacing `clusteradm` or the existing CLIs.
- Write actions in the first version.

## Proposal

Create a new Go binary, `ocm-mcp-server`, that connects to the hub cluster and
exposes a set of MCP tools grouped into toolsets.

### Where the code lives

The server starts as a project in the
[lab repo](https://github.com/open-cluster-management-io/lab), under the
`ai-assistance` subproject: `lab/ai-assistance/mcp-server/`. This follows the
maintainer guidance on
[ocm#1790](https://github.com/open-cluster-management-io/ocm/issues/1790):
all AI-related work goes under `ai-assistance` for now.

- It fits the subproject's
  [vendor-neutral rules](https://github.com/open-cluster-management-io/lab/blob/main/ai-assistance/VENDOR_NEUTRALITY.md):
  MCP is an open standard, and the server works with any MCP client. It has
  no AI-provider APIs, keys or provider-specific code.
- `ai-assistance` holds only markdown today (prompts and guides). The
  `mcp-server/` folder adds Go code, with its own `go.mod`, `Makefile` and
  `README.md`, using the shared lab CI workflows (like
  `fleetconfig-controller`). The `ai-assistance` README gets a short "MCP
  server" entry next to prompts and guides.
- Reviews go through the `ai-assistance` owners.
- If the project grows and the community agrees, it can move to its own repo
  later.

### Related work

This is a new upstream project. It does not port an existing downstream
component. Related projects:

| Project | What it does | How this proposal differs |
|---|---|---|
| [stolostron/search-mcp-server](https://github.com/stolostron/search-mcp-server) | ACM (downstream) Tech Preview. Searches resources across managed clusters using the ACM search database. | Needs ACM search, which upstream OCM does not have. This proposal reads OCM APIs on the hub and explains OCM logic (placement, ManifestWork rollout). The two can be used side by side. |
| [stolostron/acm-mcp-server](https://github.com/stolostron/acm-mcp-server) | ACM proof of concept (TypeScript). Runs kubectl commands and gives resource schemas. No commits since 2025-10. | Uses typed OCM tools instead of running kubectl, and reuses the placement debugger. |
| [yanmxa/multicluster-mcp-server](https://github.com/yanmxa/multicluster-mcp-server) | Personal project (Python). Reads resources on the hub and on managed clusters. No commits since 2025-07. | Focuses on OCM relationships instead of generic resource access. |
| [ocm-mcp-server/ocm-mcp-server](https://github.com/ocm-mcp-server/ocm-mcp-server) | Independent project outside the OCM community (Python). Focuses on write actions with policy, approval and audit. | Read-only first, lives in the OCM community, and reuses OCM code such as the placement debugger. |

Ideas from these projects (for example search-style filters and write
approval) can be reused where they fit.

### User Stories

#### Story 1: Check fleet health

As a platform admin, I ask "which clusters are unhealthy?". The assistant
calls `list_clusters` and `get_cluster`, and tells me that `cluster-7` has not
renewed its lease for 10 minutes and its `application-manager` add-on is
degraded.

#### Story 2: Understand a placement

As an app owner, my placement selects 2 clusters but I expected 3. I ask
"why was cluster-3 not selected?". The assistant calls `explain_placement` and
tells me cluster-3 has a `NoSelect` taint that my placement does not tolerate.

#### Story 3: Debug a workload rollout

As an app owner, I ask "is my app running everywhere?". The assistant calls
`diagnose_workload` on my ManifestWorkReplicaSet and tells me that 9 of 10
clusters report Available, and on `cluster-4` the Deployment has 0 ready
replicas according to status feedback.

#### Story 4: Try a placement before applying it

As an app owner, I describe a placement ("3 clusters in us-east, preferring
the ones with the most allocatable CPU"). The assistant drafts a Placement
with a label selector and the `ResourceAllocatableCPU` prioritizer, and calls
`dry_run_placement` to show which clusters it would select, before anything is
created.

The assistant can only draft what a Placement can express: labels, claims,
CEL selectors, taints/tolerations and prioritizers. OCM does not track free
(unused) CPU, so a request like "at least 8 CPUs free" cannot be enforced. In
that case the assistant says so and offers the closest supported option (for
example, ranking by allocatable CPU), instead of showing a result that looks
like it meets the request.

### Tools

All tools in the first version are read-only.

| Toolset | Tool | What it does |
|---|---|---|
| clusters | `list_clusters` | List ManagedClusters with status (Available, Joined, Accepted), labels and clusterset |
| clusters | `get_cluster` | One cluster in detail: conditions, lease, claims, taints, capacity, add-ons |
| clusters | `list_clustersets` | ManagedClusterSets and which namespaces they are bound to |
| placement | `list_placements` | Placements and how many clusters each one selected |
| placement | `explain_placement` | Per-cluster result: filtered out (and by which step) or selected (and with what score) |
| placement | `dry_run_placement` | Run a draft Placement through the scheduler without creating it |
| work | `list_works` | ManifestWorks on a cluster, or across all clusters, with status |
| work | `where_is_deployed` | Find which clusters a given resource (group, version, kind, namespace, name) is delivered to |
| work | `diagnose_workload` | Follow MWRS → Placement → decisions → ManifestWorks → feedback and report where it fails (needs the ManifestWorkReplicaSet API, see Version Skew Strategy) |
| addons | `list_addons` | ClusterManagementAddOns and their install strategy |
| addons | `get_addon_status` | Health of one add-on on each cluster, and the config in use |

Users can enable only the toolsets they need, for example
`--toolsets=clusters,placement`.

`where_is_deployed` matches on the full resource identity, because
`kind/namespace/name` alone can match resources from different API groups.
Group and version are optional inputs. If they are left out and more than one
resource matches, the tool returns an "ambiguous" result that lists each match
with its group and version, instead of merging them.

### Implementation Details/Notes/Constraints

**Language and libraries.** Go, using the official MCP Go SDK
(`github.com/modelcontextprotocol/go-sdk`) and the typed clients from
`open-cluster-management.io/api`.

**Connection.** The server reads a hub kubeconfig, like `kubectl` and
`clusteradm` do. Every call uses the user's own identity, so hub RBAC decides
what the user can see.

**Transports.**
- `stdio`: for local use with Claude Code, VS Code, Cursor and similar.
- `streamable HTTP`: for running on the hub as a shared service. How identity
  works in this mode:
  - The connection from the MCP client to the server must use TLS, because
    each request carries a bearer token. The server serves HTTPS by default.
    Plain HTTP is allowed only behind a trusted proxy that terminates TLS
    (for example an ingress or route), and only when this is set explicitly
    with a flag. The server never sends tokens over plain HTTP by default.
  - Each request must carry the caller's bearer token. Requests without a
    token are rejected.
  - The server does not check the token itself. It passes the token on to the
    hub API server (and to the placement debug endpoint), which authenticates
    it and applies RBAC as usual.
  - User calls never fall back to the server's own kubeconfig or service
    account.

**Placement tools.** The placement controller already serves a debug
endpoint at `/debug/placements/` (`pkg/placement/debugger`). It returns filter
results per step, prioritizer results and final scores, so answers always
match the real scheduler. The endpoint checks the caller's permissions itself.

**Cluster names in placement results.** The debug endpoint only checks
`get`/`create` on placements. Its response still lists every cluster it
looked at, including clusters that were filtered out, with their scores. A
user who can read Placements but not ManagedClusters would learn about
clusters they cannot otherwise see. To avoid this, the MCP server:
- Checks with a SelfSubjectAccessReview (using the caller's token) whether
  the caller can `list` ManagedClusters.
- If they can, returns the full result.
- If they cannot, returns only the selected clusters (which the caller can
  already see in PlacementDecisions). Filtered-out clusters and all scores
  are left out, and the tool says the result is limited by permissions.

The same gap exists in the debug endpoint itself, outside of MCP. It is
tracked in [ocm#1802](https://github.com/open-cluster-management-io/ocm/issues/1802).

**How the server reaches the debug endpoint.** What exists in OCM today:

- The endpoint is behind the `PlacementDebugServer` feature gate (alpha,
  off by default). It is turned on in the ClusterManager with
  `spec.placementConfiguration.featureGates`.
- When on, a `debug-server` sidecar runs in the placement controller pod and
  serves HTTPS on port `9443`, behind the ClusterIP Service
  `<cluster-manager>-placement` (by default
  `cluster-manager-placement.open-cluster-management-hub.svc`).
- The operator creates and rotates its TLS certificate
  (`placement-debug-serving-cert`); the CA is in `ca-bundle-configmap` in the
  same namespace.
- It checks the caller's bearer token with a TokenReview, then checks
  `get`/`create` on placements with a SubjectAccessReview.

So the MCP server needs **no OCM changes** to use it:

| Mode | How it connects |
|---|---|
| HTTP (on the hub) | Calls the Service directly, trusts the CA from `ca-bundle-configmap`, and forwards the caller's token. |
| stdio (local) | Opens a port-forward to the Service through the hub API server (like `kubectl port-forward`), sets the TLS server name to the Service hostname, and sends the user's token. This needs `pods/portforward` in the hub namespace. |

The API server's `services/proxy` path is not used, because it needs a broad
permission and the debug server would not see the caller's own identity.

If the feature gate is off, or the user cannot port-forward, the placement
tools say so clearly ("enable PlacementDebugServer" or "no access") and
`explain_placement` uses its partial fallback.

- `explain_placement` uses `GET /debug/placements/<namespace>/<name>`. The
  caller needs `get` on placements. If the endpoint cannot be reached, the
  tool falls back to reading the PlacementDecisions and clearly says the
  explanation is partial (it shows which clusters were selected, but not why
  others were not).
- `dry_run_placement` sends the draft Placement with `POST /debug/placements/`.
  The endpoint requires `create` on placements in that namespace, so read-only
  access is not enough for this tool. Nothing is created.
  - The draft is first validated with a server-side dry-run create
    (`dryRun=All`). An invalid draft is returned as an error, not as a result.
  - If the endpoint cannot be reached, the tool reports that dry-run is
    unavailable. It does not fall back to existing PlacementDecisions, because
    those belong to a different spec.
- If the endpoint returns an error, or returns no scheduling results, the tool
  reports an error instead of an empty result.

**Output format.** Tools return short, structured summaries (JSON with the key
fields), not full objects. This keeps responses small and easy for the
assistant to reason about. A `verbose` option can return the full object when
needed.

**Secret data is always hidden.** In both normal and `verbose` mode:
- Secret values are never returned. Only the Secret's name and keys are shown.
- Secrets inside ManifestWork payloads are redacted the same way.
- Status feedback values read through JSONPaths are redacted when the source
  resource is a Secret.

**Large fleets.** List tools support label selectors and a result limit, and
return a count plus a "truncated" flag, so a hub with thousands of clusters
does not produce huge responses.

### Future work (separate proposals or follow-ups)

- **Write tools** behind `--read-only=false`: accept a cluster, add labels,
  create a Placement or ManifestWork. Each write tool would support dry-run
  and show the object before creating it.
- **clusteradm integration**, for example `clusteradm mcp serve`.
- **MCP prompts** for common tasks, such as "debug my rollout".

### Risks and Mitigation

| Risk | Mitigation |
|---|---|
| An AI assistant changes the fleet by mistake | First version is read-only. Write tools are off by default and support dry-run. |
| The server sees more than the user should | It uses the caller's credentials only; hub RBAC applies to every call. Placement debug results are trimmed for users who cannot list ManagedClusters. |
| Bearer tokens are captured on the network (HTTP mode) | TLS is required on the client-to-server connection; plain HTTP only behind an explicitly trusted TLS-terminating proxy. |
| Sensitive data is sent to an AI provider | Tools return only status and metadata by default. Secret data is redacted in every mode, including inside ManifestWork payloads and status feedback. This is documented clearly. |
| Prompt injection through cluster data (for example a label value with instructions) | Tools return data as structured fields, and the docs recommend that clients ask for approval before any write tool. Read-only default limits the impact. |
| Placement debug endpoint is not reachable from the server | `explain_placement` gives a partial answer from PlacementDecisions; `dry_run_placement` reports "unavailable". |

Security review: OCM maintainers. UX review: early users through the OCM
community meeting and Slack.

## Design Details

### Open Questions

1. **Logins without a token.** The debug server checks bearer tokens only.
   Users who log in with a client certificate (for example the default kind
   admin kubeconfig) cannot use the placement tools. Is that acceptable for
   alpha, or should the debug server also accept client certificates?
2. **Packaging.** Should the hub mode be installed with a Helm chart from the
   `ai-assistance` subproject (like `fleetconfig-controller` does in lab), or
   only documented?

### Test Plan

- **Unit tests** for each tool using fake OCM clients.
- **Integration tests** with envtest for the tools that join several resources
  (`diagnose_workload`, `where_is_deployed`).
- **E2E tests** on kind with one hub and two managed clusters (same setup as
  the current OCM e2e). Tests call the tools through a real MCP client over
  stdio and check the results.
- **RBAC tests**: a user with limited permissions only sees allowed data.
  This includes a user who can read Placements but not ManagedClusters: the
  placement tools must not return filtered-out cluster names or scores.
- **TLS test** (HTTP mode): the server refuses to start on plain HTTP unless
  the trusted-proxy flag is set, and the deployed path (direct HTTPS, or
  behind the proxy) is tested end to end.
- **Feature gate tests**: `diagnose_workload` works when the
  ManifestWorkReplicaSet gate is on, and returns "not supported" when it is
  off.
- **Resource identity tests**: `where_is_deployed` returns an "ambiguous"
  result when the same kind/namespace/name exists in two API groups.
- **Identity isolation test** (HTTP mode): two callers with different tokens
  each see only their own allowed data, and a request without a token is
  rejected.
- **Redaction tests**: Secret data is hidden in normal and `verbose` mode,
  both in ManifestWork payloads and in status feedback.
- **Placement error tests**: invalid drafts, debug endpoint errors and an
  unreachable endpoint each give a clear error or "partial"/"unavailable"
  message.

### Graduation Criteria

**Alpha**
- Read-only toolsets: clusters, placement, work, addons.
- stdio transport.
- Docs on the OCM website with setup for common MCP clients.

**Beta**
- Streamable HTTP transport with per-user identity.
- Feedback from real users; tool names and output stable.
- E2E tests run in CI.

**GA**
- Used in production by at least a few adopters.
- Community decides whether it stays in `lab` or moves to its own repo.
- Write tools (if accepted) have been in beta for at least one release.

### Upgrade / Downgrade Strategy

The server is a separate client binary and does not change any OCM API or
controller. Upgrading or downgrading it does not affect the hub or the managed
clusters.

### Version Skew Strategy

The server uses these OCM APIs: `cluster.open-cluster-management.io/v1`,
`v1beta1`, `v1beta2`, `work.open-cluster-management.io/v1`,
`addon.open-cluster-management.io/v1alpha1`, and
`work.open-cluster-management.io/v1alpha1` for ManifestWorkReplicaSet.

ManifestWorkReplicaSet is an Alpha API behind the `ManifestWorkReplicaSet`
feature gate. The gate is on by default in recent OCM releases, but it can be
turned off, and older hubs may not serve the API. When the API is not served,
`diagnose_workload` returns "not supported on this hub" and the other work
tools keep working.

Older hubs are handled like this:
- **Missing resource:** the server uses API discovery at startup. If a
  resource is not served, the related tool returns "not supported on this
  hub".
- **Missing field:** where it matters, objects are read as unstructured data,
  so a field that is not present shows as "not set" instead of a zero value.
- **`Forbidden` responses** are reported as a permission error, not as
  "not supported".
- The placement debug endpoint is optional (see the placement tools section
  for what happens when it is not reachable).

## Implementation History

- 2026-10-30: Initial proposal. Tracking issue: https://github.com/open-cluster-management-io/ocm/issues/1790
- 2026-10-30: Code location set to `lab/mcp-server/`, following maintainer
  feedback on the tracking issue.
- 2026-10-30: Clarified HTTP identity, placement tool errors and permissions,
  Secret redaction, and version skew handling (review feedback).
- 2026-11-02: Described how the server reaches the existing placement debug
  server (feature gate, Service, TLS, auth). No OCM changes needed.
- 2026-11-03: Limited Story 4 to what a Placement can express, used full
  resource identity in `where_is_deployed`, required TLS in HTTP mode, hid
  cluster names and scores from users who cannot list ManagedClusters, and
  documented the ManifestWorkReplicaSet API dependency (review feedback).
- 2026-11-04: Moved the code location to `lab/ai-assistance/mcp-server/`,
  following maintainer feedback on the tracking issue. Added related work.

## Drawbacks

- One more component for the community to maintain.
- MCP and AI tooling change quickly, so the SDK and tool design may need
  updates more often than other OCM components.

## Alternatives

- **Use a generic Kubernetes MCP server only.** It works for reading single
  objects but does not understand how OCM resources connect, so users still
  have to do the joining by hand.
- **Improve `clusteradm` output only.** Useful, and the two ideas work well
  together, but it does not help users who work through AI assistants.
- **Document kubectl recipes for assistants.** Low cost, but answers depend on
  the assistant getting many steps right, and placement scores cannot be
  rebuilt correctly from outside the scheduler.
- **A new repo now** (`open-cluster-management-io/ocm-mcp-server`). Gives a
  separate release cycle, but is too heavy for an early idea. `lab` is the
  right place to start.
- **A `clusteradm` subcommand** (`clusteradm mcp serve`). Easy to install, but
  adds MCP dependencies to `clusteradm` and ties releases together. It can
  still be added later as a thin wrapper.
- **A separate `lab/mcp-server/` folder.** Keeps Go code apart from the
  markdown content, but maintainers asked to keep all AI-related work under
  `ai-assistance` for now.
- **An OCM toolset in `containers/kubernetes-mcp-server`.** Reaches existing
  users of that server, but puts OCM logic in a repo outside the OCM
  community.

## Infrastructure Needed

- A new `ai-assistance/mcp-server/` folder in the `lab` repo, covered by the
  `ai-assistance` owners.
- CI and image publishing through the existing lab workflows (test, e2e,
  release image to quay.io).
