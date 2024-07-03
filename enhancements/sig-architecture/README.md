## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

OCM's use of a CSR based mechanism for registering managed clusters with the hub cluster is incompatible with Kubernetes environments that cannot issue client auth certificates such as Amazon Elastic Kubernetes Service (EKS).
This enhancement provides an AWS IAM based registration mechanism so that OCM can support EKS-based hub clusters.

## Motivation

For cluster administrators, it is often preferable to leverage their cloud provider’s managed Kubernetes service (e.g. AKS, EKS, IKS, GKE etc) rather than self-managing the Kubernetes control plane and worker nodes - reducing cluster management overhead and complexity.
However, in some of these environments (for example EKS), the use of CSRs for client authentication is unsupported.

Adopting OCM should not require its users to change how they deploy and manage their hub Kubernetes cluster.

One option is to run [multicluster-controlplane](https://github.com/open-cluster-management-io/multicluster-controlplane) service inside or outside Kubernetes and use that as a hub. However, multicluster-controlplane comes with following limitations:
- It is not a full Kubernetes controlplane and hence some OCM addons might not work
- The user is responsible for maintaining HA etcd cluster, instead of being able to leverage managed etcd that comes with EKS controlplane. 

As such, OCM needs to support an alternative registration mechanism to CSRs that is expected to work in EKS, providing maximum compatibility with managed EKS.

### Goals

- The OCM hub cluster can run on EKS.
- `clusteradm` tooling provides options to select aws-irsa registration while initializing the hub as well as joining a managed cluster to the hub.
- Make authentication strategy pluggable, making it easier for other cloud providers to add support for their platform in the future.

### Non-Goals

- Support for running a hub cluster on other cloud providers like Google Cloud Platform GKE and Azure AKS, even though this enhancement will make future support for other cloud providers easier.
- Support for eks cluster as hub and any other cloud provider kubernetes cluster as a managed cluster. For this enhancement, The requirement is to have both the managed as well as hub cluster in the same.

## Proposal

### User Stories

#### Story 1 - Hub administrator initializes a hub cluster using AWS IRSA authentication strategy

It must be possible for the cluster administrator to specify they wish to authenticate registration requests using `aws-irsa` authentication strategy in the `clusteradm init` command. The default authentication strategy will be `csr`:

```
% clusteradm init \
     --wait \
     --registration-auth=aws-irsa \
     --context ${CTX_HUB_CLUSTER}
```
As part of the clusteradm init command, we will create a dummy CSR (Certificate Signing Request), attempt to approve it, and check if it succeeds or fails. If client CSR authentication is not available for the hub cluster, the CSR will receive a "failed" condition. This failure will result in an error for the user attempting to run the clusteradm init command with --registration-auth=csr or by leaving it at its default setting.

#### Story 2 - Managed cluster administrator joins a cluster to the hub using AWS IRSA authentication strategy

It must be possible for the cluster administrator to specify they wish to use `aws-irsa` authentication strategy in the `clusteradm join` command to register with hub. The default authentication strategy will be `csr`:

```
% clusteradm join \
     --registration-auth=aws-irsa \
     --hub-token XXX \
     --hub-apiserver https://hub-0.k8s.example.com \
     --cluster-name managed-0
```


#### Story 3 - Hub administrator accepts a managed cluster using AWS IRSA authentication strategy

It must be possible for the cluster administrator to specify they wish to accept the cluster registration requests using `aws-irsa` authentication strategy in the `clusteradm accept` command, so that it can create identities on hub IAM for managed cluster. The default authentication strategy will be `csr`:

```
% clusteradm accept \
     --registration-auth=aws-irsa \
     --clusters managed-0
```


#### Story 4 - Managed cluster administrator un-joins a cluster from the hub using AWS IRSA authentication strategy

It must be possible for the cluster administrator to specify they wish to use `aws-irsa` authentication strategy in the `clusteradm unjoin` command to un-register with hub. The default authentication strategy will be `csr`:

```
% clusteradm unjoin \
     --registration-auth=aws-irsa \
     --cluster-name managed-0
```

#### Story 5 - EKS Hub administrator must accept the registration request using aws-irsa authentication strategy only, using csr authentication strategy will throw an error

Since EKS doesn't support CSR based client auth and, self managed or clusters running on other cloud providers will not support AWS IRSA, one hub cluster will not support both the authentication strategies at the same time. Running `clusteradm accept` command on EKS with --registration-auth=csr will throw an error as there is no csr to approve, provided both `clusteradm init` and `clusteradm join` were run using --registration-auth=aws-irsa on hub and managed cluster respectively. 


#### Story 6 - Procedure for managed cluster administrator to follow to refresh the token

The temporary IAM credentials obtained by klusterlet-agent from sts will have TTL of 1 hour by default, for the role assumed by pod, which will be [renewed](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/Credentials.html#:~:text=Occasionally%20credentials%20can%20expire%20in%20the%20middle%20of%20a%20long%2Drunning%20application.%20In%20this%20case%2C%20the%20SDK%20will%20automatically%20attempt%20to%20refresh%20the%20credentials%20from%20the%20storage%20location%20if%20the%20Credentials%20class%20implements%20the%20refresh()%20method.) by sdk automatically. More info can be found [here](https://docs.aws.amazon.com/eks/latest/userguide/pod-configuration.html).

The EKS kubeconfig will use these temporary IAM creds to generate temporary sts token using `aws eks get-token` command present in kubeconfig with [TTL of 15 mins](https://aws.github.io/aws-eks-best-practices/security/docs/iam/#:~:text=The%20token%20has%20a%20time,each%20time%20the%20token%20expires.). This sts token authenticates every request sent to EKS api-server, and is automatically renewed by kubeconfig on expiry. Snippet of user from an EKS kubeconfig:
```yaml
users:
- name: test-user
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: aws
      args:
      - --region
      - my-aws-region
      - eks
      - get-token
      - --cluster-name
      - my-cluster
```

We don't have to do anything additional to make it refresh the tokens, aws/eks infrastructure takes care of that for us.

#### Story 7 - Procedure for hub administrator to follow, to revoke access to a managed cluster

It must be possible for the hub cluster administrator to revoke access to a managed cluster. The hub admin can simply delete the `ManagedCluster` from hub and the new `aws-irsa` controller will delete all managed cluster specific resources on hub.

#### Story 8 - Managed cluster permission isolation on hub

One managed cluster should not be able to access resources on hub that belong to another managed cluster. 

### Implementation Details/Notes/Constraints

#### Changes required on hub
The IAM roles and policies for managed cluster will be created when hub-admin accepts the registration request by running `clusteradm accept` command on hub. The role name and policies will be created using templates to ensure standardization. The role name should follow a standard naming convention so that the role on managed cluster can assume the hub role with a standardized name. It will also create an entry aws-auth configmap which will bind the iam role with permissions inside the cluster so that the incoming requests from managed cluster can have the required access inside the hub cluster.

If the feature gate `ManagedClusterAutoApproval` feature gate is enabled, choosing `aws-irsa` instead of `csr` will disable [csr controller](https://github.com/open-cluster-management-io/ocm/blob/main/pkg/registration/hub/csr/controller.go) on hub and enable a new `aws-irsa` controller, which will auto approve all registration requests, and create IAM roles and policies and RBAC permissions. 

The hub registration controller must assume an IAM role with the permissions allowing it to create roles for managed clusters and attach policies. The name of this role will follow a standard template which will contain cluster name which can be parsed from the kubeconfig used to run `clusteradm init` command. The role name should follow a standard name so that the role can be created before running `clusteradm init` command and the hub controller can assume the role with a standardized name, on startup:
```shell
arn:aws:iam::<account-id>:role/<hub-cluster-name>_managed-cluster-identity-creator
```

#### Changes required on the managed cluster
Choosing `aws-irsa` instead of `csr` will disable [cert_controller](https://github.com/open-cluster-management-io/ocm/blob/main/pkg/registration/clientcert/cert_controller.go) on managed cluster and enable a new `aws-irsa` controller in klusterlet/registration agent, which will create `ManagedCluster` only on hub and not CSR. The `aws-irsa` controller will watch for the updates on `ManagedCluster` , and once accepted on hub and has HubAcceptedManagedCluster condition, it will generate a kubeconfig using following command and save it to a secret called hub-kubeconfig-secret on managed cluster:
```shell
aws eks update-kubeconfig --name <hub-eks-cluster-name> --kubeconfig <path/to/create/kubeconfig> --role-arn <role-on-hub-for-managed>
```

The klusterlet/registration agent on managed cluster must assume an IAM role on its IAM, on startup, with the permissions allowing it assume the role on hub IAM. The name of this role will follow a standard template which will contain managed cluster name which can be parsed from the `--cluster-name` option passed to `clusteradm join` command. The role name and policies will be created using templates to ensure standardization. The managed cluster admin must know in advance which hub cluster to join, as the name and the aws account of the hub cluster will be used in the templates to create IAM role and policies on managed cluster IAM. Another reason to follow a standard name is to ensure that the role can be created before running `clusteradm join` command and the klusterlet/registration agent can assume that role, on startup, with a standardized name.
```shell
arn:aws:iam::<account-id>:role/managed_<managed-cluster-name>_<hub-account-id>_<hub-cluster-name>
```

#### Graceful resource cleanup on both hub and managed cluster when managed cluster triggers unjoin

The `ManagedCluster` will have a new boolean field `clientUnjoinsHub` with the default value of false. The `clusteradm unjoin` command by managed cluster admin will update the `ManagedCluster` on hub and set the `spec.clientUnjoinsHub` to true, which will trigger the new `aws-irsa` controller on hub to clean-up all the managed cluster specific resources, like IAM roles and policies on hub, and ultimately delete the `ManagedCluster`.

or

The unjoin command will update the `ManagedCluster` on hub and add a new condition `ManagedClusterUnjoined`, which will trigger the new `aws-irsa` controller on hub to clean-up all the managed cluster specific resources, like IAM roles and policies on hub, and ultimately delete the `ManagedCluster`.


#### Managed cluster permission isolation on hub

Every managed cluster will have a separate role on both managed cluster as well as hub IAM. Every managed cluster role on hub will allow access to managed cluster's namespace only. All roles will follow a standard naming convention, and the IAM trust policies allow roles to be assumed by relevant resources (pods, clusters, etc) only.

This will ensure that roles can only be assumed by the consumers they are meant for, such that one managed cluster cannot assume roles and access resources belonging to other managed cluster

### Workflow Details

This is to describe the various process of cluster join

Actors:
1. cluster-admin on managed cluster
2. AWS IAM admin on managed cluster AWS account
3. cluster-admin on hub cluster
4. AWS IAM admin on hub cluster AWS account
5. hub controller
6. agent on managed cluster

Some rules on cluster join:
- The name of the cluster must be globally unique on hub and conforms to dns label format.

#### Managed Cluster Prerequisites

An IAM role named using following template:
```
arn:aws:iam::<account-id>:role/managed_<managed-cluster-name>_<hub-account-id>_<hub-cluster-name>
```

Having following access policy:
```json
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Action": "sts:AssumeRole",
         "Resource": "arn:aws:iam::<account-id>:role/hub_<hub-cluster-name>_<managed-account-id>_<managed-cluster-name>"
      }
   ]
}
```

And following trust relationship policy:
```json
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Principal": {
            "Federated": "arn:aws:iam::<account-id>:oidc-provider/<oidc-provider>"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
            "StringEquals": {
               "<oidc-provider>:sub": "system:serviceaccount:open-cluster-management-agent:<klusterlet-name>-registration-sa",
               "<oidc-provider>:aud": "sts.amazonaws.com"
            }
         }
      },
      {
         "Effect": "Allow",
         "Principal": {
            "Federated": "arn:aws:iam::<account-id>:oidc-provider/<oidc-provider>"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
            "StringEquals": {
               "<oidc-provider>:sub": "system:serviceaccount:open-cluster-management-agent:<klusterlet-name>-work-sa",
               "<oidc-provider>:aud": "sts.amazonaws.com"
            }
         }
      }
   ]
}
```

#### Hub Cluster Prerequisites

An IAM role named using following template:
```
arn:aws:iam::<account-id>:role/<hub-cluster-name>_managed-cluster-identity-creator
```

Having the following access policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:UpdateAssumeRolePolicy",
        "iam:PutRolePolicy",
        "iam:DeleteRolePolicy",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:CreatePolicy",
        "iam:DeletePolicy",
        "iam:CreatePolicyVersion",
        "iam:DeletePolicyVersion",
        "iam:ListAttachedRolePolicies",
        "iam:ListRolePolicies",
        "iam:ListPolicyVersions",
        "iam:GetRole",
        "iam:GetRolePolicy",
        "iam:GetPolicy",
        "iam:GetPolicyVersion",
        "iam:PassRole"
      ],
      "Resource": [
        "arn:aws:iam::<hub-account-id>:role/hub_<hub-cluster-name>_*_*",
        "arn:aws:iam::<hub-account-id>:policy/hub_<hub-cluster-name>_*_*"
      ]
    }
  ]
}
```

And following trust relationship policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<account-id>:oidc-provider/<oidc-provider>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "<oidc-provider>:sub": "system:serviceaccount:open-cluster-management-hub:registration-controller-sa",
          "<oidc-provider>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

#### Cluster join initiated from managed cluster
1. cluster-admin on managed cluster gets a bootstrap kubeconfig to connect to hub,
   and deploy the agent on managed cluster.
- it has the identity to create `ManagedCluster`.

2. agent on managed cluster creates `ManagedCluster` on hub if it does not exist.
- The name of `ManagedCluster` is passed using the `--cluster-name` option while running `clusteradm join` command.
- The agent on the managed cluster will store it as `agent-name` in `Secret` `hub-kubeconfig-secret`(step 8), so restarting agent or redeploying agent will not lose the UID after the cluster is managed successfully.

3. cluster-admin on hub-cluster accepts the registration request using `clusteradm accept` command, and triggers following. Only a user on hub who has IAM permissions to create IAM resources and RBAC permission to update subresource of `managedclusters/accept` can run this command:
- Creates identities including role, access policy and trust policy using the following templates for the `ManagedCluster` on hub IAM:

    Role
    ```
    arn:aws:iam::<hub-account-id>:role/hub_<hub-cluster-name>_<managed-account-id>_<managed-cluster-name>
    ```

    Access Policy
    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "eks:DescribeCluster",
            "eks:ListClusters"
          ],
          "Resource": "arn:aws:eks:<hub-region>:<hub-account-id>:cluster/<hub-cluster-name>"
        }
      ]
    }
    ```

    Trust relationship policy
    ```shell
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": {
            "AWS": "arn:aws:iam::<managed-cluster-account-id>:role/managed_<managed-cluster-name>_<hub-account-id>_<hub-cluster-name>"
          },
          "Action": "sts:AssumeRole",
          "Condition": {}
        }
      ]
    }
    ```
- Creates an entry in the `aws-auth` configmap for the newly created IAM role and maps it to a group with following name. The `ManagedCluster` gets permissions inside the hub through `system:open-cluster-management:<clusterName>` group.
- Updates `spec.hubAcceptsClient` on `ManagedCluster` to `true`

4. hub-controller creates a namespace as the name of managed cluster on hub cluster if it does not exist.
- managed cluster can only join a hub once, and it can join to multiple hubs.
- The UID of the managed cluster is identical on each of the hub the Klusterlet agent joins.
 
5. hub-controller creates a clusterrolebinding on the hub, binds role with the identity of
   `open-cluster-management:managedcluster:<clusterName>` to group `system:open-cluster-management:<clusterName>`
     - Allows status update of `ManagedCluster`

6. hub-controller creates rolebinding `open-cluster-management:managedcluster:<clusterName>:registration` bound to cluster role `open-cluster-management:managedcluster:registration` on the cluster namespace on the hub
- Allows the access of agent on managed cluster to the namespace.

7. hub-controller creates rolebinding `open-cluster-management:managedcluster:<clusterName>:work` bound to cluster role `open-cluster-management:managedcluster:work` on the cluster namespace on the hub
- Allows the access of agent on managed cluster to pull `ManifestWork` from the namespace.

8. hub-controller updates condition of `ManagedCluster` to `HubAcceptedManagedCluster`.

9. agent on managed cluster assumes its role on hub, and create a new kubeconfig `hub-kubeconfig-secret` using following command and saves it as secret:
    ```shell
    aws eks update-kubeconfig --name <hub-eks-cluster-name> --kubeconfig <path/to/create/kubeconfig> --role-arn <role-on-hub-for-managed>
    ```

10. agent on managed cluster connects to hub apiserver using the new kubeconfig.

11. agent on managed cluster updates conditions of `ManagedCluster` as `ManagedClusterJoined`.

12. agent on managed cluster appends updates other fields in status of `ManagedCluster`. 

13. agent on managed cluster pulls the`ManifestWork` and creates resource.

Registration workflow
![OCM-using-AWS-IAM.jpg](OCM-using-AWS-IAM.jpg)

### Risks and Mitigation

TODO

## Design Details

### TODO
1. Confirm with a POC, that temporary IAM creds set in klusterlet-agent pod env are refreshed automatically after 1 hour.
2. Confirm with a POC, that the permissions listed in the proposed IAM policies is the least permission set required for this solution to work. Also ensure that, the permission set is a complete set to required by work-agent.
3. Ensure that the role names fit in the aws role name limit of 64 character per role.

### Open Questions

#### New feature: Cleanup hub resources when a managed cluster unjoins

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
- CSR remains the preferred approach to managed cluster authentication with the hub, where usable.

## Infrastructure Needed [optional]

No specific infrastructure required. 