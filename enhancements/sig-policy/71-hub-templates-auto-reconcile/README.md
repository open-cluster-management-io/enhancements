# Automatic Reconciliation When Hub Template Referred Object is Updated

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in
      [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

Policies support dynamic values through policy templating. One such option is Hub policy templating, which are templates
that are executed on the Hub before the policy is distributed to the managed clusters. Hub policy templating supports
Kubernetes API queries such as getting a field from a ConfigMap. For example,
`{{hub fromConfigMap "" "my-configmap" "my-field" hub}}`.

The current limitation is that if a referenced object in the Hub policy template is updated, it does not immediately
cause the policy to be reconciled with this update. This does happen every ten hours or every time the policy is
reconciled such as when the policy status is updated or the policy applies to a new cluster. A workaround exists where
the `policy.open-cluster-management.io/trigger-update` annotation can be updated on the policy to cause the policy to be
reconciled immediately.

The current options that trigger the policy to be reconciled are insufficient and it should happen automatically and
immediately when the referenced object is updated.

## Motivation

A common use-case for Hub policy templates is to copy a Secret from the Hub to a managed cluster. If the Secret on the
Hub is updated, it is desired for the Secret to be immediately updated on the managed cluster. This isn’t possible
without the `policy.open-cluster-management.io/trigger-update` annotation being manually updated or waiting for up to
[ten hours](https://github.com/kubernetes-sigs/controller-runtime/blob/02dc46444cfe0e8bbe61bb3d4c3a12a0faadcd24/pkg/manager/manager.go#L109-L134).

### Goals

1. A policy is automatically reconciled when it has a Hub policy template that references an object that was updated.
1. The implementation efficiently uses Kubernetes API watches to achieve this.

### Non-Goals

N/A

## Proposal

The Policy Propagator, which is the controller on the Hub that executes the Hub policy templates, should watch all
individual objects that are referenced in Hub policy templates. Whenever a referenced object changes, the policy should
be reconciled.

### User Stories

#### Story 1

As a Policy addon user, I want my policies to automatically reconcile immediately when a Hub policy template references
an object that was updated.

### Implementation Details/Notes/Constraints [optional]

#### New Client Library

A new library should be written to dynamically watch objects and notify the consumer through a
[client-go rate limiting queue](https://pkg.go.dev/k8s.io/client-go@v0.23.5/util/workqueue?utm_source=gopls#NewNamedRateLimitingQueue),
which is the same queue used by the controller-runtime library. Note that the reason that this is a library is so that
this functionality can be used in other parts of the Policy addon such as the Configuration Policy controller if it
becomes event-driven. It could also be useful for the open-source community.

The queue in this new library would output ObjectIdentifier objects (shown below) that reference the watcher object
(e.g. Policy) that is watching dependent objects (e.g. Secret or ConfigMap) whenever a dependent object changes. The
[client-go rate limiting queue](https://pkg.go.dev/k8s.io/client-go@v0.23.5/util/workqueue?utm_source=gopls#NewNamedRateLimitingQueue)
is well suited for this use-case because it only maintains a queue of unique objects, so if a watched object updates
several times in a short period of time, this will greatly reduce the number of queue entries and therefore, greatly
reduce the work on the consumer side.

As shown below, the
[client-go rate limiting queue](https://pkg.go.dev/k8s.io/client-go@v0.23.5/util/workqueue?utm_source=gopls#NewNamedRateLimitingQueue)
will be abstracted away and the consumer of the library would implement a struct using the Reconciler interface. The
Reconcile method would behave exactly as the controller-runtime Reconcile method, but it instead receives an
ObjectIdentifier object rather than just the namespace and name. The consumer of the library can easily add or update
watches by calling the AddOrUpdateWatcher method and can remove watches with the RemoveWatcher method.

```go
import (
   "context"

   "k8s.io/client-go/tools/cache"
   "sigs.k8s.io/controller-runtime/pkg/ratelimiter"
   "sigs.k8s.io/controller-runtime/pkg/reconcile"
)

type Reconciler interface {
   Reconcile(ctx context.Context, watcher ObjectIdentifier) (reconcile.Result, error)
}


// Options are the arguments for creating a new DynamicWatcher.
type Options struct {
   // Reconciler reconciles the watched object.
   Reconciler Reconciler

   // RateLimiter is used to limit how frequently requests may be queued.
   // Defaults to controller-runtime's MaxOfRateLimiter which has both overall and per-item rate limiting.
   // The overall is a token bucket and the per-item is exponential.
   RateLimiter ratelimiter.RateLimiter
}

// Identifies an object on the API.
type ObjectIdentifier struct {
   Group     string
   Version   string
   Kind      string
   Namespace string
   Name      string
}

// DynamicWatcher implementations enable a consumer to be notified of updates to Kubernetes objects that other
// Kubernetes objects subscribe to.
type DynamicWatcher interface {
   // Add or update watches for the watcher. When updating, any previously watched objects not specified will stop
   // being watched.
   AddOrUpdateWatcher(watcher ObjectIdentifier, watchedObjects ...ObjectIdentifier)
   // Removes a watcher and any of its API watches solely owned by the watcher.
   RemoveWatcher(watcher ObjectIdentifier)
   // Start the reconcile loop with all the watches.
   Start(ctx context.Context) error
   // Returns the total number of active API watch requests which can be used for metrics.
   GetWatchCount() int
}

func New(options *Options) (*DynamicWatcher, error) {
   ...
}
```

The new library will deduplicate watches, meaning if two objects (e.g. a Policy) watch the same object (e.g. a
ConfigMap), then only one Kubernetes API watch request must be made. Watch API requests must also be limited to just the
objects being watched and not all objects in the namespace for efficiency and scale.

The new library would likely use the client-go dynamic Watch method since it is high level enough where retries,
backoffs, and unmarshaling is handled while not caching the results like a shared informer would. This is important
because there isn’t a need to cache the watched objects, but only to know when they have changed. If there are a lot of
watched objects, this will keep memory consumption low.

The implementation will need two data structures. The first needs to map watcher objects (e.g. a Policy) to the watches
it's "subscribed" to. This will be helpful when updating or removing watches. The second data structure maps the watched
objects to the watcher objects (e.g. a Policy). This will be helpful to populate the
[client-go rate limiting queue](https://pkg.go.dev/k8s.io/client-go@v0.23.5/util/workqueue?utm_source=gopls#NewNamedRateLimitingQueue)
when a watch notifies of an object updating. It will also be helpful to determine how many watcher objects (e.g a
Policy) are watching this object to determine if the watch API request should be canceled after the RemoveWatcher method
is called.

#### Using The Library

The Policy controller in the Governance Policy Propagator would initially be the only consumer of the library. To use
it, the controller would essentially have two event loops. The first is the existing reconcile loop that runs on updates
to Policy and Placement related objects. This event loop would be updated so that any time a root Policy object is
updated that uses Hub policy templates, dynamic watches would be added, updated, or removed using the new library. To do
this, the [go-template-utils](https://github.com/stolostron/go-template-utils) library would need to be updated to
return the objects it queried for on the API. The reconcile loop would then collect references to all objects that were
queried for using Hub policy templates for that root Policy object and call AddOrUpdateWatcher method from the library.
When the root Policy object is deleted, the RemoveWatcher method from the library is called.

The second (new) event loop would be the event loop from the new library, where anytime an object referenced in a Hub
policy template is updated, the root Policy would be reconciled. This would have the same logic as the first event loop,
and thus this new Reconcile method would just wrap the Reconcile method from the first event loop. Note that since these
event loops will be run as different Goroutines, locking should be added so that only one root Policy object is
reconciled at a time.

Note that the Governance Policy Propagator would also expose a new metric called `hub_templates_active_watch_count`
which would be a guage of how many active watch API requests are on going. This can help when measuring the performance
impact on the Kubernetes API. The data would come from the `GetWatchCount` method in the client library.

### Risks and Mitigation

One risk is that creating too many Kubernetes watch API requests could be expensive since we would need to create a
watch request per object to avoid client-side filtering. Measuring the CPU usage of the Kubernetes API once this is
implemented is important. If the CPU usage is too high, we could measure the performance of client-side filtering to see
if it performs better.

In this usage, there will be a limited amount of watches because in practice, many policies don’t use Hub policy
templates, and when they do, it is usually referencing one object multiple times.

If the performance test indicates high resource utilization on the Kubernetes API at a certain scale, this feature can
be disabled. The user will have to rely on setting an annotation manually to trigger the Policy to be reconciled.

### Open Questions [optional]

N/A

### Test Plan

**Note:** _Section not required until targeted at a release._

There will be unit tests within the new library and integration tests in the Policy Propagator controller.

### Graduation Criteria

It would be GA in the release after implementation.

### Upgrade / Downgrade Strategy

There are no concerns with upgrades since this is not a breaking change and does not require user changes.

### Version Skew Strategy

This is not applicable because the change is only on the Hub side.

## Implementation History

N/A

## Drawbacks

See the Risks section.

## Alternatives

### Configurable Periodic Evaluation

The Policy Propagator’s periodic reconcile of all policies currently set to
[ten hours](https://github.com/kubernetes-sigs/controller-runtime/blob/02dc46444cfe0e8bbe61bb3d4c3a12a0faadcd24/pkg/manager/manager.go#L109-L134)
could be made configurable so that users could make this more frequent. The issue is that this reconciles all policies
and not just the ones with Hub policy templates.

Alternatively, the Policy Propagator could requeue policies with Hub policy templates based on a configurable amount of
time so that only those policies are reconciled more frequently.

These options do make it better but users desire and expect an immediate reconciliation. Depending on how often
reconciliation happens, it could end up being too resource intensive on the Hub.

## Infrastructure Needed [optional]

N/A
