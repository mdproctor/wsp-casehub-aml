# Design: Trust Attestation Reconciliation + SAR Officer Reviewed Ledger Entry

**Issues:** casehubio/aml#44 (attestation reconciliation), casehubio/aml#55 (SAR officer reviewed)  
**Branch:** issue-44-attestation-reconciliation  
**Date:** 2026-06-07

---

## 1. Problem Statement

### #44 — Silent attestation gaps

`AmlTrustRoutingObserver.onWorkerDecision()` captures trust scores at routing time via
`@ObservesAsync`. If it fails (constraint violation, DB error, transaction rollback), the
attestation for that capability is silently lost — `@ObservesAsync` swallows exceptions by
default. When `GET /api/investigations/{caseId}/compliance-evidence` is called, the
`TrustRoutingRequirement` reports `PARTIAL` even though routing happened correctly.

No recovery path exists: the missing attestation is permanent unless detected and repaired.

### #55 — Hollow GDPR erasure demo

`POST /api/actors/{actorId}/erasure` exists and works, but every ledger entry carries
`actorId = "aml-orchestrator"` (system actor). No human actor writes a ledger entry during
an investigation, so GDPR Art.17 erasure has no real PII to erase.

The compliance officer who approves a SAR is a human actor. Their decision must be recorded
in the tamper-evident ledger under their identity — both for the audit chain and to give
GDPR erasure meaningful PII to act on.

---

## 2. Design Goals

- Observer failures become visible in the ledger rather than disappearing silently
- Attestation gaps are filled lazily on evidence read using authoritative `WorkerDecisionEntry` data
- Status semantics distinguish "all original" (CLOSED) from "gap repaired" (PARTIAL)
- Officer approval is recorded with the officer's identity as `actorId = HUMAN`
- `AuditChainRequirement` covers the full case lifecycle including the officer decision
- GDPR erasure can be demonstrated against a real human actor
- Observer hardening applies the PP-20260530-49856c pattern to both new observers

---

## 3. Data Model Changes

### 3.1 `AmlTrustRoutingAttestation` — `reconstructed` column (V2009)

```sql
ALTER TABLE aml_trust_routing_attestation
    ADD COLUMN reconstructed BOOLEAN NOT NULL DEFAULT FALSE;
```

- `false` for all entries written by the live observer (success or failure-record)
- `true` for entries written by the reconciler to fill gaps
- Needed to distinguish "score was null at routing time" from "entry was reconstructed later"
  (both have `trustScoreAtRouting = null`; only the boolean makes this unambiguous)

### 3.2 `AmlSarOfficerReviewedLedgerEntry` (V2010)

New `LedgerEntry` subclass. JOINED inheritance per `PP-20260513-ledger-subclass`.

```sql
CREATE TABLE aml_sar_officer_reviewed_ledger_entry (
    id UUID PRIMARY KEY REFERENCES ledger_entry(id),
    review_decision VARCHAR(20) NOT NULL  -- 'APPROVED' | 'REJECTED'
);
```

Entity:
```java
@Entity
@Table(name = "aml_sar_officer_reviewed_ledger_entry")
@DiscriminatorValue("AML_SAR_OFFICER_REVIEWED")
public class AmlSarOfficerReviewedLedgerEntry extends LedgerEntry {
    @Column(name = "review_decision", nullable = false, length = 20)
    public String reviewDecision;
}
```

Standard `LedgerEntry` fields carry the officer identity:
- `actorId` = officer actor ID (from `WorkItemLifecycleEvent.actor()`)
- `actorType` = `ActorType.HUMAN`
- `actorRole` = `"ComplianceOfficer"`
- `subjectId` = `caseId` (same chain as CASE_OPENED and COMPLIANCE_REVIEW_OPENED)
- `causedByEntryId` = `AmlComplianceReviewLedgerEntry.id` for this case (self-derived)
- `sequenceNumber` = next in subject chain (via `AmlLedgerService.nextSequenceNumber`)

---

## 4. Issue #44: Observer Hardening + Lazy Reconciliation

### 4.1 Observer hardening (PP-20260530-49856c)

`AmlTrustRoutingObserver.onWorkerDecision()` gains the double-try/catch pattern.

