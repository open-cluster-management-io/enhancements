# Token Based Add-On Registration

## Release Signoff Checklist

- [ ] Enhancement is implementable
- [ ] Design details are clear and agreed upon
- [ ] Test plan is defined
- [ ] Graduation criteria established
- [ ] User-facing documentation created

## Summary

This enhancement proposes a new registration driver that uses Kubernetes service account tokens (via the TokenRequest API) for add-on authentication. This provides a simpler, more Kubernetes-native alternative to certificate-based authentication for add-ons while maintaining security through automatic token refresh. Add-ons can use token-based authentication independently of the cluster registration driver, allowing operators to balance security and operational requirements differently for cluster and add-on components.

## Background

### Understanding Registration in OCM

In Open Cluster Management (OCM), both managed clusters and add-ons need to authenticate with the hub cluster. This authentication is handled by **registration drivers**.

**Two Types of Registration:**

1. **Cluster Registration**: The managed cluster (klusterlet agent) authenticates to the hub
2. **Add-On Registration**: Add-on agents authenticate to the hub

**Current State:**

When you deploy a cluster, you choose a **cluster registration driver** (like `csr`, `awsirsa`, or `grpc`). This driver determines:
- How the cluster (klusterlet agent) communicates with the hub (the **registration API**)
- What type of credentials the cluster (klusterlet agent) uses
- How credentials are created and managed (the **credential API**)

For add-on registration, add-ons currently:
- Inherit the registration API used by cluster registration from the cluster driver
- Use **certificate-based authentication**
- Obtain certificates via the API determined by the cluster driver type (Kubernetes API for `csr`/`awsirsa`, gRPC API for `grpc`)

## Motivation

The current CSR-based add-on authentication has several operational challenges:

1. **Complex Approval Workflows**: CSR requires certificate approval infrastructure and management, adding operational overhead
2. **Rigid Security Model**: Add-ons can only use certificate-based authentication via CSR, regardless of cluster driver
3. **Development Friction**: Setting up CSR infrastructure for add-on development and testing is time-consuming

This enhancement introduces token-based authentication as an alternative to address these challenges:

1. **Simplified Operations**: Uses Kubernetes TokenRequest API with automatic rotation, eliminating CSR approval workflows
2. **Flexible Security Posture**: Allows different credential types for add-ons and clusters (e.g., token for add-ons, certificate for clusters)
3. **Lower Barrier**: Easier setup for add-on development and testing

### Goals

- Implement token-based registration driver for add-on authentication using Kubernetes TokenRequest API
- Support automatic token refresh before expiration
- Allow add-ons to use different credential types than cluster registration

### Non-Goals

- Token-based cluster registration (add-ons only)
- Replace existing csr driver (this is an additional option)
- External token providers (OIDC, OAuth)

## Proposal

### User Stories

#### Story 1: Simplified Add-On Development
As an add-on developer, I want to test add-ons without setting up CSR approval infrastructure, so I can iterate faster during development.

#### Story 2: Independent Authentication
As a platform operator, I want to use different credential types for cluster registration and add-on registration, so I can meet different security and operational requirements for different components.

### Implementation Details

#### Add-On Registration Types

Add-ons need credentials to communicate with the hub cluster. Depending on what they need to access, add-ons can have two types of registrations:

1. **`kubeClient` type**: For accessing hub kube-apiserver. The driver can be `token` or `csr`.
2. **`customSigner` type**: For accessing non-kube endpoints with client certificates. Only `csr` driver is supported.

**Example - Multiple Registration Configurations:**

```yaml
status:
  registrations:
  - type: kubeClient  # For accessing hub kube-apiserver
    kubeClient:
      driver: token  # Can be "token" or "csr"
      subject:  # Only user is set when driver is token; both user and groups are set when driver is csr
        user: system:serviceaccount:cluster1:cluster1-addon1-agent
  - type: customSigner  # For accessing non-kube endpoints with client certificates
    customSigner:
      signerName: example.com/addon-signer  # Always uses csr driver
      subject:
        user: addon1-client
        groups:
        - example:addon1
```

Each registration configuration item is handled by a separate driver instance. Multiple registration items can share the same registration type (e.g., multiple `customSigner` type registrations). The token driver handles only the `kubeClient` type registration.

##### Configuration

