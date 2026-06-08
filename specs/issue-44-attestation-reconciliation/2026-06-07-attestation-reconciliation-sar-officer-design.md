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

### #55 — Hollow GDPR erasure

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

This is fixed by moving `writeComplianceReviewOpened()` into `openReview()` itself.
The `caseId` parameter added in §5.1 makes this wiring possible.

---

## 2. Design Goals

- Observer failures become visible in the ledger rather than disappearing silently
- Attestation gaps are filled lazily on evidence read using authoritative `WorkerDecisionEntry` data
- Status semantics distinguish: all original (`CLOSED`), observer-failed cap (`PARTIAL`), gap repaired (`PARTIAL`), no data (`GAP`)
- "Open compliance review" is consolidated into one method call: WorkItem creation + ledger entry always written together, regardless of caller path (consolidated — not transactionally atomic; see §5.1)
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

**`reconstructed`**: `true` for entries written by the reconciler to fill a gap. Distinguishes
"trust score was null at routing time (Phase 0)" from "entry reconstructed later" — both
have `trustScoreAtRouting = null`.

**`observerFailed`**: `true` for entries written by the observer's outer catch. Distinguishes
a live attestation from a failure-marker. An observer-failure entry covers the capability
(`PARTIAL`, not `GAP`) but prevents `CLOSED`.

**Partial unique index** prevents multi-JVM double-writes of reconstructed entries. When
two JVMs both detect the same gap, the second insert fails. The reconciler catches this as:

```java
} catch (jakarta.persistence.PersistenceException e) {
    if (e.getCause() instanceof org.hibernate.exception.ConstraintViolationException) {
        // Peer JVM reconciled first — idempotent
        LOG.debugf("Peer JVM reconciled caseId=%s cap=%s", caseId, capTag);
        continue;
    }
    throw e;  // unexpected failure — re-raise
}
```

