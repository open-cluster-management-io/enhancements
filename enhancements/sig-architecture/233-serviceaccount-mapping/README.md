# Subject Mapping for Transparent Authentication

## Release Signoff Checklist

- [ ] Enhancement is implementable
- [ ] Design details are clear and agreed upon
- [ ] Test plan is defined
- [ ] Graduation criteria established
- [ ] User-facing documentation created

## Summary

This enhancement proposes a SubjectMapping API that enables transparent authentication for hub subjects accessing managed clusters. By mapping hub subjects to managed cluster service account names, applications and users can access managed clusters without managing tokens directly. The enhancement eliminates token storage on the hub, simplifies RBAC, and requires zero code changes to existing applications while maintaining security through short-lived tokens generated on-demand.

**Implementation Phases:**

- **Alpha/Beta**: ServiceAccount subject mapping (primary implementation)
- **Future**: User subject mapping (designed but not implemented initially)

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
- Provide cross-namespace ServiceAccount mapping on the hub
- Support managed clusters not using cluster-proxy
- Support mapping to Users or Groups on managed clusters (only ServiceAccounts as targets)

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

### Implementation Details

#### SubjectMapping CRD

A new cluster-scoped custom resource that specifies the mapping configuration:

**Example 1: ServiceAccount Mapping**
```yaml
apiVersion: authentication.open-cluster-management.io/v1alpha1
kind: SubjectMapping
metadata:
  name: argocd-cluster-access
spec:
  # Hub subject to map
  hubSubject:
    kind: ServiceAccount
    serviceAccount:
      namespace: argocd
      name: argocd-application-controller

  # Service account name to create on managed clusters
  managedServiceAccount:
    name: hub-argocd-sa

  # Token expiration duration (default: 3600 seconds)
  tokenExpirationSeconds: 3600
```

**Example 2: User Mapping with Placement (Future)**

> **Note**: User subject support is planned for future releases. Alpha/Beta releases will focus on ServiceAccount subjects only.

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
    name: hub-admin-user

  # Reference a Placement for cluster selection
  placementRef:
    name: production-clusters

  tokenExpirationSeconds: 3600
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: production-clusters
spec:
  clusterSets:
    - production
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchLabels:
            environment: production
