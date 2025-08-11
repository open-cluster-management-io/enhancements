# ManagedClusterSet Override

## Release Signoff Checklist
 
- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary
The proposed work enhances the managedClusterSet API to support managedClusterSet Override.

## Motivation
Today, managedClusterSet is designed to group the managedClusters, a managedCluster can only be included in one managedClusterSet, by setting a label `cluster.open-cluster-management.io/clusterset` on the managedCluster.
A user must have explicit RBAC rules, `managedclustersets/join`, to add such labels.
There are also raising requirements on using managedClusterSet for the following purposes. Within a purpose, the managedClusterSet must be exclusive, but for different purposes it can overlap.
- As cluster admin, I want to create a managedClusterSet which includes all clusters in the apac region. Then I can apply certain configurations to all clusters in apac.
- As a SRE team member, I want to use a managedClusterSet for Disaster Recovery purposes. The managedClusterSet should select two or more clusters in different regions. 
- As a DEV team member, I want to use a managedClusterSet to select clusters based on the middleware enabled on these clusters, so I could run special applications in these clusters.

## Proposal

### Changes on managedClusterSet APIs.

Currently, managedClusterSets use the label `cluster.open-cluster-management.io/clusterset:<ManagedClusterSet Name>` to select managedClusters. 
But for some use cases, I want to use other labels to select managedClusters in a managedClusterSets.

So, In this proposal, we change the managedClusterSets spec and want to provide a flexible way to select managedClusters. 

```go
type ManagedClusterSetSpec struct {
    // Selector represents a selector of ManagedClusters by labels.
    ClusterSelector ManagedClusterSelector  `json:"clusterSelector"`
}

type ManagedClusterSelector struct{
    // SelectorType could only be "ExclusiveClusterSetLabel" or "LabelSelector"
    // "ExclusiveClusterSetLabel" means to use label "cluster.open-cluster-management.io/clusterset:<ManagedClusterSet Name>"" to select target clusters.
    // "LabelSelector" means to use LabelSelector to select target clusters
    // +kubebuilder:validation:Enum=ExclusiveClusterSetLabel
    // +kubebuilder:default:=ExclusiveClusterSetLabel
    // +required
    SelectorType SelectorType `json:"selectorType,omitempty"`

    // LabelSelector define the general labelSelector which clusterset will use to select target managedClusters
    LabelSelector *metav1.LabelSelector `json:"labelSelector"`
}

type SelectorType string

const (
    LabelSelector             SelectorType = "LabelSelector"
	  ExclusiveClusterSetLabel  SelectorType = "ExclusiveClusterSetLabel"
)
```

### RBAC
Actually, what we discuss about RBAC is `who can add/remove a managedCluster to/from a managedClusterSet.`
Currently, if a managedCluster which have label `cluster.open-cluster-management.io/clusterset:<ManagedClusterSet Name>` means the cluster is in a managedClusterSet.
And If I want to add a managedCluster to a managedClusterSet, I must have `create` permission for subresource `managedclustersets/join` with resource name `<ManagedClusterSet Name>`.

In this proposal, managedClusterSet could use two ways to select target clusters, `ExclusiveClusterSetLabel` and `LabelSelector`.
1. ExclusiveClusterSetLabel
In this selector, managedClusterSet select target clusters using label `cluster.open-cluster-management.io/clusterset:<ManagedClusterSet Name>`. And the RBAC model will not change. If I want to add a managedCluster to a managedClusterSet, I must have `create` permission for subresource `managedclustersets/join` with resource name `<ManagedClusterSet Name>`.

2. LabelSelector
With the `LabelSelector`, any labels may be used to select target clusters. And we will not have external RBAC control for this kind of clustersets.

We will use the following questions to consider the RBAC control.

#### What purposes will managedClusterSet may be used for
a. Create a managedClusterSet for disaster recovery purposes, which should include specific clusters.
b. Create a managedClusterSet which should include all clusters in one region or cloud provider.
c. Create a managedClusterSet which has the same capacity(like: all clusters enabled middleware).
d. Create a managedClusterSet for each squad for resource group purposes.

#### Who can create a managedClusterSet
Same as the current way, Anyone who has `create` permission to `managedclustersets` could create managedClusterSet.
But generally, Cluster admin should not give this permission to others. So only cluster admin could create the managedClusterSet.

