# Improved Support for Managing Operators through Policy

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
  [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

Enhance the policy framework with a new kind, `OperatorPolicy`, to allow for better management of
OLM Operator installations on managed clusters. The controller for this kind would be included in
the configuration-policy-controller container, on managed clusters. Compared to current solutions,
the new kind and controller will provide more control and insight into the version of the operator
being managed, more feedback in the policy status regarding the operator's health, and more
integrated support for uninstallation scenarios.

## Motivation

It is currently possible to install OLM Operators on managed clusters with `ConfigurationPolicies`,
but multiple policies are often required, and these require the policy author to have a deep
understanding of the OLM resources being managed. For example, the policy author might need to
consider whether an OperatorGroup must be created, and incorrectly creating multiple OperatorGroups
can cause operator installations to fail. If the policy author wants to monitor the health of the
operator, they would need to create policies to monitor things like the status of the Subscription
(which can not necessarily be incorporated into the policy to create the Subscription), and related
resources which the operator manages like Deployments. Upgrading or uninstalling the operator would
require additional policies, and the status feedback from `ConfigurationPolicies` might not provide
enough useful information when things do not go according to plan in these scenarios, requiring
manual intervention from an admin.

A new controller can handle the relationships between resources better, for example knowing when an
OperatorGroup should be created, or identifying the Deployments and CustomResourceDefinitions
managed by a ClusterServiceVersion. This information could be incorporated into the policy status,
helping administrators understand the status of the operators from the hub cluster. The new kind can
also provide new controls over the upgrades and uninstallation of these operators, which would be
difficult with current solutions.

### Goals

1. It should be easy to install an operator on a managed cluster with just one policy.
2. It should be possible to specify what upgrades should be allowed performed on an operator.
3. The policy status should report on the health of the operator, both the OLM resources like the
   Subscription, as well as the associated Deployments.
4. The policy should report when the CatalogSource for the operator is unhealthy. It should be
   configurable whether this will cause the policy to become NonCompliant.
5. What happens when the policy is deleted should be configurable: the operator might be deleted, or
   the operator and its CRDs might be deleted, or nothing might be deleted. By default, only the
   Subscription and CSVs will be deleted (matching the suggestion from OLM).

### Non-Goals

1. The installation of OLM on managed clusters will not be automated by this enhancement.
2. Management of "Operands" will not be included in this new type - they should be defined in
   separate `ConfigurationPolicies`, possibly with a
   [dependency](../74-order-policy-execution/README.md) on the `OperatorPolicy`. ("Operand" here
   refers to resources that would be created by a user, and managed by the operator, eg instances of
   CRDs which are defined by the operator.)
3. Creation of `CatalogSources` will not be included in the new type - they should be defined in
   separate `ConfigurationPolicies`.

## Proposal

### OperatorPolicy Syntax

```yaml
apiVersion: policy.open-cluster-management.io/v1beta1
kind: OperatorPolicy
metadata:
  name: my-operator
spec:
  remediationAction: enforce # or inform
  severity: medium
  operatorGroup: # optional
    name: my-operator-group
    namespace: own-namespace
    target:
      namespaces:
        - foo
        - bar
    # selector: # One of selector or namespaces is allowed.
    #   matchLabels: # or matchExpressions
    #     mydomain.io/prod: "true"
    serviceAccountName: my-og-account # optional
  subscription:
    config: {} # optional
    channel: stable
    name: my-operator
    namespace: own-namespace
    installPlanApproval: Automatic # may be overridden to Manual by other settings
    source: my-catalog
    sourceNamespace: my-catalog-namespace
    startingCSV: my-operator.v0.1.0 # optional
  allowedCSVs:
    - my-operator.v0.1.1
    - my-operator.v0.2.0
  # - "*" # Allow all
  removalBehavior:
    operatorGroups: DeleteIfUnused
    subscriptions: Delete
    clusterServiceVersions: Delete
    installPlans: Keep
    customResourceDefinitions: Keep
    apiServiceDefinitions: Keep
  statusConfig:
    catalogSourceUnhealthy: Condition
    deploymentsUnavailable: NonCompliant
    upgradesAvailable: Condition
    upgradesProgressing: NonCompliant
```

When `remediationAction` is set to `inform`, no actions (creates, deletes, updates) will be taken on
any resources, but the current state may be *read* to identify if the operator is installed as
specified. When resources are missing, the policy will be NonCompliant, with a message specifying
what is missing, and that it will not be created because the policy is not being enforced. When it
is set to `enforce`, OperatorGroups and Subscriptions may be created by the controller, InstallPlans
may be approved, and resources may be removed when the policy is deleted.

The optional `spec.operatorGroup` section defines an
[OperatorGroup](https://github.com/operator-framework/api/blob/master/crds/operators.coreos.com_operatorgroups.yaml).
If not specified, and if the operator supports this InstallMode, then it will ensure that an
[AllNamespaces](https://olm.operatorframework.io/docs/advanced-tasks/operator-scoping-with-operatorgroups/#targetnamespaces-and-their-relationship-to-installmodes)
OperatorGroup exists in the namespace of the subscription.

The `spec.subscription` section includes the fields from the spec of an [OLM
Subscription](https://github.com/operator-framework/api/blob/master/crds/operators.coreos.com_subscriptions.yaml).
For more information on the `config` field, see:
https://github.com/operator-framework/api/blob/v0.17.3/pkg/operators/v1alpha1/subscription_types.go#L42.
The `installPlanApproval` field is optional. If the policy is in `enforce` mode and the allowed CSVs
are restricted as specified below, then the `installPlanApproval` field on the Subscription will
always be Manual, regardless of the setting here.

The `spec.allowedCSVs` list specifies what installed versions are considered Compliant when the
policy is in `inform` mode, and which InstallPlans can be approved when in `enforce` mode. Only
exact matches are considered, except for a special value, "*" which matches any version. This field
must be set, and must have at least one item.

The `spec.removalBehavior` field allows configuration of what will be removed when the policy is
deleted. Each kind that might be deleted should support `Keep` and `Delete`, and can potentially
have additional values specific to the resources in question. For example, the OperatorGroup should
likely only be removed if it isn't being used by another subscription. 

The `spec.statusConfig` field allows the author to specify what effect specific resource statuses
will have on the OperatorPolicy status and compliance events. The currently planned values are:
`Condition`, which will record the status in `status.conditions` and in the compliance event; and
`NonCompliant`, which additionally updates the `status.compliant` field to NonCompliant which can
cause the root policy on the hub to become NonCompliant and may throw an alert. Additional values
could be added in the future.

### OperatorPolicy Status and Compliance Event Messages

```yaml
status:
  compliant: Compliant
  conditions:
    - lastTransitionTime: "2023-07-14T07:34:28Z"
      message: All operator Deployments have their minimum availability.
      reason: DeploymentsAvailable
      status: "True"
      type: Compliant
    - lastTransitionTime: "2023-07-11T14:58:54Z"
      message: all available catalogsources are healthy
      reason: AllCatalogSourcesHealthy
      status: "False"
      type: CatalogSourcesUnhealthy
    - lastTransitionTime: "2023-07-11T14:59:06Z"
      reason: RequiresApproval
      status: "True"
      type: InstallPlanPending
    - lastTransitionTime: "2023-07-14T07:34:28Z"
      message: All operator Deployments have their minimum availability.
      reason: MinimumReplicasAvailable
      status: "True"
      type: DeploymentsAvailable
  relatedObjects:
    - compliant: Compliant
      object:
        apiVersion: operators.coreos.com/v1alpha1
        kind: Subscription
        metadata:
          name: my-operator
          namespace: own-namespace
      reason: Resource created
    - compliant: NonCompliant
      object:
        apiVersion: operators.coreos.com/v1alpha1
        kind: InstallPlan
        metadata:
          name: install-ww7c2
          namespace: own-namespace
      reason: InstallPlan not approved
    - compliant: Compliant
      object:
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: my-operator
          namespace: own-namespace
      reason: Deployment Available
    ...
```

As required for all policies, there is a `status.compliant` field for the overall status. The status
will also have a `conditions` section, which includes the conditions from the Subscription, such as
"InstallPlanPending", as well a condition "DeploymentsAvailable" which will reflect whether all of
the related Deployments are available. It will also include a condition of type "Compliant", which
has a status of "True" when the policy is Compliant. The reason and message will reflect the cause
of the most recent compliance event that was emitted.

There is also a `status.relatedObjects` for the OLM objects (CatalogSource, Subscription,
InstallPlan, etc) related to this operator installation, and the Deployments and ServiceAccounts
listed in the OperatorCondition. Each of those related objects has a `compliant` field, but a
NonCompliant object will not necessarily cause the policy to become NonCompliant: for example if the
CatalogSource is unhealthy, `spec.statusConfig.catalogSourceUnhealthy` must be considered. The
`reason` associated with each object is a brief human-readable explanation for the value of the
`compliant` field. This matches the conventions of ConfigurationPolicy.

When a condition changes the controller will emit a policy compliance event like other policy
controllers. In particular, the event message should begin with "Compliant; " or "NonCompliant; "
based on the `status.compliant` field, followed by an explanation of the current state. That
explanation will include information from all of the current conditions, ordered by their last
transition times, so that the most recent change is first. For the status above, the event would
look like: 

```yaml
action: ComplianceStateUpdate
apiVersion: v1
count: 1
eventTime: null
firstTimestamp: "2023-07-14T07:34:28Z"
involvedObject:
  apiVersion: policy.open-cluster-management.io/v1
  kind: Policy
  name: policies.my-operator
  namespace: local-cluster
kind: Event
lastTimestamp: "2023-07-14T07:34:28Z"
message: >-
  Compliant; all operator deployments have their minimum availability, all available catalogsources 
  are healthy, an installplan is pending but not causing noncompliance.
metadata:
  creationTimestamp: "2023-07-14T07:34:28Z"
  name: policies.my-operator.1771b8bf88adf1ec
  namespace: local-cluster
reason: 'policy: local-cluster/my-operator'
related:
  apiVersion: policy.open-cluster-management.io/v1beta1
  kind: OperatorPolicy
  name: my-operator
  namespace: local-cluster
reportingComponent: operator-policy-controller
reportingInstance: config-policy-controller-7488748b9d-kkvvw
source:
  component: operator-policy-controller
  host: config-policy-controller-7488748b9d-kkvvw
type: Normal
```

### User Stories

#### Story 1

As a policy user, I want to install an operator of a specific version, from a specific source, on a
managed cluster, in "AllNamespaces" mode. The policy should become compliant when the InstallPlan is
complete and all of Deployments listed in the operator's CSV are healthy.

Example yaml to apply (not including the Placement):
```yaml
apiVersion: policy.open-cluster-management.io/v1beta1
kind: OperatorPolicy
metadata:
  name: strimzi-policy
spec:
  remediationAction: enforce
  severity: medium
  subscription:
    channel: stable
    name: strimzi-kafka-operator
    namespace: openshift-operators
    installPlanApproval: Manual
    source: community-operators
    sourceNamespace: openshift-marketplace
    startingCSV: strimzi-cluster-operator.v0.35.0
  allowedCSVs:
    - strimzi-cluster-operator.v0.35.0
```

#### Story 2

As a policy user, I want to be able to control updates to operators so the managed clusters only use
versions that I've verified work properly. After I verify a newer version works, I should be able to
update the policy with that version, and only then should the installations on managed clusters
progress to that version.

For example, to upgrade the operator, the policy from story 1 could be updated to match this:
```yaml
apiVersion: policy.open-cluster-management.io/v1beta1
kind: OperatorPolicy
metadata:
  name: strimzi-policy
spec:
  remediationAction: enforce
  severity: medium
  subscription:
    channel: stable
    name: strimzi-kafka-operator
    namespace: openshift-operators
    installPlanApproval: Manual
    source: community-operators
    sourceNamespace: openshift-marketplace
    startingCSV: strimzi-cluster-operator.v0.35.0
  allowedCSVs:
    - strimzi-cluster-operator.v0.35.0
    - strimzi-cluster-operator.v0.35.1
```

#### Story 3

As a policy user, I want to be alerted (via a NonCompliant policy) when an operator has an upgrade
available. The policy status should clearly indicate what new version is available.

The policy yaml would look like this, including some relevant fields from the status:
```yaml
apiVersion: policy.open-cluster-management.io/v1beta1
kind: OperatorPolicy
metadata:
  name: strimzi-policy
spec:
  remediationAction: enforce
  severity: medium
  subscription:
    channel: stable
    name: strimzi-kafka-operator
    namespace: openshift-operators
    installPlanApproval: Manual
    source: community-operators
    sourceNamespace: openshift-marketplace
    startingCSV: strimzi-cluster-operator.v0.35.0
  allowedCSVs:
    - strimzi-cluster-operator.v0.35.0
  statusConfig:
    upgradesAvailable: NonCompliant
status:
  compliant: NonCompliant
  conditions:
    - lastTransitionTime: "2023-07-14T07:34:28Z"
      message: An upgrade to strimzi-cluster-operator.v0.35.1 is available on the stable channel.
      reason: UpgradeAvailable
      status: "False"
      type: Compliant
    - lastTransitionTime: "2023-07-11T14:59:06Z"
      reason: RequiresApproval
      status: "True"
      type: InstallPlanPending
    ...
  relatedObjects:
    - compliant: NonCompliant
      object:
        apiVersion: operators.coreos.com/v1alpha1
        kind: InstallPlan
        metadata:
          name: install-ww7c2
          namespace: openshift-operators
      reason: InstallPlan not approved
    ...
```

#### Story 4

As a policy user, I want to monitor the health of an operator installation and related objects, with
just one policy. I want the policy to be NonCompliant whenever any of those are unhealthy, and when
the operator is in the process of being upgraded (basically, whenever something might be impacting
the performance of the operator).

```yaml
apiVersion: policy.open-cluster-management.io/v1beta1
kind: OperatorPolicy
metadata:
  name: strimzi-policy
spec:
  remediationAction: inform
  severity: medium
  subscription:
    channel: stable
    name: strimzi-kafka-operator
    namespace: openshift-operators
    installPlanApproval: Automatic
    source: community-operators
    sourceNamespace: openshift-marketplace
  allowedCSVs:
    - "*"
  statusConfig:
    catalogSourceUnhealthy: NonCompliant
    deploymentsUnavailable: NonCompliant
    upgradesAvailable: Condition
    upgradesProgressing: NonCompliant
```

The `status` of this policy will be populated with details about the OperatorGroup, Subscription,
CatalogSource, ClusterServiceVersion, and InstallPlan, as well as the Deployments that are listed in
the CSV.

#### Story 5

As a policy user, I want to install an operator to a specific namespace via a policy. The policy
should become Compliant when the operator Deployments are fully available.

```yaml
apiVersion: policy.open-cluster-management.io/v1beta1
kind: OperatorPolicy
metadata:
  name: strimzi-policy
spec:
  remediationAction: enforce
  severity: medium
  operatorGroup:
    name: og-strimzi
    namespace: strimzi-app-one
    target:
      namespaces:
        - strimzi-app-one
  subscription:
    channel: stable
    name: strimzi-kafka-operator
    namespace: strimzi-app-one
    installPlanApproval: Automatic
    source: community-operators
    sourceNamespace: openshift-marketplace
  allowedCSVs:
    - "*"
```

#### Story 6

As a policy user, I want an operator that I installed via a policy to be cleaned up when I remove
the policy from the cluster. The types that I specify in the policy should not be present when the
policy deletion is finalized.

```yaml
apiVersion: policy.open-cluster-management.io/v1beta1
kind: OperatorPolicy
metadata:
  name: strimzi-policy
spec:
  remediationAction: enforce
  severity: medium
  subscription:
    channel: stable
    name: strimzi-kafka-operator
    namespace: strimzi-app-one
    installPlanApproval: Automatic
    source: community-operators
    sourceNamespace: openshift-marketplace
  allowedCSVs:
    - "*"
  removalBehavior:
    operatorGroups: DeleteIfUnused
    subscriptions: Delete
    clusterServiceVersions: Delete
    installPlans: Keep
    customResourceDefinitions: Keep
    apiServiceDefinitions: Keep
```

It should be noted that the `removalBehavior` specified here is just the default setting made
explicit.

### Implementation Details/Notes/Constraints [optional]

#### Implementation of Pruning

In order to prune objects when the policy is deleted, the policy will need finalizers on the managed
cluster. Care should be taken to ensure that the finalizers will be cleaned up properly when the
controller is uninstalled. Deleting CRDs will be limited to the objects listed as "owned" in the
bundle.

#### Status and Compliance Event

What is specified in this doc for the status and the event is meant to match what is already being
done in other policy controllers (with the addition of conditions, which are not implemented in
other controllers). If this spec disagrees with what is being done in other controllers, the
behavior of the established controllers should be used.

#### Approving InstallPlans

InstallPlans have ownerReferences to all Subscriptions in that namespace, regardless of whether or
not those operators would be upgraded or installed by the InstallPlan. Additionally, when multiple
operators have updates or new installations, OLM might create one InstallPlan for resources
specified by any or all of those operators. It should also be noted that if *any* Subscriptions in
the namespace have `installPlanApproval: Manual`, then *no* InstallPlans in the namespace will be
automatically approved by OLM.

Therefore, approving an InstallPlan will not be as simple as looking up one relevant OperatorPolicy
and checking its rules for approval. *Every* OperatorPolicy that works on that namespace on the
cluster must be checked, and the planned resources must be matched to the relevant policies in order
to determine whether it should be approved. If any planned resources are not specifically approved
by an OperatorPolicy (for example, if they are the result of a Subscription created by another
process), then the InstallPlan should not be approved.

A potential way to track resources to OperatorPolicies is to use the `status.bundleLookups` list in
the InstallPlan. In each of these, there is an `identifier` field which matches a CSV name, and a
`properties` field which is an embedded JSON that includes information identifying the package's
name. Here's an elided example:
```yaml
status:
  bundleLookups:
  - catalogSourceRef:
      name: community-operators
      namespace: openshift-marketplace
    identifier: etcdoperator.v0.9.4
    properties: '{"properties":[...,{"type":"olm.package","value":{"packageName":"etcd","version":"0.9.4"}}]}'
    replaces: etcdoperator.v0.9.2
```

That `packageName` should match a Subscription's `spec.name` (which conventionally is the same as
its `metadata.name`, and *will* always be the same when created by an OperatorPolicy).

### Risks and Mitigation

### Open Questions [optional]

1. Is making `spec.operatorGroup` optional, and defaulting to an AllNamespaces OperatorGroup, the
   best option?
2. Could/should there be default catalogs, so that users only need to specify the operator name?

### Test Plan

**Note:** *Section not required until targeted at a release.*

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

It would be GA in the release after implementation.

### Upgrade / Downgrade Strategy

### Version Skew Strategy

## Implementation History

N/A

## Drawbacks

An additional policy type and controller will potentially increase resource usage.

## Alternatives

It is already possible to manage specific parts of OLM operators with just ConfigurationPolicies.

## Infrastructure Needed [optional]

No new infrastructure needed.
