# Enhanced IgnoreField Selectors for ManifestWork

## Release Signoff Checklist

- [ ] Enhancement is `provisional`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal introduces enhanced field selector options for the `ignoreFields` configuration in ManifestWork and ManifestWorkReplicaSet. Currently, OCM only supports basic JSONPath expressions for specifying fields to ignore. This enhancement adds two additional selector types inspired by Argo CD's `ignoreDifferences` implementation:

1. **JSONPointers** - RFC 6901 compliant JSON Pointer expressions
2. **JQPathExpressions** - Full jq query language support for complex field selection

These additions will provide users with fine-grained control over which fields the work-agent should ignore when applying resources to spoke clusters, enabling more sophisticated multi-actor coordination scenarios.

## Motivation

The current `ignoreFields` option in ManifestWork's `ServerSideApply` update strategy only supports basic Kubernetes JSONPath expressions. While this works for simple field paths like `.spec.replicas` or `.metadata.annotations`, it falls short in complex scenarios where:

- Fields are nested within arrays and require conditional filtering
- Multiple fields matching a pattern need to be ignored
- Complex selection logic is required (e.g., "ignore weight field only for routes with specific destination subsets")

Argo CD has successfully implemented a more flexible approach with `jsonPointers` and `jqPathExpressions` in their `ignoreDifferences` configuration. This proposal brings similar capabilities to OCM, improving interoperability and providing users familiar with Argo CD patterns a consistent experience.

### Goals

1. Add `jsonPointers` field to `IgnoreField` struct supporting RFC 6901 JSON Pointer syntax
2. Add `jqPathExpressions` field to `IgnoreField` struct supporting jq query language
3. Maintain backward compatibility with existing `jsonPaths` field
4. Apply changes to both ManifestWork and ManifestWorkReplicaSet APIs
5. Provide clear documentation and examples for each selector type

### Non-Goals

1. Deprecating the existing `jsonPaths` field - all three selector types will coexist
2. Adding `managedFieldsManagers` support (may be considered in a future enhancement)
3. Changing the `condition` field behavior (`OnSpokePresent`, `OnSpokeChange`)

## Proposal

### User Stories

#### Story 1: Istio VirtualService Weight Management

As a platform engineer using OCM with Istio service mesh and a progressive delivery controller (like Argo Rollouts), I want to ignore specific route weights in VirtualService resources that are managed by the rollout controller, while still allowing the work-agent to manage other fields.

Currently, this is not possible with simple JSONPath because the weight fields are nested within arrays and require conditional selection based on the destination subset name.

With jqPathExpressions:
```yaml
updateStrategy:
  type: ServerSideApply
  serverSideApply:
    ignoreFields:
      - condition: OnSpokeChange
        jqPathExpressions:
          - '.spec.http[]? | select(.name? == "rollout") | .route[]? | select(.destination?.subset? == "canary" or .destination?.subset? == "stable") | .weight?'
```

#### Story 2: Deployment Annotation Management

As a cluster administrator, I want to ignore specific annotations added by monitoring tools on the spoke cluster without ignoring all annotations.

With jsonPointers:
```yaml
updateStrategy:
  type: ServerSideApply
  serverSideApply:
    ignoreFields:
      - condition: OnSpokeChange
        jsonPointers:
          - /metadata/annotations/prometheus.io~1scrape
          - /metadata/annotations/prometheus.io~1port
```

#### Story 3: Complex ConfigMap Data Fields

As a developer, I want to ignore specific keys within a ConfigMap's data field that are dynamically updated by spoke-side controllers.

With jqPathExpressions:
```yaml
updateStrategy:
  type: ServerSideApply
  serverSideApply:
    ignoreFields:
      - condition: OnSpokePresent
        jqPathExpressions:
          - '.data | to_entries | map(select(.key | startswith("dynamic-"))) | from_entries'
```

### Design Details

#### API Changes

The `IgnoreField` struct in the work API will be extended with two new fields:

```go
type IgnoreField struct {
    // Condition defines the condition that the fields should be ignored when apply the resource.
    // Fields are ignored when condition is met, otherwise no fields are ignored in the apply operation.
    // +kubebuilder:default=OnSpokePresent
    // +kubebuilder:validation:Enum=OnSpokePresent;OnSpokeChange
    // +kubebuilder:validation:Required
    // +required
    Condition IgnoreFieldsCondition `json:"condition"`

    // JSONPaths defines the list of json path in the resource to be ignored.
    // Uses Kubernetes JSONPath syntax.
    // +optional
    JSONPaths []string `json:"jsonPaths,omitempty"`

    // JSONPointers defines the list of JSON Pointers (RFC 6901) in the resource to be ignored.
    // JSON Pointers provide precise field targeting using a string syntax like "/spec/replicas".
    // Special characters in keys must be escaped: '~' becomes '~0' and '/' becomes '~1'.
    // +optional
    JSONPointers []string `json:"jsonPointers,omitempty"`

    // JQPathExpressions defines the list of jq path expressions in the resource to be ignored.
    // jq expressions provide powerful querying capabilities including array filtering,
    // conditional selection, and complex transformations.
    // Reference: https://stedolan.github.io/jq/manual/
    // +optional
    JQPathExpressions []string `json:"jqPathExpressions,omitempty"`
}
```

#### Validation Rules

1. At least one of `jsonPaths`, `jsonPointers`, or `jqPathExpressions` must be specified
2. All three can be used together - they will be applied in order: jsonPaths, jsonPointers, jqPathExpressions
3. Invalid expressions should be validated at admission time where possible
4. jqPathExpressions should have a configurable execution timeout (default: 1 second) to prevent resource exhaustion

#### Error Handling and Condition Reporting

**Validation Phase (ManifestWork Webhook)**:

Syntax and compilation errors for `jsonPointers` and `jqPathExpressions` will be caught during ManifestWork creation/update by the validation webhook:

- **JSON Pointer syntax errors**: Invalid pointer format (e.g., missing leading `/`, invalid escape sequences)
- **jq expression syntax errors**: Parse failures (e.g., unclosed quotes, invalid operators)
- **jq expression compilation errors**: Semantic errors (e.g., undefined functions, invalid type operations)

These errors will **reject the ManifestWork creation** with a clear error message from the webhook.

**Runtime Phase (Work-Agent on Spoke Cluster)**:

Runtime errors that occur during `ignoreFields` processing will be reported in the manifest-level `Applied` condition:

```go
const (
    // Manifest-level Applied condition reasons
    AppliedManifestComplete                = "AppliedManifestComplete"
    AppliedManifestFailed                  = "AppliedManifestFailed"
    AppliedManifestSSAIgnoreFieldError     = "AppliedManifestSSAIgnoreFieldError"  // NEW
)
```

**Runtime errors include**:

1. **jq expression timeout**: Expression runs longer than configured timeout (default: 1 second)
   ```
   Reason: AppliedManifestSSAIgnoreFieldError
   Message: Failed to process ignoreFields: JQ expression timed out after 1s: '.spec.containers[] | until(false; .)'
   ```

2. **jq expression type mismatch**: Expression expects different data type than actual resource
   ```
   Reason: AppliedManifestSSAIgnoreFieldError
   Message: Failed to process ignoreFields: JQ expression runtime error in '.spec.replicas[]': cannot iterate over number
   ```

3. **jq expression function errors**: Runtime errors from jq functions
   ```
   Reason: AppliedManifestSSAIgnoreFieldError
   Message: Failed to process ignoreFields: JQ expression runtime error in '.metadata.labels | fromjson': fromjson only works on strings
   ```

4. **JSON Pointer application errors**: Errors applying JSON Pointer patches (rare, as most are caught at validation)
   ```
   Reason: AppliedManifestSSAIgnoreFieldError
   Message: Failed to process ignoreFields: JSON Pointer error in '/spec/nonexistent': path not found
   ```

**Error Handling Strategy**:

- **Validation errors** → ManifestWork creation **rejected** by webhook
- **Runtime errors** → ManifestWork **created**, but manifest-level `Applied` condition set to `False` with `AppliedManifestSSAIgnoreFieldError` reason
- **Work-level condition** → Aggregates manifest-level conditions; if any manifest has `Applied=False`, work-level `Applied` is also `False`

This approach ensures:
- Early feedback for syntax errors (at ManifestWork creation time)
- Clear error reporting for runtime issues (in status conditions)
- Consistent with OCM's existing error handling patterns (specific error types with detailed messages in conditions)

#### Agent Implementation

