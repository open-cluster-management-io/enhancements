# Addon: Managed ServiceAccount

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal will be adding a new OCM addon named `managed-serviceaccount` 
which helps the administrators to automatically distribute service-accounts
to the managed clusters and manage the lifecycle of those service-accounts 
including token rotation, ensuring token validity, destroying the token, etc.
The token of the distributed service-account will also the periodically
projected / reported to the hub cluster as a secret resource with the same
name and the same namespace as the corresponding "ManagedServiceAccount" 
resource. A valid "ManagedServiceAccount" should be either in a "cluster 
namespace" or a "cluster-set namespace" which is bind to a valid 
"ManagedClusterSet" resource.

## Motivation

### Distribute and manage a multi-cluster service-account

Creating such a "multi-cluster service-account" will help us easily distribute
service-accounts to the managed cluster and on the other hand destroying the 
"multi-cluster service-account" will manage the lifecycle of ensuring, 
revoking, removing the service-accounts from each of the managed cluster. Not 
only the addon will protect these distributed service-account from unexpected 
modification and removal, and also it will synchronize the lifecycle of
the service-accounts according to its parent resource "multi-cluster 
service-account".

### Token projection

After ensuring the service-accounts' presence on each managed cluster, we will
be able to dynamically asking the managed cluster for signing a token w/ expected
token audience prescribed. The existing `authenticationv1.TokenRequest` api is 
particularly designed for such usages, so it's rather simple for the addon 
to request new token from each managed cluster as long as rbac permission of
token request is granted.

### Authenticate hub->spoke outbound requests

In some cases when a component (such as an operator) deployed in the hub 
cluster needs to proactively invoke the kube-apiservers of the managed 
clusters, the outbound requests ought to be authenticated before being 
served by the kube-apiserver. Then attaching the token to those outbound 
requests will pass the managed cluster's authentication.


## Goals & Non-Goals

### Goals

- Manage the lifecycle of service-accounts across multiple cluster.
- Persist the token of the service-account back to the hub cluster.

### Non-Goals

- The addon will not be an external token signer.
- The addon will not be managing rbac authorization for the managed tokens
  (i.e. authorization).

## Design

### Component & API

We purpose to add two components by this design:

1. An addon manager named `managed-serviceaccount-addon-manager` which 
   is developed based on addon-framework in the hub cluster.
2. Addon agents in each of the managed cluster named `managed-serviceaccount
   -addon-agent`.

Additionally there will also be new custom resource named 
`ManagedServiceAccount` introduced into OCM by this proposal:

A sample of the `ManagedServiceAccount` will be:

```yaml
# `ManagedServiceAccount`is cluster-scoped
apiVersion: proxy.open-cluster-management.io/v1alpha1
kind: ManagedServiceAccount
metadata:
  name: my-foo-operator-serviceaccount
  namespace: cluster-1
spec:
  audiences: 
  - foo
  - bar
  rotation:
    enabled: true
    validity: 4320h0m0s
status:
  conditions:
    - type: AllServiceAccountPresent
      status: True
```


The `ManagedServiceAccount` resource is expected to be created under:

- "cluster namespace": a namespace with the same name as the managed cluster, 
  which indicates that the `ManagedServiceAccount` will be bound to a service
  account with the same name as the `ManagedServiceAccount` and under the 
  namespace where the "managed-serviceaccount-addon-agent" deploys.
  
- "cluster-set namespace": a namespace with valid clusterset bounded, which
  indicates the `ManagedServiceAccount` to replicate the service accounts (with
  the same name as the `ManagedServiceAccount` and the namespace where the agent
  lives) to all the clusters connected to that cluster set.

A sample of the reported/persisted service-account token will be:

(We make the token persisted as a secret resource here so that users
can mount it easier)

```yaml
apiVersion: v1
kind: Secret
metadata: 
  ownerReferences:
    - apiVersion: proxy.open-cluster-management.io/v1alpha1
      resource: managedserviceaccounts
      name: my-foo-operator-serviceaccount
  name: my-foo-operator-serviceaccount
  namespace: cluster-1
type: Opaque
data:
  ca.crt: ...
  token: ...
```

Based on the extensibility provided by the addon-framework, the addon manager
will be doing the following things upon detecting the presence of the addon:

- Install addon-agent to each managed cluster.
- Grant the addon-agent the permission of writing secrets to its "cluster 
  namespace" in the hub cluster.
- Grant the addon-agent the permission of list-watching `ManagedServiceAccount`
  in the hub cluster and updating the status subresource.
- Grant addon-agent the permission of writing service-account in the managed
  cluster.
- Grant addon-agent the permission of issuing `authenticationv1.TokenRequest`
  in the managed cluster.
  
Then addon-agent will be list-watching the `ManagedServiceAccount` in its 
"cluster namespace" after started and dynamically provisions local service-accounts 
and report the token back to the hub cluster based on the rotation policy.

### Consuming the tokens

#### By directly list-watching the secrets

The component consuming the tokens is expected to list-watch the secrets w/i
across the cluster namespaces. Optionally the project can provide a library for
discovering and reading tokens dynamically from the hub cluster.

#### By copying and mounting the secrets

The other consumers working in the hub cluster are supposed to copy the token
secret by themselves e.g. in order to the mount the token.

### Test Plan

- Unit tests
- Integration tests