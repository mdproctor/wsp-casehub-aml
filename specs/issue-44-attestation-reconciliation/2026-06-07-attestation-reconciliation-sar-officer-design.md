# Design: Trust Attestation Reconciliation + SAR Officer Reviewed Ledger Entry

**Issues:** casehubio/aml#44 (attestation reconciliation), casehubio/aml#55 (SAR officer reviewed)  
**Branch:** issue-44-attestation-reconciliation  
**Date:** 2026-06-07 (revised 2026-06-08)

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

The compliance officer who approves or rejects a SAR is a human actor. Their decision must
be recorded in the tamper-evident ledger under their identity — for the audit chain and to
give GDPR erasure meaningful PII to act on.

---

## 2. Design Goals

- Observer failures become visible in the ledger rather than disappearing silently
- Attestation gaps are filled lazily on evidence read using authoritative `WorkerDecisionEntry` data
- Status semantics distinguish: all original (`CLOSED`), observer-failed cap (`PARTIAL`), gap repaired (`PARTIAL`), no data (`GAP`)
- Officer approval or rejection is recorded with the officer's identity as `actorId = HUMAN`
- `AuditChainRequirement` covers the full case lifecycle including the officer decision
- GDPR erasure can be demonstrated against a real human actor
- Observer hardening applies PP-20260530-49856c to both new observers

---

## 3. Data Model Changes

### 3.1 `AmlTrustRoutingAttestation` — V2009 changes

Two new columns:

```sql
ALTER TABLE aml_trust_routing_attestation
    ADD COLUMN reconstructed   BOOLEAN NOT NULL DEFAULT FALSE,
    ADD COLUMN observer_failed BOOLEAN NOT NULL DEFAULT FALSE;
```

**`reconstructed`**: `true` for entries written by the reconciler to fill a gap. Required to
distinguish "trust score was null at routing time (Phase 0)" from "entry was reconstructed
later" — both have `trustScoreAtRouting = null`.

**`observerFailed`**: `true` for entries written by the observer's outer catch (the failure
audit record). Required to distinguish a live attestation from a failure-marker in status
computation. An observer-failure entry for capability X covers X (`PARTIAL`, not `GAP`), but
must not allow the requirement to reach `CLOSED` — the trust score was never captured.

Partial unique index to prevent multi-JVM double-writes of reconstructed entries:

```sql
CREATE UNIQUE INDEX UQ_TRUST_ATTEST_CASE_CAP_RECONSTRUCTED
    ON aml_trust_routing_attestation (investigation_case_id, capability_tag)
    WHERE reconstructed = TRUE;
```

The reconciler catches the unique-constraint violation and treats it as idempotent (a peer
JVM wrote first).

### 3.2 `AmlSarOfficerReviewedLedgerEntry` — V2010

New `LedgerEntry` subclass. JOINED inheritance per PP-20260513-ledger-subclass.

```sql
CREATE TABLE aml_sar_officer_reviewed_ledger_entry (
    id              UUID    PRIMARY KEY REFERENCES ledger_entry(id),
    review_decision VARCHAR(20) NOT NULL   -- 'APPROVED' | 'REJECTED'
);
```

Entity:

```java
@Entity
@Table(name = "aml_sar_officer_reviewed_ledger_entry")
@DiscriminatorValue("AML_SAR_OFFICER_REVIEWED")
public class AmlSarOfficerReviewedLedgerEntry extends LedgerEntry {
    @Column(name = "review_decision", nullable = false, length = 20)
    public String reviewDecision;   // "APPROVED" | "REJECTED"
}
```

Standard `LedgerEntry` fields:
- `actorId` = officer actor ID (from `WorkItemLifecycleEvent.actor()`)
- `actorType` = `ActorType.HUMAN`
- `actorRole` = `"ComplianceOfficer"`
- `subjectId` = `caseId` (same chain as `AmlCaseOpenedLedgerEntry` and `AmlComplianceReviewLedgerEntry`)
- `causedByEntryId` = `AmlComplianceReviewLedgerEntry.id` for this case (self-derived; may be
  `null` when the engine path has not written `COMPLIANCE_REVIEW_OPENED` — aml#56 not yet
  fixed. Observer writes the entry anyway; `null causedByEntryId` keeps `auditChain.status`
  at `PARTIAL` rather than `CLOSED`.)
- `sequenceNumber` = next in subject chain via `nextSequenceNumber(caseId)`

---

## 4. Issue #44: Observer Hardening + Lazy Reconciliation

### 4.1 Observer hardening (PP-20260530-49856c)

