# Subject Mapping for Transparent Authentication

## Release Signoff Checklist

- [ ] Enhancement is implementable
- [ ] Design details are clear and agreed upon
- [ ] Test plan is defined
- [ ] Graduation criteria established
- [ ] User-facing documentation created

## Summary

This enhancement proposes a SubjectMapping API that enables transparent authentication for hub subjects accessing managed clusters. By mapping hub subjects to managed cluster service account names, applications and users can access managed clusters without managing tokens directly. The enhancement eliminates token storage on the hub, simplifies RBAC, and requires zero code changes to existing applications while maintaining security through short-lived tokens generated on-demand.

## Motivation

The current ManagedServiceAccount implementation presents two significant challenges:

1. **Token Storage Security Risk**: Service account tokens are synchronized from managed clusters to the hub as Secrets, creating a centralized attack surface with long-lived credentials (360 days). This violates security best practices by concentrating high-value credentials in a single location.

2. **RBAC Complexity**: Hub components need Secret read permissions across multiple cluster namespaces. For example, ArgoCD accessing 100 managed clusters requires Secret read permissions in 100 namespaces, creating operational overhead and potential security gaps.

These challenges become more pronounced as cluster count scales and security requirements tighten.

### Goals

- Eliminate token storage on the hub cluster by generating tokens on-demand
- Simplify RBAC requirements for applications and users accessing managed clusters
- Support mapping for multiple subject types: ServiceAccounts and Users
- Provide transparent authentication requiring zero code changes to existing applications
- Automatically provision ManagedServiceAccounts for all bound clusters
- Maintain backward compatibility with existing ManagedServiceAccount resources
- Improve security posture through short-lived tokens and distributed token generation

### Non-Goals

- Replace ManagedServiceAccount API (both will coexist)
- Support authentication methods other than service account tokens on managed clusters
- Support managed clusters not using cluster-proxy
- Support mapping to Users or Groups on managed clusters (only ServiceAccounts as targets)
- Support x509 client certificate authentication for User subjects (Alpha supports bearer tokens only: ServiceAccount tokens, OIDC tokens)

## Proposal

### User Stories

#### Story 1: Simplified Multi-Cluster Application Access

As a platform administrator running ArgoCD on the hub cluster, I want ArgoCD to access 100+ managed clusters without needing Secret read permissions in 100 namespaces, so that I can maintain a simpler and more secure RBAC configuration.

#### Story 2: Enhanced Security Posture

As a security engineer, I want to eliminate long-lived service account tokens stored on the hub cluster, so that I can reduce the attack surface and comply with zero-trust security principles.

#### Story 3: Transparent Migration

As an application developer, I want to use SubjectMapping without changing my application code, so that I can improve security without development overhead.

#### Story 4: Dynamic Cluster Binding

As a platform operator, I want service account permissions to automatically extend to newly bound clusters, so that I don't need to manually provision access for each new cluster.

#### Story 5: Human User Access

As a cluster administrator, I want to grant my team members access to managed clusters through the hub without distributing individual kubeconfigs, so that I can centrally manage access and use short-lived credentials.

### Overview

SubjectMapping enables transparent authentication by mapping hub subjects (ServiceAccounts or Users) to managed cluster service accounts. The feature eliminates the need for applications to manage tokens directly - tokens are generated on-demand by the proxy-agent on managed clusters and cached for performance.

**Key Components:**
- **SubjectMapping Controller** (hub): Creates ManagedServiceAccount resources for bound clusters
- **proxy-agent** (spoke): Intercepts requests, validates hub tokens, generates managed cluster tokens, and injects them transparently

**How It Works:**

1. Administrator creates a SubjectMapping mapping a hub subject (ServiceAccount or User) to a managed cluster ServiceAccount name
2. Controller discovers bound clusters (via InferFromServiceAccount using ManagedClusterSetBinding, or explicit ClusterList) and creates ManagedServiceAccount resources
3. When the hub subject makes a request through cluster-proxy, the proxy-agent validates the token and generates a short-lived token for the mapped ServiceAccount on the managed cluster
4. Token is cached and injected transparently - the application requires zero code changes

#### Architecture Flow

The following diagram illustrates the complete end-to-end flow:

