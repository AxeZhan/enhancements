<!--
**Note:** When your KEP is complete, all of these comment blocks should be removed.

To get started with this template:

- [x] **Pick a hosting SIG.**
  Make sure that the problem space is something the SIG is interested in taking
  up. KEPs should not be checked in without a sponsoring SIG.
- [x] **Create an issue in kubernetes/enhancements**
  When filing an enhancement tracking issue, please make sure to complete all
  fields in that template. One of the fields asks for a link to the KEP. You
  can leave that blank until this KEP is filed, and then go back to the
  enhancement and add the link.
- [x] **Make a copy of this template directory.**
  Copy this template into the owning SIG's directory and name it
  `NNNN-short-descriptive-title`, where `NNNN` is the issue number (with no
  leading-zero padding) assigned to your enhancement above.
- [x] **Fill out as much of the kep.yaml file as you can.**
  At minimum, you should fill in the "Title", "Authors", "Owning-sig",
  "Status", and date-related fields.
- [x] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary" and "Motivation" sections.
  These should be easy if you've preflighted the idea of the KEP with the
  appropriate SIG(s).
- [x] **Create a PR for this KEP.**
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
# [KEP-5007](https://github.com/kubernetes/enhancements/issues/5007): DRA: Device Binding Conditions

<!--
This is the title of your KEP. Keep it short, simple, and descriptive. A good
title can help communicate what the KEP is and should be considered as part of
any review.
-->

<!--
A table of contents is helpful for quickly jumping to sections of a KEP and for
highlighting any additional information provided beyond the standard KEP
template.

Ensure the TOC is wrapped with
  <code>&lt;!-- toc --&rt;&lt;!-- /toc --&rt;</code>
