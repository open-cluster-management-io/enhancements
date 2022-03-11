# Governance Policy Addon Controller

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
      [website](https://github.com/open-cluster-management/website/)

## Summary

The goal of this feature is to automate and simplify deployment of the governance policy components
to managed clusters by creating a controller that leverages the
[addon framework](https://github.com/open-cluster-management-io/addon-framework/) in order to deploy
components using the ManagedClusterAddOn kind.

## Motivation

The governance policy framework is currently deployed using individual manifest files deployed
directly by the user. While this is functional, it does not lend itself to automation,
customization, scalability, nor hub-centered management that Open Cluster Management demands.

### Goals

- Develop a controller that deploys governance policy components to managed clusters using the
  ManagedClusterAddOn kind.

### Non-Goals

- None at this time.

## Proposal

### User Stories

1. As a k8s admin, I want to deploy, uninstall, and customize the governance policy framework on my
   managed clusters.

### Architecture

The governance policy addon controller manages installations of policy addons on managed clusters by
using
[ManifestWorks](https://github.com/open-cluster-management-io/api/blob/main/docs/manifestwork.md).
The addons can be enabled, disabled, and configured via their `ManagedClusterAddOn` resources.

(For more information on the addon framework, see the
[addon-framework enhancement/design](https://github.com/open-cluster-management-io/enhancements/tree/main/enhancements/sig-architecture/8-addon-framework).)

The addons managed by this controller are:

- The "config-policy-controller" consisting of the
  [Configuration Policy Controller](https://github.com/open-cluster-management-io/config-policy-controller).
- The "governance-policy-framework" consisting of the
  [Policy Spec Sync](https://github.com/open-cluster-management-io/governance-policy-spec-sync), the
  [Policy Status Sync](https://github.com/open-cluster-management-io/governance-policy-status-sync),
  and the
  [Policy Template Sync](https://github.com/open-cluster-management-io/governance-policy-template-sync).

For example, this manifest deploys the Configuration Policy Controller to a managed cluster called
`my-managed-cluster` to the `open-cluster-management-agent-addon` namespace:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: config-policy-controller
  namespace: my-managed-cluster
spec:
  installNamespace: open-cluster-management-agent-addon
```

To modify the image used by the Configuration Policy Controller on this managed cluster, you can add
an annotation like:

```shell
kubectl -n my-managed-cluster annotate ManagedClusterAddOn config-policy-controller addon.open-cluster-management.io/values='
  { "global": {
      "imageOverrides": {
        "config_policy_controller":"quay.io/my-repo/my-configpolicy:imagetag"
      }}}'
```

Helm charts are contained within the addon controller repo, and any values in the Helm chart's
`values.yaml` can be modified using such annotations.

Of particular relevance is modifying this value for the framework addon:

- `addon.open-cluster-management.io/values={"onMulticlusterHub":true}`
  - This will modify the deployment to accommodate for connections and synchronization unique to a
    hub that manages itself via Open Cluster Management.

Additionally, there will be specific simplified annotations for the addons, including:

- `policy-addon-pause=true`
  - Pause the addon controller from making changes to and reconciling with the deployment on the
    managed cluster.
- `log-level=4`
  - Set the logging level of the addon to adjust verbosity of the logs.

### Test Plan

- Unit tests are not realistic because this is an implementation of the addon-framework
- E2E/integration tests to ensure that the hub is able to manage itself and that all components can
  successfully be deployed via the governance addon controller.

### Graduation Criteria

#### Alpha

1. The proposal is reviewed and accepted
2. Implementation is completed to support the primary goals
3. Test cases developed to demonstrate that the user stories are fulfilled

#### Beta

1. E2E/Integration tests complete
