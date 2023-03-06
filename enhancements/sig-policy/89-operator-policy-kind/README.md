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
the configuration-policy-controller container, on managed clusters. The new controller would better
handle imperative installation steps for OLM objects, and provide better support for upgrade and
uninstallation scenarios.

## Motivation

It is currently possible to install OLM Operators on managed clusters with `ConfigurationPolicies`,
but it can be complex and error-prone. For example, multiple policies are often needed in order to
manage the different OLM resources such as OperatorGroup, Subscription, InstallPlan, and
ClusterServiceVersion. Upgrading or uninstalling an operator can also be difficult with
ConfigurationPolicies, since it is not generally as easy as just updating one value or deleting a
resource.

Creating a new policy type will help streamline the process of declaring an operator to be installed
on a cluster. A new controller will help cover specialized cases that come up when managing operator
installations, rather than teaching the existing configuration-policy-controller to deal with
specific types.

### Goals

1. It should be easy to install an operator on a managed cluster with just one policy.
2. It should be possible to limit updates to operators on clusters, so they do not upgrade beyond a
   known good version defined in the policy.
3. The policy managing the operator should clearly report back the status of the installation, with
   clear violation messages if something is not right.
4. What happens when the policy is deleted should be configurable: the operator might be deleted, or
   the operator and its CRDs might be deleted, or nothing might be deleted.

### Non-Goals

1. The installation of OLM on managed clusters will not be automated by this enhancement.
2. Management of "Operands" will not be included in this new type - they should be defined in
   separate `ConfigurationPolicies`, possibly with a
   [dependency](../74-order-policy-execution/README.md) on the `OperatorPolicy`.
3. Management of `CatalogSources` will not be included in the new type - they should be defined in
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
    #   selector: # One of selector or namespaces is allowed.
    #     matchLabels: # or matchExpressions
    #       mydomain.io/prod: "true"
    serviceAccountName: my-og-account # optional
  subscription:
    channel: stable
    name: my-operator
    source: my-catalog
    sourceNamespace: my-catalog-namespace
    startingCSV: my-operator.v0.1.0 # optional
    endingCSV: Any # or None or e.g. 'my-operator.v0.2.0`
  requireLatestVersion: false
  pruneOnDeletion:
    - subscriptions
    - clusterserviceversions
    - installplans
    - customresourcedefinitions
    - apiservicedefinitions
```

The optional `spec.operatorGroup` section defines an
[OperatorGroup](https://github.com/operator-framework/api/blob/master/crds/operators.coreos.com_operatorgroups.yaml).
If not specified, and if the operator supports this InstallMode, then an
[AllNamespaces](https://olm.operatorframework.io/docs/advanced-tasks/operator-scoping-with-operatorgroups/#targetnamespaces-and-their-relationship-to-installmodes)
OperatorGroup will be created.

The `spec.subscription` section includes fields from the spec of an [OLM
Subscription](https://github.com/operator-framework/api/blob/master/crds/operators.coreos.com_subscriptions.yaml),
excluding `config` and `installPlanApproval`. The `endingCSV` field allows configuration of which
InstallPlans will be approved by the controller when the policy is set to `enforce`.

The `spec.requireLatestVersion` field configures whether the "latest" version of the operator
defined in the catalog on this cluster "should" be installed. Note that if the "latest" version is
greater than the allowed `endingCSV`, then that version will not be automatically approved even if
the `remediationAction` is set to enforce: instead, the policy will be NonCompliant.

The `pruneOnDeletion` field allows configuration of what will be removed when the policy is deleted.
A "recommended" setting for most workloads would be `[subscriptions, clusterserviceversions]`,
leaving behind CRDs since they can often require extra attention to properly remove.

### OperatorPolicy Status Syntax

```yaml
status:
  compliant: Compliant
  conditions:
    - lastTransitionTime: '2023-02-23T19:58:52Z'
      message: 'operator is installed as specified'
      reason: OperatorInstalled
      status: 'True'
      type: Compliant
  relatedObjects:
    - compliant: Compliant
      object:
        apiVersion: operators.coreos.com/v1alpha1
        kind: Subscription
        metadata:
          name: my-operator
          namespace: own-namespace
      reason: Resource created
```

As required for all policies, there is a `status.compliant` field for the overall status. There is
also a `status.relatedObjects` field with the same format as the one in ConfigurationPolicy, for the
OLM objects related to this operator installation.

New (compared to other policies) here is a `status.conditions` field compatible with `oc wait
--for=...`. At the very least, this will include a condition of type "Compliant`, but additional
fields for internal state may be exposed, for example when waiting for a ClusterServiceVersion after
approving an InstallPlan.

### User Stories

#### Story 1

As a policy user, I want to install, configure, and uninstall operators on my managed clusters
through policies.

### Implementation Details/Notes/Constraints [optional]

In order to prune objects when the policy is deleted, the policy will need finalizers on the managed
cluster. Care should be taken to ensure that the finalizers will be cleaned up properly when the
controller is uninstalled.

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
