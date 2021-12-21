# Securing Sensitive Data in Hub Policy Templates

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
      [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

When a Hub policy template is used to configure a managed cluster, the data is
stored as plaintext in the replicated policy. This limitation prevents a hub
cluster from securely distributing sensitive data to managed clusters using
policies.

To provide this capability, a new hub template function of `protect` will be
added. This would encrypt the data that is piped to it using the AES-CBC
algorithm with a 256 bit key that is unique per managed cluster. In addition, a
`fromSecret` hub template function will be added which allows referencing data
in a secret in the same namespace as the policy. The data from the secret would
be automatically encrypted in the same manner as the `protect` template
function.

## Motivation

When a Hub policy template is used to configure a managed cluster with sensitive
data, it is stored in plaintext in the replicated policy. This is an issue
because it is expected that read access to a replicated policy should not allow
a user to see sensitive data. Additionally, when etcd encryption is enabled,
policy objects are not encrypted. In the previous release, as a stopgap measure,
the `fromSecret` template function was disabled to discourage storing sensitive
data in plaintext in replicated policies. Users have requested the ability to
configure managed clusters with sensitive data on the Hub using hub policy
templates, so the stopgap measure is not a solution.

### Goals

1. A user can securely distribute sensitive data from the hub to managed
   clusters using policies.
1. Sensitive data can be manipulated (e.g. Base64 decode) by hub policy
   templates before being protected in the replicated policy.
1. The performance impact is neglible when protecting the sensitive data.

### Non-Goals

N/A

## Proposal

To allow users to configure managed clusters using policies containing arbitrary
sensitive data, it is proposed to add a template function called `protect`,
which will allow the user to encrypt a string in a Hub policy template. In the
example below, you can see that this is using the existing Hub policy templates
functionality of `lookup`. The first parameter is the `apiVersion` of the
object, the second is the object `kind`, the third is the object’s namespace
that is restricted to the namespace of where the root policy is stored, and the
fourth is the object’s name.

```yaml
'{{hub "(lookup "v1" "Secret" "default" "my-hub-secret").data.message | protect
hub}}'
```

On the replicated policy in the managed cluster namespace, the value would look
like the following:

```yaml
$ocm_encrypted:okrrBqt72oI+3WT/0vxeI3vGa+wpLD7Z0ZxFMLvL204=
```

This resulting value is a Base64 encoded value of the encrypted value. When the
policy is evaluated on the managed cluster, it would result in the decrypted
value of `hello world`. This decrypted value is never stored in the policy.

To make things easier for the user, a new `fromSecret` template function will be
made available for Hub policy templates. It’ll accept the same input as the
`fromSecret` template function available on managed cluster templates, but
output a base64 encoded string of the encrypted value, just as the previous
example looked like on the replicated policy. If a user needs to alter the
secret value such as base64 decode it, they would need to use the first example.
Below is an example of the modified `fromSecret` template function. The first
parameter is the namespace of the secret, which is restricted to the namespace
of the root policy. The second parameter is the secret’s name. The last
parameter is the secret’s data key/field to use.

```yaml
'{{hub fromSecret "default" "my-hub-secret" "message" hub}}'
```

On the replicated policy in the managed cluster namespace, the value would look
like the following:

```yaml
$ocm_encrypted:okrrBqt72oI+3WT/0vxeI3vGa+wpLD7Z0ZxFMLvL204=
```

### User Stories

#### Story 1

As a user, I would like to securely distribute sensitive data from the Hub
cluster to managed clusters using policies.

### Implementation Details

#### Encryption

In order to support encrypting large values such as configuration files, it is
recommended to use a symmetric encryption algorithm such as AES-256 (in
particular AES-CBC with a 256 bit key). The reason is that RSA asymmetric
encryption is limited to roughly the bit size of the key or less. For instance
an RSA 4096 key is limited to roughly 4096 bits of encryption. There are
workarounds such as encrypting several chunks of the data individually, but this
is much less efficient than just using an algorithm such as AES-CBC. It does add
some risk since an OpenShift user that does have access to the symmetric key
stored on the Hub and the replicated policy would be able to decrypt the
contents stored in the replicated policy even if they didn’t have access to the
source of the encrypted content. This is easily mitigated with the principle of
least privilege and is a beneficial tradeoff considering the reduced complexity,
improved encryption security, and performance gains over asymmetric encryption.
This is why it is proposed to move forward with using AES-256 encryption.

To avoid identifying the same encrypted data across different policies, a random
initialization vector (IV) per policy will be used. This will be stored as an
annotation on the replicated policy since it does not need to be private. This
implementation does make it so that the duplicate plaintext values that are
encrypted in a policy for a particular managed cluster would appear the same
since they would use the same IV and AES key. This added risk is minimal though
since this can already be determined by looking at the original root policy and
seeing the Hub policy templates referencing the same key (i.e. field name) of
the data source (e.g. a Kubernetes secret).

#### Key Generation and Storage

The AES-256 key needs to be generated and stored both on the Hub cluster and the
managed cluster. There would be a unique key per managed cluster. This would
mean that if a single managed cluster is compromised, it does not mean that all
encrypted values in the other managed clusters have also been compromised.

The Policy Propagator should be responsible for generating the AES-256 key for
each managed cluster. Each key would be stored in a secret called
`policy-encryption-key` in each respective managed cluster namespace. The key
would be stored on the secret’s `key` property. With this in place, the Policy
Propagator is able to encrypt the data using the `protect` template function,
however, the managed cluster also needs the key when it decrypts the data.

The Policy Spec Sync controller is responsible for syncing the
`policy-encryption-key` secret to the managed cluster in the managed cluster
namespace by watching the managed cluster namespace for the
`policy-encryption-key` secret on the Hub. This means whenever the secret is
created or modified, it’ll be synced on the managed cluster.

To account for key rotation, a new controller will be added to the Policy
Propagator. Its responsibility is to watch the `policy-encryption-key` secret
objects from all namespaces. The watch will be limited by using a field
selector, thus causing the watch to be very tightly scoped and efficient.
Additionally, the ACLs of the controller’s service account will be limited to
just the `policy-encryption-key` secrets. This controller will read the
`policy.open-cluster-management.io/last-rotated` annotation on the
`policy-encryption-key` secret and use this to determine when to rotate the key.
This will be set to be rotated every 30 days. When it performs a rotation, the
old key will be set to the `previousKey` field on the secret and the new key
will be set on the `key` field. The annotation will be updated with the current
timestamp. It’ll then signal the Policy Propagator to reprocess all policies
that use the old encryption key. By setting the `previousKey` field, it’ll allow
the managed cluster to use both the new key and the previous key for decrypting
policy templates while all policies are being reprocessed.

If a managed cluster administrator wants to trigger an immediate key rotation,
they can delete the `policy.open-cluster-management.io/last-rotated` annotation
and this will cause the key to be rotated immediately by the new controller
previously mentioned. If the managed cluster administrator wants to provide
their own key, they are responsible for the rotation and must set the
`policy.open-cluster-management.io/disable-rotation` annotation to `true` on the
`policy-encryption-key` secret so the new controller will not perform automatic
key rotation.

### Risks and Mitigation

See the Implementation Details section.

## Design Details

See the Implementation Details section.

### Test Plan

Test cases:

1. Test that the `protect` hub template function encrypts the data on the
   replicated policy and it can be decrypted on the managed cluster
1. Test that the `fromSecret` hub template function encrypts the data on the
   replicated policy and it can be decrypted on the managed cluster.

### Graduation Criteria

It is proposed that this be considered stable in the first release after the
implementation is complete.

### Upgrade / Downgrade Strategy

This new feature does not modify existing functionality, so upgrades are
unaffected. If the Hub is downgraded after the user starts using this feature,
the Policy Propagator will mark the affected policies as noncompliant since the
user would be using a non-existent template function.

### Version Skew Strategy

If the Hub is upgraded to support this feature but the managed cluster is not,
then any use of this feature will deliver an encrypted string that the managed
cluster will not attempt to decrypt.

## Implementation History

N/A

## Drawbacks

See the Implementation Details section.

## Alternatives

### Resolve All Sensitive Data at the Point of Enforcement

The general idea with this approach is that any sensitive data on the hub that
needs to be propagated to the managed cluster would need to be stored in the
managed cluster namespace. Then when the Configuration Policy controller
assesses the policy on the managed cluster, the sensitive data is retrieved on
the Hub. This means that the sensitive data is never stored in a policy in any
form.

From a user experience perspective, this is not ideal since it means that a user
cannot maintain just a single Secret for configuring all their managed clusters.
They’d have to have a separate Secret per managed cluster, which largely reduces
the benefit of Hub policy templates.

Another downside to this approach is that this would involve continuous API
calls to the Hub cluster every time the Configuration Policy is assessed. By
default, the Configuration Policy Controller does this every 10 seconds
excluding the delay of actually processing each policy. This would particularly
be an issue in environments with limited network connectivity or in temporary
disconnected environments (e.g. a cluster on a ship). If the Configuration
Policy controller cannot connect to the Hub, it would not be able to assess if
the policy is compliant, and thus would not be able to enforce a policy and
would likely cause the policy status to change to non-compliant depending on the
implementation.

Additionally, there is concern about the additional load that this would put on
the Hub cluster since even just one such policy distributed to 2000 managed
clusters would cause an average of 2000 API calls over 10 seconds. Lastly, it
would also slow down the processing of policies on the managed cluster since
it'd be blocked by waiting on each API call to the Hub.

If the design were altered slightly to have the Policy Propagator create Secrets
on the fly based on the user's templates, this would lead to a large number of
Secrets on the Hub. For example, if there were a 1000 managed clusters that each
use 5 policies that use the new `fromSecret` Hub policy template, it would lead
to 5000 secrets being created on the Hub. This would lead to more load on the
Kubernetes API on the Hub since the Secrets would always need to be evaluated to
ensure they are in sync every time a policy is reprocessed on the Hub. The
secrets could be cached but that would involve adding a watch on all Secrets in
all namespaces. This could be potentially reduced by using a field selector on
the watch to limit the Secrets based on some predefined label that is added to
these generated Secrets, but that would still be 5000 secrets to watch based on
the above scenario. The managed cluster would then also need to watch all
secrets in the managed cluster namespace rather than just the single
`policy-encryption-key` Secret in the preferred proposal.

There is also the performance concern on the managed cluster since every time
the Configuration Policy is assessed, it would have to perform API queries to
get the synced Secrets on the managed cluster. This could be alleviated with
caching by adding a watch on the managed cluster namespace, however, this adds
additional load to the Kubernetes API server.

Additionally, the preferred proposal leads to tighter scoped ACLs since the
managed cluster just needs access to sync the `policy-encryption-key` Secret in
the managed cluster namespace. This alternative approach would require the
managed cluster to need access to all Secrets in the managed cluster namespace
on the Hub since the Secret names are not known at OCM deployment time. Also, on
the Hub, the preferred approach requires only the Policy Propagator to have
access to the `policy-encryption-key` Secrets in all namespaces. This
alternative approach would require access to all Secrets in all namespaces since
at OCM deployment time, the managed cluster namespaces and Secret names are not
known.

### Syncing Secrets

Another option is for a call to the `protect` template function to create a
secret to contain the encrypted data in the managed cluster namespace on the
Hub. The managed cluster would then sync the secrets from the Hub and then the
managed cluster would decrypt the value from the synced secret.

The benefit to this is we do not need to perform custom encryption. The downside
is that it has some scalability issues. Imagine that there are 2000 managed
clusters and each has a few policies with encryption, that would generate 6000
secrets on the Hub and on the managed clusters. Additionally, each managed
cluster would have to watch for secrets on the hub to sync back to the managed
cluster. This would mean an additional 2000 watches which is difficult for
managed clusters which do not have a good network connection back to the Hub
(e.g. far-edge).

### Encrypting the Entire Object Definition in the Configuration Policy

Another option is to encrypt the whole object definition in the Configuration
Policy. This would make it easier for the user creating the policy since they
would not have to change how they write their Hub policy templates, however, it
makes it so that the replicated policy cannot be properly viewed for auditing
since the whole object definition would be encrypted.
