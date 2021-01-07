
## Overview

The `enhancements` repo is made available to discuss specific new features or architectural improvements for the community around `open-cluster-management`.

Inspired by the [Kubernetes enhancement](https://github.com/kubernetes/enhancements) process.

This repository provides a rally point to discuss, debate, and reach consensus for how Open Cluster Management enhancements are introduced.

Enhancements may take multiple releases to ultimately complete and thus provide the basis of a community roadmap. Enhancements may be filed from anyone in the community, but require consensus from domain specific project maintainers in order to implement and accept into the release.

For template references, see the [template](guidelines/README.md).

## Is My Thing an Enhancement?

A rough heuristic for an enhancement is anything that:

- a blog post would be written about after its release
- needs significant effort or changes in a significant way (ex. something that would take 10 person-weeks to implement, introduce or redesign a system component, or introduces API changes)
- impacts upgrade/downgrade 
- requires consensus/code across multiple domains/repositories
- will be graduating from one stage to another (ex. alpha to beta, beta to GA)
- users will notice and come to rely on

It is unlikely to require an enhancement if it:

- fixes a bug
- adds more testing
- internally refactors a code or component only visible to that components
  domain
- minimal impact to distribution as a whole

If you are not sure if the proposed work requires an enhancement, file an issue
and ask!

## When to Create a New Enhancement

Create an enhancement here once you:

- have circulated your idea to see if there is interest
- (optionally) have done a prototype in your own fork
- have identified people who agree to work on and maintain the enhancement
  - many enhancements will take several releases to complete  

## Why are Enhancements Tracked

As the project evolves, its important that the open-cluster-management community understands how we
build, test, and document our work.  Individually it is hard to understand how
all parts of the system interact, but as a community we can lean on each other
to build the right design and approach before getting too deep into an
implementation.

## When to Comment on an Enhancement Issue

Please comment on the enhancement issue to:
- request a review or clarification on the process
- update status of the enhancement effort
- link to relevant issues in other repos

Please do not comment on the enhancement issue to:
- discuss a detail of the design, code or docs. Use a linked-to-issue or design PR
  for that