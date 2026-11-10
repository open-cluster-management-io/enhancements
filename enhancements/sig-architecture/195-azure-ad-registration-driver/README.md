## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

OCM's CSR-based registration mechanism requires the cluster's Kubernetes API server to support issuing client certificates. This is discouraged on some  Kubernetes environment, and some operators of Azure Kubernetes Service (AKS) hubs and managed clusters are under a compliance requirement that forbids storing any long-lived credential - client certificate, token, or secret - inside a cluster at all. This enhancement adds an Azure AD (Entra ID) based registration mechanism so a managed cluster can join a hub using an Azure AD access token, obtained through Managed Identity, an environment-supplied credential, or Workload Identity Federation.

## Motivation

Cluster administrators increasingly prefer a managed Kubernetes service (AKS, EKS, GKE, etc.) over
self-managing the control plane. On AKS specifically, some registration flows benefit from avoiding
client-certificate issuance altogether: certificate lifecycle (issuance, rotation, storage) is one
more thing to secure and audit, and organizations with a strict no-stored-secrets policy need a way to
authenticate that never persists credential material on the cluster in the first place.

Azure provides three standard ways for a workload to obtain an Azure AD identity: Managed Identity
(the identity already attached to the underlying node, no separate setup needed), an
environment-supplied service-principal credential for cases where credential provisioning is already
handled by an external pipeline, and Workload Identity Federation (a short-lived, automatically-rotated
token tied to a specific Kubernetes ServiceAccount, with nothing persisted anywhere). None of these are
usable for OCM registration today.

### Goals

- Let a managed cluster register to a hub using an Azure AD identity.
- Support all three Azure credential mechanisms above through the same driver, with the operator
  selecting explicitly which one applies (`azure.credential`) rather than an automatic runtime
  fallback - required so the Helm chart can expose four distinct, validated configuration shapes
  (Managed Identity, Workload Identity Federation, service-principal secret, service-principal
  certificate) instead of one chart input path that behaves differently depending on which
  environment variables happen to be present at runtime.
- Make the authentication strategy pluggable at the config level, consistent with how OCM already
  supports multiple registration strategies (`csr`, gRPC, and one other cloud-IAM-based strategy
  today).
- Grant an Azure-authenticated cluster the same hub-side permissions a CSR-authenticated cluster
  already gets, through the existing permission model - no new permission surface.
- Work correctly across all four klusterlet deployment shapes: `Default`, `Singleton`, `Hosted`, and
  `SingletonHosted`.
- `clusteradm` tooling provides options to select `azure` registration while initializing the hub as
  well as joining a managed cluster to the hub.

### Non-Goals

- Support for addons authenticating to the hub using this same Azure AD identity. Addons continue to
  use the existing `token`/`csr` addon drivers regardless of which driver the managed cluster itself
  joined with.
- Any change to the `RegisterDriver` or `HubDriver` interfaces themselves. This is a new
  implementation of both, not a change to their shape.
- A generic multi-cloud identity abstraction spanning more than one cloud provider in a single driver.
- Any dynamic creation of Azure cloud resources (identities, role assignments, federated credentials)
  by the hub controller. Unlike a model where the hub provisions cloud-provider identities on demand,
  the Azure AD identity and its trust relationship are established out-of-band by whoever administers
  the Azure AD tenant, before a managed cluster ever attempts to join. The hub controller's role is
  limited to Kubernetes-native RBAC.

## Proposal

### User Stories

#### Story 1 - Hub administrator initializes a hub cluster using Azure AD authentication strategy

It must be possible for the hub administrator to specify they wish to authenticate registration
requests using the `azure` authentication strategy in the `clusteradm init` command. The default
authentication strategy remains `csr`:

```
% clusteradm init \
     --wait \
     --registration-auth=azure \
     --context ${CTX_HUB_CLUSTER}
```

This enables the driver hub-wide; it does not name any specific managed cluster's identity. The
only azure-specific hub configuration is optional and applies to every joining cluster equally: a
list of regex patterns matched against a joining cluster's Azure AD object ID for auto-approval
(`--auto-approved-azure-identity-patterns`, mirroring the existing `awsirsa` driver's
`autoApprovedARNPatterns`). A specific managed cluster's identity (its object ID and client ID) is
configured only on that cluster's own `Klusterlet`, in Story 2 below - never on the hub.

#### Story 2 - Managed cluster administrator joins a cluster using an Azure AD identity

