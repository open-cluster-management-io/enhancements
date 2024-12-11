# Support multiple bootstrap kubeconfigs and switching between hubs - MultipleHubs

## Release Signoff Checklist

- [] Enhancement is `implemented`
- [] Design details are appropriately documented from clear requirements
- [] Test plan is defined
- [] Graduation criteria for dev preview, tech preview, GA
- [] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary
This proposal aims to provide a mechanism to support configuring multiple bootstrapkubeconfigs and that will enable agents to smoothly switch between hubs at scale with fewer manual operations.

## Motivation
Currently, an agent can only point to one hub, if we want to switch to another hub, we need replace the secret `bootstrap-hub-kubeconfig` in `open-cluster-management-agent` namespace manually.

To address this, we will support configuring multiple bootstrapkubeconfig secrets and provide a simple mechanism for agents to determine which hub they should connect to. And with multiple bootstrap kubeconfigs configured, agents are possible to complete the switching process without manual operations.

This mechanism will be beneficial for various scenarios, including upgrading, disaster recovery, improving availability, etc.

## Goals
- Trigger the agent to reselect when the current hub it connected to is not available.
- Trigger the agent to reselect when user explicitly deny the connection to the current hub.

## Non-Goals
- To ensure bootstrapkubeconfig candidates are valid **during the candidate selection phase** is out of the scope.
- To determine which bootstrap kubeconfig is suitable for a sepcific hub at any given time. This depends on the user's specific scenario.
- To synchronize customer workloads and configurations between hubs, ensure that the resources of the hubs are synchronized.
    - If the resources are not synchronized, the old customer workload may be wiped out after the switch.

## Use cases

### Story1 - Backup and restore
As a user, I want to switch agents to the new(restored) hub **without manual operations**.

### Story2 - Rolling upgrade
As a user, I want to switch half of the agents to the upgraded hub to test the functionality of the upgraded components.

### Story3 - High availability architecture
As a user, I want to configure 2 hubs in active-active mode to improve management availability. In this setup, if one hub goes down, the other hub will continue to provide management capabilities.

## Risk and Mitigation
N/A

## Design Details

The `MultipleHubs` feature should be controlled by a feature gate of Klusterlet:

```
apiVersion: operator.open-cluster-management.io/v1
kind: Klusterlet
...
spec:
  ...
  registrationConfiguration:
    ...
    featureGates:
      - feature: MultipleHubs
        mode: Enable
```

The multiple bootstrapkubeconfigs can be configured in the Klusterlet by the `bootstrapKubeConfigs` field:

```
apiVersion: operator.open-cluster-management.io/v1
kind: Klusterlet
...
spec:
  ...
  registrationConfiguration:
    ...
    featureGates:
      - feature: MultipleHubs
        mode: Enable
    bootstrapKubeConfigs:
      type: "LocalSecrets"
      localSecretsConfig:
        kubeConfigSecrets:
            - name: "hub1-bootstrap"
            - name: "hub2-bootstrap"
        hubConnectionTimeoutSeconds: 600
```

At the initial stage, we only support the `LocalSecrets` type, which means the bootstrapkubeconfig secrets are stored in the managed cluster.

### Workflow

Let’s seen an example, assume we have 2 hubs: `hub1` and `hub2`, and a managed cluster named `cluster1`. We want `cluster1` register to `hub1` first, and then switch to `hub2`.

First, we create 2 secrets on the agent `open-cluster-management-agent` namespace on the managed cluster `cluster1`:

```yaml
# The kubeconfig points to hub1
kind: Secret
apiVersion: v1
metadata:
  name: hub1-bootstrap
  namespace: open-cluster-management-agent
data:
  kubeconfig: YXBpVmVyc2lvbjogdjEK...
```

```yaml
# The kubeconfig points to hub2
kind: Secret
apiVersion: v1
metadata:
  name: hub2-bootstrap
  namespace: open-cluster-management-agent
data:
  kubeconfig: YXBpVmVyc2lvbjogdjEK...
```

And the `bootstrapKubeConfigs` field in the Klusterlet is configured as follows:

```yaml
bootstrapKubeConfigs:
  type: "LocalSecrets"
  localSecretsConfig:
    kubeConfigSecrets:
        - name: "hub1-bootstrap"
        - name: "hub2-bootstrap"
    hubConnectionTimeoutSeconds: 600
```

The agent will choose bootstrap kubeconfigs in order from the `kubeConfigSecrets` list. It will first try the kubeconfig from the first secret in the list ("hub1-bootstrap"), and if that fails it will try the second one ("hub2-bootstrap"), and so on. Once a working kubeconfig is found, the agent will use it to start bootstrapping.

In our case, the agent will first register on hub1 successfully. And to trigger it switches to `hub2`, we simply set the `hubAcceptsClient` to `false` on the managed cluster CR on hub1:

```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
...
spec:
  ...
  hubAcceptsClient: false
```

If the hubAcceptsClient is set to false, the managed cluster currently connected to the hub will immediately disconnect from the hub and try to connect to another hub cluster.

On the other hand, if the agent lost connection to the hub over the `hubConnectionTimeoutSeconds`, it will try to connect to another hub cluster as well.