**Why `PersistenceException`, not `ConstraintViolationException` directly:** when
`saveWithSequence()` (`@Transactional(REQUIRES_NEW)`) flushes and hits the unique index,
Hibernate wraps the JDBC `BatchUpdateException` in
`org.hibernate.exception.ConstraintViolationException`, then the JPA session factory
wraps that in `jakarta.persistence.PersistenceException`. The CDI `@Transactional`
interceptor rolls back REQUIRES_NEW and rethrows the `PersistenceException` — that is
the actual exception at the call boundary. Catching
`org.hibernate.exception.ConstraintViolationException` directly never fires. The
`throw e` rethrow is essential: swallowing all `PersistenceException` would silently
discard connection failures and mapping errors.

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
- `actorType` = `ActorType.HUMAN`, `actorRole` = `"ComplianceOfficer"`
- `subjectId` = `caseId`
- `causedByEntryId` = `AmlComplianceReviewLedgerEntry.id` (self-derived; non-null now that
  #56 is fixed — `writeComplianceReviewOpened()` is always called by `openReview()`)
- `sequenceNumber` = next in subject chain

---

## 4. Issue #44: Observer Hardening + Lazy Reconciliation

### 4.1 Observer hardening (PP-20260530-49856c)

**Pre-try block** (pure computation, no DB — consistent with current observer code structure):
```java
final double threshold = policyProvider.forCapability(event.capabilityTag()).threshold();
final Double score = trustScoreCache.getCapabilityScore(event.workerId(), event.capabilityTag())
        .stream().boxed().findFirst().orElse(null);
final UUID attestationSubject = attestationSubjectFor(event.caseId());
```

If `policyProvider.forCapability()` throws before the try block, the method fails without
writing a failure entry — AUDIT GAP log path. `threshold` is always in scope when the
outer catch fires.

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
- `trustScoreAtRouting = null`, `thresholdApplied = threshold` (actual policy threshold,
  not 0.0 — informative for the examiner)
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
   - Wrap the entire synchronized block in the catch described in §3.1:
     ```java
     try {
         synchronized (lock) {
             attestationRepo.saveWithSequence(reconstructedEntry);
         }
         result.add(reconstructedEntry);
     } catch (jakarta.persistence.PersistenceException e) {
         if (e.getCause() instanceof org.hibernate.exception.ConstraintViolationException) {
             LOG.debugf("Peer JVM reconciled caseId=%s cap=%s — skipping", caseId, capTag);
             // Peer's entry is in DB but absent from this request's merged list.
             // Status correctly shows PARTIAL for this request; next request reads it.
             continue;
         }
         throw e;
     }
     ```
   - Entry fields: `reconstructed=true`, `observerFailed=false`,
     `trustScoreAtRouting=decisionEntry.trustScoreAtRouting`,
     `thresholdApplied = decisionEntry.thresholdApplied != null ? decisionEntry.thresholdApplied : 0.0`,
     `selectedWorkerId=decisionEntry.workerId`, `capabilityTag=decisionEntry.capabilityTag`,
     `actorRole="AmlInvestigationOrchestrator"`, `actorId="aml-orchestrator"`, `actorType=SYSTEM`
3. Return `existing + newly written`

**Source data note:** `WorkerDecisionEntry.trustScoreAtRouting` and `.thresholdApplied` (both
`Double`, nullable) are populated by `WorkerDecisionEventCapture`. Reconstruction copies
the authoritative engine-captured values, not approximations. `thresholdApplied` is
`double` (primitive) on `AmlTrustRoutingAttestation`; null maps to `0.0` for Phase-0 cases.

**Why `AmlTrustRoutingAttestation` coexists with `WorkerDecisionEntry`:**
`WorkerDecisionEntry` is the engine's operational record. `AmlTrustRoutingAttestation` is
AML's explicit compliance commitment in its own Merkle chain under a namespaced subject,
preventing sequence collision with case lifecycle entries.

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

Updated construction call in `buildTrustRouting()` (breaks all existing call sites — all
must be updated to 7-arg):

```java
new RoutingDecisionRecord(
    a.capabilityTag, a.selectedWorkerId,
    a.trustScoreAtRouting, a.thresholdApplied, a.id,
    a.reconstructed, a.observerFailed)
```

### 4.5 `AmlComplianceEvidenceService` — constructor, `build()`, `buildTrustRouting()`

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
```

**Status logic:**

| Condition | Status |
|---|---|
| `dispatched.isEmpty()` | `GAP` |
| All dispatched: `observerFailed=false, reconstructed=false` | `CLOSED` |
| Any dispatched: only `observerFailed=true` coverage | `PARTIAL` |
| Any dispatched: only `reconstructed=true` coverage | `PARTIAL` |
| Any cap uncovered after reconcile (reconcile write also failed) | `PARTIAL` |

**Why lazy on-read:** the compliance evidence endpoint is the natural inspection point.
Lazy repair ensures the response is self-consistent. A background job creates a visibility
gap; an explicit repair endpoint adds operational ceremony for a system with no human
operators. The cost is a GET with occasional write side-effects — explicit and accepted.

**Peer-JVM one-request window:** when the reconciler skips due to a duplicate unique index
violation, the peer's entry exists in the DB but is absent from this request's merged list.
Status correctly shows `PARTIAL`; the next request reads the peer's entry correctly.

---

## 5. Issue #55 + #56: ComplianceReviewLifecycle consolidation + WorkItem Observer + Ledger Entry

### 5.1 `ComplianceReviewLifecycle` — consolidate WorkItem + ledger entry

**Why the consolidation is not `@Transactional`:** `WorkItemService.create()` writes to the
default datasource; `AmlLedgerService.writeComplianceReviewOpened()` writes to the qhorus
datasource via `@PersistenceContext(unitName = "qhorus")`. These are two different non-XA
datasources. Narayana JTA's Last Resource Commit Optimization (LRC) allows exactly one
non-XA resource per XA transaction; two non-XA resources in one JTA transaction require
full XA configuration — not warranted for this tutorial system. `openReview()` is therefore
**not** annotated `@Transactional`. The two operations run in separate JTA transactions:
`workItemService.create()` (REQUIRED) on the default datasource, then
`ledgerService.writeComplianceReviewOpened()` (REQUIRED) on qhorus. Partial-failure risk:
if the ledger write fails after the WorkItem commits, the WorkItem exists without a ledger
record. This is accepted — the observer-hardening pattern in the surrounding code already
treats ledger-write failure as an AUDIT GAP condition.

**Signature change:**
```java
// Before
public String openReview(SuspiciousTransaction transaction, InvestigationSummary summary)

// After
public String openReview(SuspiciousTransaction transaction, InvestigationSummary summary, UUID caseId)
```

`callerRef` changes from `"aml:investigation/" + transaction.id()` to `"aml:investigation:" + caseId`.

**Consolidation (closes #56):**

```java
final WorkItem workItem = creator.apply(...);
final String taskId = workItem.id.toString();
ledgerService.writeComplianceReviewOpened(caseId, taskId);  // always called now
return taskId;
```

**Injects `AmlLedgerService`:**

```java
private final Function<WorkItemCreateRequest, WorkItem> creator;
private final AmlLedgerService ledgerService;

@Inject
public ComplianceReviewLifecycle(WorkItemService workItemService, AmlLedgerService ledgerService) {
    this.creator = workItemService::create;
    this.ledgerService = ledgerService;
}

// Unit test constructor
ComplianceReviewLifecycle(Function<WorkItemCreateRequest, WorkItem> creator,
                          AmlLedgerService ledgerService) {
    this.creator = creator;
    this.ledgerService = ledgerService;
}
```

**`AmlInvestigationCoordinator` changes — two edits:**

1. Call site: `compliance.openReview(transaction, summary)` → `compliance.openReview(transaction, summary, caseId)` (caseId already in scope at that point in `investigate()`)
2. Remove: `ledgerService.writeComplianceReviewOpened(caseId, taskId)` — now redundant (called by `openReview()` internally)

`AmlInvestigationCoordinator` still retains its `AmlLedgerService` injection for `writeCaseOpened()`.

**Broken call sites and tests — complete list:**

| Location | What breaks | Fix |
|---|---|---|
| `AmlInvestigationCoordinator.investigate()` | `openReview(transaction, summary)` → 3-param | Pass `caseId` |
| `AmlInvestigationCoordinator.investigate()` | `ledgerService.writeComplianceReviewOpened(caseId, taskId)` → redundant | Remove |
| `AmlInvestigationCoordinatorTest` line ~52 | `new ComplianceReviewLifecycle(req -> workItemWith(expectedId))` → 1-arg | Add `AmlLedgerService.noOp()` |
| `AmlInvestigationCoordinatorTest` line ~82 | `new ComplianceReviewLifecycle(req -> workItemWith(UUID.randomUUID()))` → 1-arg | Add `AmlLedgerService.noOp()` |
| `ComplianceReviewLifecycleTest` (both tests) | `new ComplianceReviewLifecycle(req -> ...)` → 1-arg constructor | Pass `@Mock AmlLedgerService` (see §7) |
| `ComplianceReviewLifecycleTest` (both tests) | `lifecycle.openReview(tx, summary)` → 2-param | Add `UUID.randomUUID()` as 3rd arg |
| `ComplianceReviewLifecycleTest` test 1 | `assertTrue(req.callerRef.contains("TXN-CLR"))` — callerRef now uses UUID, not tx.id() | Replace with: `assertTrue(req.callerRef.startsWith("aml:investigation:"))` and `assertDoesNotThrow(() -> UUID.fromString(req.callerRef.substring("aml:investigation:".length())))` |

**In-flight WorkItems** created before deployment carry `"aml:investigation/"` format.
The observer's guard filters on `startsWith("aml:investigation:")` — these are silently
ignored when the officer completes them. Hard cutover, no back-fill. Acceptable for a
tutorial system with no production data.

### 5.2 `AmlEngineCoordinator` — signal `caseId` into blackboard after case start

`startInvestigation()` builds `initialContext` before calling `startCase()` — `caseId` is
the **return value** of `startCase()` and does not exist before the call returns. It cannot
be put into `initialContext` before the call.

`CaseHub.signal(UUID caseId, String path, Object value)` (confirmed from bytecode: returns
`void`, delegates to `runtime.signal()`, synchronous blackboard update) is called
immediately after `startCase()` unblocks:

```java
caseId = caseHub.startCase(initialContext)
        .toCompletableFuture()
        .get(CASE_START_TIMEOUT_SECONDS, TimeUnit.SECONDS);

// Signal caseId into the blackboard immediately — workers read it from input context.
// Safe: sar-drafting is the last worker to execute (entity-resolution → pattern/OSINT
// parallel → sar-drafting). Quartz job scheduling gives ample time for the in-memory
// blackboard update to propagate before sar-drafting reads input.get("caseId").
caseHub.signal(caseId, "caseId", caseId.toString());

ledgerService.writeCaseOpened(transaction, caseId);
```

Sar-drafting workers extract it with a null guard (fail loudly rather than NPE if signal
somehow arrives late):

```java
final String rawCaseId = (String) input.get("caseId");
if (rawCaseId == null) {
    LOG.errorf("caseId not in context for sar-drafting worker — signal may not have arrived");
    throw new RuntimeException("caseId missing from case context");
}
final UUID caseId = UUID.fromString(rawCaseId);
// then:
complianceReviewLifecycle.openReview(tx, buildSummary(input, tx, sarNarrative), caseId)
```

### 5.3 `AmlWorkItemLifecycleObserver` (new `@ApplicationScoped`)

Observes `WorkItemLifecycleEvent`. Handles `COMPLETED` and `REJECTED`.

```
@ObservesAsync WorkItemLifecycleEvent event:
  1. Guard: event.status() not in {COMPLETED, REJECTED} → return
  2. Guard: event.workItem() == null → LOG.warn, return
  3. Guard: callerRef == null || !callerRef.startsWith("aml:investigation:") → return
  4. Parse caseId from callerRef suffix; on IllegalArgumentException → LOG.warn, return
  5. officerId = event.actor() != null ? event.actor() : "unknown-officer"
  6. reviewDecision = event.status() == COMPLETED ? "APPROVED" : "REJECTED"
  7. Apply PP-20260530-49856c double try/catch
```

`WorkItemService.complete()` and `.reject()` each fire both `lifecycleEvent.fire()` (sync)
and `lifecycleEvent.fireAsync()` (async). This observer receives the async delivery.

### 5.4 `AmlLedgerService` additions

**`writeSarOfficerReviewed(UUID caseId, String officerId, String reviewDecision)`**

```java
@Transactional(TxType.REQUIRED)
// Starts a new transaction — no transaction is propagated from @ObservesAsync context.
```

Self-derives `causedByEntryId` from first `AmlComplianceReviewLedgerEntry` for the case.

**`writeSarOfficerReviewedFailure(UUID caseId, String officerId)`**

```java
@Transactional(TxType.REQUIRES_NEW)
// Isolated transaction — failure record commits independently of any outer context.
// Per PP-20260530-49856c: failure entry writer must use REQUIRES_NEW.
```

`actorRole = "ComplianceOfficer-observer-failed"`, `actorId = "aml-orchestrator"`,
`actorType = SYSTEM`, `reviewDecision = "UNKNOWN"`.

### 5.5 `AmlComplianceEvidenceService` — SAR_OFFICER_REVIEWED in build pipeline

**Updated `findEvidence()` empty guard:**
```java
List<AmlSarOfficerReviewedLedgerEntry> officerReviewEntries = filterSarOfficerReviewed(all);
if (caseEntries.isEmpty() && reviewEntries.isEmpty() && officerReviewEntries.isEmpty())
    return Optional.empty();
// officer-only entries (no preceding case/review entries): return evidence, auditChain=GAP
```

**Updated method signatures:**
```java
private ComplianceEvidence build(UUID caseId,
        List<AmlCaseOpenedLedgerEntry> caseEntries,
        List<AmlComplianceReviewLedgerEntry> reviewEntries,
        List<AmlSarOfficerReviewedLedgerEntry> officerReviewEntries)

private AuditChainRequirement buildAuditChain(UUID caseId,
        List<AmlCaseOpenedLedgerEntry> caseEntries,
        List<AmlComplianceReviewLedgerEntry> reviewEntries,
        List<AmlSarOfficerReviewedLedgerEntry> officerReviewEntries)
```

**`assembleEvidence()` must also be updated** (package-private, used by all 5 existing unit
tests via `service.assembleEvidence(caseId)`). It calls `build()` internally and will fail
to compile without the 4th argument:

```java
// Before
return build(caseId, filterCaseOpened(all), filterComplianceReview(all));

// After
return build(caseId,
    filterCaseOpened(all), filterComplianceReview(all),
    filterSarOfficerReviewed(all));  // ADD
```

The 5 existing unit tests call `service.assembleEvidence(caseId)` — their call sites are
unchanged (method signature stays `assembleEvidence(UUID caseId)`), but they require the
`mockReconciler` stub in setUp (already covered in §7 `AmlComplianceEvidenceServiceTest`).

Officer review entries appear as `"SAR_OFFICER_REVIEWED"` events in `LedgerEventRecord` list.

**Extended `allLinked` check:**
```java
boolean allLinked = reviewEntries.stream().allMatch(e -> e.causedByEntryId != null)
        && officerReviewEntries.stream().allMatch(e -> e.causedByEntryId != null);
```

**`AuditChainRequirement.status`:**

| Condition | Status |
|---|---|
| No AML ledger entries | `GAP` |
| Chain verified + `allLinked` + ≥ 1 officer review entry | `CLOSED` |
| Case opened + review opened, no officer review yet | `PARTIAL` |
| Chain not fully verified | `PARTIAL` |
| Any `causedByEntryId` null | `PARTIAL` |

Note: `chainVerified` requires `verificationService.verify(caseId)` to return `true`.
With `casehub.ledger.hash-chain.enabled=false` (H2 test config), `verify()` throws
`IllegalStateException` → caught → `chainVerified = false`. **CLOSED is unreachable in
H2 tests.** The GDPR integration test must not assert `CLOSED`.

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
- 3 dispatched, 2 originals → writes 1 `reconstructed=true`; result size 3
- All covered → no write; original list returned
- Idempotency: second call sees reconstructed entry in coveredCaps; no duplicate
- `observerFailed=true` for cap X → X covered; reconciler skips X (stays PARTIAL, not re-reconciled)

**`AmlWorkItemLifecycleObserverTest`**
- `COMPLETED` + valid callerRef → `writeSarOfficerReviewed("APPROVED")`
- `REJECTED` + valid callerRef → `writeSarOfficerReviewed("REJECTED")`
- `IN_PROGRESS` → no write
- Null `workItem()` → no write, warning
- Old-format `aml:investigation/` callerRef → no write (hard cutover)
- Invalid UUID in callerRef → no write, warning
- Null actor → fallback `"unknown-officer"`

**`AmlComplianceEvidenceServiceTest`**

Constructor: 6 args. All 5 existing tests add `@Mock AmlAttestationReconciler mockReconciler`
and `when(mockReconciler.reconcileIfNeeded(any(), any(), any())).thenReturn(raw)` in setUp.

Existing `attestation()` helper: `false` defaults on both new fields — correct for originals.

New assertions for updated status logic and SAR_OFFICER_REVIEWED in audit chain.

**`ComplianceReviewLifecycleTest`**

Both tests updated to 2-arg test constructor with `@Mock AmlLedgerService mockLedger`
(not `noOp()` — noOp discards the call and cannot be verified):

```java
@Mock AmlLedgerService mockLedger;

// Test 1
ComplianceReviewLifecycle lifecycle = new ComplianceReviewLifecycle(captureCreator, mockLedger);
UUID caseId = UUID.randomUUID();
lifecycle.openReview(tx, summary, caseId);
verify(mockLedger).writeComplianceReviewOpened(eq(caseId), any());

// Test 1 callerRef assertion (replaces assertTrue(req.callerRef.contains("TXN-CLR")))
assertTrue(req.callerRef.startsWith("aml:investigation:"));
assertDoesNotThrow(() -> UUID.fromString(req.callerRef.substring("aml:investigation:".length())));

// Test 2
ComplianceReviewLifecycle lifecycle = new ComplianceReviewLifecycle(
        req -> workItemWith(expectedId), mockLedger);
String taskId = lifecycle.openReview(tx, summary, UUID.randomUUID());
```

**`AmlInvestigationCoordinatorTest`**

Both `ComplianceReviewLifecycle` constructions → 2-arg with `AmlLedgerService.noOp()`
(coordinator test verifies coordinator behavior, not ledger call — noOp is correct here):

```java
// was: new ComplianceReviewLifecycle(req -> workItemWith(expectedId))
new ComplianceReviewLifecycle(req -> workItemWith(expectedId), AmlLedgerService.noOp())

// was: new ComplianceReviewLifecycle(req -> workItemWith(UUID.randomUUID()))
new ComplianceReviewLifecycle(req -> workItemWith(UUID.randomUUID()), AmlLedgerService.noOp())
```

### `@QuarkusTest` — full GDPR integration test

New test in `AmlLayer7ResourceTest`:

1. `POST /api/layer6/investigations` with PEP transaction → `caseId`
2. Poll `GET /api/layer6/investigations/{caseId}` until `status=completed`
3. `GET /api/investigations/{caseId}/compliance-evidence` → assert `sla.workItemId` non-null;
   extract `taskId = sla.workItemId` (#56 fixed — `openReview()` writes the ledger entry on all paths)
   // JSON field is "workItemId" — SlaRequirement.workItemId serialized by Jackson as-is
4. `workItemService.claim(taskId, "compliance-officer-001")` → ASSIGNED
5. `workItemService.start(taskId, "compliance-officer-001")` → IN_PROGRESS
6. `workItemService.complete(taskId, "compliance-officer-001", "SAR approved", "APPROVED")`
   // 4-param: id, actorId, resolution, outcome — fires both sync and async WorkItemLifecycleEvent
7. Await `@ObservesAsync` delivery: poll evidence endpoint until `auditChain.events`
   contains `SAR_OFFICER_REVIEWED`, or Awaitility with ≤ 5s timeout
8. Assert: `auditChain.events` contains event with `actorId = "compliance-officer-001"` and
   `actorRole = "ComplianceOfficer"`; assert `auditChain.status` is one of `{CLOSED, PARTIAL}`
   // CLOSED requires chainVerified=true; hash-chain is disabled in H2
   // (casehub.ledger.hash-chain.enabled=false) so chainVerified=false always in tests
   // The meaningful assertion is the presence and fields of the SAR_OFFICER_REVIEWED event
9. `POST /api/actors/compliance-officer-001/erasure` → 200, `erasedCount >= 1`
10. `GET /api/investigations/{caseId}/compliance-evidence` → assert SAR_OFFICER_REVIEWED
    event `actorId` is pseudonymized (not `"compliance-officer-001"`)

### `@QuarkusTest` — reconciliation path

`AmlTrustAttestationRepository` has no `delete()` method. Use JPQL in a `@Transactional`
test helper — no production code change required:

```java
@Transactional
void deleteAttestation(UUID id) {
    em.createQuery("DELETE FROM AmlTrustRoutingAttestation a WHERE a.id = :id")
      .setParameter("id", id)
      .executeUpdate();
    em.clear(); // flush second-level cache so reconciler sees the gap
}
```

1. Start investigation, poll to complete
2. Query `attestationRepo.findByInvestigationCaseId(caseId)` — pick first entry's `id`.
   Call `deleteAttestation(id)` — removes the row and clears the cache
3. `GET /api/investigations/{caseId}/compliance-evidence` → assert `trustRouting.status = PARTIAL`;
   deleted capability appears in decisions with `reconstructed = true`
4. Call evidence endpoint again → assert no duplicate reconstructed entry; count still 1

---

## 8. Out of Scope

None. aml#56 is closed in this spec.

---

## 9. Platform Coherence Review

- Ledger subclass rule (PP-20260513): JOINED inheritance ✅, V2009/V2010 numbering ✅, consumer-owned migrations ✅
- Observer failure pattern (PP-20260530-49856c): double try/catch on both new observers; `writeSarOfficerReviewedFailure` uses `REQUIRES_NEW` ✅
- Observer failure naming (PP-20260531-11724b): `actorRole = "<role>-observer-failed"` ✅
- Application-tier rule: all logic requires AML domain knowledge ✅
- Dual-trail audit pattern: `WorkItemLifecycleEvent` path maintains both operational and compliance record ✅
- Multi-JVM idempotency: partial unique index `WHERE reconstructed = TRUE` ✅
- Exception catch: `PersistenceException` with `getCause() instanceof ConstraintViolationException`; surrounds entire synchronized block; re-throws non-constraint errors ✅
- `RoutingDecisionRecord`: updated 7-field declaration with construction call shown ✅
- `openReview()` consolidation: not `@Transactional` (cross-datasource — default + qhorus); consolidated call not atomic; partial-failure risk acknowledged ✅
- All broken call sites and test constructors documented ✅
- H2 hash-chain limitation acknowledged in GDPR test ✅