```
┌────────────────────────────────────────────────────────────────┐
│                         Hub Cluster                            │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 1. Admin Creates SubjectMapping                          │  │
│  │    argocd-hub-sa → argocd-sa-for-hub                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                 │
│                              ▼                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 2. Controller watches SubjectMapping                     │  │
│  │    - Finds ManagedClusterSetBinding in argocd namespace  │  │
│  │    - Discovers bound clusters (cluster1...cluster100)    │  │
│  │    - Creates ManagedServiceAccount in each cluster NS:   │  │
│  │      * cluster1/argocd-sa-for-hub (MSA in cluster1 NS)   │  │
│  │      * cluster2/argocd-sa-for-hub (MSA in cluster2 NS)   │  │
│  │      * ...                                               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 3. ArgoCD App (using argocd-hub-sa)                      │  │
│  │    Makes request via cluster-proxy user-server           │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                          │
│                     ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 4. cluster-proxy user-server (HTTP proxy)                │  │
│  │    - Receives HTTP request from ArgoCD                   │  │
│  │    - Forwards through tunnel to proxy-agent              │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                          │
└─────────────────────┼──────────────────────────────────────────┘
                      │ cluster-proxy gRPC tunnel
                      ▼
┌────────────────────────────────────────────────────────────────┐
│                    Managed Cluster (cluster1)                  │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 5. proxy-agent (spoke-side)                              │  │
│  │    - Watches SubjectMapping on hub                       │  │
│  │    - Receives request via tunnel                         │  │
│  │    - Detects caller: argocd-hub-sa (from request auth)   │  │
│  │    - Looks up mapping in local cache                     │  │
│  │      argocd/argocd-hub-sa → argocd-sa-for-hub            │  │
│  │    - Checks token cache (miss)                           │  │
│  │    - Calls LOCAL TokenRequest API                        │  │
│  │      POST /api/v1/namespaces/                            │  │
│  │           open-cluster-management-agent-addon/           │  │
│  │           serviceaccounts/argocd-sa-for-hub/token        │  │
│  │    - Caches token (55 minutes)                           │  │
│  │    - Injects token into Authorization header             │  │
│  │    - Forwards to local API server                        │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                          │
│                     ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 6. Kubernetes API Server                                 │  │
│  │    - Receives request with argocd-sa-for-hub token       │  │
│  │    - Processes request                                   │  │
│  │    - Returns response                                    │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                          │
│                     ▼                                          │
│                  Response                                      │
│                     │                                          │
└─────────────────────┼──────────────────────────────────────────┘
                      │ back through tunnel
                      ▼
┌────────────────────────────────────────────────────────────────┐
│                         Hub Cluster                            │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 7. ArgoCD App                                            │  │
│  │    - Receives response from cluster1                     │  │
│  │    - No token management code needed!                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

This diagram shows how SubjectMapping enables transparent authentication through automatic token resolution and caching within the cluster-proxy architecture.

### API Design

A new cluster-scoped custom resource that specifies the mapping configuration:

**Example 1: ServiceAccount Mapping (Automatic Discovery)**
```yaml
apiVersion: authentication.open-cluster-management.io/v1alpha1
kind: SubjectMapping
metadata:
  name: argocd-cluster-access
spec:
  hubSubject:
    kind: ServiceAccount
    serviceAccount:
      namespace: argocd
      name: argocd-hub-sa

  managedServiceAccount:
    name: argocd-sa-for-hub
    tokenExpirationSeconds: 3600

  # Optional: defaults to InferFromServiceAccount for ServiceAccount subjects
  # clusters:
  #   type: InferFromServiceAccount
  #   # Discovers clusters via ManagedClusterSetBindings in the 'argocd' namespace
```

For ServiceAccount subjects, the `clusters` field is **optional** and defaults to `InferFromServiceAccount`, which discovers clusters via ManagedClusterSetBindings in the ServiceAccount's namespace. For User subjects, the `clusters` field is **required** and must use `ClusterList` type.

**Example 2: User Mapping (Explicit Cluster List)**
```yaml
apiVersion: authentication.open-cluster-management.io/v1alpha1
kind: SubjectMapping
metadata:
  name: admin-user-access
spec:
  hubSubject:
    kind: User
    user:
      name: "admin@example.com"

  managedServiceAccount:
    name: admin-sa-for-hub
    tokenExpirationSeconds: 3600

  # User subjects must specify clusters explicitly
  clusters:
    type: ClusterList
    clusterList:
      clusters:
        - cluster1
        - cluster2
        - cluster3
