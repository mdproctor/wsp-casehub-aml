# Design: Trust Attestation Reconciliation + SAR Officer Reviewed Ledger Entry

**Issues:** casehubio/aml#44 (attestation reconciliation), casehubio/aml#55 (SAR officer reviewed), aml#56 (engine path COMPLIANCE_REVIEW_OPENED — closed in scope)
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
be recorded in the tamper-evident ledger under their identity.

### #56 — Engine path never writes COMPLIANCE_REVIEW_OPENED (closed in this spec)

`AmlInvestigationCoordinator` (Layer 3 sync path) calls `openReview()` then separately
calls `ledgerService.writeComplianceReviewOpened()`. The engine path (Quartz worker)
calls only `openReview()` — no ledger entry is written. Consequence: `sla.status = GAP`
for all engine-path investigations.

This is fixed here by moving `writeComplianceReviewOpened()` into `openReview()` itself,
making both WorkItem creation and ledger entry atomic in one call regardless of caller.
The `caseId` parameter added in §5.1 makes this wiring possible.

---

## 2. Design Goals

- Observer failures become visible in the ledger rather than disappearing silently
- Attestation gaps are filled lazily on evidence read using authoritative `WorkerDecisionEntry` data
- Status semantics distinguish: all original (`CLOSED`), observer-failed cap (`PARTIAL`), gap repaired (`PARTIAL`), no data (`GAP`)
- "Open compliance review" is a single atomic operation: WorkItem creation + ledger entry, regardless of caller path
- Officer approval or rejection is recorded with the officer's identity as `actorId = HUMAN`
- `AuditChainRequirement` covers the full case lifecycle including the officer decision
- GDPR erasure demonstrated against a real human actor
- Observer hardening applies PP-20260530-49856c to both new observers

---

## 3. Data Model Changes

### 3.1 `AmlTrustRoutingAttestation` — V2009

Two new columns and one partial unique index:

```sql
ALTER TABLE aml_trust_routing_attestation
    ADD COLUMN reconstructed   BOOLEAN NOT NULL DEFAULT FALSE,
    ADD COLUMN observer_failed BOOLEAN NOT NULL DEFAULT FALSE;

CREATE UNIQUE INDEX UQ_TRUST_ATTEST_CASE_CAP_RECONSTRUCTED
    ON aml_trust_routing_attestation (investigation_case_id, capability_tag)
    WHERE reconstructed = TRUE;
```

**`reconstructed`**: `true` for entries written by the reconciler to fill a gap. Required to
distinguish "trust score was null at routing time (Phase 0 — no trust data)" from "entry
was reconstructed later" — both have `trustScoreAtRouting = null`.

**`observerFailed`**: `true` for entries written by the observer's outer catch (the failure
audit record). Required to distinguish a live attestation from a failure-marker in status
computation. An observer-failure entry for capability X covers X (`PARTIAL`, not `GAP`) but
must not allow the requirement to reach `CLOSED` — the trust score was never captured live.

**Partial unique index** prevents multi-JVM double-writes: if two application instances both
detect a gap for the same (case, capability), the second `reconstructed=true` insert fails
with `org.hibernate.exception.ConstraintViolationException`. The reconciler catches this
and treats it as idempotent (the peer instance already wrote the entry).

### 3.2 `AmlSarOfficerReviewedLedgerEntry` — V2010

New `LedgerEntry` subclass. JOINED inheritance per PP-20260513-ledger-subclass.

