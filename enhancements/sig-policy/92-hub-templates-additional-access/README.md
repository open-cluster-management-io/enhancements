# Support for granting additional access to Hub templates  

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
      [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

Hub templates enable dynamic value injection into the policies on the Hub cluster, before the policy is distributed  to the managed clusters.  Templates support accessing Kubernetes objects and reading their fields, for example `{{hub fromConfigMap "" "test-cmap" "test-field1" hub}}` reads the value of the field "test-field1" from ConfigMap "test-configmap".  However,  Hub templates are limited to accessing Kubernetes objects that are in the same namespace as the policy containing the template. A new field `serviceAccountName` will be introduced to enable Hub templates to access objects outside the policy namespace based on the permissions bound to this ServiceAccount. If the ServiceAccount has access to a given Kubernetes object then that object can be accessed by all  Hub templates in the policy. 

## Motivation

Hub Policy templates are limited to only accessing Kubernetes objects that are in the policy namespace, this was done to prevent unauthorized access of Hub resources by policy owners but this also prevents  use of cluster-scoped resources and resources in other namespaces even if the Policy owner has access to them.  To workaround, in some cases policies are forced to be created in the same namespace as the object that it need to access or additional policies are created and distributed to `local-cluster` just to copy objects between namespaces  using managed cluster templates which can access any resource on the cluster. These approaches not only clutter the policy organization and definition, they may not work for all use cases.

### Goals

1. Provide a way to grant addtitional access to Policies
2. Verify Hub templates are authorized to access the objects referenced based on the access granted to the policy.

### Non-Goals

N/A

## Proposal

An optional field of `serviceAccountName` will be added to the Policy object. Inorder to allow Hub templates to access objects outside the policy namespace, a ServiceAccount should be created in the same namespace as the policy and relevant permissions should be set through roles and rolebindings with this ServiceAaccount as the subject. This service account's name should then be specified in the Policy object to map its permissions to the Hub templates in the policy. When the `serviceAccountName` field is not set, the default behavior of accessing objects only within the policy namespace is observed.

```yaml
# Policy with serviceAccountName field set and Hub template accessing a ConfigMap (is in ns 'default' which is not in the policy namespace i.e 'test-policies')
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: test-config
  namespace: test-policies
spec:
  disabled: false
  remediationAction: enforce
  serviceAccountName: testsa
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: test-config
        spec:
          remediationAction: enforce
          severity: high
          object-templates: 
          - complianceType: musthave
            objectDefinition:
              apiVersion: v1
              data:
                field1: '{{hub fromConfigMap "default" "test-configmap" "test-field1" hub}}'
                field2: '{{hub fromConfigMap "default" "test-configmap" "test-field1" hub}}'
              kind: ConfigMap
              metadata:
                name: my-configmap
                namespace: test
---
# Service Account in the same namespaces as the Policy
apiVersion: v1
kind: ServiceAccount
metadata:
  name: testsa
  namespace: test-policies
---
# ClusterRole with permissions to read ConfigMap
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: test-reader
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
---
# Rolebinding that gives the ServiceAccount permissions to read ConfigMap   in namespace "default"
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: sample-res-reader-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: testsa
    namespace: test
roleRef:
  kind: ClusterRole
  name: test-reader
  apiGroup: rbac.authorization.k8s.io
---
```

### User Stories

#### Story 1

As a Policy user, I want the Hub templates in my policies to be able to safely access Kubernetes objects outside the policy namespace also.

### Implementation Details/Notes/Constraints [optional]

During policy processing, PolicyPropagator controller on the Hub should pass the serviceAccountName to the go-template-utils's  TemplateResolver, which would then execute SubjectAccessReview API queries  with the specified serviceAccountName as the subject to verify access to every object referenced in the templates.

```yaml
#example  SAR for the policy and hub-templates defined above
apiVersion: authorization.k8s.io/v1
kind: SubjectAccessReview
spec:
  resourceAttributes:
    group: ""
    resource: configmaps
    name: test-configmap
    verb: get
    namespace: default
  user: system:serviceaccount:test-policies:testsa
```
### Risks and Mitigation

### Open Questions [optional]

N/A

### Test Plan

**Note:** _Section not required until targeted at a release._

### Graduation Criteria

It would be GA in the release after implementation.

### Upgrade / Downgrade Strategy

Specifying `serviceAccountName` is optional, When not it defaults to the existing behavior.

### Version Skew Strategy

## Implementation History

N/A

## Drawbacks

See the Risks section.

## Alternatives

N/A

## Infrastructure Needed [optional]

N/A