The `addOnKubeClientRegistrationDriver` field in the Klusterlet resource controls the authentication driver for `kubeClient` type registration:
- `kubeClient` type registration: Configurable via `addOnKubeClientRegistrationDriver.authType` (can be `csr` or `token`)
- `customSigner` type registration: Always uses `csr` driver (not configurable)

**Configuration Example:**

```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: Klusterlet
metadata:
  name: klusterlet
spec:
  clusterName: cluster1
  registrationConfiguration:
    registrationDriver:
      authType: csr  # Cluster uses csr driver
    addOnKubeClientRegistrationDriver:
      authType: token  # kubeClient type registration uses token driver
      token:
        expirationSeconds: 31104000  # Optional: 1 year (default if not specified)
```

The operator reads the configuration and adds flags to the agent deployment:
- `--addon-kubeclient-registration-auth=token`
- `--addon-token-expiration-seconds=<value>` (if `token.expirationSeconds` is specified)

**Note**: If `token.expirationSeconds` is not set or is 0, the default duration (typically 1 year) will be used. The minimum valid value for production use is 3600 seconds (1 hour), though smaller values are allowed for testing.

##### Add-On Registration Flow

Agent performs add-on registration after cluster registration completes, using cluster credentials (client certificate or token) for hub communication.

**Registration Flow:**

**Step 1: ManagedClusterAddOn Created**
- ManagedClusterAddOn resource is created on hub by the common add-on manager (addon-manager component) or administrator

**Step 2: Add-On Specific Hub Controller Updates Registration Configuration**
- Add-on specific hub controller updates `status.registrations[]` with registration configurations
- Adds registration items based on add-on requirements (e.g., `kubeClient`, `customSigner`)
- Leaves both `driver` (for `kubeClient` type) and `subject` fields empty for agent to populate

**Step 3: Agent Starts Add-On Registration**
- For each add-on's registration item in `status.registrations[]`, creates a driver instance
- If registration type is `kubeClient`, uses value from `--addon-kubeclient-registration-auth` flag. If not specified, defaults to `csr` driver
- Each driver instance handles one registration item independently

**Step 4: Driver Declares Authentication Driver**
- Driver sets `status.registrations[{type: "kubeClient"}].kubeClient.driver = "token"` on ManagedClusterAddOn
- Sets `kubeClient.subject.user` to `system:serviceaccount:<cluster-namespace>:<cluster-name>-<addon-name>-agent`
- Waits for `TokenInfrastructureReady=True` before proceeding

**Step 5: Add-On Specific Hub Controller Binds Permissions**
- Add-on specific hub controller binds add-on specific ClusterRole/Role to the `kubeClient.subject` in the ManagedClusterAddOn status
- Creates ClusterRoleBinding or RoleBinding to grant add-on permissions based on add-on requirements

**Step 6: Common Add-On Manager Creates Token Infrastructure**
- The common add-on manager (addon-manager component) identifies token authentication from `status.registrations[{type: "kubeClient"}].kubeClient.driver == "token"`
- Creates per-add-on resources:
  - **ServiceAccount** `<cluster-name>-<addon-name>-agent`
  - **Role for token rotation**: Allows requesting tokens for the add-on ServiceAccount
  - **RoleBinding for token rotation**: Binds to cluster credential identity (Groups: `system:open-cluster-management:<cluster-name>` and `open-cluster-management:<cluster-name>`)
- Sets condition `TokenInfrastructureReady=True` with ServiceAccount UID in message

**Step 7: Driver Requests Initial Token**
- Driver watches ManagedClusterAddOn resource for `TokenInfrastructureReady=True` condition
- Calls TokenRequest API using cluster credentials

**Step 8: Agent Stores Add-On Token**
- Stores token in add-on secret (e.g., `<addon-name>-hub-kubeconfig`) with metadata (token, expiration, UID)
- Sets condition `AddonTokenRefreshed=True` with expiration timestamp
- Add-on registration complete

**Ongoing Operations:**

Token refresh:
- Triggered when:
  - Token is nearing expiration (80% of lifetime consumed)
  - ServiceAccount UID mismatch (ServiceAccount was recreated on hub)
  - Add-on secret (`<addon-name>-hub-kubeconfig`) is deleted from managed cluster
- Refresh process:
  - Agent uses cluster credentials to call TokenRequest API for add-on ServiceAccount
  - Stores new token with updated metadata (token, expiration, UID) in add-on secret
  - Updates `AddonTokenRefreshed` condition with new expiration timestamp

