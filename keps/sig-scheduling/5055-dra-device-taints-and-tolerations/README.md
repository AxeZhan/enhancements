<!--
**Note:** When your KEP is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Pick a hosting SIG.**
  Make sure that the problem space is something the SIG is interested in taking
  up. KEPs should not be checked in without a sponsoring SIG.
- [ ] **Create an issue in kubernetes/enhancements**
  When filing an enhancement tracking issue, please make sure to complete all
  fields in that template. One of the fields asks for a link to the KEP. You
  can leave that blank until this KEP is filed, and then go back to the
  enhancement and add the link.
- [ ] **Make a copy of this template directory.**
  Copy this template into the owning SIG's directory and name it
  `NNNN-short-descriptive-title`, where `NNNN` is the issue number (with no
  leading-zero padding) assigned to your enhancement above.
- [ ] **Fill out as much of the kep.yaml file as you can.**
  At minimum, you should fill in the "Title", "Authors", "Owning-sig",
  "Status", and date-related fields.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary" and "Motivation" sections.
  These should be easy if you've preflighted the idea of the KEP with the
  appropriate SIG(s).
- [ ] **Create a PR for this KEP.**
  Assign it to people in the SIG who are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the KEP clarified and merged quickly. The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a KEP is merged does not mean it is complete or approved. Any KEP
marked as `provisional` is a working document and subject to change. You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing KEPS, aim for tightly-scoped, single-topic PRs to keep discussions
focused. If you disagree with what is already in a document, open a new PR
with suggested changes.

One KEP corresponds to one "feature" or "enhancement" for its whole lifecycle.
You do not need a new KEP to move from beta to GA, for example. If
new details emerge that belong in the KEP, edit the KEP. Once a feature has become
"implemented", major changes should get new KEPs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/keps/NNNN-kep-template/README.md).

