# Add-on Configuration

## Release Signoff Checklist

- [] Enhancement is `implemented`
- [] Design details are appropriately documented from clear requirements
- [] Test plan is defined
- [] Graduation criteria for dev preview, tech preview, GA
- [] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal includes:

1. The add-on configuration reference in the add-on APIs to help add-on developers to locate their add-on configuration.
2. An add-on deployment configuration API, the add-on developers can choose to use this API to configure their add-on
deployment.

## Motivation

For some add-ons, they need to run with configuration, we want to provide a way to locate their configuration with
add-on APIs.

## Use cases

1. As a user, I want to specify a default nodeSelector/tolerations configuration for all my add-ons.
2. As a user, I want to specify nodeSelector/tolerations configuration for some of my add-ons.

## Proposal

We propose to enhance the `ClusterManagementAddOn` API to support declaring the add-on supported configuration types,
and introduce a way to reference the add-on configurations in the `ManagedClusterAddOn` API. 

### Design Details

Currently, we have a `ConfigCoordinates` in the `ClusterManagementAddOn` API. It helps to locate the add-on
configuration information.

```golang
// ClusterManagementAddOnSpec provides information for the add-on.
type ClusterManagementAddOnSpec struct {
    // addOnMeta is a reference to the metadata information for the add-on.
    // +optional
    AddOnMeta AddOnMeta `json:"addOnMeta"`

    // addOnConfiguration is a reference to configuration information for the add-on.
    // +optional
    AddOnConfiguration ConfigCoordinates `json:"addOnConfiguration"`
}

// ConfigCoordinates represents the information for locating the CRD and CR that configures the add-on.
type ConfigCoordinates struct {
    // crdName is the name of the CRD used to configure instances of the managed add-on.
    // This field should be configured if the add-on have a CRD that controls the configuration of the add-on.
    // +optional
    CRDName string `json:"crdName"`

    // crName is the name of the CR used to configure instances of the managed add-on.
    // This field should be configured if add-on CR have a consistent name across the all of the ManagedCluster
    // instaces.
    // +optional
    CRName string `json:"crName"`

    // lastObservedGeneration is the observed generation of the custom resource for the configuration of the add-on.
    // +optional
    LastObservedGeneration int64 `json:"lastObservedGeneration,omitempty"`
}
```

But the `ConfigCoordinates` has some limits:

- it only supports using `CustomResourceDefinition` as the add-on configuration
- it only supports cluster scope add-on configuration
- it only supports one kind of add-on configuration type

To make the add-on APIs more flexible for configuration, we will deprecate the `ConfigCoordinates` and add a new field
`supportedConfigs` to support declaring the add-on supported configuration types in the `ClusterManagementAddOn` API.

``` golang
type ClusterManagementAddOnSpec struct {
  // addOnMeta is a reference to the metadata information for the add-on.
  // +optional
  AddOnMeta AddOnMeta `json:"addOnMeta,omitempty"`

  // Deprecated: Use supportedConfigs filed instead
  // +optional
  AddOnConfiguration ConfigCoordinates `json:"addOnConfiguration,omitempty"`

  // supportedConfigs is a list of configuration types supported by add-on.
  // An empty list means the add-on does not require configurations.
  // The default is an empty list
  // +optional
  SupportedConfigs []ConfigMeta `json:"supportedConfigs,omitempty"`
}

// ConfigMeta represents a collection of metadata information for add-on configuration.
type ConfigMeta struct {
  // group and resouce of the add-on configuration.
  ConfigGroupResource `json:",inline"`

  // defaultConfig represents the namespace and name of the default add-on configuration.
  // In scenario where all add-ons have a same configuration.
  // +optional
  DefaultConfig *ConfigReferent `json:"defaultConfig,omitempty"`
}

// ConfigGroupResource represents the GroupResource of the add-on configuration
type ConfigGroupResource struct {
    // group of the add-on configuration.
    Group string `json:"group"`

    // resource of the add-on configuration.
    Resource string `json:"resource"`
}

// ConfigReferent represents the namespace and name for an add-on configuration.
type ConfigReferent struct {
    // namespace of the add-on configuration.
    // If this field is not set, the configuration is in the cluster scope.
    // +optional
    Namespace string `json:"namespace,omitempty"`

    // name of the add-on configuration.
    Name string `json:"name"`
}
```

In the `ManagedClusterAddOn` API, we will add a new field `configs` to support referencing the add-on configurations.