I run my hub and managed clusters on AKS. Depending on which of the three Azure credential mechanisms
fits my environment, I select it explicitly with `--azure-credential` - all use the same `clusteradm
join` shape, differing only in the credential flag and what's supplied for the identity. This explicit
selection (rather than an automatic runtime fallback) is what lets the Helm chart expose four
distinct, independently-validated input shapes instead of one chart path whose effective behavior
depends on which environment variables happen to be set at deploy time:

**Managed Identity** - I'd rather use the identity already attached to my node than manage a
certificate lifecycle or set up a federated credential at all. `--azure-client-id` is optional here:
provide it to use a specific user-assigned identity, or omit it entirely to use the node's
system-assigned identity instead:

```
% clusteradm join \
     --registration-auth=azure \
     --hub-token XXX \
     --hub-apiserver https://hub-0.k8s.example.com \
     --azure-credential managed-identity-credential \
     --azure-principle-id <principle-id-of-the-service-principle>
     --azure-client-id <client-id-of-user-assigned-identity> \   # omit for system-assigned
     --cluster-name managed-0
```

**Existing credential pipeline (environment variables)** - a client secret or certificate is already
provisioned by infrastructure I don't want to change. `clusteradm` never accepts secret material as
a CLI flag value or via a literal `echo` argument - unlike a flag's value, argv is visible to any
local user or process that can read `/proc/<pid>/cmdline` while `clusteradm` runs, and a shell
history file persists whatever was typed on the command line, so piping from `echo '<secret>'`
leaks the secret exactly the same way a flag would. Instead the secret-bearing flags below are
boolean switches that make `clusteradm` read the value from **stdin**, sourced from an interactive
`read -s` prompt or a restricted-permission file - never a literal token on the command line.

For `AZURE_CLIENT_SECRET` and `AZURE_CLIENT_CERTIFICATE_PASSWORD` - plain string values -
`clusteradm` creates a Kubernetes `Secret` holding the value and configures the klusterlet agent's
container to consume it via `env[].valueFrom.secretKeyRef`, the standard way a Pod exposes a
`Secret` as an environment variable without the value ever appearing in the Pod spec itself. The
certificate is different: `AZURE_CLIENT_CERTIFICATE_PATH` must resolve to a real file inside the
agent's container, and `secretKeyRef` only injects a string into an environment variable - it
cannot materialize a file. So the certificate bytes go into a separate key of that same `Secret`
and are projected into the container as a mounted file via `volumes`/`volumeMounts` (not
`secretKeyRef`) at a fixed, driver-documented path; `AZURE_CLIENT_CERTIFICATE_PATH` is then set to
that fixed in-container path as a plain, non-secret environment variable, not sourced from the
`Secret` itself. Only the non-secret identifying fields - `azure.clientID`, `azure.tenantID` - end
up in the CR itself.

**Service Principal with Secret**

```shell
% read -s -p 'Client secret: ' CLIENT_SECRET && echo
% clusteradm join \
     --registration-auth=azure \
     --hub-token XXX \
     --hub-apiserver https://hub-0.k8s.example.com \
     --azure-principle-id <principle-id-of-the-service-principle>
     --azure-credential environment-credential-secret \
     --azure-tenant-id <tenant-id-of-service-principal> \
     --azure-client-id <client-id-of-service-principal> \
     --azure-client-secret-stdin \
     --cluster-name managed-0 <<< "$CLIENT_SECRET"
```

`read -s` reads the secret into a shell variable without echoing it to the terminal or the shell
history (only the `read -s -p ...` command itself is logged, never the value typed at the prompt);
the `<<<` here-string then feeds it to `clusteradm` over a pipe, never as an argv token.
`--azure-client-secret-stdin` tells `clusteradm` to read the secret from stdin rather than take it as
a flag value - never appearing in argv, the `Secret`/`secretKeyRef` described above, never the CR.
Only `--azure-tenant-id` and `--azure-client-id` land in `Klusterlet.spec` as `azure.tenantID`/
`azure.clientID`.

**Service Principal with Certificate**

```shell
% read -s -p 'Certificate password: ' CERT_PASSWORD && echo
% clusteradm join \
     --registration-auth=azure \
     --hub-token XXX \
     --hub-apiserver https://hub-0.k8s.example.com \
     --azure-credential environment-credential-certificate \
     --azure-principle-id <principle-id-of-the-service-principle>
     --azure-tenant-id <tenant-id-of-service-principal> \
     --azure-client-id <client-id-of-service-principal> \
     --azure-client-cert-path <local-certificate-path-of-service-principal> \
     --azure-client-cert-password-stdin \
     --azure-client-send-cert-chain <true/false> \
     --cluster-name managed-0 <<< "$CERT_PASSWORD"
```

