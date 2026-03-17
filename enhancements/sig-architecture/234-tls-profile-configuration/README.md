# TLS Profile Configuration for OCM Components

## Release Signoff Checklist

- [ ] Enhancement is implementable
- [ ] Design details are clear and agreed upon
- [ ] Test plan is defined
- [ ] Graduation criteria established
- [ ] User-facing documentation created

## Summary

This enhancement proposes a standard mechanism for configuring TLS profiles across OCM hub and spoke components. By consuming TLS configuration from well-known ConfigMaps, OCM components can adapt their TLS settings dynamically without requiring custom resource definitions or platform-specific dependencies. This enables better security posture, operational flexibility, and integration with platform TLS policies while maintaining OCM's Kubernetes-agnostic nature.

## Motivation

OCM components currently use hardcoded or minimally configurable TLS settings. As security requirements evolve and platforms introduce centralized TLS policies, OCM components need a standard way to:

1. **Consume TLS Configuration**: Accept TLS settings from external sources (platform policies, security frameworks, compliance requirements)
2. **Maintain Platform Independence**: Support TLS configuration without depending on platform-specific APIs
3. **Enable Dynamic Updates**: Allow TLS settings to change without component rebuilds
4. **Support Security Compliance**: Meet varying security requirements across different deployment environments

### Current Limitations

- **Hardcoded Defaults**: Most OCM components use Go's default TLS 1.2 settings
- **No Central Configuration**: Each component may have different TLS settings
- **Limited Observability**: Difficult to verify what TLS settings are actually in use
- **Platform Integration Gap**: No standard way for platforms to influence OCM's TLS configuration

### Goals

- Define a standard ConfigMap-based pattern for TLS configuration consumption
- Support dynamic TLS profile updates without component restarts where possible
- Enable operator-managed TLS configuration injection via command-line flags
- Provide fallback behavior for deployments without TLS configuration
- Support both hub and spoke components consistently
- Extend pattern to addon managers and agents

### Non-Goals

- Define how TLS configuration is created or managed (platform/deployment-specific)
- Implement client-side TLS configuration (server-side only)
- Change existing ManagedServiceAccount or authentication mechanisms
- Enforce specific TLS versions or cipher suites (configuration is deployment-specific)
- Provide runtime TLS compliance monitoring

## Proposal

### Architecture Overview

OCM follows a **two-tier architecture** for TLS configuration:

```
┌──────────────────────────────────────────────────────────┐
│ Tier 1: Operators                                        │
│   (cluster-manager-operator, klusterlet-operator)        │
│                                                          │
│ • Watch ConfigMap for TLS configuration                 │
│ • Apply TLS config to their own servers                 │
│ • Inject TLS flags into managed component deployments   │
│ • Trigger component rollouts when config changes        │
└──────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────┐
│ Tier 2: Components                                       │
│   (controllers, webhooks, agents)                        │
│                                                          │
│ • Receive TLS config via command-line flags only        │
│ • Parse flags on startup                                │
│ • Apply TLS config to their servers                     │
│ • No ConfigMap watching needed                          │
└──────────────────────────────────────────────────────────┘
```

**Key Principle:** Components are configured by operators via flags. Only operators watch ConfigMaps.

### Simplified Pattern Summary

All OCM scenarios follow 4 fundamental patterns:

#### Case 1: ConfigMap Creation + Self-Managed Components

**ConfigMap Source:**

- Controller OR manual configuration creates `ocm-tls-profile` ConfigMap

**Who Consumes:**

