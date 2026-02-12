# AWS Custom IAM

## Release Signoff Checklist

- [] Enhancement is `provisional`
- [] Design details are appropriately documented from clear requirements
- [] Test plan is defined
- [] Graduation criteria for dev preview, tech preview, GA
- [] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal expands existin awsirsa registration to support a wider range of installations.
By disabling IAM management in the hub, OCM awsirsa registration can be more flexible in how
it operates and what types of spoke clusters can be connected.

## Motivation

The current support for [registration with AWS auth](https://github.com/open-cluster-management-io/ocm/tree/main/solutions/joining-hub-and-spoke-with-aws-auth)
is highly prescriptive in the way it functions and limited to spoke clusters running on AWS.
This causes problems in cases where IAM management permissions cannot be granted (e.g. when standing up
infrastructure in high-security customer accounts) and for connecting clusters from outside of AWS
to an EKS hub.

### Goals

- Support running an OCM hub in EKS without granting it any IAM permissions
- Support connecting spoke clusters outside of AWS to an OCM hub in EKS
- Support both EKS clusters with managed IAM and non-EKS clusters connecting to the same hub
  - A spoke cluster that joins without a ManagedClusterArn will not create a role in the hub even with managed IAM enabled

## Proposal

Add `DisableManagedIam` boolean to `AwsIrsaConfig` on the `ClusterManager` CRD. This flag instructs the hub registration
components not to manage any IAM components for registering a spoke cluster. It will be the responsibility of the admin(s)
setting up the spoke/hub to create all necessary IAM roles and access policies to connect the clusters. Documentation
will be provided to explain how to set this up.

Change `ManagedClusterArn` to optional in `AwsIrsa` on the `Klusterlet` CRD. Managed clusters will no longer be required
to be running in AWS, and in this case they will not have an ARN.

Add `IamConfigSecret` string to `AwsIrsa` on the `Klusterlet` CRD. This will be the name of a secret that contains (AWS
config/credential files)[https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html] for authentication
from spoke clusters that are not running on AWS (or do not have EKS Pod Identity enabled on the cluster). It will be
the responsibility of the admin(s) in charge of the spoke cluster to ensure these credentials are properly managed and kept
up to date. Because they will be mounted as files in the spoke agent, they can be rotated by externally managed jobs and
the spoke agent will recieve the updated credentials from kubelet injection. The only required permissions for the credentials
configured in this secret will be to assume the role in the hub AWS account that is associated with the managed spoke cluster.
Documentation will be provided to explain how to create this secret and the trust policy for assuming the managed cluster's
role in the hub account.

### User Stories

#### Story 1 - Installing OCM hub without permission to manage IAM

Not all IT organizations freely give out permission to create IAM roles/policies. It must be possible to install OCM on AWS EKS
without requiring that it has permissions in AWS to create/delete IAM resources.

This allows installation in high-security accounts, as well as giving flexibility to the user to manage permissions in any
way they need.

```
% clusteradm init \
      --context ${CTX_HUB_CLUSTER} \
      --wait \
      --registration-drivers awsirsa \
      --disable-managed-iam
```

#### Story 2 - Joining cluster from outside AWS to an OCM hub on AWS EKS

It must be possible to join spoke clusters running in any environment to a hub that is running on AWS EKS. The ability
to run spoke clusters in a wide range of environments is one of the biggest selling points for OCM, but the limitations
of awsirsa registration currently make that not possible.

```
% kubectl create secret generic iam-config \
      --context ${CTX_SPOKE_CLUSTER} \
      --namespace open-cluster-management-agent \
      --from-file config=./aws.config \
      --from-file credentials=./aws.credentials

% clusteradm join \
      --context ${CTX_SPOKE_CLUSTER} \
      --wait \
      --hub-token ${HUB_TOKEN} \
      --hub-apiserver ${HUB_API_SERVER} \
      --registration-auth awsirsa \
      --hub-cluster-arn ${HUB_CLUSTER_ARN} \
      --iam-config-secret iam-config
```

### Design Details

#### API change

To disable IAM management in the hub, `disableManagedIam` will be added to `awsirsa` in the `ClusterManager` CRD

```go
type AwsIrsaConfig struct {
  ...

	// DisableManagedIam disables creation and management of IAM roles and policies on the hub.
	// If true, all AWS permissions for awsirsa registration must be managed manually by the administrator.
	// Used in cases where IAM permissions cannot be granted to OCM, or to run an EKS hub with non-aws spoke clusters.
	// +optional
	DisableManagedIam bool `json:"disableManagedIam,omitempty"`
}
```

To support spoke clusters running outside of AWS, `managedClusterArn` will be marked optional and `iamConfigSecret`
will be added to `awsIrsa` in the `Klusterlet` CRD.

```go
type AwsIrsa struct {
  ...

	// The arn of the managed cluster (ie: an EKS cluster). This will be used when managed IAM is enabled to generate the md5hash as a suffix to create IAM role on hub
	// as well as used by kluslerlet-agent, to assume role suffixed with the md5hash, on startup.
	// Example - arn:eks:us-west-2:12345678910:cluster/managed-cluster1.
	// +optional
	// +kubebuilder:validation:Pattern=`^arn:aws:eks:([a-zA-Z0-9-]+):(\d{12}):cluster/([a-zA-Z0-9-]+)$`
	ManagedClusterArn string `json:"managedClusterArn"`

	// IamConfigSecret is the name of a secret containing "config" and/or "credentials" files mounted to ~/.aws/config and ~/.aws/credentials respectively.
  // Used in cases where the managed cluster is running outside of AWS or is unable to use EKS Pod Identity to retrieve credentials from ServiceAccounts.
	// More Info: https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html
	IamConfigSecret string `json:"iamConfigSecret"`
}
```

#### hub implementation

`DisableManagedIam` will be passed into `AWSIRSAHubDriver`. When set, `CreatePermissions` and `Cleanup` functions will
skip making any changes when registering a new spoke cluster. The `ManagedClusterIdentityCreatorRole` will be empty
since the hub registration controller no longer needs any IAM permissions.

Instructions will be added to the [documentation](https://github.com/open-cluster-management-io/ocm/tree/main/solutions/joining-hub-and-spoke-with-aws-auth) for how to register new clusters when `DisableManagedIam` is true.
- Create a role for the managed cluster `ocm-hub-<md5hash>`
- Create an access policy on the EKS cluster for the managed cluster role
  - IAM principal ARN: `arn:aws:iam::<hub-account-id>:role/ocm-hub-<md5hash>`
  - Type: Standard
  - Username: <managed-cluster-name>
  - Group Names:
    - open-cluster-management:<managed-cluster-name>

#### agent implementation

`IamConfigSecret` will be passed into `AwsIrsa` on `RegistrationDriver`. When set, the named secret will be mounted to `/.aws` on the klusterlet-agent.
Since `ManagedClusterArn` is now optional, the klusterlet-work-sa serviceaccount will only be annotated with a role ARN when present. Otherwise it is
assumed that the cluster admin(s) have provided credentials in some other way (e.g. with `IamConfigSecret`).

The formula for computing the role md5hash remains the same, but if `ManagedClusterArn` is empty then "managedClusterName" retrieved from the
`ManagedCluster` object and "managedClusterAccountId" is left empty.


#### clusteradm implementation

`ManagedClusterArn` will not be required. This primarily impacts the `getManagedClusterArn` function, which returns an error if it is unable to retrieve the ARN
from flags or kubeconfig. It will now return empty string if no valid ARN can be found.

Add a `--iam-config-secret` flag to the join command, which will be passed into the Klusterlet object if set.

Add a `--disable-managed-iam` flag to the init command, which will be passed into the ClusterManager object if set.

#### examples

##### Computing Md5HashSuffix for a non-aws cluster
Input string format: `<hubClusterAccountId>#<hubClusterName>##<managedClusterName>`

Example: `123456789#eks-hub##localcluster` => `1c4185a896c3fca92fa78ecae00dddd1`

##### Disable Managed IAM
```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: ClusterManager
metadata:
  name: cluster-manager
spec:
  ...
  registrationConfiguration:
    ...
    registrationDrivers:
    - authType: awsirsa
      awsirsa:
        disableManagedIam: true
        hubClusterArn: arn:aws:eks:<region>:<account>:cluster/<hub-cluster>
```

##### Klusterlet IAM credentials
```yaml
apiVersion: operator.open-cluster-management.io/v1
kind: Klusterlet
metadata:
  name: klusterlet
spec:
  clusterName: <managed-cluster>
  ...
  namespace: open-cluster-management-agent
  registrationConfiguration:
    ...
    registrationDriver:
      authType: awsirsa
      awsIrsa:
        hubClusterArn: arn:aws:eks:<region>:<account>:cluster/<hub-cluster>
        iamConfigSecret: iam-config
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: iam-config
  namespace: open-cluster-management-agent
data:
  config: <.aws/config>
  credentials: <.aws/credentials>
type: Opaque
```

### Test Plan
- Test joining a spoke cluster with hub `DisableManagedIam: true`
  - Hub ServiceAccount `open-cluster-management-hub/registration-controller-sa` has no IAM role annotations
  - IAM role and EKS access policy are not created automatically in Hub account
- Test unjoining a spoke cluster with hub `DisableManagedIam: true`
  - IAM role and EKS access policy are not deleted in Hub account
- Test joining a spoke cluster from outside AWS
  - Hub does not throw errors if `ManagedClusterArn` is not set (even when DisableManagedIam=false)
  - Spoke does not throw errors if `ManagedClusterArn` is not set
  - Spoke pulls credentials from `IamConfigSecret` and is able to assume role in the hub account

### Graduation Criteria
N/A

### Upgrade Strategy
It will need upgrade on CRD of ClusterManager on hub cluster, and upgrade of Klusterlet on managed cluster.

To disable managed IAM on an existing hub, a user can update the ClusterManager with the appropriate field. All existing
roles and access policies for managed clusters will not be deleted, and any future registrations will not create new ones.

### Version Skew Strategy
- The new fields are optional, and if not set, the manifestwork will be handled the same as by previous versions
- Older versions of the registration controller and agent will ignore the newly added fields

## Alternatives

An argument could be made that since a Hub with `DisableManagedIam` and Spoke(s) with `IamConfigSecret` and empty `ManagedClusterArn`
are not using IRSA, this should not be part of the "awsirsa" registration driver. The initial POC and design were built with the minimal
changes possible to make AWS registration more flexible, which is why it piggybacks off of the existing driver. However, it may be desireable
to leave these as two distinct registration drivers. In which case, the design can be updated to be a more generic "aws" driver that does not
impose any restrictions on IAM management, and the existing "awsirsa" driver would be left as it is.
