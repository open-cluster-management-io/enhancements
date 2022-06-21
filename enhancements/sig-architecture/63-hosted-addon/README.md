# Running addons agents outside of the managed cluster

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in 
[website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal is for the addon framework to support running addons agents outside of the managed cluster.

## Motivation

Currently, all addons agents should be deployed on the managed cluster, it is mandatory to expose the hub cluster to 
the managed cluster even if the managed cluster resides in an insecure environment.

The proposal [hosted-deploy-mode](./../33-hosted-deploy-mode/README.md) introduced a new deploy mode `Hosted` to deploy 
klusterlet outside of the managed cluster, but keeps the addons agents remaining on the managed cluster, this proposal 
makes the addon framework support running addons agents outside of the managed cluster, which is defined as `Hosted` 
mode deploy for addons agents.

### Goals

- Make addon-framework deploy addons agents outside of the managed cluster.
- Define how to deploy an addon in hosted mode

### Non-Goals

- Implementation details of addon
- Support deploying all addons in hosted mode. Not all addons can run in hosted mode(ex: an addon agent 
needs to analyze some managed cluster node related information, which can only be collected in the managed cluster and 
cannot be exposed outside the managed cluster).

## Proposal

We propose to define an annotation of the `ManagedClusterAddon` for addon deployers to indicate that the addon is 
deployed in hosted mode and where agents deployments will be deployed. And provide a label for addon developers to 
specify which manifests need to be deployed outside the managed cluster.

### User Stories

#### Story 1

As a managed cluster administrator, I hope as few as possible workloads running in my cluster while the addon 
functionality is not affected.

#### Story 2

As a hub cluster administrator, on the premise that the addon functionality is not affected, I do not want the addon 
agent to talk to the hub cluster directly. 

### Terms

- `Hosted` mode: deploy addons agents deployments outside the managed cluster.
- `Hosting cluster`: The cluster where the addons agents deployments runn.

### Constraints

There are several constraints:
- The managed cluster must be exposed to the hosting cluster
- The managed cluster(Klusterlet) must be imported to hub in `Hosted` mode
- The hosting cluster must be a managed cluster of the hub, hosting cluster should be managed by the hub
- The hosting cluster must be the same cluster where the managed cluster klusterlet(registration-agent & work-agent) 
runs
- After the addon is deployed, the deployment mode cannot be switched

## Design Details

### For addon developers

We propose to add a label `addon.open-cluster-management.io/manifest-type` for the addon agent manifests to mark which 
manifests should be deployed where in Hosted mode:
- No matter in Default mode or Hosted mode, all manifests will be deployed on the managed cluster

- When the label does not exist or its value is `hosted-mode-managed`: the manifest will be deployed on the managed 
cluster in Hosted mode
- When the label's value is `hosted-mode-hosting`: the manifest will be deployed on the hosting cluster in Hosted mode
- When the label's value is `hosted-mode-none`: the manifest will not be deployed in Hosted mode


<table class="tg">
<thead>
  <tr>
    <th class="tg-0pky" rowspan="2"><br>Label Value</th>
    <th class="tg-c3ow">Default Mode</th>
    <th class="tg-c3ow" colspan="2">Hosted Mode</th>
    <th class="tg-c3ow" rowspan="2"><br>Example</th>
  </tr>
  <tr>
    <th class="tg-c3ow">Managed Cluster</th>
    <th class="tg-c3ow">Managed Cluster</th>
    <th class="tg-c3ow">Hosting Cluster</th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0pky"><span style="font-weight:400;font-style:normal">nonexist | hosted-mode-managed</span></td>
    <td class="tg-c3ow">Y</td>
    <td class="tg-c3ow">Y</td>
    <td class="tg-c3ow"></td>
    <td class="tg-c3ow">CRD owned by the addon</td>
  </tr>
  <tr>
    <td class="tg-0pky"><span style="font-weight:400;font-style:normal">hosted-mode-hosting</span></td>
    <td class="tg-c3ow">Y</td>
    <td class="tg-c3ow"></td>
    <td class="tg-c3ow">Y</td>
    <td class="tg-c3ow">deployment</td>
  </tr>
  <tr>
    <td class="tg-0pky"><span style="font-weight:400;font-style:normal">hosted-mode-none</span></td>
    <td class="tg-c3ow">Y</td>
    <td class="tg-c3ow"></td>
    <td class="tg-c3ow"></td>
    <td class="tg-c3ow"></td>
  </tr>
</tbody>
</table>

### For addon deployers

Once the managed cluster is imported to the hub in `Hosted` mode, we propose to add an annotation 
`addon.open-cluster-management.io/hosting-cluster-name` for `ManagedClusterAddon` to 
indicate the addon should be deployed in Hosted mode, and the value of the annotation is the hosting cluster which 
should:
- be a managed cluster of the hub as well
- be the same cluster where the managed cluster klusterlet(registration-agent & work-agent) runs

#### How does the addon agent connect to the managed cluster?

Addon agents deployments will be deployed in the `{ManagedClusterAddon.Spec.InstallNamespace}` namespeace, and the 
addon deployer is required to provide a secret, which name is `{addon-name}-managed-kubeconfig` and contains the key 
`kubeconfig` to connect to the managed cluster, in the `{ManagedClusterAddon.Spec.InstallNamespace}` namespace on the 
hosting cluster.

Some potential ways to provide the secret:
- The addon deployer asks the managed cluster administrator for a specific permission kubeconfig, then create it on the 
hosting cluster. Because the kubeconfig is generated by the managed cluster administrator, the addon developer does not 
need to deploy any service-account, role, rolebinding on the managed cluster, labels these resources 
`addon.open-cluster-management.io/manifest-type: hosted-mode-none`
- The addon developer deploys service-account, role, rolebinding on the managed cluster, labels these resources 
`addon.open-cluster-management.io/manifest-type: hosted-mode-managed`(or does not add this label, keep this label 
non-exist), then develops a component to get the service account on the remote managed cluster, and retrieve the token 
back to the hosting cluster to generate a kubeconfig.

### Architecture

When deploying an addon, which is built with the addon framework and marked with the label where each manifest needs to 
be deployed, the addon-controller will package the manifests with manifestwork and create them into the corresponding 
hosting/managed namespace according to the label, then work-agents will apply these manifests to the correct cluster.

```
               +------------------------------------------------------------+                                  
               | hub-cluster                                                |                                  
               |  +------------------------------------------------+        |                                  
               |  |             hello-addon-controller             |        |                                  
               |  +------------------------------------------------+        |                                  
               |                                                            |                                  
               |  +---------------------+   +----------------------+        |                                  
               |  | ns: managed-cluster |   | ns: hosting-cluster  |        |                                  
               |  |  +-----------------+|   | +-------------------+|        |                                  
               |  |  |manifestwork:    ||   | |manifestwork:      ||        |                                  
       |-------------- role            ||   | | deployment:       ------------------|                          
       |       |  |  | rolebinding     ||   | |  hello-addon-agnet||        |       |                          
       |       |  |  +-----------------+|   | +-------------------+|        |       |                          
       |       |  |          ...        |   |          ...         |        |       |                          
       |       |  +---------------------+   +----------------------+        |       |                          
       |       +--------------------------------------^---------------------+       |apply                     
       |                                              |                             |                          
       |       +--------------------------------------|---------------------+       |                          
       |       | hosting-cluster                      |connect              |       |                          
       |       |                                      |                     |       |                          
       |       |  +---------------------+  +---------------------+          |       |                          
       |       |  | deployment:         |  | deployment:         |          |       |                          
  apply|       |  |  registration-agent |  |  hello-addon-agent  |          |       |                          
       |       |  |  work-agent         |  |                     <-------------------                          
       |       |  |                     |  |                     |          |                                  
       |       |  +---------------------+  +---------------------+          |                                  
       |       |                                      |connect              |                                  
       |       +--------------------------------------|---------------------+                                  
       |                                              |                                                        
       |       +--------------------------------------v---------------------+                                  
       |       | managed-cluster                                            |                                  
       |       |                                                            |                                  
       |       | +---------------------+   +---------------------+          |                                  
       |       | |CRDs:                |   |Configurations:      |          |                                  
       |       | | clusterClaim        |   | role,rolebinding,sa |  etc.    |                                  
       |       | | appliedmanifestwork |   +----------^----------+          |                                  
       |       | +---------------------+              |                     |                                  
       |       +--------------------------------------|---------------------+                                  
       |                                              |                                                        
       |                                              |                                                        
       -----------------------------------------------|                                                        
```

### Guidance for transforming Default mode addon to hosted mode

Here are some guidelines for transforming addons, which are built with addon-framework in Default mode, to Hosted mode. 
After transforming, the addon should be deployed successfully either in Default mode or Hosted mode.

1. Consider carefully whether the addon can run in Hosted mode. Will addon functions be affected when addon agent is 
running outside the managed cluster? If the answer is no, stop.
2. Refactor the code of the addon agent to accept a flag like `managed-kubeconfig`, and use the kubeconfig received 
with the flag to build the managed cluster kube client, and use this client to process resources on the managed cluster.
3. Refactor the deployment manifest of the addon agent, use `if...end...` statement to add the `managed-kubeconfig` 
flag only in the Hosted mode deploying.
```
      ...
      volumes:
      {{- if eq .InstallMode "Hosted" }}
      - name: managed-kubeconfig-secret
        secret:
          secretName: helloworld-managed-kubeconfig
      {{- end }}
      ...
      containers:
      - name: helloworld-agent
        args:
          ...
          {{- if eq .InstallMode "Hosted" }}
          - "--managed-kubeconfig=/managed/config/kubeconfig"
          {{- end }}
          ...
        volumeMounts:
          {{- if eq .InstallMode "Hosted" }}
          - name: managed-kubeconfig-secret
            mountPath: "/managed/config"
            readOnly: true
          {{- end }}
          ...
```
4. Consider each agent manifest need to be deployed where in Hosted mode, and label them with 
`addon.open-cluster-management.io/manifest-type`, Usually, deployments need to be deployed in the hosting 
cluster and CRDs owned by the addon need to be deployed in the managed cluster.
5. Service account, (cluster)role, (cluster)rolebinding are a little special. In general, there is only one copy of 
these resources in default mode, but now we need to consider splitting them into two copies, one for the hosting 
cluster (permissions for running the controller, such as operating configmaps, leases, and events), and another for the 
managed cluster (permissions for the agent to operate the managed cluster resources)
6. Test the addon deploying and deleting in Hosted mode AND Default mode, make sure it can work on both mode. Default 
mode can be seen as a special Hosted mode when the managed cluster itself is treated as the hosting cluster.

### Test Plan

E2E tests can use kind to create two clusters to test, one cluster acts as hub and hosting cluster(self-join), and the 
other acts as the managed cluster, an example should be created using addon framework for the purpose of testing, and 
the addon agent deployment can be successfully installed on the hosting cluster.

### Graduation Criteria

Alpha:
* Hosted mode addon is implemented and works well on KinD cluster for at least one addon
* E2E tests exists

## Alternatives

1. Change the ManagedClusterAddon API to add a field `DeployOption` to describe the deploy option(mode) of the addon.

2. Registration-agent generates the external-managed-kubeconfig-<addon-name> for addon agent
