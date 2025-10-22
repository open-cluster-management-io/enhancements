# Addon Dependency Management

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary

This proposal introduces a dependency management mechanism for managed cluster addons, allowing addon authors to declare dependencies between addons. The system will ensure that dependent addons are installed and available before installing or marking an addon as healthy.

## Motivation

Currently, there is no way to declare or enforce dependencies between addons in Open Cluster Management. This creates several challenges:

1. **Manual coordination required**: Administrators must manually ensure that prerequisite addons are installed before installing dependent addons.

2. **Silent failures**: When a dependent addon is missing, addons may fail at runtime with unclear error messages, making troubleshooting difficult.

3. **Configuration complexity**: Some addons rely on Custom Resource Definitions (CRDs) or resources provided by other addons (e.g., Managed Service Account addon provides ManagedServiceAccount API that other addons use). Without dependency tracking, these relationships are implicit and undocumented.

4. **Installation ordering**: There's no automated way to ensure correct installation ordering when multiple interdependent addons are deployed simultaneously.

### Goals

- Provide a declarative way for addon authors to specify dependencies on other addons
- Automatically validate that all dependencies are satisfied before considering an addon healthy
- Provide clear status feedback when dependencies are not met
- Maintain backward compatibility with existing addons that have no dependencies

### Non-Goals

- Automatic installation of dependencies (users still need to explicitly install required addons)
- Dependency resolution and ordering during installation
- Support for circular dependencies
- Dependency management across different managed clusters (dependencies are validated per cluster)
- Version constraints for dependencies (addon API does not currently track version information)

## Proposal

We propose to extend the `ClusterManagementAddOn` API to include a `dependencies` field that allows addon authors to declare dependencies on other addons. The addon-manager component will validate these dependencies on each managed cluster and set appropriate status conditions on the `ManagedClusterAddOn` resource.

### User Stories

#### Story 1: Addon depending on Managed Service Account

As an addon author, I want to declare a dependency on the Managed Service Account addon because my addon uses the `ManagedServiceAccount` API to provision service accounts on managed clusters. When the Managed Service Account addon is not installed, I want users to see a clear error message in the addon status.


## Design Details

### API Changes

#### ClusterManagementAddOn

Add a new `dependencies` field to the `ClusterManagementAddOnSpec`:

```go
// ClusterManagementAddOnSpec provides information for the add-on.
type ClusterManagementAddOnSpec struct {
    // ... existing fields ...

    // dependencies is a list of add-ons that this add-on depends on.
    // The add-on will only be installed and considered healthy if all its dependencies
    // are installed and available on the managed cluster.
    // An empty list means the add-on has no dependencies.
    // The default is an empty list.
    // +optional
    Dependencies []AddonDependency `json:"dependencies,omitempty"`
}

// AddonDependency represents a dependency on another add-on.
type AddonDependency struct {
    // name is the name of the dependent add-on.
    // This should match the name of a ClusterManagementAddOn resource.
    // +required
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:MinLength:=1
    Name string `json:"name"`

    // type specifies the type of the dependency.
    // Valid values are:
    // - "Required" (default): The addon cannot function without this dependency.
    //   The Degraded condition will be set with reason RequiredDependencyNotSatisfied, and the Available
    //   condition should be set to False when the dependency is not satisfied.
    // - "Optional": The addon can work with reduced functionality without this dependency.
    //   The Degraded condition will be set with reason DependencyNotSatisfied when the dependency is not satisfied,
    //   but the Available condition may remain True if the addon is otherwise functional.
    // +optional
    // +kubebuilder:validation:Enum=Optional;Required
    // +kubebuilder:default=Required
    Type DependencyType `json:"type,omitempty"`

    // message is a human-readable message describing what functionality will be impacted
    // when this dependency is not satisfied.
    // This is particularly useful for Optional dependencies to inform users about the reduced functionality.
    // The message will be included in the Degraded condition when the dependency is not satisfied.
    // Example: "Token-based access to managed clusters is unavailable"
    // +optional
    Message string `json:"message,omitempty"`
}

// DependencyType describes the type of dependency
// +kubebuilder:validation:Enum=Optional;Required
type DependencyType string

const (
    // DependencyTypeOptional indicates the addon can work with reduced functionality without this dependency
    DependencyTypeOptional DependencyType = "Optional"
    // DependencyTypeRequired indicates the addon cannot function without this dependency
    DependencyTypeRequired DependencyType = "Required"
)
```

#### ManagedClusterAddOn Status

Add a new reason for the existing `Degraded` condition:

