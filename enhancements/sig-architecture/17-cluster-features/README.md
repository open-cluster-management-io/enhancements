# Managed cluster feature labels

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

There are requirements about selecting `ManagedClusters` based on the certain features of the managed cluster by `Placement` API. These requirements include:

1. Select a set of the clusters that a certain `ManagedClusterAddon` is enabled. For instance, a user wants to install an application that needs multicluster service connectivity, the use will need to select a set of clusters that `Submariner` addon is enabled by using `Placement` API.
2. Select a set of the clusters with a certain capability. For instance, a user wants to deploy an AI workload to a set of the clusters in which GPUs are installed.

## Motivation

This proposal is to standardise the labels and `ClusterClaims` that are categorized as features of a `ManagedCluster`.  The idea is inspired by [node features](https://kubernetes-sigs.github.io/node-feature-discovery/) in kubernetes.

### Goals

- define the format of labels and `ClusterClaims` for `ManagedClusters` to represent a feature in managed cluster
- implement addon as feature labels in `registration`.

## Proposal

### Design Details

Features can be classified to 3 categories:

1. Features enabled by hub cluster: these features are enabled by user on the hub cluster to a certain set of managedclusters. For example, when user enables an addon on a managed cluster, the cluster will have a feature relates to the addon. Each of these features can be represented as a label on the `ManagedCluster`.

2. Existing node features on mananged cluster: these are features discovered by node feature discovery daemon in kubernetes cluster and has been represented as a label in the node of the managed cluster. Most of such features are physical device related. Each of these features should be represented as a `ManagedClusterClaim`.

3. Existing cluster-wide features on managed cluster: these features resides in  managed cluster, but are not specific to a certain node. Examples includes whether a global load balancer service is supported in this managed cluster or a certain storage can be used with `PVC`. Each of these features should be represented as a `ManagedClusterClaim`.


### Feature labels

Feature labels should have the prefix of `feature.open-cluster-management.io`. Each addon will be represented as a feature label in the format of `feature.open-cluster-management.io/addon-<addon name>`. The values of the lable are `available`, `unhealthy`, `unreachable`. For example, the `application-manager` addon will be represented as a label on `ManagedCluster` as:

```
feature.open-cluster-management.io/addon-application-manager: available
```

`registration` controller will update these labels based on the existence and status of each `ManagedClusterAddon` resources.

### Feature claims

Features in the managed cluster should be represented as `ClusterClaim` instead of labels. A node feature in a managed cluster should be represented as a `ClusterClaim` with the name of `<feature name>.node.feautre.open-cluster-management.io`. For instance, if a node in the cluster has a feature of `feature.node.kubernetes.io/iommu-enabled: true`. A corresponding cluster claim should be created as

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ClusterClaim
metadata:
  name: iommu-enabled.node.feautre.open-cluster-management.io
spec:
  value: true
```

A cluster featurre should be represented as a `ClusterClaim` with the name of `<feature name>.cluster.feautre.open-cluster-management.io`. For instance, another discovery agent can check the support of load balancer service and create a claim as:

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: ClusterClaim
metadata:
  name: lb-enabled.cluster.feautre.open-cluster-management.io
spec:
  value: true
```

The management of these two types of `ClusterClaim` will require additional discovery agent, so we will not implement them in the existing components. The details of the implementation is out of the scope and should be discussed in an additional proposal.

### Test Plan

Add integration/e2e test to ensure that

- when an addon is added, the label is set on the `ManagedCluster`
- when an addon is deleted, the label is removed from the `ManagedCluster`
- when the status of an addon is updated, the value of the corresponding label is updated.

### Graduation Criteria

#### Alpha
1. The proposal is reviewed and accepted;
2. Implementation is completed to support the minimum functionalities;
3. Develop test cases to demonstrate that the above user stories work correctly;
4. The feature is under `feature-discovery` feature gate and disabled by default

#### Beta
1. At least one component uses this feature;
2. The feature is under `feature-discovery` feature gate and enabled by default
3. At least one implementation of node/cluster feature discovery agent.

#### GA
1. Pass the performance/scalability testing;

### Upgrade Strategy

This will be new features, so no need to consider upgrade.

### Version Skew Strategy

This will be new features, so no need to consider version skew.

## Alternatives

In the `Placement` API, we introduce a new field `addons` to indicate that the selected clusters should have those addons enabled (or disabled). This will require `Placement` controllers to watch all the `ManagedClusterAddon` resources. 