**Note:** Any PRs to move a KEP to `implementable`, or significant changes once
it is marked `implementable`, must be approved by each of the KEP approvers.
If none of those approvers are still appropriate, then changes to that list
should be approved by the remaining approvers and/or the owning SIG (or
SIG Architecture for cross-cutting KEPs).
-->
# [KEP-5055](https://github.com/kubernetes/enhancements/issues/5055): DRA: device taints and tolerations

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Degraded Devices](#degraded-devices)
    - [External Health Monitoring](#external-health-monitoring)
    - [Safe Pod Eviction](#safe-pod-eviction)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [API](#api)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Extending node taint controller](#extending-node-taint-controller)
  - [Tolerating taints in pods](#tolerating-taints-in-pods)
  - [Storing result of patching in ResourceSlice](#storing-result-of-patching-in-resourceslice)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [kubernetes/enhancements] referencing this KEP and targeting a release
milestone **before the [Enhancement Freeze](https://git.k8s.io/sig-release/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core
Kubernetes—i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [x] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [x] (R) KEP approvers have approved the KEP status as `implementable`
- [x] (R) Design details are appropriately documented
- [x] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [x] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
- [x] (R) Production readiness review completed
- [x] (R) Production readiness review approved
- [x] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

With Dynamic Resource Allocation (DRA), DRA drivers publish information about
the devices that they manage in ResourceSlices. This information is used by the
scheduler when selecting devices for user requests in ResourceClaims.

With this KEP, DRA drivers can mark devices as tainted such that they won't be
used for scheduling new pods. In addition, pods already running with access to
a tainted device can be stopped automatically. Cluster administrators can do
the same by creating a DeviceTaintRule which applies a taint to all devices
matching certain selection criteria, like all devices of a certain driver.

Users can decide to ignore specific taints by adding tolerations to their
ResourceClaim.

## Motivation

### Goals

- Enable taking devices offline for maintenance while still allowing test pods
  to request and use those devices. Being able to do this one device at a time
  minimizes service level disruption.

- Enable users to decide whether they want to keep running a workload in a degraded
  mode while a device is unhealthy or prefer to get pods rescheduled.

- Publish information about devices ("device health") such that control plane
  components or admins can decide how to react, without immediately affecting
  scheduling or workloads.

### Non-Goals

- Not part of the plan for alpha: developing a kubectl command for managing device taints.
  This may be reconsidered.

## Proposal

### User Stories

#### Degraded Devices

A driver itself can detect problems which may or may not be tolerable for
workloads, like degraded performance due to overheating. Removing such devices
from the ResourceSlice would unconditionally prevent using them for new
pods. Instead, taints can be added to ResourceSlices to provide sufficient information
for decision making on the indicated problems. This may include information
in JSON format with a schema that is defined by the vendor together with
the semantic of taints.

A control plane component or the admins react to that information. They may
publish a DeviceTaintRule which prevents using the degraded device for new pods
or even evict all pods using it at the moment to replace or reset the device
once it is idle.

Users can decide to tolerate less critical taints in their workload, at their
own risk. Admins scheduling maintenance pods need to tolerate their own taints
to get the pod scheduled.

#### External Health Monitoring

As cluster admin, I am deploying a vendor-provided DRA driver together with a
separate monitoring component for hardware aspects that are not available or
not supported by that DRA driver. When that component detects problems, it can
check its policy configuration and decide to take devices offline by creating
a DeviceTaintRule with a taint for affected devices.

#### Safe Pod Eviction

Selecting the wrong set of devices in a DeviceTaintRule can have potentially
disastrous consequences, including quickly evicting all workloads using any
device in the cluster instead of those using a single device. To avoid this, a
cluster admin can first create a DeviceTaintRule such that it has no immediate
effect. `kubectl describe` then includes information about the matched devices
and how many pods would be evicted if the effect was to evict. At this point,
scheduling is not affected and pods keep running.

Then the admin can edit the DeviceTaintRule to set the desired effect to
"evict all affected pods".
Once eviction starts, it happens at a low enough rate that admins have a chance
to delete the DeviceTaintRule before all pods are evicted if they made a
mistake after all. This rate is configurable to enable faster eviction, if
admins are sure that this is what they want. The DeviceTaintRule status
provides information about the progress.

It can happen that a pod still gets scheduled after activating the taint because
the scheduler hasn't observed that change yet. Such a pod then gets evicted.
Either way, eventually no affected pods are running and no new ones get
scheduled either.

### Risks and Mitigations

A device can be identified by its names (`<driver name>/<pool name>/<device
name>`) and/or by its attributes (for example, a unique ID). It was a conscious
decision for core DRA to not require that the name is tied to one particular
hardware instance to support hot-swapping. Admins might favor using the names
whereas health monitoring might prefer to be specific and use a vendor-defined
unique ID. Both are supported, which creates additional complexity.

Without a kubectl extension similar to `kubectl taint nodes`, the user
experience for admins will be a bit challenging. They need to decide how to
identify the device (by name or with a CEL expression), manually create a
DeviceTaintRule with a unique name, then remember to remove that
DeviceTaintRule again. For beta, support in `kubectl` for common
operations may be needed.

Users might be tempted to tolerate taints to get their pods running. They do
that at their own risk. Depending on the taint, the application then may not
get the performance it needs (degraded hardware) or may fail at runtime
(hardware gets turned off). Admission controllers or validating admission
policies could be deployed to limit which tolerations may be used, but as
taints are not defined by Kubernetes itself, none of that is part of Kubernetes
itself.

A taint is not meant to be used as an access control mechanism. Users are
allowed to ignore taints (at their own risk). Adding a taint in a live cluster
is inherently racy because it first needs to be observed by e.g. the scheduler.

## Design Details

The feature is following the approach and APIs taken for node taints and
applies them to devices. A new controller watches tainted devices and deletes
pods using them unless they tolerate the device taint, similar to the
[taint-eviction-controller](https://github.com/kubernetes/kubernetes/blob/32130691a4cb8a1034b999341c40e48d197f5465/pkg/controller/tainteviction/taint_eviction.go#L81-L83). A pod which is running or has finalizers will not get removed
immediately. Instead, the `DeletionTimestamp` gets set. That's okay for
the purpose of this KEP:
- The kubelet will stop any running containers and mark the pod as completed.
- The ResourceClaim controller will remove such a completed pod from the claim's
  `ReservedFor` and deallocate the claim once it has no consumers.

The semantic of the value associated with a taint key is defined by whoever
publishes taints with that key. DRA drivers should use the driver name as
domain in the key to avoid conflicts. To support tolerating taints by value,
values should not require parsing to extract information. It's better to use
different keys with simple values than one key with a complex value.

The taint value may or may not be sufficient to represent the state of a
device. Therefore the API also allows publishing structured information in JSON
format as a raw extension field, similar to the ResourceClaim device status.
For humans, a description field can be used to provide a summary or additional
explanations why the taint was added.

If some common patterns emerge, then Kubernetes could standardize the name,
value and data for certain taints. Those then have a certain semantic. This
is similar to standardizing device attributes. The recommended pattern for
names is:
- `[<prefix].]<some domain owned by the vendor of a DRA driver>/<some-vendor-specific-taint>`:
  the domain part avoids name clashes. This can be the same as a DRA driver name,
  but isn't required to. Descriptive names for the different taints defined
  by a vendor are useful.
- `kubernetes.io/...`: reserved for use by Kubernetes.

`Effect: None` can be used to publish taints which are merely informational.
Such taints are ignored during scheduling and cause no eviction. Therefore
it is not necessary to specify tolerations for them. Nonetheless a toleration
for `Effect: None` is allowed. Such tolerations then have no effect either.

Taints are cumulative:
- Taints defined by an admin in a DeviceTaintRule get added to the
  set of taints defined by the DRA driver in a ResourceSlice.
- All taints that are not tolerated apply their effect.

It is valid to have the same taint key with different effects, both within a
ResourceSlice and when one is in a ResourceSlice and the other in a
DeviceTaintRule:
- Key: "A", Effect: NoSchedule
- Key: "A", Effect: NoExecute

What this means is that "A" implies both NoSchedule and NoExecute. The first
one could be dropped, but it's merely redundant, not in conflict with the
second one. It would be possible to prevent redundant entries during validation
of a ResourceSlice like it [is done for node taints](https://github.com/kubernetes/kubernetes/blob/2a3ca42c917a698e9fd3e07c55f369a9accbe2c2/pkg/apis/core/validation/validation.go#L6450-L6456),
but not when they are in different objects (ResourceSlice and DeviceTaintRule), so instead the
consumers need to handle this for device taints at runtime by checking that all
of them are tolerated without assuming uniqueness by key and effect (meaning
NoExecute toleration does not imply NoSchedule toleration).

To ensure consistency among all pods sharing a ResourceClaim, the toleration
for taints gets added to the request in a ResourceClaim, not the pod. This also
avoids conflicts like one pod tolerating a taint for scheduling and some other
pod not tolerating that.

Device and node taints are applied independently. A node taint applies to all
pods on a node, whereas a device taint affects claim allocation and only those
pods using the claim.

### API

The ResourceSlice content gets extended. A natural place for a taint that
affects one specific device would be inside the `Device` struct. The problem
with that approach is that with 128 allowed devices per ResourceSlice the
maximum size of each taint would have to be very small to keep the worst-case
size of a ResourceSlice within the limits imposed by etcd and the API request
machinery.

Therefore a different approach is used where each ResourceSlice either provides
information about devices or about taints, but never both. Using the
ResourceSlice for taints instead of a different type has the advantage that the
existing mechanisms and code for publishing and consuming the information can
be reused. The same approach is likely to be used for other additional
ResourceSlice pool information, like mixin definitions. The downside is that at
least two ResourceSlices are necessary once taints get published by a DRA
driver.

If changes of taints are limited to the content of single ResourceSlice and
their number stays the same, too, then there is no need to bump the generation
of the pool: a consumer will see and use either the old set of ResourceSlices
or the new, updated set. This makes updating taints efficient because it avoids
having to update the other ResourceSlices with a different generation or
count. DRA drivers which support taints are therefore encouraged to always publish
a ResourceSlice for taints even if there are currently none.

An empty ResourceSlice is valid. This was already allowed previously to let DRA
drivers indicate that they are up and running on a node even if they found no
devices.

```Go
type ResourceSliceSpec struct {
    ...

    // Devices lists some or all of the devices in this pool.
    //
    // Must not have more than 128 entries. Either Devices or Taints may be set, but not both.
    //
    // +optional
    // +listType=atomic
    // +oneOf=ResourceSliceContent
    Devices []Device

    // If specified, these are driver-defined taints.
    //
    // The maximum number of taints is 32. Either Devices or Taints may be set, but not both.
    //
    // This is an alpha field and requires enabling the DRADeviceTaints
    // feature gate.
    //
    // +optional
    // +listType=atomic
    // +featureGate=DRADeviceTaints
    // +oneOf=ResourceSliceContent
    Taints []SliceDeviceTaint
}

// DeviceTaintsMaxLength is the maximum number of taints per ResourceSlice.
const DeviceTaintsMaxLength = 32

// SliceDeviceTaint defines one taint within a ResourceSlice.
type SliceDeviceTaint struct {
   // Device is the name of the device in the pool that the ResourceSlice belongs to
   // which is affected by the taint. Multiple taints may affect the same device.
   //
   // It is not required that such a device exists in the pool. If there is none
   // with the same name, the taint is ignored by Kubernetes. One possible usage
   // for this is publishing of health information which applies to the entire
   // pool of devices and which then gets used by some vendor-specific
   // controller that implements custom policies for dealing with problems.
   //
   // The name must not be empty to prevent accidentally leaving it unset.
   // For the health use case a special name can be used which does not
   // match the names used for actual devices.
   //
   // +required
   Device string

   Taint DeviceTaint
}
```

DeviceTaint has all the fields of a v1.Taint, but the description is a bit
different. In particular, PreferNoSchedule is not valid. Other fields got added
to satisfy additional use cases like device health information.

```Go
// The device this taint is attached to has the "effect" on
// any claim which does not tolerate the taint and, through the claim,
// to pods using the claim.
type DeviceTaint struct {
    // The taint key to be applied to a device.
    // Must be a label name.
    //
    // +required
    Key string

    // The taint value corresponding to the taint key.
    // Must be a label value.
    //
    // +optional
    Value string

    // The effect of the taint on claims that do not tolerate the taint
    // and through such claims on the pods using them.
    // Valid effects are None, NoSchedule and NoExecute. PreferNoSchedule as used for
    // nodes is not valid here.
    //
    // +required
    Effect DeviceTaintEffect

    // ^^^^
    //
    // Implementing PreferNoSchedule would depend on a scoring solution for DRA.
    // It might get added as part of that.

    // TimeAdded represents the time at which the taint was added.
    // Added automatically during create or update if not set.
    //
    // +optional
    TimeAdded *metav1.Time

    // ^^^
    //
    // This field was defined as "It is only written for NoExecute taints." for node taints.
    // But in practice, Kubernetes never did anything with it (no validation, no defaulting,
    // ignored during pod eviction in pkg/controller/tainteviction).

    // Data contains arbitrary data specific to the taint key.
    //
    // The length of the raw data must be smaller or equal to 10 Ki.
    //
    // +optional
    Data *runtime.RawExtension

    // EvictionsPerSecond controls how quickly Pods get evicted if that is
    // the effect of the taint.
    //
    // Evictions are tracked separately for each taint. Each eviction has
    // a rate limiter which uses the EvictionsPerSecond configured for the
    // taint. Deleting a pod that is evicted by the taint is delayed by that
    // rate limiter.
    //
    // A pod may be affected by more than one taint. In that case, the
    // smallest delay required to satisfy any of the rate limits is used
    // to delay eviction of a pod. The effect is that pods affected
    // by multiple taints get evicted at the highest rate defined by any
    // of those taints.
    //
    // The default is 10 Pods/s.
    //
    // +optional
    EvictionsPerSecond *int64
}

const (
    // DefaultEvictionsPerSecond is the default for [DeviceTaint.EvictionsPerSecond]
    // if none is specified explicitly.
    DefaultEvictionsPerSecond = 10

    // TaintDescriptionMaxLength is the maximum size of [DeviceTaint.Description].
    TaintDescriptionMaxLength = 1024

    // TaintDataMaxLength is the maximum size of [DeviceTaint.Data].
    TaintDataMaxLength = 10 * 1024
)
```

The expected usage of EvictionsPerSecond is to speed up eviction because the
default is fairly conservative. Evicting at the highest rate defined by any of
the taints affecting a pod is consistent with that expected usage. The
alternative would be to evict at the smallest rate, but then admins might not
get what they asked for. It also has the disadvantage that a compromised node
can slow down pod eviction by publishing taints in ResourceSlices with a very
low EvictionsPerSecond.

The other parameter for a rate limiter besides the rate is the burst, the
number of pods which get evicted without delay. The rate starts to apply after
that many pods have been evicted. The API has no separate parameter for
configuring the burst to keep the API simple. The burst parameter that is used
by the implementation is also 10. This is a somewhat arbitrary compromise
between evicting small number of pods quickly and applying the rate for larger
number of pods.

```

// +enum
type DeviceTaintEffect string

const (
    // No effect, the taint is purely informational.
    DeviceTaintEffectNone DeviceTaintEffect = "None"

    // Do not allow new pods to schedule which use a tainted device unless they tolerate the taint,
    // but allow all pods submitted to Kubelet without going through the scheduler
    // to start, and allow all already-running pods to continue running.
    DeviceTaintEffectNoSchedule DeviceTaintEffect = "NoSchedule"

    // Evict any already-running pods that do not tolerate the device taint.
    DeviceTaintEffectNoExecute DeviceTaintEffect = "NoExecute"
)
```

Tolerations get added to a DeviceRequest:

```Go
type DeviceRequest struct {
    ...

    // If specified, the request's tolerations.
    //
    // Tolerations for NoSchedule are required to allocate a
    // device which has a taint with that effect. The same applies
    // to NoExecute.
    //
    // In addition, should any of the allocated devices get tainted
    // with NoExecute after allocation and that effect is not tolerated,
    // then all pods consuming the ResourceClaim get deleted to evict
    // them. The scheduler will not let new pods reserve the claim while
    // it has these tainted devices. Once all pods are evicted, the
    // claim will get deallocated.
    //
    // The maximum number of tolerations is 16.
    //
    // This is an alpha field and requires enabling the DRADeviceTaints
    // feature gate.
    //
    // +optional
    // +listType=atomic
    // +featureGate=DRADeviceTaints
    Tolerations []DeviceToleration
}

// DeviceTolerationsMaxLength is the maximum number of tolerations in a DeviceRequest.
const DeviceTolerationsMaxLength = 16

// The ResourceClaim this DeviceToleration is attached to tolerates any taint that matches
// the triple <key,value,effect> using the matching operator <operator>.
type DeviceToleration struct {
    // Key is the taint key that the toleration applies to. Empty means match all taint keys.
    // If the key is empty, operator must be Exists; this combination means to match all values and all keys.
    // Must be a label name.
    //
    // +optional
    Key string

    // Operator represents a key's relationship to the value.
    // Valid operators are Exists and Equal. Defaults to Equal.
    // Exists is equivalent to wildcard for value, so that a ResourceClaim can
    // tolerate all taints of a particular category.
    //
    // +optional
    // +default="Equal"
    Operator DeviceTolerationOperator

    // Value is the taint value the toleration matches to.
    // If the operator is Exists, the value must be empty, otherwise just a regular string.
    // Must be a label value.
    //
    // +optional
    Value string

    // Effect indicates the taint effect to match. Empty means match all taint effects.
    // When specified, allowed values are NoSchedule and NoExecute.
    //
    // +optional
    Effect DeviceTaintEffect

    // TolerationSeconds represents the period of time the toleration (which must be
    // of effect NoExecute, otherwise this field is ignored) tolerates the taint. By default,
    // it is not set, which means tolerate the taint forever (do not evict). Zero and
    // negative values will be treated as 0 (evict immediately) by the system.
    // If larger than zero, the time when the pod needs to be evicted is calculated as <time when
    // taint was adedd> + <toleration seconds>.
    //
    // +optional
    TolerationSeconds *int64
}

// A toleration operator is the set of operators that can be used in a toleration.
//
// +enum
type DeviceTolerationOperator string

const (
    DeviceTolerationOpExists DeviceTolerationOperator = "Exists"
    DeviceTolerationOpEqual  DeviceTolerationOperator = "Equal"
)
```

As with Taint, these structs get duplicated to document DRA specific
behavior and to ensure that future extensions do not get inherited
accidentally.

The implementation of the taint logic gets copied from node taints:
[resourceclaim.ToleratesTaint](https://github.com/kubernetes/kubernetes/blob/85734ac6b38e29a9a390520e7a5b6de1fbf5ff6b/staging/src/k8s.io/dynamic-resource-allocation/resourceclaim/devicetoleration.go#L23-L53)
corresponds to
[v1.Toleration.ToleratesTaint](https://github.com/kubernetes/kubernetes/blob/85734ac6b38e29a9a390520e7a5b6de1fbf5ff6b/staging/src/k8s.io/api/core/v1/toleration.go#L29-L57).

To enable setting of taints without reconfiguring DRA drivers, the
cluster-scoped DeviceTaintRule gets added. The scheduler must merge these
additional taints with the ones provided by
the DRA drivers on-the-fly while it gathers information about available
devices. This could be done each time a scheduling cycle starts, but it
would be repetitive work. Instead, a
[ResourceSlice tracker](https://github.com/kubernetes/kubernetes/blob/85734ac6b38e29a9a390520e7a5b6de1fbf5ff6b/staging/src/k8s.io/dynamic-resource-allocation/resourceslice/tracker/tracker.go#L56-L59)
reacts to informer events for ResourceSlice and DeviceTaintRule and
maintains a set of updated slices which also contain the taints
set via a DeviceTaintRule and, starting with Kubernetes 1.35,
the taints published by DRA drivers in separate ResourceSlices.

The tracker provides the API of an informer and thus can be used as a
replacement for a ResourceSlice informer. In the initial implementation
it produced ResourceSlices with taints added to the `Device` struct.
Now that this struct no longer contains taints,
it uses the types from `k8s.io/dynamic-resource-allocation/api` to represent
ResourceSlices. The difference compared to the `k8s.io/resource/v1` Go types are:

- Usage of `unique.Handle[String]` = `api.UniqueString` to speed up certain
  string comparisons - this had turned out to improve scheduling performance.
  It is less relevant for device taints.
- `api.TrackedDevice` extends `Device` with a list of taints.
- `api.TrackedDeviceTaint` extends `DeviceTaint` with a pointer back to
  the `DeviceTaintRule` for updating the status (if applicable) and
  with a numeric ID.

This ID is generated on-the-fly by the ResourceSlice tracker. Each new taint
that has not been seen before is assigned a new number. During updates, taints
from the same rule (when available) or with the same attributes (otherwise) retain
their ID. It does not persist across process restarts, which is sufficient
because the ID is only used internally.

The eviction controller uses this ID to keep track of on-going evictions. Each
eviction has its own rate limiter which gets instantiated on demand when
eviction of pods starts. Idle evictions get removed to avoid constant memory
overhead.

```Go
// DeviceTaintRule adds one taint to all devices which match the selector.
// This has the same effect as if the taint was specified directly
// in the ResourceSlice by the DRA driver.
type DeviceTaintRule struct {
    metav1.TypeMeta
    // Standard object metadata
    // +optional
    metav1.ObjectMeta

    // Spec specifies the selector and one taint.
    //
    // Changing the spec automatically increments the metadata.generation number.
    Spec DeviceTaintRuleSpec

    // Status provides information about what was requested in the spec.
    //
    // +optional
    Status DeviceTaintRuleStatus
}

// DeviceTaintRuleSpec specifies the selector and one taint.
type DeviceTaintRuleSpec struct {
    // DeviceSelector defines which device(s) the taint is applied to.
    // All selector criteria must be satisfied for a device to
    // match. The empty selector matches all devices. Without
    // a selector, no devices are matched.
    //
    // +optional
    DeviceSelector *DeviceTaintSelector

    // The taint that gets applied to matching devices.
    //
    // +required
    Taint DeviceTaint
}

// DeviceTaintSelector defines which device(s) a DeviceTaintRule applies to.
// The empty selector matches all devices. Without a selector, no devices
// are matched.
type DeviceTaintSelector struct {
    // If DeviceClassName is set, the selectors defined there must be
    // satisfied by a device to be selected. This field corresponds
    // to class.metadata.name.
    //
    // +optional
    DeviceClassName *string

    // If driver is set, only devices from that driver are selected.
    // This fields corresponds to slice.spec.driver.
    //
    // +optional
    Driver *string

    // If pool is set, only devices in that pool are selected.
    //
    // Also setting the driver name may be useful to avoid
    // ambiguity when different drivers use the same pool name,
    // but this is not required because selecting pools from
    // different drivers may also be useful, for example when
    // drivers with node-local devices use the node name as
    // their pool name.
    //
    // +optional
    Pool *string

    // If device is set, only devices with that name are selected.
    // This field corresponds to slice.spec.devices[].name.
    //
    // Setting also driver and pool may be required to avoid ambiguity,
    // but is not required.
    //
    // +optional
    Device *string

    // Selectors contains the same selection criteria as a ResourceClaim.
    // Currently, CEL expressions are supported. All of these selectors
    // must be satisfied.
    //
    // +optional
    // +listType=atomic
    Selectors []DeviceSelector
}

// DeviceTaintRuleStatus provides information about an on-going pod eviction.
type DeviceTaintRuleStatus struct {
    // Conditions provide information about the current state of the DeviceTaintRule
    // in a machine-readable and human-readable format.
    //
    // The following condition is currently defined as part of this API, more may
    // get added:
    // - Type: EvictionInProgress
	// - Status: True if there are currently pods which need to be evicted, False otherwise
	//   (includes the effects which don't cause eviction).
    // - Reason: not specified, may change
    // - Message: includes information about number of pending pods and already evicted pods
    //   in a human-readable format, updated periodically, may change
    //
    // For `effect: None`, the condition above gets set once for each change to
    // the spec, with the message containing information about what would happen
    // if the effect was `NoExecute`. This feedback can be used to decide whether
    // changing the effect to `NoExecute` will work as intended. It only gets
    // set once to avoid having to constantly update the status.
    //
    // Must have 8 or less entries.
    //
    // +optional
    // +listType=map
    // +listMapKey=type
    Conditions []metav1.Condition
}

const DeviceTaintRuleStatusMaxConditions = 8
```

The condition is meant to be used to check for completion of a triggered eviction with:

    kubectl wait --for=condition=EvictionInProgress=false DeviceTaintRule/my-taint-rule

Reasons are not specified as part of the API because it's not required by the
use cases and would restrict future changes unnecessarily. For example, `reason: Idle`
can go together with `evictionInProgress: False` when momentarily there are no
pods which need to be evicted.

### Test Plan

[X] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

None.

##### Unit tests

<!--
Generated with:

go test -cover ./pkg/apis/resource/validation  ./staging/src/k8s.io/dynamic-resource-allocation/structured ./pkg/controller/devicetainteviction | sed -e 's/.*\(k8s.io[a-z/-]*\).*coverage: \(.*\) of statements/- `\1`: \2/' | sort

-->

v1.32.0:

- `k8s.io/dynamic-resource-allocation/structured`: 91.3%
- `k8s.io/kubernetes/pkg/apis/resource/validation`: 98.6%
- `k8s.io/kubernetes/pkg/controller/tainteviction`: 81.8%

v1.33.0:

- `k8s.io/dynamic-resource-allocation/structured`: 91.3%
- `k8s.io/kubernetes/pkg/apis/resource/validation`: 97.8%
- `k8s.io/kubernetes/pkg/controller/devicetainteviction`: 89.9%

Test cases that are worth calling out:

- Ensure that eviction happens at the desired rate for:
  - One set of pods affected by one taint.
  - One set of pods affected by multiple taints with different rates.
  - Different set of pods which are affected by multiple taints with different rates.
- Eviction at default rate and different explicitly chosen rates.
- Combinations of tolerations and taints.

##### Integration tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
-->

Integration tests for the new eviction manager will be useful to ensure that
permissions are correct.

##### e2e tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html

We expect no non-infra related flakes in the last month as a GA graduation criteria.
-->

Useful E2E tests are checking that the scheduler really honors taints during
scheduling. Adding a taint in a ResourceSlice must evict a running pod. Same
for adding a taint through a DeviceTaintRule. A toleration for a NoExecute
taint must allow a pod to run.

### Graduation Criteria

#### Alpha

- Feature implemented behind a feature flag
- Initial e2e tests completed and enabled

#### Beta

- Gather feedback from developers and surveys
- Additional tests are in Testgrid and linked in KEP

#### GA

- 3 examples of real-world usage
- Allowing time for feedback
- [Conformance tests]

[conformance tests]: https://git.k8s.io/community/contributors/devel/sig-architecture/conformance-tests.md

### Upgrade / Downgrade Strategy

Tainting gets disabled when downgrading to a release without support for it or
when disabling the feature. The effect is as if the taints weren't set.

The alpha fields for taints in ResourceSlices changed in 1.35. DRA drivers
which use the common helper code and are trying to publish taints directly for
devices (API in 1.33 and 1.34) in a cluster with the API from 1.35 will get
[notified about the dropped fields](https://github.com/kubernetes/kubernetes/blob/f28b4c9efbca5c5c0af716d9f2d5702667ee8a45/staging/src/k8s.io/dynamic-resource-allocation/kubeletplugin/draplugin.go#L124-L143).
The recommended reaction is to log and fail, which indicates to admins that
they need to update or reconfigure the DRA driver. The same applies for the
other direction (trying to use the new API on an older Kubernetes).

The DeviceTaintRule in the alpha API got extended. The new field for setting
the rate limit is ignored by previous releases. `effect: None` is only valid
for Kubernetes >= 1.35 and DeviceTaintRules with that value must be removed
before a downgrade.

### Version Skew Strategy

During version skew where the apiserver supports the feature and the scheduler
doesn't, taints can be set without encountering errors or
warnings, but they won't have any effect.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

It is possible to disable the feature through the feature gate while leaving
the API group enabled. This enables cleanup through the API.

Re-enabling is supported because DeviceTaintRules remain in etcd even if
they are inaccessible and existing taints and tolerations are preserved during
updates.

###### How can this feature be enabled / disabled in a live cluster?

- [X] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: DRADeviceTaints
  - Components depending on the feature gate:
    - kube-apiserver
    - kube-scheduler
    - kube-controller-manager
- [X] Other
  - Describe the mechanism: resource.k8s.io/v1alpha3 API group
  - Will enabling / disabling the feature require downtime of the control
    plane? Yes, in the apiserver.
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node? No.

###### Does enabling the feature change any default behavior?

No.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. The behavior of scheduling changes when it was in use.
Running applications are not affected.

###### What happens if we reenable the feature if it was previously rolled back?

It takes effect again for scheduling and may evict pods.

###### Are there any tests for feature enablement/disablement?

This will be covered through unit tests for the apiserver and scheduler.

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->

###### How can a rollout or rollback fail? Can it impact already running workloads?

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?

Be sure to consider highly-available clusters, where, for example,
feature flags will be enabled on some API servers and not others during the
rollout. Similarly, consider large clusters and how enablement/disablement
will rollout across nodes.
-->

###### What specific metrics should inform a rollback?

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!--
Even if applying deprecation policies, they may still surprise some users.
-->

### Monitoring Requirements

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### How can an operator determine if the feature is in use by workloads?

<!--
Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
checking if there are objects with field X set) may be a last resort. Avoid
logs or events for this purpose.
-->

###### How can someone using this feature know that it is working for their instance?

<!--
For instance, if this is a pod-related feature, it should be possible to determine if the feature is functioning properly
for each individual pod.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so that they can verify correct enablement
and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

- [ ] Events
  - Event Reason: 
- [ ] API .status
  - Condition name: 
  - Other field: 
- [ ] Other (treat as last resort)
  - Details:

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

<!--
This is your opportunity to define what "normal" quality of service looks like
for a feature.

It's impossible to provide comprehensive guidance, but at the very
high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99.9% of /health requests per day finish with 200 code

These goals will help you determine what you need to measure (SLIs) in the next
question.
-->

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->

- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [ ] Other (treat as last resort)
  - Details:

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->

### Dependencies

<!--
This section must be completed when targeting beta to a release.
-->

###### Does this feature depend on any specific services running in the cluster?

<!--
Think about both cluster-level services (e.g. metrics-server) as well
as node-level agents (e.g. specific version of CRI). Focus on external or
optional services that are needed. For example, if this feature depends on
a cloud provider API, or upon an external software-defined storage or network
control plane.

For each of these, fill in the following—thinking about running existing user workloads
and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:
-->

### Scalability

Applying taints to devices scales with `number of DeviceTaintRules` *
`number of devices` when CEL selectors need to be evaluated. Without them,
filtering scales with `number of DeviceTaintRules` * `number of
ResourceSlices` but then may still need to compare device names and of course
modify selected devices.

Handling eviction scales with the number of claims and pods using those claims.

###### Will enabling / using this feature result in any new API calls?

A fixed, small number of clients (primarily the scheduler and controller
manager) need to start watching DeviceTaintRules.

Pods are already watched in the controller manager. Evicting them adds one call
per pod.

###### Will enabling / using this feature result in introducing new API types?

DeviceTaintRules must be created explicitly by admins or controller
operated by admins. Kubernetes itself does not create them.

The number of DeviceTaintRules is expected to be orders of
magnitude smaller than the number of ResourceSlices.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

Enabling it doesn't. Using tolerations increases the size of ResourceClaims and
ResourceClaimTemplates.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

Pod scheduling may become a bit slower because of the additional checks, but
only when pods use claims. There are no SLI/SLOs for pods using claims.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

For scheduling, tracking taints should be comparable to the overhead for
patching attributes.

For eviction, additional data structures will be needed to track taints and
tolerations. This should not be too large.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

No, because the feature is not used on nodes.

### Troubleshooting

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

###### How does this feature react if the API server and/or etcd is unavailable?

###### What are other known failure modes?

<!--
For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.
-->

###### What steps should be taken if SLOs are not being met to determine the problem?

## Implementation History

- 1.33: first KEP revision and implementation

## Drawbacks

Distributing information across different objects of different types makes it
harder for users to get a complete view.

## Alternatives

### Extending node taint controller

The existing taint-eviction-controller could be extended to cover device
taints. However, cloning it lowers the risk of breaking existing stable functionality.

### Tolerating taints in pods

Tolerations for device taints could also be added to individual pods. This
seems less useful because if pods share the same claim, they are typically part
of one larger application with identical tolerations. Experimenting with a new
API in the beta ResourceClaim type is a bit easier than it would be in the GA
Pod type.

### Storing result of patching in ResourceSlice

A controller could read DeviceTaintRules and apply them to
ResourceSlices. Then consumers like the scheduler and users would only need to
look at ResourceSlices. This has several drawbacks.

We would need to duplicate the taints in the slice status. If we didn't and
directly modified the spec, this controller and the CSI driver as the
owner of the slice spec would fight against each other. Also, after removing a
patch the original state must be available somewhere, otherwise the controller
cannot restore it.

Duplicating the taints might make a slice too large. The limits were chosen
so that we have some space left for a status, but not enough for a status that
is potentially as large as the spec.

Creating a single DeviceTaintRule could force the controller to update a
potentially large number of ResourceSlices. When using rate limiting, updating
them all will take longer than client-side patching. When not using rate
limiting, this could overwhelm the apiserver.
