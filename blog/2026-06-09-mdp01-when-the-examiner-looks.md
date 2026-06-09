---
layout: post
title: "When the examiner looks"
date: 2026-06-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [compliance, ledger, GDPR, trust-routing, observer-failure]
---

There is a gap between what the compliance evidence endpoint promises and what it actually delivers. Issue #44 closed that gap. Two related issues came with it.

The original problem: if `AmlTrustRoutingObserver.onWorkerDecision()` throws — constraint violation, DB error, anything — the attestation for that capability disappears silently. `@ObservesAsync` swallows exceptions by default. The compliance evidence endpoint then tells an examiner that trust routing is `PARTIAL`, as if the investigation went wrong. It didn't. The routing happened; the record just didn't make it.

The fix is two things working together. The observer now follows PP-20260530-49856c — a double-try/catch where any failure writes an explicit `observerFailed=true` entry instead of vanishing. And when compliance evidence is read, a new `AmlAttestationReconciler` fills any remaining gap lazily, pulling the authoritative trust scores from `WorkerDecisionEntry` (the engine's own record of what it dispatched). The choice to reconcile on read rather than in a background job came down to inspection semantics: the evidence endpoint is where an examiner looks; the response should be complete when they look at it.

The reconciled entries are flagged — `reconstructed=true` in the response, `PARTIAL` status rather than `CLOSED`. The examiner can see the difference between a live attestation and a repaired one. That matters for a compliance system.

Issue #56 came out of the same work. The Layer 3 synchronous path (via `AmlInvestigationCoordinator`) called `openReview()` and then separately called `ledgerService.writeComplianceReviewOpened()`. The engine path (Quartz workers) called only `openReview()`. The ledger entry never got written. SLA requirement: `GAP`, always, for any investigation run through the engine. This was a known deferral. Moving `writeComplianceReviewOpened()` into `openReview()` itself closes it — both paths write the entry now, regardless of which called it.

That refactoring required `caseId` to be available inside the sar-drafting worker. The spec called for `caseHub.signal(caseId, "caseId", ...)` immediately after `startCase()` returns. Claude read the bytecode before implementing it and came back with a correction: `signal()` isn't synchronous. It calls `eventBus.publish()` — fire-and-forget on the Vert.x event bus, returning before the blackboard updates. `WorkerExecutionContext.current().caseId()` is the correct API — the engine populates that thread-local before any worker executes, guaranteed. The spec was wrong; the implementation is right.

Issue #55 was the GDPR demo gap. Every ledger entry carried `actorId = "aml-orchestrator"`. GDPR Art.17 erasure existed but had no real PII to erase. The compliance officer who approves a SAR is a human. A new `AmlSarOfficerReviewedLedgerEntry` now records that decision with the officer's actorId, and a new `AmlWorkItemLifecycleObserver` writes it when the officer completes or rejects the WorkItem via `WorkItemLifecycleEvent`. The GDPR test flow now works end-to-end: start investigation, officer approves, `POST /api/actors/{officerId}/erasure`, verify the actorId is pseudonymized in the next read.

One constraint surfaced during migrations: H2 doesn't support partial unique indexes even in `MODE=PostgreSQL`. The V2009 migration was designed with a `WHERE reconstructed = TRUE` clause on the unique index to prevent multi-JVM duplicate reconciliation. H2 rejected it at startup. The index was dropped from the migration; it's tracked as aml#57 for production PostgreSQL. The application-level lock handles the single-JVM case; multi-JVM idempotency is a production-only concern.

The other API correction came from implementing `AmlWorkItemLifecycleObserver`: `WorkItemLifecycleEvent` has no `workItem()` method. It exposes the entity via `event.source()` cast to `WorkItem`. The factory method `of(...)` also NPEs if `workItem` is null — tests that need a null-snapshot event must use `fromWire()` instead.