```go
// the reasons of condition ManagedClusterAddOnConditionDegraded
const (
    // AddonDegradedReasonDependencyNotSatisfied is the reason of condition Degraded indicating that one or more
    // soft (optional) dependencies of the addon are not satisfied (not installed or not available).
    // The addon may still be Available with reduced functionality.
    AddonDegradedReasonDependencyNotSatisfied = "DependencyNotSatisfied"

    // AddonDegradedReasonRequiredDependencyNotSatisfied is the reason of condition Degraded indicating that one or more
    // hard (required) dependencies of the addon are not satisfied (not installed or not available).
    // The Available condition should also be set to False as the addon cannot function.
    AddonDegradedReasonRequiredDependencyNotSatisfied = "RequiredDependencyNotSatisfied"
)
```

#### Dependency Types

There are two types of dependencies:

- **Required dependencies (default, type=Required)**: The addon cannot function at all without the dependency. When a required dependency is not satisfied, the addon-manager will set `Degraded=True` with reason `RequiredDependencyNotSatisfied`, and the klusterlet-agent will detect this specific reason and set `Available=False`.

- **Optional dependencies (type=Optional)**: The addon can still function with reduced functionality when the dependency is missing. When an optional dependency is not satisfied, the addon-manager will set `Degraded=True` with reason `DependencyNotSatisfied`, but the klusterlet-agent will not modify the `Available` condition, allowing the addon to remain available if its health checks pass.

### Implementation Details

#### Dependency Validation

**Addon-Manager (on Hub):**

The addon-manager component will implement dependency validation with the following logic:

1. **Read dependencies**: When reconciling a `ManagedClusterAddOn`, read the corresponding `ClusterManagementAddOn` to get the list of dependencies.

2. **Check each dependency**: For each dependency in the list:
   - Check if a `ManagedClusterAddOn` with the same name exists in the same namespace (managed cluster namespace)
   - Check if the dependent addon's `Available` condition is `True`

3. **Set Degraded condition based on dependency type**:
   - **If all dependencies are satisfied**:
     - Ensure the `Degraded` condition is not set with reason `DependencyNotSatisfied` or `RequiredDependencyNotSatisfied`

   - **If any optional dependency is not satisfied** (type=Optional):
     - Set the `Degraded` condition to `True` with:
       - Reason: `DependencyNotSatisfied`
       - Message: Clear description of which dependencies are missing, including the custom message if provided (e.g., "Optional addon 'managed-serviceaccount' is not installed or not available. Token-based access to managed clusters is unavailable.")
     - Do NOT modify the `Available` condition

   - **If any required dependency is not satisfied** (type=Required):
     - Set the `Degraded` condition to `True` with:
       - Reason: `RequiredDependencyNotSatisfied` (different reason!)
       - Message: Clear description of which dependencies are missing, including the custom message if provided (e.g., "Required addon 'managed-serviceaccount' is not installed or not available. This addon cannot function without ManagedServiceAccount API.")
     - Do NOT modify the `Available` condition (klusterlet-agent will do this)

**Klusterlet-Agent (on Managed Cluster):**

The klusterlet-agent component will check the `Degraded` condition when determining addon availability:

1. **When reconciling addon health**: After checking lease/probe health status
2. **Check for required dependency failures**:
   - If `Degraded=True` with reason `RequiredDependencyNotSatisfied`, set `Available=False`
   - If `Degraded=True` with reason `DependencyNotSatisfied` (optional dependency), do not modify `Available` - let the addon's own health checks determine availability

This design ensures clear component ownership and avoids conflicts:
- **addon-manager** owns dependency validation and sets `Degraded` with appropriate reason
- **klusterlet-agent** owns `Available` and considers dependency information when making availability decisions

**Key Design Benefits:**
1. The klusterlet-agent does not need access to the `ClusterManagementAddOn` API - all dependency type information is encoded in the `Degraded` condition reason
2. Component ownership is clear and prevents components from fighting over the same condition
3. Different dependency types (Optional vs Required) result in different status behaviors automatically

### Example Usage

#### Example 1: Optional Dependency

An addon that can work with reduced functionality without the dependency:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: my-addon
spec:
  addOnMeta:
    displayName: "My Addon"
    description: "An addon that optionally uses ManagedServiceAccount API"
  dependencies:
  - name: managed-serviceaccount
    type: Optional  # Must explicitly set to Optional (Required is the default)
    message: "Token-based access to managed clusters is unavailable"
  installStrategy:
    type: Manual
```

When the Managed Service Account addon is not installed, the `ManagedClusterAddOn` status would show:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: my-addon
  namespace: cluster1
status:
  conditions:
  - type: Available
    status: "True"  # Addon is still available with reduced functionality
    reason: AddonAvailable
    message: "Addon is available"
    lastTransitionTime: "2025-10-22T10:00:00Z"
  - type: Degraded
    status: "True"
    reason: DependencyNotSatisfied
    message: "Optional addon 'managed-serviceaccount' is not installed or not available. Token-based access to managed clusters is unavailable"
    lastTransitionTime: "2025-10-22T10:00:00Z"
```

#### Example 2: Required Dependency (Default)

An addon that cannot function without the dependency:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ClusterManagementAddOn
metadata:
  name: my-critical-addon
