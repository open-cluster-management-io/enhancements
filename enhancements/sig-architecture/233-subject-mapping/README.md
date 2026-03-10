# ServiceAccount Mapping for Transparent Authentication

## Release Signoff Checklist

- [ ] Enhancement is implementable
- [ ] Design details are clear and agreed upon
- [ ] Test plan is defined
- [ ] Graduation criteria established
- [ ] User-facing documentation created

## Summary

This enhancement proposes a SubjectMapping API that enables transparent authentication for hub ServiceAccounts accessing managed clusters. By mapping hub ServiceAccounts to managed cluster service account names, applications can access managed clusters without managing tokens directly. The enhancement eliminates token storage on the hub, simplifies RBAC, and requires zero code changes to existing applications while maintaining security through short-lived tokens generated on-demand.

## Motivation

The current ManagedServiceAccount implementation presents two significant challenges:

1. **Token Storage Security Risk**: Service account tokens are synchronized from managed clusters to the hub as Secrets, creating a centralized attack surface with long-lived credentials (360 days). This violates security best practices by concentrating high-value credentials in a single location.

2. **RBAC Complexity**: Hub components need Secret read permissions across multiple cluster namespaces. For example, ArgoCD accessing 100 managed clusters requires Secret read permissions in 100 namespaces, creating operational overhead and potential security gaps.