```

The controller watches SubjectMappings, discovers clusters via the `clusters` field, and creates ManagedServiceAccount resources in each cluster's namespace. See [API Specification](#api-specification) for type definitions.

### Architecture and Components

The SubjectMapping feature will be implemented in the **cluster-proxy** repository as it is fundamentally a transparent authentication feature built on top of cluster-proxy's existing proxy infrastructure.

**Implementation Location**: `open-cluster-management.io/cluster-proxy`

#### 1. SubjectMapping Controller (Hub-side)

**Location**: `pkg/proxyserver/controllers/subjectmapping_controller.go`

**Deployment**: Runs in the cluster-proxy addon-manager on the hub cluster

**Responsibilities**:
- Watch SubjectMapping CRs, ManagedClusterSetBindings, and ManagedClusters
- Discover target clusters based on clusters field (InferFromServiceAccount or ClusterList)
- Create and maintain ManagedServiceAccount resources for each bound cluster
- Update SubjectMapping status conditions (Ready, PartiallyReady, Failed, Conflicted)
- Handle cluster binding/unbinding lifecycle

#### 2. Authentication Module (Spoke-side)

**Location**: `pkg/proxyagent/authentication/`

**Deployment**: Runs in the proxy-agent on each managed cluster

**Responsibilities**:
- Watch SubjectMapping CRs on the hub via informer
- Intercept requests from mapped hub subjects
- Validate hub tokens via TokenReview API
- Resolve hub subjects to managed cluster ServiceAccount names
- Generate tokens via local TokenRequest API
- Cache tokens for performance optimization
- Inject tokens into request Authorization headers

#### Repository Structure

```
cluster-proxy/
├── pkg/
│   ├── apis/
│   │   └── authentication/v1alpha1/      # NEW: SubjectMapping API
│   │       ├── subjectmapping_types.go
│   │       ├── groupversion_info.go
│   │       └── zz_generated.deepcopy.go
│   ├── proxyserver/controllers/
│   │   └── subjectmapping_controller.go  # NEW: SubjectMapping controller
│   └── proxyagent/authentication/        # NEW: Authentication module
│       ├── resolver.go                   # Subject → ServiceAccount mapping
│       ├── cache.go                      # Token caching
│       └── tokenrequest.go               # TokenRequest API client
└── cmd/
    ├── addon-manager/main.go             # Register SubjectMapping controller
    └── addon-agent/main.go               # Integrate authentication module
```

### Implementation Details

#### ManagedServiceAccount Creation

The SubjectMapping controller creates ManagedServiceAccount resources with a special annotation to prevent token synchronization to the hub:

```go
const (
    // Annotation to control token sync behavior
    AnnotationTokenProjection = "authentication.open-cluster-management.io/token-projection"

    TokenProjectionNone   = "none"   // Don't sync token to hub
    TokenProjectionSecret = "secret" // Sync token to hub (default)
)

