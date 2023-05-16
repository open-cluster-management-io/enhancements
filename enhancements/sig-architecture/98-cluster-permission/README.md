# New API: ClusterPermission

## Release Signoff Checklist

- [ ] Enhancement is `implemented`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal will be adding a new OCM API named `ClusterPermission`
which helps the administrators to automatically distribute RBAC resources
to the managed clusters and manage the lifecycle of those RBAC resources
including Role, ClusterRole, RoleBinding, ClusterRoleBinding.
A valid `ClusterPermission` resource should be in a "cluster namespace" and
the associated RBAC resources will be deliver to the associated managed cluster
with that "cluster namespace".

## Users of this API

- The hub cluster admin is allowed to crud `ClusterPermission` in the managed cluster namespace.
- By default users and controllers do not have privilege to crud `ClusterPermission`
in the managed cluster namespace unless their permission is elevated through Role and ClusterRole.

## Motivation

### Distribute and manage multi-cluster RBAC resources

The `ClusterPermission` will help us easily distribute
RBAC resources to the managed cluster. The `ClusterPermission` controller using
`ManifestWork` API will ensuring, updating, removing the RBAC resources
from each of the managed cluster. 
The `ClusterPermission` API will protect these distributed RBAC resources from unexpected 
modification and removal.

### Enhance the usability of the existing ManagedServiceAccount API