```

The SubjectMapping controller will:
- Watch SubjectMapping resources
- Discover target clusters based on subject type:
  - **ServiceAccount subjects**: Via ManagedClusterSetBinding in the ServiceAccount's namespace
  - **User subjects**: Via referenced Placement and its PlacementDecisions
- Automatically create ManagedServiceAccount resources for each bound cluster
- Maintain status with creation progress and readiness conditions

#### Architecture Components

The SubjectMapping feature will be implemented in the **cluster-proxy** repository as it is fundamentally a transparent authentication feature built on top of cluster-proxy's existing proxy infrastructure.

**Implementation Location**: `open-cluster-management.io/cluster-proxy`

**1. SubjectMapping Controller (Hub-side)**

**Location**: `pkg/proxyserver/controllers/subjectmapping_controller.go`

**Deployment**: Runs in the cluster-proxy addon-manager on the hub cluster

Responsibilities:
- Watch SubjectMapping CRs and referenced Placements/PlacementDecisions
- Discover target clusters based on subject type:
  - **ServiceAccount**: Via ManagedClusterSetBinding in the ServiceAccount's namespace
  - **User**: Via referenced Placement's PlacementDecisions
- Create and maintain ManagedServiceAccount resources for each bound cluster (imports ManagedServiceAccount API from `managed-serviceaccount` repository)
- Update status conditions (Ready, PartiallyReady, Failed)
- Handle cluster binding/unbinding events (ManagedClusterSetBinding changes or PlacementDecision updates)

**2. Authentication Module (Spoke-side)**

**Location**: `pkg/proxyagent/authentication/`

**Deployment**: Runs in the proxy-agent on each managed cluster

The proxy-agent component will be enhanced with an authentication module to:
- Watch SubjectMapping CRs on the hub (via informer)
- Intercept requests from mapped hub subjects
- Extract subject information from request authentication:
  - **ServiceAccount**: Extract from `system:serviceaccount:<namespace>:<name>` user
  - **User**: Extract from user identity (e.g., `admin@example.com`)
- Resolve hub subject to managed cluster service account names via SubjectMapping lookup
- Request tokens from the local TokenRequest API
- Cache tokens for performance (keyed by hub subject + managed SA name)
- Inject tokens transparently into Authorization headers
- Forward authenticated requests to the managed cluster API server

**Repository Structure**:
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

#### High-Level Architecture

The following diagram illustrates the complete SubjectMapping flow for transparent authentication:

```
┌─────────────────────────────────────────────────────────────────┐
│                         Hub Cluster                             │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 1. Admin Creates SubjectMapping                          │  │
│  │    namespace: argocd                                     │  │
│  │    hubServiceAccount: argocd-hub-sa                      │  │
│  │    managedServiceAccount: argocd-spoke-sa                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 2. Controller watches SubjectMapping                     │  │
│  │    - Finds ManagedClusterSetBinding in argocd namespace  │  │
│  │    - Discovers bound clusters (cluster1...cluster100)    │  │
│  │    - Creates ManagedServiceAccount for each cluster:     │  │
│  │      * argocd/cluster1 → argocd-spoke-sa                │  │
│  │      * argocd/cluster2 → argocd-spoke-sa                │  │
│  │      * ...                                                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 3. ArgoCD App (using argocd-hub-sa)                      │  │
│  │    Makes request via cluster-proxy user-server           │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                            │
│                     ▼                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 4. cluster-proxy user-server (HTTP proxy)                │  │
│  │    - Receives HTTP request from ArgoCD                   │  │
│  │    - Forwards through tunnel to proxy-agent              │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                            │
└─────────────────────┼────────────────────────────────────────────┘
                      │ cluster-proxy gRPC tunnel
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Managed Cluster (cluster1)                   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 5. proxy-agent (spoke-side)                              │  │
│  │    - Watches SubjectMapping on hub                      │  │
│  │    - Receives request via tunnel                         │  │
│  │    - Detects caller: argocd-hub-sa (from request auth)   │  │
│  │    - Looks up mapping in local cache                     │  │
│  │      argocd/argocd-hub-sa → argocd-spoke-sa             │  │
│  │    - Checks token cache (miss)                           │  │
│  │    - Calls LOCAL TokenRequest API                        │  │
│  │      POST /api/v1/namespaces/argocd/serviceaccounts/     │  │
│  │           argocd-spoke-sa/token                          │  │
│  │    - Caches token (55 minutes)                           │  │
│  │    - Injects token into Authorization header             │  │
│  │    - Forwards to local API server                        │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                            │
│                     ▼                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 6. Kubernetes API Server                                 │  │
│  │    - Receives request with argocd-spoke-sa token         │  │
│  │    - Processes request                                   │  │
│  │    - Returns response                                    │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                            │
│                     ▼                                            │
│                  Response                                        │
│                     │                                            │
└─────────────────────┼────────────────────────────────────────────┘
                      │ back through tunnel
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Hub Cluster                             │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 7. ArgoCD App                                            │  │
│  │    - Receives response from cluster1                     │  │
│  │    - No token management code needed!                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

This diagram shows how SubjectMapping enables transparent authentication through automatic token resolution and caching within the cluster-proxy architecture.

#### Authentication Flow

1. **Setup Phase:**
   - Admin creates SubjectMapping on the hub
   - For User subjects: Admin also creates a Placement resource (or references existing one)
   - Controller discovers bound clusters:
     - ServiceAccount: via ManagedClusterSetBinding in the ServiceAccount's namespace
     - User: via Placement's PlacementDecisions
   - ManagedServiceAccount resources are automatically created for each cluster

