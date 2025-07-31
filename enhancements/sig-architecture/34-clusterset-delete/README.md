# ManagedClusterSet Delete policy
 
## Release Signoff Checklist
- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)
 
## Summary
The proposed work enhances the managedClusterSet API to handle managedClusterSet Delete policy.
 
## Motivation
Currently, when we add a cluster to a clusterset, we will add a label `cluster.open-cluster-management.io/clusterset:<Clusterset Name>` to the cluster. But when the clusterset is deleted, this label will not be removed.
 
So we need to design the delete policy for clusterset.
 
## Proposal Detail
Actually, we always use clusterset as a group of clusters. So when deleting a clusterset, there are some options for the delete policy.
1. When clusterset deleted, delete the clusterset labels in managedClusters.
2. When clusterset deleted, delete the managedClusters in clusterset.
3. Hold the clusterset deletion if there are clusters in the set.

### API Change
We add a new field `DeletePolicy` in `ManagedClusterSetSpec`.

```go
type ManagedClusterSetSpec struct {
   // Selector represents a selector of ManagedClusters.
   ClusterSelector ManagedClusterSelector  `json:"clusterSelector"`

   // Only works for the ClusterSelector.SelectorType is "LegacyClusterSetLabel"
   // +optional
   DeletePolicy DeletePolicyType
}

type DeletePolicyType string

type ManagedClusterSelector struct{
   SelectorType SelectorType `json:"selectorType"`
}

type SelectorType string

const (
   LegacyClusterSetLabel SelectorType = "LegacyClusterSetLabel"
)
 
const (
   DeleteClusterSetLabel       DeletePolicyType = "DeleteClusterSetLabel"
   HoldClusterSetDeletion      DeletePolicyType = "HoldClusterSetDeletion"
   DeleteClusters              DeletePolicyType = "DeleteClusters"
)
 
```
### Details
Notes: the `DeletePolicy` only works for the `ClusterSelector.SelectorType` is `LegacyClusterSetLabel`. It means the clusterset uses the label `cluster.open-cluster-management.io/clusterset:<Clusterset Name>` to select target clusters. So In the doc, we only cover the clusterset which `ClusterSelector.SelectorType` is `LegacyClusterSetLabel`
There are some options to delete a clusterset.

1. DeleteClusterSetLabel
In each managed cluster in a clusterset, it should have a label `cluster.open-cluster-management.io/clusterset:<Clusterset Name>`. When the clusterset is deleted. This label is useless. So we could delete the label in each managedClusters when the clusterset is deleted.

2. HoldClusterSetDeletion
In this option, when a clusterset is deleted, if there still have clusters in this set, we will hold the clusterset deletion.

3. DeleteClusters
In this option, when a clusterset is deleted, we will delete all clusters in this set.

4. ""
As this field is optional, so if the value is "", we will only delete the clusterset, and all other related resources will remain.
 
## Upgrade / Downgrade Strategy
The new api is compatible with the previous version. So there is no external work needed when upgrading
