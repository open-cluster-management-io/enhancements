# Policy Collection

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
      [website](https://github.com/open-cluster-management/website/)

## Summary

The goal of this feature is to automate and simplify deployment of policies, policy sets and
policy generator projects by creating (transferring) a repository that provides best practices, samples, blogs
and deployment resources.  The repository for this policy collection will be
[policy-collection](https://github.com/open-cluster-management-io/policy-collection/).  The content will be transferred 
from [existing policy-collection](https://github.com/stolostron/policy-collection/).

## Motivation

The governance policy framework is currently provided without any samples or guidance on
how to create policies other than by detailing the architecture of the 
[policy framework](https://open-cluster-management.io/getting-started/integration/policy-framework/).
While this is functional, it does not lend itself to easy adoption by new community users.

### Goals

- Transfer the policy collection community that provides samples, best practices and encourages quicker
  adoption of the policy framework.
- Make sure the transferred community is easy to consume by first time users of ocm policy.

### Non-Goals

- Because policies can be intended to work in many different environments with prerequisites that
  may not be available, validation of policies is focused on syntax initially and additional validation
  can be considered in the future.

## Proposal

### User Stories

1. As a k8s admin, I want to deploy policies from the community with an experience that fosters:
  - Best practice community policies and policy sets to achieve governance goals for common k8s platforms
  - Donations of policies from users that have forked the community repository and added extra value
    to existing or created new policies
  - Quick adoption of the policy framework through blogs and documentation that re-enforces concepts
    documented through the policy framework
  - Community adoption.  Validate documentation is centered around the open cluster management 
    community and not product specific.

### Architecture

The policy-collection does not add any architectural changes to the Policy Framework.  It provides
samples and resources that help community members adopt the Policy Framework and take advantage of the
features that are provided.  By showing the existing features, we also hope to make it easier for
community members to see where new features can be added or where existing enhancements can be made.

### Test Plan

- Basic policy validation will be added
- Policies can be tested by applying them to a cluster
- New users should be able to understand how to get started

### Graduation Criteria

#### Alpha

1. The proposal is reviewed and accepted
2. Implementation is completed to support the primary goals

#### Beta

1. Policy validation is performed
2. Stable policy process is defined