Pre-try block (pure, no DB): derive `attestationSubject`, acquire per-subject lock reference.

```
boolean attestationWritten = false;
try {
    // cache lookup, policy lookup, build entry, synchronized-saveWithSequence
    attestationWritten = true;
} catch (Exception e) {
    LOG.warnf(e, "AmlTrustRoutingObserver failed caseId=%s cap=%s", ...);
    if (!attestationWritten) {
        try {
            attestationRepo.saveObserverFailureEntry(event, attestationSubject);
        } catch (Exception inner) {
            LOG.errorf(inner, "AUDIT GAP: failure entry also failed caseId=%s cap=%s", ...);
        }
    }
}
```

**`saveObserverFailureEntry()` on `AmlTrustAttestationRepository`** — new `@Transactional(REQUIRES_NEW)` method. Writes an `AmlTrustRoutingAttestation` with:
- `actorRole = "AmlInvestigationOrchestrator-observer-failed"` (PP-20260531-11724b naming)
- `actorId = "aml-orchestrator"`, `actorType = ActorType.SYSTEM`
- `trustScoreAtRouting = null`, `thresholdApplied` from policy provider (re-read in new tx)
- `reconstructed = false`
- sequence assigned via cross-all-subtypes MAX query (same as `saveWithSequence`)

This entry records the observer failure event. The reconciler subsequently detects the
gap and writes a separate `reconstructed = true` entry — two distinct events in the ledger.

### 4.2 `AmlTrustRoutingAttestation` model update

Add `reconstructed` field:
```java
@Column(name = "reconstructed", nullable = false)
public boolean reconstructed = false;
```

### 4.3 `AmlAttestationReconciler` (new `@ApplicationScoped`)

```java
public List<AmlTrustRoutingAttestation> reconcileIfNeeded(
        UUID caseId,
        List<WorkerDecisionEntry> dispatched,
        List<AmlTrustRoutingAttestation> existing)
```

