# Native Support for Gatekeeper Constraints in Policies

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
      [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

Enhance the Open Cluster Management (OCM) policy framework to allow Gatekeeper `ConstraintTemplate` and constraint
manifests to be directly embedded in OCM `Policy` object's `policy-templates` array for multicluster distribution and
status aggregation.

## Motivation

The Open Cluster Management (OCM) Policy addon does not have native support for an admission controller. Gatekeeper is
currently favored by the OCM Policy SIG for its flexibility and performance. Though the installation of Gatekeeper will
not be simplified by this enhancement, there is a desire to provide native OCM Policy support for Gatekeeper
`ConstraintTemplate` and constraint objects. This will allow users to skip using `ConfigurationPolicy` objects that wrap
Gatekeeper constraint manifests.

This also allows the audit messages from Gatekeeper constraints to be aggregated in the policy status on the OCM hub,
which is not possible by the wrapping approach. The wrapping approach involves an inform mode `ConfigurationPolicy` that
checks that the Gatekeeper constraint's `status.totalViolations` field is set to `0`. If it's not, a violation message
of `violation - k8srequiredlabels not found: [ns-must-have-gk] found but not as specified` is returned. This vague
message does not indicate what the constraint violations are, which is not comparable to the status messages of native
OCM Policy addon controllers.

### Goals

1. Gatekeeper `ConstraintTemplate` and constraint manifests can be embedded in OCM `Policy` object's `policy-templates`
   array.
2. Status of Gatekeeper constraints are aggregated in the OCM `Policy` objects on the OCM hub.

### Non-Goals

1. The installation of Gatekeeper will not be simplified in this enhancement.
2. When Gatekeeper denies a request, the OCM `Policy` status will be unaffected since the constraint is working as
   expected.

## Proposal

### Design

#### Policy Syntax

This proposes adding native support to the OCM Policy addon for Gatekeeper constraints. As shown in the example below, a
user would be able to directly embed Gatekeeper `ConstraintTemplate` (v1beta1 and v1 should both be supported) and
constraint manifests in the OCM `Policy` object's `policy-templates` array. Note that the `ConstraintTemplate` is not
required to be in the OCM `Policy`. It can be deployed in another way such as with a `ConfigurationPolicy` or with
ArgoCD.

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: gk-policy
spec:
  remediationAction: inform
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: templates.gatekeeper.sh/v1beta1
        kind: ConstraintTemplate
        metadata:
          name: k8srequiredlabels
        spec:
          crd:
            spec:
              names:
                kind: K8sRequiredLabels
              validation:
                openAPIV3Schema:
                  properties:
                    labels:
                      type: array
                      items: string
          targets:
            - target: admission.k8s.gatekeeper.sh
              rego: |
                package k8srequiredlabels

                violation[{"msg": msg, "details": {"missing_labels": missing}}] {
                  provided := {label | input.review.object.metadata.labels[label]}
                  required := {label | label := input.parameters.labels[_]}
                  missing := required - provided
                  count(missing) > 0
                  msg := sprintf("you must provide labels: %v", [missing])
                }
    - objectDefinition:
        apiVersion: constraints.gatekeeper.sh/v1beta1
        kind: K8sRequiredLabels
        metadata:
          name: ns-must-have-gk
        spec:
          enforcementAction: dryrun
          match:
            kinds:
              - apiGroups: [""]
                kinds: ["Namespace"]
          parameters:
            labels: ["gatekeeper"]
```

When the `Policy` has the `spec.remediationAction` field set to `inform`, it would override the constraint's
`spec.enforcementAction` with `warn`. This will not deny requests but instead, it will
[warn](https://open-policy-agent.github.io/gatekeeper/website/docs/violations#warn-enforcement-action) the user making
the request if their request violated the constraint. Existing and new violations would show up in the constraint's
`status.violations` section.

If the `remediationAction` field is set to `enforce`, the constraint's `spec.enforcementAction` would be overriden with
`deny`. Existing violations would show up in the constraint's `status.violations` section.

If the `remediationAction` field is not set, the constraint's `spec.enforcementAction` would not be modified.

#### Status Reporting

After applying a Gatekeeper constraint, Kubernetes objects on the cluster that violate the constraint will show up in
the constraint's `status.violations` field as shown in the example below.

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: ns-must-have-gk
spec:
  enforcementAction: dryrun
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
  parameters:
    labels: ["gatekeeper"]
status:
  auditTimestamp: "2019-08-15T01:46:13Z"
  enforced: true
  violations:
    - enforcementAction: dryrun
      kind: Namespace
      message: 'you must provide labels: {"gatekeeper"}'
      name: default
    - enforcementAction: dryrun
      kind: Namespace
      message: 'you must provide labels: {"gatekeeper"}'
      name: gatekeeper-system
```

The status message for the constraint in the OCM `Policy` would be in the format of
`<enforcementAction> - <message> (on <kind> <namespace/name or name>)` for each violation entry, each separated by a
semicolon. For example, the above example would produce a status message of
`dryrun - you must provide labels: {"gatekeeper"} (on Namespace default); dryrun - you must provide labels: {"gatekeeper"} (on Namespace gatekeeper-system)`.

If Gatekeeper is not installed, the OCM `Policy` will be marked as non-compliant with a message stating that the user
needs to install Gatekeeper to apply such a constraint.

If there are no violations, the OCM `Policy` status would have a compliant message of
`<spec.enforcementAction> - the Gatekeeper constraint has not detected any active violations`. For example,
`dryrun - the Gatekeeper constraint has not detected any active violations`.

#### Policy Deletion

When an OCM `Policy` is deleted that embeds a Gatekeeper `ConstraintTemplate`/constraint, the embeded Gatekeeper
`ConstraintTemplate`/constraint will be deleted to match the behavior of other OCM policy templates.

### User Stories

#### Story 1

As a policy user, I would like to have native multicluster support for an admission controller so that I can have a
consistent distribution and status aggregation mechanism for all my policies.

### Implementation Details/Notes/Constraints [optional]

#### Template Sync Updates

The Template Sync controller is a controller in the `governance-policy-framework-addon` controller manager responsible
for creating and updating policies on the managed cluster that are defined in the `Policy` object's `policy-templates`
array. So far, all these policies are namespace scoped. Gatekeeper `ConstraintTemplate` objects and constraint objects
are cluster scoped.

The changes required to support cluster scoped policy templates would mean that the Template Sync controller will need
to detect when a policy template is namespaced before querying or creating it. Additionally, owner references are
currently relied upon to properly delete the policy templates when the parent policy is deleted. Since namespaced
objects can't own cluster scoped objects in Kubernetes, an alternative is required.

The Template Sync controller would not put owner references on Gatekeeper `ConstraintTemplate`/constraints, but instead
add a finalizer of `policy.open-cluster-management.io/cleanup-cluster-scoped-policies` to the `Policy` object that
defines the Gatekeeper `ConstraintTemplate`/constraints. This finalizer would indicate that Gatekeeper
`ConstraintTemplate`/constraints need to be removed when the OCM `Policy` object is deleted and if applicable, when the
`Policy` object is updated. To be able to track which Gatekeeper constraints belong to a `Policy` object, a new label
should be added to the Gatekeeper `ConstraintTemplate` objects and constraints of
`policy.open-cluster-management.io/policy` with the policy name. Note that a finalizer will also need to be added to the
`governance-policy-framework` `Deployment` like the Configuration Policy controller does to prevent the Template Sync
from being deleted before the finalizer can be handled.

The Template Sync controller would also need to generate a status event when it creates or updates Gatekeeper
`ConstraintTemplate` objects since they are implementation details for the Gatekeeper constraints and don't provide any
validation on their own.

If the Gatekeeper `ConstraintTemplate` includes invalid rego syntax, a Gatekeeper validation webhook automatically
rejects the requst as shown below. When this occurs, the OCM `Policy` will be set to non-compliant and include the error
returned by the validation webhook.

```bash
$ kubectl apply -f bad-constraint-template.yaml
Error from server (1 error occurred: templates["admission.k8s.gatekeeper.sh"]["K8sRequiredLabels"]:7: rego_parse_error: unexpected eof token
          mis
            ^): error when creating "temp6.yaml": admission webhook "validation.gatekeeper.sh" denied the request: 1 error occurred: templates["admission.k8s.gatekeeper.sh"]["K8sRequiredLabels"]:7: rego_parse_error: unexpected eof token
          mis
            ^
```

#### New Controller

In the `governance-policy-framework-addon` controller manager, a new controller called `gatekeeper-constraint` would be
added. This is a bit tricky because each Gatekeeper `ConstraintTemplate` produces a new custom resource definition (CRD)
and thus the watches aren't known at controller instantiation time.

To handle this, the controller would watch all `Policy` objects in the managed cluster namespace. During the reconcile,
the controller will detect if Gatekeeper constraints are managed by the policy. If so, the
[dynamic watch](https://github.com/stolostron/kubernetes-dependency-watches) library will be used to watch the
individual Gatekeeper constraints and trigger reconciles on the `Policy` parent object whenever they have status
updates.

The new controller would also examine the `Policy` status before sending a status event to avoid sending duplicate
status events. It's possible that duplicate status events could be sent if the Status Sync controller is overwhelmed and
the Gatekeeper constraint has multiple reconciles in succession. This is considered acceptable because the Status Sync
controller filters out duplicate status events.

### Risks and Mitigation

### Open Questions [optional]

N/A

### Test Plan

**Note:** _Section not required until targeted at a release._

### Graduation Criteria

It would be GA in the release after implementation.

### Upgrade / Downgrade Strategy

There are no concerns with upgrades since this is not a breaking change and does not require user changes.

### Version Skew Strategy

## Implementation History

N/A

## Drawbacks

N/A

## Alternatives

N/A

## Infrastructure Needed [optional]

N/A