2. **Request Phase:**
   - Hub application (e.g., ArgoCD) makes request to managed cluster via cluster-proxy
   - Request includes hub ServiceAccount token in Authorization header
   - cluster-proxy user-server forwards request through gRPC tunnel to proxy-agent
   - proxy-agent on managed cluster receives the request

3. **Token Validation and Resolution Phase:**
   - proxy-agent extracts token from Authorization header
   - proxy-agent calls **hub apiserver TokenReview API** to validate the token:

     ```go
     POST /apis/authentication.k8s.io/v1/tokenreviews
     Spec: {Token: "<hub-token>"}
     ```

   - Hub apiserver validates token and returns user identity:

     ```go
     Status: {
       Authenticated: true,
       User: {
         Username: "system:serviceaccount:argocd:argocd-sa",
         Groups: ["system:serviceaccounts", "system:authenticated"]
       }
     }
     ```

   - proxy-agent extracts subject identity from TokenReview response
   - proxy-agent looks up SubjectMapping to find the mapped service account name
   - For ServiceAccount subjects: match `username` against `system:serviceaccount:{namespace}:{name}` format

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
  - Time-based expiration (80% of token lifetime)
  - SubjectMapping deletion or update
  - ManagedServiceAccount deletion

This distributed caching approach:
- Scales horizontally (each spoke caches independently)
- Reduces managed cluster API server load
- Maintains security through short-lived tokens
- Provides fast request handling after initial token generation
- Supports multiple subject types transparently

#### Cluster Discovery

The controller determines which clusters should receive the mapped service account based on subject type:

**For ServiceAccount Subjects:**

Uses ManagedClusterSetBinding in the ServiceAccount's namespace:

1. List all ManagedClusterSetBindings in the ServiceAccount's namespace
2. Extract ManagedClusterSet names from bindings
3. List all ManagedClusters with matching `cluster.open-cluster-management.io/clusterset` labels
4. Create/update ManagedServiceAccount for each discovered cluster

Example:
```yaml
# In argocd namespace
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSetBinding
metadata:
  name: production-clusters
  namespace: argocd
spec:
  clusterSet: production
---
# SubjectMapping will create ManagedServiceAccounts
# for all clusters in the "production" clusterset
```

**For User Subjects:**

Uses Placement API for cluster selection:

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
    name: hub-admin-user

  # Reference a Placement resource
  placementRef:
    name: admin-clusters
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: admin-clusters
spec:
  clusterSets:
    - production
    - staging
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: environment
              operator: In
              values: [production, staging]
```

The SubjectMapping controller:
1. Watches the referenced Placement resource
2. Reads PlacementDecisions created by the Placement controller
3. Extracts the list of selected clusters from PlacementDecisions
4. Creates/updates ManagedServiceAccount for each selected cluster

This approach leverages OCM's existing Placement API capabilities:
- Sophisticated cluster selection with predicates and priorities
- Dynamic cluster selection as PlacementDecisions update
- Consistent with Policy, ManifestWork, and other OCM resources

### Risks and Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| cluster-proxy complexity | Medium - adds authentication logic | - Isolated authentication module<br>- Comprehensive unit tests<br>- Feature flag for gradual rollout |
| Token cache memory usage | Low - distributed across spokes | - 55-minute expiration<br>- LRU eviction if needed<br>- Monitoring and alerts |
| Token request failures | Medium - temporary access disruption | - Retry with exponential backoff<br>- Fallback to synchronous token request<br>- Error logging and metrics |
| SubjectMapping misconfiguration | Low - wrong permissions granted | - Validation webhook<br>- Clear documentation<br>- RBAC examples |
| Backward compatibility | High - breaking existing workflows | - Coexist with ManagedServiceAccount<br>- Clear migration guide<br>- No forced migration |

## Design Details

### API Changes

**New API Group**: `authentication.open-cluster-management.io/v1alpha1`

```go
// SubjectMapping maps hub subjects (ServiceAccounts, Users) to managed cluster service accounts
type SubjectMapping struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   SubjectMappingSpec   `json:"spec,omitempty"`
    Status SubjectMappingStatus `json:"status,omitempty"`
}