#### What kinds of roles exist related to managedClusterSets and managedClusters
In General, there are three kinds of roles related to managedClusterSet.
1. Cluster admin
   - Cluster admin could `create`, `get`, `list`, `update`, `delete` all managedClusterSets.
   - Cluster admin could `bind` all managedClusterSets to all namespaces.
   - Cluster admin could `create`, `get`, `list`, `update`, `delete` all managedClusters.
   - Only Cluster admin could `accept` managedClusters.

2. Controller/agent which run in managedClusters
   - The Controller/agent should not `create`, `get`, `list`, `update`, `delete` managedClusterSets.
   - The Controller/agent should not `bind` managedClusterSet.
   - The Controller/agent only should `get` related managedCluster and `update` the managedCluster status and labels.

3. Non Cluster admin users
   - These users should `get` some managedClusterSets, and could not `create`, `update`, `delete`, `list` managedClusterSets.
   - These users should `bind` some managedClusterSets to their namespaces.
   - These users may need to `create` managedClusters(For Resource Group Purpose managedClusterSet[d], in each group, the group members may need to create the managedClusters). They could also `delete` these managedClusters, and `update` these managedClusters labels.

#### Who can add labels for managedClusters
1. Cluster admin
Cluster admin could add any labels to any managedClusters.

2. Controller/agent which run in managedClusters
Currently, the controller/agent may add labels like: `region: apac`, `cloud: Amazon`, `middlewareEnabled: true`. And these labels mean these clusters have some attributes.
- `region: apac` means the cluster is provisioned in apac region
- `cloud: Amazon` means the cluster is provisioned in Amazon cloud provider
- `middlewareEnabled: true` means this cluster enables the middleware.

3. Non Cluster admin users
Currently, non cluster admin users may also `add/update` some managedClusters labels. 

#### [Not Implement] Add restrictions for adding/updating/deleting labels to managedClusters
In this proposal, managedClusterSet could use any labels to select managedClusters. And the new added labels may lead to this managedClusters added to a managedClusterSet.

But for some managedClusterSet(like: Disaster Recovery ManagedClusterSet[a], Resource Group ManagedClusterSet[d]), we do not want any users to add their managedClusters to these managedClusterSets. 

So we propose adding restrictions for some labels, and if a user wants to add these kinds of labels, he/she must have a "special" permission.
Then the managedClusterSets which do not want anyone to join could use these labels to select managedClusters.

We propose adding restrictions for labels which key have prefix `cluster.open-cluster-management.io/` and `info.open-cluster-management.io/`
- `cluster.open-cluster-management.io/` is encouraged for non cluster admin users. Like for "Resource Group ManagedClusterSet[d]", the DEV team users may add a label like: `cluster.open-cluster-management.io/clusterset: dev`
- `info.open-cluster-management.io/` is encouraged for spoke agents, and the agents should have permissions on their own managedcluster to modify these kinds of labels.
  The label key should be: `info.open-cluster-management.io/region`, `info.open-cluster-management.io/cloud`, `info.open-cluster-management.io/vendor`

