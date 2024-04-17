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
5. It should be possible to remove an operator installation with a policy. Exactly which resources
   are removed should be configurable to match what the user wants: by default only the Subscription
   and ClusterServiceVersion will be removed (matching the suggestion from OLM) but additionally
   things like the CustomResourceDefinitions or InstallPlans could be removed.

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

Note: See Implementation Details for the previous `v1beta1` version.

```yaml
apiVersion: policy.open-cluster-management.io/v1beta2
kind: OperatorPolicy
metadata:
  name: my-operator
spec:
  remediationAction: enforce # or inform
  severity: medium
  complianceType: musthave # or mustnothave
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
    source: my-catalog
    sourceNamespace: my-catalog-namespace
    startingCSV: my-operator.v0.1.0 # optional
  versions:
    - my-operator.v0.1.1
    - my-operator.v0.2.0
  upgradeApproval: Automatic # or Never
  removalBehavior:
    operatorGroups: DeleteIfUnused
    subscriptions: Delete
    clusterServiceVersions: Delete
    installPlans: Keep
    customResourceDefinitions: Keep
  statusConfig:
    catalogSourceUnhealthy: StatusMessageOnly
    deploymentsUnavailable: NonCompliant
    upgradesAvailable: StatusMessageOnly
    upgradesProgressing: NonCompliant
```

When `remediationAction` is set to `inform`, no actions (creates, deletes, updates) will be taken on
any resources, but the current state may be *read* to identify if the operator is installed as
specified. When resources are missing, the policy will be NonCompliant, with a message specifying
what is missing, and that it will not be created because the policy is not being enforced. When it
is set to `enforce`, OperatorGroups and Subscriptions may be created by the controller, InstallPlans
may be approved, and resources may be removed when the policy is deleted.

The `complianceType` is either "musthave" or "mustnothave" and follows the idea of the same field in
ConfigurationPolicy. In "musthave" mode, the operator must be present on the cluster as specified in
the policy for the policy to be marked as Compliant. In "mustnothave" mode, if the specified
operator is on the cluster (found by matching the name, namespace, source and sourceNamespace, and
optionally matching one of the specified versions), then the policy will be marked as NonCompliant.