`--azure-client-cert-path` here is a path on the machine running `clusteradm`, read once to build the
`Secret` above - not a path inside the agent's container. Its *bytes* are written into a dedicated
key of that `Secret` and projected into the agent's container as a mounted file, at a fixed path the
driver documents (e.g. `/var/run/secrets/ocm/azure/tls.crt`); the container's
`AZURE_CLIENT_CERTIFICATE_PATH` environment variable is set to that fixed in-container path as a
plain string, not sourced via `secretKeyRef` from the `Secret` - `secretKeyRef` only injects a string
value into an env var and cannot place a file on disk, so a certificate handled that way would never
be readable by `EnvironmentCredential` at runtime. As with the client secret, the password is read
via `read -s` and piped in as a here-string, so `--azure-client-cert-password-stdin` never sees it as
an argv token. `--azure-client-send-cert-chain` is different again: `EnvironmentCredential` reads it
as the boolean `AZURE_CLIENT_SEND_CERTIFICATE_CHAIN` environment variable, not from a file, so it
goes into the same `Secret` as a string value and reaches the container via `secretKeyRef` like
`AZURE_CLIENT_SECRET` - it isn't secret material itself, but it's meaningless without the
certificate it describes, so it travels alongside it in the `Secret` rather than the CR.

**Workload Identity Federation** - I run under a policy that forbids storing long-lived credentials
on a cluster. My Azure AD identity's federated credential is already configured against my managed
cluster's Kubernetes ServiceAccount, so no client secret exists anywhere for this to leak:

```
% clusteradm join \
     --registration-auth=azure \
     --hub-token XXX \
     --hub-apiserver https://hub-0.k8s.example.com \
     --azure-credential workload-identity-credential \
     --azure-principle-id <principle-id-of-the-service-principle>
     --azure-client-id <client-id-of-service-principal>  \
     --cluster-name managed-0
```

In every case, the resulting Azure AD access token - the credential presented to the hub
apiserver - is obtained fresh via the exec credential plugin and held only in memory; it is never
written to disk, a `Secret`, or `Klusterlet.spec`. This is distinct from the *input* credential
(client secret or certificate password): as described above, that is intentionally persisted as a
Kubernetes `Secret` on the managed cluster so the agent can re-authenticate and mint new tokens.

Rotating that input credential (e.g. issuing a new client secret and updating the `Secret`) does
**not** take effect on its own for `AZURE_CLIENT_SECRET`/`AZURE_CLIENT_CERTIFICATE_PASSWORD`/
`AZURE_CLIENT_SEND_CERTIFICATE_CHAIN`: `env[].valueFrom.secretKeyRef` values are resolved once when
the container starts, and the kubelet does not re-inject them into a running process, so the agent
keeps using the old value until its Pod restarts. The certificate file itself is the exception -
Kubernetes refreshes a mounted `Secret` volume's file contents in place on a change, without a
restart. Rotating a client secret or certificate password therefore requires restarting or rolling
out the klusterlet agent afterward (e.g. `kubectl rollout restart`) to pick up the new value; this
should be exercised as part of testing old-credential revocation followed by a `Secret` update.

#### Story 3 - Hub administrator accepts a managed cluster's registration request

It must be possible for the hub administrator to accept the registration request using the `azure`
authentication strategy based on the presence of the `managed-cluster-azure-identity` annotation on
the `ManagedCluster`:

```
% clusteradm accept \
     --clusters managed-0
```

Equivalently, without `clusteradm`, the hub administrator can set `spec.hubAcceptsClient: true`
directly on the `ManagedCluster`, or enable the existing `ManagedClusterAutoApproval` feature gate
with a regex pattern matched against the Azure AD object ID recorded in that annotation.

#### Story 4 - Managed cluster administrator un-joins a cluster and cleans up hub resources

Run against the managed cluster's own context, this removes the klusterlet and its agents from the
managed cluster:

```
% clusteradm unjoin \
     --cluster-name managed-0
```

This does not by itself touch the hub. Deleting the `ManagedCluster` on the hub (or setting
`spec.hubAcceptsClient: false` on it) is what triggers the hub to remove the RBAC bindings created for
that cluster's identity - there is no separate `clusteradm` command for that side, since it's the same
hub-side cleanup path every other driver already goes through. There are no cloud-provider resources
(IAM roles, policies, federated credentials) for the hub to clean up, since none were created by the
hub controller in the first place - the Azure AD identity and its trust relationship continue to exist
independently of OCM.

#### Story 5 - Managed cluster permission isolation on hub