`AmlTrustRoutingObserver.onWorkerDecision()` gains the double-try/catch pattern.

Before the `try` block (pure computation, no DB): derive `attestationSubject`, acquire the
per-subject lock reference from `subjectLocks`.

```
boolean attestationWritten = false;
try {
    // cache lookup, policy lookup, build entry, synchronized-saveWithSequence
    attestationWritten = true;
} catch (Exception e) {
    LOG.warnf(e, "AmlTrustRoutingObserver failed caseId=%s cap=%s workerId=%s",
              event.caseId(), event.capabilityTag(), event.workerId());
    if (!attestationWritten) {
        try {
            attestationRepo.saveObserverFailureEntry(event, attestationSubject);
        } catch (Exception inner) {
            LOG.errorf(inner,
                "AUDIT GAP: observer failure entry also failed caseId=%s cap=%s",
                event.caseId(), event.capabilityTag());
        }
    }
}
```

**`saveObserverFailureEntry()` on `AmlTrustAttestationRepository`** — new
`@Transactional(REQUIRES_NEW)` method. Writes `AmlTrustRoutingAttestation` with:
- `actorRole = "AmlInvestigationOrchestrator-observer-failed"` (PP-20260531-11724b)
- `actorId = "aml-orchestrator"`, `actorType = ActorType.SYSTEM`
- `trustScoreAtRouting = null`, `thresholdApplied = 0.0`
- `reconstructed = false`, `observerFailed = true`
- Sequence assigned via `MAX(sequenceNumber) FROM LedgerEntry WHERE subjectId = :sid`

This entry records the failure event distinctly from both a successful attestation
(`observerFailed=false`) and a reconciled entry (`reconstructed=true`). It covers the
capability in status computation but prevents `CLOSED`.

### 4.2 `AmlTrustRoutingAttestation` model update

Add the two new fields to the entity, with defaults matching the migration:

```java
@Column(name = "reconstructed", nullable = false)
public boolean reconstructed = false;

@Column(name = "observer_failed", nullable = false)
public boolean observerFailed = false;
```

### 4.3 `AmlAttestationReconciler` (new `@ApplicationScoped`)

```java
public List<AmlTrustRoutingAttestation> reconcileIfNeeded(
        UUID caseId,
        List<WorkerDecisionEntry> dispatched,
        List<AmlTrustRoutingAttestation> existing)
```

Logic:

1. Build `Set<String> coveredCaps` from `existing` (any `reconstructed` or `observerFailed` value)
2. For each `WorkerDecisionEntry` whose `capabilityTag` is NOT in `coveredCaps`:
   - Acquire per-subject lock (same `ConcurrentHashMap<UUID, Object>` pattern as the observer)
   - In a `synchronized` block: call `attestationRepo.saveWithSequence()` with:
     - `reconstructed = true`, `observerFailed = false`
     - `trustScoreAtRouting = decisionEntry.trustScoreAtRouting` (may be null — Phase 0)
     - `thresholdApplied = decisionEntry.thresholdApplied != null ? decisionEntry.thresholdApplied : 0.0`
     - `selectedWorkerId = decisionEntry.workerId`
     - `capabilityTag = decisionEntry.capabilityTag`
     - `actorRole = "AmlInvestigationOrchestrator"`, `actorId = "aml-orchestrator"`, `actorType = SYSTEM`
   - On `ConstraintViolationException` (unique index violation): another JVM already
     reconciled — treat as idempotent, log at DEBUG, continue
   - Add new entry to result list
3. Return `existing + newly written`

**Source data note:** `WorkerDecisionEntry.trustScoreAtRouting` and `.thresholdApplied` are
populated by the engine's `WorkerDecisionEventCapture` from `TrustScoreCache` at event time.
These are the same values `AmlTrustRoutingObserver` would have captured had it succeeded.
Reconstruction is therefore faithful — the reconciler doesn't approximate, it uses the
engine's own captured values. (Note: `thresholdApplied` is `Double` on
`WorkerDecisionEntry` — nullable — but `double` on `AmlTrustRoutingAttestation` — primitive,
non-nullable. Null maps to `0.0` for Phase-0 cases where no threshold was applied.)

**Why `AmlTrustRoutingAttestation` still exists alongside `WorkerDecisionEntry`:**
`WorkerDecisionEntry` is the engine's operational record. `AmlTrustRoutingAttestation`
is the AML domain's explicit compliance commitment, in its own Merkle chain under a
namespaced subject (preventing sequence collision with case lifecycle entries). They answer
different questions: the engine records what happened; AML records its compliance
accountability.