// Example ManagedServiceAccount created by SubjectMapping controller
// IMPORTANT: ManagedServiceAccount is created in the cluster namespace (e.g., "cluster1"),
// NOT the subject's namespace. This follows OCM's standard pattern where cluster-specific
// resources live in cluster namespaces.
msa := &authv1beta1.ManagedServiceAccount{
    ObjectMeta: metav1.ObjectMeta{
        Name:      managedServiceAccountName,  // From spec.managedServiceAccount.name
        Namespace: clusterName,                // Cluster namespace (e.g., "cluster1", "cluster2")
        Annotations: map[string]string{
            AnnotationTokenProjection: TokenProjectionNone,
        },
        Labels: map[string]string{
            // Track which SubjectMapping created this MSA for lifecycle management
            "authentication.open-cluster-management.io/subject-mapping": subjectMappingName,
            "authentication.open-cluster-management.io/managed-by":      "subject-mapping-controller",
        },
    },
    Spec: authv1beta1.ManagedServiceAccountSpec{
        Rotation: authv1beta1.ManagedServiceAccountRotation{
            Validity: metav1.Duration{
                Duration: time.Duration(tokenExpirationSeconds) * time.Second,
            },
        },
    },
}
```

The managed-serviceaccount addon agent honors this annotation:

- **Missing annotation or `"secret"`**: Sync token to hub as Secret (default, backward compatible)
- **`"none"`**: Create ServiceAccount on managed cluster only, don't sync token to hub

This approach:

- Requires **no ManagedServiceAccount API changes**
- Is **fully backward compatible** (existing ManagedServiceAccounts continue working)
- Eliminates token storage on hub for SubjectMapping use case
- Allows managed-serviceaccount addon to support both use cases simultaneously

#### Lifecycle Management

Since SubjectMapping is cluster-scoped and ManagedServiceAccount is namespaced (in cluster namespaces), owner references cannot be used. The controller uses:

- **Finalizers** on SubjectMapping to ensure cleanup of all associated ManagedServiceAccounts before deletion
- **Labels** on ManagedServiceAccounts (see code example above) to track which SubjectMapping created them
- On SubjectMapping deletion, the controller deletes all ManagedServiceAccounts with matching labels across all cluster namespaces

This approach follows OCM patterns (consistent with ManifestWork, Placement, etc.).

#### Cluster Discovery

The controller discovers target clusters based on the `clusters` field:

- **InferFromServiceAccount** (default for ServiceAccount subjects): Lists ManagedClusterSetBindings in ServiceAccount's namespace, discovers all clusters in bound cluster sets
- **ClusterList** (required for User subjects): Uses explicitly listed cluster names

For each discovered cluster, the controller creates/updates a ManagedServiceAccount in that cluster's namespace.

#### Cluster Binding/Unbinding

The controller continuously reconciles cluster membership via watches on ManagedClusterSetBinding and ManagedCluster resources:

- **Cluster Added**: Creates ManagedServiceAccount in new cluster's namespace, updates status
- **Cluster Removed**: Deletes corresponding ManagedServiceAccount, updates status
- **Cleanup**: Orphaned resources removed during reconciliation

#### Authentication Flow

1. **Setup Phase:**
   - Admin creates SubjectMapping on the hub with appropriate `clusters` configuration
   - Controller discovers bound clusters based on `clusters` field
   - ManagedServiceAccount resources are automatically created for each cluster

2. **Request Phase:**
   - Hub subject (application or user) makes request to managed cluster via cluster-proxy
   - Request includes bearer token in Authorization header
     - ServiceAccount subjects: ServiceAccount token
     - User subjects: OIDC token or other bearer token (x509 client certs not supported in Alpha)
   - cluster-proxy user-server forwards request through gRPC tunnel to proxy-agent
   - proxy-agent on managed cluster receives the request

3. **Token Validation and Resolution Phase:**
   - proxy-agent extracts bearer token from Authorization header
   - proxy-agent calls **hub apiserver TokenReview API** to validate the token:

     ```go
     POST /apis/authentication.k8s.io/v1/tokenreviews
     Spec: {Token: "<bearer-token>"}
     ```

   - Hub apiserver validates token and returns user identity

   - **For ServiceAccount subjects:**
     ```go
     Status: {
       Authenticated: true,
       User: {
         Username: "system:serviceaccount:argocd:argocd-sa",
         Groups: ["system:serviceaccounts", "system:authenticated"]
       }
     }
     ```

   - **For User subjects** (example with OIDC token):
     ```go
     Status: {
       Authenticated: true,
       User: {
         Username: "admin@example.com",
         Groups: ["system:authenticated", "admins"]
       }
     }
     ```

   - proxy-agent extracts subject identity from TokenReview response
   - proxy-agent looks up SubjectMapping to find the mapped service account name
   - For ServiceAccount subjects: match `username` against `system:serviceaccount:{namespace}:{name}` format
   - For User subjects: match `username` against user name directly

4. **Token Generation Phase:**
   - proxy-agent checks token cache (keyed by hub subject + managed SA name)
   - If cache miss, proxy-agent requests token from **local managed cluster TokenRequest API**:

     ```go
     POST /api/v1/namespaces/{namespace}/serviceaccounts/{name}/token
     Spec: {ExpirationSeconds: 3600}
     ```

   - Token is cached for 80% of expiration time (e.g., 48 minutes for 1-hour tokens)

5. **Authentication Phase:**
   - proxy-agent injects the token into the Authorization header
   - Request is forwarded to the managed cluster API server
   - Managed cluster authenticates using the service account token

#### Token Caching Strategy

To optimize performance and reduce API server load:

- **Cache Key**: Hash of `{subjectKind}/{subjectIdentifier}/{managedServiceAccountName}`
  - ServiceAccount: `ServiceAccount/{namespace}/{name}/{managedSA}`
  - User: `User/{username}/{managedSA}`
- **Cache Duration**: 80% of token expiration time (e.g., 48 minutes for 1-hour tokens)
- **Cache Location**: In-memory on proxy-agent (distributed across spokes)
- **Cache Invalidation**:
  - **Time-based expiration**: Tokens automatically evicted after 80% of their lifetime
  - **SubjectMapping changes** (detected via informer watch on hub):
    - Deletion: Invalidate all cached tokens for the mapped subject
    - Spec updates that affect token generation:
      - `hubSubject.*` → invalidate (different source subject)
      - `managedServiceAccount.name` → invalidate (different target SA)
      - `tokenExpirationSeconds` → invalidate (new expiration policy)
      - `clusters.*` → invalidate (different cluster selection)
    - Spec updates that do NOT trigger invalidation:
      - Metadata-only updates (labels, annotations)
  - **Detection mechanism**: proxy-agent runs informer watching SubjectMapping resources on the hub cluster

This distributed caching approach:

- Scales horizontally (each spoke caches independently)
- Reduces managed cluster API server load
- Maintains security through short-lived tokens
- Provides fast request handling after initial token generation
- Supports multiple subject types transparently

### Risks and Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| cluster-proxy complexity | Medium - adds authentication logic | - Isolated authentication module<br>- Comprehensive unit tests<br>- Feature flag for gradual rollout |
| Token cache memory usage | Low - distributed across spokes | - 55-minute expiration<br>- LRU eviction if needed<br>- Monitoring and alerts |
| Token request failures | Medium - temporary access disruption | - Retry with exponential backoff<br>- Fallback to synchronous token request<br>- Error logging and metrics |
| SubjectMapping misconfiguration | Low - wrong permissions granted | - Validation webhook<br>- Clear documentation<br>- RBAC examples |
| Backward compatibility | High - breaking existing workflows | - Coexist with ManagedServiceAccount<br>- Clear migration guide<br>- No forced migration |

### API Specification

**New API Group**: `authentication.open-cluster-management.io/v1alpha1`

```go
// SubjectMapping maps hub subjects (ServiceAccounts, Users) to managed cluster service accounts
type SubjectMapping struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   SubjectMappingSpec   `json:"spec,omitempty"`
    Status SubjectMappingStatus `json:"status,omitempty"`
}

