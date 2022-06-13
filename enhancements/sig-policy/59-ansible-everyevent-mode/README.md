# Add an "everyEvent" Ansible Job Run Mode to the Policy addon

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
      [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

The Open Cluster Management (OCM) Policy addon Ansible integration should support a new run mode of `everyEvent` so that
an Ansible job can be run for every unique policy violation per managed cluster. This means that if you have two
noncompliant clusters that a policy applies to, the Ansible job would run only once for every cluster until the cluster
becomes compliant and then noncompliant again.

## Motivation

The Open Cluster Management (OCM) Policy addon supports running Ansible jobs on policy violations. The Policy addon
currently supports the run modes `once` and `disabled`. The `once` mode is limiting since after a policy violation
occurs, the Ansible job runs once and then it is disabled. A user must manually renable it for the Ansible job to run
again on the next policy violation.

An additional mode of `everyEvent` should be added so that an Ansible job can be run for every unique policy violation
per managed cluster. A simple example is that a user might use the OCM Policy addon Ansible integration to create
tickets in their ticketing system on every policy violation per cluster. The new `everyEvent` mode would enable this
use-case.

### Goals

1. The Policy addon is able to run an Ansible job on every unique policy violation per managed cluster.
1. Provide an option to avoid excessive Ansible job creation on an unstable policy (e.g. frequest compliant ->
   noncompliant -> compliant).

### Non-Goals

1. Keeping track of every historical `PolicyAutomation` run.
1. Adding additional `eventHook` options.
1. Avoiding infinite creation of Ansible jobs on unstable policies (e.g. frequest compliant -> noncompliant ->
   compliant).

## Proposal

### New Automation Mode

To configure Ansible automation when a policy violation occurs, a `PolicyAutomation` object is created which references
the triggering policy as shown below.

```yaml
apiVersion: policy.open-cluster-management.io/v1beta1
kind: PolicyAutomation
metadata:
  name: create-ticket
spec:
  mode: everyEvent
  policyRef: enable-etcd-encryption
  eventHook: noncompliant
  automationDef:
    name: Demo Job Template
    secret: ansible-automation-platform-secret-name
    extra_vars:
      sn_severity: 1
      sn_priority: 1
```

The proposal is to add a new mode of `everyEvent` that runs an Ansible job on every unique policy violation per managed
cluster. In order to achieve this, the status of the `PolicyAutomation` must be able to track which managed clusters the
Ansible job has run for already.

Take for example a scenario where a policy called `enable-etcd-encryption` applies to managed clusters `cluster1` and
`cluster2`. The user has also defined a `PolicyAutomation` object and set the `mode` to `everyEvent`. Then the
`enable-etcd-encryption` policy becomes noncompliant (i.e. violated) on `cluster1`. The Governance Policy Propagator
controller would create an Ansible job on the Hub with the `extra_var` of `target_clusters: ["cluster1"]` like when
using the `once` mode. If there were more noncompliant clusters at the time, then `target_clusters` would also contain
those as well. The controller would then set the following status on the `PolicyAutomation` object. The
`automationStartTime` represents the time the Ansible job was created and the `eventTime` represents the time of the
last time the cluster became noncompliant. Note that there is currently no status that ever gets set on
`PolicyAutomation` objects.

```yaml
status:
  clustersWithEvent:
    cluster1:
      automationStartTime: "2022-06-22T23:59:30Z"
      eventTime: "2022-06-22T23:59:29Z"
```

This status update would tell the Governance Policy Propagator controller to not run the Ansible job again on `cluster1`
the next time it notices that the `enable-etcd-encryption` policy is noncompliant on `cluster1` such as during periodic
reconciling or status updates. This isn't meant to store a complete history of which clusters had an Ansible job run due
to a policy violation. It is meant to track the clusters **currently** in the state based on the `eventHook` value in
the `PolicyAutomation` object and the appropriate automation was run for those clusters. Such a history can be found on
the Ansible Tower/Ansible Automation Platform job history.

Several minutes later, the `enable-etcd-encryption` policy becomes noncompliant on `cluster2`. The Governance Policy
Propagator controller would run the Ansible job from the Hub while passing the `extra_var` of
`target_clusters: ["cluster2"]`. It would then set the following status on the `PolicyAutomation` object.

```yaml
status:
  clustersWithEvent:
    cluster1:
      automationStartTime: "2022-06-22T23:58:30Z"
      eventTime: "2022-06-22T23:58:29Z"
    cluster2:
      automationStartTime: "2022-06-22T23:59:30Z"
      eventTime: "2022-06-22T23:59:29Z"
```

Several minutes later, the `enable-etcd-encryption` policy becomes compliant on `cluster1`. The Governance Policy
Propagator controller would not run an Ansible job and set the following status on the `PolicyAutomation` object.

```yaml
status:
  clustersWithEvent:
    cluster2:
      automationStartTime: "2022-06-22T23:59:30Z"
      eventTime: "2022-06-22T23:59:29Z"
```

Now consider the case where the user made a change to their Ansible job and wants the Governance Policy Propagator
controller to rerun the Ansible jobs for all current noncompliant managed clusters. The user would set the existing
annotation of `policy.open-cluster-management.io/rerun` to `true` on the `PolicyAutomation` object and thus signal the
Governance Policy Propagator controller to ignore `status.clustersWithEvent` and run an Ansible job for every
noncompliant managed cluster.

### Limiting the Number of Ansible Jobs

If a user has a policy that they are concerned with often going between compliant and non-compliant, they may not want
to run an Ansible job every time the cluster becomes noncompliant in order to not overwhelm Ansible Tower/AAP. In this
case, a new optional spec field of `delayAfterRunSeconds` (follows the
[API naming conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#naming-conventions))
would be added so the user could specify the minimum amount of seconds before an automation can be retriggered on the
same cluster. This would default to 0 seconds and only be applicable for the `mode` of `everyEvent`.

If a user sets `delayAfterRunSeconds` to `600` for example, and the cluster becomes noncompliant a second time and stays
noncompliant past the 600 seconds, the Ansible job would be run again for that cluster a maximum of one time. This is
detected by looking at the `eventTime` in the `status.clustersWithEvent` entry to see if the timestamp is newer than the
`automationStartTime` value. When the `delayAfterRunSeconds` value is set, the `PolicyAutomation` controller would not
remove the `status.clustersWithEvent` entry for the cluster until it has become compliant and the number of seconds set
in `delayAfterRunSeconds` have elapsed since the `automationStartTime` value. From an implementation point of view, this
would mean requeing the reconcile request to that point in time.

### User Stories

#### Story 1

As an OCM Policy addon user, I would like to run an Ansible job for every unique policy violation per managed cluster so
that I can guarantee that my Ansible automation is run for every managed cluster.

#### Story 2

As a Kubernetes administrator, I would like to use Ansible to create a ServiceNow ticket for each managed cluster that
my OCM policy is noncompliant on.

### Implementation Details/Notes/Constraints [optional]

N/A

### Risks and Mitigation

N/A

## Design Details

See the [Proposal](#proposal) section.

### Open Questions [optional]

N/A

### Test Plan

**Note:** _Section not required until targeted at a release._

Consider the following in developing a test plan for this enhancement:

- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything that would count as tricky in the
implementation and anything particularly challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage expectations).

All code is expected to have sufficient e2e tests.

### Graduation Criteria

The `PolicyAutomation` kind is still `v1beta1` and it would remain so after this enhancement.

### Upgrade / Downgrade Strategy

There are no concerns with upgrades since this is not a breaking change.

Downgrading would lead to the `PolicyAutomation` object influencing nothing on policy violations because the `mode` of
`everyEvent` would be considered invalid. This is an acceptable outcome.

### Version Skew Strategy

This is not applicable because the change is only on the Hub side.

## Implementation History

N/A

## Drawbacks

1. This does not track the total number of Ansible jobs that were created.

## Alternatives

N/A

## Infrastructure Needed [optional]

N/A