The optional `spec.operatorGroup` section defines an
[OperatorGroup](https://github.com/operator-framework/api/blob/master/crds/operators.coreos.com_operatorgroups.yaml).
If not specified, the controller will check for an existing OperatorGroup in the namespace that can
be used. If none exists, it will create an
[AllNamespaces](https://olm.operatorframework.io/docs/advanced-tasks/operator-scoping-with-operatorgroups/#targetnamespaces-and-their-relationship-to-installmodes)
OperatorGroup with a generated name.

The `spec.subscription` section includes the fields from the spec of an [OLM
Subscription](https://github.com/operator-framework/api/blob/master/crds/operators.coreos.com_subscriptions.yaml).
For more information on the `config` field, see:
https://github.com/operator-framework/api/blob/v0.17.3/pkg/operators/v1alpha1/subscription_types.go#L42.
We plan to make many of these fields optional, the controller can fill in required Subscription
fields based on specified defaults in the operator's PackageManifest. For example, this allows the
default channel to possibly be different on different clusters, reflecting the default channel in
each cluster's catalog. It is not allowed to set `spec.subscription.installPlanApproval`; the policy
will determine and set a value for that field based on the policy's `spec.versions` and
`spec.upgradeApproval` settings.

In "musthave" mode, the `spec.versions` list specifies what installed versions are considered
Compliant when the policy is in `inform` mode, and which InstallPlans can be approved when in
`enforce` mode. Conversely, in "mustnothave" mode, this list specifies which versions are considered
NonCompliant. Only exact matches are considered. If the list is unset or empty, then any version on
the cluster will be considered a match. If `spec.subscription.startingCSV` is set, that version can
be approved, even if it is not in this list.

The `spec.upgradeApproval` field specifies whether an enforced "musthave" policy will approve any
upgrade InstallPlans for the operator. *It has no effect when the policy is in "mustnothave" mode*.
This only affects InstallPlans for operators that are already installed on the cluster,
which upgrade or replace the operator; initial InstallPlans for an operator can be approved
regardless of this flag. Allowed values here include `Automatic` and `None`.

Only when the policy's `spec.upgradeApproval` is set to `Automatic`, and the `spec.versions` field
is empty (allowing all versions of the operator) will the subscription managed by this policy have
its `installPlanApproval` field set to `Automatic`. Otherwise, the field will be set to `Manual`,
but it should be noted that this controller will approve InstallPlans matching the desired versions
automatically.

The `spec.removalBehavior` field allows configuration of what might be removed by the controller
when the policy is in "mustnothave" mode. *It has no effect when the policy is in "musthave" mode.*
Resources that would not be removed in `enforce` mode will not cause NonCompliance when the policy
is in `inform` mode. Status messages should reflect that those resources were kept because of the
configuration of the policy. Each kind here will support `Keep` and `Delete`, and can potentially
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
    - lastTransitionTime: '2024-04-09T14:41:44Z'
      message: CatalogSource was found
      reason: CatalogSourcesFound
      status: 'False'
      type: CatalogSourcesUnhealthy
    - lastTransitionTime: '2024-04-09T14:42:11Z'
      message: ClusterServiceVersion - install strategy completed with no errors
      reason: InstallSucceeded
      status: 'True'
      type: ClusterServiceVersionCompliant
    - lastTransitionTime: '2024-04-09T14:42:11Z'
      message: >-
        Compliant; the policy spec is valid, the policy does not specify an
        OperatorGroup but one already exists in the namespace - assuming that
        OperatorGroup is correct, the Subscription matches what is required by
        the policy, no InstallPlans requiring approval were found,
        ClusterServiceVersion - install strategy completed with no errors, All
        operator Deployments have their minimum availability, CatalogSource was
        found
      reason: Compliant
      status: 'True'
      type: Compliant
    - lastTransitionTime: '2024-04-09T14:42:10Z'
      message: All operator Deployments have their minimum availability
      reason: DeploymentsAvailable
      status: 'True'
      type: DeploymentCompliant
    ...
  relatedObjects:
    - compliant: Compliant
      object:
        apiVersion: operators.coreos.com/v1alpha1
        kind: CatalogSource
        metadata:
          name: redhat-operators
          namespace: openshift-marketplace
      reason: Resource found as expected
      cluster: local-cluster
    - compliant: Compliant
      object:
        apiVersion: operators.coreos.com/v1alpha1
        kind: ClusterServiceVersion
        metadata:
          name: quay-operator.v3.10.4
          namespace: openshift-operators
      properties:
        uid: 0fdb6c2d-f37a-4bf8-be0a-a6ffbd6a397d
      reason: InstallSucceeded
      cluster: local-cluster
    - compliant: Compliant
      object:
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: quay-operator.v3.10.4
          namespace: openshift-operators
      properties:
        uid: 5f390b4b-4576-4f59-a89c-46c35a6d64d8
      reason: Deployment Available
      cluster: local-cluster
    - compliant: Compliant
      object:
        apiVersion: operators.coreos.com/v1alpha1
        kind: Subscription
        metadata:
          name: quay-operator
          namespace: openshift-operators
      properties:
        createdByPolicy: true
        uid: 8f8d438b-f62a-4966-ad1e-67edf1181c58
      reason: Resource found as expected
      cluster: local-cluster
    ...
```

As required for all policies, there is a `status.compliant` field for the overall status. The status
will also have a `conditions` section, which includes the conditions from the Subscription, such as
"CatalogSourcesUnhealthy", as well a condition "DeploymentCompliant" which will reflect whether all 
of the related Deployments are available. It also includes a condition of type "Compliant", which
has a status of "True" when the policy is Compliant. The reason and message will reflect the cause
of the most recent compliance event that was emitted.

There is also a `status.relatedObjects` for the OLM objects (CatalogSource, Subscription,
InstallPlan, etc) related to this operator installation, and the Deployment(s) for the operator. 
Each of those related objects has a `compliant` field, but it should be noted that a NonCompliant
object will not necessarily cause the policy to become NonCompliant: for example if the
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
firstTimestamp: "2024-04-09T14:42:11Z"
involvedObject:
  apiVersion: policy.open-cluster-management.io/v1
  kind: Policy
  name: open-cluster-management-global-set.quay-op-oppolicy
  namespace: local-cluster
  uid: 900ea3bd-7efd-4272-9031-482c566fb99a
kind: Event
lastTimestamp: "2024-04-09T14:42:11Z"
message: Compliant; the policy spec is valid, the policy does not specify an OperatorGroup
  but one already exists in the namespace - assuming that OperatorGroup is correct,
  the Subscription matches what is required by the policy, no InstallPlans requiring
  approval were found, ClusterServiceVersion - install strategy completed with no
  errors, All operator Deployments have their minimum availability, CatalogSource
  was found
metadata:
  creationTimestamp: "2024-04-09T14:42:11Z"
  name: open-cluster-management-global-set.quay-op-oppolicy.17c4a3af2d7bd804
  namespace: local-cluster
  resourceVersion: "155720"
  uid: 122b274e-a364-4d40-b45a-c1f3b00ade9c
reason: 'policy: local-cluster/quay-operator'
related:
  apiVersion: policy.open-cluster-management.io/v1beta1
  kind: OperatorPolicy
  name: quay-operator
  namespace: local-cluster
  uid: 74eb02ff-2d23-41fa-a3c6-447be0712564
reportingComponent: configuration-policy-controller
reportingInstance: config-policy-controller-6b7f78b87-q697c
source:
  component: configuration-policy-controller
  host: config-policy-controller-6b7f78b87-q697c
type: Normal
```

### User Stories

#### Story 1

As a policy user, I want to install an operator of a specific version, from a specific source, on a
managed cluster, in "AllNamespaces" mode. The policy should become compliant when the InstallPlan is
complete and all of Deployments listed in the operator's CSV are healthy.

Example yaml to apply (not including the Placement):
```yaml
apiVersion: policy.open-cluster-management.io/v1beta2
kind: OperatorPolicy
metadata:
  name: strimzi-policy
spec:
  remediationAction: enforce
  severity: medium
  complianceType: musthave
  subscription:
    channel: stable
    name: strimzi-kafka-operator
    namespace: openshift-operators
    source: community-operators
    sourceNamespace: openshift-marketplace
    startingCSV: strimzi-cluster-operator.v0.35.0
  upgradeApproval: None
  versions:
    - strimzi-cluster-operator.v0.35.0
```

#### Story 2

As a policy user, I want to be able to control updates to operators so the managed clusters only use
versions that I've verified work properly. After I verify a newer version works, I should be able to
update the policy with that version, and only then should the installations on managed clusters
progress to that version.

For example, to upgrade the operator, the policy from story 1 could be updated to match this:
```yaml
apiVersion: policy.open-cluster-management.io/v1beta2
kind: OperatorPolicy
metadata:
  name: strimzi-policy
spec:
  remediationAction: enforce
  severity: medium
  complianceType: musthave
  subscription:
    channel: stable
    name: strimzi-kafka-operator
    namespace: openshift-operators
    source: community-operators
    sourceNamespace: openshift-marketplace
    startingCSV: strimzi-cluster-operator.v0.35.0
  upgradeApproval: Automatic
  versions:
    - strimzi-cluster-operator.v0.35.0
    - strimzi-cluster-operator.v0.35.1
```

#### Story 3

As a policy user, I want to be alerted (via a NonCompliant policy) when an operator has an upgrade
available. The policy status should clearly indicate what new version is available.

The policy yaml would look like this, including some relevant fields from the status:
```yaml
apiVersion: policy.open-cluster-management.io/v1beta2
kind: OperatorPolicy
metadata:
  name: strimzi-policy
spec:
  remediationAction: enforce
  severity: medium
  complianceType: musthave
  subscription:
    channel: stable
    name: strimzi-kafka-operator
    namespace: openshift-operators
    source: community-operators
    sourceNamespace: openshift-marketplace
    startingCSV: strimzi-cluster-operator.v0.35.0
  upgradeApproval: None
  versions:
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
apiVersion: policy.open-cluster-management.io/v1beta2
kind: OperatorPolicy
metadata:
  name: strimzi-policy
spec:
  remediationAction: inform
  severity: medium
  complianceType: musthave
  subscription:
    channel: stable
    name: strimzi-kafka-operator
    namespace: openshift-operators
    source: community-operators
    sourceNamespace: openshift-marketplace
  upgradeApproval: Automatic
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
apiVersion: policy.open-cluster-management.io/v1beta2
kind: OperatorPolicy
metadata:
  name: strimzi-policy
spec:
  remediationAction: enforce
  severity: medium
  complianceType: musthave
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
    source: community-operators
    sourceNamespace: openshift-marketplace
  upgradeApproval: Automatic
```

#### Story 6

As a policy user, I want to ensure a certain operator is not on the cluster. When enforced, objects
should be deleted based on what I specify in `spec.removalBehavior`.

```yaml
apiVersion: policy.open-cluster-management.io/v1beta2
kind: OperatorPolicy
metadata:
  name: strimzi-policy
spec:
  remediationAction: enforce
  severity: medium
  complianceType: mustnothave
  subscription:
    channel: stable
    name: strimzi-kafka-operator
    namespace: strimzi-app-one
    source: community-operators
    sourceNamespace: openshift-marketplace
  upgradeApproval: Automatic
  removalBehavior:
    operatorGroups: DeleteIfUnused
    subscriptions: Delete
    clusterServiceVersions: Delete
    installPlans: Keep
    customResourceDefinitions: Keep
```

It should be noted that the `removalBehavior` specified here is just the default setting made
explicit.

#### Story 7

As a policy user, I want to ensure that a certain version of an operator is not present on the
cluster. If that version of the operator is found, I want its installation to be deleted.

```yaml
apiVersion: policy.open-cluster-management.io/v1beta1
kind: OperatorPolicy
metadata:
  name: strimzi-policy
spec:
  remediationAction: enforce
  severity: medium
  complianceType: mustnothave
  subscription:
    channel: stable
    name: strimzi-kafka-operator
    namespace: strimzi-app-one
    source: community-operators
    sourceNamespace: openshift-marketplace
  upgradeApproval: Automatic
  versions:
    - "strimzi-cluster-operator.v0.35.0"
  removalBehavior:
    operatorGroups: DeleteIfUnused
    subscriptions: Delete
    clusterServiceVersions: Delete
    installPlans: Keep
    customResourceDefinitions: Keep
```

### Implementation Details/Notes/Constraints [optional]

#### Implementation of Pruning

Originally, we planned to use finalizers to remove the operator when the policy is deleted. But,
with some more experience with that feature in ConfigurationPolicy, we believe it is not the most
reliable method. It is not obvious that removing a policy should remove those resources; it requires
some policy expertise. Additionally, because policies are distributed through Placements, there are
many potential ways the policy could be removed from a cluster unexpectedly.

We are currently only planning a "mustnothave" mode, simliar to what was originally released for
ConfigurationPolicy. If a user wants to remove the operator, they can switch the policy to that
mode, and wait for it to become compliant. This also allows for a better reporting of this status:
when a policy is deleted, how can it report if a resource it is trying to delete gets stuck?

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

Initially, we plan to only approve InstallPlans dealing with one operator: if an OperatorPolicy
exists for that operator, and the version to be installed is allowed by that policy's `versions`
setting, then that InstallPlan will be approved. If an InstallPlan is created for multiple operator
installations, it will not be approved by this controller, and it might cause an OperatorPolicy to
become NonCompliant.

#### v1beta1 and v1beta2 Differences and Conversions

For reference, the `v1beta1` implementation used:
```yaml
apiVersion: policy.open-cluster-management.io/v1beta1
kind: OperatorPolicy
metadata:
  name: my-operator
spec:
  remediationAction: enforce # or inform
  severity: medium
  complianceType: musthave
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
    installPlanApproval: Automatic # or Manual, may be overridden by other settings.
    source: my-catalog
    sourceNamespace: my-catalog-namespace
    startingCSV: my-operator.v0.1.0 # optional
  versions:
    - my-operator.v0.1.1
    - my-operator.v0.2.0
```

An initial implementation of this kind used `v1beta1` and focused on installation and upgrade
scenarios. It *required* `spec.subscription.installPlanApproval` to be set to either `Automatic` or
`Manual`, but that setting was believed to be confusing. When set to `Automatic`, the contoller
needed to override that setting in the Subscription in order to restrict the possible upgrades, and
approve any upgrades matching the `versions` specified in the policy. And when set to `Manual`, the
controller would still approve those upgrades without user input, ie "automatically".

The breaking change in the `v1beta2` design is to *forbid* the user from setting the 
`spec.subscription.installPlanApproval` field in the subscription, while providing a similar setting 
via `spec.upgradeApproval`. The revised design provides more details on the interactions of this
setting and the other fields.

The controller will expose a conversion webhook so that either version of the resource can be used.
It will convert `installPlanApproval: Automatic` to `upgradeApproval: Automatic` and vice-versa, and
`installPlanApproval: Manual` to `upgradeApproval: None` and vice-versa. This is effectively a
change in behavior: as stated earlier, originally policies using `installPlanApproval: Manual` would
approve some upgrades, but now those policies will *not approve any upgrades*. Since the original
behavior was unclear in the design, and the new behavior results in fewer actions by the controller
(and therefore, fewer possible surprises for the user), we believe this change is acceptable.

The policy framework does not need to "know" about this change. Thanks to the conversion webhook,
it can apply whatever version is defined in the Policy: if that version is not registered on the
managed cluster, it would have a `template-error` caused by the API call failing.

### Risks and Mitigation

### Open Questions [optional]

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