### 4.4 Status logic

**`TrustRoutingRequirement` status** (from merged attestation list):

| Condition | Status |
|---|---|
| `dispatched.isEmpty()` | `GAP` |
| All dispatched caps have `observerFailed=false, reconstructed=false` attestations | `CLOSED` |
| Any dispatched cap has only `observerFailed=true` coverage | `PARTIAL` |
| Any dispatched cap has only `reconstructed=true` coverage | `PARTIAL` |
| A cap remains uncovered after reconcile (reconcile write also failed) | `PARTIAL` |

**`RoutingDecisionRecord`** gains two boolean fields: `reconstructed` and `observerFailed`.
Both are exposed in the API response so an examiner can distinguish authoritative, failed,
and repaired attestations without inspecting `actorRole`.

### 4.5 `AmlComplianceEvidenceService` changes

Constructor gains a sixth parameter — `AmlAttestationReconciler reconciler`:

```java
@Inject
public AmlComplianceEvidenceService(
        LedgerEntryRepository ledgerRepo,
        LedgerVerificationService verificationService,
        AmlTrustAttestationRepository attestationRepo,
        AmlWorkerDecisionRepository workerDecisionRepo,
        EntityManager em,
        AmlAttestationReconciler reconciler) { ... }
```

`buildTrustRouting()` updated:

```java
List<WorkerDecisionEntry> dispatched = workerDecisionRepo.findAllByCaseId(caseId);
List<AmlTrustRoutingAttestation> raw = attestationRepo.findByInvestigationCaseId(caseId);
List<AmlTrustRoutingAttestation> merged = reconciler.reconcileIfNeeded(caseId, dispatched, raw);
// compute status from merged using §4.4 logic
```

**Why lazy on-read rather than a background job or repair endpoint:** The compliance
evidence endpoint is the natural inspection point — an examiner reading it should see a
complete picture. Lazy repair ensures the response is self-consistent: gaps are filled
atomically in the same request. A background job would create a window where a gap is
visible but not yet repaired. An explicit repair endpoint adds operational ceremony for a
system with no human operators. The cost is a GET with occasional write side-effects; this
is explicit in the endpoint contract and acceptable for a compliance inspection surface.

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

Two things change: the separator (`/` → `:`) and the identity (`transaction.id` → `caseId`).
Both are intentional: `caseId` is the stable investigation UUID (not the external transaction
reference), and `:` avoids confusion with path-style refs. **In-flight WorkItems** created
before deployment carry the old `"aml:investigation/"` format and will be silently ignored
by `AmlWorkItemLifecycleObserver` guard #3. This is a hard cutover with no back-fill — acceptable
for a tutorial system with no production data.

`WorkItemCallerRef` exists in the API if callerRef lookup by prefix is ever needed externally.
Not needed for this design.

### 5.2 `AmlWorkItemLifecycleObserver` (new `@ApplicationScoped`)

Observes `WorkItemLifecycleEvent`. Handles both `COMPLETED` and `REJECTED` officer decisions.

```
@ObservesAsync WorkItemLifecycleEvent event:
  1. Guard: event.status() not in {COMPLETED, REJECTED} → return
  2. Guard: event.workItem() == null → LOG.warn("no WorkItem snapshot in event"), return
  3. Guard: event.workItem().callerRef == null
             || !event.workItem().callerRef.startsWith("aml:investigation:") → return
  4. Parse caseId = UUID.fromString(callerRef.substring("aml:investigation:".length()))
     On IllegalArgumentException: LOG.warn, return
  5. officerId = event.actor() != null ? event.actor() : "unknown-officer"
  6. reviewDecision = event.status() == COMPLETED ? "APPROVED" : "REJECTED"
  7. Apply PP-20260530-49856c:
       boolean written = false;
       try {
           amlLedgerService.writeSarOfficerReviewed(caseId, officerId, reviewDecision);
           written = true;
       } catch (Exception e) {
           LOG.warnf(e, "Failed to write SAR_OFFICER_REVIEWED caseId=%s officer=%s", caseId, officerId);
           if (!written) {
               try { amlLedgerService.writeSarOfficerReviewedFailure(caseId, officerId); }
               catch (Exception inner) {
                   LOG.errorf(inner, "AUDIT GAP: SAR_OFFICER_REVIEWED observer failure also failed caseId=%s", caseId);
               }
           }
       }
```

`@ObservesAsync` — decouples the ledger write from the casehub-work transaction.
`WorkItemService.complete()` and `.reject()` fire both `lifecycleEvent.fire()` (sync) and
`lifecycleEvent.fireAsync()` (async). This observer receives the async delivery.