type SubjectMappingSpec struct {
    // HubSubject specifies the subject on the hub cluster to map
    // +required
    // +kubebuilder:validation:XValidation:rule="self.hubSubject.kind == 'ServiceAccount' ? !has(self.placementRef) : has(self.placementRef)",message="placementRef is required for User subjects and must not be set for ServiceAccount subjects"
    HubSubject HubSubject `json:"hubSubject"`

    // ManagedServiceAccount specifies the service account to create on managed clusters
    // +required
    ManagedServiceAccount ManagedServiceAccountSpec `json:"managedServiceAccount"`

    // PlacementRef references a Placement resource for cluster selection.
    // Required for User subjects.
    // Ignored for ServiceAccount subjects (uses ManagedClusterSetBinding instead).
    // +optional
    PlacementRef *PlacementRef `json:"placementRef,omitempty"`

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

type HubSubject struct {
    // Kind is the type of subject (ServiceAccount or User)
    // +required
    // +kubebuilder:validation:Enum=ServiceAccount;User
    // +kubebuilder:validation:XValidation:rule="self.kind == 'ServiceAccount' ? has(self.serviceAccount) : true",message="serviceAccount is required when kind is ServiceAccount"
    // +kubebuilder:validation:XValidation:rule="self.kind == 'User' ? has(self.user) : true",message="user is required when kind is User"
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

type PlacementRef struct {
    // Name is the name of the Placement resource
    // +required
    // +kubebuilder:validation:MinLength=1
    Name string `json:"name"`

    // Namespace is the namespace of the Placement resource
    // Required since SubjectMapping is cluster-scoped
    // +required
    // +kubebuilder:validation:MinLength=1
    Namespace string `json:"namespace"`
}

type ManagedServiceAccountSpec struct {
    // Name of the service account to create on managed clusters
    // +required
    // +kubebuilder:validation:MinLength=1
    Name string `json:"name"`
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
- `Failed`: Unable to create ManagedServiceAccounts for any cluster
- `Conflicted`: Multiple SubjectMappings reference the same hub subject (see Conflict Resolution below)

### Subject Matching and Conflict Resolution

**Hub Token Validation:**

proxy-agent validates hub tokens using the existing cluster-proxy pattern ([reference implementation](https://github.com/open-cluster-management-io/cluster-proxy/blob/main/pkg/serviceproxy/service_proxy.go#L253-L265)):

```go
// Call hub apiserver TokenReview API
tokenReview, err := hubKubeClient.AuthenticationV1().TokenReviews().Create(ctx, &authenticationv1.TokenReview{
    Spec: authenticationv1.TokenReviewSpec{
        Token: token,
    },
}, metav1.CreateOptions{})

// Extract authenticated user identity
username := tokenReview.Status.User.Username
// Examples:
//   ServiceAccount: "system:serviceaccount:argocd:argocd-sa"
//   User: "admin@example.com" or "system:admin"
```

**Subject Matching Logic:**

For **ServiceAccount subjects** (Alpha/Beta implementation):

```go
expectedUsername := fmt.Sprintf("system:serviceaccount:%s:%s",
    mapping.Spec.HubSubject.ServiceAccount.Namespace,
    mapping.Spec.HubSubject.ServiceAccount.Name)

if username == expectedUsername {
    return mapping.Spec.ManagedServiceAccount.Name
}
```

For **User subjects** (Future implementation):

```go
if username == mapping.Spec.HubSubject.User.Name {
    return mapping.Spec.ManagedServiceAccount.Name
}
```

**Conflict Resolution:**

When multiple SubjectMappings map the same hub subject:

1. **Controller Detection**: Marks all conflicting mappings with `Conflicted` condition:

   ```yaml
   conditions:
   - type: Conflicted
     status: "True"
     reason: DuplicateHubSubject
     message: "ServiceAccount 'argocd/argocd-sa' also mapped by: mapping-b"
   ```

2. **Runtime Resolution**: proxy-agent selects the **first created** SubjectMapping by `creationTimestamp`
   - Deterministic and stable
   - Prevents name-based hijacking (attacker cannot override by choosing alphabetically earlier name)
   - Admin must delete first mapping to change it

3. **No Validation Webhook Required**: Conflicts detected by controller, visible in status

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

- First-created wins prevents name-based hijacking
- Admin must delete existing mapping to override
- All conflicts visible via `Conflicted` condition

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

### Test Plan

#### Unit Tests

**Alpha/Beta (ServiceAccount subjects):**

- SubjectMapping controller logic (cluster discovery, ManagedServiceAccount creation)
- ManagedClusterSetBinding watching for ServiceAccount subjects
- proxy-agent token resolution and caching for ServiceAccount subjects
- Token cache eviction and expiration
- Subject extraction from TokenReview response (ServiceAccount username format)
- Error handling for TokenRequest API failures
- Conflict detection for duplicate ServiceAccount subjects

**Future (User subjects):**

- Placement and PlacementDecision watching for User subjects
- proxy-agent token resolution for User subjects
- Subject extraction from TokenReview response (User username)

#### Integration Tests

**Alpha/Beta (ServiceAccount subjects):**

- SubjectMapping creation triggers ManagedServiceAccount creation
- ManagedClusterSetBinding changes trigger reconciliation
- Token generation and caching in proxy-agent for ServiceAccount subjects
- Token injection into requests
- Cache invalidation on SubjectMapping deletion
- Multiple SubjectMappings for different ServiceAccount subjects
- Conflict resolution (first-created wins)

**Future (User subjects):**

- User subject PlacementDecision changes trigger reconciliation
- Token generation for User subjects

#### E2E Tests

**Alpha/Beta (ServiceAccount subjects):**

- End-to-end request flow from hub application (ServiceAccount) to managed cluster
- Token lifecycle (generation, caching, expiration, refresh)
- Multiple concurrent requests using cached tokens
- SubjectMapping for clusters with different cluster-proxy versions
- ArgoCD integration scenario with ServiceAccount subject (100+ clusters)
- Token expiration and regeneration during long-running operations

**Future (User subjects):**

- End-to-end request flow from human user (User subject) to managed cluster
- User accessing clusters via kubectl with User subject mapping
- Placement predicate changes affecting cluster selection for User subjects

### Graduation Criteria

#### Alpha (cluster-proxy v0.x.0) - ServiceAccount subjects only

- [ ] SubjectMapping CRD in cluster-proxy repository (`pkg/apis/authentication/v1alpha1`)
- [ ] SubjectMapping controller implementation in addon-manager for ServiceAccount subjects
- [ ] ManagedClusterSetBinding-based cluster discovery
- [ ] Authentication module in proxy-agent for ServiceAccount token resolution and caching
- [ ] TokenReview-based hub token validation
- [ ] Conflict detection with status conditions (no webhook)
- [ ] Unit tests with >80% coverage
- [ ] Integration tests for ServiceAccount subject mapping
- [ ] Basic documentation and examples
- [ ] Feature flag for gradual rollout (optional, can be enabled by default in alpha)

#### Beta (cluster-proxy v0.x+1.0) - ServiceAccount subjects only

- [ ] E2E tests covering all ServiceAccount scenarios
- [ ] Performance testing with 100+ clusters
- [ ] Production-ready token caching implementation with metrics
- [ ] Comprehensive user documentation
- [ ] Migration guide from ManagedServiceAccount pattern
- [ ] Metrics and monitoring support (cache hit rate, token generation latency, conflict count, etc.)
- [ ] Conflict resolution testing and documentation
- [ ] ArgoCD integration example and guide

#### GA (cluster-proxy v1.x.0) - ServiceAccount subjects

- [ ] Used in production by at least 2 organizations
- [ ] No critical bugs in 2+ beta release cycles
- [ ] Security review completed
- [ ] Troubleshooting guide with common scenarios
- [ ] Performance benchmarks published (compared to ManagedServiceAccount)
- [ ] Stable API (move from v1alpha1 to v1)

#### Future Enhancements - User subjects

- [ ] Placement-based cluster discovery for User subjects
- [ ] User subject matching from TokenReview username
- [ ] E2E tests for User subject mapping
- [ ] kubectl integration examples

### Upgrade / Downgrade Strategy

**Upgrade:**
- SubjectMapping is additive; existing ManagedServiceAccount workflows are unaffected
- New CRD installed via operator upgrade
- SubjectMapping controller deployed alongside existing controllers
- proxy-agent enhancement is backward compatible (no token mapping if SubjectMapping doesn't exist)
- Applications can migrate incrementally by creating SubjectMapping resources

**Migration Path:**
1. Upgrade hub and managed cluster components
2. Create SubjectMapping for specific applications
3. Verify application functionality
4. Gradually migrate other applications
5. Optionally clean up old ManagedServiceAccount and Secret resources

**Downgrade:**
- If SubjectMapping is removed, applications fall back to existing authentication
- ManagedServiceAccount resources created by SubjectMapping controller persist
- Applications need to manage tokens directly again (previous behavior)
- No data loss; graceful degradation

### Version Skew Strategy

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

An alternative approach using an explicit, on-demand token generation API instead of transparent proxy-based authentication:

**Design Overview:**

- Introduce a CREATE-only `ManagedServiceAccountTokenRequest` resource
- Applications explicitly call the API to request short-lived tokens
- Aggregated API server handles requests and connects to managed clusters via cluster-proxy
- Tokens returned in API response (never stored persistently)
- Spec includes: `targetCluster`, `managedServiceAccount`, `expirationSeconds`, `audiences`, `boundObjectRef`
- Status returns: ephemeral `token` and `expirationTimestamp`

**Implementation Components:**

- Aggregated API server for `authentication.open-cluster-management.io/v1beta1`
- TokenRequest REST handler orchestrating the workflow
- Cluster-proxy client factory for managed cluster connections
- APIService registration with hub cluster

**Workflow:**

1. Application submits TokenRequest to hub cluster API
2. Aggregated API server validates permissions and parameters
3. Server connects to target managed cluster via cluster-proxy
4. Calls Kubernetes TokenRequest API on managed cluster
5. Returns short-lived token (hours-to-minutes validity) in response
6. Application uses token for managed cluster access

**Rejected because:**

- **Requires code changes**: Applications must explicitly call the TokenRequest API and handle token lifecycle
- **Not transparent**: Unlike SubjectMapping which uses existing ServiceAccount tokens transparently
- **Additional API server**: Requires deploying and managing an aggregated API server
- **Manual token management**: Applications responsible for token refresh and expiration handling
- **Less integration**: Doesn't leverage cluster-proxy's existing authentication flow

SubjectMapping provides a superior user experience by eliminating code changes and providing transparent authentication for supported subject types (ServiceAccount, User) through the existing cluster-proxy infrastructure.

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
- Imports `ManagedServiceAccount` API from `open-cluster-management.io/managed-serviceaccount`
- Imports `Placement` and `PlacementDecision` APIs from `open-cluster-management.io/api`
- Imports `ManagedClusterSetBinding` API from `open-cluster-management.io/api`

### Additional Infrastructure

- CI/CD pipeline updates in cluster-proxy repository
- Integration test environment with multiple clusters (10+)
- Performance test environment with 100+ clusters
- Documentation updates on open-cluster-management.io website
- Example configurations for common use cases (ArgoCD, GitOps, custom applications)
