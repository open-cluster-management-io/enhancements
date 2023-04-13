# New API: AddOnPlacementScoreGenerator

## Release Signoff Checklist

- [ ] Enhancement is `implemented`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal will be adding a new OCM API named `AddOnPlacementScoreGenerator`
which helps the administrators to provide custom values to Addon controllers
that generate `AddOnPlacementScores`.

A valid `AddOnPlacementScoreGenerator` resource should be in a "cluster namespace" and
the associated config resources will be delivered to the associated managed cluster
with that "cluster namespace".

## Motivation

### Influence Addon controllers behavior

Currently, when writing an Addon in order to extend the scheduling capabilities of OCM
there is no way to influence behavior of that addon via an API. The controller will run
and update the status for the `AddOnPlacementScores` object.

One of the use-cases we want to cover is testing latency from managed clusters to a set
of user-define locations. With the current `AddOnPlacementScores` implementation we would
need to hard code these locations or maybe use something like a `ConfigMap` that gets consumed
by the controller. We don't find these solutions flexible enough, so our proposal would be having
a new API to influence the behavior of such controllers.

Let's say I want to test latencies to redhat.com and google.com and place my application based on
the managed cluster with the lowest latency to redhat.com.

Providing I created the required controller with the hardcoded locations (redhat.com and google.com)
an `AddOnPlacementScores` similar to this one would be created on each managed cluster running this addon:

~~~yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: AddOnPlacementScore
metadata:
  name: cluster1-generator
  namespace: cluster1
status:
  conditions:
  - lastTransitionTime: "2021-10-28T08:31:39Z"
    message: AddOnPlacementScore updated successfully
    reason: AddOnPlacementScoreUpdated
    status: "True"
    type: AddOnPlacementScoreUpdated
  validUntil: "2021-10-29T18:31:39Z"
  scores:
  - name: "redhat-com-avgLatency"
    value: 30
  - name: "google-com-avgLatency"
    value: 50
~~~

Now, a `Placement` like this could be used:

~~~yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: latency-placement
  namespace: ns1
spec:
  numberOfClusters: 3
  prioritizerPolicy:
    mode: Exact
    configurations:
      - scoreCoordinate:
          type: AddOn
          addOn:
            resourceName: cluster1-generator
            scoreName: redhat-com-avgLatency
        weight: -1
~~~

Other applications may use latency to google.com.

Now, we want to add linux.com to the list of locations to test. With the current implementation we 
will need to edit the code of the addon controller and include the test to linux.com + the result to be added
to the `AddOnPlacementScore`.

To fix above issue, the proposal is to create a new API `AddOnPlacementScoreGenerator`, which could 
look like this for the example above:

~~~yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: AddOnPlacementScoreGenerator
metadata:
  name: cluster1-generator
  namespace: cluster1
spec:
  addOnPlacementSelector:
    name: latency
    namespace: cluster1
  latencies:
    - name: redhat-com-avgLatency
      url: https://redhat.com
      runs: 2
      waitBetweenRuns: 10s
    - name: google-com-avgLatency
      url: https://google.com
    - name: linux-com-avgLatency
      url: https://linux.com
~~~

Our controller can now read the `AddOnPlacementScoreGenerator` and our specific controller will be interested on 
everything below `.spec.latencies`. For example, the redhat-com-avgLatency will have the result of running two latency 
tests to https://redhat.com and waiting 10s between runs, then the mean value will be posted to the `AddOnPlacementScore` 
status.

## Goals & Non-Goals

### Goals

- Help the administrators to provide custom values to addon controllers that generate `AddOnPlacementScores`

### Non-Goals

- Addond developers need to develop their own controller to consume this new API.

### Future goals

- It is currently assumed that the user of `AddOnPlacementScoreGenerator` is either a 
cluster admin or a user who can create `AddOnPlacementScoreGenerator` in the hub cluster's 
managed "cluster namespace".

## Design

### Component & API

We purpose to adding a new custom resource named
`AddOnPlacementScoreGenerator` introduced into OCM by this proposal:

A sample of the `AddOnPlacementScoreGenerator` will be:

~~~yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: AddOnPlacementScoreGenerator
metadata:
  name: cluster1-generator
  namespace: cluster1
spec:
  addOnPlacementSelector:
    name: latency
    namespace: cluster1
  latencies:
    - name: redhat-com-avgLatency
      url: https://redhat.com
      runs: 2
      waitBetweenRuns: 10s
  otherPlugin:
    - name: score-name
      optionConsumedByPlugin: optionValue
      
~~~

The `AddOnPlacementScoreGenerator` resource is expected to be 
created under the "cluster namespace" which is a namespace with 
the same name as the managed cluster, the `AddOnPlacementScoreGenerator` 
delivered to the managed cluster will have the same name as the
`AddOnPlacementScoreGenerator` resource.

The addon controller must setup a watcher and reconcile `AddOnPlacementScoreGenerator`.

### Test Plan

- Unit tests
- Integration tests

### Graduation Criteria

#### Alpha

At first, This proposal will be in the alpha stage and needs to meet

1. The new APIs are reviewed and accepted;
2. Implementation is completed to support the functionalities;
3. Develop test cases to demonstrate this proposal works correctly;

#### Beta
1. Need to revisit the API shape before upgrading to beta based on user feedback.

### Upgrade / Downgrade Strategy
TBD

### Version Skew Strategy
N/A

## Alternatives
