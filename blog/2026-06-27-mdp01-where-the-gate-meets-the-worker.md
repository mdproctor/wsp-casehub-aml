---
layout: post
title: "Where the gate meets the worker"
date: 2026-06-27
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [oversight-gate, PlannedAction, CDI, testing]
series: issue-58-separate-sar-drafting
---

The sar-drafting workers had been calling `openReview()` unconditionally — creating a compliance officer WorkItem before any MLRO gate could fire. The design for #58 was always clear: split the side effect out, gate the SAR filing behind `PlannedAction(SAR_FILING)`, and add a post-gate `compliance-review-opening` worker to handle the WorkItem creation.

The constraint I hadn't anticipated was engine#564. `FlowWorkerFunctionHandler` hardcodes `WorkerResult.of(model.asMap())` — the single-argument overload with no `PlannedAction` parameter. There's no way to declare a consequential action from a Flow worker. The sar-drafting workers had to convert from Flow to `WorkerFunction.Sync`, the same pattern `entityLinkProposalWorker` uses in the oversight case hub.

After the conversion, the descriptor has a mixed execution model: 6 of 8 workers use FlowWorkerFunction; the two sar-drafting workers use Sync. The execution model classification test — which previously asserted all workers were Flow — needed a redesign into explicit `FLOW_WORKERS` and `SYNC_WORKERS` sets. A better test than the original, in retrospect. It forces every new worker to be classified rather than assuming homogeneity.

The implementation itself was clean. The real surprise came from the test suite. Every integration test that starts an AML investigation now needs to approve the MLRO gate before the case can drain to completion. I'd updated the Layer 6 tests in the plan but missed that six other test classes also start investigations — Layer 5, Layer 7, Layer 8, SAR outcome, and trust routing attestation tests.

Adding gate approval to those classes fixed most of them. But two kept failing: `AmlLayer7ResourceTest` and `AmlTrustRoutingAttestationTest`. The error — `CaseInstance not found or wrong tenant` — comes from `InMemoryCaseInstanceRepository.update()`, and it only happens with #58's larger case definition. From main's code, both pass.

Digging into CDI, I found that `casehub-engine-persistence-memory` ships six `@Alternative` beans, but AML's test properties only explicitly selected two of them (`MemorySubCaseGroupRepository` and `MemoryPlanItemStore`). The other four — including `InMemoryCaseInstanceRepository` — relied on Quarkus Arc's implicit fallback activation as the sole implementation on the classpath. Explicitly adding all four to `selected-alternatives` fixed four of the six failing test classes.

The remaining two still fail. Layer 7 passes from main's code, passes when Layer 6 runs first in the same JVM, but fails with #58's case definition. The error happens at `startCase()` time — before the gate even fires. Something about the larger case definition (more workers, mixed execution models, new binding) triggers a race in the engine's `CaseStartedEventHandler` → `caseInstanceRepository.update()` path that I haven't been able to isolate.

The implicit `@Alternative` activation issue is now a garden entry (GE-20260531-1e51d4 variant). The Layer 7 root cause is the next session's problem.
