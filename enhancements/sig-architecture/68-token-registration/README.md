## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

OCM's use of a CSR based mechanism for registering spoke clusters with the hub cluster is incompatible with Kubernetes environments that cannot issue client auth certificates such as Amazon Elastic Kubernetes Service (EKS).
This enhancement provides a secondary Service Account Token based registration mechanism that is universally supported, and can be used as an alternative to CSR in such environments. 

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

#### Story 1 - Spoke administrator joins a cluster to the hub using Service Account Token

It must be possible for the cluster administrator to specify they wish to use `token` registration in the `clusteradm join` command:

```
% clusteradm join \
     --registration=token \
     --hub-token XXX \
     --hub-apiserver https://spoke-0.k8s.example.com \
     --cluster-name spoke-0
```

#### Story 2 - Spoke administrator joins a cluster to the hub using the default (csr) mechanism

When not specified, we should continue to default to the `csr` registration mechanism. 

```
% clusteradm join \
     --hub-token XXX \
     --hub-apiserver https://spoke-0.k8s.example.com \
     --cluster-name spoke-0 # default to csr registration
```

This can also be explicitly set to `csr`, using `--registration=csr` :

```
% clusteradm join \
     --registration=csr \
     --hub-token XXX \
     --hub-apiserver https://spoke-0.k8s.example.com \
     --cluster-name spoke-0
```

#### Story 3 - Hub administrator accepts a spoke cluster using csr or token registration in the same way

From a hub administrator point of viw, the existing `clusteradm accept` command will continue to work, regardless of whether the spoke cluster is using csr or token registration.

```
% clusteradm accept --clusters spoke-0 # No additional options required if spoke-0 used token registration
```

#### Story 3 - Spoke Service Account Tokens are refreshed automatically prior to expiry

When the service account token is nearing expiry, the spoke cluster should retrieve a replacement token from the hub, without administrator intervention. 

#### Story 4 - Hub administrator can accept both csr and token registrations on a single hub cluster

In order to future-proof OCM for additional registration types (e.g. cloud provider IAM), it must be possible for a hub to support spoke clusters using both csr and token registration.

#### Story 5 - Hub administrator unjoining a spoke cluster, results in the associated cluster service account being deleted

When a spoke cluster is unjoined, it must no longer be possible to authenticate with the hub using the spoke's Service Account token

#### Story 6 - Procedure for administrators to follow should a token expire or be deleted and the spoke cluster was unable to refresh the token

There needs to be a procedure to follow covering scenarios where the spoke cluster is unable to refresh its service account token, and has lost its ability to authenticate with the hub.
It should be possible for administrators to restore functionality to the spoke cluster.

Some example scenarios:

1. OCM agents in the spoke cluster were offline (e.g. due to an outage) during which its service account token expired
2. A network outage resulted in the spoke cluster loosing connectivity to the hub api server for an extended period of time
3. A service account was intentionally deleted (e.g. the associated token was compromised) and replaced.

### Implementation Details/Notes/Constraints [optional]

TODO

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

TODO

### Graduation Criteria

**Note:** *Section not required until targeted at a release.*

TODO

### Upgrade / Downgrade Strategy

TODO

### Version Skew Strategy

TODO

## Implementation History

## Drawbacks

## Alternatives

- Cloud provider native IAM support - this will be covered in a new enhancement.
- CSR remains the preferred approach to spoke cluster authentication with the hub, where usable.

## Infrastructure Needed [optional]

No specific infrastructure required. 