# Workload Completion

## Release Signoff Checklist

- [] Enhancement is `provisional`
- [] Design details are appropriately documented from clear requirements
- [] Test plan is defined
- [] Graduation criteria for dev preview, tech preview, GA
- [] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

This proposal introduces condition rules in manifestwork to extract status conditions from
resources it controls and orchestrate higher level behaviors. The first condition to be managed
in this way will be the `Complete` condition, which identifies that one or more of its workloads
has completed and should not be updated or recreated. Automated garbage collection can optionally
be enabled with a TTL to delete the entire manifestwork once all of its `Complete` condition rules
have been satisfied.

## Motivation

OCM currently does not have support for workloads that are expected to complete and be deleted
from spoke clusters. Often a user wants to run a workload exactly once, and then have the
resources they created be automatically garbage collected.

Examples:
- Job with `ttlSecondsAfterFinished` set would get cleaned up by the spoke's job controller after
  completion
- Pod with `restartPolicy` set to `Never` or `OnFailure` that eventually exits, and is deleted
  along with its node by the cluster-autoscaler

In both cases, OCM would recreate the deleted resources so long as the manifestwork still exists,
while the intention is most likely to only run them once.

There may be additional conditions in future that the manifestwork controller, or others built
on top of it, want to use as triggers for various behaviors. With that in mind, this feature is
designed in a generic way such that any condition can be specified.

## Proposal

Add new `ConditionRules` to `ManifestConfigs` to set certain conditions on the ManifestWork based
on its resources. Certain conditions may control higher level behaviors of the ManifestWork.
The first condition rule to be supported will be `Complete`, which identifies that a particular
workload is completable, and how to determine when it is done. A new field `TTLSecondsAfterFinished`
will be added to `DeleteOption` which enables garbage collection for the entire manifestwork after
all `Complete` conditions have been satisfied. If a TTL is set with no `Complete` condition rules,
then the manifestwork will never be considered finished and therefore never be garbage collected.
Likewise if `Complete` condition rules are set but not TTL, the manifestwork will complete but
not be automatically deleted.

Once a particular manifest has been marked as completed, it is no longer elligible to be updated or
recreated by the spoke controller, regardless of the completion status of other manifests in the
manifestwork. When all manifests with `Complete` condition rules have completed, the entire manifestwork is
considered to be complete (including manifests that did NOT specify `Complete` rules), and no more
updates will be applied.

If a user wishes to complete some of the manifests but not the entire manifestwork, all they would
need to do is set a `Complete` condition rule with the CEL expression `false` for at least
one other manifest to make the manifestwork incompletable.

Custom condition messages may be provided on condition rules as a static string or as a CEL expression
that evaluates to a string. If MessageExpression is set on a rule, in addition to the primary `object`
variable it will also be given a `result` boolean variable with the computed value of the CEL expressions
on the rule. If both Message and MessageExpression are set, the expression will take precedence as long
as it returns a non-empty string.

### Design Details

#### API change

To configure conditions of a manifestwork, `conditionRules` will be added to `manifestConfigs`