// +kubebuilder:validation:XValidation:rule="self.hubSubject.kind == 'User' ? has(self.clusters) : true",message="clusters is required for User subjects"
// +kubebuilder:validation:XValidation:rule="has(self.clusters) && self.clusters.type == 'InferFromServiceAccount' ? self.hubSubject.kind == 'ServiceAccount' : true",message="InferFromServiceAccount type can only be used with ServiceAccount subjects"
type SubjectMappingSpec struct {
    // HubSubject specifies the subject on the hub cluster to map
    // +required
    HubSubject HubSubject `json:"hubSubject"`

    // ManagedServiceAccount specifies the service account to create on managed clusters
    // +required
    ManagedServiceAccount ManagedServiceAccountSpec `json:"managedServiceAccount"`

    // Clusters specifies how to discover target clusters.
    // For ServiceAccount subjects: Optional, defaults to InferFromServiceAccount
    // For User subjects: Required, must be ClusterList
    // +optional
    Clusters *Clusters `json:"clusters,omitempty"`
}

// +kubebuilder:validation:XValidation:rule="self.kind == 'ServiceAccount' ? has(self.serviceAccount) : true",message="serviceAccount is required when kind is ServiceAccount"
// +kubebuilder:validation:XValidation:rule="self.kind == 'User' ? has(self.user) : true",message="user is required when kind is User"
type HubSubject struct {
    // Kind is the type of subject (ServiceAccount or User)
    // +required
    // +kubebuilder:validation:Enum=ServiceAccount;User
    Kind string `json:"kind"`

    // ServiceAccount specifies a ServiceAccount subject
    // Required when Kind is ServiceAccount
    // +optional
    ServiceAccount *ServiceAccountSubject `json:"serviceAccount,omitempty"`

    // User specifies a User subject
    // Required when Kind is User
    // +optional
    User *UserSubject `json:"user,omitempty"`
}

type ServiceAccountSubject struct {
    // Namespace of the ServiceAccount
    // +required
    // +kubebuilder:validation:MinLength=1
    Namespace string `json:"namespace"`

    // Name of the ServiceAccount
    // +required
    // +kubebuilder:validation:MinLength=1
    Name string `json:"name"`
}

type UserSubject struct {
    // Name is the user identity (e.g., "admin@example.com", "system:admin")
    // +required
    // +kubebuilder:validation:MinLength=1
    Name string `json:"name"`
}

// +kubebuilder:validation:XValidation:rule="self.type == 'InferFromServiceAccount' ? !has(self.clusterList) : true",message="InferFromServiceAccount type must not have clusterList field"
// +kubebuilder:validation:XValidation:rule="self.type == 'ClusterList' ? has(self.clusterList) : true",message="clusterList is required when type is ClusterList"
type Clusters struct {
    // Type specifies the cluster discovery method
    // +required
    // +kubebuilder:validation:Enum=InferFromServiceAccount;ClusterList
    Type string `json:"type"`

    // ClusterList specifies an explicit list of cluster names
    // Required when Type is ClusterList
    // +optional
    ClusterList *ClusterListReference `json:"clusterList,omitempty"`
}

type ClusterListReference struct {
    // Clusters is an explicit list of ManagedCluster names
    // +required
    // +kubebuilder:validation:MinItems=1
    Clusters []string `json:"clusters"`
}

