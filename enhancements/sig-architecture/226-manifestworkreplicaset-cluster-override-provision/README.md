# ClusterSet-Level Manifest Overrides for ManifestWorkReplicaSet

## Release Signoff Checklist

- [ ] Enhancement is `provisional`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal introduces clusterSet-level manifest override capabilities to ManifestWorkReplicaSet (MWRS), enabling users to customize manifests for specific clusterSets while maintaining the progressive rollout and scalability benefits of MWRS.

## Motivation

### Current Limitations

ManifestWorkReplicaSet currently uses a simple template-based approach where the same `ManifestWorkTemplate` is applied to all selected clusters. This creates limitations when:

1. **Environment-specific configurations** are needed (e.g., different replica counts for prod vs staging clusterSets)
2. **ClusterSet-specific resource requirements** vary (e.g., different CPU/memory limits based on clusterSet capacity profiles)
3. **Per-clusterSet customizations** are required (e.g., different images, environment variables, or labels)
4. **Namespace variations** exist across clusterSets (e.g., different naming conventions)

### Use Cases

- **Multi-environment deployments**: Deploy the same application with environment-specific configurations (prod clusterSet: 5 replicas, staging clusterSet: 1 replica)
- **ClusterSet capacity optimization**: Adjust resource requests/limits based on clusterSet capacity profiles
- **Compliance requirements**: Apply different security contexts or labels based on clusterSet compliance levels
- **Regional customizations**: Use different images or configurations for different geographical clusterSets

## Proposal

### API Design

Extend the `ManifestWorkReplicaSetSpec` to include clusterSet-level overrides:

```go
type ManifestWorkReplicaSetSpec struct {
    // Existing fields
    ManifestWorkTemplate work.ManifestWorkSpec `json:"manifestWorkTemplate"`
    PlacementRefs []LocalPlacementReference `json:"placementRefs"`
    CascadeDeletionPolicy CascadeDeletionPolicy `json:"cascadeDeletionPolicy,omitempty"`
    
    // New field for clusterSet-level overrides
    ClusterSetOverrides []ClusterSetManifestOverride `json:"clusterSetOverrides,omitempty"`
}

type ClusterSetManifestOverride struct {
    // Target clusterSet name
    ClusterSetName string `json:"clusterSetName"`
    
    // Manifest-specific overrides
    ManifestOverrides []ManifestOverride `json:"manifestOverrides,omitempty"`
}

type ManifestOverride struct {
    // Identifies the target manifest
    ResourceIdentifier ResourceIdentifier `json:"resourceIdentifier"`
    
    // Override specifications
    Override OverrideSpec `json:"override"`
}

type ResourceIdentifier struct {
    APIVersion string `json:"apiVersion,omitempty"`
    Kind       string `json:"kind,omitempty"`
    Name       string `json:"name,omitempty"`
    Namespace  string `json:"namespace,omitempty"`
}

type OverrideSpec struct {
    // Metadata overrides (name, namespace, labels, annotations)
    MetadataOverride *MetadataOverride `json:"metadataOverride,omitempty"`
    
    // JSON patch operations for spec modifications
    SpecOverride []JSONPatch `json:"specOverride,omitempty"`
    
    // Complete manifest replacement
    ReplaceManifest *runtime.RawExtension `json:"replaceManifest,omitempty"`
}
```

### Override Types

1. **Metadata Overrides**: Modify resource metadata (name, namespace, labels, annotations)
2. **Spec Overrides**: Use JSON patches to modify any field in the resource spec
3. **Complete Replacement**: Replace the entire manifest for specific clusters

### Implementation Strategy

#### Phase 1: Core Override Engine
- Implement override application logic in the controller
- Add resource identification and matching capabilities
- Support metadata and JSON patch overrides

#### Phase 2: Enhanced Features
- Add validation webhooks for override syntax
- Implement complete manifest replacement
- Add comprehensive test coverage

#### Phase 3: Advanced Capabilities
- Support for conditional overrides based on cluster labels
- Template-based overrides with cluster context variables
- Override inheritance and composition

### Example Usage

```yaml
apiVersion: work.open-cluster-management.io/v1alpha1
kind: ManifestWorkReplicaSet
metadata:
  name: web-app-deployment
  namespace: default
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
          selector:
            matchLabels:
              app: web-app
          template:
            metadata:
              labels:
                app: web-app
            spec:
              containers:
              - name: web
                image: nginx:1.20
                resources:
                  requests:
                    cpu: 100m
                    memory: 128Mi

  clusterSetOverrides:
  - clusterSetName: "prod-clusterset"
    manifestOverrides:
    - resourceIdentifier:
        apiVersion: apps/v1
        kind: Deployment
        name: web-app
      override:
        metadataOverride:
          labels:
            environment: production
            tier: high-performance
        specOverride:
        - op: replace
          path: /spec/replicas
          value: 5
        - op: replace
          path: /spec/template/spec/containers/0/resources/requests/cpu
          value: 500m
        - op: replace
          path: /spec/template/spec/containers/0/resources/requests/memory
          value: 512Mi

  - clusterSetName: "staging-clusterset"
    manifestOverrides:
    - resourceIdentifier:
        apiVersion: apps/v1
        kind: Deployment
        name: web-app
      override:
        metadataOverride:
          labels:
            environment: staging
            tier: standard
        specOverride:
        - op: replace
          path: /spec/replicas
          value: 1
        - op: replace
          path: /spec/template/spec/containers/0/resources/requests/cpu
          value: 50m

  placementRefs:
  - name: web-app-placement
    rolloutStrategy:
      type: Progressive
      progressive:
        minSuccessTime: 2m
        progressDeadline: 10m
```

