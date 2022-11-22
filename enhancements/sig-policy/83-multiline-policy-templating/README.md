# Support for Multiline YAML Values in Policy Templates

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
      [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

Policy templates are limited to one line of YAML and cannot produce data structures that extend beyond that. A new field
of `object-templates-raw` will be introduced so that a single multiline string of YAML with any form of templates can be
specified. The result of `object-templates-raw` after template execution must be in the format of `object-templates`.

## Motivation

Policy templates are limited to one line of YAML and cannot produce data structures that extend beyond that. Policy
template output was historically limited to just strings, integers, and booleans. This has recently improved with the
addition of the `toLiteral` template function, but this is complex to use when the data structures are large as they
must be all in one line of JSON.

Users also cannot use policy templates to define a variable set of objects using `range` to configure in a particular
`ConfigurationPolicy`. This leads to duplication within the `object-templates` array.

Lastly, one cannot use a `ConfigurationPolicy` to enforce a value on all objects of a certain type in a namespace. Being
able to use `range` would allow this.

### Goals

1. To provide a flexible way where policy templates aren't restricted to a single line of YAML.

### Non-Goals

N/A

## Proposal

A new field of `object-templates-raw` will be introduced so that a single multiline string of YAML with any form of
templates can be specified. The result of `object-templates-raw` after template execution must be in the format of
`object-templates`. A user cannot specify both `object-templates` and `object-templates-raw`.

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: machine-config
  namespace: policies
spec:
  disabled: false
  remediationAction: enforce
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: machine-config
        spec:
          remediationAction: enforce
          severity: high
          object-templates-raw: |
            {{- range (lookup "machine.openshift.io/v1beta1" "MachineSet" "openshift-machine-api" "").items }}
            - complianceType: musthave
              objectDefinition:
                apiVersion: machine.openshift.io/v1beta1
                kind: MachineSet
                metadata:
                  labels:
                    machine.openshift.io/cluster-api-cluster: {{ .metadata.name }}
                  name: {{ .metadata.name }}
                  namespace: openshift-machine-api
                spec:
                  {{- if eq .metadata.name "policy-grc-cp-autocla-j245q-worker-us-east-1c" }}
                  replicas: 2
                  {{- else }}
                  replicas: 1
                  {{- end }}
            {{- end }}
```

### User Stories

#### Story 1

As a policy user, I would like to use ranges in my policy templates to avoid duplication in my `object-templates`
definition.

#### Story 2

As a policy user, I would like use conditionals around arrays and objects so that I can avoid duplicating policies for
different environments.

### Implementation Details/Notes/Constraints [optional]

On every `ConfigurationPolicy` evaluation, the Configuration Policy controller will resolve the templates in the
`object-templates-raw` string and then store the unmarshaled result in the `object-templates` field. The rest of the
processing will continue as is.

The OpenAPI validation in the CRD must disallow specifying both `object-templates` and `object-templates-raw` in the
same `ConfigurationPolicy` manifest.

### Risks and Mitigation

### Open Questions [optional]

N/A

### Test Plan

**Note:** _Section not required until targeted at a release._

### Graduation Criteria

It would be GA in the release after implementation.

### Upgrade / Downgrade Strategy

There are no concerns with upgrades since this is not a breaking change and does not require user changes.

### Version Skew Strategy

## Implementation History

N/A

## Drawbacks

See the Risks section.

## Alternatives

N/A

## Infrastructure Needed [optional]

N/A
