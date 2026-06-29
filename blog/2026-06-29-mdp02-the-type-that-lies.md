---
layout: post
title: "The type that lies and the key that doesn't match"
date: 2026-06-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [gdpr, compliance, domain-model, erasure]
---

Two issues that looked independent turned out to share a root: data models that quietly misrepresent what they know.

`InvestigationResolution` had two fields — `status` and `outcome`. For completed investigations, `outcome` carried the review decision: SAR filed, gate rejected. For everything else — failed, cancelled, suspended — `outcome` was null. The type said "I have a resolution" while silently withholding the most operationally useful information: *why* the investigation didn't complete.

The failure data existed. The engine's EventLog records every terminal transition — `CASE_FAULTED`, `WORKER_EXECUTION_FAILED`, `ACTION_GATE_REJECTED` — with metadata capturing goal names, worker IDs, error messages. The gap was that `InvestigationResolution` had no way to carry it. The fix is a parallel field: `failureContext` sits alongside `outcome`, populated for FAILED and CANCELLED states, null for COMPLETED. Not a sealed interface — the callers already switch on status, and adding five types to enforce an invariant that one method already guarantees is complexity with no architectural payoff.

The design review caught something I hadn't considered: the engine writes *two* `CASE_FAULTED` EventLog entries for non-goal-triggered faults. `WorkerRetriesExhaustedEventHandler` writes one, then `CaseStatusChangedHandler` writes another via the event bus. Goal-triggered faults produce only one. Without disambiguation, `resolveFailureContext()` would return the wrong timestamp and miss the goal metadata entirely. The rule is simple — iterate all terminal entries, use the earliest timestamp, extract goal metadata from whichever entry has it — but you wouldn't know to write it without seeing the double-write.

The same review caught a semantic error in my original design: I'd mapped SUSPENDED to `FailureContext`. Suspension isn't a failure — it's a pause. The case may resume. Giving it a failure context implies a terminal condition that doesn't exist. Both fields null, same as IN_PROGRESS.

The second issue — entity-level memory erasure — exposed a different kind of mismatch. The existing GDPR endpoint pseudonymises *actor* identity in ledger entries: who performed the investigation. But AML memories are keyed by *entity* ID: which account was investigated. When an account holder exercises their Art.17 right to be forgotten, the actor erasure endpoint does nothing for them — it operates on the wrong key space entirely. The platform already had `CaseMemoryStore.eraseEntity(entityId, tenantId)` for cross-domain hard deletion. The gap was that AML never called it.

The entity erasure writes a tamper-evident receipt to the ledger — the same pattern as actor erasure, but with a deterministic `subjectId` derived from the entity ID. Every erasure for the same entity chains under the same subject, so `findBySubjectId()` returns the complete erasure history. A retry after partial failure produces `memoriesErased=0` and a valid receipt — the zero count means "nothing left to erase," not "nothing happened."

The design review also flagged the V2013 migration's `id` column type. I'd written it as `BIGINT`. Every existing AML ledger subclass migration uses `UUID`. The mismatch would have failed silently in H2 tests and exploded in PostgreSQL.