The work-agent implementation in `pkg/work/spoke/apply/server_side_apply.go` will be extended to handle the new selector types.

##### JSON Pointer Implementation

For JSON Pointers, we will use the `github.com/evanphx/json-patch` library (same as Argo CD):

```go
import jsonpatch "github.com/evanphx/json-patch"

func removeFieldByJSONPointer(obj []byte, pointer string) ([]byte, error) {
    patchData, err := json.Marshal([]map[string]string{{"op": "remove", "path": pointer}})
    if err != nil {
        return nil, err
    }
    patch, err := jsonpatch.DecodePatch(patchData)
    if err != nil {
        return nil, err
    }
    return patch.Apply(obj)
}
```

##### jq Expression Implementation

For jq expressions, we will use the `github.com/itchyny/gojq` library (pure Go implementation, same as Argo CD):

```go
import "github.com/itchyny/gojq"

type jqNormalizerPatch struct {
    code               *gojq.Code
    jqExecutionTimeout time.Duration
}

func (np *jqNormalizerPatch) Apply(data []byte) ([]byte, error) {
    dataJSON := make(map[string]any)
    if err := json.Unmarshal(data, &dataJSON); err != nil {
        return nil, err
    }

    ctx, cancel := context.WithTimeout(context.Background(), np.jqExecutionTimeout)
    defer cancel()

    // Wrap expression in del() to remove matched fields
    iter := np.code.RunWithContext(ctx, dataJSON)
    first, ok := iter.Next()
    if !ok {
        return nil, errors.New("jq expression did not return any data")
    }
    if err, ok := first.(error); ok {
        if errors.Is(err, context.DeadlineExceeded) {
            return nil, fmt.Errorf("jq expression execution timed out")
        }
        return nil, fmt.Errorf("jq expression error: %w", err)
    }

    return json.Marshal(first)
}

// During initialization, compile the jq expression wrapped in del()
func compileJQExpression(expression string, timeout time.Duration) (*jqNormalizerPatch, error) {
    query, err := gojq.Parse(fmt.Sprintf("del(%s)", expression))
    if err != nil {
        return nil, err
    }
    code, err := gojq.Compile(query)
    if err != nil {
        return nil, err
    }
    return &jqNormalizerPatch{
        code:               code,
        jqExecutionTimeout: timeout,
    }, nil
}
```

##### Processing Order

When multiple selector types are specified, they will be processed in the following order:
1. `jsonPaths` (existing behavior using `k8s.io/client-go/util/jsonpath`)
2. `jsonPointers` (RFC 6901 using `github.com/evanphx/json-patch`)
3. `jqPathExpressions` (jq queries using `github.com/itchyny/gojq`)

Each selector removes its matched fields from the resource before the next selector is applied.

#### Examples

##### Example 1: Mixed Selector Types

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  namespace: cluster1
  name: istio-virtualservice
spec:
  workload:
    manifests:
      - apiVersion: networking.istio.io/v1beta1
        kind: VirtualService
        metadata:
          name: my-service
          namespace: default
        spec:
          hosts:
            - my-service
          http:
            - name: rollout
              route:
                - destination:
                    host: my-service
                    subset: canary
                  weight: 0
                - destination:
                    host: my-service
                    subset: stable
                  weight: 100
  manifestConfigs:
    - resourceIdentifier:
        group: networking.istio.io
        resource: virtualservices
        namespace: default
        name: my-service
      updateStrategy:
        type: ServerSideApply
        serverSideApply:
          ignoreFields:
            - condition: OnSpokeChange
              jqPathExpressions:
                - '.spec.http[]? | select(.name? == "rollout") | .route[]? | select(.destination?.subset? == "canary" or .destination?.subset? == "stable") | .weight?'
```

##### Example 2: JSON Pointer for Precise Field Targeting

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  namespace: cluster1
  name: deployment-with-hpa
spec:
  workload:
    manifests:
      - apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: my-app
          namespace: default
        spec:
          replicas: 3
          # ... rest of deployment spec
  manifestConfigs:
    - resourceIdentifier:
        group: apps
        resource: deployments
        namespace: default
        name: my-app
      updateStrategy:
        type: ServerSideApply
        serverSideApply:
          ignoreFields:
            - condition: OnSpokePresent
              jsonPointers:
                - /spec/replicas
```

