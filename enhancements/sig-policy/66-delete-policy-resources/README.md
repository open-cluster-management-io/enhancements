# Removing Objects That Are Created by Policies Upon Policy Deletion

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

When the remediationAction of a policy is set to enforce and an object is referenced
in one of its configurationPolicies that does not exist, the configurationPolicy
controller will create that object on the managed cluster. However, when the policy
is deleted, there is no logic to remove that object, so it will remain on the cluster
indefinitely until another policy is created to clean it up (this is how the majority
of our E2E tests handle clean-up) or the object is manually deleted via the CLI. We 
propose keeping track of the objects that are created by each configurationPolicy in the status of the
configurationPolicy and adding a new field to the configurationPolicy spec that allows
users to specify whether they want to delete any objects created by the configurationPolicy.

## Motivation

The current implementation, which leaves old resources on the cluster, can make it a
pain to remove policies and confusing for users if they delete a policy without
keeping track of the objects that policy created. There is no way for users to
know which resources are left over after a policy is deleted, so those resources
will persist on the cluster and have potential to cause confusion. Even if users
know which resources need to be cleaned up after deleting a policy, it is time-consuming
to do so. By allowing users to specify that they would like to automatically delete
managed resources, we would make policy clean-up significantly simpler and easier.

### Goals

1. Object deletion: objects that are created by policies will be deleted when
the policy is deleted
2. Opt-in: add an optional new field to the configurationPolicy spec and default
to leaving the object on the cluster upon policy deletion so that the behavior
of old policies is not changed

### Non-Goals

1. Deleting objects not created by the parent policy: we will not support
deleting objects that are not directly created by a configurationPolicy when
that configurationPolicy is deleted (i.e. existing resources that are edited
by the configurationPolicy). We determined implementing this would create
more problems than it would solve, such as users accidentally deleting objects
that are created by the cluster or other policies.

## Proposal

We would add a new boolean field to the configurationPolicy spec called
`spec.removeChildrenOnDelete`. When this field is set to true, the controller
will automatically delete any objects that it has created when the
configurationPolicy is deleted. When it is set to false or not set, the
controller will leave any created objects intact once the configurationPolicy
is deleted. This will preserve existing behavior and make child object deletion
“opt-in.”

### User Stories

As a k8s admin, I want resources that are created by a policy to be cleaned
up automatically when I delete the policy that created them.

### Implementation Details/Notes/Constraints 

Although kubebuilder will automatically pull API validation from embedded
structs (like when types embed `metav1.TypeMeta`), it will not automatically
pull RBAC permissions on methods into the project. So RBAC requirements on
common functions (eg `get namespaces` for selecting namespaces) will need to
be well-documented.

This module can include features that aren't strictly required or currently used
by the policy framework. For example, storing additional structured information
about k8s objects that the policy looks at in a `status.relatedObjects` field.
Experimental features can be added, but should be marked as such by the
versioning of the repository.

### Risks and Mitigation

The main risk of adding an enhancement like this would be users deleting objects
that they want to remain on the cluster after they delete a policy. We plan to
mitigate this risk by making the `spec.removeChildrenOnDelete` field default to
`false`, so users will not delete anything without having to set that field first.

## Design Details

The `spec.removeChildrenOnDelete` field would look something like this:

```
spec:
  namespaceSelector:
    exclude:
    - kube-*
    include:
    - default
  object-templates:
  - complianceType: musthave
    objectDefinition:
      apiVersion: v1
      kind: Pod
      metadata:
        name: proposal-pod
      spec:
        containers:
        - image: nginx:1.18.0
          name: nginx
          ports:
          - containerPort: 80
  remediationAction: enforce
  severity: low
  removeChildrenOnDelete: true //controller will remove proposal-pod
```

`removeChildrenOnDelete` will be at the spec level for the configurationPolicy and thus the
value it is set to will apply to all policy templates contained in the configurationPolicy.
This will make it simpler for users to set than having the field be set at the template level.
If users would like some objects to remain on deletion and others to be deleted, they can
separate those objects into multiple configurationPolicies contained under the same parent policy.

ConfigurationPolicies already maintain a list of objects they manage in their status field called
`relatedObjects`. Since most of the information we would need to track which objects need to be
deleted already exists in this part of the status, we propose adding a boolean field inside
`status.relatedObjects.Object` that tells the controller whether a resource needs to be deleted
when the configurationPolicy is deleted. We would set this value when the configurationPolicy
controller creates a resource during the handling of an “enforce” policy template. When processing
`spec.removeChildrenOnDelete`, we will set a finalizer on the configurationPolicy, such as
`policy.open-cluster-management.io/delete-related-objects`. When the policy is deleted, the
controller will perform cleanup logic on the objects in `relatedObjects` that have been marked
for deletion by the new status field. Once that cleanup is complete, the finalizer will be removed
and kubernetes will handle removing the configurationPolicy from the cluster. Any resources without
this value will persist on the cluster.

Here is what the configurationPolicy status would look like with the new field:

```
status:
  compliancyDetails:
  - Compliant: Compliant
    Validity: {}
    conditions:
    - lastTransitionTime: "2022-06-09T19:27:13Z"
      message: pods [demo] in namespace default found as specified, therefore this Object template is compliant
      reason: K8s `must have` object already exists
      status: "True"
      type: notification
  compliant: Compliant
  lastEvaluated: "2022-06-09T19:27:53Z"
  lastEvaluatedGeneration: 1
  relatedObjects:
  - compliant: Compliant
    object:
      apiVersion: v1
      kind: pods
      metadata:
        name: demo
        namespace: default
      createdByParent: true //delete when parent configPolicy is deleted
    reason: Resource found as expected
```

### Open Questions

### Test Plan

- Unit tests can be implemented on some of the helper functions that we add to the controller, but
will not be able to test whether resources are being removed properly.
- E2E/integration tests to ensure that resources are being removed when policies are deleted and
resources persist when the spec field is not set

### Graduation Criteria

#### Alpha

1. The proposal is reviewed and accepted
2. Implementation is completed to support the primary goals
3. Test cases developed to demonstrate that the user stories are fulfilled

#### Beta

1. E2E/Integration tests complete

### Upgrade / Downgrade Strategy

The `spec.removeChildrenOnDelete` field being optional means that the controller will remain
backwards compatible with any older policies.

## Implementation History

N/A

## Drawbacks

- Users may accidentally delete resources if they are not familiar with how the new field works,
although they will not be able to do so unless they set the field without checking the documentation
on it.
- Since this will only remove objects that are specified in the policy yaml, objects created elsewhere
will not be deleted when the policy is deleted. This means the deletion will probably not work well
for removing operators, since they can create extra resources on the cluster that need to be cleaned
up.

## Infrastructure Needed

No new infrastructure is needed, since all the necessary changes can take place in the existing
configurationPolicy controller.
