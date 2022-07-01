# Removing Objects That Are Created by Policies Upon Policy Deletion

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

When the `remediationAction` of a policy is set to enforce and an object is referenced
in one of its `ConfigurationPolicy` objects that does not exist, the config policy
controller will create that object on the managed cluster. However, when the policy
is deleted, there is no logic to remove that object, so it will remain on the cluster
indefinitely until another policy is created to clean it up (this is how the majority
of our E2E tests handle clean-up) or the object is manually deleted via the CLI. We 
propose keeping track of the objects that are created by each `ConfigurationPolicy` in the status of the
`ConfigurationPolicy` and adding a new field to the `ConfigurationPolicy` spec that allows
users to specify whether they want to delete any objects created by the `ConfigurationPolicy`.

## Motivation

The current implementation, which leaves old objects on the cluster, can make it a
pain to remove policies and confusing for users if they delete a policy without
keeping track of the objects that policy created. There is no way for users to
know which objects are left over after a policy is deleted, so those objects
will persist on the cluster and have potential to cause confusion. Even if users
know which objects need to be cleaned up after deleting a policy, it is time-consuming
to do so. By allowing users to specify that they would like to automatically delete
managed objects, we would make policy clean-up significantly simpler and easier.

### Goals

1. Object deletion: objects that are created by policies will be deleted when
the policy is deleted
2. Objects that are not created by policies but are edited by them can be deleted
when the policy is deleted
3. Opt-in: add an optional new field to the `ConfigurationPolicy` spec and default
to leaving the object on the cluster upon policy deletion so that the behavior
of old policies is not changed

### Non-Goals

## Proposal

We would add a new field to the `ConfigurationPolicy` spec called
`spec.pruneObjectBehavior`. This field will be an enum with the options `DeleteAll`,
`DeleteIfCreated`, and `None`. When this field is set to `DeleteAll` and `remediationAction`
is set to `enforce`, the controller will automatically delete all objects
referenced in the `ConfigurationPolicy` when the `ConfigurationPolicy` is deleted.
For `DeleteIfCreated`, if `remediationAction` is `enforce`, the controller
will automatically delete any objects that it has created when the
`ConfigurationPolicy` is deleted and leave the objects that existed already when the
`ConfigurationPolicy` was created. When it is set to `None`, not set, or the `remediationAction`
of the policy is `inform`, the controller will leave any created objects intact
once the `ConfigurationPolicy` is deleted. This will preserve existing behavior
and make child object deletion “opt-in.”

### User Stories

As a k8s admin, I want objects that are created by a policy to be cleaned
up automatically when I delete the policy that created them.

### Implementation Details/Notes/Constraints 

`pruneObjectBehavior` will be at the spec level for the `ConfigurationPolicy`
and thus the value it is set to will apply to all object templates contained in
the `ConfigurationPolicy`. This will make it simpler for users to set than having
the field be set at the template level. If users would like some objects to
remain on deletion and others to be deleted, they can separate those objects
into multiple `ConfigurationPolicy` objects contained under the same parent policy.

`ConfigurationPolicy` objects already maintain a list of objects they manage in their
status field called `relatedObjects`. Since most of the information we would need
to track which objects need to be deleted already exists in this part of the status,
we propose adding a boolean field inside `status.relatedObjects.Object` that tells
the controller whether an object needs to be deleted when the `ConfigurationPolicy`
is deleted. We would set this value when the config policy controller creates
an object during the handling of an “enforce” policy template. 

Additionally, we will add the object's UID to the status field so that we can
determine whether the object is owned by the `ConfigurationPolicy` or not. It
will be used to verify the identity of the object to be deleted. Say a config
policy creates `pod/foo`. Then, between evaluation loops, something else deletes
and recreates that pod. In this case, if the behavior is `DeleteIfCreated`, we
don't want the policy to delete the pod. In some sense, the "responsibility"
for the pod has been passed to whatever else recreated it.

When processing `spec.pruneObjectBehavior`, we will set a finalizer on the `ConfigurationPolicy`,
such as `policy.open-cluster-management.io/delete-related-objects`. When the policy
is deleted, the controller will perform cleanup logic on the objects in `relatedObjects`
that have been marked for deletion by the new status field. Once that cleanup is complete,
the finalizer will be removed and kubernetes will handle removing the `ConfigurationPolicy`
from the cluster. Any objects without this value will persist on the cluster.