##### Example 3: ManifestWorkReplicaSet with jq Expressions

```yaml
apiVersion: work.open-cluster-management.io/v1alpha1
kind: ManifestWorkReplicaSet
metadata:
  name: rollout-config
  namespace: ocm-ns
spec:
  placementRefs:
    - name: all-clusters
  manifestWorkTemplate:
    workload:
      manifests:
        - # ... manifest content
    manifestConfigs:
      - resourceIdentifier:
          group: networking.istio.io
          resource: virtualservices
          namespace: default
          name: my-service
        updateStrategy:
          type: ServerSideApply
          serverSideApply:
            ignoreFields:
              - condition: OnSpokeChange
                jqPathExpressions:
                  - '.spec.http[].route[].weight'
```

##### Example 4: Combined JSONPath and JQPathExpressions for Operator-Injected Containers

This example demonstrates a scenario where a service mesh operator (like Istio) injects sidecar containers into Pods via a mutating admission webhook. The hub wants to manage the main application container but allow the operator to inject and manage sidecars, volumes, and init containers.

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  namespace: cluster1
  name: app-pod
spec:
  workload:
    manifests:
      - apiVersion: v1
        kind: Pod
        metadata:
          name: my-application
          namespace: production
          labels:
            app: my-application
            version: v1
        spec:
          containers:
            - name: application
              image: myapp:1.2.3
              ports:
                - containerPort: 8080
              env:
                - name: LOG_LEVEL
                  value: "info"
          # When this Pod is created, Istio mutating webhook intercepts and injects:
          # - containers: istio-proxy sidecar
          # - initContainers: istio-init
          # - volumes: istio-envoy, istio-data, istio-token, istiod-ca-cert
          # - annotations: sidecar.istio.io/status, prometheus.io/scrape, etc.
  manifestConfigs:
    - resourceIdentifier:
        resource: pods
        namespace: production
        name: my-application
      updateStrategy:
        type: ServerSideApply
        serverSideApply:
          ignoreFields:
            - condition: OnSpokePresent
              # Use JSONPath for simple, well-known static fields
              jsonPaths:
                - .metadata.annotations.kubectl.kubernetes.io/restartedAt
                - .metadata.annotations.sidecar.istio.io/status
                - .metadata.annotations.prometheus.io/scrape
                - .metadata.annotations.prometheus.io/port
              # Use JQPathExpressions for complex conditional logic on arrays
              # Ignore all containers except the main 'application' container
              jqPathExpressions:
                - '.spec.containers[] | select(.name != "application")'
                # Ignore all volumes injected by Istio (have istio- prefix)
                - '.spec.volumes[]? | select(.name | startswith("istio-"))'
                # Ignore all init containers (typically istio-init, istio-validation)
                - '.spec.initContainers[]?'
                # Ignore volume mounts in injected containers
                - '.spec.containers[] | select(.name != "application") | .volumeMounts[]?'