- Components deployed on hub (addon-managers)
- Operators (hub or spoke, doesn't matter)

**Behavior:**

- Watch `ocm-tls-profile` ConfigMap in their namespace
- Restart (`os.Exit(0)`) when ConfigMap changes

**Applies to:**

- cluster-manager-operator
- addon-managers (e.g., cluster-proxy-addon-manager, submariner-addon-manager)
- klusterlet-operator

#### Case 2: Operators Inject Flags into Components

**Who:**

- Operators (hub or spoke, doesn't matter)

**Behavior:**

- Watch ConfigMap from Case 1
- Read ConfigMap and inject TLS values as command-line flags into component deployments
- Add `tls-config-hash` annotation to trigger rollout when config changes

**Applies to:**

- cluster-manager-operator injects flags into hub components
- klusterlet-operator injects flags into spoke agents

**Note:** Operators do BOTH Case 1 (self-restart) AND Case 2 (inject flags for their components)

#### Case 3: Components Receive Flags

**Who:**

- Components deployed by operators

**Behavior:**

- Receive TLS config via flags: `--tls-min-version=VersionTLS13 --tls-cipher-suites=...`
- Parse flags on startup (no ConfigMap watching)
- Restarted by Kubernetes when operator changes deployment annotation

**Applies to:**

- registration-controller, registration-webhook, work-controller, placement-controller, work-webhook, addon-manager-controller, addon-webhook
- registration-agent, work-agent, klusterlet-agent

#### Case 4: Addon Agents with ConfigMap Copy (Optional)

**Infrastructure:**

- klusterlet-operator copies `ocm-tls-profile` ConfigMap to addon namespaces

**Who:**

- Addon agents (optional - addon squads may choose their own approach)

**Behavior:**

- Watch `ocm-tls-profile` ConfigMap in their namespace
- Restart when ConfigMap changes
- Same as Case 1, but with ConfigMap copy infrastructure

### TLS Configuration Sources

**For Operators:**
1. **ConfigMap** (well-known `ocm-tls-profile` ConfigMap in operator namespace)
2. **Go runtime defaults** (fallback: TLS 1.2 when ConfigMap not found)

**For Components:**
1. **Command-line flags** (`--tls-min-version`, `--tls-cipher-suites`) injected by operators
2. **Go runtime defaults** (fallback: TLS 1.2 when flags not provided)

### ConfigMap Format

Operators will watch for a ConfigMap with well-known naming conventions:

**Hub Components:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ocm-tls-profile
  namespace: open-cluster-management-hub  # or component namespace
data:
  minTLSVersion: "VersionTLS12"  # or VersionTLS13
  cipherSuites: "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
```

**Spoke Components:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ocm-tls-profile
  namespace: open-cluster-management-agent  # or component namespace
data:
  minTLSVersion: "VersionTLS12"
  cipherSuites: "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
```

**Field Specifications:**

- **`minTLSVersion`** (required): Minimum TLS version. Valid values: `VersionTLS10`, `VersionTLS11`, `VersionTLS12`, `VersionTLS13`
- **`cipherSuites`** (optional): Comma-separated list of cipher suite names as defined by Go's `crypto/tls` package

**Important TLS 1.3 Behavior:**

When `minTLSVersion: VersionTLS13`:
- The `cipherSuites` field is ignored (TLS 1.3 uses hardcoded cipher suites in Go runtime)
- Go's `crypto/tls` automatically uses these cipher suites:
  - `TLS_AES_128_GCM_SHA256`
  - `TLS_AES_256_GCM_SHA384`
  - `TLS_CHACHA20_POLY1305_SHA256`

### Implementation Patterns

OCM uses two simple patterns for TLS configuration:

#### Pattern 1: Operators (Self-Configure)

**Applies to:** cluster-manager-operator, klusterlet-operator

**Behavior:**
- Watch `ocm-tls-profile` ConfigMap in their namespace
- Apply TLS config to their own webhook/metrics servers
- Inject TLS flags into managed component deployments
- Restart themselves when ConfigMap changes (via `os.Exit(0)`)

**Example deployment with injected flags:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registration-controller
spec:
  template:
    metadata:
      annotations:
        tls-config-hash: "abc123"  # Updated to trigger rollout
    spec:
      containers:
      - name: registration-controller
        args:
        - "--tls-min-version=VersionTLS12"
        - "--tls-cipher-suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,..."
```

#### Pattern 2: Components (Operator-Configured)

**Applies to:** All hub controllers/webhooks and spoke agents managed by operators

**Behavior:**
- Receive TLS configuration via command-line flags only
- Parse flags on startup
- Apply TLS config to their servers
- **Do NOT watch ConfigMaps** (operators handle updates via rollouts)

**Lifecycle:** When TLS configuration changes, operators update deployment annotations to trigger Kubernetes rolling updates.

### Addon Framework Integration

The klusterlet-operator provides **AddonTLSConfigController** (similar to the existing AddonPullImageSecretController pattern) that automatically copies the `ocm-tls-profile` ConfigMap to all addon namespaces.

**Controller Behavior:**

- Watches namespaces labeled with `addon.open-cluster-management.io/namespace=true`
- Copies `ocm-tls-profile` from `open-cluster-management-agent` to addon namespaces
- Keeps ConfigMap copies synchronized

**Flow:**

```text
Source: open-cluster-management-agent/ocm-tls-profile
  ↓ (AddonTLSConfigController copies)
Copies:
  - open-cluster-management-agent-addon/ocm-tls-profile
  - addon-app/ocm-tls-profile
  - addon-policy/ocm-tls-profile
```

**Addon Team Options:**

Using the ConfigMap is **optional**. Addon teams can choose:

- **Option A (Recommended):** Watch `ocm-tls-profile` in addon namespace, restart on changes (like operators)
- **Option B:** Implement custom TLS configuration (independent from OCM infrastructure)

### Command-Line Flags

All OCM components with TLS servers support these standard flags:

```bash
--tls-min-version string
    Minimum TLS version: VersionTLS10, VersionTLS11, VersionTLS12, VersionTLS13
    (default: VersionTLS12)

--tls-cipher-suites string
    Comma-separated cipher suite list (ignored when minVersion is VersionTLS13)
    Example: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
```

Components use `k8s.io/component-base/cli/flag` helpers for parsing and validation.

### Component Coverage

#### Hub Components

| Component | Repository | Pattern | Priority |
|-----------|-----------|---------|----------|
| cluster-manager-operator | ocm | Operator (self-configure) | P0 |
| registration-controller | ocm | Component (operator-configured) | P0 |
| registration-webhook | ocm | Component (operator-configured) | P0 |
| work-controller | ocm | Component (operator-configured) | P0 |
| placement-controller | ocm | Component (operator-configured) | P1 |
| work-webhook | ocm | Component (operator-configured) | P1 |
| addon-manager-controller | ocm | Component (operator-configured) | P1 |
| addon-webhook | ocm | Component (operator-configured) | P1 |

#### Spoke Components

| Component | Repository | Pattern | Priority |
|-----------|-----------|---------|----------|
| klusterlet-operator | ocm | Operator (self-configure) | P0 |
| registration-agent | ocm | Component (operator-configured) | P0 |
| work-agent | ocm | Component (operator-configured) | P0 |
| klusterlet-agent | ocm | Component (operator-configured) | P0 |

#### Addons

| Component | Repository | Notes | Priority |
|-----------|-----------|-------|----------|
| Addon managers | addon-framework | Optional ConfigMap usage | P1 |
| Addon agents | various | Optional ConfigMap usage (auto-copied by AddonTLSConfigController) | P1 |

### New Components Required

| Component | Repository | Purpose |
| --- | --- | --- |
| **Shared TLS library** | `open-cluster-management-io/sdk-go` | ConfigMap/flag parsing, TLS config application to `crypto/tls.Config` |
| **AddonTLSConfigController** | `open-cluster-management-io/ocm` | Copies TLS ConfigMap from agent namespace to addon namespaces (similar to AddonPullImageSecretController) |

### Fallback Behavior

OCM works on vanilla Kubernetes without requiring TLS configuration:

- **No ConfigMap + No Flags**: Component uses Go default TLS 1.2 settings
- **ConfigMap Parse Error**: Component logs error and uses Go defaults (fail-safe)
- **Invalid Values**: Component filters invalid values and proceeds with valid ones

### Observability

Components log TLS configuration on startup:

```text
I0317 TLS server configuration: minVersion=VersionTLS12, cipherSuites=[...]
```

### Migration Path

- **Phase 1:** Add flag support + shared TLS library (sdk-go)
- **Phase 2:** Operators watch ConfigMaps, inject flags into components
- **Phase 3:** AddonTLSConfigController implementation
- **Phase 4:** Documentation and E2E tests

### Workflow Description

#### Hub Components

1. Administrator creates `ocm-tls-profile` ConfigMap in operator namespace
2. cluster-manager-operator watches ConfigMap, applies to its own servers
3. Operator injects TLS flags into component deployments (e.g., registration-controller)
4. Operator updates deployment annotation to trigger rollout
5. Components start with TLS configuration from flags

#### Spoke Components

1. Administrator creates `ocm-tls-profile` ConfigMap in `open-cluster-management-agent`
2. klusterlet-operator watches ConfigMap, applies to its own servers
3. Operator injects TLS flags into agent deployments
4. AddonTLSConfigController copies ConfigMap to addon namespaces
5. Addon agents optionally watch ConfigMap in their namespace

### API Extensions

This enhancement does not introduce new CRDs. TLS configuration uses standard Kubernetes ConfigMaps with well-known names and formats.

**ConfigMap Naming Convention:**
- Hub: `ocm-tls-profile` in `open-cluster-management-hub` (or component namespace)
- Spoke: `ocm-tls-profile` in `open-cluster-management-agent` (or component namespace)
- Addons: `ocm-tls-profile` in addon namespace (e.g., `addon-app`, `open-cluster-management-agent-addon`)

**ConfigMap Schema:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ocm-tls-profile
  namespace: <component-namespace>
  labels:
    ocm.io/config-type: tls-profile
data:
  # Required: Minimum TLS version
  minTLSVersion: "VersionTLS12"  # or VersionTLS10, VersionTLS11, VersionTLS13

  # Optional: Comma-separated cipher suite names (ignored for TLS 1.3)
  cipherSuites: "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
```

### Risks and Mitigations

**Risk:** Components may have bugs in TLS configuration parsing
**Mitigation:** Shared sdk-go library with comprehensive unit tests. Fallback to safe defaults on parse errors.

**Risk:** TLS configuration rollout may cause service disruption
**Mitigation:** Rolling updates ensure gradual rollout. Operators validate configuration before injection.

**Risk:** Incompatible TLS settings may break component communication
**Mitigation:** Document recommended profiles. Provide validation in operators where possible. E2E tests verify component communication.

**Risk:** Addon agents may not detect ConfigMap updates
**Mitigation:** Use standard Kubernetes informers with established retry mechanisms. Log configuration changes for observability.

**Risk:** ConfigMap propagation delays to addon namespaces
**Mitigation:** AddonTLSConfigController watches efficiently. Initial ConfigMap created before addon deployment.

## Design Details

### Test Plan

#### Unit Tests

- TLS ConfigMap parsing (valid/invalid inputs)
- Flag parsing and validation
- TLS config application to `crypto/tls.Config`
- Cipher suite validation and filtering

#### Integration Tests

- cluster-manager-operator watches ConfigMap and updates deployments
- klusterlet-operator watches ConfigMap and updates deployments
- AddonTLSConfigController copies ConfigMap to addon namespaces
- Operator self-restart on TLS config change

#### E2E Tests

- Create TLS ConfigMap, verify hub components use correct TLS settings
- Update TLS ConfigMap, verify components roll out with new settings
- Test addon agent receives ConfigMap in its namespace
- Verify TLS handshake works with configured settings (client/server verification)
- Test fallback behavior (missing ConfigMap, invalid values)
- Verify component communication works with different TLS profiles

### Graduation Criteria

#### Alpha

- [ ] Shared TLS library in sdk-go
- [ ] TLS flag support in operators (cluster-manager-operator, klusterlet-operator)
- [ ] ConfigMap watching in operators and flag injection into components
- [ ] TLS flag support in P0 components
- [ ] Basic E2E tests validating TLS configuration

#### Beta

- [ ] AddonTLSConfigController implemented
- [ ] TLS flag support in all core components
- [ ] Comprehensive E2E tests including addon scenarios
- [ ] Documentation for TLS configuration patterns

#### GA

- [ ] Production deployments validating the design
- [ ] Performance testing (ConfigMap watch overhead)
- [ ] User-facing documentation with examples

### Upgrade / Downgrade Strategy

**Upgrade:**
- Existing deployments continue using default TLS settings
- No ConfigMap = no behavior change
- Operators gain ConfigMap watching capability
- Platform can optionally create ConfigMap to configure TLS

**Downgrade:**
- Components revert to default TLS settings
- ConfigMaps remain but are ignored
- No data loss or migration required

### Version Skew Strategy

- **Flag support added first**: Components accept flags before operators use them
- **Backward compatible**: Components without flag support use Go defaults
- **Operator-led rollout**: Operators only inject flags after components support them
- **ConfigMap optional**: Missing ConfigMap is always valid (fallback to defaults)

## FAQ

**Q: Why do operators watch ConfigMaps but components don't?**

A: Operators are not deployed by another controller, so they must self-configure. Components are deployed by operators, so operators can inject flags during deployment rendering.

**Q: What if the ConfigMap doesn't exist?**

A: Components fall back to TLS 1.2 with Go default cipher suites. This ensures OCM works on vanilla Kubernetes without any configuration.

**Q: Can multiple components share one ConfigMap?**

A: Yes, if they're in the same namespace. This is the pattern for hub components and spoke agents.

**Q: How do platforms inject TLS configuration?**

A: Platforms create the `ocm-tls-profile` ConfigMap in operator namespaces. The mechanism (sidecar, controller, manual) is platform-specific.

**Q: Does this support hot-reload?**

A: Operators restart when ConfigMap changes. Components are restarted by operators via deployment rollout (triggered by annotation). This is simpler than hot-reload and sufficient for infrequent TLS changes.

**Q: What about addons?**

A: AddonTLSConfigController automatically copies the TLS ConfigMap to addon namespaces. Addons can optionally use it or implement custom TLS configuration.

## Implementation History

- 2026-03-17: Initial proposal

## Alternatives

### Alternative 1: Custom Resource Definition (TLSProfile CRD)

Create a dedicated `TLSProfile` CRD:
```yaml
apiVersion: config.open-cluster-management.io/v1alpha1
kind: TLSProfile
metadata:
  name: cluster
spec:
  minTLSVersion: VersionTLS12
  cipherSuites:
  - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

**Rejected because:**
- Adds API maintenance burden
- ConfigMaps are simpler and more universally understood
- Doesn't provide significant benefits over ConfigMap approach
- Creates additional RBAC complexity

### Alternative 2: Environment Variables

Inject TLS configuration via environment variables:
```yaml
env:
- name: OCM_TLS_MIN_VERSION
  value: "VersionTLS12"
```

**Rejected because:**
- Less discoverable than ConfigMaps
- Harder to update dynamically
- Doesn't provide type safety or validation
- Command-line flags are more standard in Kubernetes ecosystem

### Alternative 3: Embedded in Operator CRDs

Add TLS configuration to existing CRDs (ClusterManager, Klusterlet):
```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: ClusterManager
spec:
  tlsProfile:
    minTLSVersion: VersionTLS12
```

**Rejected because:**
- Couples TLS configuration to operator lifecycle
- Harder for platforms to inject TLS policies
- Requires CRD updates for each new TLS parameter
- ConfigMap pattern is more flexible

## Infrastructure Needed

- Add `pkg/tls` package to `open-cluster-management-io/sdk-go`
- Add `AddonTLSConfigController` to `open-cluster-management-io/ocm`
- Add TLS flag support to all components with TLS servers
- E2E test infrastructure for TLS verification

## References

- [Go crypto/tls documentation](https://pkg.go.dev/crypto/tls)
- [Kubernetes component-base TLS flags](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/component-base/cli/flag/ciphersuites_flag.go)
- [Go TLS 1.3 cipher behavior](https://github.com/golang/go/issues/29349)