```go
type ManifestConfigOption struct {
    ...

    // ConditionRules defines how to set manifestwork conditions for a specific manifest.
    // +optional
    ConditionRules []ConditionRule `json:"conditionRules,omitempty"`
}

// +kubebuilder:validation:XValidation:rule="self.type != 'CEL' || self.condition != ''",message="Condition is required for CEL rules"
type ConditionRule struct {
    // Condition is the type of condition that is set based on this rule.
    // Any condition is supported, but certain special conditions can be used to
    // to control higher level behaviors of the manifestwork.
    // If the condition is Complete, the manifest will no longer be updated once completed.
    // Required for CEL rules, WellKnownCompletions will default to the "Complete" condition
    // +optional
    Condition string `json:"condition"`

    // Type defines how a manifest should be evaluated for a condition.
    // It can be CEL, or WellKnownCompletions.
    // If the type is CEL, user should specify the celExpressions field
    // If the type is WellKnownCompletions, certain common types in k8s.io/api will be considered
    // completed as defined by hardcoded rules.
    // +kubebuilder:validation:Required
    // +required
    Type ConditionRuleType `json:"type"`

    // CelExpressions defines the CEL expressions to be evaluated for the condition.
    // Final result is the logical AND of all expressions.
    // +optional
    CelExpressions []CelConditionExpressions `json:"celExpressions"`
    // Message is set on the condition created for this rule
    // +optional
    Message string `json:"message"`
    // MessageExpression uses a CEL expression to generate a message for the condition
    // Will override message if both are set and messageExpression returns a non-empty string.
    // Variables:
    // - object: The current instance of the manifest
    // - result: Boolean result of the CEL expressions
    // +optional
    MessageExpression string `json:"messageExpression"`
}

// +kubebuilder:validation:Enum=WellKnownCompletions;CEL
type ConditionRuleType string

const (
    // WellKnownCompletionsType represents a standard Complete condition for some common types, which
    // is reflected with a hardcoded rule for types in k8s.io/api
    WellKnownCompletionsType ConditionRuleType = "WellKnownCompletions"

    // CelConditionExpressionsType enables user defined rules to set the status of the condition
    CelConditionExpressionsType CompletionType = "CEL"
)

type CelConditionExpressions struct {
    // Expression represents the CEL expression to be evaluated on the manifest.
    // The expression must evaluate to a bool.
    // If the expression evaluates to any other type, the condition's status will be False.
    // Ref to https://kubernetes.io/docs/reference/using-api/cel/ on how to write CEL
    // Variables:
    // - object: The current instance of the manifest
    // +kubebuilder:validation:Required
    // +required
    Expression string `json:"expression"`
}
```

To enable automated garbage collection after completion, `ttlSecondsAfterFinished` will be added to `deleteOption`

```go
type DeleteOption struct {
  ...

  // TTLSecondsAfterFinished limits the lifetime of a ManifestWork that has been marked Complete
  // by one or more conditionRules set for its manifests. If this field is set, and
  // the manifestwork has completed, then it is elligible to be automatically deleted.
  // If this field is unset, the manifestwork won't be automatically deleted even afer completion.
  // If this field is set to zero, the manfiestwork becomes elligible to be deleted immediately
  // after completion.
  // +optional
  TTLSecondsAfterFinished *int64 `json:"ttlSecondsAfterFinished,omitempty"`
}
```

Additional named condition type

```go
const (
  ...

	// ManifestComplete represents that the manifest has completed and should no longer be updated
  ManifestComplete string = "Complete"
)
```

WellKnownCompletions rules for Job and Pod
```go
var jobConditionRule = workapiv1.ConditionRule{
  Condition: workapiv1.ManifestComplete,
  Type: workapiv1.CelConditionExpressionsType,
  CelExpressions: []workapiv1.CelConditionExpressions{
    {
      Expression: "object.status.conditions.filter(c, c.type == 'Complete' || c.type == 'Failed').exists(c, c.status == 'True')",
    },
  },
  MessageExpression: `result ? "Job is finished" : "Job is not finished`
}