```

**Scenario Explanation**:

1. **Hub defines in ManifestWork**: 
   - Pod with single "application" container
   - Application image, ports, environment variables
   - Pod labels and basic metadata

2. **Istio mutating admission webhook intercepts Pod creation** and injects:
   - `istio-proxy` sidecar container into `spec.containers[]`
   - `istio-init` into `spec.initContainers[]`
   - Multiple volumes: `istio-envoy`, `istio-data`, `istio-token`, `istiod-ca-cert`
   - Annotations: `sidecar.istio.io/status`, `prometheus.io/scrape`, `prometheus.io/port`

3. **Work-agent behavior with ignoreFields**:
   - Ignores the injected `istio-proxy` container (jq expression filters it out)
   - Ignores all Istio-related volumes (jq expression with `startswith("istio-")`)
   - Ignores all init containers
   - Ignores operator-added annotations (JSONPath)
   - Still manages the "application" container (not filtered by jq)

**Why both selector types**:
- **JSONPath** (`jsonPaths`): For simple annotation paths that are always at the same location in pod metadata
- **JQPathExpressions** (`jqPathExpressions`): For conditional filtering within the containers/volumes arrays - "ignore all containers except 'application'" or "ignore volumes matching a pattern"

**Behavior**:
- When Istio webhook injects `istio-proxy` sidecar during Pod creation, work-agent won't try to remove it
- When you update the application container image in ManifestWork, work-agent applies only that change
- When Istio adds volumes for certificates and config, work-agent ignores them
- The operator can freely manage its injected resources without conflicts

### Risks and Mitigation

#### Risk 1: jq Expression Performance

Complex jq expressions could potentially consume significant CPU resources or run indefinitely.

**Mitigation**: 
- Implement a configurable execution timeout (default: 1 second)
- Add metrics to track jq expression execution time
- Document best practices for writing efficient jq expressions

#### Risk 2: Security Concerns with jq

jq is a powerful query language that could potentially be misused.

**Mitigation**:
- The `gojq` library is a pure Go implementation that doesn't execute external commands
- jq expressions only operate on the resource data, not on the system
- Timeout prevents resource exhaustion attacks

#### Risk 3: Backward Compatibility

Existing ManifestWorks using `jsonPaths` should continue to work.

**Mitigation**:
- The `jsonPaths` field remains unchanged and fully supported
- New fields are optional
- Validation ensures at least one selector type is specified

### Test Plan

1. **Unit Tests**:
   - Test JSON Pointer parsing and field removal
   - Test jq expression compilation and execution
   - Test timeout behavior for long-running jq expressions
   - Test combined usage of all three selector types
   - Test error handling for invalid expressions

2. **Integration Tests**:
   - Test `ignoreFields` with `jsonPointers` for `OnSpokePresent` and `OnSpokeChange`
   - Test `ignoreFields` with `jqPathExpressions` for `OnSpokePresent` and `OnSpokeChange`
   - Test complex nested field selection with jq expressions
   - Test array element selection with jq expressions

3. **E2E Tests**:
   - Test real-world scenario with Istio VirtualService and progressive delivery
   - Test HPA-managed deployment replicas with JSON Pointer
   - Test ManifestWorkReplicaSet with jq expressions across multiple clusters

### Graduation Criteria

#### Alpha (Tech Preview)
- API changes implemented with feature gate
- Basic unit and integration tests passing
- Documentation for new fields

#### Beta
- Feature gate enabled by default
- Comprehensive test coverage
- Performance benchmarks for jq expression execution
- User feedback incorporated

#### GA
- Feature gate removed
- Production usage validated
- Complete documentation and examples

### Upgrade / Downgrade Strategy

**Upgrade**:
- CRD update required on hub cluster for ManifestWork and ManifestWorkReplicaSet
- Work-agent upgrade required on managed clusters to process new fields
- Existing ManifestWorks continue to work without modification

**Downgrade**:
- Older work-agents will ignore the new `jsonPointers` and `jqPathExpressions` fields
- ManifestWorks using new fields should be updated to use only `jsonPaths` before downgrade

### Version Skew Strategy

- New hub with old work-agent: New fields are ignored, existing `jsonPaths` behavior preserved
- Old hub with new work-agent: No impact, new fields not present in CRD
- The fields are optional, so partial upgrades are safe

## Implementation History

- 2026-02-09: Initial proposal created

## Drawbacks

1. **Increased Complexity**: Adding two new selector types increases API surface and implementation complexity
2. **Dependency Addition**: Requires adding `github.com/itchyny/gojq` and `github.com/evanphx/json-patch` dependencies
3. **Learning Curve**: Users need to understand three different selector syntaxes

## Alternatives

### Alternative 1: Only Add JSON Pointers

Add only `jsonPointers` without jq support. This would be simpler but wouldn't address complex array filtering scenarios.

**Rejected because**: jq expressions are essential for scenarios like Istio VirtualService weight management where conditional array element selection is required.

### Alternative 2: Extend JSONPath Syntax

Extend the existing `jsonPaths` to support more complex expressions.

**Rejected because**: Kubernetes JSONPath has a fixed syntax, and extending it would create a non-standard dialect that could confuse users.

### Alternative 3: Use CEL Instead of jq

Use Common Expression Language (CEL) instead of jq for complex expressions.

**Considered for future**: CEL is already used in OCM for condition rules. However, jq is more widely known for JSON manipulation and aligns with Argo CD's approach. CEL support could be added in a future enhancement.

## Infrastructure Needed

No new infrastructure required. The implementation uses existing OCM repositories:
- `open-cluster-management.io/api` - API type changes
- `open-cluster-management.io/ocm` - Work-agent implementation changes