```sql
CREATE TABLE aml_sar_officer_reviewed_ledger_entry (
    id              UUID        PRIMARY KEY REFERENCES ledger_entry(id),
    review_decision VARCHAR(20) NOT NULL   -- 'APPROVED' | 'REJECTED'
);
```

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
- `causedByEntryId` = `AmlComplianceReviewLedgerEntry.id` for this case (self-derived;
  non-null now that #56 is fixed — `COMPLIANCE_REVIEW_OPENED` is always written by `openReview()`)
- `sequenceNumber` = next in subject chain via `nextSequenceNumber(caseId)`

---

## 4. Issue #44: Observer Hardening + Lazy Reconciliation

### 4.1 Observer hardening (PP-20260530-49856c)

**Pre-try block** (pure computation, no DB):
```java
final double threshold = policyProvider.forCapability(event.capabilityTag()).threshold();
final Double score = trustScoreCache.getCapabilityScore(event.workerId(), event.capabilityTag())
        .stream().boxed().findFirst().orElse(null);
final UUID attestationSubject = attestationSubjectFor(event.caseId());
```

If `policyProvider.forCapability()` throws here, the method fails before the try block —
AUDIT GAP log path. `threshold` is always in scope when the outer catch fires.

**Double try/catch:**
```java
boolean attestationWritten = false;
try {
    // build entry, synchronized-saveWithSequence
    attestationWritten = true;
} catch (Exception e) {
    LOG.warnf(e, "AmlTrustRoutingObserver failed caseId=%s cap=%s workerId=%s",
              event.caseId(), event.capabilityTag(), event.workerId());
    if (!attestationWritten) {
        try {
            attestationRepo.saveObserverFailureEntry(event, attestationSubject, threshold);
        } catch (Exception inner) {
            LOG.errorf(inner,
                "AUDIT GAP: observer failure entry also failed caseId=%s cap=%s",
                event.caseId(), event.capabilityTag());
        }
    }
}
```

**`saveObserverFailureEntry(WorkerDecisionEvent, UUID, double threshold)`** — new
`@Transactional(REQUIRES_NEW)` method on `AmlTrustAttestationRepository`. Writes
`AmlTrustRoutingAttestation` with:
- `actorRole = "AmlInvestigationOrchestrator-observer-failed"` (PP-20260531-11724b)
- `actorId = "aml-orchestrator"`, `actorType = ActorType.SYSTEM`
- `trustScoreAtRouting = null`, `thresholdApplied = threshold` (actual policy threshold —
  not 0.0; the computed threshold is passed from the caller so the failure record is informative)
- `reconstructed = false`, `observerFailed = true`
- Sequence via `MAX(sequenceNumber) FROM LedgerEntry WHERE subjectId = :sid`

### 4.2 `AmlTrustRoutingAttestation` model update

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

1. Build `Set<String> coveredCaps` from `existing` (any `reconstructed`/`observerFailed` value)
2. For each `WorkerDecisionEntry` whose `capabilityTag` is NOT in `coveredCaps`:
   - Acquire per-subject lock (same `ConcurrentHashMap<UUID, Object>` pattern as the observer)
   - In a `synchronized` block: call `attestationRepo.saveWithSequence()` with:
     - `reconstructed = true`, `observerFailed = false`
     - `trustScoreAtRouting = decisionEntry.trustScoreAtRouting` (may be null — Phase 0)
     - `thresholdApplied = decisionEntry.thresholdApplied != null ? decisionEntry.thresholdApplied : 0.0`
     - `selectedWorkerId = decisionEntry.workerId`
     - `capabilityTag = decisionEntry.capabilityTag`
     - `actorRole = "AmlInvestigationOrchestrator"`, `actorId = "aml-orchestrator"`, `actorType = SYSTEM`
   - Catch block wraps the entire synchronized block (not just the inner call):
     ```java
     } catch (org.hibernate.exception.ConstraintViolationException e) {
         // Peer JVM wrote first — idempotent; peer's entry is in DB but not in this
         // request's memory. This request shows PARTIAL for this capability; the next
         // request will read the peer's reconstructed entry and show correctly.
         LOG.debugf("Peer JVM reconciled caseId=%s cap=%s — skipping duplicate", caseId, capTag);
         continue;
     }
     ```
   - Add new entry to result list
3. Return `existing + newly written`

**Source data note:** `WorkerDecisionEntry.trustScoreAtRouting` and `.thresholdApplied` (both
`Double`, nullable) are populated by the engine's `WorkerDecisionEventCapture` from
`TrustScoreCache` at event time — the same values `AmlTrustRoutingObserver` would have
captured. Reconstruction is faithful, not approximated. `thresholdApplied` is `double`
(primitive) on `AmlTrustRoutingAttestation`; null maps to `0.0` for Phase-0 cases.

**Why `AmlTrustRoutingAttestation` coexists with `WorkerDecisionEntry`:**
`WorkerDecisionEntry` is the engine's operational record. `AmlTrustRoutingAttestation` is
AML's explicit compliance commitment in its own Merkle chain under a namespaced subject
(preventing sequence collision with case lifecycle entries). They answer different questions:
the engine records what happened; AML records its compliance accountability.

### 4.4 `RoutingDecisionRecord` — updated record declaration

`RoutingDecisionRecord` is in `api/` (`io.casehub.aml.compliance`). Updated declaration:

```java
public record RoutingDecisionRecord(
    String capabilityTag,
    String selectedWorker,
    Double trustScoreAtRouting,
    double thresholdApplied,
    UUID attestationEntryId,
    boolean reconstructed,      // NEW
    boolean observerFailed      // NEW
) {}
```

All existing construction calls in `buildTrustRouting()` break at compile time and must be
updated to the 7-arg form:

```java
new RoutingDecisionRecord(
    a.capabilityTag, a.selectedWorkerId,
    a.trustScoreAtRouting, a.thresholdApplied, a.id,
    a.reconstructed, a.observerFailed)
```

### 4.5 `AmlComplianceEvidenceService` — constructor, `build()`, and `buildTrustRouting()`

**Constructor** gains `AmlAttestationReconciler reconciler` as sixth parameter:

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

**`buildTrustRouting()`:**
```java
List<WorkerDecisionEntry> dispatched = workerDecisionRepo.findAllByCaseId(caseId);
List<AmlTrustRoutingAttestation> raw = attestationRepo.findByInvestigationCaseId(caseId);
List<AmlTrustRoutingAttestation> merged = reconciler.reconcileIfNeeded(caseId, dispatched, raw);
// status + RoutingDecisionRecord list built from merged
```

**Status logic:**

| Condition | Status |
|---|---|
| `dispatched.isEmpty()` | `GAP` |
| All dispatched caps: `observerFailed=false` AND `reconstructed=false` | `CLOSED` |
| Any dispatched cap: only `observerFailed=true` coverage | `PARTIAL` |
| Any dispatched cap: only `reconstructed=true` coverage | `PARTIAL` |
| Any cap uncovered after reconcile (reconcile write also failed) | `PARTIAL` |

**Constraint-catch one-request window:** when the reconciler skips due to a peer-JVM
duplicate, the peer's entry is in the DB but absent from this request's `merged` list.
The status correctly shows `PARTIAL` for this request; the next request reads the peer's
entry from the DB and resolves correctly. This is acceptable eventual consistency for a
compliance inspection surface.

**Why lazy on-read rather than a background job or repair endpoint:** the compliance
evidence endpoint is the natural inspection point — an examiner reading it should see a
complete picture. Lazy repair ensures the response is self-consistent within a request.
A background job creates a gap window; an explicit repair endpoint adds operational
ceremony for a system with no human operators. The cost is a GET with occasional write
side-effects. This is acceptable and is documented here as the explicit design choice.

---

## 5. Issue #55 + #56: ComplianceReviewLifecycle consolidation + WorkItem Observer + Ledger Entry

### 5.1 `ComplianceReviewLifecycle` — consolidate WorkItem + ledger entry

**Signature change:**
```java
// Before
public String openReview(SuspiciousTransaction transaction, InvestigationSummary summary)

// After
public String openReview(SuspiciousTransaction transaction, InvestigationSummary summary, UUID caseId)
```

`callerRef` changes from `"aml:investigation/" + transaction.id()` to `"aml:investigation:" + caseId`.
Two changes: separator (`/` → `:`) and identity (`transaction.id` → `caseId`).

**Consolidation (closes #56):** `openReview()` now also writes the `COMPLIANCE_REVIEW_OPENED`
ledger entry, making "open compliance review" atomic regardless of caller:

```java
final WorkItem workItem = creator.apply(...);
final String taskId = workItem.id.toString();
ledgerService.writeComplianceReviewOpened(caseId, taskId);  // NEW — closes #56
return taskId;
```

**Requires injecting `AmlLedgerService`:**

```java
private final Function<WorkItemCreateRequest, WorkItem> creator;
private final AmlLedgerService ledgerService;

@Inject
public ComplianceReviewLifecycle(WorkItemService workItemService, AmlLedgerService ledgerService) {
    this.creator = workItemService::create;
    this.ledgerService = ledgerService;
}

// Unit test constructor — both dependencies stubbed
ComplianceReviewLifecycle(Function<WorkItemCreateRequest, WorkItem> creator,
                          AmlLedgerService ledgerService) {
    this.creator = creator;
    this.ledgerService = ledgerService;
}
```

`ComplianceReviewLifecycleTest` must be updated to use the new 2-arg test constructor,
passing `AmlLedgerService.noOp()` (the existing no-op stub already exists for this purpose).

**`AmlInvestigationCoordinator` change:** remove the separate
`ledgerService.writeComplianceReviewOpened(caseId, taskId)` call — it is now redundant.
`AmlInvestigationCoordinator` still injects `AmlLedgerService` for `writeCaseOpened()`.

**In-flight WorkItems** created before deployment carry the old `"aml:investigation/"`
format. The observer's guard filters on `startsWith("aml:investigation:")` — these will
be silently ignored when the officer completes them. This is a hard cutover with no
back-fill, acceptable for a tutorial system with no production data.

### 5.2 `AmlEngineCoordinator` — inject `caseId` into blackboard initial context

`startInvestigation()` currently puts only `"transaction"` and `"priorEntityContext"` into
the initial context map. Add:

```java
initialContext.put("caseId", caseId.toString());
```

Sar-drafting workers extract it:
```java
final UUID caseId = UUID.fromString((String) input.get("caseId"));
```

Both workers call:
```java
complianceReviewLifecycle.openReview(tx, buildSummary(input, tx, sarNarrative), caseId)
```

### 5.3 `AmlWorkItemLifecycleObserver` (new `@ApplicationScoped`)

Observes `WorkItemLifecycleEvent`. Handles both `COMPLETED` and `REJECTED` decisions.

```
@ObservesAsync WorkItemLifecycleEvent event:
  1. Guard: event.status() not in {COMPLETED, REJECTED} → return
  2. Guard: event.workItem() == null → LOG.warn("no WorkItem snapshot"), return
  3. Guard: callerRef == null || !callerRef.startsWith("aml:investigation:") → return
  4. Parse caseId = UUID.fromString(callerRef.substring(prefix.length()))
     On IllegalArgumentException → LOG.warn, return
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
                   LOG.errorf(inner, "AUDIT GAP: SAR_OFFICER_REVIEWED failure entry also failed caseId=%s", caseId);
               }
           }
       }
```

`@ObservesAsync` — decouples the ledger write from the casehub-work transaction.
`WorkItemService.complete()` and `.reject()` fire both `lifecycleEvent.fire()` (sync, for
`@Observes` listeners) and `lifecycleEvent.fireAsync()` (async, for `@ObservesAsync`
listeners). This observer receives the async delivery.

**REJECTED handling:** a rejected SAR is a compliance decision. Not recording it leaves the
audit chain in `PARTIAL` permanently after the officer acts. `reviewDecision = "REJECTED"`
closes the chain for the examiner regardless of outcome.

### 5.4 `AmlLedgerService` additions

**`writeSarOfficerReviewed(UUID caseId, String officerId, String reviewDecision)`**

```java
@Transactional(TxType.REQUIRED)
// Starts a new transaction — no transaction is propagated from @ObservesAsync context.
```

Writes `AmlSarOfficerReviewedLedgerEntry`:
- `actorId = officerId`, `actorType = HUMAN`, `actorRole = "ComplianceOfficer"`
- `subjectId = caseId`, `sequenceNumber = nextSequenceNumber(caseId)`
- `causedByEntryId` — self-derived: first `AmlComplianceReviewLedgerEntry.id` for this case.
  Non-null now that #56 is fixed (`writeComplianceReviewOpened()` is always called by `openReview()`).

**`writeSarOfficerReviewedFailure(UUID caseId, String officerId)`**

```java
@Transactional(TxType.REQUIRES_NEW)
// Isolated transaction — failure record commits independently of any outer context.
// Per PP-20260530-49856c: failure entry writer must use REQUIRES_NEW.
```

Per PP-20260531-11724b: `actorRole = "ComplianceOfficer-observer-failed"`,
`actorId = "aml-orchestrator"`, `actorType = SYSTEM`, `reviewDecision = "UNKNOWN"`.

### 5.5 `AmlComplianceEvidenceService` — SAR_OFFICER_REVIEWED in build pipeline

Add `filterSarOfficerReviewed()` helper. Propagate officer review entries through the full
call chain.

**Updated `findEvidence()` guard:**
```java
List<AmlSarOfficerReviewedLedgerEntry> officerReviewEntries = filterSarOfficerReviewed(all);
if (caseEntries.isEmpty() && reviewEntries.isEmpty() && officerReviewEntries.isEmpty())
    return Optional.empty();
// If officer entries exist without preceding entries (unlikely), evidence is returned:
// audit chain = GAP, other requirements computed normally.
```

**Updated `build()` signature:**
```java
private ComplianceEvidence build(UUID caseId,
        List<AmlCaseOpenedLedgerEntry> caseEntries,
        List<AmlComplianceReviewLedgerEntry> reviewEntries,
        List<AmlSarOfficerReviewedLedgerEntry> officerReviewEntries)
```

**Updated `buildAuditChain()` signature:**
```java
private AuditChainRequirement buildAuditChain(UUID caseId,
        List<AmlCaseOpenedLedgerEntry> caseEntries,
        List<AmlComplianceReviewLedgerEntry> reviewEntries,
        List<AmlSarOfficerReviewedLedgerEntry> officerReviewEntries)
```

Officer review entries added as `"SAR_OFFICER_REVIEWED"` events in the `LedgerEventRecord`
list using `entry.actorId` (the officer) and `entry.actorRole`.

**Extended `allLinked` check:**
```java
boolean allLinked = reviewEntries.stream().allMatch(e -> e.causedByEntryId != null)
        && officerReviewEntries.stream().allMatch(e -> e.causedByEntryId != null);
```

**Updated `AuditChainRequirement.status`:**

| Condition | Status |
|---|---|
| No AML ledger entries | `GAP` |
| Chain verified + `allLinked` + ≥ 1 officer review entry | `CLOSED` |
| Case opened + review opened, no officer review yet | `PARTIAL` |
| Chain not fully verified | `PARTIAL` |
| Any `causedByEntryId` null (review or officer entry) | `PARTIAL` |

---

## 6. Flyway Migrations

| Version | Path | Datasource | What |
|---|---|---|---|
| V2009 | `db/aml-trust-routing/migration/` | qhorus | Add `reconstructed`, `observer_failed`; partial unique index |
| V2010 | `db/aml-ledger/migration/` | qhorus | Create `aml_sar_officer_reviewed_ledger_entry` join table |

---

## 7. Testing

### Unit tests (pure Java, no Quarkus)

**`AmlAttestationReconcilerTest`**
- 3 dispatched, 2 original attestations → reconciler writes 1 `reconstructed=true`; result size 3
- All 3 covered → no write; returns original list unchanged
- Idempotency: second call reads the reconstructed entry as covered; no duplicate write
- One `observerFailed=true` entry for cap X: X is in coveredCaps; reconciler skips; X stays PARTIAL in status (not reconstructed, not re-reconciled)

**`AmlWorkItemLifecycleObserverTest`**
- `COMPLETED` + valid `aml:investigation:` callerRef → `writeSarOfficerReviewed("APPROVED")`
- `REJECTED` + valid callerRef → `writeSarOfficerReviewed("REJECTED")`
- `IN_PROGRESS` → no write
- Null `workItem()` → no write, warning logged
- callerRef for different domain → no write
- Old-format `aml:investigation/` callerRef → no write (hard cutover)
- Invalid UUID in callerRef → no write, warning logged
- Null actor → falls back to `"unknown-officer"`

**`AmlComplianceEvidenceServiceTest`**

Constructor: now 6 args. All 5 existing tests add `@Mock AmlAttestationReconciler mockReconciler`
and `when(mockReconciler.reconcileIfNeeded(any(), any(), any())).thenReturn(raw)` in setUp.

Existing `attestation()` helper creates entries without setting `reconstructed`/`observerFailed`
— `false` defaults are correct for non-reconstructed originals. No change needed to helper.

New assertions:
- All original (`observerFailed=false, reconstructed=false`) → `trustRouting.status = CLOSED`
- One `observerFailed=true` → `trustRouting.status = PARTIAL`; `observerFailed=true` in record
- One `reconstructed=true` → `trustRouting.status = PARTIAL`; `reconstructed=true` in record
- SAR_OFFICER_REVIEWED entry present + all linked → `auditChain.status = CLOSED`
- SAR_OFFICER_REVIEWED missing → `auditChain.status = PARTIAL`

**`ComplianceReviewLifecycleTest`**

Updated to use 2-arg test constructor, passing `AmlLedgerService.noOp()`:
```java
new ComplianceReviewLifecycle(mockCreator, AmlLedgerService.noOp())
```

Add assertion: `writeComplianceReviewOpened()` is called with the correct caseId.

### `@QuarkusTest` — full GDPR integration test

New test in `AmlLayer7ResourceTest`:

1. `POST /api/layer6/investigations` with PEP transaction → `caseId`
2. Poll `GET /api/layer6/investigations/{caseId}` until `status=completed`
3. `GET /api/investigations/{caseId}/compliance-evidence` → assert `sla.taskId` non-null;
   extract `taskId` (now populated because #56 is fixed — `openReview()` writes the ledger entry)
4. `workItemService.claim(taskId, "compliance-officer-001")` → ASSIGNED
5. `workItemService.start(taskId, "compliance-officer-001")` → IN_PROGRESS
6. `workItemService.complete(taskId, "compliance-officer-001", "SAR approved", "APPROVED")`
   // 4-param: id, actorId, resolution, outcome — fires both sync and async WorkItemLifecycleEvent
7. Await `@ObservesAsync` delivery: poll `GET /api/investigations/{caseId}/compliance-evidence`
   until `auditChain.events` contains `SAR_OFFICER_REVIEWED`, timeout ≤ 5s (Awaitility)
8. Assert event has `actorId = "compliance-officer-001"`, `actorRole = "ComplianceOfficer"`;
   assert `auditChain.status = CLOSED`
9. `POST /api/actors/compliance-officer-001/erasure` → 200, `erasedCount >= 1`
10. `GET /api/investigations/{caseId}/compliance-evidence` → assert SAR_OFFICER_REVIEWED
    `actorId` is pseudonymized (not `"compliance-officer-001"`)

### `@QuarkusTest` — reconciliation path

1. Start investigation, poll to complete
2. In a `@Transactional` test helper: delete one `aml_trust_routing_attestation` row for
   the completed case via injected `AmlTrustAttestationRepository`; call `em.clear()` after
   delete to flush second-level cache
3. `GET /api/investigations/{caseId}/compliance-evidence` → assert `trustRouting.status = PARTIAL`;
   deleted capability in decisions with `reconstructed = true`
4. Call evidence endpoint again → assert no duplicate reconstructed entry; count is still 1

---

## 8. Out of Scope (tracked separately)

No remaining out-of-scope items. aml#56 is closed in this spec.

---

## 9. Platform Coherence Review

- Ledger subclass rule (PP-20260513): JOINED inheritance ✅, V2009/V2010 numbering ✅, consumer-owned migrations ✅
- Observer failure pattern (PP-20260530-49856c): double try/catch applied to both new observers; `writeSarOfficerReviewedFailure` uses `REQUIRES_NEW` ✅
- Observer failure naming (PP-20260531-11724b): `actorRole = "<role>-observer-failed"` ✅
- Application-tier rule: all logic requires AML domain knowledge ✅
- Dual-trail audit pattern: `WorkItemLifecycleEvent` path maintains both operational and compliance record ✅
- Multi-JVM idempotency: partial unique index `WHERE reconstructed = TRUE` ✅
- `ConstraintViolationException` catch: `org.hibernate.exception.ConstraintViolationException`; surrounds entire synchronized block ✅
- `RoutingDecisionRecord` updated record declaration with construction call shown ✅
- `openReview()` consolidation: WorkItem + ledger entry atomic; both engine and sync paths now write COMPLIANCE_REVIEW_OPENED ✅
- No foundation primitives re-implemented ✅