Logic:
1. Build `Set<String> coveredCaps` from existing attestations (any `reconstructed` value)
2. For each `WorkerDecisionEntry` whose `capabilityTag` is NOT in `coveredCaps`:
   - Acquire per-subject lock (same `ConcurrentHashMap<UUID,Object>` pattern as the observer)
   - In a `synchronized` block: call `attestationRepo.saveWithSequence()` with `reconstructed=true`,
     `trustScoreAtRouting=null` (engine#403 not yet landed),
     `selectedWorkerId` from the `WorkerDecisionEntry`,
     `actorRole="AmlInvestigationOrchestrator"`, `actorId="aml-orchestrator"`, `actorType=SYSTEM`
   - Add new entry to result list
3. Return `existing + newly written`

**Idempotent:** coveredCaps check prevents re-reconciling already-covered capabilities,
including `reconstructed=true` entries from a prior call.

**`AmlTrustAttestationRepository.saveWithSequence()`** updated to accept `AmlTrustRoutingAttestation`
with `reconstructed` already set on the entry (caller sets it before passing).

### 4.4 `AmlComplianceEvidenceService` — `buildTrustRouting()` update

```java
private TrustRoutingRequirement buildTrustRouting(UUID caseId) {
    List<WorkerDecisionEntry> dispatched = workerDecisionRepo.findAllByCaseId(caseId);
    List<AmlTrustRoutingAttestation> raw = attestationRepo.findByInvestigationCaseId(caseId);
    List<AmlTrustRoutingAttestation> merged = reconciler.reconcileIfNeeded(caseId, dispatched, raw);
    // ... build RoutingDecisionRecord list from merged ...
    // ... compute status from merged ...
}
```

**Status logic:**

| Condition | Status |
|---|---|
| `dispatched.isEmpty()` | `GAP` |
| All dispatched caps covered by `reconstructed=false` attestations | `CLOSED` |
| Any dispatched cap covered only by `reconstructed=true` attestation | `PARTIAL` |
| A cap is in dispatched but has no attestation after reconcile | `PARTIAL` (reconcile write failed) |

**`RoutingDecisionRecord`** gains `boolean reconstructed` field so the API response is
transparent to the examiner.

---

## 5. Issue #55: ComplianceReviewLifecycle + WorkItem Observer + Ledger Entry

### 5.1 `ComplianceReviewLifecycle.openReview()` — add `caseId` parameter

```java
// Before
public String openReview(SuspiciousTransaction transaction, InvestigationSummary summary)

// After
public String openReview(SuspiciousTransaction transaction, InvestigationSummary summary, UUID caseId)
```

`callerRef` changes from `"aml:investigation/" + transaction.id()` to `"aml:investigation:" + caseId`.

This is a breaking change at all call sites — intentional. All callers must be explicit about
the case identity. The slash-vs-colon separator change also prevents callerRef format confusion.

**Call sites:** both sar-drafting workers in `AmlInvestigationCaseHub`. `caseId` must be
available in the worker input map. If it is not already injected by `AmlEngineCoordinator`
into the blackboard initial state, add it there (confirmed during implementation).

### 5.2 `AmlWorkItemLifecycleObserver` (new `@ApplicationScoped`)

Observes `WorkItemLifecycleEvent`. Writes `AmlSarOfficerReviewedLedgerEntry` when the
compliance officer completes the review WorkItem.

```
@ObservesAsync WorkItemLifecycleEvent event:
  1. Guard: event.status() != DONE → return
  2. Guard: event.workItem() == null → LOG.warn, return
  3. Guard: event.workItem().callerRef not startsWith "aml:investigation:" → return
  4. Parse caseId = UUID.fromString(callerRef after prefix); catch IAE → LOG.warn, return
  5. officerId = event.actor() != null ? event.actor() : "unknown-officer"
  6. reviewDecision = "APPROVED"  (DONE status implies approval for this WorkItem type)
  7. Apply PP-20260530-49856c:
       boolean written = false;
       try { amlLedgerService.writeSarOfficerReviewed(caseId, officerId, reviewDecision); written=true; }
       catch (e) { LOG.warn(...); if (!written) try { amlLedgerService.writeSarOfficerReviewedFailure(caseId, officerId); }
                                                catch (inner) { LOG.errorf("AUDIT GAP: ..."); } }
```

`@ObservesAsync` — decouples ledger write from the casehub-work transaction that fires the event.

### 5.3 `AmlLedgerService` additions

**`writeSarOfficerReviewed(UUID caseId, String officerId, String reviewDecision)`**

Writes `AmlSarOfficerReviewedLedgerEntry`:
- `actorId = officerId`, `actorType = HUMAN`, `actorRole = "ComplianceOfficer"`
- `subjectId = caseId`, `sequenceNumber = nextSequenceNumber(caseId)`
- `causedByEntryId` — self-derived: query `findBySubjectId(caseId)`, find first
  `AmlComplianceReviewLedgerEntry`, take its `id`
- `reviewDecision = reviewDecision`

**`writeSarOfficerReviewedFailure(UUID caseId, String officerId)`**

Observer failure entry following PP-20260531-11724b:
- `actorRole = "ComplianceOfficer-observer-failed"`, `actorId = "aml-orchestrator"`, `actorType = SYSTEM`
- `reviewDecision = "UNKNOWN"`, same subject/sequence logic

Both methods are `@Transactional(TxType.REQUIRED)` (run in caller's transaction).
`writeComplianceReviewOpened()` is already `@Transactional` via `LedgerEntryRepository.save()`.

### 5.4 `AmlComplianceEvidenceService` — include SAR_OFFICER_REVIEWED in audit chain

`findEvidence()` and `build()` already load all `findBySubjectId(caseId)` entries.
Add a `filterSarOfficerReviewed()` helper (parallel to `filterCaseOpened`, `filterComplianceReview`).

`buildAuditChain()` includes these entries as `"SAR_OFFICER_REVIEWED"` events in the
`LedgerEventRecord` list. Event record uses `entry.actorId` (the officer) and `entry.actorRole`.

**`AuditChainRequirement.status` update:**

| Condition | Status |
|---|---|
| No AML ledger entries | `GAP` |
| Case opened + review opened + all linked + at least one officer review | `CLOSED` |
| All prior conditions except officer review missing | `PARTIAL` |
| Chain not fully verified | `PARTIAL` |

This correctly models a completed investigation: the audit chain is not `CLOSED` until the
officer has reviewed, matching the actual FinCEN evidence requirement.

---

## 6. Flyway Migrations

| Version | Path | What |
|---|---|---|
| V2009 | `db/aml-trust-routing/migration/` | Add `reconstructed` column to `aml_trust_routing_attestation` |
| V2010 | `db/aml-ledger/migration/` | Create `aml_sar_officer_reviewed_ledger_entry` join table |

Both run on the qhorus datasource (all AML `LedgerEntry` subclasses use qhorus).

---

## 7. Testing

### Unit tests (pure Java, no Quarkus)

**`AmlAttestationReconcilerTest`**
- 3 dispatched capabilities, 2 existing attestations → reconciler writes 1 `reconstructed=true`; returns merged list of 3
- All attestations present → reconciler writes nothing; returns original list
- Idempotency: call twice → still 1 reconstructed entry (not 2)

**`AmlWorkItemLifecycleObserverTest`**  
- Status `DONE`, valid `aml:investigation:` callerRef → `writeSarOfficerReviewed` called with correct args
- Status `IN_PROGRESS` → no write
- Null `workItem()` on event → no write, warning logged
- callerRef for different domain → no write
- Invalid caseId UUID in callerRef → no write, warning logged

**`AmlComplianceEvidenceServiceTest`**
- All original attestations → `trustRouting.status = CLOSED`
- One reconstructed attestation → `trustRouting.status = PARTIAL`, `reconstructed=true` in response
- SAR_OFFICER_REVIEWED entry present → `auditChain.status = CLOSED`
- SAR_OFFICER_REVIEWED missing → `auditChain.status = PARTIAL`

### `@QuarkusTest` — full GDPR integration test

New test in `AmlLayer7ResourceTest`:

1. `POST /api/layer6/investigations` with PEP transaction → get `caseId`
2. Poll `GET /api/layer6/investigations/{caseId}` until `status=completed`
3. `GET /api/investigations/{caseId}/compliance-evidence` → assert `sla.taskId` present; extract it
4. Claim WorkItem as `"compliance-officer-001"` via `WorkItemService.claim(taskId, "compliance-officer-001")`
5. Complete WorkItem via `WorkItemService.complete(taskId, ...)` → fires `WorkItemLifecycleEvent`
6. Await `@ObservesAsync` delivery (poll compliance evidence, or Awaitility with short timeout)
7. `GET /api/investigations/{caseId}/compliance-evidence` → assert:
   - `auditChain.events` contains `SAR_OFFICER_REVIEWED` with `actorId = "compliance-officer-001"`
   - `auditChain.status = CLOSED`
8. `POST /api/actors/compliance-officer-001/erasure` → assert `200`, `erasedCount >= 1`
9. `GET /api/investigations/{caseId}/compliance-evidence` → assert SAR_OFFICER_REVIEWED event's
   `actorId` is pseudonymized (not `"compliance-officer-001"`)

### `@QuarkusTest` — reconciliation path

New test:
1. Start investigation, poll to complete
2. Directly delete one `aml_trust_routing_attestation` row from the DB (simulate observer failure)
3. `GET /api/investigations/{caseId}/compliance-evidence` → assert `trustRouting.status = PARTIAL`;
   assert missing capability appears in decisions with `reconstructed = true`
4. Call evidence again → assert no duplicate reconstruction (idempotent)

---

## 8. Out of Scope (tracked separately)

- **aml#56** — engine path not writing `COMPLIANCE_REVIEW_OPENED` (`sla.status` always `GAP`)
- **engine#403** — `WorkerDecisionEntry` missing `trustScoreAtRouting`; until landed,
  reconstructed attestations will always have `trustScoreAtRouting = null`

---

## 9. Platform Coherence Review

- Ledger subclass rule (PP-20260513): JOINED inheritance ✅, V2009/V2010 numbering ✅, consumer-owned migrations ✅
- Observer failure pattern (PP-20260530-49856c): both new observers apply double try/catch ✅
- Observer failure naming (PP-20260531-11724b): `actorRole = "<role>-observer-failed"` ✅
- Application-tier rule: all logic requires AML domain knowledge ✅
- Dual-trail audit pattern: `WorkItemLifecycleEvent` path maintains both operational and compliance record ✅
- No foundation primitives re-implemented ✅