After ensuring the service-accounts' presence on each managed cluster using the existing
[ManagedServiceAccount](https://github.com/open-cluster-management-io/managed-serviceaccount)
API, the `ClusterPermission` can reference the `ManagedServiceAccount` and use that
as the Subject for the RBAC resources. This will help regulate the access level for
the `ManagedServiceAccount`.

### New API instead of just using ManifestWork API

There are several reasons why we should create a new API instead of using `ManifestWork`:
- `ManifestWork` API is meant to be a multicluster workload delivery primitive type.
- `ClusterPermission` API is a higher level concept that users can understand its multicluster
RBAC resources scope easily.
- `ManifestWork` API payload does not have RBAC specific fields definitions.
- `ClusterPermission` can determine the `ManagedServiceAccount` subject and create `ManifestWork`
with the correct RoleBinding/ClusterRoleBinding subject.

## Goals & Non-Goals

### Goals

- Manage the lifecycle of RBAC resources across multiple cluster.
- Allows the typical Subject Kind: Group, ServiceAccount, and User
for RoleBinding and ClusterRoleBinding.
- Allows the `ManagedServiceAccount` as Subject Kind 
for RoleBinding and ClusterRoleBinding.

### Use cases

The primary use case of the `ClusterPermission` is to allow a controller on the hub cluster to
gain access to the managed cluster API server with authorization regulations.
The access to the managed cluster API server is done via the ServiceAccount token 
managed by the `ManagedServiceAccount` and the access level of the ServiceAccount
is handled by the `ClusterPermission`.

### Future goals

- It is currently assumed that the user of `ClusterPermission` is either a cluster admin or
a user who can create `ClusterPermission` in the hub cluster's managed "cluster namespace".
It will be useful if there is an approval process that a cluster admin can approve,
which will allow for other users to create `ClusterPermission` resources.
- Support creating a RoleBinding/ClusterRoleBinding without defining any Role/ClusterRole
rules.

## Design

### Component & API

We propose to adding a new custom resource named
`ClusterPermission` introduced into OCM by this proposal:

A sample of the `ClusterPermission` will be:

```yaml
apiVersion: rbac.open-cluster-management.io/v1alpha1
kind: ClusterPermission
metadata:
  name: my-managed-rbac
  namespace: cluster1
spec:
  clusterRole:
    rules:
    - apiGroups: ["apps"]
      resources: ["deployments"]
      verbs: ["update"]
  clusterRoleBinding:
    subject:
      apiGroup: rbac.authorization.k8s.io
      kind: Group
      name: system:serviceaccounts:qa
  roles:
  - namespace: default
    rules:
    - apiGroups: ["apps"]
      resources: ["deployments"]
      verbs: ["update"]
  - namespaceSelector:
      matchExpressions:
      - key: foo.com/managed-state
        operator: In
        values:
        - managed
    rules:
    - apiGroups: [""]
      resources: ["configmaps"]
      verbs: ["update"]
  roleBindings:
  - namespace: kube-system
    roleRef:
      kind: ClusterRole
    subject:
      apiGroup: authentication.open-cluster-management.io
      kind: ManagedServiceAccount
      name: managed-sa-sample
  - namespace: default
    roleRef:
      kind: Role
    subject:
      apiGroup: rbac.authorization.k8s.io
      kind: Group
      name: system:serviceaccounts
  - namespaceSelector:
      matchExpressions:
      - key: foo.com/managed-state
        operator: In
        values:
        - managed
    roleRef:
      kind: Role
    subject:
      apiGroup: rbac.authorization.k8s.io
      kind: User
      name: "alice@example.com"   
status:
  conditions:
    - type: RBACManifestWorkCreated
      status: True
    - type: SubjectPresent
      status: True    
```

The `ClusterPermission` resource is expected to be created under the
"cluster namespace" which is a namespace with the same name as
the managed cluster, the Role/RoleBinding/ClusterRole/ClusterRoleBinding
delivered to the managed cluster will have the same name as the
`ClusterPermission` resource.

We propose adding a new hub cluster controller that watches
and reconcile `ClusterPermission`. When `ClusterPermission` is reconciled, create/update
a `ManifestWork` with the payload of Roles, ClusterRole,
RoleBindings, and ClusterRoleBinding. All the RBAC resources will have
the same name as the `ClusterPermission`. The `ManifestWork` owner is set to 
`ClusterPermission` so that when the  `ClusterPermission` is deleted, the `ManifestWork` 
will be garbage collected, which will trigger the removal of all the RBAC 
related resources in the managed cluster. It's expected that the executor 
of the `ManifestWork` is the work agent. Therefore, a Role of the following
content will be needed for the controller:

```
rules:
- apiGroups:
  - work.open-cluster-management.io
  resources:
  - manifestworks
  verbs:
  - execute-as
  resourcenames:
  - system:serviceaccount::klusterlet-work-sa
```

It is expected that the work agent has "escalate" and "bind" role verbs 
which allows it to manage the RBAC resources inside the `ManifestWork` payload.

*Note* If the feature gate `NilExecutorValidating` is enabled on the ClusterManager.
It is expected that the `ManifestWork` validator will populate that field so error
won't occur.

If the subject of the binding is
a kind `ManagedServiceAccount` then the binding will be updated to be
a kind `ServiceAccount` and the namespace will be the value of
`ManagedServiceAccount`'s managed cluster service account namespace.
The `ManagedServiceAccount` will be watched by the `ClusterPermission` controller.
If a previously exist `ManagedServiceAccount` is now deleted then
`ClusterPermission` the condition type `SubjectPresent` will be set to false.

A sample of the `ClusterPermission` using Role/ClusterRole references:

```yaml
apiVersion: rbac.open-cluster-management.io/v1alpha1
kind: ClusterPermission
metadata:
  name: my-managed-rbac-with-refs
  namespace: cluster1
spec:
  roleRefs:
  - kind: Role
    name: pod-reader
    namespace: default
    apiGroup: rbac.authorization.k8s.io
  - kind: ClusterRole
    name: secret-reader
    apiGroup: rbac.authorization.k8s.io
...
```

You can also use references Role/ClusterRole on the hub cluster and the same
Role/ClusterRole will be delivered to the managed cluster.

As an enhancement to the usability of the existing `ManagedServiceAccount`,
there is no "Placement Reference" in the `ClusterPermission` API because we want to
follow the same convention as the `ManagedServiceAccount` API. A `ClusterPermission` 
resource lives in the hub cluster managed cluster "cluster namespace" similar
to `ManagedServiceAccount`'s design and implementation.

### Test Plan

- Unit tests
- Integration tests

### Graduation Criteria
#### Alpha
At first, This proposal will be in the alpha stage and needs to meet
1. The new APIs are reviewed and accepted;
2. Implementation is completed to support the functionalities;
3. Develop test cases to demonstrate this proposal works correctly;

#### Beta
1. Need to revisit the API shape before upgrading to beta based on user feedback.

### Upgrade / Downgrade Strategy
TBD

### Version Skew Strategy
N/A

## Alternatives

We can create a ClusterSetPermission API which grants permissions to managed clusters
that belong to a managed cluster set.
Pros:
- Reduce the number of permission CRs needed to set permission to multiple managed clusters.
Cons:
- Users will always need to define a cluster set to set the permission.
Since `ClusterPermission` is trying to follow the same principal as `ManagedServiceAccount`
and `ManagedServiceAccount` does not need to have a cluster set, `ClusterPermission` will
be on a per managed cluster basis for now.
