# Providing Context to the Ansible Automation for a Policy Violation 

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
 [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

The Open Cluster Management (OCM) Policy add-on Ansible integration should pass the additional information 
into an Ansible job when triggering the creation. It means that the Ansible user can get more policy violation 
information and use them in the Ansible playbook.

## Motivation

Policy Automation uses the `policyRef` field to match the target policy and monitor its non-compliance status. 
A non-compliance policy can trigger its Policy Automation to create an Ansible job on the Ansible Tower, which 
launches a predefined Ansible playbook against an inventory of hosts. During this creation, we can pass the 
additional information to the Ansible job as `extra_vars`.

The current limitation is that we only pass the non-compliant cluster as `extra_vars`, which isn't enough to 
describe the policy violation status. The following fields could be added:
1. The violated policy name
2. The namespace of the root policy
3. The hub cluster name
4. The policy set name
5. The policy violation message
6. The replicated policy status for each non-compliant cluster


### Goals

Adding the name of the violated policy, the root policy namespace, the hub cluster name, the policy_set name, 
the violation message, and the replicated policy status for each non-compliant cluster into the Ansible Job.

### Non-Goals

- None at this time

## Proposal
Policy automation runs on the hub cluster. It should not get the propagated policy status from the managed cluster,
which requires Search or Observability dependency. The same design also applies to the hub of the cluster scenario.

### New Content Structure

Current `extra_vars` has a field `target_clusters` contains non-compliant cluster name list:
```json
{
  "target_clusters": [
    "cluster1", 
    "cluster2", 
    "cluster3",
  ]
}
```

The new content structure will pass more information:
```json
{
  "target_clusters": [
  "cluster1", 
  "cluster2", 
  "cluster3",
  ],
  "policy_name": "policy1",
  "namespace": "policy-namespace",
  "hub_cluster": "acm-hub-cluster",
  "policy_set": [
    "policyset-1",
    "policyset-2",
  ]
  "violation_message": "NonCompliant; violation - xxxxxx",
  "policy_violation_context" : {
    "cluster1": {
      "compliant": "NonCompliant",
      "details": [
        {
          "compliant": "NonCompliant",
          "history": [
            {
            "lastTimestamp": "2022-09-08T16:03:23Z",
            "message": "NonCompliant; violation - pods not found: [testpod] in namespace default missing"
            },
          ],
        },
      ]
    },
    "cluster2": {
      ...
    },
  },
}
```

- `policy_name` is the name of the non-compliance policy that triggers the ansible job.
 `violation_message` is the latest policy violation message. 
 They are from the root policy on the hub cluster. 
 The field path of `policy_name` is `policy.metadata.name`.
 The field path of `violation_message` is `policy.policyStatus.details[0].history[0].message`

- `namespace` is the namespace of the root policy, which is from `rootPolicy.GetNamespace()`

- `hub_cluster` is the name of the hub cluster, which is from `rootPolicy.GetClusterName()`

- `policy_set` contains all associated policy set names of the root policy, which are from `metadata.name`
  under `kind: PolicySet`. If the policy doesn't belong to any policy set, `policy_set` will be empty.

- `policy_violation_context` is a map. The keys are non-compliant cluster names, and the value is the raw replicated 
 policy status for each non-compliant cluster.
 The policy automation looks at the `cluster namespaces` of the hub cluster and gets the replicated policies. 
 Then it filters out the non-compliant replicated policies and puts the status into the `policy_violation_context` map.

 Suppose the policy automation runs on the hub cluster which is self-managed. In that case, `policy_violation_context` 
 would be a map that only contains `local-cluster`.
 
 The field path of the raw status is `policy.policyStatus`. The below unnecessary fields in `policy.policyStatus` will be
 skipped and not in `policy_violation_context`:
 1. `policy.policyStatus.details[n].templateMeta`.
 2. `policy.policyStatus.details[n].history[n].eventName`

### User Stories

#### Story 1

As an OCM Policy add-on user, I would like to get the additional status in an Ansible job's `extra_vars` field. For example,
regarding the CertificatePolicy policy, which checks for expiring certificates in secrets, I want to get the replicated policy 
status for each non-compliant cluster so I can know which secret expires on a specific cluster in the Anisble Job. Then, the 
Ansible Automation Platform can create a notice ticket and provide the violation details. 

### Implementation Details/Notes/Constraints [optional]

N/A

### Risks and Mitigation

Since `policy_violation_context` is the replicated policy status for all non-compliant managed clusters, the length of 
`policy_violation_context` may be oversized when the amount of managed clusters is large.

To solve this, we will only pass the top `N` replicated policy status into `policy_violation_context`, where `N` is configurable
via an optional policy automation field called `policyViolationContextLimit`. If this field is not set, `N` will default to the
first 1000 managed clusters.

## Design Details

See the [Proposal](#proposal) section.

### Open Questions [optional]

N/A

### Test Plan

- Unit tests can be implemented to check the created Ansible job. For each scenario, the `extra_vars` contain the designed data.

### Graduation Criteria

1. The proposal is reviewed and accepted
2. Implementation is completed to support the primary goals
3. Test cases developed to demonstrate that the user stories are fulfilled

### Upgrade / Downgrade Strategy

There are no concerns with upgrades since this is not a breaking change.

Downgrading would only pass `target_clusters` to Ansible Job, which is an acceptable outcome.

### Version Skew Strategy

N/A

## Implementation History

N/A

## Drawbacks

1. This design does not restructure `policy_violation_context` data for different policies.

## Alternatives

Add the additional replicated policy status parse logic in [`governance-policy-propagator`](https://github.com/open-cluster-management-io/governance-policy-propagator) 
for each kind of policy. Policies can have different '`policy_violation_context` data structures and only display the needed information.

- Advantages - Reduce the `policy_violation_context` size. The data will be more user-friendly.
- Disadvantages - We need to add a new parse code for a new kind of policy and update the existing parse code if the new 
fields are wanted. If the policy kind isn't in the existing parse codes, policy automation will not pass any replicated 
policy status or pass the raw replicated policy status.

## Infrastructure Needed [optional]

No new infrastructure is needed since all the necessary changes can take place in the existing
`governance-policy-propagator`](https://github.com/open-cluster-management-io/governance-policy-propagator).
