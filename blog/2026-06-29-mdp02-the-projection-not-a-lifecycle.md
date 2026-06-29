---
layout: post
title: "The projection that wasn't a lifecycle"
date: 2026-06-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [domain-model, gdpr, compliance]
---

## The tempting wrong answer

I was about to give `InvestigationStatus` a lifecycle — stored state, transitions, a proper state machine. The reasoning felt sound: casehub-life has `LifeCaseStatus`, clinical has `EligibilityScreeningCaseStatus`, both look like domain lifecycles. AML should have its own.

Except it shouldn't. `InvestigationStatus` isn't stored anywhere. It's computed fresh on every API call from `CaseStatus`. There's no independent state, no transitions, no synchronisation logic. Adding a lifecycle would create a second source of truth alongside the engine's `CaseStatus`, with all the divergence risk that implies — the engine moves to FAULTED, but InvestigationStatus is still IN_PROGRESS until some sync event fires.

The right framing is a **projection**: every `CaseStatus` value maps to exactly one `InvestigationStatus` value. The bug was that the projection was lossy — a single `if (state != COMPLETED)` collapsed FAULTED, CANCELLED, and SUSPENDED into IN_PROGRESS, hiding compliance-significant distinctions. The fix is making the projection exhaustive, not adding state management.

The exhaustive switch with no `default` arm is the key design property. If the engine adds a new `CaseStatus` value, AML fails to compile until someone decides what it means for an investigation. That forced decision is the whole point.

## The erasure that would have been a no-op

The design review caught something I'd missed. The original spec had `AmlErasureService` calling `CaseMemoryStore.eraseEntity(actorId, tenantId)` alongside the ledger erasure — clean memory alongside identity pseudonymisation.

The problem: `eraseEntity()` keys on `entityId`, and AML's memory entries are keyed by **account IDs** (the transaction's origin and destination accounts), not by **actorId** (the ledger identity — a compliance officer or agent). Calling `eraseEntity("compliance-officer-jane", tenantId)` would silently return 0, because no memory entries have that entityId. The API call succeeds, returns a count of zero, and the caller assumes memory was cleaned up. It wasn't.

The fix was removing the call entirely rather than leaving a silent no-op that gives false compliance assurance. Entity-level memory erasure — erasing what the system knows about an investigated account holder — is a genuinely different use case that needs its own endpoint, keyed by account ID rather than actor ID.

The distinction matters because GDPR Art.17 is an obligation you have to demonstrate compliance with. A call that succeeds but does nothing is worse than no call at all — it creates an audit record that says "memory cleaned" when nothing was cleaned.
