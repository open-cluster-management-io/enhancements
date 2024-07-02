# Support multiple bootstrap kubeconfigs and switching between hubs

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
- To select the matched `leader` bootstrap kubeconfig from candidates. A required `leader` can be claimed from the hub side.
- To re-select a bootstrap kubeconfig when lost connection to the hub or required `leader` is changed.

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

### Terms

#### Bootstrapkubeconfig candidates
A set of secret contains the bootstrapkubeconfig of each hub. The bootstrapkubeconfig secrets that are labeled with `cluster.open-cluster-management.io/bootstrap-kubeconfig-candidate`.

#### The Chosen Bootstrapkubeconfig Candidate
We add a new annotation `cluster.open-cluster-management.io/bootstrap-kubeconfig-choose` on the `bootstrap-hub-kubeconifg` secret.
The value of this annotation represents the secret name of the candidate that the agent currently using to do bootstrapping.

#### The Required Bootstrapkubeconfig Candidate

We add a new annotation `cluster.open-cluster-management.io/bootstrap-kubeconfig-require` on the managed cluster CR.
The value of this annotation represents the secret name of the candidate that required by hub.

### Workflow

Let’s seen an example, assume we have 2 hubs: `hub1` and `hub2`, and a managed cluster named `cluster1`. We want `cluster1` register to `hub1` first, and then switch to `hub2`.

First, we create 3 secrets on the agent `open-cluster-management-agent` namespace on the managed cluster `cluster1`:

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: bootstrap-hub-kubeconfig
  namespace: open-cluster-management-agent
  annotations:
    cluster.open-cluster-management.io/bootstrap-kubeconfig-choose: "bootstrap-hub-kubeconfig-hub1" # The chosen candidate is `bootstrap-hub-kubeconfig-hub1`
```

```yaml
# The kubeconfig points to hub1
kind: Secret
apiVersion: v1
metadata:
  name: bootstrap-hub-kubeconfig-hub1
  namespace: open-cluster-management-agent
  labels:
    cluster.open-cluster-management.io/bootstrap-kubeconfig-candidate: ""
data:
  kubeconfig: YXBpVmVyc2lvbjogdjEK...
```

```yaml
# The kubeconfig points to hub2
kind: Secret
apiVersion: v1
metadata:
  name: bootstrap-hub-kubeconfig-hub2
  namespace: open-cluster-management-agent
  labels:
    cluster.open-cluster-management.io/bootstrap-kubeconfig-candidate: ""
data:
  kubeconfig: YXBpVmVyc2lvbjogdjEK...
```

Now we have:
1. A set of bootstrapkubeconfig candidates: `[bootstrap-hub-kubeconfig-hub1, bootstrap-hub-kubeconfig-hub2]`
2. A chosen bootstrapkubeconfig: `bootstrap-hub-kubeconfig-hub1`.

Then, we create a managedcluster CR with a specific annotation `cluster.open-cluster-management.io/bootstrap-kubeconfig-require` on hub1, the value is the name of the bootstrapkubeconfig candidate of `hub1`:

```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: cluster1
  annotations:
    cluster.open-cluster-management.io/bootstrap-kubeconfig-require: "bootstrap-hub-kubeconfig-hub1"
```

Agent will use the bootstrap kubeconfig in `bootstrap-hub-kubeconfig` to start bootstrapping.

As the registration process going on, the agent will first use kubeconfig in `bootstrap-hub-kubeconfig` and then the kubeconfig in `hub-kubeconfig`(generated by the registration process) to watch the values of `bootstrap-kubeconfig-choose`(from bootstrap-hub-kubeconfig secret's annnotations) and `bootstrap-kubeconfig-require`(from ManagedCluster's annotations).

If two of them doesn't equal, the agent update the secret `bootstrap-hub-kubeconfig` by:
1. Override the `bootstrap-kubeconfig-choose` with the value of `bootstrap-kubeconfig-require`.

There will also be another controller contiously sync the content of `chosen` candidate to `bootstrap-hub-kubeconfig`.

In our case, the agent will first register on hub1 successfully. And to trigger it switches to `hub2`, we simply update the annotation `cluster.open-cluster-management.io/bootstrap-kubeconfig-require` on the managed cluster CR to `bootstrap-hub-kubeconfig-hub2`:

```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: cluster1
  annotations:
    cluster.open-cluster-management.io/bootstrap-kubeconfig-leader: "bootstrap-hub-kubeconfig-hub2"
```

On the other hand, if the agent lost connection to the hub, it means `bootstrap-kubeconfig-choose` is not accessable at that given time, so it will randomly pick up a one from the other candidates to replace the `bootstrap-kubeconfig-choose`.
