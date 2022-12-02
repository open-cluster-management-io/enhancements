`kubectl kustomize enhancements/sig-architecture/81-addon-lifecycle/kustomize-rollout-examples/02-rollout-strategy/`

In this directory I was playing with the idea of having a reference from the ClusterManagementAddOn
to a new hub-based, cluster-scoped resource called HubScopedAddOnConfiguration that holds a
configuration to be provided to the managed cluster namespace "somehow" (hand-wave).

It also contains a rolloutStrategy that allows the specification of a canary placement that will
take all new changes first.
By adding a rollout strategy, we're able to have a single HubScopedAddOnConfiguration field that
is updated to a new name, the name way that a Deployment's ConfigMap reference in kustomize
based environment updates to a new ConfigMap when the content needs to change.

The ManagedClusterAddOn needs to be able to acknowledge, "I'm done updating my configuration",
but we're very close to that now and it's generally useful to know.