3. **ClusterProfile Plugin Limitations**: While OCM supports the [ClusterProfile credentials plugin](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/5339-clusterprofile-plugin-credentials) ([issue #1163](https://github.com/open-cluster-management-io/ocm/issues/1163)) which provides exec-based credential retrieval, it actually exacerbates the token storage security risk by copying token Secrets from cluster namespaces to the ClusterProfile namespace, creating additional copies of long-lived credentials on the hub. Additionally, applications must install and configure exec plugins, and still require Secret read permissions in the ClusterProfile namespace, adding operational complexity.

These challenges become more pronounced as cluster count scales and security requirements tighten.

### Goals

- Eliminate token storage on the hub cluster by generating tokens on-demand
- Simplify RBAC requirements for applications accessing managed clusters
- Provide transparent authentication requiring zero code changes to existing applications
- Automatically provision ManagedServiceAccounts for all bound clusters via ManagedClusterSetBinding discovery
- Maintain backward compatibility with existing ManagedServiceAccount resources
- Improve security posture through short-lived tokens and distributed token generation
- Prevent duplicate mappings through namespace-scoped API design

### Non-Goals

- Replace ManagedServiceAccount API (both will coexist)
- Support authentication methods other than service account tokens on managed clusters
- Support managed clusters not using cluster-proxy
- Support mapping to Users or Groups on managed clusters (only ServiceAccounts as targets)
- Support mapping User or Group subjects on the hub (ServiceAccount subjects only)

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

### Overview

SubjectMapping enables transparent authentication by mapping hub ServiceAccounts to managed cluster service accounts. The feature eliminates the need for applications to manage tokens directly - tokens are generated on-demand by the proxy-agent on managed clusters and cached for performance.

**Key Components:**
- **SubjectMapping Controller** (hub): Creates ManagedServiceAccount resources for bound clusters
- **proxy-agent** (spoke): Intercepts requests, validates hub tokens, generates managed cluster tokens, and injects them transparently

**How It Works:**

1. Administrator creates a SubjectMapping in the same namespace as the hub ServiceAccount
2. Controller automatically discovers bound clusters via ManagedClusterSetBindings in that namespace and creates ManagedServiceAccount resources
3. When the hub ServiceAccount makes a request through cluster-proxy, the proxy-agent validates the token and generates a short-lived token for the mapped ServiceAccount on the managed cluster
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
│  │ 2. Controller watches SubjectMapping (argocd namespace)  │  │
│  │    - Automatically discovers clusters via                │  │
│  │      ManagedClusterSetBindings in argocd namespace       │  │
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
│  │    - Watches ManagedServiceAccount in cluster namespace  │  │
│  │    - Receives request via tunnel                         │  │
│  │    - Detects caller: argocd-hub-sa (from request auth)   │  │
│  │    - Lists ManagedServiceAccounts with annotations:      │  │
│  │      hub-namespace=argocd, hub-sa-name=argocd-hub-sa     │  │
│  │    - Finds mapped SA: argocd-sa-for-hub                  │  │
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

A new namespace-scoped custom resource that specifies the mapping configuration:

**Example: ServiceAccount Mapping**
```yaml
apiVersion: authentication.open-cluster-management.io/v1alpha1
kind: SubjectMapping
metadata:
  name: argocd-hub-sa      # Must match hubSubject.serviceAccount.name
  namespace: argocd         # ServiceAccount's namespace
spec:
  hubSubject:
    kind: ServiceAccount
    serviceAccount:
      name: argocd-hub-sa  # Must match metadata.name
  managedServiceAccount:
    name: argocd-sa-for-hub
    tokenExpirationSeconds: 3600
```

The SubjectMapping name must match the ServiceAccount name specified in `hubSubject.serviceAccount.name` (enforced by CEL validation). The controller automatically discovers clusters via ManagedClusterSetBindings in the SubjectMapping's namespace and creates mapping records in each cluster's namespace.

**Multiple SubjectMappings (from the same or different namespaces) can reference the same managed ServiceAccount name**, enabling shared service accounts. Access control is enforced via ValidatingAdmissionPolicy checking RBAC on the virtual subresource `subjectmappings/managedserviceaccounts` (see [Virtual Subresource for Access Control](#virtual-subresource-for-access-control)).

See [API Specification](#api-specification) for type definitions.

### Architecture and Components

The SubjectMapping feature will be implemented in the **cluster-proxy** repository as it is fundamentally a transparent authentication feature built on top of cluster-proxy's existing proxy infrastructure.

**Implementation Location**: `open-cluster-management.io/cluster-proxy`

#### 1. SubjectMapping Controller (Hub-side)

**Location**: `pkg/proxyserver/controllers/subjectmapping_controller.go`

**Deployment**: Runs in the cluster-proxy addon-manager on the hub cluster

**Responsibilities**:
- Watch SubjectMapping CRs, ManagedClusterSetBindings, and ManagedClusters
- Discover target clusters via ManagedClusterSetBindings in the SubjectMapping's namespace
- Create and maintain ManagedServiceAccount resources for each bound cluster
- Update SubjectMapping status conditions (Ready, PartiallyReady, Failed)
- Handle cluster binding/unbinding lifecycle

#### 2. Authentication Module (Spoke-side)

**Location**: `pkg/proxyagent/authentication/`

**Deployment**: Runs in the proxy-agent on each managed cluster

**Responsibilities**:
- Watch ManagedServiceAccount resources in cluster namespace for cache invalidation
- Intercept requests from mapped hub subjects
- Validate hub tokens via TokenReview API
- Resolve hub subjects to managed cluster ServiceAccount names via annotation-based lookup
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

#### Virtual Subresource for Access Control

To control which managed ServiceAccount names a SubjectMapping can use, we introduce a virtual subresource with a custom verb:

**Virtual Subresource**: `subjectmappings/managedserviceaccounts`
**Custom Verb**: `use`

Administrators can grant RBAC permissions to control which namespaces can use specific managed ServiceAccount names:

```yaml
# Example: Allow argocd namespace to use specific managed SA names
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argocd-managed-sa-usage
  namespace: argocd
rules:
- apiGroups: ["authentication.open-cluster-management.io"]
  resources: ["subjectmappings/managedserviceaccounts"]
  verbs: ["use"]
  resourceNames: ["argocd-prod", "argocd-staging", "shared-admin"]
```

A ValidatingAdmissionPolicy enforces this permission when SubjectMapping is created or updated (requires Kubernetes 1.28+):

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: subjectmapping-managed-sa-authorization
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: ["authentication.open-cluster-management.io"]
      apiVersions: ["v1alpha1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["subjectmappings"]
  validations:
  - expression: |
      authorizer.group('authentication.open-cluster-management.io')
        .resource('subjectmappings')
        .subresource('managedserviceaccounts')
        .name(object.spec.managedServiceAccount.name)
        .namespace(object.metadata.namespace)
        .check('use').allowed()
    message: "User does not have 'use' permission for managed ServiceAccount name"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: subjectmapping-managed-sa-authorization-binding
spec:
  policyName: subjectmapping-managed-sa-authorization
  validationActions: [Deny]
```

This uses CEL's `authorizer` function to check RBAC permissions declaratively, without requiring a webhook deployment. It enables fine-grained control over managed ServiceAccount name usage while allowing multiple SubjectMappings to reference the same name when authorized.

#### ManagedServiceAccount Creation with Multiple Hub Identities

The SubjectMapping controller creates or updates ManagedServiceAccount resources to store multiple hub identity mappings in an annotation. Multiple SubjectMappings can reference the same managed ServiceAccount name, with each maintaining its own token expiration setting.

```go
const (
    // Annotations for SubjectMapping feature
    AnnotationTokenProjection = "authentication.open-cluster-management.io/token-projection"
    AnnotationHubIdentities   = "authentication.open-cluster-management.io/hub-identities"

    TokenProjectionNone   = "none"   // Don't sync token to hub
    TokenProjectionSecret = "secret" // Sync token to hub (default)
)

// HubIdentity represents a single hub ServiceAccount mapping
type HubIdentity struct {
    Namespace          string `json:"namespace"`
    Name               string `json:"name"`
    TokenExpiration    int64  `json:"tokenExpiration"`
}

// Controller logic to add/update hub identity in ManagedServiceAccount
func (c *Controller) reconcileManagedServiceAccount(sm *authv1alpha1.SubjectMapping, clusterName string) error {
    msaName := sm.Spec.ManagedServiceAccount.Name

    // Get or create ManagedServiceAccount
    msa := &authv1beta1.ManagedServiceAccount{}
    err := c.client.Get(ctx, types.NamespacedName{
        Namespace: clusterName,
        Name:      msaName,
    }, msa)

    if apierrors.IsNotFound(err) {
        // Create new ManagedServiceAccount
        // Note: spec.rotation.validity is NOT set because token-projection: "none"
        // means no token Secret is created. proxy-agent generates tokens on-demand
        // using per-identity tokenExpiration from hub-identities annotation.
        msa = &authv1beta1.ManagedServiceAccount{
            ObjectMeta: metav1.ObjectMeta{
                Name:      msaName,
                Namespace: clusterName,
                Annotations: map[string]string{
                    AnnotationTokenProjection: TokenProjectionNone,
                },
                Labels: map[string]string{
                    "authentication.open-cluster-management.io/managed-by": "subject-mapping-controller",
                },
            },
        }
    }

    // Parse existing hub identities from annotation
    var hubIdentities []HubIdentity
    if existing := msa.Annotations[AnnotationHubIdentities]; existing != "" {
        json.Unmarshal([]byte(existing), &hubIdentities)
    }

    // Add or update this SubjectMapping's hub identity
    newIdentity := HubIdentity{
        Namespace:       sm.Namespace,
        Name:            sm.Spec.HubSubject.ServiceAccount.Name,
        TokenExpiration: sm.Spec.ManagedServiceAccount.TokenExpirationSeconds,
    }

    found := false
    for i, identity := range hubIdentities {
        if identity.Namespace == newIdentity.Namespace && identity.Name == newIdentity.Name {
            hubIdentities[i] = newIdentity
            found = true
            break
        }
    }
    if !found {
        hubIdentities = append(hubIdentities, newIdentity)
    }

    // Serialize back to annotation
    data, _ := json.Marshal(hubIdentities)
    msa.Annotations[AnnotationHubIdentities] = string(data)

    // Add label to track this SubjectMapping references this MSA
    labelKey := fmt.Sprintf("authentication.open-cluster-management.io/mapping-%s-%s",
        sm.Namespace, sm.Name)
    msa.Labels[labelKey] = "true"

    // Create or update
    if err == nil {
        return c.client.Update(ctx, msa)
    }
    return c.client.Create(ctx, msa)
}
```

**Example ManagedServiceAccount with multiple hub identities:**

```yaml
apiVersion: authentication.open-cluster-management.io/v1beta1
kind: ManagedServiceAccount
metadata:
  name: shared-admin
  namespace: cluster1
  annotations:
    authentication.open-cluster-management.io/token-projection: "none"
    authentication.open-cluster-management.io/hub-identities: |
      [
        {"namespace":"argocd","name":"argocd-hub-sa","tokenExpiration":3600},
        {"namespace":"gitops","name":"gitops-sa","tokenExpiration":7200}
      ]
  labels:
    authentication.open-cluster-management.io/managed-by: subject-mapping-controller
    # Track which SubjectMappings reference this MSA
    authentication.open-cluster-management.io/mapping-argocd-argocd-hub-sa: "true"
    authentication.open-cluster-management.io/mapping-gitops-gitops-sa: "true"
```

**How proxy-agent uses this:**

When proxy-agent receives a request from a hub ServiceAccount, it:

1. Validates the hub token via TokenReview and gets the hub ServiceAccount identity (namespace + name)
2. Lists ManagedServiceAccounts in its cluster namespace
3. For each ManagedServiceAccount, parses the `hub-identities` annotation
4. Finds the matching hub identity in the JSON array
5. Uses that identity's `tokenExpiration` when generating the token via TokenRequest API

The managed-serviceaccount addon agent honors the `token-projection` annotation:

- **Missing annotation or `"secret"`**: Sync token to hub as Secret (default, backward compatible)
- **`"none"`**: Create ServiceAccount on managed cluster only, don't sync token to hub

This approach:

- **Supports multiple hub identities** mapping to the same managed SA name
- **Each SubjectMapping maintains independent token expiration** settings
- Requires **no ManagedServiceAccount API changes**
- Is **fully backward compatible** (existing ManagedServiceAccounts continue working)
- Eliminates token storage on hub for SubjectMapping use case
- Allows managed-serviceaccount addon to support both use cases simultaneously

#### Lifecycle Management

Since SubjectMapping is namespace-scoped and ManagedServiceAccount is namespaced in different namespaces (SubjectMapping in hub ServiceAccount namespace, ManagedServiceAccount in cluster namespaces), owner references cannot be used for automatic cleanup. The controller uses:

- **Finalizers** on SubjectMapping to ensure proper cleanup before deletion
- **Labels** on ManagedServiceAccounts to track which SubjectMappings reference them
  - Label format: `authentication.open-cluster-management.io/mapping-{namespace}-{name}: "true"`
  - Example: `authentication.open-cluster-management.io/mapping-argocd-argocd-hub-sa: "true"`

**Cleanup Process on SubjectMapping Deletion:**

1. Controller lists all ManagedServiceAccounts with the SubjectMapping's tracking label
2. For each ManagedServiceAccount:
   - Parse the `hub-identities` annotation JSON array
   - Remove the hub identity entry matching this SubjectMapping's namespace/name
   - Remove the tracking label for this SubjectMapping
   - If `hub-identities` array is now empty:
     - Delete the ManagedServiceAccount (no SubjectMappings reference it)
   - Otherwise:
     - Update the ManagedServiceAccount with the modified annotation and labels
3. Remove finalizer from SubjectMapping

**Example cleanup:**

```go
func (c *Controller) cleanupManagedServiceAccount(sm *authv1alpha1.SubjectMapping, msa *authv1beta1.ManagedServiceAccount) error {
    // Parse hub identities
    var hubIdentities []HubIdentity
    json.Unmarshal([]byte(msa.Annotations[AnnotationHubIdentities]), &hubIdentities)

    // Remove this SubjectMapping's identity
    filtered := []HubIdentity{}
    for _, identity := range hubIdentities {
        if identity.Namespace != sm.Namespace || identity.Name != sm.Name {
            filtered = append(filtered, identity)
        }
    }

    // Remove tracking label
    labelKey := fmt.Sprintf("authentication.open-cluster-management.io/mapping-%s-%s",
        sm.Namespace, sm.Name)
    delete(msa.Labels, labelKey)

    // If no more identities, delete MSA; otherwise update
    if len(filtered) == 0 {
        return c.client.Delete(ctx, msa)
    }

    data, _ := json.Marshal(filtered)
    msa.Annotations[AnnotationHubIdentities] = string(data)
    return c.client.Update(ctx, msa)
}
```

This approach follows OCM patterns (consistent with ManifestWork, Placement, etc.) and supports shared ManagedServiceAccounts across multiple SubjectMappings.

#### Cluster Discovery

The controller discovers target clusters by listing ManagedClusterSetBindings in the SubjectMapping's namespace and discovering all clusters in the bound ManagedClusterSets.

For each discovered cluster, the controller creates/updates a ManagedServiceAccount in that cluster's namespace.

#### Cluster Binding/Unbinding

The controller continuously reconciles cluster membership via watches on ManagedClusterSetBinding and ManagedCluster resources:

- **Cluster Added**: Creates ManagedServiceAccount in new cluster's namespace, updates status
- **Cluster Removed**: Deletes corresponding ManagedServiceAccount, updates status
- **Cleanup**: Orphaned resources removed during reconciliation

#### Authentication Flow

1. **Setup Phase:**
   - Admin creates SubjectMapping in the same namespace as the hub ServiceAccount
   - Controller discovers bound clusters via ManagedClusterSetBindings in that namespace
   - ManagedServiceAccount resources are automatically created for each cluster

2. **Request Phase:**
   - Hub ServiceAccount makes request to managed cluster via cluster-proxy
   - Request includes ServiceAccount token in Authorization header
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
         Username: "system:serviceaccount:argocd:argocd-hub-sa",
         Groups: ["system:serviceaccounts", "system:authenticated"]
       }
     }
     ```

   - proxy-agent extracts ServiceAccount identity from TokenReview response
   - proxy-agent parses namespace and name from username (`system:serviceaccount:{namespace}:{name}` format)
   - proxy-agent lists ManagedServiceAccounts in its own cluster namespace with matching annotations:
     - `authentication.open-cluster-management.io/hub-namespace: argocd`
     - `authentication.open-cluster-management.io/hub-serviceaccount-name: argocd-hub-sa`
   - If found, proxy-agent reads the ManagedServiceAccount name from the result

4. **Token Generation Phase:**
   - proxy-agent checks token cache (keyed by hub ServiceAccount namespace/name + managed SA name)
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

- **Cache Key**: Hash of `{namespace}/{serviceAccountName}/{managedServiceAccountName}`
  - Example: `argocd/argocd-hub-sa/argocd-sa-for-hub`
- **Cache Duration**: 80% of token expiration time (e.g., 48 minutes for 1-hour tokens)
- **Cache Location**: In-memory on proxy-agent (distributed across spokes)
- **Cache Invalidation**:
  - **Time-based expiration**: Tokens automatically evicted after 80% of their lifetime
  - **ManagedServiceAccount changes** (detected via informer watch in cluster namespace):
    - Deletion: Invalidate cached tokens for the mapped ServiceAccount
    - Annotation updates that affect mapping:
      - `hub-namespace` or `hub-serviceaccount-name` changed → invalidate
      - `token-projection` changed → invalidate
  - **Detection mechanism**: proxy-agent runs informer watching ManagedServiceAccount resources in its own cluster namespace only

This distributed caching approach:

- Scales horizontally (each spoke caches independently)
- Reduces managed cluster API server load
- Maintains security through short-lived tokens
- Provides fast request handling after initial token generation

### Risks and Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| cluster-proxy complexity | Medium - adds authentication logic | - Isolated authentication module<br>- Comprehensive unit tests<br>- Feature flag for gradual rollout |
| Token cache memory usage | Low - distributed across spokes | - 55-minute expiration<br>- LRU eviction if needed<br>- Monitoring and alerts |
| Token request failures | Medium - temporary access disruption | - Retry with exponential backoff<br>- Fallback to synchronous token request<br>- Error logging and metrics |
| SubjectMapping misconfiguration | Low - wrong permissions granted | - ValidatingAdmissionPolicy enforcement<br>- CEL validation rules<br>- Clear documentation<br>- RBAC examples |
| Backward compatibility | High - breaking existing workflows | - Coexist with ManagedServiceAccount<br>- Clear migration guide<br>- No forced migration |

### API Specification

**New API Group**: `authentication.open-cluster-management.io/v1alpha1`

```go
// SubjectMapping maps a hub ServiceAccount to managed cluster service accounts.
// It is namespace-scoped and must be created in the same namespace as the ServiceAccount it maps.
// +kubebuilder:validation:XValidation:rule="self.hubSubject.kind == 'ServiceAccount' ? self.metadata.name == self.hubSubject.serviceAccount.name : true",message="metadata.name must match hubSubject.serviceAccount.name when kind is ServiceAccount"
type SubjectMapping struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   SubjectMappingSpec   `json:"spec,omitempty"`
    Status SubjectMappingStatus `json:"status,omitempty"`
}

type SubjectMappingSpec struct {
    // HubSubject specifies the ServiceAccount on the hub cluster to map.
    // The ServiceAccount namespace is implicit from the SubjectMapping's namespace.
    // +required
    HubSubject HubSubject `json:"hubSubject"`

    // ManagedServiceAccount specifies the service account to create on managed clusters
    // +required
    ManagedServiceAccount ManagedServiceAccountSpec `json:"managedServiceAccount"`
}

// +kubebuilder:validation:XValidation:rule="self.kind == 'ServiceAccount' ? has(self.serviceAccount) : true",message="serviceAccount is required when kind is ServiceAccount"
type HubSubject struct {
    // Kind is the type of subject. Only ServiceAccount is supported.
    // +required
    // +kubebuilder:validation:Enum=ServiceAccount
    Kind string `json:"kind"`

    // ServiceAccount specifies the ServiceAccount subject
    // Required when Kind is ServiceAccount
    // +optional
    ServiceAccount *ServiceAccountSubject `json:"serviceAccount,omitempty"`
}

type ServiceAccountSubject struct {
    // Name of the ServiceAccount in the SubjectMapping's namespace.
    // Must match the SubjectMapping's metadata.name.
    // +required
    // +kubebuilder:validation:MinLength=1
    Name string `json:"name"`
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
- `Failed`: Unable to create ManagedServiceAccounts for any cluster, or no clusters discovered (e.g., no ManagedClusterSetBindings found in namespace)

### Subject Matching

**Hub Token Validation:**

proxy-agent validates hub ServiceAccount tokens using the existing cluster-proxy TokenReview pattern ([reference](https://github.com/open-cluster-management-io/cluster-proxy/blob/main/pkg/serviceproxy/service_proxy.go#L253-L265)). The hub apiserver returns the authenticated user identity:

```
Username: "system:serviceaccount:<namespace>:<name>"
Groups: ["system:serviceaccounts", "system:authenticated"]
```

**ManagedServiceAccount Lookup:**

1. proxy-agent extracts namespace and name from TokenReview username (`system:serviceaccount:{namespace}:{name}` format)
2. proxy-agent lists ManagedServiceAccounts in **its own cluster namespace**
3. For each ManagedServiceAccount, proxy-agent:
   - Parses the `authentication.open-cluster-management.io/hub-identities` annotation as JSON array
   - Searches for an entry matching the hub ServiceAccount namespace and name
   - If found, returns the ManagedServiceAccount name and token expiration from the matched identity

   ```go
   // Example lookup logic in proxy-agent
   func (a *Authenticator) findManagedServiceAccount(hubNamespace, hubSAName string) (string, int64, error) {
       msaList := &authv1beta1.ManagedServiceAccountList{}
       if err := a.client.List(ctx, msaList, client.InNamespace(a.clusterNamespace)); err != nil {
           return "", 0, err
       }

       for _, msa := range msaList.Items {
           identitiesJSON := msa.Annotations[AnnotationHubIdentities]
           if identitiesJSON == "" {
               continue
           }

           var hubIdentities []HubIdentity
           if err := json.Unmarshal([]byte(identitiesJSON), &hubIdentities); err != nil {
               continue
           }

           // Find matching hub identity
           for _, identity := range hubIdentities {
               if identity.Namespace == hubNamespace && identity.Name == hubSAName {
                   return msa.Name, identity.TokenExpiration, nil
               }
           }
       }

       return "", 0, fmt.Errorf("no mapping found for %s/%s", hubNamespace, hubSAName)
   }
   ```

4. proxy-agent generates a token for that ServiceAccount with the specified expiration

### User Access Workaround

**Current Approach (ServiceAccount-only):**

If a hub user needs to access managed clusters through SubjectMapping, they can:

1. Create a dedicated ServiceAccount in a namespace they control
2. Create a SubjectMapping for that ServiceAccount
3. Use the ServiceAccount token instead of their user credentials when accessing managed clusters

**Example:**
```yaml
# Step 1: Create a ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alice-sa
  namespace: alice-workspace

---
# Step 2: Create SubjectMapping
apiVersion: authentication.open-cluster-management.io/v1alpha1
kind: SubjectMapping
metadata:
  name: alice-sa
  namespace: alice-workspace
spec:
  hubSubject:
    kind: ServiceAccount
    serviceAccount:
      name: alice-sa
  managedServiceAccount:
    name: alice-sa-on-clusters
    tokenExpirationSeconds: 3600

# Step 3: Alice uses alice-sa token to access managed clusters via cluster-proxy
```

**Future Consideration (User Subject Support):**

If there is strong demand for native User subject support, the main challenge to address would be:

**Conflict Resolution:** Unlike ServiceAccounts (where namespace-scoped API prevents duplicates), User subjects are cluster-wide identities. Multiple SubjectMappings could potentially map the same user, requiring conflict resolution logic:

- **Challenge**: User "alice@example.com" could be mapped by multiple SubjectMappings in different namespaces
- **Required Solution**: Implement conflict detection and resolution (e.g., first-created wins, admin-defined priority, or webhook validation to prevent duplicates)
- **Design Impact**: Would need to add back conflict resolution logic, `Conflicted` status condition, and potentially namespace field to User subject spec

The current ServiceAccount-only design avoids this complexity entirely while still enabling user access through the ServiceAccount workaround.

### Granting Permissions on Managed Clusters

SubjectMapping creates ServiceAccounts on managed clusters, but **does not grant any permissions**. Organizations can grant permissions using their preferred operational pattern. Common approaches include:

#### Hub-Driven via ManifestWork

Hub administrators can use ManifestWork to deploy RoleBindings:

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: argocd-rbac
  namespace: cluster1
spec:
  workload:
    manifests:
    - apiVersion: rbac.authorization.k8s.io/v1
      kind: ClusterRoleBinding
      metadata:
        name: argocd-prod-admin
      roleRef:
        kind: ClusterRole
        name: admin
      subjects:
      - kind: ServiceAccount
        name: argocd-prod  # Must match SubjectMapping's managedServiceAccount.name
        namespace: open-cluster-management-agent-addon
```

#### Hub-Driven via ClusterPermission API

Use OCM's ClusterPermission API for declarative permission management (if available):

```yaml
apiVersion: rbac.open-cluster-management.io/v1alpha1
kind: ClusterPermission
metadata:
  name: argocd-permissions
spec:
  clusterSelector:
    matchLabels:
      environment: production
  roleRef:
    kind: ClusterRole
    name: admin
  subjects:
  - kind: ServiceAccount
    name: argocd-prod  # Must match SubjectMapping's managedServiceAccount.name
    namespace: open-cluster-management-agent-addon
```

#### Spoke-Driven via Direct RBAC

Managed cluster administrators can create RoleBindings directly:

```bash
kubectl create clusterrolebinding argocd-prod-admin \
  --clusterrole=admin \
  --serviceaccount=open-cluster-management-agent-addon:argocd-prod
```

#### Coordination and Naming

User-specified `managedServiceAccount.name` enables coordination between hub and spoke administrators:

- Hub admin creates SubjectMapping with predictable name (e.g., `argocd-prod`, `shared-admin`)
- Permissions are granted via organization's chosen approach referencing that name
- Access control enforced by ValidatingAdmissionPolicy checking RBAC on virtual subresource `subjectmappings/managedserviceaccounts` with `use` verb

**Naming conventions**: `{application}-{environment}` (e.g., `argocd-prod`, `flux-staging`) or shared names (e.g., `shared-admin`, `dev-team`)

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

**Duplicate Prevention:**

- Namespace-scoped API with CEL validation ensures only one SubjectMapping per ServiceAccount
- Kubernetes enforces unique namespace/name combinations
- No webhook required - simpler operations, no single point of failure

**Required RBAC:**

Hub (SubjectMapping creation):

```yaml
# Recommended: cluster-admins only
apiGroups: ["authentication.open-cluster-management.io"]
resources: ["subjectmappings"]
verbs: ["create", "update", "delete"]
```

proxy-agent on hub (TokenReview):

```yaml
# Already exists in cluster-proxy
# proxy-agent needs this on the hub to validate hub ServiceAccount tokens
apiGroups: ["authentication.k8s.io"]
resources: ["tokenreviews"]
verbs: ["create"]
```

proxy-agent on hub (ManagedServiceAccount lookup):

```yaml
# Role (not ClusterRole!) - only in proxy-agent's own cluster namespace on hub
# proxy-agent needs this to find ManagedServiceAccounts with hub SA identity annotations
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: proxy-agent-subjectmapping
  namespace: cluster1  # The cluster namespace where this proxy-agent runs
rules:
- apiGroups: ["authentication.open-cluster-management.io"]
  resources: ["managedserviceaccounts"]
  verbs: ["list", "get", "watch"]  # watch for cache invalidation
```

proxy-agent on managed cluster (TokenRequest):

```yaml
apiGroups: [""]
resources: ["serviceaccounts/token"]
verbs: ["create"]
```

## Test Plan

### Unit Tests

- SubjectMapping controller: cluster discovery via ManagedClusterSetBinding, ManagedServiceAccount creation, finalizers
- proxy-agent: ServiceAccount token resolution, caching, cache invalidation, TokenReview integration
- CEL validation: metadata.name must match hubSubject.serviceAccount.name

### Integration Tests

- SubjectMapping lifecycle: creation, updates, deletion with cleanup
- ManagedClusterSetBinding changes trigger reconciliation
- Token generation, caching, and injection into requests
- CEL validation prevents mismatched names
- ValidatingAdmissionPolicy enforces virtual subresource permissions

### E2E Tests

- Complete request flow: hub ServiceAccount → cluster-proxy → managed cluster
- Token lifecycle: generation, caching, expiration, refresh
- Multi-cluster scenarios (100+ clusters)
- ArgoCD integration with ServiceAccount mapping

## Graduation Criteria

### Alpha (cluster-proxy v0.x.0)

- [ ] **managed-serviceaccount addon enhancement** to honor `authentication.open-cluster-management.io/token-projection` annotation
  - [ ] Agent skips token sync when annotation value is `"none"`
  - [ ] Backward compatible (missing annotation defaults to existing behavior)
  - [ ] Unit and integration tests in managed-serviceaccount repository
- [ ] SubjectMapping CRD in cluster-proxy repository (`pkg/apis/authentication/v1alpha1`)
  - [ ] Namespace-scoped API
  - [ ] CEL validation: metadata.name == hubSubject.serviceAccount.name
- [ ] ValidatingAdmissionPolicy for virtual subresource access control
  - [ ] Requires Kubernetes 1.28+
- [ ] SubjectMapping controller implementation in addon-manager
  - [ ] Creates ManagedServiceAccount with `token-projection: none` annotation
  - [ ] ManagedClusterSetBinding-based cluster discovery
- [ ] Authentication module in proxy-agent for ServiceAccount token resolution and caching
- [ ] TokenReview-based hub token validation
- [ ] Unit tests with >80% coverage
- [ ] Integration tests for ServiceAccount mapping
- [ ] Basic documentation and examples
- [ ] Feature flag for gradual rollout (optional, can be enabled by default in alpha)

### Beta (cluster-proxy v0.x+1.0)

- [ ] E2E tests covering all ServiceAccount scenarios
- [ ] Performance testing with 100+ clusters
- [ ] Production-ready token caching implementation with metrics
- [ ] Comprehensive user documentation
- [ ] Migration guide from ManagedServiceAccount pattern
- [ ] Metrics and monitoring support (cache hit rate, token generation latency, etc.)
- [ ] ArgoCD integration example and guide

### GA (cluster-proxy v1.x.0)

- [ ] Used in production by at least 2 organizations
- [ ] No critical bugs in 2+ beta release cycles
- [ ] Security review completed
- [ ] Troubleshooting guide with common scenarios
- [ ] Performance benchmarks published (compared to ManagedServiceAccount)
- [ ] Stable API (move from v1alpha1 to v1)

### Future Enhancements

- [ ] kubectl integration examples and user guide

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

- Hub: Kubernetes 1.28+ (ValidatingAdmissionPolicy with CEL authorizer, stable TokenRequest API)
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