```golang
type ManagedClusterAddOnSpec struct {
  // installNamespace is the namespace on the managed cluster to install the add-on agent.
  InstallNamespace string `json:"installNamespace,omitempty"`

  // configs is a list of add-on configurations.
  // +optional
  Configs []AddOnConfig `json:"configs,omitempty"`
}

type AddOnConfig struct {
  // group and resource of add-on configuration.
  ConfigGroupResource `json:",inline"`

  // name and namespace of add-on configuration.
  ConfigReferent `json:",inline"`
}
```

and we will use a new field `configReferences` to represent the add-on current configuration references instead of the
`addOnConfiguration` in the `ManagedClusterAddOn` status.

```golang
type ManagedClusterAddOnStatus struct {
    // Deprecated: Use configReference instead
    // addOnConfiguration is a reference to configuration information for the add-on.
    // This resource is use to locate the configuration resource for the add-on.
    // +optional
    AddOnConfiguration ConfigCoordinates `json:"addOnConfiguration",omitempty`

    // configReferences is a list of current add-on configuration references.
    // +optional
    ConfigReferences []ConfigReference `json:"configReferences,omitempty"`
}

// ConfigReference is a reference to the current add-on configuration.
// This resource is used to locate the configuration resource for the current add-on.
type ConfigReference struct {
    // This field is synced from ClusterManagementAddOn configGroupResource field.
    ConfigGroupResource ConfigGroupResource `json:",inline"`

    // This field is synced from ClusterManagementAddOn defaultConfig and ManagedClusterAddOn config fields.
    // If both them are specified, the ManagedClusterAddOn config will overwrite the ClusterManagementAddOn
    // defaultConfig.
    Config ConfigReferent `json:",inline"`

    // lastObservedGeneration is the observed generation of the add-on configuration.
    LastObservedGeneration int64 `json:"lastObservedGeneration"`
}
```

We will introduce a `Configured` condition for the `ManagedClusterAddOn` to report the condition when we configure
an addon with its configuration.

```golang
// ManagedClusterAddOnCondtionConfigured represents that the addon agent is configured with its configuration
const ManagedClusterAddOnCondtionConfigured string = "Configured"
```

We will enhance the add-on framework, make it can detect the configuration changes automatically, and after the
configuration was changed, the add-on framework will re-render the add-on manifests.

The add-on framework will watch the `ManagedClusterAddOn` to get the add-on configuration references from the 
`ManagedClusterAddOn` status.

