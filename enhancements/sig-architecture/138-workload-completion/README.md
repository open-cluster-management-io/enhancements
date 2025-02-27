# Workload Completion

## Release Signoff Checklist

- [] Enhancement is `provisional`
- [] Design details are appropriately documented from clear requirements
- [] Test plan is defined
- [] Graduation criteria for dev preview, tech preview, GA
- [] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal introduces completion rules in manifestwork to identify that one or more of its workloads has completed
and should not be updated or recreated. Automated garbage collection can optionally be enabled with a TTL to delete the entire manifestwork
once all of its completion rules have been satisfied.

## Motivation

OCM currently does not have support for workloads that are expected to complete and be deleted from spoke clusters.
Often a user wants to run a workload exactly once, and then have the resources they created be automatically garbage collected.

Examples:
- Job with `ttlSecondsAfterFinished` set would get cleaned up by the spoke's job controller after completion
- Pod with `restartPolicy` set to `Never` or `OnFailure` that eventually exits, and is deleted along with its node by the cluster-autoscaler

In both cases, OCM would recreate the deleted resources so long as the manifestwork still exists, while the intention
is most likely to only run them once.

## Proposal

Add new `CompletionRules` to `ManifestConfigs` to identify that a particular workload is completable,
and how to determine when it is done. `ManifestCondition` would be updated with a new `StatusCompletions`
containing the status of each completion rule for the manifest. A new field `TTLSecondsAfterFinished` will be
added to `DeleteOption` which enables garbage collection for the entire manifestwork after all completion rules
have been satisfied. If a TTL is set with no completion rules, then the manifestwork will never be considered
finished and therefore never be garbage collected. Likewise if completion rules are set but not TTL, the manifestwork
will complete but not be automatically deleted.

Once a particular manifest has been marked as completed, it is no longer elligible to be updated or recreated
by the spoke controller, regardless of the completion status of other manifests in the manifestwork. When all
manifests with completion rules have completed, the entire manifestwork is considered to be complete (including
manifests that did NOT specify completion rules), and no more updates will be made.

### Design Details

#### API change

To configure completion of a manifestwork, `completionRules` will be added to `manifestConfigs`

```go
type ManifestConfigOption struct {
    ...

    // CompletionRules defines when a manifest should be considered completed. A completed manifest
    // will no longer recieve updates or be recreated on deletion. If it is not set or empty,
    // the manifest will not be considered completable.
    // +optional
    CompletionRules []CompletionRule `json:"completionRules,omitempty"`
}

type CompletionRule struct {
    // Type defines the option of how a manifest should be considered completed.
    // It can be CEL, or WellKnownCompletions.
    // If the type is CEL, user should specify the celExpressions field
    // If the type is WellKnownCompletions, certain common types in k8s.io/api will be considered
    // completed as defined by hardcoded rules
    // +kubebuilder:validation:Required
    // +required
    Type CompletionType `json:"type"`

    // CelExpressions defines the CEL expressions to be evaluated for completion
    // +optional
    CelExpressions []CelCompletionExpressions `json:"celExpressions"`
}

// +kubebuilder:validation:Enum=WellKnownCompletions;CEL
type CompletionType string

const (
    // WellKnownCompletionType represents standard completions for some common types, which
    // is reflected with a hardcoded rule only for types in k8s.io/api
    WellKnownCompletionsType CompletionType = "WellKnownCompletions"

    CelCompletionExpressionsType CompletionType = "CEL"
)

type CelCompletionExpressions struct {
    // Expression represents the CEL expression to be evaluated on the manifest.
    // The expression must evaluate to a single value  in the type of integer, bool or string.
    // If the expression evaluates to a structure, map or slice, the manifest will not be considered complete.
    // Ref to https://cel.dev/ on how to write CEL
    // +kubebuilder:validation:Required
    // +required
    Expression string `json:"expression"`
    // Value represents the expected value of the expression to be considered complete.
    // The value will be parsed as an integer, bool, or string depending on the type of the field retrieved from path.
    // If the value is empty and expression evaluates to bool, it will be compared to `true` by default
    // If parsing to the appropriate type fails, the manifest will not be considered complete.
    // +optional
    Value string `json:"value"`
}
```

To enable automated garbage collection after completion, `ttlSecondsAfterFinished` will be added to `deleteOption`

```go
type DeleteOption struct {
  ...

  // TTLSecondsAfterFinished limits the lifetime of a ManifestWork that has satisfied
  // all completionRules configured for its manifests. If this field is set, and
  // the manifestwork has completed, then it is elligible to be automatically deleted.
  // If this field is unset, the manifestwork won't be automatically deleted even afer completion.
  // If this field is set to zero, the manfiestwork becomes elligible to be deleted immediately
  // after completion.
  // +optional
  TTLSecondsAfterFinished *int64 `json:"ttlSecondsAfterFinished,omitempty"`
}
```

To track completion of individual manifests, `statusCompletions` will be added to `status.manifests`

```go
type ManifestCondition struct {
    ...

    // StatusCompletion represents the completion status of rules defined for the manifest
    // +optional
    StatusCompletions []StatusCompletionResult `json:"statusCompletion,omitempty"`
}

type StatusCompletionResult struct {
    // CompletionTime represents that time at which a completion rule was first observed to be completed
    CompletionTime *metav1.Time `json:"completionTime"`
}
```

