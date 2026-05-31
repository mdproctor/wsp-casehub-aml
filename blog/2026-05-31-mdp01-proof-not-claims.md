---
layout: post
title: "Proof, not claims"
date: 2026-05-31
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [compliance, FinCEN, FATF, Merkle, JAX-RS, Quarkus, ledger]
---

Layer 7 had a clear framing: for each FinCEN/FATF requirement the architecture claims to close, surface the evidence an examiner can use to verify the claim independently.

The question I kept coming back to was what "evidence" actually means. A `verify() = true` boolean is what the service says about itself — a self-attestation. A regulator asking "prove this investigation happened correctly" needs something they can verify without trusting the service. That's what an inclusion proof is: take the siblings, reconstruct the Merkle root offline, either it matches or it doesn't. No callback required.

I brought Claude in to build the whole thing. The endpoint — `GET /api/investigations/{caseId}/compliance-evidence` — returns requirement-scoped evidence for each FinCEN/FATF obligation: Merkle inclusion proofs for the audit chain, WorkItem fields for the 30-day SLA, worker routing attestations captured at dispatch time.

That last one required a workaround. `WorkerDecisionEntry` (the engine-ledger record for each routing decision) captures which worker was selected but not the trust score that drove the selection. The score drifts as attestations accumulate. So we wrote `AmlTrustRoutingAttestation` — a new ledger entity written by a CDI observer at the moment `WorkerDecisionEvent` fires, freezing the score before it can change.

The first implementation used `@Observes WorkerDecisionEvent`. Compiled cleanly. Zero attestations written. The engine fires these events via `Event.fireAsync()` — synchronous observers are silently skipped with no error, no log, nothing. Claude caught it during integration testing. One annotation:

```java
// Before — silently never called
public void onWorkerDecision(@Observes WorkerDecisionEvent event) { ... }

// After
public void onWorkerDecision(@ObservesAsync WorkerDecisionEvent event) { ... }
```

JAX-RS routing took longer. The implementation used `@Path("/")` as the class-level annotation, with full paths on each method. Sounds reasonable. What actually happens is that `@Path("/")` matches every URI as a prefix. RESTEasy Reactive selects that resource class as a candidate for all requests — including those targeted at more specific resources — finds no matching method, returns 404. The investigation endpoint returned 404. So did `POST /api/layer5/investigations`. The fix: two separate resource classes, each with a fully-specified class-level `@Path`.

A code review surfaced a semantic error in the SLA status logic. When a WorkItem exists but the compliance officer hasn't acted yet — deadline in the future, `completedAt = null` — the status was returning `CLOSED`. The `RequirementStatus` enum defines CLOSED as "requirement demonstrably met with evidence." An open WorkItem hasn't met the SLA; it just hasn't breached it. PARTIAL is correct: mechanism present, evidence not yet complete. The finding mattered — a compliance dashboard reporting an open review as CLOSED is exactly the kind of silent error that erodes trust in compliance tooling.

The Merkle chain doesn't verify cleanly in H2. Pattern-analysis and osint-screening fire as parallel workers; both write ledger entries for the same `subjectId`; H2's unique constraint on `(subject_id, level)` in the Merkle frontier fires. In production PostgreSQL, row-level locking handles concurrent writes. In tests it doesn't. The compliance endpoint reports PARTIAL for audit chain status in the test environment — honest, and the tests are written to accept it.