type ManagedServiceAccountSpec struct {
    // Name of the service account to create on managed clusters.
    // The ServiceAccount will be created in the namespace where the managed-serviceaccount
    // addon agent is running on the managed cluster (typically 'open-cluster-management-agent-addon').
    // +required
    // +kubebuilder:validation:MinLength=1
    Name string `json:"name"`

    // TokenExpirationSeconds represents the seconds of a token to expire.
    // Default: 3600 (1 hour)
    // Minimum: 600 (10 minutes)
    // Maximum: 86400 (24 hours)
    // +optional
    // +kubebuilder:default=3600
    // +kubebuilder:validation:Minimum=600
    // +kubebuilder:validation:Maximum=86400
    TokenExpirationSeconds int64 `json:"tokenExpirationSeconds,omitempty"`
}

type SubjectMappingStatus struct {
    // Conditions represent the latest available observations of the mapping state
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`

    // TotalClusters is the total number of bound clusters
    // +optional
    TotalClusters int32 `json:"totalClusters,omitempty"`

    // ReadyClusters is the number of clusters with ready ManagedServiceAccount
    // +optional
    ReadyClusters int32 `json:"readyClusters,omitempty"`

    // FailedClusters is the number of clusters with failed ManagedServiceAccount
    // +optional
    FailedClusters int32 `json:"failedClusters,omitempty"`
}
```

**Condition Types:**
- `Ready`: All bound clusters have successfully created ManagedServiceAccounts
- `PartiallyReady`: Some clusters have created ManagedServiceAccounts, others pending or failed
- `Failed`: Unable to create ManagedServiceAccounts for any cluster, or no clusters discovered (e.g., InferFromServiceAccount finds no ManagedClusterSetBindings)
- `Conflicted`: Multiple SubjectMappings reference the same hub subject (see Conflict Resolution below)

### Subject Matching and Conflict Resolution

**Hub Token Validation:**

proxy-agent validates hub tokens using the existing cluster-proxy TokenReview pattern ([reference](https://github.com/open-cluster-management-io/cluster-proxy/blob/main/pkg/serviceproxy/service_proxy.go#L253-L265)). The hub apiserver returns the authenticated user identity:

- ServiceAccount: `system:serviceaccount:<namespace>:<name>`
- User: `admin@example.com` or `system:admin`

**Subject Matching:**

- **ServiceAccount**: Match TokenReview username against `system:serviceaccount:{namespace}:{name}` format
- **User**: Match TokenReview username against user name directly

**Conflict Resolution:**

When multiple SubjectMappings map the same hub subject:

1. **Controller Detection**:
   - Winner (first-created): Gets `Ready` condition when all clusters are ready
   - Losers (later-created): Get `Conflicted` status condition with message identifying the winning SubjectMapping by name and creation timestamp
2. **Runtime Resolution**: proxy-agent selects the first created SubjectMapping using deterministic ordering:
   - Sort by `creationTimestamp` (earliest wins)
   - Tiebreaker: `metadata.uid` (lexicographically smaller)
   - First-created SubjectMapping wins **globally across all clusters**
   - All ManagedServiceAccounts from the winner take effect; all from losers are ignored
   - Even if losers target non-overlapping clusters, they are completely ignored
   - Prevents name-based hijacking
   - Admin must delete first mapping to change it
3. **No Webhook Required**: Conflicts detected asynchronously via status conditions

### Security Considerations

**Authentication:**

- Hub tokens validated via TokenReview API (prevents forgery)
- Subject identity from TokenReview response (prevents header injection)
- Uses existing cluster-proxy TokenReview pattern

**Authorization:**

- **SubjectMapping RBAC**: Restrict create/update/delete to cluster-admins only
  - Creating SubjectMapping grants managed cluster access
- **Managed Cluster Control**: Managed cluster admins control actual SA permissions via RoleBindings
  - SubjectMapping only creates ManagedServiceAccount resource
  - No validation of "appropriate" permissions

**Token Caching:**

- Tokens cached in proxy-agent memory for 80% of expiration time
- Compromised proxy-agent exposes tokens for ~48 minutes (default)
- Mitigation: Short token lifetimes (default 1 hour), distributed caching (per-cluster)

**Conflict Resolution Security:**

- First-created wins (by `creationTimestamp`, with `UID` tiebreaker) prevents name-based hijacking
- Deterministic resolution across all proxy-agents ensures consistent behavior
- Admin must delete existing mapping to override
- All conflicts visible via `Conflicted` condition in status
- No webhook required - simpler operations, no single point of failure

**Required RBAC:**

Hub (SubjectMapping creation):

```yaml
# Recommended: cluster-admins only
apiGroups: ["authentication.open-cluster-management.io"]
resources: ["subjectmappings"]
verbs: ["create", "update", "delete"]
```

Hub proxy-agent (TokenReview):

```yaml
# Already exists in cluster-proxy
apiGroups: ["authentication.k8s.io"]
resources: ["tokenreviews"]
verbs: ["create"]
```

Managed cluster proxy-agent (TokenRequest):

```yaml
apiGroups: [""]
resources: ["serviceaccounts/token"]
verbs: ["create"]
```

## Test Plan

### Unit Tests

- SubjectMapping controller: cluster discovery, ManagedServiceAccount creation, finalizers
- Cluster discovery: InferFromServiceAccount (ManagedClusterSetBinding), ClusterList
- proxy-agent: token resolution, caching, cache invalidation, TokenReview integration
- Conflict detection and resolution logic

### Integration Tests

- SubjectMapping lifecycle: creation, updates, deletion with cleanup
- ManagedClusterSetBinding changes trigger reconciliation for InferFromServiceAccount
- Token generation, caching, and injection into requests
- Conflict resolution: first-created wins with UID tiebreaker, status conditions

### E2E Tests

- Complete request flow: hub application/user → cluster-proxy → managed cluster
- Token lifecycle: generation, caching, expiration, refresh
- Multi-cluster scenarios (100+ clusters)
- ArgoCD integration with ServiceAccount subjects

## Graduation Criteria

### Alpha (cluster-proxy v0.x.0) - ServiceAccount subjects only

- [ ] **managed-serviceaccount addon enhancement** to honor `authentication.open-cluster-management.io/token-projection` annotation
  - [ ] Agent skips token sync when annotation value is `"none"`
  - [ ] Backward compatible (missing annotation defaults to existing behavior)
  - [ ] Unit and integration tests in managed-serviceaccount repository
- [ ] SubjectMapping CRD in cluster-proxy repository (`pkg/apis/authentication/v1alpha1`)
- [ ] SubjectMapping controller implementation in addon-manager for ServiceAccount subjects
  - [ ] Creates ManagedServiceAccount with `token-projection: none` annotation
- [ ] ManagedClusterSetBinding-based cluster discovery
- [ ] Authentication module in proxy-agent for ServiceAccount token resolution and caching
- [ ] TokenReview-based hub token validation
- [ ] Conflict detection with status conditions (no webhook)
- [ ] Unit tests with >80% coverage
- [ ] Integration tests for ServiceAccount subject mapping
- [ ] Basic documentation and examples
- [ ] Feature flag for gradual rollout (optional, can be enabled by default in alpha)

### Beta (cluster-proxy v0.x+1.0) - ServiceAccount subjects only

- [ ] E2E tests covering all ServiceAccount scenarios
- [ ] Performance testing with 100+ clusters
- [ ] Production-ready token caching implementation with metrics
- [ ] Comprehensive user documentation
- [ ] Migration guide from ManagedServiceAccount pattern
- [ ] Metrics and monitoring support (cache hit rate, token generation latency, conflict count, etc.)
- [ ] Conflict resolution testing and documentation
- [ ] ArgoCD integration example and guide

### GA (cluster-proxy v1.x.0) - ServiceAccount subjects

- [ ] Used in production by at least 2 organizations
- [ ] No critical bugs in 2+ beta release cycles
- [ ] Security review completed
- [ ] Troubleshooting guide with common scenarios
- [ ] Performance benchmarks published (compared to ManagedServiceAccount)
- [ ] Stable API (move from v1alpha1 to v1)

### Future Enhancements

**Additional cluster discovery types:**
- [ ] ManagedClusterSet type: Reference cluster-scoped ManagedClusterSets
- [ ] ClusterSelector type: Label-based cluster selection (more flexible than ClusterList)
- [ ] Placement type: Advanced cluster selection with priorities, spreading, etc.

**User subject enhancements:**
- [ ] x509 client certificate authentication support for User subjects (Alpha supports bearer tokens only)

**Other enhancements:**
- [ ] Group subject support (map entire user groups)
- [ ] kubectl integration examples

## Upgrade / Downgrade Strategy

**Upgrade:**
- SubjectMapping is additive; existing ManagedServiceAccount workflows are unaffected
- **managed-serviceaccount addon** must be upgraded to support `token-projection` annotation
  - Fully backward compatible: existing ManagedServiceAccounts (without annotation) continue syncing tokens
  - New ManagedServiceAccounts created by SubjectMapping won't sync tokens
- New SubjectMapping CRD installed via cluster-proxy operator upgrade
- SubjectMapping controller deployed alongside existing controllers
- proxy-agent enhancement is backward compatible (no token mapping if SubjectMapping doesn't exist)
- Applications can migrate incrementally by creating SubjectMapping resources

**Migration Path:**
1. Upgrade managed-serviceaccount addon (both hub and managed clusters)
2. Upgrade cluster-proxy (both hub and managed clusters) with SubjectMapping support
3. Create SubjectMapping for specific applications
4. Verify application functionality
5. Gradually migrate other applications
6. Optionally clean up old ManagedServiceAccount and Secret resources created manually

**Downgrade:**
- If SubjectMapping is removed, applications fall back to existing authentication
- ManagedServiceAccount resources created by SubjectMapping controller persist
- Applications need to manage tokens directly again (previous behavior)
- No data loss; graceful degradation

## Version Skew Strategy

**Hub-Spoke Version Compatibility:**

| Hub Version | Spoke Version | SubjectMapping Support |
|-------------|---------------|------------------------|
| v0.11.0+    | v0.11.0+      | Full support           |
| v0.11.0+    | v0.10.x       | No support (graceful degradation) |
| v0.10.x     | v0.11.0+      | No impact (feature not enabled) |

**Behavior with Version Skew:**
- If proxy-agent doesn't support SubjectMapping, requests fail with clear error message
- SubjectMapping controller sets condition to indicate incompatible spoke versions
- Applications must ensure all target clusters have compatible proxy-agent versions
- Recommendation: Upgrade spokes before enabling SubjectMapping

**Kubernetes Version Requirements:**
- Hub: Kubernetes 1.20+ (stable TokenRequest API)
- Managed Clusters: Kubernetes 1.20+ (stable TokenRequest API)
- cluster-proxy: v0.11.0+ (SubjectMapping support)

## Implementation History

- 2026-03-05: Initial proposal created

## Drawbacks

1. **Increased cluster-proxy Complexity**: Adds authentication logic to proxy-agent, increasing maintenance burden
2. **Kubernetes Version Dependency**: Requires Kubernetes 1.20+ for stable TokenRequest API
3. **cluster-proxy Dependency**: Only works with clusters using cluster-proxy for hub communication
4. **Token Lifetime Trade-offs**: Short-lived tokens increase token generation overhead
5. **Debugging Complexity**: Transparent token injection may complicate troubleshooting authentication issues
6. **Cache Memory Usage**: Token caching consumes memory on proxy-agent (mitigated by distributed caching)

## Alternatives

### ManagedServiceAccountTokenRequest API

An alternative approach using an explicit API for on-demand token generation: applications call a `ManagedServiceAccountTokenRequest` API to get short-lived tokens. The aggregated API server connects to managed clusters via cluster-proxy and returns ephemeral tokens.

**Rejected because:**
- Requires application code changes (explicit API calls, token lifecycle management)
- Not transparent (unlike SubjectMapping which works with existing ServiceAccount tokens)
- Additional infrastructure (aggregated API server deployment and management)
- Less integrated with cluster-proxy's existing authentication flow

SubjectMapping provides transparent authentication without code changes by leveraging cluster-proxy infrastructure.

## Infrastructure Needed

### Implementation Repository

The SubjectMapping feature will be implemented in the **cluster-proxy** repository:
- Repository: `https://github.com/open-cluster-management-io/cluster-proxy`
- API Group: `authentication.open-cluster-management.io/v1alpha1`
- Feature is self-contained within cluster-proxy (similar to the ClusterProfile feature gate)

**Rationale for cluster-proxy:**
- SubjectMapping is fundamentally a transparent authentication feature for cluster-proxy
- Both hub controller and spoke authentication logic belong together
- Enables atomic development and versioning of the complete feature
- Follows OCM patterns where components contain both hub and spoke code (e.g., addon-framework, work)

**Dependencies:**

1. **managed-serviceaccount addon** (runtime dependency):
   - SubjectMapping controller creates ManagedServiceAccount CRs annotated with `authentication.open-cluster-management.io/token-projection: none`
   - managed-serviceaccount addon agent must be enhanced to honor this annotation:
     - When annotation is `"none"`: Create ServiceAccount on managed cluster, skip token sync to hub
     - When annotation is missing or `"secret"`: Create ServiceAccount and sync token to hub (existing behavior)
   - **No API changes required** - fully backward compatible
   - Both use cases supported simultaneously (traditional ManagedServiceAccount + SubjectMapping)

2. **API Imports**:
   - Imports `ManagedServiceAccount` API from `open-cluster-management.io/managed-serviceaccount`
   - Imports `ManagedClusterSetBinding` and `ManagedCluster` APIs from `open-cluster-management.io/api`

### Additional Infrastructure

- CI/CD pipeline updates in cluster-proxy repository
- Integration test environment with multiple clusters (10+)
- Performance test environment with 100+ clusters
- Documentation updates on open-cluster-management.io website
- Example configurations for common use cases (ArgoCD, GitOps, custom applications)