spec:
  addOnMeta:
    displayName: "My Critical Addon"
    description: "An addon that requires ManagedServiceAccount API"
  dependencies:
  - name: managed-serviceaccount
    # type: Required is the default, can be omitted
    message: "This addon cannot function without ManagedServiceAccount API"
  installStrategy:
    type: Manual
```

When the Managed Service Account addon is not installed, the `ManagedClusterAddOn` status would show:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: my-critical-addon
  namespace: cluster1
status:
  conditions:
  - type: Available
    status: "False"  # Set by klusterlet-agent
    reason: RequiredDependencyNotSatisfied
    message: "Required addon 'managed-serviceaccount' is not installed or not available. This addon cannot function without ManagedServiceAccount API"
    lastTransitionTime: "2025-10-22T10:00:00Z"
  - type: Degraded
    status: "True"  # Set by addon-manager
    reason: RequiredDependencyNotSatisfied
    message: "Required addon 'managed-serviceaccount' is not installed or not available. This addon cannot function without ManagedServiceAccount API"
    lastTransitionTime: "2025-10-22T10:00:00Z"
```

### Risks and Mitigations

#### Risk: Circular Dependencies

**Risk**: Users might accidentally create circular dependencies (A depends on B, B depends on A).

**Mitigation**:
- Document that circular dependencies are not supported and will result in both addons being marked as degraded
- Consider adding validation webhook to detect and reject circular dependencies
- Future enhancement: Add a validation controller that detects circular dependencies

#### Risk: Dependency Chain Complexity

**Risk**: Long dependency chains might make troubleshooting difficult.

**Mitigation**:
- Provide clear error messages that list all unsatisfied dependencies
- Document best practices for keeping dependency chains shallow
- Consider adding status field to show the full dependency tree

### Test Plan

#### Unit Tests

- Test dependency validation logic with various scenarios:
  - No dependencies
  - Single dependency (satisfied/unsatisfied)
  - Multiple dependencies (all satisfied, some unsatisfied, none satisfied)

#### Integration Tests

- Test complete workflow:
  1. Install addon A with dependency on addon B (B not installed) - verify Degraded condition
  2. Install addon B - verify addon A becomes Available
  3. Delete addon B - verify addon A becomes Degraded

#### E2E Tests

- Deploy real addons with dependencies on a test cluster
- Verify status conditions are correctly set
- Verify addon behavior when dependencies are not met

### Graduation Criteria

#### Beta (v1beta1)

- API changes implemented in `open-cluster-management.io/api` addon v1beta1
- Dependency validation fully supported by addon-manager component
- Klusterlet-agent updated to handle `RequiredDependencyNotSatisfied` reason
- Unit and integration tests passing
- E2E tests with real addons
- At least 2 real addons using dependency declarations
- Metrics for dependency validation failures
- Documentation of the feature and API

#### GA (v1)

- Proven stability over at least 2 releases in v1beta1
- Comprehensive documentation including troubleshooting guides
- No critical bugs reported
- Widely adopted by addon authors (at least 5 addons using dependencies in production)
- Performance validated at scale (tested with 1000+ clusters)

### Upgrade / Downgrade Strategy

#### Upgrade

- **From version without dependency support to version with dependency support**:
  - Existing addons without dependencies continue to work unchanged
  - New or updated addons can add dependencies field
  - No migration required

#### Downgrade

- **From version with dependency support to version without dependency support**:
  - Dependencies field will be ignored by older controllers
  - Addons will function as if they have no dependencies
  - Status conditions related to dependencies will not be updated

### Version Skew Strategy

- Hub and managed cluster components need to be aware of version compatibility
- Dependency validation happens on the hub side in the addon-manager component
- No version skew issues expected as long as the API version is compatible

## Implementation History

- 2025-10-22: Initial KEP draft
- TBD: Beta implementation
- TBD: GA promotion

## Drawbacks

- Adds complexity to the addon API
- Requires addon authors to maintain accurate dependency information
- Does not automatically install dependencies (users must still manually install them)

## Alternatives

### Alternative 1: Only Set Degraded Condition

Instead of adding a `required` field, the addon-manager could only set the `Degraded` condition when dependencies are not satisfied, and never modify the `Available` condition.

**Pros**:
- Simpler implementation - no need for the `required` field
- Addon controllers have full control over their own availability status
- Cleaner separation of concerns

**Cons**:
- Less clear semantics - users cannot declare hard dependencies in the API
- Addon controllers must implement their own logic to set `Available=False` when dependencies are missing
- More work for addon authors who want hard dependency behavior

**Decision**: Not chosen as the primary approach because the `type` field provides clearer semantics and reduces boilerplate for addon authors. However, this remains a valid alternative if the `type` field proves too complex in practice.

## Infrastructure Needed

- No new infrastructure required
- Existing CI/CD pipelines can be used for testing
- Documentation updates needed in open-cluster-management website