Additionally, once all `CompletionRules` for a manifest have been satisfied, a new `Condition` will be added to `status.manifests.conditions`,
and to `status.conditions` once all completable manifests have been completed.

```yaml
status:
  conditions:
  - lastTransitionTime: "2025-02-20T18:53:40Z"
    message: All manifests completed
    reason: ManifestWorkCompleted
    status: "True"
    type: Complete
  manifests:
    - conditions:
        - lastTransitionTime: "2025-02-20T18:53:40Z"
          message: Manifest completion rules passed
          reason: ManifestCompleted
          status: "True"
          type: Complete
      ...
```

WellKnownCompletions rules for Job and Pod
```go
var jobCompletionRule = workapiv1.CompletionRule{
  Type: workapiv1.CelCompletionExpressionsType,
  CelExpressions: []workapiv1.CelCompletionExpressions{
    {
      Expression: ".status.conditions.filter(c, c.type == 'Complete' || c.type == 'Failed').exists(c, c.status == 'True')",
    },
  }
}

var podCompletionRule = workapiv1.CompletionRule{
  Type: workapiv1.CelCompletionExpressionsType,
  CelExpressions: []workapiv1.CelCompletionExpressions{
    {
      Expression: ".status.phase in ['Succeeded', 'Failed']",
    },
  }
}
```

#### evaluating completion rules

Completion for a particular manifest is considered true when the AND of all rules evaluates to true.
The completion of a CelExpressions slice will be the AND of all given expressions.

Once all completion rules have been satisfied for a manifest, it will no longer be evaluated in future
reconciliations, even if the values of the targeted object have changed such that it would no longer
be considered complete.

#### agent implementation

On each manifestwork reconciliation the agent will first check for the `Complete` condition. If set,
it will check if `ttlSecondsAfterFinished` has ellapsed, and if so delete the manifestwork from the hub cluster.

Next manifests that have been marked completed will be filtered out of the manifests slice before
applying. Completion will be evaluated by Appliers in order to reduce API traffic on the spoke,
since they already must get the existing resource in almost all cases. ServerSideApply currently
only issues a GET if a hash comparison is required, and this change will add an additional case
where it must first fetch the existing resource before applying.

If a resource is determined to be completed then it will not be updated or created, regardless of
the update strategy, and the existing object will be returned along with an error of type
`CompletedManifestError`. The `manifestworkReconciler` will check for this error type when creating
status conditions for the manifest, and ignore it when creating the final list of errors.

Finally if all manifests with completion rules are marked completed it will set the `Complete`
condition on the top level `status.conditions` for the manifestwork. Garbage collection based
on `ttlSecondsAfterFinished` will not be evaluated until the next reconciliation, but the reconciler
will set requeue based on the TTL in order to schedule GC at the appropriate time.

#### examples

Run a job once

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  namespace: <target managed cluster>
  name: example-job
spec:
  workload: ...
  manifestConfigs:
    - resourceIdentifier:
        resource: jobs
        name: some-job
        namespace: default
      completionRules:
        - type: WellKnownCompletions
status:
  resourceStatus:
    manifests:
      - resourceMeta:
          resource: jobs
          name: some-job
          namespace: default
          ...
        statusCompletions:
          - completionTime: nil

Run a job once and delete all associated manifests on completion

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  namespace: <target managed cluster>
  name: example-job
spec:
  deleteOption:
    ttlSecondsAfterFinished: 30
  manifestConfigs:
    - resourceIdentifier:
        resource: jobs
        namespace: default
        name: some-job
      completionRules:
        - type: WellKnownCompletions
  workload: ...
```

Identify custom completion states for any arbitrary resource

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  namespace: <target managed cluster>
  name: example-custom-resource
spec:
  workload: ...
  manifestConfigs:
    - resourceIdentifier:
        resource: mycustomresources
        namespace: default
        name: some-custom-resource
      completionRules:
        - type: CEL
          celExpressions:
           - expression: |
              .status.conditions.filter(
                c, c.type == 'Complete' || c.type == 'Failed'
              ).exists(
                c, c.status == 'True'
              )
```

### Test Plan
- test that jobs and pods that are completed with WellKnownCompletions and no ttl are no longer updated
- test that a workload that is completed with a ttl will be deleted after the time has ellapsed
- test that a workload with custom CelExpressions completion rules are evaluated correctly

### Graduation Criteria
N/A

### Upgrade Strategy
It will need upgrade on CRD of ManifestWork on hub cluster, and upgrade of work agent on managed cluster.

When a user needs to use this feature with an existing `ManifestWork`, the user needs to update the `ManifestWork` to
add `completionRules` to any of the manifests that are required to complete before the `ManifestWork` should be considered complete.

### Version Skew Strategy
- The new fields are optional, and if not set, the manifestwork will be handled the same as by previous versions
- Older versions of the work agent will ignore the newly added fields

## Alternatives

- Treat celCompletionExpressions more like metav1.LabelSelectorRequirement from LabelSelector.MatchExpression
  - Values would be a slice instead of a single value
  - Add an Operator field with options:
    - In
    - NotIn
    - Exists
    - DoesNotExist
  - Probably not really needed for CEL, since you can express all of these operators in CEL itself and then compare to a final boolean result.
    If it is decided to keep JSONPaths support in addition to CEL this this would make more sense, but otherwise it's not really necesssary
    and increases complexity of the design.