**REJECTED handling:** a rejected SAR is a compliance decision. The examiner must see it in
the tamper-evident record. Not recording a rejection would leave the audit chain in `PARTIAL`
permanently after an officer acts. `reviewDecision = "REJECTED"` makes the outcome explicit.
Both `APPROVED` and `REJECTED` entries move `auditChain.status` to `CLOSED`.

### 5.3 `AmlLedgerService` additions

**`writeSarOfficerReviewed(UUID caseId, String officerId, String reviewDecision)`**

```
@Transactional(TxType.REQUIRED)   // starts a new transaction — no transaction is propagated
                                   // from @ObservesAsync context
```

Writes `AmlSarOfficerReviewedLedgerEntry`:
- `actorId = officerId`, `actorType = HUMAN`, `actorRole = "ComplianceOfficer"`
- `subjectId = caseId`, `sequenceNumber = nextSequenceNumber(caseId)`
- `causedByEntryId` — self-derived: query `findBySubjectId(caseId)`, take first
  `AmlComplianceReviewLedgerEntry.id` (null when engine path + aml#56 unresolved)
- `reviewDecision = reviewDecision`

**`writeSarOfficerReviewedFailure(UUID caseId, String officerId)`**

Per PP-20260531-11724b: `actorRole = "ComplianceOfficer-observer-failed"`, `actorId = "aml-orchestrator"`, `actorType = SYSTEM`, `reviewDecision = "UNKNOWN"`.

```
@Transactional(TxType.REQUIRED)   // starts a new transaction (no propagated context)
```

### 5.4 `AmlComplianceEvidenceService` — include SAR_OFFICER_REVIEWED in audit chain

Add `filterSarOfficerReviewed()` helper (parallel to `filterCaseOpened`, `filterComplianceReview`).

`buildAuditChain()` includes `AmlSarOfficerReviewedLedgerEntry` entries as `"SAR_OFFICER_REVIEWED"` events in the `LedgerEventRecord` list, using `entry.actorId` (the officer) and `entry.actorRole`.

**`AuditChainRequirement.status` update:**

| Condition | Status |
|---|---|
| No AML ledger entries | `GAP` |
| Chain verified + all review entries linked + ≥ 1 officer review entry | `CLOSED` |
| Case opened + review opened but no officer review yet | `PARTIAL` |
| Chain not fully verified | `PARTIAL` |
| `causedByEntryId` null on review or officer entry (aml#56) | `PARTIAL` |

### 5.5 `AmlEngineCoordinator` — inject `caseId` into blackboard initial context

`startInvestigation()` currently puts only `"transaction"` and `"priorEntityContext"` into
the initial context map. The sar-drafting workers need `caseId` to pass to `openReview()`.

Add:

```java
initialContext.put("caseId", caseId.toString());
```

before starting the engine case. Workers extract it as:

```java
final UUID caseId = UUID.fromString((String) input.get("caseId"));
```

Both sar-drafting workers (junior and senior) call:

```java
complianceReviewLifecycle.openReview(tx, buildSummary(input, tx, sarNarrative), caseId)
```

---

## 6. Flyway Migrations

| Version | Path | Datasource | What |
|---|---|---|---|
| V2009 | `db/aml-trust-routing/migration/` | qhorus | Add `reconstructed`, `observer_failed` columns; add partial unique index |
| V2010 | `db/aml-ledger/migration/` | qhorus | Create `aml_sar_officer_reviewed_ledger_entry` join table |

---

## 7. Testing

### Unit tests (pure Java, no Quarkus)

**`AmlAttestationReconcilerTest`**
- 3 dispatched capabilities, 2 existing attestations (original) → reconciler writes 1 `reconstructed=true`; result has 3 entries
- All 3 covered → no write; returns original list unchanged
- Idempotency: call twice → 1 reconstructed entry, not 2 (second call: cap already in coveredCaps)
- Observer-failed entry present for cap X → reconciler sees X as covered, writes reconstructed entry
  to replace (no: observer-failed IS in coveredCaps — reconciler skips, X remains PARTIAL not CLOSED)

**`AmlWorkItemLifecycleObserverTest`**
- `COMPLETED` status + valid `aml:investigation:` callerRef → `writeSarOfficerReviewed("APPROVED")` called
- `REJECTED` status + valid callerRef → `writeSarOfficerReviewed("REJECTED")` called
- `IN_PROGRESS` status → no write
- Null `workItem()` on event → no write, warning logged
- callerRef for different domain → no write
- Old-format callerRef (`aml:investigation/`) → no write (hard cutover)
- Invalid UUID in callerRef → no write, warning logged
- `null` actor → falls back to `"unknown-officer"`

**`AmlComplianceEvidenceServiceTest`**

Constructor signature after change (6 args):
```java
service = new AmlComplianceEvidenceService(
        mockLedgerRepo, mockVerificationService,
        mockAttestationRepo, mockWorkerDecisionRepo,
        mockEntityManager, mockReconciler);  // new arg
```

All 5 existing tests require a new `@Mock AmlAttestationReconciler mockReconciler` and a
`when(mockReconciler.reconcileIfNeeded(any(), any(), any())).thenReturn(raw)` stub in setUp.
The existing `attestation()` helper creates entries without setting `reconstructed` or
`observerFailed` — `false` defaults are correct for non-reconstructed originals.

New assertions:
- All original attestations (`observerFailed=false, reconstructed=false`) → `trustRouting.status = CLOSED`
- One `observerFailed=true` attestation → `trustRouting.status = PARTIAL`; `observerFailed=true` in `RoutingDecisionRecord`
- One `reconstructed=true` attestation → `trustRouting.status = PARTIAL`; `reconstructed=true` in record
- SAR_OFFICER_REVIEWED entry present → `auditChain.status = CLOSED`
- SAR_OFFICER_REVIEWED missing → `auditChain.status = PARTIAL`

### `@QuarkusTest` — full GDPR integration test

New test in `AmlLayer7ResourceTest`:

1. `POST /api/layer6/investigations` with PEP transaction → `caseId`
2. Poll `GET /api/layer6/investigations/{caseId}` until `status=completed`
3. `GET /api/investigations/{caseId}/compliance-evidence` → extract `taskId` from `sla`
4. `workItemService.claim(taskId, "compliance-officer-001")` → ASSIGNED
5. `workItemService.start(taskId, "compliance-officer-001")` → IN_PROGRESS
6. `workItemService.complete(taskId, "compliance-officer-001", "SAR approved", "APPROVED")`
   (4-param overload: `id, actorId, resolution, outcome`) → fires both sync and async `WorkItemLifecycleEvent`
7. Await `@ObservesAsync` delivery: poll `GET /api/investigations/{caseId}/compliance-evidence`
   until `auditChain.events` contains `SAR_OFFICER_REVIEWED`, or Awaitility with ≤5s timeout
8. Assert `auditChain.events` contains event with `actorId = "compliance-officer-001"` and
   `actorRole = "ComplianceOfficer"`; assert `auditChain.status = CLOSED`
9. `POST /api/actors/compliance-officer-001/erasure` → assert 200, `erasedCount >= 1`
10. `GET /api/investigations/{caseId}/compliance-evidence` → assert SAR_OFFICER_REVIEWED
    event `actorId` is pseudonymized (not `"compliance-officer-001"`)

### `@QuarkusTest` — reconciliation path

1. Start investigation, poll to complete
2. Delete one `aml_trust_routing_attestation` row for the completed case via injected
   `AmlTrustAttestationRepository` (or direct `EntityManager` in `@Transactional` helper
   method); call `em.clear()` after delete to flush second-level cache
3. `GET /api/investigations/{caseId}/compliance-evidence` → assert `trustRouting.status = PARTIAL`;
   assert missing capability in decisions with `reconstructed = true`
4. Call evidence endpoint again → assert no duplicate reconstructed entry; `reconstructed=true`
   count for that capability is still 1

---

## 8. Out of Scope (tracked separately)

- **aml#56** — engine path not writing `COMPLIANCE_REVIEW_OPENED`; `sla.status` always `GAP`
  when running via engine path. `causedByEntryId` on `AmlSarOfficerReviewedLedgerEntry` will
  be null; audit chain stays `PARTIAL`.

---

## 9. Platform Coherence Review

- Ledger subclass rule (PP-20260513): JOINED inheritance ✅, V2009/V2010 numbering ✅, consumer-owned migrations ✅
- Observer failure pattern (PP-20260530-49856c): both observers apply double try/catch ✅
- Observer failure naming (PP-20260531-11724b): `actorRole = "<role>-observer-failed"` ✅
- Application-tier rule: all logic requires AML domain knowledge ✅
- Dual-trail audit pattern: `WorkItemLifecycleEvent` path maintains both operational and compliance record ✅
- Multi-JVM idempotency: partial unique index on `(investigation_case_id, capability_tag) WHERE reconstructed = TRUE` ✅
- No foundation primitives re-implemented ✅