When deleting the objects from the cluster, we will set the status message of the
`ConfigurationPolicy` to indicate whether the deletion is in progress. That way, we can
provide a policy that monitors the status of all configuration policies on each managed
cluster and determines whether any of them have gotten stuck during their deletion. Users
will be able to check the status of this policy to see which `ConfigurationPolicy` has
failed to be deleted and try to remediate it.

Since `spec.pruneObjectBehavior` is scoped to the
`ConfigurationPolicy` level, the config policy controller will handle removing
objects on each managed cluster once a `ConfigurationPolicy` is deleted there. This also
means that when the placement of a policy changes to a different managed cluster
and that policy contains a `ConfigurationPolicy` with 
`pruneObjectBehavior: DeleteAll|DeleteIfCreated`, the object will be deleted on 
the old cluster and created on the new one. This means that
changing the placement of an `enforce` policy will no longer result in objects
persisting indefinitely on the old managed cluster.


### Risks and Mitigation

The main risk of adding an enhancement like this would be users deleting objects
that they want to remain on the cluster after they delete a policy. We plan to
mitigate this risk by making the `spec.pruneObjectBehavior` field default to
`None`, so users will not delete anything without having to set that field first.

## Design Details

The `spec.pruneObjectBehavior` field would look something like this:

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
  pruneObjectBehavior: DeleteIfCreated //controller will remove proposal-pod
```

Here is what the `ConfigurationPolicy` status would look like with the new field:

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
    properties:
      createdByPolicy: true //delete when parent configPolicy is deleted
      uid: 1234567890
    reason: Resource found as expected
```
### Scalability
The extra fields that will be needed to store data for this change should not impact scalability much, as we
would only add a few boolean fields to each `ConfigurationPolicy`. Deleting a policy will take a bit longer if
it is placed on a large amount of managed clusters, since the controller will now have to remove all created
objects from each managed cluster in addition to the `ConfigurationPolicy`. However, this would still be much
faster than it would take a user to clean up those objects otherwise - without this change, the easiest
way for users to clean up objects created by a policy would be to create another policy to delete the
objects after deleting the original policy, so this could end up helping with scalability because users
will not have to create as many policies on the hub cluster.

### Open Questions

### Test Plan

- Unit tests can be implemented on some of the helper functions that we add to the controller, but
will not be able to test whether objects are being removed properly.
- E2E/integration tests to ensure that objects are being removed when policies are deleted and
objects persist when the spec field is not set

### Graduation Criteria

#### GA

1. The proposal is reviewed and accepted
2. Implementation is completed to support the primary goals
3. Test cases developed to demonstrate that the user stories are fulfilled
4. E2E/Integration tests complete

### Upgrade / Downgrade Strategy

The `spec.pruneObjectBehavior` field being optional means that the controller will remain
backwards compatible with any older policies.

## Implementation History

N/A

## Drawbacks

- Users may accidentally delete objects if they are not familiar with how the new field works,
although they will not be able to do so unless they set the field without checking the documentation
on it.
- Since this will only remove objects that are specified in the policy yaml, objects created elsewhere
will not be deleted when the policy is deleted. This means the deletion will probably not work well
for removing operators, since they can create extra objects on the cluster that need to be cleaned
up.
- Currently, some policy tooling defaults to adding `clusterConditions` to the placementRule when
creating a policy that make the placement only apply to healthy clusters. This means `ConfigurationPolicy` objects
contained in the policy are deleted from a cluster that is unhealthy or hibernating, which would also
mean that with the changes detailed above, objects created by `ConfigurationPolicy` objects on that cluster
would be deleted when the cluster is not running properly. The best solution to this would be to
change the policy creation UI to not add that cluster condition by default and then document that
changing the placement of a policy will cause objects to be removed from the old cluster if 
`spec.pruneObjectBehavior` is set to `DeleteAll` or `DeleteIfCreated`.

## Infrastructure Needed

No new infrastructure is needed, since all the necessary changes can take place in the existing
config policy controller.