##### Supported Driver Combinations

**Understanding the Inheritance Model:**

When you configure a cluster driver, it determines:
- **Registration API**: The communication channel (Kubernetes API, gRPC API, etc.)
- **Cluster Credential API**: How cluster credentials are obtained

Add-on registration **inherits** the registration API from the cluster driver, but you can **independently configure** the credential type for kubeClient type registration (certificate or token).

**Valid Combinations:**

| Cluster Driver | Add-on kubeClient Driver | What Changes for Add-ons |
|----------------|--------------------------|--------------------------|
| csr            | csr (default)            | Nothing - all use certificates via Kubernetes API |
| csr            | token                    | Add-on uses tokens instead of certificates (still via Kubernetes API) |
| awsirsa        | csr (default)            | Add-on uses certificates via Kubernetes API (cluster uses IRSA tokens) |
| awsirsa        | token                    | Add-on uses tokens via Kubernetes API (cluster uses IRSA tokens) |
| grpc           | csr (default)            | Nothing - all use certificates via gRPC API |
| grpc           | token                    | Add-on uses tokens instead of certificates (still via gRPC API) |

**Key Points:**
- The **registration API is always inherited** from the cluster driver (you cannot change this)
- For **add-on credentials**, you can choose between `csr` (certificate) or `token` authentication
- If `addOnKubeClientRegistrationDriver` is not set, it defaults to `csr`
- Token-based authentication is available **regardless of which cluster driver you use**

**Credential API Details:**

The credential API (where credentials are obtained) depends on the cluster driver:

| Cluster Driver | Cluster Credential API | Add-on kubeClient Type Credential API | Add-on customSigner Type Credential API |
|----------------|------------------------|---------------------------------------|----------------------------------------|
| csr            | Kubernetes API         | Kubernetes API                        | Kubernetes API                         |
| awsirsa        | AWS IRSA API           | Kubernetes API                        | Kubernetes API                         |
| grpc           | gRPC API               | gRPC API                              | gRPC API                               |

*Note: When using `csr` or `awsirsa` cluster drivers, add-ons always obtain credentials via Kubernetes API. When using `grpc` cluster driver, add-ons obtain credentials via gRPC API.*

### Risks and Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| Add-on token theft/leakage | High - compromised token allows add-on impersonation | - Tokens scoped to specific add-on ServiceAccount with minimal RBAC<br>- Automatic rotation limits exposure window<br>- Stored in Kubernetes secrets (not in code/logs) |
| Cluster credential failure | Medium - add-on cannot refresh token | - Add-ons depend on cluster credentials for token refresh<br>- Monitor cluster credential health<br>- Document dependency clearly |
| Token expiration during rotation | Low - add-on loses hub connectivity | - Rotation triggered at 80% lifetime (ample buffer)<br>- Retry logic for token request failures |
| RBAC misconfiguration | Medium - over-permissioned add-on tokens | - Follow principle of least privilege for add-ons<br>- Document required permissions clearly<br>- Provide RBAC templates and examples |

## Design Details

### API Changes

**Klusterlet API (`operator.open-cluster-management.io/v1`)**

Update `RegistrationConfiguration` to include `addOnKubeClientRegistrationDriver` field:

```go
type RegistrationConfiguration struct {
    // ... existing fields (ClientCertExpirationSeconds, FeatureGates, etc.) ...

    // This provides driver details required to register klusterlet agent with hub
    // +optional
    RegistrationDriver RegistrationDriver `json:"registrationDriver,omitempty"`

    // This provides driver details for kubeClient type registration
    // +optional
    AddOnKubeClientRegistrationDriver AddOnRegistrationDriver `json:"addOnKubeClientRegistrationDriver,omitempty"`

    // ... other fields ...
}
```

Add `AddOnRegistrationDriver` type for configuring `kubeClient` type registration authentication:

```go
type AddOnRegistrationDriver struct {
    // AuthType is the authentication driver used for add-on registration.
    // Possible values are csr and token.
    // Currently, this field only affects kubeClient type registration. The customSigner type registration always uses csr driver.
    // In the future, this may be extended to customize authentication for customSigner type registration as well.
    // +optional
    // +kubebuilder:validation:Enum=csr;token
    AuthType string `json:"authType,omitempty"`

    // Token contains the configuration for token-based registration.
    // +optional
    Token *TokenConfig `json:"token,omitempty"`
}

type TokenConfig struct {
    // ExpirationSeconds represents the seconds of a token to expire.
    // If it is not set or 0, the default duration will be used, which is
    // the same as the certificate expiration set by the hub cluster's
    // kube-controller-manager (typically 1 year).
    // The minimum valid value for production use is 3600 (1 hour), though smaller values are allowed for testing.
    // +optional
    ExpirationSeconds int64 `json:"expirationSeconds,omitempty"`
}
```

**ManagedClusterAddOn API (`addon.open-cluster-management.io/v1beta1`)**

Update `KubeClientConfig` structure in `ManagedClusterAddOnStatus.Registrations` to add `Driver` field:

```go
type KubeClientConfig struct {
    // subject is the user subject of the addon agent to be registered to the hub.
    // +optional
    Subject KubeClientSubject `json:"subject,omitempty"`

    // Driver is the authentication driver used by ManagedClusterAddOn for kubeClient registration. Possible values are csr and token.
    // This field is set by the agent to declare which driver it is using.
    // +optional
    // +kubebuilder:validation:Enum=csr;token
    Driver string `json:"driver,omitempty"`
}
```

### Test Plan

#### Unit Tests
- Token validation and rotation logic
- Error handling for TokenRequest API failures
- Driver interface implementation

#### Integration Tests
- Add-on registration flow with token driver
- Token refresh when nearing expiration (80% lifetime)
- Recovery from ServiceAccount recreation (UID mismatch)
- Recovery from token secret deletion
- Multiple add-ons with different cluster drivers (csr, awsirsa, grpc)

#### E2E Tests
- Add-on registration and operations across multiple clusters
- Token lifecycle management (creation, refresh, expiration)
- Multiple klusterlets with different add-on registration drivers (e.g., cluster1 uses csr, cluster2 uses token)
- Cluster using grpc driver with add-on using token driver (EKS support scenario)

### Graduation Criteria

**Alpha**
- [ ] Basic token driver implementation for add-ons (agent + hub)
- [ ] API changes in ManagedClusterAddOn (v1beta1) and Klusterlet (v1)
- [ ] Unit tests with >80% coverage
- [ ] Integration tests for happy path
- [ ] Documentation for add-on configuration

**Beta**
- [ ] Token refresh and recovery implemented and tested
- [ ] E2E tests covering multiple scenarios
- [ ] Production-ready RBAC templates for add-ons
- [ ] User documentation and examples
- [ ] Performance testing with 100+ add-ons

**GA**
- [ ] Used in production by at least 2 organizations
- [ ] No critical bugs in 2+ release cycles
- [ ] Security review completed
- [ ] Comprehensive troubleshooting guide
- [ ] Metrics and monitoring support

### Upgrade / Downgrade Strategy

**Upgrade:**
- Existing add-ons using csr driver are not affected
- Add-ons can opt-in to token driver via `addOnKubeClientRegistrationDriver.authType=token` in Klusterlet resource
- No data migration required (token driver uses separate secrets per add-on)

**Downgrade:**
- If an add-on using token driver needs to switch to another driver:
  1. Update Klusterlet resource `addOnKubeClientRegistrationDriver` to use different authType
  2. Agent will re-register add-on using new authentication method
  3. Old add-on ServiceAccount and RBAC can be cleaned up manually or via garbage collection

### Version Skew Strategy

- Token driver requires Kubernetes 1.20+ (TokenRequest API stable)
- Hub and agent versions can differ as long as both support token auth type for add-ons
- Backward compatibility: older agents won't recognize `token` auth type (graceful failure with clear error message)

## Implementation History

- 2025-12-15: Initial proposal created

## Drawbacks

1. **Additional Complexity**: Adds a second driver option for kubeClient type registration, increasing maintenance burden
2. **Cluster Credential Dependency**: Add-on token refresh depends on cluster credentials remaining valid
3. **Token Lifetime Trade-offs**: Long-lived tokens increase risk if compromised; short-lived tokens increase rotation overhead
4. **Kubernetes Dependency**: Requires Kubernetes 1.20+ for stable TokenRequest API

## Infrastructure Needed

- Standard Kubernetes RBAC (no additional infrastructure)
- Integration test environment with multiple clusters
- CI/CD pipeline updates for new driver testing