var podConditionRule = workapiv1.ConditionRule{
  Condition: workapiv1.ManifestComplete,
  Type: workapiv1.CelConditionExpressionsType,
  CelExpressions: []workapiv1.CelConditionExpressions{
    {
      Expression: "object.status.phase in ['Succeeded', 'Failed']",
    },
  },
  MessageExpression: `result ? ("Pod " + object.status.phase) : "Pod is not finished`
}
```

#### evaluating condition rules

A condition for a particular manifest is considered "True" when the AND of all matching rules evaluates to true.
The status of a CelExpressions slice will be the AND of all given expressions.

Once all `Complete` condition rules have been satisfied for a manifest, it will no longer be evaluated in future
reconciliations, even if the values of the targeted object have changed such that it would no longer
be considered complete.

#### agent implementation

On each manifestwork reconciliation the agent will first check for the `Complete` condition.
If set, it will check if `ttlSecondsAfterFinished` has ellapsed, and if so delete the manifestwork
from the hub cluster.

Manifests that have already been marked `Complete` will be filtered out before applying.
Conditions will be evaluated on the result of apply. This means an update that would change the state of a
resource from complete to incomplete (based on CEL rules) will be applied before the condition is evaluated,
resulting in the condition being "False". Likewise an update that would change a resource from incomplete to
complete will result in the condition being evalutated to "True" after apply.

Top level conditions for the ManifestWork will be aggregated as the logical AND of
conditions speficied for all of its manifests.

Finally if the `Complete` condition is true on the top level conditions, the ManifestWork becomes
elligible for deletion. Garbage collection based on `ttlSecondsAfterFinished` will not be evaluated
until the next reconciliation, but the reconciler can requeue based on the TTL in order to schedule
GC at the appropriate time. If `ttlSecondsAfterFinished` is not set, then the ManifestWork will
be ignored until it is deleted by an external action.

#### examples

Run a job once

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  namespace: <target managed cluster>
  name: example-job
spec:
  manifestConfigs:
    - resourceIdentifier:
        resource: jobs
        name: some-job
        namespace: default
      conditionRules:
        - type: WellKnownCompletions
  workload: ...
```

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
      conditionRules:
        - type: WellKnownCompletions
  workload: ...
```

CEL expression conditions for a custom resource

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
      conditionRules:
        - condition: Complete
          type: CEL
          celExpressions:
           - expression: |
              object.status.conditions.filter(
                c, c.type == 'Complete' || c.type == 'Failed'
              ).exists(
                c, c.status == 'True'
              )
          messageExpression: |
            result ? "Object is complete" : "Object is not complete"
        - condition: MyCondition
          type: CEL
          celExpressions:
           - expression: |
              object.status.conditions.exists(
                c, c.type == 'MyCondition' && c.status == 'True'
              )
          messageExpression: |
            result
              ? object.status.conditions.filter(c, c.type == 'MyCondition')[0].message
              : "Object does not have MyCondition"
```

Status conditions

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  namespace: <target managed cluster>
  name: example-statuses
spec:
  ...
status:
  conditions:
  - lastTransitionTime: "2025-02-20T18:53:40Z"
    message: One or more manifests is not Complete
    reason: ConditionRulesFailed
    status: "False"
    type: Complete
  - lastTransitionTime: "2025-02-20T18:53:40Z"
    message: One or more manifests is not Initialized
    reason: ConditionRulesFailed
    status: "False"
    type: Initialized
  manifests:
    - conditions:
        - lastTransitionTime: "2025-02-20T19:12:22Z"
          message: Manifest is Complete
          reason: ConditionRulesPassed
          status: "True"
          type: Complete
      ...
    - conditions:
        - lastTransitionTime: "2025-02-20T18:53:40Z"
          message: Manifest is not Complete
          reason: ConditionRulesFailed
          status: "False"
          type: Complete
        - lastTransitionTime: "2025-02-20T18:53:40Z"
          // CEL error
          message: "failed to evaluate: no such key: initialzed"
          reason: ConditionRulesFailed
          status: "False"
          type: Initialized
      ...
```

### Test Plan
- test that jobs and pods that are completed with WellKnownCompletions and no ttl are no longer updated
- test that a workload that is completed with a ttl will be deleted after the time has ellapsed
- test that a workload with custom CelExpressions condition rules are evaluated correctly
- test that CEL evaluation errors are set in condition messages

### Graduation Criteria
N/A

### Upgrade Strategy
It will need upgrade on CRD of ManifestWork on hub cluster, and upgrade of work agent on managed cluster.

When a user needs to use this feature with an existing `ManifestWork`, the user needs to update the `ManifestWork` to
add `conditionRules` for the `Complete` condition to any of the manifests that are required to complete
before the `ManifestWork` should be considered complete.

### Version Skew Strategy
- The new fields are optional, and if not set, the manifestwork will be handled the same as by previous versions
- Older versions of the work agent will ignore the newly added fields