One managed cluster's Azure AD identity must not be able to access another managed cluster's
resources on the hub. This is provided by the existing per-cluster RBAC model unchanged: each
identity is bound only to the `ClusterRole`s scoped to its own cluster name/namespace, the same
isolation every other driver already relies on. This is a property of the existing RBAC model, not an
action anyone takes, so there is no corresponding `clusteradm` command either.

This isolation assumes each cluster is configured with a distinct Azure AD object ID.
`AzureAuthHubDriver.CreatePermissions` enforces that: before binding RBAC for a cluster, it checks
whether that object ID already has bindings under a *different* cluster name and, if so, refuses to
create them - see [Changes required on the hub](#changes-required-on-the-hub).

### Implementation Details/Notes/Constraints

#### Changes required on the managed cluster

A new package, `pkg/registration/register/azure_auth/`, provides a `RegisterDriver` implementation
that:
- Builds the kubeconfig's `exec` credential plugin to run a `get-azure-token` subcommand.
- Decorates the `ManagedCluster` it creates with a `managed-cluster-azure-identity` annotation holding
  the configured Azure AD object ID, so the hub can see which identity is claiming this cluster before
  any token exchange happens.

Because OCM can run the registration and work agents as one combined process (`Singleton` mode) or as
two separate processes (`Default` mode), and either can run on a separate management cluster
(`Hosted`/`SingletonHosted`), the process that ends up needing to run `get-azure-token` is not fixed.
The subcommand is registered on every binary that could end up in that position: `registration`,
`registration-operator` (which also serves as the `Singleton`-mode combined agent process), and
`work`.

#### Changes required on the hub

A `HubDriver` implementation binds the presented identity - as a Kubernetes `User` - to the same
three `ClusterRole`s a CSR-joined cluster is bound to, once `hubAcceptsClient` is set to `true`. The
exact subject name it binds to depends on which apiserver-side trust is in place:
- **AKS's native Azure AD integration**: the bare Azure AD object (principal) ID.
- **A self-managed apiserver trusting Azure AD as a generic OIDC issuer**: kube-apiserver's default
  username-prefix behavior applies whenever `--oidc-username-claim` is set to something other than
  `email` and no `--oidc-username-prefix` is given - the authenticated username becomes
  `<oidc-issuer-url>#<object-id>`, not the bare ID (see [Hub cluster
  prerequisites](#hub-cluster-prerequisites)). `AzureAuthHubDriver` reproduces that same prefixed
  form when binding RBAC, using a configured `azure.oidcIssuerURL` value (see [Config
  surface](#config-surface)) - left unset for the AKS-native case, where the bare ID is used
  directly. This value must be byte-for-byte identical to whatever the apiserver is actually
  configured with via `--oidc-issuer-url`; even a trailing-slash difference produces a username that
  doesn't match, so the token still validates but every authorization check then fails with a 403.
- `open-cluster-management:managedcluster:<clusterName>` (`ClusterRoleBinding`) - status/identity
  permissions for the cluster's own `ManagedCluster` object.
- `open-cluster-management:managedcluster:<clusterName>:registration` (`RoleBinding` to the shared
  `open-cluster-management:managedcluster:registration` `ClusterRole`) - lease renewal, addon status.
- `open-cluster-management:managedcluster:<clusterName>:work` (`RoleBinding` to the shared
  `open-cluster-management:managedcluster:work` `ClusterRole`) - reading and updating `ManifestWork`.

No new `ClusterRole` is introduced; the identity is added as a subject on existing roles, the same
ones any other driver's accepted cluster is bound to.

Each `ClusterRoleBinding`/`RoleBinding` `CreatePermissions` creates is labeled with the Azure AD
object ID it was created for, in addition to the existing cluster-name label. Before creating
bindings for a cluster, `CreatePermissions` lists existing bindings carrying that same object-ID
label; if any belong to a *different* cluster name, it refuses to create the new bindings and
returns an error instead, which surfaces as a failed/retrying condition on the `ManagedCluster`
rather than a silent, incorrectly-shared grant of access. This runs on every acceptance path -
manual (`hubAcceptsClient: true`) and automatic alike - since `CreatePermissions` is called
regardless of which one accepted the cluster, unlike `Accept`, which only runs for automatic
approval.

Automatic approval reuses the existing `ManagedClusterAutoApproval` feature gate: a list of regex
patterns (`--auto-approved-azure-identity-patterns` on the hub, `azure.autoApprovedIdentityPatterns`
in the `ClusterManager` CR) is matched against the Azure AD object ID on the
`managed-cluster-azure-identity` annotation.

Unlike a design where the hub controller provisions cloud-provider identities dynamically per managed
cluster, no Azure resources are created, updated, or deleted by the hub controller at any point in
this flow. `Cleanup` only removes the Kubernetes RBAC bindings above.

#### Config surface

A new `authType: azure` option is added to the existing discriminated union already used by other
registration strategies, with a different sub-object shape on each side - there is no shared
"identity" schema between hub and spoke:

- **`Klusterlet.spec.registrationConfiguration.registrationDriver.azure`** (spoke), mirroring the
  flags Story 2 adds to `clusteradm join`. Two fields apply regardless of credential type:
  `azure.managedClusterAzureID` (required in every case - the Azure AD object/principal ID being
  claimed, set via `--azure-principle-id` in Story 2, used for hub-side auto-approval matching and
  RBAC binding) and `azure.tokenAudience`. `azure.tokenAudience` is only safely left unset for
  AKS's native Azure AD integration, where it defaults to the well-known AKS AAD Server application
  - see [References](#references). For a self-managed apiserver it is effectively required: it must
  be set to match whatever `--oidc-client-id` that apiserver is configured with (see [Hub cluster
  prerequisites](#hub-cluster-prerequisites)), and the AKS default is very unlikely to match an
  arbitrary self-managed apiserver's own client ID. `azure.credential` selects which of the four `--azure-credential`
  values from Story 2 is in use, and each one has its own mandatory/optional fields - there is no
  single required-field set that applies uniformly across all four, since each corresponds to a
  distinct Helm chart input form that should only validate the fields it actually needs:
  - **`managed-identity-credential`**: `azure.clientID` is *optional* - set it to use a
    user-assigned identity, or omit it entirely to fall back to the node's system-assigned identity.
    No other fields apply.
  - **`environment-credential-secret`**: `azure.clientID` and `azure.tenantID` are *required* - a
    secret-based service principal always has a client ID and tenant. The client secret itself is
    never a CR field - it reaches the agent only as a `Secret`-backed environment variable (see
    Story 2), so it's absent from this list entirely, not merely optional.
  - **`environment-credential-certificate`**: `azure.clientID` and `azure.tenantID` are *required*.
    The certificate, its password, and whether to send the certificate chain are never CR fields -
    all three travel together as `Secret`-backed environment variables (see Story 2), since
    `clientSendCertChain` only means anything alongside the certificate it modifies.
  - **`workload-identity-credential`**: `azure.clientID` is *required* - unlike Managed Identity,
    there is no system-assigned equivalent for Workload Identity Federation (see [Managed cluster
    prerequisites](#managed-cluster-prerequisites)), so a specific identity must always be named.
    `azure.tenantID` and `azure.federatedTokenFile` are *optional*: both are normally auto-injected
    onto the pod by the Azure Workload Identity mutating webhook, and these fields exist only to
    override that.
- **`ClusterManager.spec.registrationConfiguration.registrationDriver.azure`** (hub):
  `azure.autoApprovedIdentityPatterns` (optional, a list of regex patterns matched against a joining
  cluster's Azure AD object ID for auto-approval), mirroring `awsirsa`'s `autoApprovedARNPatterns`;
  and `azure.oidcIssuerURL` (optional, unset for AKS's native Azure AD integration - set only when
  the hub apiserver instead trusts Azure AD as a generic OIDC issuer, to the exact
  `--oidc-issuer-url` value configured there, so `AzureAuthHubDriver` can reproduce the same
  default-prefixed username the apiserver derives - see [Changes required on the
  hub](#changes-required-on-the-hub)). There is no per-managed-cluster identity field here - the
  hub's configuration applies uniformly to every cluster that joins with `authType: azure`; a
  specific cluster's identity lives only on that cluster's own `Klusterlet`.

#### Container image dependency

The Workload Identity Federation and environment-credential paths make one outbound network call:
the Azure AD token exchange itself (`https://login.microsoftonline.com`), a normal
public-CA-signed TLS connection. Any base image used to build the affected binaries needs a
standard root CA trust bundle for this to succeed - a certificate-based registration flow never
needs to validate a public CA, so a minimal base image built for that flow can omit one without
anything failing until this specific code path is exercised.

The Managed Identity path instead calls the Azure Instance Metadata Service at the link-local
address `169.254.169.254` over plain HTTP - no public CA involved, but only reachable from inside
Azure's own network (a VM, an AKS node, or equivalent). This path fails in any other environment
regardless of CA trust, and its failure mode is distinct from a CA trust problem - worth surfacing
as a separate, clearly-labeled error rather than a generic connection failure.

### Workflow Details

Actors:
1. cluster-admin on the managed cluster
2. Azure AD administrator (configures the identity and, for Workload Identity Federation, its
   federated credential - out of band, before any cluster join is attempted)
3. cluster-admin on the hub cluster
4. hub controller
5. agent on the managed cluster

#### Managed cluster prerequisites

An Azure AD identity - a user-assigned managed identity, a system-assigned managed identity, or an
app registration - known in advance to the managed cluster administrator by its client ID and
object ID. System-assigned managed identities only work with the Managed Identity credential path;
Workload Identity Federation requires a user-assigned managed identity or an app registration, since
a system-assigned identity has no separate resource to attach a federated credential to.

For Workload Identity Federation specifically, a federated identity credential on that identity,
whose **Issuer** depends on which cluster actually runs the agent pod - not which cluster is being
managed:

- **`Default`/`Singleton`**: the agent pod runs on the managed cluster itself, so **Issuer** is the
  managed cluster's own OIDC issuer URL, and the Azure Workload Identity mutating webhook must be
  installed on the managed cluster.
- **`Hosted`/`SingletonHosted`**: the klusterlet agent runs entirely on a separate hosting cluster -
  the managed cluster is only ever reached remotely via an `external-managed-kubeconfig`, nothing
  from the agent runs on it. The projected ServiceAccount token is therefore signed by the
  **hosting cluster's** OIDC issuer, so **Issuer** must be the hosting cluster's issuer URL, and the
  mutating webhook must be installed on the hosting cluster instead. Using the managed cluster's
  issuer here doesn't degrade gracefully - Azure AD rejects the token exchange outright, since the
  real token's `iss` claim never matches what the federated credential trusts.

In every mode, the **Subject** is `system:serviceaccount:<agent-namespace>:<klusterlet-name>-registration-sa`,
plus a second federated credential with subject
`system:serviceaccount:<agent-namespace>:<klusterlet-name>-work-sa` (in `Singleton`/`SingletonHosted`
mode, registration and work share one ServiceAccount, so only the `-work-sa` subject is needed) -
`<agent-namespace>` is wherever the agent actually runs (the managed cluster's own namespace in
`Default`/`Singleton`; a namespace on the hosting cluster, typically named after the managed
cluster, in `Hosted`/`SingletonHosted`). **Audience** is `api://AzureADTokenExchange` in every mode.

This subject-matching is structurally the same idea as any federated-identity trust condition scoped
to a specific Kubernetes ServiceAccount - the identity can only be assumed by a token presented by
that exact ServiceAccount, in that exact namespace, nothing broader. Getting the issuer wrong for a
mode other than the default is exactly the kind of silent, mode-specific misconfiguration called out
in [Risks and Mitigation](#risks-and-mitigation) - it fails at registration time, not at deploy time,
and unlike a bad pod label or ServiceAccount annotation, there's no fallback credential source to
silently fall through to.

#### Hub cluster prerequisites

The hub *controller* needs nothing beyond what it already has: creating/updating
`ClusterRoleBinding`/`RoleBinding` objects is covered by its existing `ServiceAccount` permissions,
and it never calls out to Azure itself.

The hub **API server**, however, must independently be configured, out of band, to authenticate the
Azure AD access token the agent presents - either via AKS's native Azure AD integration, or by
configuring a self-managed apiserver to trust Azure AD as a generic OIDC issuer.

**AKS's native Azure AD integration**: the hub AKS cluster must be created or updated with Azure AD
integration enabled (`az aks create/update --enable-aad`), and - specifically - **without** Azure
RBAC for Kubernetes Authorization (`--enable-azure-rbac`). That mode replaces Kubernetes-native RBAC
with Azure role assignments as the actual authorization mechanism, which would make the
`ClusterRoleBinding`/`RoleBinding` objects `AzureAuthHubDriver.CreatePermissions` creates
irrelevant to whether access is actually granted. With plain `--enable-aad` (Azure AD integration
for *authentication* only, standard Kubernetes RBAC for *authorization*), AKS maps the presented
identity's `oid` claim directly to the Kubernetes username, unprefixed - no `azure.oidcIssuerURL`
or further hub configuration is needed for this path, and `azure.tokenAudience` can be left unset to
take AKS's own default.

**A self-managed apiserver trusting Azure AD as a generic OIDC issuer**:

```text
--oidc-issuer-url=https://login.microsoftonline.com/<tenant-id>/v2.0
--oidc-client-id=<the audience configured as azure.tokenAudience>
--oidc-username-claim=oid
```

`--oidc-username-prefix` is deliberately left unset, taking kube-apiserver's default: since
`--oidc-username-claim` is set to something other than `email`, the resulting Kubernetes username is
`<oidc-issuer-url>#<oid>`, not the bare object ID - this default exists to avoid username collisions
across multiple trusted issuers. `AzureAuthHubDriver` accounts for this by constructing the same
prefixed form when binding RBAC (see [Changes required on the hub](#changes-required-on-the-hub)),
using the `azure.oidcIssuerURL` value configured on the hub. That value must be byte-for-byte
identical to the `--oidc-issuer-url` given here - even a trailing-slash difference produces a
username the RBAC bindings don't match, so the token still validates but every authorization check
then fails with a 403.

`--oidc-client-id` must equal whatever `azure.tokenAudience` is configured on the spoke (see [Config
surface](#config-surface)) - the apiserver only accepts tokens whose audience matches its configured
client ID. This is specific to the self-managed apiserver path: AKS's native Azure AD integration
has its own default audience (the well-known AKS AAD Server application `azure.tokenAudience`
defaults to), which does not apply when trusting Azure AD as a generic OIDC issuer instead.

`AzureAuthHubDriver.CreatePermissions` assumes this trust already exists - it only manages the RBAC
bindings on top of it, the same way the `csr` driver assumes the apiserver already trusts the CA
that signs its issued client certificates.

#### Cluster join initiated from the managed cluster

The join flow itself is the one the community already knows; azure-auth only changes what happens
around authentication:
- The klusterlet is deployed with `registrationConfiguration.registrationDriver.authType: azure`.
- The bootstrap call that creates `ManagedCluster` is unchanged - it still authenticates with the
  bootstrap kubeconfig's own shared, low-privilege ServiceAccount token, not yet an Azure AD token.
  Azure-auth's only addition at this step is setting the `managed-cluster-azure-identity` annotation to
  the configured Azure AD object ID.
- Once the hub accepts the request, the RBAC bindings created bind the Azure AD object ID as a `User`
  subject (see [Changes required on the hub](#changes-required-on-the-hub)) rather than relying on a
  certificate's Organization field.
- From that point on, the agent's kubeconfig `exec` plugin authenticates every call with a fresh Azure
  AD token fetched via `get-azure-token`, instead of a client certificate. Only the exec plugin's
  configuration (hub server address, CA data, command, and arguments) is ever persisted to the
  `hub-kubeconfig-secret` on the managed cluster - never a token, certificate, or other secret.

#### Un-join and cleanup

Un-join goes through the generic path the community already knows; azure-auth adds nothing to it.
The RBAC bindings removed there are the same ones azure-auth created on accept (see [Cluster join
initiated from the managed cluster](#cluster-join-initiated-from-the-managed-cluster)) - there's no
certificate to revoke, since none was ever issued, and nothing to clean up on the Azure side, since no
Azure resource was ever created by OCM in the first place.

### Risks and Mitigation

**Ownership and maintenance.** Azure-specific code requires Azure access and expertise to maintain,
which is why this proposal includes adding an `OWNERS` file scoped to the new package, naming a
dedicated maintainer for it distinct from the rest of the registration package.

**Silent failure mode across deployment shapes.** The pod label and ServiceAccount annotation that
enable Workload Identity Federation have to be correctly present on whichever manifest actually backs
the running agent pod, and that manifest differs across all four deployment shapes. Getting this wrong
for a shape other than the default doesn't produce a loud error - it silently falls through to a
different credential source, or fails much later, during actual registration rather than at deploy
time. Mitigated by explicit test coverage across all four shapes (see Test Plan).

**Shared, cross-binary credential state.** The `hub-kubeconfig-secret` written by one process (e.g. the
registration agent) may later be read by a different process (e.g. a separately-deployed work agent in
`Default` mode) that doesn't share the same binary. The exec plugin's command path in that persisted
config has to resolve correctly regardless of which binary reads it later - mitigated by every
relevant binary exposing `get-azure-token` at the same fixed path rather than assuming it can always
resolve its own executable path at write time.

**Duplicate identity across clusters.** Without a check, two different `ManagedCluster`s configured
with the same Azure AD object ID - administrator error, or a deliberately reused identity - would
each get their own RBAC bindings targeting the same Kubernetes `User` name, so that one identity
would end up with the union of both clusters' permissions, undermining the per-cluster isolation
described in Story 5. Mitigated by `CreatePermissions` itself refusing to bind a second cluster name
to an object ID already bound to a different one (see [Changes required on the
hub](#changes-required-on-the-hub)), on both the manual and automatic approval paths. This is a
check-then-act comparison against existing `ClusterRoleBinding`/`RoleBinding` objects rather than an
atomic, server-enforced constraint - two clusters with the same identity racing through
`CreatePermissions` at nearly the same instant could theoretically both pass the check before either
finishes creating its bindings. Given this requires either administrator error or a deliberately
reused identity to trigger at all, the residual race is treated as an accepted, low-probability edge
case rather than something warranting an atomic claim mechanism (e.g. a dedicated lock object keyed
by object ID) in this proposal.

### Test Plan

- Unit tests for each of the four `azure.credential` shapes (correct construction from valid input,
  and rejection of missing/invalid required fields per shape), and for the hub-side RBAC
  binding/approval logic, in isolation, without a real Azure identity - including `CreatePermissions`
  refusing to bind a second cluster name to an object ID already bound to a different one.
- Integration tests (envtest) for the hub driver's `CreatePermissions`/`Cleanup`/`Accept` behavior.
- Verification against a real Azure identity, since the specific failure modes here are about real
  credential exchange and real RBAC - not something envtest or a fake client exercises. Before this is
  marked `implementable`, this should cover, at minimum:
  - Both Managed Identity and genuine Workload Identity Federation, against a real hub and managed
    cluster.
  - All four klusterlet deployment shapes (`Default`, `Singleton`, `Hosted`, `SingletonHosted`).
  - Both manual and automatic approval.
  - A real `ManifestWork` sent from the hub and confirmed applied on the managed cluster in each
    configuration above - status conditions alone ("Joined", "Available") are not sufficient
    verification, since a cluster can report healthy status while still missing the specific
    permission needed to receive any actual work.

### Graduation Criteria

New driver, opt-in by construction (`authType: azure` must be explicitly selected; every other
driver's behavior is unaffected). Proposed to ship directly as a supported driver once the Test Plan
above passes and an `OWNERS` file is in place, rather than progressing through a separate alpha/beta
gate.

### Upgrade / Downgrade Strategy

No impact on existing clusters. A cluster already registered via any other driver is unaffected by
this driver's addition; nothing here changes a shared code path those drivers rely on.

### Version Skew Strategy

Not applicable in a way that differs from adding any other driver - a hub or spoke predating this
change simply doesn't offer `authType: azure` as an option.

## Implementation History

## Drawbacks

Adds a new external dependency (Azure's identity SDK) and a new package that needs ongoing,
Azure-specific maintenance the existing maintainer group cannot provide directly - the ownership model
above exists specifically to address this.

## Alternatives

- CSR remains the preferred approach to managed cluster authentication with the hub, where usable.
- A design where the hub controller dynamically provisions Azure cloud resources (identities, role
  assignments) per managed cluster was considered and rejected in favor of relying entirely on
  pre-established Azure AD trust, keeping the hub controller's responsibility limited to
  Kubernetes-native RBAC (see Non-Goals).
- **Make registration auth pluggable out-of-tree instead of adding another in-tree driver.** Rather
  than growing the set of cloud-specific drivers shipped inside `ocm` itself, OCM core could expose
  the `RegisterDriver`/`HubDriver` interfaces as a stable extension point and ship only the `csr`
  driver as the built-in implementation. Every cloud- or environment-specific driver - this one,
  the existing AWS IAM driver, gRPC, and anything added later - would live and be maintained in its
  own separately-versioned repository, built and released independently by whoever actually needs
  and understands that specific cloud's identity model. This would directly address the maintenance
  concern raised in [ocm#1676](https://github.com/open-cluster-management-io/ocm/issues/1676): each
  driver's owner would carry its own release cadence, its own CI, and its own on-call for issues
  specific to that cloud, without needing OCM core maintainers to have expertise in every cloud a
  driver exists for.

  Not pursued as part of this proposal because it's a much larger, cross-cutting change than adding
  one driver: it would mean designing a real plugin/extension mechanism where none exists today (how
  an out-of-tree driver is discovered, loaded, and kept compatible with a core interface that evolves
  independently of it), and would imply migrating the already-shipped AWS IAM and gRPC drivers out of
  the core repository as well, not just deciding where this new one lands. Worth considering
  separately, as its own enhancement, rather than deciding it implicitly as a side effect of this one.

## References

- Azure Identity client library for Go (`azidentity`) - package documentation:
  https://pkg.go.dev/github.com/Azure/azure-sdk-for-go/sdk/azidentity
- Azure Workload Identity overview:
  https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview
- ocm#1676 (original proposal and community discussion):
  https://github.com/open-cluster-management-io/ocm/issues/1676