### JSON Patch Examples

The `specOverride` field uses RFC 6902 JSON Patch operations. Here are common patch patterns:

#### Replace Operations
```yaml
specOverride:
- op: replace
  path: /spec/replicas
  value: 3
- op: replace
  path: /spec/template/spec/containers/0/image
  value: nginx:1.21
```

#### Add Operations
```yaml
specOverride:
- op: add
  path: /spec/template/spec/containers/0/env
  value:
  - name: ENVIRONMENT
    value: production
- op: add
  path: /spec/template/spec/containers/0/volumeMounts
  value:
  - name: config-volume
    mountPath: /etc/config
```

#### Remove Operations
```yaml
specOverride:
- op: remove
  path: /spec/template/spec/containers/0/resources/limits
- op: remove
  path: /spec/template/spec/containers/0/env/1
```

#### Complex Nested Operations
```yaml
specOverride:
- op: replace
  path: /spec/template/spec/containers/0/resources
  value:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
- op: add
  path: /spec/template/spec/securityContext
  value:
    runAsNonRoot: true
    runAsUser: 1000
```

## Design Principles

### 1. Backward Compatibility
- Existing ManifestWorkReplicaSets continue to work unchanged
- New override field is optional
- No breaking changes to current API

### 2. Progressive Rollout Preservation
- All existing rollout strategies work with clusterSet overrides
- Override application happens during ManifestWork creation based on cluster's clusterSet membership
- Rollout behavior remains consistent across clusterSets

### 3. Flexibility and Power
- Support multiple override types (metadata, JSON patches, replacement)
- Precise resource targeting using ResourceIdentifier
- ClusterSet-based grouping provides logical organization
- Extensible design for future enhancements

### 4. Performance Considerations
- Overrides applied at ManifestWork creation time, not runtime
- Efficient manifest matching using structured identifiers
- ClusterSet-based approach reduces override complexity
- Minimal impact on controller performance

## Implementation Details

### Controller Changes

1. **Enhanced CreateManifestWork Function**
   ```go
   func CreateManifestWorkWithOverrides(
       mwrSet *workapiv1alpha1.ManifestWorkReplicaSet,
       clusterName string,
       clusterSetName string,
       placementRefName string,
   ) (*workv1.ManifestWork, error)
   ```

2. **ClusterSet Resolution Logic**
   - Determine cluster's clusterSet membership
   - Match clusterSet name with override specifications
   - Handle clusters belonging to multiple clusterSets

3. **Override Application Logic**
   - Resource identification and matching
   - Metadata override application
   - JSON patch processing
   - Manifest replacement handling

4. **Validation and Error Handling**
   - Validate override syntax
   - Validate clusterSet references
   - Handle missing target resources gracefully
   - Provide clear error messages for invalid overrides

### Dependencies

- **JSON Patch Library**: `github.com/evanphx/json-patch/v5` for spec overrides
- **Unstructured Objects**: Enhanced use of `k8s.io/apimachinery/pkg/apis/meta/v1/unstructured`

## Testing Strategy

### Unit Tests
- Override application logic
- Resource identification and matching
- Error handling scenarios

### Integration Tests
- End-to-end override scenarios
- Rollout behavior with overrides
- Multiple cluster configurations

### E2E Tests
- Real cluster deployments with overrides
- Progressive rollout validation
- Performance impact assessment

## Migration and Rollout

### Phase 1: Development (4-6 weeks)
- Implement core API changes
- Develop override application logic
- Add basic validation

### Phase 2: Testing (2-3 weeks)
- Comprehensive test coverage
- Performance validation
- Documentation updates

### Phase 3: Release (1-2 weeks)
- Feature flag for gradual rollout
- Community feedback integration
- Production readiness validation

## Alternatives Considered

### 1. Cluster-Level Overrides
**Rejected**: While providing fine-grained control, cluster-level overrides would result in verbose configurations and reduced maintainability for large-scale deployments. ClusterSet-level overrides provide better logical grouping.

### 2. Placement-Level Overrides
**Rejected**: Would require changes to Placement API and doesn't provide the right level of granularity for environment-based configurations.

### 3. Separate Override Resources
**Rejected**: Increases complexity and makes it harder to manage overrides alongside MWRS.

### 4. Template-Based Approach
**Deferred**: Could be added in Phase 3 as an advanced feature, but JSON patches provide more flexibility initially.


## Conclusion

ClusterSet-level manifest overrides will significantly enhance ManifestWorkReplicaSet's flexibility while maintaining its core strengths of scalable, progressive workload deployment. This feature addresses real-world use cases where one-size-fits-all deployments are insufficient, enabling more sophisticated multi-cluster application management with logical grouping.

The proposed design is backward-compatible, performant, and extensible, providing a more maintainable approach than cluster-level overrides while still offering the necessary granularity for environment-based configurations. This makes it a valuable addition to OCM's workload management capabilities.
