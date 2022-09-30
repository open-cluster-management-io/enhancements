# Guarantee the Ordering of Policy Template Processing

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

Multiple policy templates are often wrapped in one policy, but all this does is create
all of the policy templates at the same time. In any use case where a policy template 
depends on another policy being processed before it, the only way to guarantee the policy
templates are processed in the correct order is to split them into separate policies and
create them in the desired order. We propose allowing users to specify conditions that
must be met in order to evaluate a policy template, which would allow the governance
framework to automatically handle cases where policy templates need to be processed in
order.

## Motivation

Currently, users have to split up policy templates that depend on others into multiple
policies and manually create them in order. This is more resource intensive as users
can have to create duplicate policies and more time consuming if users have to manually
manage policy processing order. If a user elects to group the templates in one policy,
there is no guarantee the templates will be processed in order, which could leave
the policy in a bad state if a required resource has not yet been processed. For example,
when using a policy to install an operator, if some objects are created before others it
can cause the operator to not be installed properly. Allowing users to specify policy
dependencies will alleviate these issues.

### Goals

1. Policy template dependencies: allow users to automatically process policy templates
in a desired order and guarantee that this order will be stable
2. Opt-in: any extra fields that are used to specify policy ordering should be optional,
and policies that do not contain these fields should function as they did before in order
to avoid issues with older policies

### Non-Goals

1. Status message matching: we plan to only allow users to conditionally process policy
templates based on compliance state, so users will not be able to specify that they want a
policy to be processed if another policy has a specific status message

## Proposal

We would add an optional new `dependencies` field to the spec on the configuration 
policy level as well as an `extraDependencies` field on the template level to
allow users to specify conditions for policy processing on both the policy and
template level. When these fields are set and their conditions are not met, 
the policy templates will not be applied on the managed cluster and the affected
policy will enter a "pending" state. If this happens, any policy sets that contain
the pending policy will also be marked as pending. When the conditions are met or
the fields are not set, the policy templates will be processed normally.

### User Stories

1. As an author of a ConfigurationPolicy, I don’t want resources to be wasted when
I know one object depends on another. If an object cannot be cleanly applied before
another is created, I want to specify this in my Policy, and not waste
resources (CPU, k8s API requests) in the policy controllers.
2. As an author of a ConfigurationPolicy, I want to specify the order resources are
applied in a policy when the API is not declarative. For some APIs (eg OLM), if
resources are applied in the wrong order they can end up in an unhealthy state
that I want to avoid.
3. As a user of PolicySets, I want policies inside a set to be applied in a
certain order.
4. As an author of many ConfigurationPolicies, I have several policies that
depend on another policy. I want to be able to reference completely separate
policies (outside of the Policy “wrapper”) as required before a configuration
is applied. (Maybe allow a policy to reference the status of a policy that runs
before it (success or failure) before it tries to process it. Allowing you to chain
them together)
5. As a cluster administrator, I want to do a declarative action on a managed
cluster when a policy becomes non-compliant. So I want to “activate” a
ConfigurationPolicy based on the state of another policy in the cluster.
6. As a user of the policy generator, I want to be able to generate policies that
will have their templates be processed in a certain order.

### Implementation Details/Notes/Constraints 

We will leverage the new dynamic watcher library (enhancement 71) to watch all
policy template dependencies. When a policy is created with a dependency, we
can start to watch the policy object that it is dependent on. If the compliance
of the dependency matches up with what the policy wants, we will process it
normally in the template sync. If not, we will assign a `Pending` state to the
policy compliance, and the watcher library will notify us when the dependency
changes, so we can re-evaluate the decision. This logic will be added into the
new combined governance-policy-framework-addon repository.

These dependencies will be able to accept any object that has a
`status.complianceState` field - policies, policy sets and, all other policy
types. If an object in `dependencies` or `extraDependencies` does not contain
that status field, the policy will remain in a pending state until the status
field appears with the correct value, but we will notify the user that
`status.complianceState` is missing in the policy status message.

We plan on writing info about templates that fail the dependency check, such
as what objects the template is waiting on, to the policy status in addition
to setting the compliance state for that policy to `Pending`. In the UI, we
will need logic to remove the "view details" link when a template is pending,
since the policy object for a pending template will not exist on the cluster.

