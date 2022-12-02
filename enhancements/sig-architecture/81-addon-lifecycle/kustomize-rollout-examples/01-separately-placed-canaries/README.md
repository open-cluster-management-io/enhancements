`kubectl kustomize enhancements/sig-architecture/81-addon-lifecycle/kustomize-rollout-examples/01-separately-placed-canaries`

In this directory I was playing with the idea of having a reference from the ClusterManagementAddOn
to a new hub-based, cluster-scoped resource called HubScopedAddOnConfiguration that holds a
configuration to be provided to the managed cluster namespace "somehow" (hand-wave).

Kustomize is used to produce the HubScopedAddOnConfiguration manifest with a SHA based suffix
and to keep the name up to date in the ClusterManagementAddOn.
This functions, but doesn't provide any obvious value since I'm still selecting the
canaries manually and a git-ops based revert would be just as easy.

This experiment made me wonder about two things
1. Directly including the configuration options in the ClusterManagementAddOn alongside the
   version directly.
   If we provided similar options for referencing a CR instance, the power is about the same,
   because you still have to update canary-config and global-config separately.
3. What if we provided rollout options like the Deployment resource and our configuration was
   treated like a ConfigMap, leading to 02-rollout-strategy.