tags, and then generate with `hack/update-toc.sh`.
-->

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [PreBind Process](#prebind-process)
  - [Handling ResourceSlices Upon Failure of Attachment](#handling-resourceslices-upon-failure-of-attachment)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1: Asynchronous Device Initialization](#story-1-asynchronous-device-initialization)
    - [Story 2: Fabric-Attached GPU Allocation](#story-2-fabric-attached-gpu-allocation)
    - [Story 3: Gang Scheduling with Deferred Binding](#story-3-gang-scheduling-with-deferred-binding)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [DRA Scheduler Plugin Design Overview](#dra-scheduler-plugin-design-overview)
    - [BasicDevice Enhancements](#basicdevice-enhancements)
    - [DeviceRequestAllocationResult Enhancements](#devicerequestallocationresult-enhancements)
    - [Scheduler DRA Plugin Modifications](#scheduler-dra-plugin-modifications)
    - [PreBind Phase Timeout](#prebind-phase-timeout)
    - [Handling ResourceSlices Upon Failure of Attachment](#handling-resourceslices-upon-failure-of-attachment-1)
  - [Alternative approach](#alternative-approach)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
    - [Deprecation](#deprecation)
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
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
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

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
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

This KEP introduces a general-purpose mechanism — BindingConditions — to improve scheduling reliability in Kubernetes environments where resource readiness is asynchronous or failure-prone.

While the original motivation for this KEP was to support fabric-attached devices in composable disaggregated infrastructure, the proposed mechanism is designed to be broadly applicable.
BindingConditions enable the scheduler to defer Pod binding until external resources (such as attachable devices, remote accelerators, or programmable hardware) are confirmed to be ready.
This allows for more deterministic scheduling and avoids premature binding that could lead to Pod failures or manual intervention.

The mechanism is not tied to any specific hardware model or infrastructure.
It can support a wide range of use cases, including:
- Fabric-attached GPUs that require dynamic attachment via PCIe or CXL switches
- FPGAs that need time-consuming reprogramming before use

If device preparation fails (e.g., due to contention or hardware error), the scheduler may clear the allocation and allow the Pod to be reconsidered for scheduling.
This fallback behavior is part of error handling, but the primary goal is to enable more robust and predictable scheduling by explicitly modeling readiness before binding.

## Motivation

As AI and ML workloads become increasingly common in Kubernetes environments, the demand for high-performance computing resources continues to grow.
At the same time, energy efficiency and flexible infrastructure utilization are becoming critical for sustainable operations.

Composable Disaggregated Infrastructure (CDI) has emerged as a promising approach to address these needs.
In CDI, hardware resources such as CPUs, memory, and GPUs are disaggregated and pooled, allowing them to be dynamically composed into custom server configurations.
Fabric-attached devices — such as GPUs connected via PCIe or CXL switches — are a key component of this model.

In Kubernetes, these fabric devices may be shared across clusters and exposed via `ResourceSlice`.
However, when device attachment occurs only after a Pod is scheduled, there is a risk that the device may not be available at the time of attachment, leading to Pod failures or manual intervention.

To address this, we propose a mechanism — `BindingConditions` — that allows the scheduler to defer Pod binding until the required device is confirmed to be ready.
This improves reliability by avoiding premature binding and enables better coordination with external device controllers.

While the original motivation came from fabric-attached devices, the mechanism is designed to be broadly applicable.
It can support other scenarios where resource readiness is asynchronous or failure-prone, such as remote accelerators or gang scheduling.
This proposal focuses on enabling readiness-aware binding as a general scheduling enhancement.

### Goals

1. **Enable Readiness-Aware Binding**:  
   Introduce a mechanism (`BindingConditions`) that allows the scheduler to defer Pod binding until external resources are confirmed to be ready.
   This improves scheduling reliability and avoids premature binding in environments where resource preparation is asynchronous or failure-prone.

2. **Define Binding and Failure Conditions Explicitly**:  
   Allow device providers to specify `BindingConditions` and `BindingFailureConditions` that the scheduler can evaluate during the PreBind phase.

3. **Prioritize Devices Based on Readiness**:  
   When multiple candidate devices are available, the scheduler should prefer devices in the following order:
   1. Devices without any `BindingConditions` (i.e., immediately usable)
   2. Devices with `BindingConditions` (i.e., require preparation)

   This prioritization naturally favors node-local devices over fabric-attached ones, assuming the latter require preparation.

### Non-Goals

- Defining a complete model for fabric-attached device coordination.
- While this KEP introduces a mechanism that supports such use cases, broader architectural questions — such as how to model attachment workflows or coordinate between node-local and fabric-aware components — will be addressed in follow-up discussions.

## Proposal

This proposal introduces a mechanism that allows the scheduler to defer Pod binding until external resources are confirmed to be ready.
This is achieved by extending the `ResourceSlice` and `DeviceRequestAllocationResult` APIs with the following fields:

- `BindingConditions`: a list of condition keys that must be satisfied (i.e., set to `True`) before binding can proceed.
- `BindingFailureConditions`: a list of condition keys that, if any are `True`, indicate that binding should be aborted and the Pod rescheduled.
- `BindingTimeoutSeconds`: a timeout value (in seconds) that defines how long the scheduler should wait for all `BindingConditions` to be satisfied. If the timeout is exceeded, the allocation is cleared and the Pod is rescheduled.
- `UsageRestrictedToNode`: a boolean field (optional) that indicates whether the scheduler should set the nodename to the ResourceClaim's nodeSelector during the scheduling process. When set to true, the scheduler records the selected node name in the claim, effectively reserving the resource for that specific node. This is particularly useful for external controllers during the PreBind phase, as they can retrieve the node name from the claim and perform node-specific operations such as device attachment or preparation.

Each entry in `BindingConditions` and `BindingFailureConditions` must be a valid Kubernetes condition type string (e.g., `dra.example.com/is-prepared`). These are interpreted as keys in the `.status.conditions` field of the corresponding `ResourceClaim`.
External controllers are responsible for updating these condition keys with standard Kubernetes condition semantics (`type`, `status`, `reason`, `message`, `lastTransitionTime`).

The scheduler evaluates only the `status` field of these conditions:
- All `BindingConditions` must have `status: "True"` to proceed with binding.
- If any `BindingFailureConditions` has `status: "True"`, the binding is aborted.

These fields are introduced as alpha features and require enabling the `DRADeviceBindingConditions` and `DRAResourceClaimDeviceStatus` feature gates.
`DRAResourceClaimDeviceStatus` is required because it controls
whether these binding conditions can be set in the ResourceClaim status.

This mechanism enables readiness-aware scheduling for resources that require asynchronous preparation, such as fabric-attached devices or remote accelerators.
It allows the scheduler to make binding decisions based on up-to-date readiness information, improving reliability and avoiding premature binding.

While this proposal supports fabric-attached devices, it is not limited to them.
The mechanism is designed to be general and can support other use cases where resource readiness is asynchronous or failure-prone.

### PreBind Process

During the scheduling cycle, the DRA scheduler plugin selects a suitable `ResourceSlice` for each `ResourceClaim` and copies relevant metadata into the `AllocationResult` of the claim.
This includes:

- `BindingConditions`
- `BindingFailureConditions`
- `BindingTimeoutSeconds`

In the `PreBind` phase, the scheduler evaluates the readiness of the selected device by checking the following:

1. **All `BindingConditions` must be `True`**  
   These conditions represent readiness signals (e.g., "device attached", "controller acknowledged") that must be satisfied before binding can proceed.

2. **No `BindingFailureConditions` may be `True`**  
   If any failure condition is set, the scheduler aborts the binding attempt and clears the allocation.

3. **Timeout Handling**  
   If `BindingConditions` are not all satisfied within the specified `BindingTimeoutSeconds`, the scheduler treats this as a timeout failure.
   The allocation is cleared, and the Pod is returned to the scheduling queue for reconsideration.

This mechanism ensures that Pods are only bound to nodes when the associated devices are confirmed to be ready, avoiding premature binding and improving scheduling reliability.

External controllers (e.g., composable DRA controllers) are responsible for monitoring the `ResourceClaim` and updating the condition statuses as device preparation progresses.
This coordination allows the scheduler to make informed binding decisions without requiring tight coupling between the scheduler and device-specific logic.

### Handling attachment failures

If device preparation fails — for example, due to fabric contention, hardware error, or controller-side timeout — the external controller (e.g., a composable DRA controller) should update the condition status in the `ResourceClaim`'s `allocationResult` to reflect the failure.
This is typically done by setting one or more `BindingFailureConditions` to `True`.
It should also ensure that the device is not picked again.
It can do that by removing the device from the ResourceSlice,
adding a [device taint](https://github.com/kubernetes/enhancements/issues/5055),
or by changing the node selector, either at the ResourceSlice
level or using per-device node selectors from the [partitionable devices](https://github.com/kubernetes/enhancements/issues/4815).
There is no guarantee that the scheduler will receive those
updates in time for the next pod scheduling cycle, but
eventually it will.

The scheduler will detect this during the `PreBind` phase and respond by:

1. **Aborting the binding cycle**: The Pod will not be bound to the node.
2. **Clearing the allocation**: The `ResourceClaim` is unbound from the `ResourceSlice`, making the resource available for other Pods.
3. **Requeuing the Pod**: The Pod is returned to the scheduling queue for reconsideration.

This failure-handling path is intended as an exception, not the primary scheduling model.
The preferred approach is for the scheduler to wait until all `BindingConditions` are satisfied, and only proceed with binding when the device is confirmed to be ready.

By modeling readiness explicitly and handling failures gracefully, this mechanism improves scheduling reliability and avoids leaving Pods in unrecoverable states.

### User Stories (Optional)

<!--
Detail the things that people will be able to do if this KEP is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1: Asynchronous Device Initialization

A user deploys a Pod that requires a specialized FPGA device.
The device must be reprogrammed before use, which takes several minutes.
The device controller exposes a `BindingCondition` indicating that initialization is in progress.

The scheduler waits in the `PreBind` phase until the FPGA is ready.
If initialization completes successfully, the Pod is bound and started.
If the process fails, a `BindingFailureCondition` is set, and the Pod is rescheduled.
This avoids binding the Pod to a node where the device is not yet usable.

#### Story 2: Fabric-Attached GPU Allocation

The composable controller exposes the GPU as a `ResourceSlice` with a `BindingCondition` indicating that attachment is required.
A user submits a Pod that requests a GPU managed by a composable infrastructure system.
The GPU is not yet attached to any node and must be dynamically connected via a PCIe fabric.

The scheduler selects a node and defers binding until the controller completes the attachment and updates the condition to `True`.
Once the device is ready, the scheduler proceeds with binding.
If the attachment fails or times out, the Pod is rescheduled.
This ensures that the Pod is only bound when the device is truly usable.

#### Story 3: Gang Scheduling with Deferred Binding

A user runs a distributed training job that requires multiple Pods to be scheduled together, each with a GPU.
The job controller coordinates the scheduling and uses `BindingConditions` to delay binding until all Pods are ready to proceed.

Once all Pods have satisfied their readiness conditions, the scheduler binds them simultaneously.
This ensures that the job starts in a coordinated fashion, avoiding partial execution or resource waste.

note: This is a speculative use case and not currently supported.

### Notes/Constraints/Caveats (Optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above?
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

- **Maximum Number of Conditions**: The maximum number of `BindingConditions` and `BindingFailureConditions` per device is limited to 4. This constraint ensures that the scheduler can efficiently evaluate conditions without excessive overhead
and that the worst-case size of a ResourceSlice does not grow
too much.

- **External Controller Dependency**: The effectiveness of `BindingConditions` relies on accurate and timely updates from external controllers (e.g., composable DRA controllers). Any delays or inaccuracies in these updates can impact scheduling reliability.

- **Error Handling**: If `BindingConditions` are not satisfied within the specified timeout, or if any `BindingFailureConditions` are set to `True`, the scheduler will clear the allocation and reschedule the Pod. This fallback behavior is intended as an error recovery mechanism, not the primary scheduling model.

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
Kubernetes ecosystem.

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside the SIG or subproject.
-->

**What if the scheduler restarts while the DRA plugin is waiting for the device(s) to be bound?**

The scheduler's restart should not pose an issue, as the decision to wait is based on the Conditions of the ResourceClaim.
However, care must be taken to ensure that condition updates are persisted correctly.
If the scheduler restarts while a Pod is waiting in PreBind, it will re-evaluate the ResourceClaim's condition status.
Since readiness is tracked via API server state, the scheduler can resume waiting or proceed based on the current values.
Controllers should avoid transient or inconsistent condition states that could lead to incorrect scheduling decisions.
After a scheduler restart, if the device attachment is not yet complete, the scheduler will wait again at PreBind.
If the attachment is complete, it will pass through PreBind.

**Scheduler does not guarantee to pick up the same node for the Pod after the reschedule (after binding failure)**

Basically scheduler should select the same node, however we need to consider the following scenarios:
 - In case of a failure, we might want to try a different node.
 - During rescheduling, if another pod is deployed on that node and uses the resources, the rescheduled pod might not be able to be deployed.
   Therefore, we need logic to prioritize the rescheduled pod on that node.

[Node nomination](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#user-exposed-information) would solve this.
If node nomination is not available, processing flow is as follows.
If the pod is assigned to another node after the scheduler restarts, additional device will be attached to that node.
If the device attached to the original node is not used, user can manually detach the device.
(Of course, we can leave it attached to that node for future use by the Pod.)

This issue needs to be resolved before the beta is released.

**Pods which are not bound yet (in api-server) and not unschedulable (in api-server) are not visible by cluster autoscaler, so there is a risk that the node will be turned down.**

Regarding collaboration with the Cluster Autoscaler, using node nomination can address the issue.
This issue needs to be resolved before the beta is released.

**The in-flight events cache may grow too large when waiting in PreBind.**

To address the PreBind concern, the solution is to modify the scheduling framework to flush the in-flight events cache before PreBind.
This prevents issues in the scheduling queue caused by keeping pods at PreBind for an extended period.
This issue will be addressed separately as outlined in kubernetes/kubernetes#129967.
This issue needs to be resolved before the beta is released.

## Design Details

### DRA Scheduler Plugin Design Overview

This document outlines the design of the DRA Scheduler Plugin.
Key additions include `BindingConditions` and `BindingFailureConditions` for device identification and preparation, enhancements to `DeviceRequestAllocationResult`, and the process for handling `ResourceSlice` upon attachment failure.

The diagram below illustrates the interaction between the scheduler, ResourceClaim, and external controller during the PreBind phase.
It shows how the scheduler waits for readiness conditions to be met before proceeding with binding, and how failure conditions or timeouts trigger rescheduling.
This flow ensures that Pods are only bound when the associated device is confirmed to be usable.
![proposal](proposal.jpg)

#### BasicDevice Enhancements

To indicate that the device is a Fabric device or other device that requires some preparation (e.g. attachment), some fields are added to the `Basic` within `Device`.
These fields will be used by the controller that exposes the `ResourceSlice` to notify whether the device is a fabric device.

```go
// BasicDevice represents a basic device instance.
type BasicDevice struct {
  ...
	// UsageRestrictedToNode indicates if the usage of an allocation involving this device
	// has to be limited to exactly the node that was chosen when allocating the claim.
	//
	// This is an alpha field and requires enabling the DRADeviceBindingConditions and DRAResourceClaimDeviceStatus
	// feature gates.
	//
	// +optional
	// +featureGate=DRADeviceBindingConditions,DRAResourceClaimDeviceStatus
	UsageRestrictedToNode *bool

	// BindingConditions defines the conditions for proceeding with binding.
	// All of these conditions must be set in the per-device status
	// conditions with a value of True to proceed with binding the pod to the node
	// while scheduling the pod.
	//
	// The maximum number of binding conditions is 4.
	// All entries are condition types, which means
	// they must be labels.
	//
	// This is an alpha field and requires enabling the DRADeviceBindingConditions and DRAResourceClaimDeviceStatus
	// feature gates.
	//
	// +optional
	// +listType=atomic
	// +featureGate=DRADeviceBindingConditions,DRAResourceClaimDeviceStatus
	BindingConditions []string

	// BindingFailureConditions defines the conditions for binding failure.
	// They may be set in the per-device status conditions.
	// If any is true, a binding failure occurred.
	//
	// The maximum number of binding conditions is 4.
	// All entries are condition types, which means
	// they must be labels.
	//
	// This is an alpha field and requires enabling the DRADeviceBindingConditions and DRAResourceClaimDeviceStatus
	// feature gates.
	//
	// +optional
	// +listType=atomic
	// +featureGate=DRADeviceBindingConditions,DRAResourceClaimDeviceStatus
	BindingFailureConditions []string

	// BindingTimeoutSeconds indicates the prepare timeout period.
	// If the timeout period is exceeded before all BindingConditions reach a True state,
	// the scheduler clears the allocation in the ResourceClaim and reschedules the Pod.
	//
	// The default timeout if not set is 600 seconds.
	//
	// No matter what timeouts were specified by the driver, the scheduler will not wait
	// longer than 20 minutes. This may change.
	//
	// This is an alpha field and requires enabling the DRADeviceBindingConditions and DRAResourceClaimDeviceStatus
	// feature gates.
	//
	// +optional
	// +listType=atomic
	// +featureGate=DRADeviceBindingConditions,DRAResourceClaimDeviceStatus
	BindingTimeoutSeconds *int64
}

const (
    BindingConditionsMaxSize    = 4
    BindingFailureConditionsMaxSize = 4
)
```

#### DeviceRequestAllocationResult Enhancements

The `BindingConditions` and `BindingFailureConditions` fields within `DeviceRequestAllocationResult` are used to indicate the status of the device attachment.
These fields will contain a list of conditions, each representing a specific state or event related to the device.

For this feature, following fields are added:

```go
// DeviceRequestAllocationResult contains the allocation result for one request.
type DeviceRequestAllocationResult struct {
 ...
	// BindingConditions contains a copy of the BindingConditions
	// from the corresponding ResourceSlice at the time of allocation.
	//
	// This is an alpha field and requires enabling the DRADeviceBindingConditions and DRAResourceClaimDeviceStatus
	// feature gates.
	//
	// +optional
	// +listType=atomic
	// +featureGate=DRADeviceBindingConditions,DRAResourceClaimDeviceStatus
	BindingConditions []string

	// BindingFailureConditions contains a copy of the BindingFailureConditions
	// from the corresponding ResourceSlice at the time of allocation.
	//
	// This is an alpha field and requires enabling the DRADeviceBindingConditions and DRAResourceClaimDeviceStatus
	// feature gates.
	//
	// +optional
	// +listType=atomic
	// +featureGate=DRADeviceBindingConditions,DRAResourceClaimDeviceStatus
	BindingFailureConditions []string

	// BindingTimeoutSeconds contains a copy of the BindingTimeoutSeconds
	// from the corresponding ResourceSlice at the time of allocation.
	//
	// This is an alpha field and requires enabling the DRADeviceBindingConditions and DRAResourceClaimDeviceStatus
	// feature gates.
	//
	// +optional
	// +featureGate=DRADeviceBindingConditions,DRAResourceClaimDeviceStatus
	BindingTimeoutSeconds *int64
}
```

#### Scheduler DRA Plugin Modifications

When `UsageRestrictedToNode: true` is set, the scheduler DRA plugin will perform the following steps:

1. **Set NodeSelector**: Before the `PreBind` phase, add the `NodeName` to the `ResourceClaim`'s `NodeSelector`.

If Conditions are present, the scheduler DRA plugin will perform the following steps during the `PreBind` phase:

2. **Copy Conditions**: Copy `UsageRestrictedToNode`, `BindingTimeout`, `BindingConditions` and `BindingFailureConditions` from `ResourceSlice.Device.Basic` to `DeviceRequestAllocationResult`.
3. **Wait for Conditions**: Wait for the following conditions:
   - Wait until all conditions in the BindingConditions are `True` before proceeding to Bind.
   - If any one of the conditions in the BindingFailureConditions becomes `True`, clear the allocation in the `ResourceClaim` and reschedule the Pod.
   - If the preparation of a device takes longer than the `BindingTimeout` period, clear the allocation in the `ResourceClaim` and reschedule the Pod.

To support these steps, for example, a DRA driver can include the following definitions in BindingConditions or BindingFailureConditions within a ResourceSlice:

```go
const (
    // IsPrepared indicates the device ready state.
    // If NeedToPreparing is True and IsPrepared is True, the scheduler proceeds to Bind.
    IsPrepared = "dra.example.com/is-prepared"

    // PreparingFailed indicates the device preparation failed state.
    // If PreparingFailed is True, the scheduler will clear the allocation in the ResourceClaim and reschedule the Pod.
    PreparingFailed = "dra.example.com/preparing-failed"
)
```

Note: There is a concern that the in-flight events cache may grow too large when waiting in PreBind.
To address this, the scheduling framework will be modified to flush the in-flight events cache before PreBind.
This prevents issues in the scheduling queue caused by keeping pods at PreBind for an extended period.
This issue will be addressed separately as outlined in kubernetes/kubernetes#129967.

#### PreBind Phase Timeout

If the device attachment is successful, we expect it to take no longer than 5 minutes.
However, to account for potential update lags, we can set a timeout in the ResourceSlice.
if it's not present in the ResourceSlice, the scheduler has 10 minutes as the default timeout.

Even if the conditions indicating that the device is attached or that the attachment failed are not updated, setting a timeout will prevent the scheduler from waiting indefinitely in the PreBind phase.

#### Handling ResourceSlices Upon Failure of Attachment

If device preparation fails — for example, due to fabric contention, hardware error, or controller-side timeout — the external controller (e.g., a composable DRA controller) should update the condition status in the `ResourceClaim`'s `allocationResult` to reflect the failure.
This is typically done by setting one or more `BindingFailureConditions` to `True`.

The scheduler will detect this during the `PreBind` phase and respond by:

1. **Aborting the binding cycle**: The Pod will not be bound to the node.
2. **Clearing the allocation**: The `ResourceClaim` is unbound from the `ResourceSlice`, making the resource available for other Pods.
3. **Requeuing the Pod**: The Pod is returned to the scheduling queue for reconsideration.

This failure-handling path is intended as an exception, not the primary scheduling model.
The preferred approach is for the scheduler to wait until all `BindingConditions` are satisfied, and only proceed with binding when the device is confirmed to be ready.

By modeling readiness explicitly and handling failures gracefully, this mechanism improves scheduling reliability and avoids leaving Pods in unrecoverable states.

### Alternative approach
This alternative approach is specific to Composable Disaggregated Infrastructure (CDI) environments.
It provides a different way to handle device readiness for fabric-attached devices, and is not intended as a general replacement for BindingConditions.

This section introduces another possible approach: using a "device autoscaler" to manage device attachment and detachment, similar to how the Cluster Autoscaler adds or removes nodes.
Instead of having the scheduler wait for devices to become ready, the autoscaler would monitor unschedulable Pods and try to attach the necessary devices in the background.
This approach could work alongside the BindingConditions mechanism, but it follows a different idea — letting the system react to scheduling failures instead of waiting for readiness before scheduling.

The key points and main process flow of this alternative proposal are as follows:

The scheduler references only node-local ResourceSlices.
If there are no available resources in the node-local ResourceSlices, the scheduler marks the Pod as unschedulable without waiting in the PreBind phase of the ResourceClaim.
And then, device autoscaler tries to attach new devices.
And it also try to detach devices if they have not been used for a period of time.
This is similar to the concept of CA.

However, if CA and device autoscaler is running independently, CA may add a node with a device at the same time as the device autoscaler attaches the device.
This is a wasted resource addition.
Therefore, there is the following idea that putting device-scale functionality in CA.

To handle fabric resources in CA, we implement the Processor for composable system within CA.
This Processor identifies unschedulable Pods and determines if attaching a fabric ResourceSlice device to an existing node would make scheduling possible.
If so, the Processor instructs the attachment of the resource, using the composable Operator for the actual attachment process.
If attaching the fabric ResourceSlice does not make scheduling possible, the Processor determines whether to add a new node as usual.

After the device is attached, the vendor DRA updates the node-local ResourceSlices.
The vendor DRA needs a rescan function to update the Pool/ResourceSlice.
The scheduler can then assign the node-local ResourceSlice devices to the unschedulable Pod, operating the same as the usual DRA from this point.

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*
The goal is to ensure that we don't accept enhancements with inadequate testing.

All code is expected to have adequate tests (eventually with coverage
expectations). Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

<!--
Based on reviewers feedback describe what additional tests need to be added prior
implementing this enhancement to ensure the enhancements have also solid foundations.
-->

##### Unit tests

<!--
In principle every added code should have complete unit test coverage, so providing
the exact set of tests will not bring additional value.
However, if complete unit test coverage is not possible, explain the reason of it
together with explanation why this is acceptable.
-->

<!--
Additionally, for Alpha try to enumerate the core package you will be touching
to implement this enhancement and provide the current unit coverage for those
in the form of:
- <package>: <date> - <current test coverage>
The data can be easily read from:
https://testgrid.k8s.io/sig-testing-canaries#ci-kubernetes-coverage-unit

This can inform certain test coverage improvements that we want to do before
extending the production code to implement this enhancement.
-->

- `<package>`: `<date>` - `<test coverage>`
- `k8s.io/kubernetes/pkg/scheduler/framework/plugins/dynamicresources`: `2025-05-26` - `79.3`
- `k8s.io/dynamic-resource-allocation/resourceclaim`: `2025-05-26` : `90.9`
- `k8s.io/dynamic-resource-allocation/resourceslice`: `2025-05-26` : `75.3` 
- `k8s.io/dynamic-resource-allocation/structured`: `2025-05-26` : `91.4` 

##### Integration tests

<!--
Integration tests are contained in k8s.io/kubernetes/test/integration.
Integration tests allow control of the configuration parameters used to start the binaries under test.
This is different from e2e tests which do not allow configuration of parameters.
Doing this allows testing non-default options and multiple different and potentially conflicting command line options.
-->

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
-->

- <test>: <link to test coverage>

- Simulate a controller updating ResourceClaim conditions
- Verify scheduler behavior on success, failure, and timeout


##### e2e tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html

We expect no non-infra related flakes in the last month as a GA graduation criteria.
-->

- <test>: <link to test coverage>

### Graduation Criteria

#### Alpha

- Initial implementation is completed and enabled

#### Beta

- Gather feedback from developers and surveys
- Resolve the following issues
  - Scheduler does not guarantee to pick up the same node for the Pod after the restart
  - If Scheduler picks up another node for the Pod after the restart, devices are unnecessarily left on the original nodes
    (Composable DRA controller needs to have the function to detach a device automatically if it is not used by a Pod for a certain period of time)
  - Pods which are not bound yet (in api-server) and not unschedulable (in api-server) are not visible by cluster autoscaler, so there is a risk that the node will be turned down
  - The in-flight events cache may grow too large when waiting in PreBind
- Additional tests are in Testgrid and linked in KEP

#### GA

TBD

#### Deprecation
<!--
- Announce deprecation and support policy of the existing flag
- Two versions passed since introducing the functionality that deprecates the flag (to address version skew)
- Address feedback on usage/changed behavior, provided on GitHub issues
- Deprecate the flag
-->

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to maintain previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to make use of the enhancement?
-->

### Version Skew Strategy

<!--
If applicable, how will the component handle version skew with other
components? What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- Does this enhancement involve coordinating behavior in the control plane and nodes?
- How does an n-3 kubelet or kube-proxy without this feature available behave when this feature is used?
- How does an n-1 kube-controller-manager or kube-scheduler without this feature available behave when this feature is used?
- Will any other components on the node change? For example, changes to CSI,
  CRI or CNI may require updating that component before the kubelet.
-->

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness.

The production readiness review questionnaire must be completed and approved
for the KEP to move to `implementable` status and be included in the release.

In some cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce review
burden and latency.

The KEP must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance.
-->

### Feature Enablement and Rollback

<!--
This section must be completed when targeting alpha to a release.
-->

###### How can this feature be enabled / disabled in a live cluster?

<!--
Pick one of these and delete the rest.

Documentation is available on [feature gate lifecycle] and expectations, as
well as the [existing list] of feature gates.

[feature gate lifecycle]: https://git.k8s.io/community/contributors/devel/sig-architecture/feature-gates.md
[existing list]: https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/
-->

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: DRADeviceBindingConditions
  - Components depending on the feature gate: kube-scheduler
- [ ] Other
  - Describe the mechanism:
  - Will enabling / disabling the feature require downtime of the control
    plane?
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node?

###### Does enabling the feature change any default behavior?

<!--
Any change of default behavior may be surprising to users or break existing
automations, so be extremely careful here.
-->
No.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

<!--
Describe the consequences on existing workloads (e.g., if this is a runtime
feature, can it break the existing applications?).

Feature gates are typically disabled by setting the flag to `false` and
restarting the component. No other changes should be necessary to disable the
feature.

NOTE: Also set `disable-supported` to `true` or `false` in `kep.yaml`.
-->
Yes. No existing claims or running pods will be affected.
This feature affects only the allocation of devices during scheduling/binding.

###### What happens if we reenable the feature if it was previously rolled back?

The feature will begin working again.
If a device that needs to be attached is selected, PreBind will wait for the device to be attached.

###### Are there any tests for feature enablement/disablement?

<!--
The e2e framework does not currently support enabling or disabling feature
gates. However, unit tests in each component dealing with managing data, created
with and without the feature, are necessary. At the very least, think about
conversion tests if API types are being modified.

Additionally, for features that are introducing a new API field, unit tests that
are exercising the `switch` of feature gate itself (what happens if I disable a
feature gate after having objects written with the new field) are also critical.
You can take a look at one potential example of such test in:
https://github.com/kubernetes/kubernetes/pull/97058/files#diff-7826f7adbc1996a05ab52e3f5f02429e94b68ce6bce0dc534d1be636154fded3R246-R282
-->

Unit tests will be written.

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->
Will consider in the beta timeframe.
###### How can a rollout or rollback fail? Can it impact already running workloads?

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?

Be sure to consider highly-available clusters, where, for example,
feature flags will be enabled on some API servers and not others during the
rollout. Similarly, consider large clusters and how enablement/disablement
will rollout across nodes.
-->
Will consider in the beta timeframe.
###### What specific metrics should inform a rollback?

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->
Will consider in the beta timeframe.
###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->
Will consider in the beta timeframe.
###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!--
Even if applying deprecation policies, they may still surprise some users.
-->
Will consider in the beta timeframe.
### Monitoring Requirements

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->
Will consider in the beta timeframe.
###### How can an operator determine if the feature is in use by workloads?

<!--
Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
checking if there are objects with field X set) may be a last resort. Avoid
logs or events for this purpose.
-->
Will consider in the beta timeframe.
###### How can someone using this feature know that it is working for their instance?

<!--
For instance, if this is a pod-related feature, it should be possible to determine if the feature is functioning properly
for each individual pod.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so that they can verify correct enablement
and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->
Will consider in the beta timeframe.

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
Will consider in the beta timeframe.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->
Will consider in the beta timeframe.

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
Will consider in the beta timeframe.

### Dependencies

<!--
This section must be completed when targeting beta to a release.
-->
Will consider in the beta timeframe.

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

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### Will enabling / using this feature result in any new API calls?

<!--
Describe them, providing:
  - API call type (e.g. PATCH pods)
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller)
Focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)
-->

###### Will enabling / using this feature result in introducing new API types?

<!--
Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
-->

###### Will enabling / using this feature result in any new calls to the cloud provider?

<!--
Describe them, providing:
  - Which API(s):
  - Estimated increase:
-->

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

<!--
Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

<!--
Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between
(e.g. need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos
-->

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

<!--
Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc.
This through this both in small and large cases, again with respect to the
[supported limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
-->

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

<!--
Focus not just on happy cases, but primarily on more pathological cases
(e.g. probes taking a minute instead of milliseconds, failed pods consuming resources, etc.).
If any of the resources can be exhausted, how this is mitigated with the existing limits
(e.g. pods per node) or new limits added by this KEP?

Are there any tests that were run/should be run to understand performance characteristics better
and validate the declared limits?
-->

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

<!--
Major milestones in the lifecycle of a KEP should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded
-->

- 2024-12: Initial proposal drafted and discussed in SIG scheduling, WG device management.
- 2025-03: KEP revised to introduce BindingConditions as a general mechanism.
- 2025-05: Updated KEP submitted for review.


## Drawbacks

<!--
Why should this KEP _not_ be implemented?
-->

- Adds complexity to the scheduler's PreBind phase, which may impact scheduling latency in large clusters.
- Relies on external controllers to update readiness conditions correctly and promptly. Misbehaving controllers could cause Pods to be stuck or rescheduled unnecessarily.
- May introduce subtle bugs if condition semantics are not well-defined or consistently implemented across different device types.
- Requires careful coordination with Cluster Autoscaler and other scheduling extensions to avoid conflicts or inefficiencies.

## Alternatives

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

- Implementing readiness checks entirely outside the scheduler, using a controller that marks Pods as schedulable only when devices are ready. This would avoid modifying the scheduler but reduce scheduling flexibility.
- Using taints and tolerations to delay scheduling until devices are ready. This approach is less granular and harder to manage dynamically.
- Relying solely on retry mechanisms after Pod failure, without modeling readiness explicitly. This leads to poor user experience and inefficient resource usage.

## Infrastructure Needed (Optional)

<!--
Use this section if you need things from the project/SIG. Examples include a
new subproject, repos requested, or GitHub details. Listing these here allows a
SIG to get the process for these resources started right away.
-->
