# Standalone Hub Templates

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
      [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

To support Open Cluster Management (OCM) hub templates without the policy framework, this feature provides an additional
`governance-standalone-hub-templating` addon which generates a user per cluster on the hub to be leveraged by the
Configuration Policy controller for hub template resolution. The OCM hub administrator will be responsible for granting
`list` and `watch` permissions to the resources needed by the hub templates. A sample `ConfigurationPolicy` can be a way
to automate this for a cluster set.

## Motivation

When an Open Cluster Management (OCM) policy user deploys their policies with external tools such as Argo CD, the hub
templating functionality is lost. This is often a critical feature for large scale deployments and prevents users from
choosing an external tool if they require an alternative to the OCM policy framework.

### Goals

1. Support hub templates for `ConfigurationPolicy` and `OperatorPolicy` when deployed through an external tool such as
   Argo CD.
1. Hub templates should resolve when referenced objects are updated on the hub.
1. Policies can still be evaluated if the managed cluster is disconnected from the hub and the policy definition remains
   the same.
1. Permissions to the hub are per managed cluster.
1. Provide a hub only `ConfigurationPolicy` to provide permissions to cluster sets.
1. It must work without the rest of the policy framework on the managed cluster.

### Non-Goals

1. `CertificatePolicy` and Gatekeeper constraints will not be directly supported. It can be wrapped in a
   `ConfigurationPolicy` if hub templates are needed for those.
1. Updating a `ConfigurationPolicy` definition with hub templates while disconnected from the hub is not explicitly
   supported. This will almost always work based on the proposed design as long as there aren't new hub template API
   queries, but if the update adds a hub template API query that isn't in the Configuration Policy controller cache, it
   will not work.

## Proposal

### Design

#### New governance-standalone-hub-templating Addon

Currently, hub templates are resolved by the Governance Policy Propagator as part of the replicated `Policy` creation.
In standalone mode, the Governance Policy Propagator and the Governance Policy Framework are not part of the equation.
Because of this, the simplest and most reliable approach is to add hub template support in the Configuration Policy
controller.

A way to achieve this is to create an additional addon that is **disabled** by default called
`governance-standalone-hub-templating`. Enabling this addon will cause the following:

- The addon framework creates a user/subject represented by a certificate on the hub for each managed cluster. The
  initial permissions will be `list` and `watch` on the managed cluster's `ManagedCluster` object on the hub. This will
  allow the hub template variable of `.ManagedClusterLabels` to be populated.
- The addon framework will create a `Secret` in the addon namespace on the managed cluster with the hub user kubeconfig
  that will be leveraged by the Configuration Policy controller.
- The Governance Policy Addon controller should configure the `config-policy-controller` to mount the generated
  kubeconfig when the `governance-standalone-hub-templating` addon is enabled and pass a flag to enable the standalone
  hub templating mode.

#### Configuration Policy changes

When the Configuration Policy controller encounters a hub template when standalone hub templating mode is disabled, it
should mark the policy as `NonCompliant` with a message such as
`The governance-standalone-hub-templating addon must be enabled to resolve hub templates from the cluster`. The
`ConfigurationPolicy` objects created on the managed cluster through the policy framework will never have hub templates
since the policy framework marks these policies as `NonCompliant` assuming the hub templates failed to resolve on the
hub. With this in mind, the user's intentions are safe to assume when a hub template is encountered.

When the Configuration Policy controller has standalone hub templating mode enabled, it should instantiate a separate
`TemplateResolver` with the `governance-standalone-hub-templating` hub user and resolve hub templates (i.e.
`{{hub ... hub}}`) prior to managed cluster templates (i.e. `{{ ... }}`). By default, this `TemplateResolver` will have
no permission on the hub cluster. The hub administrator must grant `list` and `watch` permissions on the objects
accessed by hub templates. The error messages for lack of permissions are clear due to the work done in the "Expand Hub
Template Access on Policies" feature.

If the changes stopped here, the case where the managed cluster is always connected to the hub would work well. Short
disconnects would be okay since objects referenced by the hub templates are cached. Once the Configuration Policy
controller restarts and thus the cache is cleared, the policy could no longer be evaluated due to hub template
resolution failing.

The proposed solution is for the Configuration Policy controller to create a `Secret` named `<policy UID>-last-resolved`
in the managed cluster namespace with owner references to the policy. This `Secret` would contain a stripped down
`ConfigurationPolicy`/`OperatorPolicy` after hub templates are resolved. This must include at least the `generation` and
the `spec`. In the event the Configuration Policy controller restarts and fails to resolve hub templates due to a
"connection refused" error, the Configuration Policy controller wil fallback to the value in the
`<policy UID>-last-resolved` `Secret` if the `generation` matches the current `ConfigurationPolicy`/`OperatorPolicy`.
The `generation` detects if the `ConfigurationPolicy`/`OperatorPolicy` was updated since it was last resolved, so in
other words, modifying a `ConfigurationPolicy`/`OperatorPolicy` with hub templates while disconnected from the hub will
not be supported. It may work depending on what's in the `TemplateResolver` cache but this feature will not guarantee
any resilience to this as noted in the "Non-Goals" section. The reason it's stored in a `Secret` is to leverage etcd
encryption in the event a hub template references a `Secret`.

### User Stories

#### Story 1

As an Argo CD user, I'd like to leverage `ConfigurationPolicy` to securely copy secrets from the Open Cluster Management
hub to the managed cluster.

#### Story 2

As a `ConfigurationPolicy` user, I'd like to leverage a centralized `ConfigMap` on the hub for dynamic configuration
parameters for each managed cluster.

#### Story 3

As a hub template user without the policy framework, I'd like to have an easy way to grant permissions to groups of
clusters for hub templates. This will be a sample ConfigurationPolicy to grant a `Role` to a cluster set.

#### Story 4

As a policy framework user, I'd like to disable standalone hub templates support since I want to decreate the attack
surface of the Open Cluster Management hub.

### Implementation Details/Notes/Constraints

#### Event Driven Aspect

Lets continue using the [go-template-utils](https://github.com/stolostron/go-template-utils) library which uses the
[kubernetes-dependency-watches](https://github.com/stolostron/kubernetes-dependency-watches) library under the hood for
resolving this style of hub templates. The difference is that the API watches are for a separate cluster than the
cluster the Configuration Policy controller is running on. In the case that the managed cluster is disconnected from the
hub, the [kubernetes-dependency-watches](https://github.com/stolostron/kubernetes-dependency-watches) library's use of
`RetryWatcher` causes it to continuously retry without a backoff. Not only does this spam the logs, it can keep the CPU
unnecessarily busy.

Proposed solution:

1. Modify
   [watchLatest](https://github.com/stolostron/kubernetes-dependency-watches/blob/85c50006641c8c5a951bdd35bea3d48708c7e9a5/client/client.go#L725-L791)
   to have an exponential backoff.
1. Lower the log level for connection refused errors in client-go in the
   [RetryWatcher doReceive](https://github.com/kubernetes/kubernetes/blob/5864a4677267e6adeae276ad85882a8714d69d9d/staging/src/k8s.io/client-go/tools/watch/retrywatcher.go#L123-L134)
   method.

#### Enabling the Addon For All Clusters

We'll need to document that a `ClusterManagementAddon` can be tied to a global `Placement` leveraging the addon
`installStrategy`.

### Risks and Mitigation

- Increased hub attack surface due to an additional hub user. The addon is disabled by default and the user must
  explicitly grant permissions to the hub user.

### Test Plan

**Note:** _Section not required until targeted at a release._

### Graduation Criteria

N/A

### Upgrade / Downgrade Strategy

A downgrade would cause the `ConfigurationPolicy` to enforce unresolved hub templates as the actual values. The user
must remove these policies before downgrading.

### Version Skew Strategy

## Implementation History

N/A

## Drawbacks

- Requires a lot of effort by the user to configure the permissions in a large environment. A provided
  `ConfigurationPolicy` example should help.

## Alternatives

- We could consider reusing the Configuration Policy hub user rather than a separate addon as it only has access to
  manage leases on the hub. This would greatly reduce the complexity of the setup, but it would violate the least
  privilege principle and the risk could be greater if the Configuration Policy controller requires additional
  permissions on the hub in the future.

## Infrastructure Needed [optional]

N/A
