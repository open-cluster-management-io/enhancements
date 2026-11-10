# Extensible Cloud Provider Interface for Agent Configuration

## Release Signoff Checklist

- [X] Enhancement is `implementable`
- [X] Design details are appropriately documented from clear requirements
- [X] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

The OCM Spoke Agent frequently needs to auto-detect environment-specific metadata from the underlying managed cluster (such as the `SpokeClusterName`) to improve operator experience and reduce manual configuration overhead during large-scale deployments. However, directly querying vendor-specific APIs (e.g., OpenShift, AWS EKS, Google GKE) from within the core OCM agent violates the vendor-neutrality principles of the project.

This proposal introduces an extensible, plugin-based **Cloud Provider Interface**. It defines a generic Go interface and registry pattern that allows any cloud vendor to implement their own detection logic in isolated sub-packages without bloating or modifying the core `agent.go` logic.

## Motivation

As OCM adoption grows, operators are deploying the agent across highly diverse Kubernetes distributions. Currently, to ensure agents register with deterministic cluster names, operators must manually mount secrets containing the cluster name prior to agent deployment. While an existing `TODO` in the codebase suggests reading the OpenShift infrastructure struct to auto-detect the name, implementing this directly in the core agent creates a slippery slope of adding AWS, GCP, and Azure SDKs to the core agent binary.

### Goals

- Establish a vendor-neutral interface for retrieving environment-specific metadata.
- Prevent core OCM agent bloat by isolating vendor-specific API calls.
- Provide a standardized registry allowing operators to select their provider via a simple CLI flag (`--cluster-name-provider`).
- Implement the initial "OpenShift" provider as the first concrete use case.

### Non-Goals

- Replacing the existing fallback mechanisms (UUID generation or mounted secrets). The provider interface acts as an optional enhancement; if it fails or is unconfigured, existing fallback mechanisms will execute seamlessly.
- Implementing providers for every possible cloud vendor in the initial PR.

## Proposal

### User Stories

#### Story 1: Large-Scale OpenShift Deployment
An enterprise operator wants to deploy the OCM Spoke Agent across 50 OpenShift clusters using a single, unified GitOps manifest. They do not want to dynamically generate a unique `cluster-name` secret for each of the 50 deployments. By setting `--cluster-name-provider=openshift` in their unified manifest, the agent on each cluster dynamically queries the `config.openshift.io/v1/infrastructures` API, extracts its official cluster name, and registers with the Hub using that deterministic name automatically.

#### Story 2: Community Extensibility
A developer for a new Kubernetes distribution (e.g., a BareMetal provider) wants the OCM agent to automatically detect names specific to their hardware architecture. Instead of proposing a massive PR that modifies the core OCM agent execution flow, they simply write a 50-line package that implements the `ClusterProvider` interface and registers it.

### Risks and Mitigation

**Risk:** A poorly written provider plugin panics or hangs indefinitely, preventing the agent from bootstrapping.
**Mitigation:** The interface requires a `context.Context` to ensure the API queries can be bounded by timeouts. Additionally, the core agent captures provider errors. If a provider returns an error or empty string, the agent gracefully falls back to generating a UUID or reading from a secret, ensuring the agent always successfully starts.

## Design Details

We introduce a new `providers` package containing the core `ClusterProvider` interface and a central registry.

```go
package providers

import (
	"context"
	"sync"
	"time"

	"k8s.io/client-go/rest"
)

// ClusterProvider is an interface for auto-detecting cluster specific information
type ClusterProvider interface {
	DetectClusterName(ctx context.Context, config *rest.Config) (string, error)
}

var registry = make(map[string]ClusterProvider)
var registryMu sync.RWMutex

func Register(name string, p ClusterProvider) {
	registryMu.Lock()
	defer registryMu.Unlock()
	registry[name] = p
}

func GetProvider(name string) (ClusterProvider, bool) {
	registryMu.RLock()
	defer registryMu.RUnlock()
	p, exists := registry[name]
	return p, exists
}
```

Vendor-specific plugins are created as isolated sub-packages (e.g., `pkg/common/providers/openshift`). They implement the interface and register themselves via Go's `init()` function.

The core `AgentOptions` struct receives a new `ClusterNameProvider` field exposed via the `--cluster-name-provider` CLI flag. During initialization in `getOrGenerateClusterAgentID()`, the agent simply asks the registry for the provider:

```go
if o.ClusterNameProvider != "" {
    if provider, exists := providers.GetProvider(o.ClusterNameProvider); exists {
        config, err := clientcmd.BuildConfigFromFlags("", o.SpokeKubeconfigFile)
        if err == nil {
            ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
            defer cancel()
            if name, err := provider.DetectClusterName(ctx, config); err == nil && name != "" {
                clusterName = name
            }
        }
    }
}
```

### Test Plan

- **Unit Testing:** The `providers` package will be unit tested to ensure the registry properly stores and retrieves interfaces.
- **Provider Testing:** The `openshift` provider uses `k8s.io/client-go/dynamic`. We will inject `k8s.io/client-go/dynamic/fake` in unit tests to simulate successful lookups, empty responses, and missing CRD errors to verify the provider logic independently of the agent.
- **Agent Integration Testing:** The `AgentOptions` CLI flag parsing will be tested in `agent_test.go` to ensure `--cluster-name-provider` is properly captured.

*(Note: The code for this test plan has already been written and verified in a PoC PR).*

### Upgrade / Downgrade Strategy

This change is entirely backwards-compatible and opt-in. 
- Existing clusters upgrading to this version will experience no behavior changes because `--cluster-name-provider` will be empty by default, causing the agent to execute its historical UUID/secret fallback logic.
- Users wishing to utilize this feature on upgrade simply add `--cluster-name-provider=<provider-name>` to their agent deployment args.

### Version Skew Strategy

Because the feature only affects the initialization phase of the Spoke Agent (determining its own name before contacting the Hub), it does not impact Hub-to-Spoke API interactions and requires no corresponding changes to the Hub cluster. It is resilient to version skew.

## Alternatives

**Alternative 1: Hardcoding vendor APIs in `agent.go`**
We could implement a massive `switch` statement in the core agent that imports SDKs from OpenShift, AWS, Google, and Azure to execute specific logic. This was rejected because it violates OCM's vendor-neutral stance and significantly bloats the core agent binary with unnecessary third-party dependencies for environments it isn't running in.

**Alternative 2: Init Containers**
Operators could solve the cluster-name detection issue by writing their own custom `initContainers` that curl cloud-provider metadata endpoints and write the cluster name to the shared volume secret before the OCM agent starts. This was rejected because it forces every enterprise operator to "reinvent the wheel" and write complex bash scripts, reducing the out-of-the-box user experience of OCM.
