## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

OCM's use of a CSR based mechanism for registering spoke clusters with the hub cluster is incompatible with Kubernetes environments that cannot issue client auth certificates such as Amazon Elastic Kubernetes Service (EKS).
This enhancement provides a secondary Service Account Token based registration mechanism that is universally supported, and can be used as an alternative CSR in such environments. 

## Motivation

For cluster administrators, it is often preferable to leverage their cloud provider’s managed Kubernetes service (e.g. AKS, EKS, IKS, GKE etc) rather than self-managing the Kubernetes control plane and worker nodes - reducing cluster management overhead and complexity.
However, in some of these environments (for example EKS), the use of CSRs for client authentication is not permissible.

Adopting OCM should not require its users to change how they deploy and manage their hub Kubernetes cluster.

As such, OCM needs to support an alternative registration mechanism to CSRs that is expected to work in all environments, providing maximum compatibility with managed Kubernetes services or restricted/locked down environments. 

### Goals

- The OCM hub cluster can run on EKS and other environments where CSR registration cannot be used.
- `clusteradm` tooling provides options to select token based registration when joining a worker to the hub.
- Suitable warnings/error messages/conditions are added to OCM such that it is easily identifiable when CSRs cannot be used.

### Non-Goals

- Support for leveraging native cloud specific IAM providers or external credential providers such as [Vault](https://www.vaultproject.io/) in the registration process (which will be raised as a separate enhancement)

## Proposal

### User Stories

#### Story 1 - Join spoke to hub cluster using Service Account Token

It must be possible for the cluster administrator to specify they wish to use `token` registration in the `clusteradm join` command:

```shell
% clusteradm join \
     --registration=token \
     --hub-token XXX \
     --hub-apiserver https://spoke-0.k8s.example.com \
     --cluster-name spoke-0
```

#### Story 2 - Default registration continues to utilize CSRs

When not specified, we should continue to default to the `csr` registration mechanism. 

```shell
% clusteradm join \
     --hub-token XXX \
     --hub-apiserver https://spoke-0.k8s.example.com \
     --cluster-name spoke-0 # default to csr registration
```

This can also be explicitly set, by setting `--registration=csr` :

```shell
% clusteradm join \
     --registration=csr \
     --hub-token XXX \
     --hub-apiserver https://spoke-0.k8s.example.com \
     --cluster-name spoke-0
```

#### Story 3 - Tokens are refreshed automatically prior to expiry

#### Story 4 - It should be possible to mix registration types within a single hub cluster

In order to future-proof OCM for additional registration types, it should be possible for a spoke cluster's to use different registration types for each worker cluster.

#### Story 4a - Spoke administrator should be able to list available registration types, and should receive an error message when using an unavailable type

Ideally it should be possible to list available registration types:

```shell
% clusteradm join \
     --hub-token XXX \
     --hub-apiserver https://spoke-0.k8s.example.com \
     --list-registration-types

Hub cluster supports the following registration types: ['token']
```

#### Story 4b - Spoke administrator should receive an error message when using an unavailable type

If the spoke administrator attempts to join the hub with an unsupported registration type, a self explanatory error should be returned:

```shell
% clusteradm join \
     --registration=csr \
     --hub-token XXX \
     --hub-apiserver https://spoke-0.k8s.example.com \
     --cluster-name spoke-0

[ERROR] 'csr' registration is unavailable - available registration types are ['token']
```

#### Story 4c - Hub administrator can configure enabled registration types

Hub administrator should be able to determine the supported registration types in their environment, and configure OCM appropriately.

To enable both `csr` and `token` the following would be used:

```shell
% clusteradm init --registration-type csr --registration-type token 
```

The default would remain only `csr`:

```shell
% clusteradm init
```

#### Story 5 - Hub administrator can enable token registration on an existing hub cluster

TBD

### Implementation Details/Notes/Constraints [optional]

### Risks and Mitigation

#### Protecting the spoke cluster's service account token

TODO

## Design Details

### Open Questions [optional]

This is where to call out areas of the design that require closure before deciding
to implement the design.  For instance,
> 1. This requires exposing previously private resources which contain sensitive
     information.  Can we do this?

### Test Plan

**Note:** *Section not required until targeted at a release.*

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

### Upgrade / Downgrade Strategy

### Version Skew Strategy

## Implementation History

## Drawbacks

## Alternatives

- Cloud provider native IAM support - this will be covered in a new enhancement.

## Infrastructure Needed [optional]

No specific infrastructure required. 