# Multi-Cluster Workload Scheduling

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal will be adding a new mutli-cluster workload functionality to OCM 
platform as either a built-in module or a pluggable addon and a new multi-cluster 
workload API will be added under a new API group `workload.open-cluster-management.io`
as the manipulating interface for users. Note that the only requirement for
the adopted local workload (e.g. Deployment, ReplicaSet) in the spoke cluster will 
be implementing the generic [scale](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#scale-subresource)
subresource, so the new multi-cluster workload controller will be scaling up/down
the local workloads regardless of whether it's a built-in workload API or custom
workload developed via CRD.


## Motivation

### Goals

#### Controlling Replicas Distribution 

In some cases, we may want to specify a total number of replicas for a multi-cluster 
workload and let the controller do the rest of replicas distribution for us according 
to different strategies such as (1) even (max-min) distribution (2) weighted 
(proportional) distribution. The distribution should be updated reactively by 
watching the selected list of clusters via the output from `PlacementDecision` API. 
Note that the computed distribution here will be an "expected" number while the 
actual distribution may diverge from the expectation depending on the allocatable 
resource or the liveness of the replicas which is elaborated in the following section.

#### Dynamic Replicas Balancing

The term "balance" or "re-schedule" here infers the process of transferring a replicas
temporarily from one cluster to another. There are some cases when we need to trigger
the process of replicas balancing:

- When the distributed local workload fails to provision effective instances over a
  period of time.
- When the distributed local workload is manually scaled down on purpose.

The process of replicas transferring can be either "bursty" or "conservative":

- __Bursty__: Increasing the replicas for one cluster then decrease the other.
- __Conservative__: Decrease first then increase.

#### Adopting Arbitrary Workload Types

Given the fact that more and more third-party extended workload API are emerging beyond 
the kubernetes community, our multi-cluster workload controller should not raise any
additional requirement on the managing workload API except for enabling the standard
[scale](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#scale-subresource)
subresource via the CRD. Hence, to scale up or down the local workload, the controller
will be simply updating/patching the subresource regardless of its concrete types.

### Non Goals

- This KEP will not cover the distribution of special workloads such as `Job`.
- This KEP will not cover the distribution of satellite resources around the workload 
  such as `ConfigMap`, `ServiceAccount`.

## Proposal

### Abstraction

To understand the functionalities of the multi-cluster workload easier, we can start by
defining the boundary of the controller's abstraction as a black box.

#### "ClusterSet" and "Placement"

#### Controller "Input"

The source of information input for the multi-cluster workload controller will be:

- __Cluster Topology__: The `PlacementDesicision` is dynamically computed according to
  the following knowledge from OCM's built-in APIs:
  - The existence and availability of the managed clusters.
  - The "hub-defined" attributes attached to the cluster model via labelling. e.g. the 
    clusterset, the feature label or other custom labels which will be read by the 
    placement controller.
  - The "spoke-reported" attributes i.e. "ClusterClaim" which is collected and reported
    by the spoke agent.
    
- __API Prescription__: There will be a new API named "ElasticWorkload" or 
  "ManagedWorkload" that prescribes the necessary information for workload distribution
  such as the content of the target workload, the expected total number of replicas, etc.
  
- __Effective Distributed Local Workload__: The new controller also need to capture the
  events from local clusters so that it can take actions e.g. when the instance crushes
  or tainted unexpectedly.

#### Controller "Output"

The new controller will be applying the latest state of the workload towards the selected
clusters and tuning its replicas on demand. As a matter of implementation, the workload
applying will be executed via the stable `ManifestWork` api.

### API Spec

```yaml
apiVerion: scheduling.open-cluster-management.io/v1
kind: ElasticWorkload
spec:
  # The target namespace to deploy the workload in the spoke cluster.
  spokeNamespace: default
  # The content of target workload, supporting:
  # - Inline: Embedding a static manifest.
  # - Import: Referencing an existing workload resource. (Note that
  #           the replicas should always be set to 0 to avoid wasting
  #           capacities in the hub cluster.)
  target:
    type: [ Inline | Import ]
    inline: ...
    import: ...
  # Referencing an OCM's placement policy in the same namespace as where
  # this elastic workload resource lives.
  placementRef:
    name: ...
  # DistributionStrategy controls the expected replicas distribution
  # across the selected clusters from the placement api above. The supported 
  # distributing strategy will be:
  # - Even: Filling the min replicas upon every round. i.e. max-min
  # - Weighted: Setting a default weight and overriding the weight for a
  #             few clusters on demand.
  distributionStrategy:
    totalReplicas: 10
    type: [ Even | Weighted ]
  # BalanceStrategy prescribes the balancing/re-scheduling behavior of the 
  # controller when the effective distributed replicas doesn't meet the
  # expectation within a period of "hesitation" time. The supported types
  # will be:
  # - None: Do not reschedule at any time.
  # - LimitRange: The reschedule is allowed within a range of numbers. The
  #               replicas scheduler will be trying the best to control the 
  #               managed replicas within the range:
  #    * "min": when the controller is attempting to transfer a replicas, 
  #             those clusters under the "min" will be the primary choices.
  #    * "max": the controller will exclude the cluster exceeding the "max"
  #             from the list of candidates upon re-scheduling.
  # - Classful: A classful prioritized rescheduling policy.
  #    * "assured": similar to "min" above
  #    * "softLimit": those cluster (assured < # of replicas <= softLimit)
  #                   will be considered as secondary choice in the candicates.
  #                   Generally the "softLimit" can be considered as a 
  #                   recommended watermark of replicas upon re-scheduling.
  #    * "hardLimit": similar to "max" above.
  balanceStrategy:
    type: [ None | LimitRange | Classful ]
    limitRange:
      min: ...
      max: ...
    classful:
      assured: ...
      softLimit: ...
      hardLimit: ...
status:
  # list of distributed resources
  manifestWorks: ...
```

### Details

#### When "Distribution strategy" and "Balance strategy" conflicts

The "Distribution strategy" works prior to "Balance strategy", so the latter can
be considered as an overriding patch upon the former. The controller will always 
be honoring the balance strategy. The following list is a few possible examples 
when the two fields conflicts when combining "Weighted" distributor and 
"LimitRange" re-balancer:

###### Some expected replicas exceeds the max watermark
- Conditions:
  - Selected # of Clusters: 2 
  - Distribution: 1:2 weighted distribution under 6 total replicas.
  - Balance: LimitRange within 2-3
    
  Result: The initial expected distribution shall be 2:4 while the re-balancing will
          reset the distribution to 3:3 in the end.

###### All expected replicas exceeds the max watermark
- Conditions:
  - Selected # of Clusters: 2
  - Distribution: 1:2 weighted distribution under 6 total replicas.
  - Balance: LimitRange within 1-2

  Result: The initial expected distribution shall be 2:4 while the re-balancing will
  reset the distribution to 2:2 even if the sum can't reach the total replicas.

###### All expected replicas doesn't reach the min watermark
- Conditions:
  - Selected # of Clusters: 2
  - Distribution: 1:2 weighted distribution under 6 total replicas.
  - Balance: LimitRange within 5-10

  Result: The initial expected distribution shall be 2:4 while the re-balancing will
  reset the distribution to 5:5 regardless of distribution strategy.

#### Workload Manifest Status Collection

Overall in OCM, there are 3 feasible ways of collecting the status from the spoke 
clusters:

1. List-Watching: The kubefed fashion status collection. It violates with OCM's
                  pull-based architecture and will be likely the bottleneck of
                  scalability when the # of managed clusters grows.
2. Polling Get: Getting resources at a fixed interval which is costing less but losing
                promptness on the other hand.
3. Delegate to `ManifestWork`: See the new status collection functionalities WIP in 
   [#30](https://github.com/open-cluster-management-io/enhancements/pull/30)

This proposal will be supporting (2) and (3) in the end and leave the choice to users.