### Risks and Mitigation

- Performance hit due to having to watch extra resources (utilizing the watch
library should help with this)
- Policies that are dependent on another policy status to be processed will
take longer to show status than normal policies. This shouldn't be a big issue
most of the time, but if the config policy controller is at its saturation point
(500+ configuration policies) it could take over a minute for the dependent policy
status to update correctly
- Circular dependencies: it is possible for users to create a loop where policy A
has a dependency on policy B, and policy B has a dependency on policy A so they
will both get stuck in a pending state. We don't plan to check for this since it
is user error and we expect the status message accompanying a pending policy
to be detailed enough that a user who has created a circular dependency would
be able to trace their chain of dependencies to find the issue.

## Design Details

Here is what the new spec field will look like in a policy. The top-level
`dependencies` field applies to all policy templates in the policy, while
the template-level `extraDependencies` field can be used to set up
dependencies for a specific policy template. The top-level `dependencies`
will take priority over the template-level `extraDependencies`, so policy
templates that have `extraDependencies` satisfied but not `dependencies`
will still be marked as `pending`. This means that the dependencies for
a policy will be the union of its `dependencies` and `extraDependencies`
fields.

```
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: my-complicated-policy
spec:
  dependencies: //for all policy templates
    - kind: Policy
      name: namespace-foo-setup-policy
      compliance: Compliant
  policy-templates:
    - extraDependencies: //for this policy template only
        - kind: ConfigurationPolicy
          name: bar
          compliance: NonCompliant
      objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: fix-bar-in-foo
        spec:
          remediationAction: enforce
          namespaceSelector:
            include: ["foo"]
          object-templates:
            ...
```

When the dependencies for a policy template to be evaluated are not met, we plan to add logic
in the template sync portion of the policy framework addon that will ignore those templates
until they are ready. Putting this logic in the template sync ensures that whether a policy
template is evaluated will be determined on the managed cluster level rather than the hub
level, which is ideal because policy templates on different managed clusters may need to be
processed differently based on the state of each managed cluster.

If a policy template meets its dependencies and gets processed properly, but at some point
afterward the state of the cluster changes and those dependencies are no longer met, we do
not want the template to be active anymore. Utilizing the watch library should allow us to
follow the dependencies for each template and put the template back into a pending state
when this happens. When the template is put into a pending state, the configuration policy
will be deleted, so if a dependency is no longer satisfied, any objects created
on the cluster as a result of that dependency will be deleted.

### Open Questions

Is setting an `apiGroup` field in `dependencies/extraDependencies` necessary? Currently we should
not have any issues with conficting `kind` for policy templates, but in theory users could create
a custom policy controller for a kind that has a name conflict with one of the build-in policy
templates. This isn't likely to happen, but if we think the `apiGroup` might be necessary in the
future, it's probably better to add it now rather than later.
- Update: We do not plan to add `apiGroup` now as it can be added in a backwards-compatible way
in the future if we feel it is needed.

### Test Plan

- Unit tests can be implemented to determine whether the logic that enables/disables policy
templates based on their dependencies is working properly
- E2E/integration tests will be necessary to ensure that synced objects are being managed
properly and policy status is what it should be when dependencies are met/not met.

### Graduation Criteria

#### Alpha

1. The proposal is reviewed and accepted
2. Implementation is completed to support the primary goals
3. Test cases developed to demonstrate that the user stories are fulfilled

#### Beta

1. E2E/Integration tests complete

### Upgrade / Downgrade Strategy

The `spec.dependencies` and `spec.policy-templates[].extraDependencies` fields being optional
means that the controller will remain backwards compatible with any older policies.

## Implementation History

N/A

## Drawbacks

- Adding two large fields to the spec could make configuring policies feel a bit more clunky
for users
- This will add a lot of complexity to the template sync, which has not needed to be updated
much in the past due to it not having to manage much logic regarding policy processing

## Infrastructure Needed

No new infrastructure is needed, since all the necessary changes can take place in the combined
governance framework addon, and we will leverage the watch library that is already going to
be created for another epic.