Then we need a way to define the "special" permission.
Investigate the existing rbac rule in current k8s, we use a rule which is similar to [csr-approver](https://github.com/kubernetes/website/blob/3ca34b9a3be454055ba234f1d2ff7d55809b5040/content/zh/examples/access/certificate-signing-request/clusterrole-approve.yaml#L25)

We define a subresource `managedclusters/label`, which resource name is the label of managedCluster. This subresource means user could add/remove the label to managedClusters.

For example: 
- This permission means the users could add/remove the label `cluster.open-cluster-management.io/clusterset:dev` to their managedClusters.

```yaml
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters/label"]
   resourceNames: ["cluster.open-cluster-management.io/clusterset:dev"]
   verbs: ["create"]
```

- This permission means the users could add/remove any labels to their managedClusters.
```yaml
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters/label"]
   resourceNames: ["*"]
   verbs: ["create"]
```

- This permission means the users could add/remove labels with `cluster.open-cluster-management.io/clusterset` key and arbitrary value to their managedClusters.
```yaml
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters/label"]
   resourceNames: ["cluster.open-cluster-management.io/clusterset:*"]
   verbs: ["create"]
```

Notes: The `managedclustersets/join` permission will be deprecated after the proposal applied, and the existing `managedclustersets/join` permission could be replaced with the following permission.
```yaml
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters/label"]
   resourceNames: ["cluster.open-cluster-management.io/clusterset:<ManagedClusterSet Name>"]
   verbs: ["create"]
```

#### Who can bind the managedClusterSet to a namespace
Same as the current way, anyone who has `managedclustersets/bind` permission could bind a managedClusterSet to a namespace.

### Examples
### Example1: As cluster admin, I want to create a managedClusterSet which includes all clusters in the apac region. Then I can apply certain configurations to all clusters in apac.
1. For each cluster, there should be an agent running on the managedCluster. The agent should have the following permissions:
- `update` permission to its related managedCluster
2. For each cluster in the apac region, related agents should add a label: `region:apac` to these managedClusters.
3. As cluster admin, I could create a managedClusterSet to select all managedClusters in `apac` region
```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSet
metadata:
  name: apacset
spec:
  clusterSelector:
    selectorType: LabelSelector
    labelSelector:
      matchLabels:
        region: apac
```

### Example2: As a DEV team member, I want to use a managedClusterSet to select clusters based on the middleware enabled on these clusters, so I could run special applications in these clusters.
1. For each cluster, there should be an agent running on the managedCluster. The agent should have the following permissions:
- `update` permission to its related managedCluster
2. For each cluster which enables the middleware, related agents should add a label: `middlewareEnabled:true` to these managedClusters.
3. Cluster admin create a managedClusterSet `middlewareenabledset`
```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSet
metadata:
  name: middlewareenabledset
spec:
  clusterSelector:
    selectorType: LabelSelector
    labelSelector:
      matchLabels:
        middlewareEnabled: true
```
4. Cluster admin should give following permissions to DEV team, so these team members could run applications in the managedClusterSet's clusters.
- `get` permission to managedClusterSet `middlewareenabledset`
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets"]
   resourceNames: ["middlewareenabledset"]
   verbs: ["get"]
- apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/bind"]
   resourceNames: ["middlewareenabledset"]
   verbs: ["create"]
```
5. As a DEV team member, I could `bind` the managedClusterSet `middlewareenabledset` to my namespace and run applications in the managedClusterSet.

### Example3: As cluster admin, I want to create two managedClusterSets for DEV and QA team. and the DEV/QA team members could deploy applications in their own managedClusterSet.
1. Cluster admin create two managedClusterSets for Dev team and QA team
```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSet
metadata:
  name: dev
spec:
  clusterSelector:
    selectorType: ExclusiveClusterSetLabel
```

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSet
metadata:
  name: qa
spec:
  clusterSelector:
    selectorType: ExclusiveClusterSetLabel
```

2. Cluster admin give the following permission to DEV team and QA team
Permission for DEV team
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets"]
   resourceNames: ["dev"]
   verbs: ["get"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/join"]
   resourceNames: ["dev"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/bind"]
   resourceNames: ["dev"]
   verbs: ["create"]
```

Permission for QA team
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: clustersetrole
rules:
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclusters"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets"]
   resourceNames: ["qa"]
   verbs: ["get"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/join"]
   resourceNames: ["qa"]
   verbs: ["create"]
 - apiGroups: ["cluster.open-cluster-management.io"]
   resources: ["managedclustersets/bind"]
   resourceNames: ["qa"]
   verbs: ["create"]
```
3. As a DEV/QA team member, I can `create` a managedCluster, and add label `cluster.open-cluster-management.io/clusterset:dev/qa` to the managedCluster.
4. As a DEV/QA team member, I could `bind` the managedClusterSet `dev/qa` to my namespace and run applications in the managedClusterSet.

## Test Plan
- Unit tests will cover the functionality of the controllers.
- Unit tests will cover the new APIs.
- And e2e tests will cover the following cases.
 - Create/update/delete managedClusterSet 
 - Create/update/delete managedClusters labels
 - Update managedClusterSet spec `clusterSelector` field

## Upgrade / Downgrade Strategy 
The new api is compatible with the previous version. So there is no external work needed when upgrading

## Graduation Criteria
### v1beta1
1. The new APIs is reviewed and accepted;
2. Implementation is completed;
