
## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

There were some discussions and feedback about how an app developer or end user can leverage ManifestWork and Placement API to deploy application workload easily. ManifestWork and Placement API are independent concepts. It needs a higher level controller to link them together for multicluster application workload scheduling and deployment.

Some existing examples that use placement/manifestwork for multicluster workload orchestration including:
* Multicloud-operator-subscription
* KubeVela

But we may need a built-in API in OCM that can do workload orchestration. It differs from Multicloud-operator-subscription and Kubevela, since it should not intend to be a gitops tool by integrating with various sources, and it should concentrate on kubernetes native workload (deployment/service etc). 

In addition, we should also consider more advanced workload scheduling beyond placement API. Some prior work including Kim Min’s proposal on https://github.com/open-cluster-management-io/enhancements/pull/31, and discussion in https://github.com/open-cluster-management-io/OCM/issues/27 

## Motivation

ManifestWork and Placement API are independent concepts. It makes sense to bring them together to achieve fleet wide orchestration.

### Goals

* Ease of use:
  * A user should be able to use this API to easily deploy a multicluster application without understanding many concepts in OCM. 
  * A user should have a CRD that is similar enought to leverage Spec from their existing manifests

### Non-Goals

* Initially out of scope:
  * Replica distribution
    * A user can choose how to distribute a resource. For example, the user can define a deployment and declare to replicate the deployment to every cluster, or to distribute the replicas of the deployment to each cluster.
    * A user can choose which cluster to deploy the workload.

  * Fallback and reschedule
    * When a workload cannot be fully run on a spoke (lack of resources, image pull backoff). It should be able to reschedule to another cluster.
    * When a cluster was added/removed from a PlacementDecision, it should be able to reschedule.
      * Including the changes upon the Placement’s matching policy.

## Proposal

### User Stories

#### Story 1
User wants to distribute work (kubernetes resources) to a number of clusters in their fleet, filtering this group of clusters with placement.

#### Story 2
User removes a distributed work (distributed to clusters in the placement decision), the resources should be cleaned up similar to exisitng work behaviours

#### Story 3
Work placement should contain enough status information to determine if the manifest(s) were successfully applied to the clusters in the placement decision.

### Implementation Details/Notes/Constraints

* The user needs to create the resource in the workspace ns on the hub cluster. 
* The controller will pick the cluster based on the placement decision.
* The controller will read the resources and generate ManifestWork on the related cluster.
* `[NON-GOLA]` If a user specifies a placement ref, the controller will use this placement to find clusters. Otherwise the controller will create a new empty placement that selects all clusters scoped in this ns. The user can mutate the placement spec afterwards for advanced cluster selection.
* `[NON-GOLA]` By default, the controller will replicate the manifest to each manifestwork, if the mode is split. The controller will split the replica field and generate manifests with different replicas in different clusters (similar as splitter in kcp).
* `[NON-GOLA]` The controller will check the status of the applied resources in the manifestwork, if the resource is degraded (e.g. deployment available replica is not enough). The controller will exclude this cluster and try to select another one.

#### API
```yaml
apiversion: work.open-cluster-management.io/v1alpha1
kind: PlacementWork # ApplicationWork?
metadata:
  name: app
  namespace: default
spec:
  manifestWorkSpec:
    workloads:
      manifest:
      - apiVersion: v1
        kind: Service
        metadata:
          name: test01
          namespace: default
        spec: … 
      - apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: test02
          namespace: default
        spec: … 
  ## The following is a spike ##
  distributionStrategy:
  - name: test01
    namespace: default
    type: Replicate   # replicate means the resource is replicated to each cluster
  - name: test02
    namespace: default
    # splits spec.Replicas by default.
    # if spec.Replicas = 3, and it is to deploy to cluster1 and cluster2
    # manifests on cluster1 will have spec.Replicas=2, that on cluster 2 will be spec.Replicas=1
    # the user should also be able to define another field to split.
    type: Split
    split: 
      type: Balance # this is to reuse the idea in this proposal
      limitRange:
        min: 1
        max: 2

      distributionStrategy:
        type: Replicate   # replicate means the resource is replicated to each cluster

  placementRef:
    # user can choose to specify an existing placement, or if the field is unspecified, 
    # a new empty placement will be generated, user can change the spec of the
    # generated placement. The controller will not revert users' change on placement.
    name: placement
status:
  conditions:
  - type: PlacementVerified
  - type: ManifestworkApplied
  ## The following is a spike ##
  # this to describe the number of clusters that the resources are applied
  clustersApplied:
    number: 4
    lastTransitionTime: xxx
  # this to describe the number of clusters that the resources are available
  clustersAvailable:
    number: 4
    lastTransitionTime: xxx
  # this to describe the number of clusters that the resources are degraded
  clustersDegraded: 2
    number: 2
    lastTransitionTime: xxx
```

### Risks and Mitigation

* Security is based on ManagedClusterSets and the bindings to the namespace where the work placement resource is created.

## Design Details

### Open Questions

None at the moment.

### Test Plan

**Note:** *Section not required until targeted at a release.*

* Functional tests are provided

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:

- [Maturity levels][maturity-levels]
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),
or by redefining what graduation means.

In general, we try to use the same stages (alpha, beta, stable), regardless how the functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

### Upgrade / Downgrade Strategy

N/A


## Implementation History

- [x] Enhancement submitted

## Drawbacks

* If the administrator is not aware that CRUD for this new resource is granted to a namespace that is bound to a ManagedClusterSet, 
user may distribute kubernetes manifests to those available clusters.

## Alternatives

* Higher level constructs can be leverages (listed above):
  * Multicloud-operator-subscription
  * KubeVela 