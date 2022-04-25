# Enhancement of cluster deletion 

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary

Currently, when we remove a cluster from the Hub, the `managedClusterAddons` and `manifestWorks` in the cluster namespace need to be deleted manually, and cannot ensure that the resources deployed on the Spoke cluster can be cleaned up. This proposal is to define clearly what happened when a cluster is deleted.

## Motivation

Today, when we delete a `managedCluster`, only the `rolebinding` for the registration/work agent is cleaned up on the hub, and the resources including the cluster namespace,`managedClusterAddons` and `manifestWorks` are left over. It will bring some issues like: 
1. We cannot delete `manifestWork` directly because the finalizer cannot be removed by the work-agent.
2. The resources deployed by `manifestWork` on the spoke cluster cannot be cleaned up even if its `manifestWork` is deleted.
3. The `managedClusterAddon` with pre-delete job cannot be deleted because the `manifestWorks` cannot be deleted too.

There is a typical use case that the lifecycle of the `klusterlet` on the spoke cluster is taken over by the `manifestWork` too. There is a `manifestWork` to deploy klusterlet manifests itself after the cluster is registered. And the klusterlet manifests of the managed cluster will be cleaned up if the klusterlet manifestWork is deleted on the hub.
The cleanup starts when the `managedCluster` is deleting.

1. Delete the `managedClusterAddons` and `manifestWorks` firstly except the `klusterlet manifestWork`.
2. Delete the `klusterlet manifestWork` after the other `managedClusterAddons` and `manifestWorks` are deleted.
3. Start to delete the cluster namespace when there is no `managedClusterAddons` and `manifestWorks`. The cluster ns will be deleted after the resources related with Hive like job, clusterDeployment, etc are deleted. Because the Hive will create some jobs in this ns when the provisioned cluster is destroyed.

The deletion process is a typical use case and example for the community. This proposal is to enhance and standardize the cluster deletion process in OCM.

### Goals

- Enhance and standardize the cluster deletion process and clean up the resources including `managedClusterAddons`, `manifestWorks` created by addon-framework    and namespace in order.

### Non-Goals

- Clean up the `manifestWorks` which are not created by addon-framework automatically.

   These customized `manifestWorks` should be cleaned up by other controllers, because there are cases like these `manifestWorks` could be evicted and rescheduled to another `managedCluster` when the `managedCluster` is deleting, or they have pre-delete jobs to do.

## Proposal

### Design Details 

Add a new feature gate named `ClusterLifeCycle` for this cluster deletion feature, which is disabled by default.
#### registration-controller 

1. Add a condition type `ContentDeleteSuccess` to the `managedCluster`, which indicates whether the cluster resources are remaining during the `managedCluster` deletion process.

2. Define a label named `cluster.open-cluster-management.io/keep-upon-cluster-deletion`. The cluster namespace with this label will not be deleted by the registration during the `managedCluster` deletion process. 
   
3. Add options for the registration-controller binary. The registration-controller will handle the deletion process.
   ```
      /registrion
      -- controller
      -- resources-to-monitor-for-deletion: resource.version.group,resource.version.group.com...
   ```

   * `resources-to-monitor-for-deletion` specifies a list of resources GVR. registration-controller will not delete anything until all resources defined here are deleted by other controller. The common format of string like either `resource.version.group` or `resource.version.group.com`. 

4. Add a managedCluster deletion controller in registration-controller to handle the resource cleanup. The deletion process like this:
   
   1. The cluster admin deletes the `managedCluster`. The `managedCluster` is under deleting since there is a cleanup finalizer on it.
   2. `deletion-controller` begins to monitor predeleted resources defined in `resources-to-monitor-for-deletion` at first, do not delete anything until all resources defined are cleaned up by other controller.
   3. `deletion-controller` begins to delete all `managedClusterAddons`.
   4. `deletion-controller` begins to delete `roleBindings` of registration-agent and work-agent after all `managedClusterAddons` and `manifestWorks` are cleaned up.
   5. `deletion-controller` begins to delete the cluster namespace without the label `cluster.open-cluster-management.io/keep-upon-cluster-deletion`.
   6. `deletion-controller` remove the finalizer `cluster.open-cluster-management.io/api-resource-cleanup` on the `managedCluster`.

#### manifestworkvalidators-webhook 

1. manifestworkvalidators webhook will deny the creation of all `managedClusterAddons` when the `managedCluster` is deleting.

2. manifestworkvalidators webhook will deny the creation of all `managedClusterAddons` and `manifestWorks` when there is no `managedCluster` with the same name as the namespace.

### Upgrade strategy 
   
1. We will put this new feature into a feature gates named `ClusterLifeCycle` which is disabled by default.

2. The user should add the label `cluster.open-cluster-management.io/keep-upon-cluster-deletion` to the cluster namespace which do not want to be deleted by registration.

### User Stories

#### Story 1

As a cluster admin, the cluster namespace should be deleted after I delete the `managedCluster`.

   Deletion process:
   1. The cluster admin deletes the `managedCluster` on the hub.
   2. `deletion-controller` deletes all `managedClusterAddons`.
   3. The cluster admin or other controller deletes all customized `manifestWorks`.
   4. `deletion-controller` deletes `roleBindings` of registration-agent and work-agent after there is no `managedClusterAddons` and `manifestWorks`.
   5. `deletion-controller` deletes the cluster namespace.
   6. The cluster admin deletes the `klusterlet` on the spoke cluster.

#### Story 2

As a cluster admin, the cluster namespace should not be deleted after I delete the `managedCluster`, because I want to remain other resources in this namespace.

  Prerequisites:
   1. Add the label `cluster.open-cluster-management.io/keep-upon-cluster-deletion` to the cluster namespace.
   
  Deletion process:
   1. The cluster admin deletes the `managedCluster` on the hub.
   2. `deletion-controller` deletes all `managedClusterAddons`.
   3. The cluster admin or other controller deletes all customized `manifestWorks`.
   4. `deletion-controller` deletes `roleBindings` of registration-agent and work-agent after there is no `managedClusterAddons` and `manifestWorks`.
   5. The cluster admin deletes the `klusterlet` on the spoke cluster.

#### Story 3

As a cluster admin, the `managedClusterAddons` and cluster namespace should be deleted after I delete all applications in the cluster namespace during the cluster deletion.
  Prerequisites:
   1. Add the application GVR to `resources-to-monitor-for-deletion` of registration-controller.
   
  Deletion process:
   1. The cluster admin deletes the `managedCluster` on the hub.
   2. The cluster admin or other controller deletes applications in the cluster namespace.
   3. `deletion-controller` deletes all `managedClusterAddons` after all applications are deleted.
   4. The cluster admin or other controller deletes all customized `manifestWorks`.
   5. `deletion-controller` deletes `roleBindings` of registration-agent and work-agent after there is no `managedClusterAddons` and `manifestWorks`.
   6. `deletion-controller` deletes the cluster namespace.
   7. The cluster admin deletes the `klusterlet` on the spoke cluster.