If the `ManagedClusterAddOn` has the configuration references, the add-on framework use the configuration references to
locate the configuration resources and watch them, once the configuration resources change, the add-on famework updates
the `ManagedClusterAddOn` status (`lastObservedGeneration`), this will trigger the
[`Manifests`](https://github.com/open-cluster-management-io/addon-framework/blob/main/pkg/agent/inteface.go#L31)
interface which add-on implement to render the add-on manifests.

As a specific case, we want to introduce a `AddOnDeploymentConfig` API to configure the add-on deployment.

This API is optional for add-on developers, add-on developers can choose to use this API to configure their add-on
deployment or use their own API for configuration.

```golang
// AddOnDeploymentConfig represents a deployment configuration for an add-on.
type AddOnDeploymentConfig struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    // spec represents a desired configuration for an add-on.
    // +required
    Spec AddOnDeploymentConfigSpec `json:"spec"`
}

type AddOnDeploymentConfigSpec struct {
    // CustomizedVariables is a list of name-value variables for the current add-on deployment.
    // The add-on implementation can use these variables to render the add-on deployment.
    // For example, using this to define the image information of one add-on agent:
    // `"customizedVariables": [
    //    {"name":"Image","value":"myimg:latest"},
    //    {"name":"ImagePullPolicy","value":"Always"}]`,
    // or using this to define a http proxy for one add-on agent:
    // `"customizedVariables": [
    //    {"name":"HTTP_PROXY","value":"http://proxy.example.com:3128"},
    //    {"name":"HTTPS_PROXY","value":"http://proxy.example.com:3128"},
    //    {"name": "NO_PROXY", "value":"svc,local"}]` 
    // The default is an empty list.
    // +optional
    CustomizedVariables []CustomizedVariable `json:"customizedVariables,omitempty"`

    // NodePlacement enables explicit control over the scheduling of the add-on.
    // +optional
    NodePlacement NodePlacement `json:"nodePlacement,omitempty"`
}

// CustomizedVariable represents a customized variable for add-on deployment.
type CustomizedVariable struct {
    // Name of this variable. Must be a C_IDENTIFIER.
    Name string `json:"name"`

    // Value of this variable.
    Value string `json:"value"`
}

// NodePlacement describes node scheduling configuration for the pods.
type NodePlacement struct {
    // NodeSelector defines which Nodes the Pods are scheduled on. The default is an empty list.
    // +optional
    NodeSelector map[string]string `json:"nodeSelector,omitempty"`

    // Tolerations is attached by pods to tolerate any taint that matches
    // the triple <key,value,effect> using the matching operator <operator>.
    // The default is an empty list.
    // +optional
    Tolerations []v1.Toleration `json:"tolerations,omitempty"`
}
```

Examples:

1. Using `AddOnDeploymentConfig` to configure add-on

    User have a default add-on deployment configuration on the hub

    ```yaml
    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: AddOnDeploymentConfig
    metadata:
      name: addon-default-placement
      namespace: open-cluster-management
    spec:
      nodeSelector:
        "kubernetes.io/arch": "amd64"
      tolerations:
      - key: "node.kubernetes.io/unreachable",
        operator: "Exists",
        effect: "NoSchedule"
    ```

    One add-on operator creates the `ClusterManagementAddOn` with the configuration referance on the hub to share this
    configuration to its add-ons

    ```yaml
    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: ClusterManagementAddOn
    metadata:
      name: myaddon
    spec:
      supportedConfigs:
      - group: addon.open-cluster-management.io
        resource: addondeploymentconfigs
        defaultConfig:
          name: addon-default-placement
          namespace: open-cluster-management
    ```

    For each managed cluster add-on, the configuration can be located with `configReferences` in the
    `ManagedClusterAddOnStatus`

    ```yaml
    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: ManagedClusterAddOn
    metadata:
      name: myaddon
      namespace: cluster1
    spec:
      installNamespace: open-cluster-management-agent-addon
    status:
      configReferences:
      - group: addon.open-cluster-management.io
        resource: addondeploymentconfigs
        name: addon-default-placement
        namespace: open-cluster-management
        lastObservedGeneration: 1

    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: ManagedClusterAddOn
    metadata:
      name: myaddon
      namespace: cluster2
    spec:
      installNamespace: open-cluster-management-agent-addon
    status:
      configReferences:
      - group: addon.open-cluster-management.io
        resource: addondeploymentconfigs
        name: addon-default-placement
        namespace: open-cluster-management
        lastObservedGeneration: 1
    ```


    If user need to configure some add-ons specially, the user can define a specifc configuration and specify the
    configuration reference in the `ManagedClusterAddOn`

    ```yaml
    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: AddOnDeploymentConfig
    metadata:
      name: addon-arm-placement
      namespace: cluster3
    spec:
      nodeSelector:
        "kubernetes.io/arch": "arm64"
      tolerations:
      - key: "node.kubernetes.io/unreachable",
        operator: "Exists",
        effect: "NoSchedule"

    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: ManagedClusterAddOn
    metadata:
      name: myaddon
      namespace: cluster3
    spec:
      installNamespace: open-cluster-management-agent-addon
      configs:
      - group: addon.open-cluster-management.io
        resource: addondeploymentconfigs
        name: addon-arm-placement
        namespace: cluster3
    ```

    then the configuration reference in this add-on status will be

    ```yaml
    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: ManagedClusterAddOn
    metadata:
      name: myaddon
      namespace: cluster3
    spec:
      installNamespace: open-cluster-management-agent-addon
      config:
        name: addon-arm-placement
    status:
      configReferences:
      - group: addon.open-cluster-management.io
        resource: addondeploymentconfigs
        name:  addon-arm-placement
        namespace: cluster3
        lastObservedGeneration: 1
    ```

2. Using developers's own configuration API

    If user's all add-ons share the same configuration, the add-on operator creates the `ClusterManagementAddOn` with
    the configuration referance on the hub to share this configuration to its add-ons

    ```yaml
    apiVersion: proxy.open-cluster-management.io/v1alpha1
    kind: ManagedProxyConfiguration
    metadata:
      name: cluster-proxy
    spec:
      authentication:
        signer:
          type: SelfSigned
      proxyServer:
        image: proxy-server-image
      proxyAgent:
        image: proxy-agent-image

    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: ClusterManagementAddOn
    metadata:
      name: cluster-proxy
    spec:
      supportedConfigs:
      - group: proxy.open-cluster-management.io
        resource: managedproxyconfigurations
        defaultConfig:
          name: cluster-proxy
    ```

    then the configuration reference in this add-on status will be

    ```yaml
    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: ManagedClusterAddOn
    metadata:
      name: cluster-proxy
      namespace: cluster1
    spec:
      installNamespace: open-cluster-management-agent-addon
    status:
      configReferences:
      - group: proxy.open-cluster-management.io
        resource: managedproxyconfigurations
        name: cluster-proxy
        lastObservedGeneration: 1
    ```

    If user's each add-on needs its own configuration, user set up the configuration reference in the `ManagedClusterAddOn`

    ```yaml
    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: ClusterManagementAddOn
    metadata:
      name: submariner
    spec:
      supportedConfigs:
      - group: submarineraddon.open-cluster-management.io
        resource: submarinerconfigs

    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: SubmarinerConfig
    metadata:
      name: submariner
      namespace: cluster1
    spec:
      IPSecIKEPort: 500
      IPSecNATTPort: 4500

    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: ManagedClusterAddOn
    metadata:
      name: submariner
      namespace: cluster1
    spec:
      configs:
      - group: submarineraddon.open-cluster-management.io
        resource: submarinerconfigs
        name: submariner
        namespace: cluster1

    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: SubmarinerConfig
    metadata:
      name: submariner
      namespace: cluster2
    spec:
      IPSecIKEPort: 501
      IPSecNATTPort: 4501
    
    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: ManagedClusterAddOn
    metadata:
      name: submariner
      namespace: cluster2
    spec:
      configs:
      - group: submarineraddon.open-cluster-management.io
        resource: submarinerconfigs
        name: submariner
        namespace: cluster2
    ```

    then the configuration reference in this add-on status will be

    ```yaml
    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: ManagedClusterAddOn
    metadata:
      name: submariner
      namespace: cluster1
    spec:
      config:
        name: submariner
        namespace: cluster1
    status:
      configReferences:
      - group: submarineraddon.open-cluster-management.io
        resource: submarinerconfigs
        name: submariner
        namesapce: cluster1
        lastObservedGeneration: 1

    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: ManagedClusterAddOn
    metadata:
      name: submariner
      namespace: cluster2
    spec:
      config:
        name: submariner
        namespace: cluster2
    status:
      configReferences:
      - group: submarineraddon.open-cluster-management.io
        resource: submarinerconfigs
        name: submariner
        namesapce: cluster2
        lastObservedGeneration: 1
    ```

3. Supporting multiple configurations for one add-on

    If one add-on has multiple configurations, its add-on operator creates the `ClusterManagementAddOn` with
    the supported configuration types on the hub

    ```yaml
    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: ClusterManagementAddOn
    metadata:
      name: cluster-proxy
    spec:
      supportedConfigs:
      - group: proxy.open-cluster-management.io
        resource: managedproxyconfigurations
        defaultConfig:
          name: cluster-proxy
      - group: addon.open-cluster-management.io
        resource: addondeploymentconfigs
    ```

    user set up the configuration reference in the `ManagedClusterAddOn`

    ```yaml
    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: ManagedClusterAddOn
    metadata:
      name: cluster-proxy
      namespace: cluster1
    spec:
      configs:
      - group: addon.open-cluster-management.io
        resource: addondeploymentconfigs
        name: deploy-config
        namespace: cluster1
    ```

    then the configuration reference in this add-on status will be

    ```yaml
    apiVersion: addon.open-cluster-management.io/v1alpha1
    kind: ManagedClusterAddOn
    metadata:
      name: cluster-proxy
      namespace: cluster1
    spec:
      configs:
      - group: addon.open-cluster-management.io
        resource: addondeploymentconfigs
        name: deploy-config
        namespace: cluster1
    status:
      configReferences:
      - group: proxy.open-cluster-management.io
        resource: managedproxyconfigurations
        name: cluster-proxy
        lastObservedGeneration: 1
      - group: addon.open-cluster-management.io
        resource: addondeploymentconfigs
        name: deploy-config
        namesapce: cluster1
        lastObservedGeneration: 1
    ```

### Test Plan
- All implementations will have thorough coverage of unit tests.
- Integration tests will cover the user cases

### Graduation Criteria
At first, This proposal will be in alpha stage and needs to meet
1. The new APIs is reviewed and accepted;
2. Implementation is completed to support the functionalities;
3. Develop test cases to demonstrate this proposal work correctly;

When below criteria is met, we will consider to promote this proposal
into beta stage.
1. at least 2-3 consumers adopt this API;
2. at lease 2-3 customized API adopt this mechanism to handle their
configuration;

#### Alpha Stage Graduation Criteria

### Upgrade Strategy
- We will deprecate the elder fields of `ConfigCoordinates` in `ClusterManagementAddOn`, the add-on developers need
adopt the new API if they used the `ConfigCoordinates` in `ClusterManagementAddOn`

### Version Skew Strategy
- For `ClusterManagementAddOn` API, we will deprecate the `ConfigCoordinates` elder fields in the `ClusterManagementAddOn`
spec.
- For `ManagedClusterAddOn` API, we will deprecate the `AddOnConfiguration` fields in the `ManagedClusterAddOn` status.

## Alternatives

### Using `ManagedClusterAddOn` values annotation

Currently, we support to use `ManagedClusterAddOn` values annotation to configure the add-on in the addon-framewok, to
support configure node selector and toleration, We introduce two annotations to set node selector and toleration for
an add-on.

1. Node selector annotation, the key of this annotation is: `addon.open-cluster-management.io/nodeSelector`, the value
of this annotation represents a node selector with json format
2. Tolerations annotation, the key of this annotation is: `addon.open-cluster-management.io/tolerations`, the value of
this annotation represents a toleration array with json format

Example:

```yaml
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: myaddon
  namespace: cluster1
  annotations:
    addon.open-cluster-management.io/nodeSelector: "{\"kubernetes.io/os\":\"linux\"}"
    addon.open-cluster-management.io/tolerations: "[{\"key\":\"node.kubernetes.io/unreachable\",\"operator\":\"Exists\",\"effect\":\"NoSchedule\"}]"
spec:
  installNamespace: open-cluster-management-agent-addon
```

In the addon-framework, there is a function [`GetValuesFromAddonAnnotation`](https://github.com/open-cluster-management-io/addon-framework/blob/v0.3.0/pkg/addonfactory/helper.go#L21),
it returns a [`Values`]([https://github.com/open-cluster-management-io/addon-framework/blob/v0.3.0/pkg/addonfactory/addonfactory.go#L22)
object (the object type is `map[string]interface{}`) that is built from the add-on values annotation, this object is used
to render the add-on deployment.

In this alternative, if one add-on has the node selector and tolerations annotations, this function will merge them to
the `Values` object with the key `NodeSelector` and `Tolerations`.

For an add-on implementation, developers use [`AgentAddonFactory`](https://github.com/open-cluster-management-io/addon-framework/blob/v0.3.0/pkg/addonfactory/addonfactory.go#L28)
to build their add-on, they use [`WithGetValuesFuncs`](https://github.com/open-cluster-management-io/addon-framework/blob/v0.3.0/pkg/addonfactory/addonfactory.go#L61)
to specify the get values functions (the function type is [`GetValuesFunc`](https://github.com/open-cluster-management-io/addon-framework/blob/v0.3.0/pkg/addonfactory/addonfactory.go#L24))
to render their add-on deployment, for get values function, the developers can choose to use addon-frameworks
`GetValuesFromAddonAnnotation` function or implement it by themselves to handle node selector and toleration for their
add-ons.

For upgrade, if there is an existing add-on CR that already used the add-on values annotation to specify the node
selector and tolerations and the add-on implementation used the `GetValuesFromAddonAnnotation` function to get them. The
add-on user can choose to keep them in add-on values annotation or use new add-on placement annotation to define them
and remove them from the add-on values annotation.

Pros:
- We do not need to change current API definitions.
- Addon developers can choose their add-on configuration implementation freely.

Cons:
- It's hard to validate the json string that is in the annotations.

### Adding the node selector and toleration to the `ManagedClusterAddOn` spec

As an alternative, we may add the node selector and toleration to the add-on spec, but this may not meet all needs, e.g.
if one add-on need to deploy more than one agent, and these agents have different node selector/toleration, and the
developers may not use this, e.g. the developers have their own add-on configuration CR and use the configuration to
render their add-on agents. So to keep the add-on flexibility, we recommend providing an optional way for the add-on
developers, they will be able to choose use the optional way or use their own way to handle the node
selector/tolerations for their add-ons.
