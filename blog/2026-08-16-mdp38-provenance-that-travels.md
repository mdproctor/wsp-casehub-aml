---
layout: post
title: "Provenance That Travels"
date: 2026-08-16
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [provenance, w3c, prov-dm, fintech, compliance, testing]
---

The `causedByEntryId` chain has been the backbone of the AML investigation audit trail since Layer 4. Every ledger entry links to its causal predecessor — case opened caused compliance review, compliance review caused SAR officer decision. Merkle inclusion proofs make each entry independently verifiable. The chain works. But it only works inside CaseHub.

A FinCEN examiner auditing a SAR filing has no access to the CaseHub API. They need the investigation's decision chain in a format their tools understand. W3C PROV-DM is that format — a standard data model for provenance that any PROV-aware tool (ProvStore, PROV-Viewer, ProvToolbox) can import, visualise, and query.

The mapping decision was the interesting part. PROV-DM has three core concepts: Entity (a thing that exists), Activity (something that happened), and Agent (who was responsible). A ledger entry is *both* a thing that exists (the tamper-evident record) and evidence that something happened (the event it records). I considered three approaches: flat (entries as entities only, `causedByEntryId` as `wasDerivedFrom`), rich (each entry maps to an Entity AND an Activity), and grouped (entries clustered into higher-level phase activities).

I went with the rich model. Each of the ~26 entries in a typical investigation becomes an Entity-Activity pair, with the discriminator value determining the PROV type. `AML_CASE_OPENED` becomes `aml:CaseOpenedRecord` (Entity) and `aml:CaseOpening` (Activity). The tamper-evidence metadata — digest, sequence number, Merkle inclusion proof — sits on the Entity. The domain facts — transaction ID, review decision, routing rationale — sit on the Activity. Agents are deduplicated: same `actorId` produces one Agent node regardless of how many entries it authored.

The flat model would have been simpler but it omits Activities entirely. A PROV-DM consumer can't query "show me all routing decisions" without them. The grouped model would have been more readable but fragile — grouping entries into investigation phases requires inference logic that breaks when a new entry type appears. The rich model is mechanical: each entry maps the same way, new types get a fallback to generic `aml:LedgerRecord` / `aml:LedgerEvent`, and standard tools render the graph correctly without custom handling.

The implementation landed cleanly. `ProvDmMapper` is a pure function — no CDI, no state, fully testable without Quarkus. The `instanceof` pattern matching in the type switches is the right tool here; a visitor would require modifying foundation classes we don't own.

The session's real surprise was in the test fixes. Three integration tests (`AmlLayer5InvestigationTest`, `AmlLayer7ResourceTest`, `AmlCaseProfileStoreObserverTest`) were intermittently timing out under full-suite thread pressure. The obvious diagnosis — Quartz thread pool exhaustion — was wrong for two of the three.

The structural issue was sequential Awaitility blocks splitting their timeout budget. A test that waits 60s for a gate, then 20s for drain, gives the investigation 80s total — but those 80s are sliced into independent countdowns. If the gate takes 50s, the drain starts fresh with only 20s. The fix: combine gate approval and drain into a single polling loop. Gate approval becomes a side-effect inside the poll, guarded by an `AtomicBoolean`. One 60s budget covers the entire sequence.

The third test had a genuine bug hiding behind the timeout. `workItemService.complete()` was being called with outcome `"APPROVED"` — which isn't in the SAR review WorkItem's allowed list (`[file, clear, escalate]`). The first run threw `IllegalArgumentException`. Surefire retried. The retry started in corrupted JVM state, and the entity-resolution worker crashed on missing context. The developer sees a timeout; the real error is a validation failure buried in Run 1's output. Claude caught it by reading the full surefire output instead of just the final failure summary.

The provenance export is at `GET /api/investigations/{caseId}/provenance`. The response is W3C PROV-JSON — self-contained, with Merkle proofs embedded as namespace-prefixed attributes on each Entity. A FinCEN examiner can reconstruct the full decision chain — who was dispatched, what they found, why that agent was selected, who approved the SAR — without ever touching the CaseHub API.
