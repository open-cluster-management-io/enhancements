# Policy Generator

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
      [website](https://github.com/open-cluster-management/website/)

## Summary

The goal of this feature is to allow automatic creation of OCM configuration policies, placement,
and placement bindings files from Kubernetes resource object manifests (e.g. a ConfigMap YAML) and
be able to deploy those policy files using existing GitOps mechanisms. This feature can lower the
bar of policy configuration for customers.

## Motivation

Users already have existing Kubernetes YAML files, so this will give users the ability to easily
create tailored OCM policies from these YAML files rather than having to create individual policies
manually from each file. Additionally, the policies can live in the same Git repo as the YAML,
allowing for version control and simplicity.

### Goals

- Generic: No special handling is required. The configuration policy’s objectDefinition field will
  contain exactly the same content as the input Kubernetes resource object manifest.
- Gatekeeper: In the case of a manifest with a gatekeeper policy, the automation must add additional
  object templates to the policy to audit/report the results as shown in
  [policy-gatekeeper-allowed-external-ips.yaml](https://github.com/open-cluster-management/policy-collection/blob/main/community/CM-Configuration-Management/policy-gatekeeper-allowed-external-ips.yaml).
- Kyverno: In the case of a manifest with a Kyverno policy, the automation must add additional
  object templates to be able to audit/report the results as shown in
  [policy-check-reports.yaml](https://github.com/open-cluster-management/policy-collection/blob/main/community/CM-Configuration-Management/policy-check-reports.yaml).
- Source manifest patching: Supply a patch to the input manifest before it is applied to the policy.
- (Stretch Goal) Compliance Operator: In case of a manifest with a Compliance Operator
  ScanSettingBinding, the automation must add additional objects to enable detecting failures in
  compliance.

### Non-Goals

- None at this time

## Proposal

### User Stories

1. As a k8s admin, I want to convert my existing k8s configuration related resource YAML on Git into
   OCM policies.
2. As a k8s admin, I want to convert existing Gatekeeper constraints/constraint templates into OCM
   policies.
3. As a k8s admin, I want to convert existing Kyverno policies into OCM policies.
4. As a k8s admin, I want to convert existing Compliance Operator scans into OCM policies.
5. As a security/compliance focal, I want to be able to specify standards that a given policy
   applies to.

### Architecture

Kustomize is a template-free way to customize Kubernetes resource object manifests and is considered
an industry standard. Kustomize has the concept of generators, which generate resource object
manifests based on the input generator configuration. In order to leverage this concept, a Kustomize
generator plugin will be written that accepts policy configuration and generates a policy with one
or more ConfigurationPolicy objects.

Through Kustomize, users will be able to override any defaults or add/remove fields to the generated
policy by altering the generator plugin configuration and/or supplying Kustomize patches. For
example, if a user wants to generate a policy that creates a Kubernetes resource object (e.g.
ConfigMap) on managed clusters, a security engineer may want to override the default policy
standards annotation. Additionally, an SRE may want to add additional labels to the policy so that
it is easily identified. These customizations can be in separate Kustomize patch files per role or
combined in a single one. This flexibility allows the workflow to reflect an organization’s
standards and structure.

### User Experience

#### The Kustomization YAML File

Kustomize is configured with a `kustomization.yaml` file. To use the generator plugin, a user must
declare it under the generators section as shown in a `kustomization.yaml` file below:

```yaml
generators:
  - policyGenerator.yaml
patchesStrategicMerge:
  - input/patch.yaml
commonLabels:
  custom: myApp
```

The `policyGenerator.yaml` file is the generator configuration file that Kustomize will provide. All
other customizations such as "patches" and "commonLabels" apply on the generated policy returned
from the new generator plugin.

#### Generator Defaults

By default, the generator plugin will fill in some default policy fields so that it’s easier for the
user when they are getting started.

The following policy fields will be required:

- name
- namespace

The following policy fields will be optional:

- annotations
- policy.open-cluster-management.io/categories - defaults to "CM Configuration Management"
- policy.open-cluster-management.io/controls - defaults to "CM-2 Baseline Configuration"
- policy.open-cluster-management.io/standards - defaults to "NIST SP 800-53" (to be confirmed)
- disabled - defaults to "false"
- remediationAction - defaults to "inform"
- severity - defaults to "low"

#### The Generator Configuration File

As shown above in the example `kustomization.yaml` file, a configuration file for the generator
plugin was provided in the `policyGenerator.yaml` file. The following example explains what each
field is used for:

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: PolicyGenerator
metadata:
  # Required. Used to uniquely identify the configuration file.
  name: ""

# Required if multiple policies are consolidated in a PlacementBinding so that the generator can
# create unique PlacementBinding names using the name given here.
placementBindingDefaults:
  # Set an explicit placement binding name to use rather than rely on the default.
  name: ""

# Required. Any default value listed here can be overridden under an entry in the policies array
# except for "namespace".
policyDefaults:
  # Optional. Array of categories to be used in the policy.open-cluster-management.io/categories
  # annotation. This defaults to ["CM Configuration Management"].
  categories:
    - "CM Configuration Management"
  # Optional. Array of controls to be used in the policy.open-cluster-management.io/controls
  # annotation. This defaults to ["CM-2 Baseline Configuration"].
  controls:
    - "CM-2 Baseline Configuration"
  # Optional. This determines if a single configuration policy should be
  # generated for all the manifests being wrapped in the policy.
  # If set to false, a configuration policy per manifest will be generated.
  # This defaults to true.
  consolidateManifests: true
  # Optional. When the policy references a Kyverno policy manifest, this determines if an additonal
  # configuration policy should be generated in order to receive policy violations in Open Cluster
  # Management when the Kyverno policy has been violated. This defaults to true.
  informKyvernoPolicies: true
  # Required. The namespace of all the policies.
  namespace: ""
  # Optional. The placement configuration for the policies. This defaults to a placement
  # configuration that matches all clusters.
  placement:
    # To specify a placement, specify key:value pair cluster selectors. (See placementRulePath to
    # specify an existing file instead.)
    clusterSelectors: {}
    # Optional. Specifying a name will consolidate placement rules that contain the same cluster
    # selectors.
    name: ""
    # To reuse an existing placement rule, specify the path here relative to the kustomization.yaml
    # file. If given, this placement rule will be used by all policies by default. (See
    # clusterSelectors to generate a new Placement instead.)
    placementRulePath: ""
  # Optional. The remediation action ("inform" or "enforce") for each configuration policy. This
  # defaults to "inform".
  remediationAction: "inform"
  # Optional. The severity of the policy violation. This defaults to "low".
  severity: "low"
  # Optional. Array of standards to be used in the policy.open-cluster-management.io/standards
  # annotation. This defaults to ["NIST SP 800-53"].
  standards:
    - "NIST SP 800-53"

# Required. The list of policies to create along with overrides to either the default values or, if
# set, the values given in policyDefaults.
policies:
  # Required. The name of the policy to create.
  - name: ""
    # Required. The list of Kubernetes resource object manifests to include in the policy.
    manifests:
      # Required. Path to a single file or a flat directory of files relative to the
      # kustomization.yaml file.
      - path: ""
        # Optional. A Kustomize patch to apply to the manifest(s) at the path. If there
        # are multiple manifests, the patch requires the apiVersion, kind, metadata.name,
        # and metadata.namespace (if applicable) fields to be set so Kustomize
        # can identify the manifest the patch applies to.
        patches:
          # Optional: Only required when there are multiple manifests in the path.
          - apiVersion: ""
            # Optional: Only required when there are multiple manifests in the path.
            kind: ""
            metadata:
              # Optional: Only required when there are multiple manifests in the path.
              name: ""
              # Optional: Only required when there are multiple manifests in the path and it's a
              # manifest to a namespaced resource.
              namespace: ""
              # An example modification to the manifest
              annotations:
                friends-character: Chandler Bing
    # Optional. (See policyDefaults.categories for description.)
    categories:
      - "CM Configuration Management"
    # Optional. Determines the policy controller behavior when comparing the manifest to objects on
    # the cluster ("musthave",  "mustonlyhave", or "mustnothave"). Defaults to "musthave".
    complianceType: "musthave"
    # Optional. (See policyDefaults.controls for description.)
    controls:
      - "CM-2 Baseline Configuration"
    # Optional. (See policyDefaults.disabled for description.)
    disabled: false
    # Optional. (See policyDefaults.informKyvernoPolicies for description.)
    informKyvernoPolicies: true
    # Optional. (See policyDefaults.consolidateManifests for description.)
    consolidateManifests: true
    # Optional. Determines the list of namespaces to check on the cluster for the given manifest. If
    # a namespace is specified in the manifest, the selector is not necessary. This defaults to no
    # selectors.
    namespaceSelector:
      include: []
      exclude: []
    # Optional. (See policyDefaults.placement for description.)
    placement: {}
    # Optional. (See policyDefaults.remediationAction for description.)
    remediationAction: ""
    # Optional. (See policyDefaults.severity for description.)
    severity: "low"
    # Optional. (See policyDefaults.standards for description.)
    standards:
      - "NIST SP 800-53"
```

#### Example Workflow

- A security engineer, SRE, infrastructure engineer, or etc. creates a pull request (PR) with the
  Kubernetes resource object manifests that should be wrapped in a policy.
- Based on the organization policy, the PR is reviewed and merged in the organization's Git
  repository.
- A security engineer, SRE, infrastructure engineer, or etc. creates a PR with the
  `kustomization.yaml` file, the generator plugin configuration, and any Kustomize patches they
  would like to apply to the generated policies.
- Based on the organization policy, the PR is reviewed and merged in the organization's Git
  repository.
- A SRE then configures an OCM subscription to apply the generated policies using GitOps. This
  subscription would be limited to just the policy directory created in the previous step.

#### Basic Example

**INPUT**

- **`kustomization.yaml`**
  ```yaml
  generators:
    - ./policyGenerator.yaml
  ```
- **`policyGenerator.yaml`**
  ```yaml
  apiVersion: policy.open-cluster-management.io/v1
  kind: PolicyGenerator
  metadata:
    name: policy-generator-name
  placementBindingDefaults:
    name: my-placement-binding
  policyDefaults:
    namespace: my-policies
    placement:
      clusterSelectors:
        cloud: red hat
    remediationAction: inform
    severity: medium
  policies:
    - name: policy-app-config-aliens
      disabled: false
      manifests:
        - path: ./configmap-aliens.yaml
          patches:
            - apiVersion: v1
              kind: ConfigMap
              metadata:
                labels:
                  chandler: bing
      remediationAction: enforce
  ```
- **Manifest**
  ```yaml
  # Taken from https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/
  ---
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: game-config-aliens
    namespace: default
  data:
    game.properties: |
      enemies=aliens
    ui.properties: |
      color.good=purple
  ```

**OUTPUT**

```yaml
apiVersion: apps.open-cluster-management.io/v1
kind: PlacementRule
metadata:
  name: placement-policy-app-config-aliens
  namespace: my-policies
spec:
  clusterConditions:
    - status: "True"
      type: ManagedClusterConditionAvailable
  clusterSelector:
    matchExpressions:
      - key: cloud
        operator: In
        values:
          - red hat
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: my-placement-binding
  namespace: my-policies
placementRef:
  apiGroup: apps.open-cluster-management.io
  kind: PlacementRule
  name: placement-policy-app-config-aliens
subjects:
  - apiGroup: policy.open-cluster-management.io
    kind: Policy
    name: policy-app-config-aliens
---
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  annotations:
    policy.open-cluster-management.io/categories: CM Configuration Management
    policy.open-cluster-management.io/controls: CM-2 Baseline Configuration
    policy.open-cluster-management.io/standards: NIST SP 800-53
  name: policy-app-config-aliens
  namespace: my-policies
spec:
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: policy-app-config-aliens
        spec:
          object-templates:
            - complianceType: musthave
              objectDefinition:
                apiVersion: v1
                data:
                  game.properties: |
                    enemies=aliens
                  ui.properties: |
                    color.good=purple
                kind: ConfigMap
                metadata:
                  labels:
                    chandler: bing
                  name: game-config-aliens
                  namespace: default
          remediationAction: enforce
          severity: medium
```

#### Example - Kyverno

**INPUT**

- **`kustomization.yaml`**
  ```yaml
  generators:
    - ./policyGenerator.yaml
  ```
- **`policyGenerator.yaml`**
  ```yaml
  apiVersion: policy.open-cluster-management.io/v1
  kind: PolicyGenerator
  metadata:
    name: policy-generator-name
  policyDefaults:
    namespace: my-policies
    placement:
      clusterSelectors:
        cloud: cumulonimbus
    remediationAction: inform
    severity: medium
  policies:
    - name: policy-check-for-labels
      manifests:
        - path: ./my-kyverno-policy.yaml
  ```
- **Manifest**
  ```yaml
  apiVersion: kyverno.io/v1
  kind: ClusterPolicy
  metadata:
    name: require-labels
  spec:
    validationFailureAction: audit
    rules:
      - name: check-for-labels
        match:
          resources:
            kinds:
              - Pod
        validate:
          message: "The label `friends-character` is required."
          pattern:
            metadata:
              labels:
                friends-character: "?*"
  ```

**OUTPUT**

```yaml
apiVersion: apps.open-cluster-management.io/v1
kind: PlacementRule
metadata:
  name: placement-policy-check-for-labels
  namespace: my-policies
spec:
  clusterConditions:
    - status: "True"
      type: ManagedClusterConditionAvailable
  clusterSelector:
    matchExpressions:
      - key: cloud
        operator: In
        values:
          - cumulonimbus
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: binding-policy-check-for-labels
  namespace: my-policies
placementRef:
  apiGroup: apps.open-cluster-management.io
  kind: PlacementRule
  name: placement-policy-check-for-labels
subjects:
  - apiGroup: policy.open-cluster-management.io
    kind: Policy
    name: policy-check-for-labels
---
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  annotations:
    policy.open-cluster-management.io/categories: CM Configuration Management
    policy.open-cluster-management.io/controls: CM-2 Baseline Configuration
    policy.open-cluster-management.io/standards: NIST SP 800-53
  name: policy-check-for-labels
  namespace: my-policies
spec:
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: policy-check-for-labels
        spec:
          object-templates:
            - complianceType: musthave
              objectDefinition:
                apiVersion: kyverno.io/v1
                kind: ClusterPolicy
                metadata:
                  name: require-labels
                spec:
                  rules:
                    - match:
                        resources:
                          kinds:
                            - Pod
                      name: check-for-labels
                      validate:
                        message: The label `friends-character` is required.
                        pattern:
                          metadata:
                            labels:
                              friends-character: ?*
                  validationFailureAction: audit
          remediationAction: inform
          severity: medium
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: inform-kyverno-require-labels
        spec:
          namespaceSelector:
            exclude:
              - kube-*
            include:
              - "*"
          object-templates:
            - complianceType: mustnothave
              objectDefinition:
                apiVersion: wgpolicyk8s.io/v1alpha2
                kind: ClusterPolicyReport
                results:
                  - policy: require-labels
                    result: fail
            - complianceType: mustnothave
              objectDefinition:
                apiVersion: wgpolicyk8s.io/v1alpha2
                kind: PolicyReport
                results:
                  - policy: require-labels
                    result: fail
          remediationAction: inform
          severity: medium
```

### Test Plan

- Unit tests for all code
- E2E/integration tests to ensure that policies being generated:
  - Can be deployed to the policy framework
  - Produce expected results when deployed

### Graduation Criteria

#### Alpha

1. The proposal is reviewed and accepted
2. Implementation is completed to support the primary goals
3. Test cases developed to demonstrate that the user stories are fulfilled
4. Update code to use Placement API instead of PlacementRule

#### Beta

1. E2E/Integration tests complete
