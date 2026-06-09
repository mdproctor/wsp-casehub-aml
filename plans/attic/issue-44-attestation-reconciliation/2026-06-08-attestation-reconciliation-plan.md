# Trust Attestation Reconciliation + SAR Officer Reviewed Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Harden `AmlTrustRoutingObserver` against silent failures, add lazy attestation gap reconciliation on evidence read, and record compliance officer SAR decisions in the tamper-evident ledger with a human actorId to enable real GDPR Art.17 erasure.

**Architecture:** Five interlinked changes — (1) schema changes (V2009/V2010), (2) `ComplianceReviewLifecycle` consolidates WorkItem creation + ledger entry write so all paths write `COMPLIANCE_REVIEW_OPENED` (closes #56), (3) `AmlTrustRoutingObserver` gains double-try/catch per PP-20260530-49856c, (4) new `AmlAttestationReconciler` repairs gaps lazily when compliance evidence is read, (5) new `AmlWorkItemLifecycleObserver` writes `AmlSarOfficerReviewedLedgerEntry` when the compliance officer completes or rejects the WorkItem.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate 6, Flyway, H2 (tests), JUnit 5, Mockito, REST Assured, Awaitility, casehub-ledger, casehub-work, casehub-engine.

**Spec:** `specs/issue-44-attestation-reconciliation/2026-06-07-attestation-reconciliation-sar-officer-design.md`

**Issues:** casehubio/aml#44 (attestation reconciliation), casehubio/aml#55 (SAR officer reviewed), aml#56 (closed in scope)

**Build/test commands:**
- Single test class: `mvn -pl app -am -Dsurefire.failIfNoSpecifiedTests=false -Dtest=<ClassName> test`
- All tests: `mvn -pl app -am test`

---

## File Map

**New files:**
| File | Responsibility |
|---|---|
| `app/src/main/resources/db/aml-trust-routing/migration/V2009__aml_trust_routing_attestation_reconstructed.sql` | Add `reconstructed`, `observer_failed` columns + partial unique index |
| `app/src/main/resources/db/aml-ledger/migration/V2010__aml_sar_officer_reviewed_ledger_entry.sql` | Create officer-reviewed join table |
| `app/src/main/java/io/casehub/aml/ledger/AmlSarOfficerReviewedLedgerEntry.java` | New `LedgerEntry` subclass for officer SAR decision |
| `app/src/main/java/io/casehub/aml/trust/AmlAttestationReconciler.java` | Lazy attestation gap repair service |
| `app/src/main/java/io/casehub/aml/compliance/AmlWorkItemLifecycleObserver.java` | `@ObservesAsync WorkItemLifecycleEvent` → writes officer ledger entry |
| `app/src/test/java/io/casehub/aml/trust/AmlAttestationReconcilerTest.java` | Unit tests for reconciler |
| `app/src/test/java/io/casehub/aml/compliance/AmlWorkItemLifecycleObserverTest.java` | Unit tests for WorkItem observer |

**Modified files:**
| File | What changes |
|---|---|
| `app/src/main/java/io/casehub/aml/trust/AmlTrustRoutingAttestation.java` | Add `reconstructed`, `observerFailed` boolean fields |
| `app/src/main/java/io/casehub/aml/trust/AmlTrustAttestationRepository.java` | Add `saveObserverFailureEntry(event, subject, threshold)` |
| `app/src/main/java/io/casehub/aml/trust/AmlTrustRoutingObserver.java` | Double-try/catch hardening (PP-20260530-49856c) |
| `app/src/main/java/io/casehub/aml/ledger/AmlLedgerService.java` | Add `writeSarOfficerReviewed()`, `writeSarOfficerReviewedFailure()`; update `noOp()`, `stub()` |
| `app/src/main/java/io/casehub/aml/ComplianceReviewLifecycle.java` | Add `caseId` param; inject `AmlLedgerService`; write ledger entry inside (closes #56) |
| `app/src/main/java/io/casehub/aml/AmlInvestigationCoordinator.java` | Pass `caseId` to `openReview()`; remove redundant `writeComplianceReviewOpened()` call |
| `app/src/main/java/io/casehub/aml/engine/AmlEngineCoordinator.java` | Add `caseHub.signal(caseId, "caseId", ...)` after `startCase()` |
| `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseHub.java` | Sar-drafting workers: read `caseId` from context, pass to `openReview()` |
| `api/src/main/java/io/casehub/aml/compliance/RoutingDecisionRecord.java` | Add `reconstructed`, `observerFailed` fields (7-param record) |
| `app/src/main/java/io/casehub/aml/compliance/AmlComplianceEvidenceService.java` | 6-param constructor; new `filterSarOfficerReviewed()`; updated `build()`, `buildAuditChain()`, `assembleEvidence()`, `findEvidence()`, `buildTrustRouting()` |
| `app/src/test/java/io/casehub/aml/compliance/AmlComplianceEvidenceServiceTest.java` | 6-arg constructor + reconciler mock; updated happy-path assertion; new status tests |
| `app/src/test/java/io/casehub/aml/ComplianceReviewLifecycleTest.java` | 2-arg test constructor; `@Mock AmlLedgerService`; updated assertions |
| `app/src/test/java/io/casehub/aml/AmlInvestigationCoordinatorTest.java` | 2-arg `ComplianceReviewLifecycle` construction with `AmlLedgerService.noOp()` |
| `app/src/test/java/io/casehub/aml/compliance/AmlLayer7ResourceTest.java` | Update existing test (sla now PARTIAL, audit chain has 2 events); add GDPR + reconciliation tests |

---

## Task 1: Flyway Migrations (V2009 + V2010)

**Files:**
- Create: `app/src/main/resources/db/aml-trust-routing/migration/V2009__aml_trust_routing_attestation_reconstructed.sql`
- Create: `app/src/main/resources/db/aml-ledger/migration/V2010__aml_sar_officer_reviewed_ledger_entry.sql`

Both paths are already in `quarkus.flyway.qhorus.locations` in `app/src/test/resources/application.properties` — no properties change needed.

- [ ] **Step 1: Write V2009 migration**

Create `app/src/main/resources/db/aml-trust-routing/migration/V2009__aml_trust_routing_attestation_reconstructed.sql`:

```sql
-- Add reconstructed and observer_failed flags to aml_trust_routing_attestation.
-- reconstructed=TRUE: written by AmlAttestationReconciler to fill an observer gap.
-- observer_failed=TRUE: written by AmlTrustRoutingObserver outer catch (failure audit record).
-- Partial unique index prevents multi-JVM double-writes of reconstructed entries.
ALTER TABLE aml_trust_routing_attestation
    ADD COLUMN reconstructed   BOOLEAN NOT NULL DEFAULT FALSE,
    ADD COLUMN observer_failed BOOLEAN NOT NULL DEFAULT FALSE;

CREATE UNIQUE INDEX UQ_TRUST_ATTEST_CASE_CAP_RECONSTRUCTED
    ON aml_trust_routing_attestation (investigation_case_id, capability_tag)
    WHERE reconstructed = TRUE;
```

- [ ] **Step 2: Write V2010 migration**

Create `app/src/main/resources/db/aml-ledger/migration/V2010__aml_sar_officer_reviewed_ledger_entry.sql`:

```sql
-- Join table for AmlSarOfficerReviewedLedgerEntry (JOINED inheritance from ledger_entry).
-- Stores the officer's review decision (APPROVED or REJECTED) when they act on the SAR WorkItem.
-- The officer's actorId (human PII) is in ledger_entry.actor_id — GDPR-erasable via LedgerErasureService.
CREATE TABLE aml_sar_officer_reviewed_ledger_entry (
    id              UUID        NOT NULL,
    review_decision VARCHAR(20) NOT NULL,
    CONSTRAINT pk_aml_sar_officer_reviewed PRIMARY KEY (id),
    CONSTRAINT fk_aml_sar_officer_reviewed_ledger FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

- [ ] **Step 3: Verify migrations run**

```bash
mvn -pl app -am -Dsurefire.failIfNoSpecifiedTests=false -Dtest=AmlLayer7ResourceTest test
```

Expected: tests pass (Flyway runs V2009 + V2010 on qhorus datasource without error). If Flyway startup fails, the `@QuarkusTest` will error before any test method runs.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/resources/db/aml-trust-routing/migration/V2009__aml_trust_routing_attestation_reconstructed.sql
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/resources/db/aml-ledger/migration/V2010__aml_sar_officer_reviewed_ledger_entry.sql
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-44): add V2009/V2010 Flyway migrations for attestation flags and officer review table

Refs #44 #55"
```

---

## Task 2: Data Model Updates

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/trust/AmlTrustRoutingAttestation.java`
- Create: `app/src/main/java/io/casehub/aml/ledger/AmlSarOfficerReviewedLedgerEntry.java`
- Modify: `api/src/main/java/io/casehub/aml/compliance/RoutingDecisionRecord.java`

- [ ] **Step 1: Add fields to `AmlTrustRoutingAttestation`**

In `app/src/main/java/io/casehub/aml/trust/AmlTrustRoutingAttestation.java`, add two fields after `investigationCaseId`:

```java
@Column(name = "reconstructed", nullable = false)
public boolean reconstructed = false;

@Column(name = "observer_failed", nullable = false)
public boolean observerFailed = false;
```

- [ ] **Step 2: Create `AmlSarOfficerReviewedLedgerEntry`**

Create `app/src/main/java/io/casehub/aml/ledger/AmlSarOfficerReviewedLedgerEntry.java`:

```java
package io.casehub.aml.ledger;

import io.casehub.ledger.runtime.model.LedgerEntry;
import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

/**
 * Layer 9: records the compliance officer's SAR approval or rejection decision.
 *
 * <p>Written by {@link io.casehub.aml.compliance.AmlWorkItemLifecycleObserver} when the
 * compliance officer completes or rejects the SAR review WorkItem. The officer's identity
 * is stored in {@code actorId} (HUMAN) — GDPR Art.17 erasable via
 * {@link io.casehub.ledger.runtime.privacy.LedgerErasureService}.
 *
 * <p>{@code causedByEntryId} points to the {@link AmlComplianceReviewLedgerEntry} that
 * opened the review, completing the causal chain.
 */
@Entity
@Table(name = "aml_sar_officer_reviewed_ledger_entry")
@DiscriminatorValue("AML_SAR_OFFICER_REVIEWED")
public class AmlSarOfficerReviewedLedgerEntry extends LedgerEntry {

    /** "APPROVED" or "REJECTED" — the officer's explicit SAR verdict. */
    @Column(name = "review_decision", nullable = false, length = 20)
    public String reviewDecision;
}
```

- [ ] **Step 3: Update `RoutingDecisionRecord`**

Replace the full content of `api/src/main/java/io/casehub/aml/compliance/RoutingDecisionRecord.java`:

```java
package io.casehub.aml.compliance;

import java.util.UUID;

public record RoutingDecisionRecord(
    String capabilityTag,
    String selectedWorker,
    Double trustScoreAtRouting,
    double thresholdApplied,
    UUID attestationEntryId,
    boolean reconstructed,
    boolean observerFailed
) {}
```

- [ ] **Step 4: Build to verify compilation**

```bash
mvn -pl app -am -Dsurefire.failIfNoSpecifiedTests=false -Dtest=AmlTrustRoutingAttestationTest test
```

Expected: FAIL — `AmlComplianceEvidenceService` construction call site still uses 5-arg constructor. Compilation error is expected and will be fixed in Task 12.

Actually — let's just build `api` and `app` to see compile errors:

```bash
mvn -pl app -am compile -q 2>&1 | head -40
```

Expected: compilation errors on `RoutingDecisionRecord` construction calls in `AmlComplianceEvidenceService` and `AmlComplianceEvidenceServiceTest`. This is correct — these will be fixed in Task 12.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/trust/AmlTrustRoutingAttestation.java
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/ledger/AmlSarOfficerReviewedLedgerEntry.java
git -C /Users/mdproctor/claude/casehub/aml add api/src/main/java/io/casehub/aml/compliance/RoutingDecisionRecord.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-44): add reconstructed/observerFailed to attestation; new AmlSarOfficerReviewedLedgerEntry; update RoutingDecisionRecord

Refs #44 #55"
```

---

## Task 3: `AmlLedgerService` — New Methods + Stubs

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/ledger/AmlLedgerService.java`

- [ ] **Step 1: Add `writeSarOfficerReviewed()` and `writeSarOfficerReviewedFailure()`**

Add these two methods to `AmlLedgerService` (after `writeComplianceReviewOpened()`):

```java
/**
 * Write an AML_SAR_OFFICER_REVIEWED entry when the compliance officer approves or rejects the SAR.
 *
 * <p>Called from {@link io.casehub.aml.compliance.AmlWorkItemLifecycleObserver} which runs in an
 * {@code @ObservesAsync} context — no transaction is propagated, so REQUIRED starts a new transaction
 * on the qhorus datasource.
 *
 * <p>{@code causedByEntryId} is self-derived from the AmlComplianceReviewLedgerEntry for this case.
 */
@Transactional(TxType.REQUIRED)
public void writeSarOfficerReviewed(final UUID caseId, final String officerId,
        final String reviewDecision) {
    final UUID causedBy = repository.findBySubjectId(caseId).stream()
            .filter(AmlComplianceReviewLedgerEntry.class::isInstance)
            .map(e -> e.id)
            .findFirst()
            .orElse(null);
    final int sequenceNumber = nextSequenceNumber(caseId);
    final AmlSarOfficerReviewedLedgerEntry entry = new AmlSarOfficerReviewedLedgerEntry();
    entry.id = UUID.randomUUID();
    entry.subjectId = caseId;
    entry.sequenceNumber = sequenceNumber;
    entry.entryType = LedgerEntryType.EVENT;
    entry.actorId = officerId;
    entry.actorType = ActorType.HUMAN;
    entry.actorRole = "ComplianceOfficer";
    entry.occurredAt = Instant.now();
    entry.causedByEntryId = causedBy;
    entry.reviewDecision = reviewDecision;
    repository.save(entry);
}

/**
 * Write an observer-failure record when the main SAR_OFFICER_REVIEWED write fails.
 *
 * <p>Per PP-20260530-49856c: failure entry writer must use REQUIRES_NEW so the record
 * commits independently of any surrounding (possibly failing) transaction context.
 */
@Transactional(TxType.REQUIRES_NEW)
public void writeSarOfficerReviewedFailure(final UUID caseId, final String officerId) {
    final int sequenceNumber = nextSequenceNumber(caseId);
    final AmlSarOfficerReviewedLedgerEntry entry = new AmlSarOfficerReviewedLedgerEntry();
    entry.id = UUID.randomUUID();
    entry.subjectId = caseId;
    entry.sequenceNumber = sequenceNumber;
    entry.entryType = LedgerEntryType.EVENT;
    entry.actorId = ACTOR_ID;
    entry.actorType = ActorType.SYSTEM;
    entry.actorRole = "ComplianceOfficer-observer-failed";
    entry.occurredAt = Instant.now();
    entry.reviewDecision = "UNKNOWN";
    repository.save(entry);
}
```

- [ ] **Step 2: Update `noOp()` and `stub()` stubs to override the new methods**

In the `noOp()` anonymous class, add overrides:

```java
public static AmlLedgerService noOp() {
    return new AmlLedgerService() {
        @Override public UUID writeCaseOpened(SuspiciousTransaction tx, UUID caseId) { return UUID.randomUUID(); }
        @Override public void writeComplianceReviewOpened(UUID caseId, String taskId) {}
        @Override public void writeSarOfficerReviewed(UUID caseId, String officerId, String decision) {}
        @Override public void writeSarOfficerReviewedFailure(UUID caseId, String officerId) {}
    };
}
```

In the `stub()` anonymous class, add the same overrides:

```java
public static AmlLedgerService stub(final UUID entryId) {
    return new AmlLedgerService() {
        @Override public UUID writeCaseOpened(SuspiciousTransaction tx, UUID caseId) { return entryId; }
        @Override public void writeComplianceReviewOpened(UUID caseId, String taskId) {}
        @Override public void writeSarOfficerReviewed(UUID caseId, String officerId, String decision) {}
        @Override public void writeSarOfficerReviewedFailure(UUID caseId, String officerId) {}
    };
}
```

Also add the missing imports at the top of `AmlLedgerService.java`:

```java
import io.casehub.aml.ledger.AmlSarOfficerReviewedLedgerEntry;
import jakarta.transaction.Transactional.TxType;
```

- [ ] **Step 3: Compile check**

```bash
mvn -pl app -am compile -q 2>&1 | grep -E "ERROR|error:" | grep -v "RoutingDecisionRecord" | head -20
```

Expected: only errors related to `RoutingDecisionRecord` call sites (Task 12). No new errors from `AmlLedgerService`.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/ledger/AmlLedgerService.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-55): AmlLedgerService — writeSarOfficerReviewed + writeSarOfficerReviewedFailure

Refs #55"
```

---

## Task 4: `AmlTrustAttestationRepository` — `saveObserverFailureEntry()`

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/trust/AmlTrustAttestationRepository.java`

- [ ] **Step 1: Add `saveObserverFailureEntry()` method**

Add after `saveWithSequence()`:

```java
/**
 * Writes an observer-failure attestation entry in an isolated REQUIRES_NEW transaction.
 *
 * <p>Called from {@link AmlTrustRoutingObserver} outer catch when the main attestation write fails.
 * Per PP-20260530-49856c: failure entry writer uses REQUIRES_NEW so the record commits
 * independently regardless of any surrounding transaction state.
 *
 * @param event     the WorkerDecisionEvent that triggered the observer
 * @param subject   the attestation-specific subject UUID (derived by the observer)
 * @param threshold the trust threshold from the policy provider (computed before the try block)
 */
@Transactional(TxType.REQUIRES_NEW)
public AmlTrustRoutingAttestation saveObserverFailureEntry(
        final WorkerDecisionEvent event,
        final UUID subject,
        final double threshold) {
    Integer max = em.createQuery(
                    "SELECT MAX(e.sequenceNumber) FROM LedgerEntry e WHERE e.subjectId = :sid",
                    Integer.class)
            .setParameter("sid", subject)
            .getSingleResult();
    final AmlTrustRoutingAttestation entry = new AmlTrustRoutingAttestation();
    entry.id = UUID.randomUUID();
    entry.subjectId = subject;
    entry.investigationCaseId = event.caseId();
    entry.capabilityTag = event.capabilityTag();
    entry.selectedWorkerId = event.workerId();
    entry.trustScoreAtRouting = null;
    entry.thresholdApplied = threshold;
    entry.entryType = LedgerEntryType.EVENT;
    entry.actorId = "aml-orchestrator";
    entry.actorType = ActorType.SYSTEM;
    entry.actorRole = "AmlInvestigationOrchestrator-observer-failed";
    entry.occurredAt = Instant.now();
    entry.sequenceNumber = max == null ? 1 : max + 1;
    entry.reconstructed = false;
    entry.observerFailed = true;
    em.persist(entry);
    return entry;
}
```

Add the missing import at the top (if not already present):

```java
import io.casehub.engine.common.spi.event.WorkerDecisionEvent;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.platform.api.identity.ActorType;
```

- [ ] **Step 2: Compile check**

```bash
mvn -pl app -am compile -q 2>&1 | grep -E "error:" | grep "AmlTrustAttestationRepository" | head -10
```

Expected: no errors for this class.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/trust/AmlTrustAttestationRepository.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-44): AmlTrustAttestationRepository — saveObserverFailureEntry (REQUIRES_NEW)

Refs #44"
```

---

## Task 5: `AmlAttestationReconciler` — New Service + Unit Tests

**Files:**
- Create: `app/src/main/java/io/casehub/aml/trust/AmlAttestationReconciler.java`
- Create: `app/src/test/java/io/casehub/aml/trust/AmlAttestationReconcilerTest.java`

- [ ] **Step 1: Write failing unit tests**

Create `app/src/test/java/io/casehub/aml/trust/AmlAttestationReconcilerTest.java`:

```java
package io.casehub.aml.trust;

import io.casehub.ledger.model.WorkerDecisionEntry;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.ledger.api.model.LedgerEntryType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class AmlAttestationReconcilerTest {

    @Mock AmlTrustAttestationRepository attestationRepo;
    AmlAttestationReconciler reconciler;
    UUID caseId = UUID.randomUUID();

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        reconciler = new AmlAttestationReconciler(attestationRepo);
        when(attestationRepo.saveWithSequence(any())).thenAnswer(inv -> inv.getArgument(0));
    }

    @Test
    void reconcileIfNeeded_missingAttestation_writesReconstructedEntry() {
        var dispatched = List.of(
            workerDecision("entity-resolution", "agent-A", 0.8, 0.7),
            workerDecision("sar-drafting", "agent-B", 0.9, 0.75),
            workerDecision("osint-screening", "agent-C", null, null)
        );
        var existing = List.of(
            attestation("entity-resolution", "agent-A"),
            attestation("sar-drafting", "agent-B")
        );

        List<AmlTrustRoutingAttestation> result =
            reconciler.reconcileIfNeeded(caseId, dispatched, existing);

        assertEquals(3, result.size(), "Should have 3 total: 2 existing + 1 reconstructed");
        ArgumentCaptor<AmlTrustRoutingAttestation> captor =
            ArgumentCaptor.forClass(AmlTrustRoutingAttestation.class);
        verify(attestationRepo).saveWithSequence(captor.capture());
        AmlTrustRoutingAttestation saved = captor.getValue();
        assertEquals("osint-screening", saved.capabilityTag);
        assertEquals("agent-C", saved.selectedWorkerId);
        assertTrue(saved.reconstructed);
        assertFalse(saved.observerFailed);
        assertNull(saved.trustScoreAtRouting, "Phase-0 case: score is null");
        assertEquals(0.0, saved.thresholdApplied, "null threshold maps to 0.0");
    }

    @Test
    void reconcileIfNeeded_allCovered_noWrite() {
        var dispatched = List.of(workerDecision("entity-resolution", "agent-A", 0.8, 0.7));
        var existing = List.of(attestation("entity-resolution", "agent-A"));

        List<AmlTrustRoutingAttestation> result =
            reconciler.reconcileIfNeeded(caseId, dispatched, existing);

        assertEquals(1, result.size());
        verify(attestationRepo, never()).saveWithSequence(any());
    }

    @Test
    void reconcileIfNeeded_idempotent_noDuplicateWrite() {
        var dispatched = List.of(workerDecision("entity-resolution", "agent-A", 0.8, 0.7));
        // Existing already has a reconstructed entry
        var reconstructed = attestation("entity-resolution", "agent-A");
        reconstructed.reconstructed = true;
        var existing = List.of(reconstructed);

        reconciler.reconcileIfNeeded(caseId, dispatched, existing);
        reconciler.reconcileIfNeeded(caseId, dispatched, existing);

        verify(attestationRepo, never()).saveWithSequence(any());
    }

    @Test
    void reconcileIfNeeded_observerFailedEntry_treatedAsCovered_notReReconciled() {
        var dispatched = List.of(workerDecision("entity-resolution", "agent-A", 0.8, 0.7));
        var failureEntry = attestation("entity-resolution", "agent-A");
        failureEntry.observerFailed = true;
        var existing = List.of(failureEntry);

        List<AmlTrustRoutingAttestation> result =
            reconciler.reconcileIfNeeded(caseId, dispatched, existing);

        assertEquals(1, result.size(), "Observer-failed entry covers the capability; no new entry");
        verify(attestationRepo, never()).saveWithSequence(any());
    }

    @Test
    void reconcileIfNeeded_copiesTrustScoreFromWorkerDecisionEntry() {
        var dispatched = List.of(workerDecision("pattern-analysis", "agent-P", 0.72, 0.65));
        var existing = List.<AmlTrustRoutingAttestation>of();

        reconciler.reconcileIfNeeded(caseId, dispatched, existing);

        ArgumentCaptor<AmlTrustRoutingAttestation> captor =
            ArgumentCaptor.forClass(AmlTrustRoutingAttestation.class);
        verify(attestationRepo).saveWithSequence(captor.capture());
        AmlTrustRoutingAttestation saved = captor.getValue();
        assertEquals(0.72, saved.trustScoreAtRouting);
        assertEquals(0.65, saved.thresholdApplied);
    }

    // -- Helpers --

    private WorkerDecisionEntry workerDecision(String cap, String worker,
            Double score, Double threshold) {
        var e = new WorkerDecisionEntry();
        e.capabilityTag = cap;
        e.workerId = worker;
        e.caseId = caseId;
        e.trustScoreAtRouting = score;
        e.thresholdApplied = threshold;
        e.tenancyId = "default";
        e.subjectId = caseId;
        e.id = UUID.randomUUID();
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = worker;
        e.actorType = ActorType.SYSTEM;
        e.actorRole = "engine";
        e.occurredAt = Instant.now();
        e.sequenceNumber = 1;
        return e;
    }

    private AmlTrustRoutingAttestation attestation(String cap, String worker) {
        var a = new AmlTrustRoutingAttestation();
        a.id = UUID.randomUUID();
        a.capabilityTag = cap;
        a.selectedWorkerId = worker;
        a.investigationCaseId = caseId;
        a.trustScoreAtRouting = 0.8;
        a.thresholdApplied = 0.7;
        a.reconstructed = false;
        a.observerFailed = false;
        a.subjectId = AmlTrustRoutingObserver.attestationSubjectFor(caseId);
        a.entryType = LedgerEntryType.EVENT;
        a.actorId = "aml-orchestrator";
        a.actorType = ActorType.SYSTEM;
        a.actorRole = "AmlInvestigationOrchestrator";
        a.occurredAt = Instant.now();
        a.sequenceNumber = 1;
        return a;
    }
}
```

- [ ] **Step 2: Run to confirm tests fail (class not found)**

```bash
mvn -pl app -am -Dsurefire.failIfNoSpecifiedTests=false -Dtest=AmlAttestationReconcilerTest test 2>&1 | tail -20
```

Expected: compilation failure — `AmlAttestationReconciler` does not exist yet.

- [ ] **Step 3: Create `AmlAttestationReconciler`**

Create `app/src/main/java/io/casehub/aml/trust/AmlAttestationReconciler.java`:

```java
package io.casehub.aml.trust;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.model.WorkerDecisionEntry;
import io.casehub.platform.api.identity.ActorType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.PersistenceException;
import org.hibernate.exception.ConstraintViolationException;
import org.jboss.logging.Logger;

import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Layer 9: lazily fills attestation gaps when compliance evidence is read.
 *
 * <p>Called from {@link io.casehub.aml.compliance.AmlComplianceEvidenceService#buildTrustRouting}
 * on every GET of the compliance evidence endpoint. For each capability dispatched by the engine
 * (via WorkerDecisionEntry) that has no AmlTrustRoutingAttestation, writes a reconstructed entry
 * using the authoritative trust data from WorkerDecisionEntry.
 *
 * <p>Idempotent: capabilities already covered (any reconstructed/observerFailed value) are skipped.
 * Multi-JVM safe: a partial unique index (UQ_TRUST_ATTEST_CASE_CAP_RECONSTRUCTED) prevents
 * duplicate reconstructed entries; a constraint violation is caught and treated as idempotent.
 */
@ApplicationScoped
public class AmlAttestationReconciler {

    private static final Logger LOG = Logger.getLogger(AmlAttestationReconciler.class);
    private static final String ACTOR_ID = "aml-orchestrator";
    private static final String ACTOR_ROLE = "AmlInvestigationOrchestrator";

    private final ConcurrentHashMap<UUID, Object> subjectLocks = new ConcurrentHashMap<>();
    private final AmlTrustAttestationRepository attestationRepo;

    @Inject
    public AmlAttestationReconciler(AmlTrustAttestationRepository attestationRepo) {
        this.attestationRepo = attestationRepo;
    }

    /**
     * Checks for attestation gaps and writes reconstructed entries for any missing capabilities.
     *
     * @param caseId     the investigation case UUID
     * @param dispatched all WorkerDecisionEntry records for this case (from the engine)
     * @param existing   all existing AmlTrustRoutingAttestation records for this case
     * @return existing list + any newly written reconstructed entries
     */
    public List<AmlTrustRoutingAttestation> reconcileIfNeeded(
            final UUID caseId,
            final List<WorkerDecisionEntry> dispatched,
            final List<AmlTrustRoutingAttestation> existing) {

        final Set<String> coveredCaps = new HashSet<>();
        for (AmlTrustRoutingAttestation a : existing) {
            if (a.capabilityTag != null) coveredCaps.add(a.capabilityTag);
        }

        final List<AmlTrustRoutingAttestation> result = new ArrayList<>(existing);
        final UUID attestationSubject = AmlTrustRoutingObserver.attestationSubjectFor(caseId);
        final Object lock = subjectLocks.computeIfAbsent(attestationSubject, k -> new Object());

        for (WorkerDecisionEntry decision : dispatched) {
            final String capTag = decision.capabilityTag;
            if (capTag == null || coveredCaps.contains(capTag)) continue;

            final AmlTrustRoutingAttestation entry = new AmlTrustRoutingAttestation();
            entry.id = UUID.randomUUID();
            entry.subjectId = attestationSubject;
            entry.investigationCaseId = caseId;
            entry.capabilityTag = capTag;
            entry.selectedWorkerId = decision.workerId;
            entry.trustScoreAtRouting = decision.trustScoreAtRouting;
            entry.thresholdApplied = decision.thresholdApplied != null
                    ? decision.thresholdApplied : 0.0;
            entry.entryType = LedgerEntryType.EVENT;
            entry.actorId = ACTOR_ID;
            entry.actorType = ActorType.SYSTEM;
            entry.actorRole = ACTOR_ROLE;
            entry.occurredAt = Instant.now();
            entry.reconstructed = true;
            entry.observerFailed = false;

            try {
                synchronized (lock) {
                    attestationRepo.saveWithSequence(entry);
                }
                result.add(entry);
                coveredCaps.add(capTag);
            } catch (PersistenceException e) {
                if (e.getCause() instanceof ConstraintViolationException) {
                    // Peer JVM already reconciled this capability — idempotent.
                    // Peer's entry is in DB but absent from this request's merged list;
                    // status correctly shows PARTIAL for this request; next request reads correctly.
                    LOG.debugf("Peer JVM reconciled caseId=%s cap=%s — skipping duplicate",
                            caseId, capTag);
                } else {
                    throw e;
                }
            }
        }

        return result;
    }
}
```

Note: this class uses a package-private method `AmlTrustRoutingObserver.attestationSubjectFor(caseId)`. That method is currently package-private (no access modifier). Since both classes are in `io.casehub.aml.trust`, this works.

- [ ] **Step 4: Run tests to verify they pass**

```bash
mvn -pl app -am -Dsurefire.failIfNoSpecifiedTests=false -Dtest=AmlAttestationReconcilerTest test
```

Expected: all 5 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/trust/AmlAttestationReconciler.java
git -C /Users/mdproctor/claude/casehub/aml add app/src/test/java/io/casehub/aml/trust/AmlAttestationReconcilerTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-44): AmlAttestationReconciler — lazy gap repair with multi-JVM idempotency

Refs #44"
```

---

## Task 6: `ComplianceReviewLifecycle` Refactor + Coordinator Update + Test Fixes

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/ComplianceReviewLifecycle.java`
- Modify: `app/src/main/java/io/casehub/aml/AmlInvestigationCoordinator.java`
- Modify: `app/src/test/java/io/casehub/aml/ComplianceReviewLifecycleTest.java`
- Modify: `app/src/test/java/io/casehub/aml/AmlInvestigationCoordinatorTest.java`

This task closes aml#56 (engine path not writing COMPLIANCE_REVIEW_OPENED).

- [ ] **Step 1: Update `ComplianceReviewLifecycleTest` (failing state first)**

Replace the full content of `app/src/test/java/io/casehub/aml/ComplianceReviewLifecycleTest.java`:

```java
package io.casehub.aml;

import io.casehub.aml.domain.*;
import io.casehub.aml.ledger.AmlLedgerService;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemCreateRequest;
import io.casehub.work.runtime.model.WorkItemPriority;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.time.Duration;
import java.time.Instant;
import java.util.UUID;
import java.util.concurrent.atomic.AtomicReference;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class ComplianceReviewLifecycleTest {

    @Mock AmlLedgerService mockLedger;

    private final SuspiciousTransaction tx = new SuspiciousTransaction(
            "TXN-CLR", "ACC-A", "ACC-B",
            new BigDecimal("75000"), "USD",
            Instant.parse("2024-06-01T00:00:00Z"), "Test");

    private final InvestigationSummary summary = new InvestigationSummary(
            tx,
            new SpecialistOutcome.Completed<>(new EntityResolutionResult("E-1", "chain", "CORPORATE", 0.35)),
            new SpecialistOutcome.Completed<>(new PatternAnalysisResult(true, "structuring")),
            new SpecialistOutcome.Declined<>("agent", "osint-screening", "clearance"),
            "narrative");

    private static WorkItem workItemWith(UUID id) {
        WorkItem wi = new WorkItem();
        wi.id = id;
        return wi;
    }

    @Test
    void openReview_creates30DayClaimDeadline_andWritesLedgerEntry() {
        AtomicReference<WorkItemCreateRequest> captured = new AtomicReference<>();
        UUID caseId = UUID.randomUUID();
        ComplianceReviewLifecycle lifecycle = new ComplianceReviewLifecycle(request -> {
            captured.set(request);
            return workItemWith(UUID.randomUUID());
        }, mockLedger);

        lifecycle.openReview(tx, summary, caseId);

        WorkItemCreateRequest req = captured.get();
        assertNotNull(req);
        assertEquals(WorkItemPriority.HIGH, req.priority);
        assertEquals("compliance-officers", req.candidateGroups);

        // callerRef uses caseId UUID (not transaction.id()) with "aml:investigation:" prefix
        assertTrue(req.callerRef.startsWith("aml:investigation:"),
            "callerRef must start with aml:investigation:");
        assertDoesNotThrow(() -> UUID.fromString(req.callerRef.substring("aml:investigation:".length())),
            "callerRef suffix must be a valid UUID");

        long days = Duration.between(Instant.now(), req.claimDeadline).toDays();
        assertTrue(days >= 29 && days <= 30, "claimDeadline should be ~30 days, was " + days);

        // Verify ledger entry was written with the correct caseId
        verify(mockLedger).writeComplianceReviewOpened(eq(caseId), any());
    }

    @Test
    void openReview_returnsWorkItemId() {
        UUID expectedId = UUID.randomUUID();
        ComplianceReviewLifecycle lifecycle = new ComplianceReviewLifecycle(
                req -> workItemWith(expectedId), mockLedger);

        String taskId = lifecycle.openReview(tx, summary, UUID.randomUUID());
        assertEquals(expectedId.toString(), taskId);
    }
}
```

- [ ] **Step 2: Run to confirm tests fail (constructor not yet updated)**

```bash
mvn -pl app -am -Dsurefire.failIfNoSpecifiedTests=false -Dtest=ComplianceReviewLifecycleTest test 2>&1 | tail -20
```

Expected: compilation error — `ComplianceReviewLifecycle` has no 2-arg test constructor and `openReview` is 2-param.

- [ ] **Step 3: Update `ComplianceReviewLifecycle`**

Replace the full content of `app/src/main/java/io/casehub/aml/ComplianceReviewLifecycle.java`:

```java
package io.casehub.aml;

import io.casehub.aml.domain.InvestigationSummary;
import io.casehub.aml.domain.SpecialistOutcome;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.ledger.AmlLedgerService;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemCreateRequest;
import io.casehub.work.runtime.model.WorkItemPriority;
import io.casehub.work.runtime.service.WorkItemService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;
import java.util.function.Function;

/**
 * Opens a compliance officer review WorkItem and writes the COMPLIANCE_REVIEW_OPENED
 * ledger entry in a single consolidated call.
 *
 * <p>Previously, AmlInvestigationCoordinator called openReview() and then separately
 * called ledgerService.writeComplianceReviewOpened(). The engine path (Quartz workers)
 * called only openReview(), never writing the ledger entry (aml#56). This consolidation
 * ensures both operations always occur together regardless of the caller path.
 *
 * <p>Not @Transactional: WorkItemService.create() writes to the default datasource;
 * writeComplianceReviewOpened() writes to the qhorus datasource. These are two separate
 * non-XA datasources — Narayana LRC allows only one non-XA resource per XA transaction.
 * Partial-failure risk (WorkItem committed, ledger write fails) is accepted and treated
 * as an AUDIT GAP condition consistent with other observer-failure patterns in the system.
 */
@ApplicationScoped
public class ComplianceReviewLifecycle {

    private final Function<WorkItemCreateRequest, WorkItem> creator;
    private final AmlLedgerService ledgerService;

    @Inject
    public ComplianceReviewLifecycle(WorkItemService workItemService,
                                     AmlLedgerService ledgerService) {
        this.creator = workItemService::create;
        this.ledgerService = ledgerService;
    }

    // Package-private test constructor
    ComplianceReviewLifecycle(Function<WorkItemCreateRequest, WorkItem> creator,
                              AmlLedgerService ledgerService) {
        this.creator = creator;
        this.ledgerService = ledgerService;
    }

    public String openReview(SuspiciousTransaction transaction, InvestigationSummary summary,
                             UUID caseId) {
        String osintNote = summary.osintScreening() instanceof SpecialistOutcome.Declined<?> d
                ? " OSINT declined: " + d.reason() + "." : "";
        WorkItem workItem = creator.apply(WorkItemCreateRequest.builder()
                .title("Compliance review — SAR for transaction " + transaction.id())
                .description(summary.sarNarrative() + osintNote)
                .category("aml-compliance")
                .priority(WorkItemPriority.HIGH)
                .candidateGroups("compliance-officers")
                .createdBy("aml-system")
                .claimDeadline(Instant.now().plus(30, ChronoUnit.DAYS))
                .callerRef("aml:investigation:" + caseId)
                .build());
        final String taskId = workItem.id.toString();
        ledgerService.writeComplianceReviewOpened(caseId, taskId);
        return taskId;
    }
}
```

- [ ] **Step 4: Run lifecycle tests**

```bash
mvn -pl app -am -Dsurefire.failIfNoSpecifiedTests=false -Dtest=ComplianceReviewLifecycleTest test
```

Expected: both tests PASS.

- [ ] **Step 5: Update `AmlInvestigationCoordinatorTest`**

In `app/src/test/java/io/casehub/aml/AmlInvestigationCoordinatorTest.java`, find and replace both `ComplianceReviewLifecycle` constructions (lines ~52 and ~82) with the 2-arg form:

```java
// Line ~52 (test 1): was new ComplianceReviewLifecycle(req -> workItemWith(expectedId))
new ComplianceReviewLifecycle(req -> workItemWith(expectedId), AmlLedgerService.noOp())

// Line ~82 (test 2): was new ComplianceReviewLifecycle(req -> workItemWith(UUID.randomUUID()))
new ComplianceReviewLifecycle(req -> workItemWith(UUID.randomUUID()), AmlLedgerService.noOp())
```

- [ ] **Step 6: Update `AmlInvestigationCoordinator`**

In `app/src/main/java/io/casehub/aml/AmlInvestigationCoordinator.java`:

1. Change `compliance.openReview(transaction, summary)` → `compliance.openReview(transaction, summary, caseId)`
2. Remove the line `ledgerService.writeComplianceReviewOpened(caseId, taskId);`

The updated `investigate()` method body:

```java
@Override
public AmlInvestigationResult investigate(final SuspiciousTransaction transaction) {
    final UUID caseId = UUID.randomUUID();
    final UUID caseEntryId = ledgerService.writeCaseOpened(transaction, caseId);
    final InvestigationSummary summary = investigator.investigate(transaction, caseId);
    final String taskId = compliance.openReview(transaction, summary, caseId);
    // writeComplianceReviewOpened is now called inside openReview() — removed from here
    return new AmlInvestigationResult(summary, taskId, caseId, caseEntryId);
}
```

- [ ] **Step 7: Run coordinator tests**

```bash
mvn -pl app -am -Dsurefire.failIfNoSpecifiedTests=false -Dtest=AmlInvestigationCoordinatorTest test
```

Expected: both tests PASS.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/ComplianceReviewLifecycle.java
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/AmlInvestigationCoordinator.java
git -C /Users/mdproctor/claude/casehub/aml add app/src/test/java/io/casehub/aml/ComplianceReviewLifecycleTest.java
git -C /Users/mdproctor/claude/casehub/aml add app/src/test/java/io/casehub/aml/AmlInvestigationCoordinatorTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-56): ComplianceReviewLifecycle consolidates WorkItem + ledger entry — closes #56

Both engine and sync paths now always write COMPLIANCE_REVIEW_OPENED.

Closes #56
Refs #44 #55"
```

---

## Task 7: `AmlEngineCoordinator` + Sar-Drafting Workers

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlEngineCoordinator.java`
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseHub.java`

- [ ] **Step 1: Signal `caseId` in `AmlEngineCoordinator.startInvestigation()`**

In `AmlEngineCoordinator.startInvestigation()`, add `caseHub.signal(caseId, "caseId", caseId.toString())` immediately after the `startCase()` call returns and before `writeCaseOpened()`:

```java
caseId = caseHub.startCase(initialContext)
        .toCompletableFuture()
        .get(CASE_START_TIMEOUT_SECONDS, TimeUnit.SECONDS);

// Signal caseId into the blackboard so sar-drafting workers can pass it to openReview().
// Safe: sar-drafting is the last worker in the chain (entity-resolution → pattern/OSINT
// parallel → sar-drafting). Quartz scheduling ensures ample propagation time.
caseHub.signal(caseId, "caseId", caseId.toString());

ledgerService.writeCaseOpened(transaction, caseId);
```

- [ ] **Step 2: Update both sar-drafting workers in `AmlInvestigationCaseHub`**

In `sarDraftingWorkerJunior()`, inside the lambda, add caseId extraction before `complianceReviewLifecycle.openReview()`:

```java
.function((final Map<String, Object> input) -> {
    @SuppressWarnings("unchecked")
    final Map<String, Object> txMap = (Map<String, Object>) input.get("transaction");
    final SuspiciousTransaction tx =
            objectMapper.convertValue(txMap, SuspiciousTransaction.class);
    @SuppressWarnings("unchecked")
    final Map<String, Object> osintMap =
            (Map<String, Object>) input.get("osintScreening");
    final boolean osintDeclined = osintMap != null
            && Boolean.TRUE.equals(osintMap.get("declined"));
    final String sarNarrative = "SAR filed for transaction " + tx.id()
            + ". Amount: " + tx.amount() + " " + tx.currency()
            + (osintDeclined ? " OSINT screening declined." : "");

    // Extract caseId signaled by AmlEngineCoordinator after startCase() returned.
    final String rawCaseId = (String) input.get("caseId");
    if (rawCaseId == null) {
        LOG.errorf("caseId not in context for sar-drafting-junior — signal may not have arrived");
        throw new RuntimeException("caseId missing from case context");
    }
    final UUID caseId = UUID.fromString(rawCaseId);

    final String complianceTaskId =
            complianceReviewLifecycle.openReview(tx, buildSummary(input, tx, sarNarrative), caseId);
    return WorkerResult.of(Map.of("sarNarrative", sarNarrative, "complianceTaskId", complianceTaskId));
})
```

Apply the identical `caseId` extraction + null guard to `sarDraftingWorkerSenior()`.

Also add the import at the top of `AmlInvestigationCaseHub.java`:

```java
import java.util.UUID;
```

And add a logger field (if not already present):

```java
private static final org.jboss.logging.Logger LOG = 
    org.jboss.logging.Logger.getLogger(AmlInvestigationCaseHub.class);
```

- [ ] **Step 3: Compile check**

```bash
mvn -pl app -am compile -q 2>&1 | grep -E "error:" | grep -v "RoutingDecisionRecord" | head -20
```

Expected: only `RoutingDecisionRecord` construction errors remain (Task 12). No errors in engine coordinator or case hub.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/engine/AmlEngineCoordinator.java
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseHub.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-56): signal caseId into blackboard; sar-drafting workers pass caseId to openReview

Refs #44 #55 #56"
```

---

## Task 8: `AmlTrustRoutingObserver` Hardening (PP-20260530-49856c)

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/trust/AmlTrustRoutingObserver.java`

- [ ] **Step 1: Replace `onWorkerDecision()` with hardened version**

Replace the entire `onWorkerDecision()` method:

```java
public void onWorkerDecision(@ObservesAsync WorkerDecisionEvent event) {
    // Pre-try: pure computation only. If policyProvider throws here, the method fails
    // without writing a failure entry — AUDIT GAP log path (threshold not available).
    final double threshold = policyProvider.forCapability(event.capabilityTag()).threshold();
    final Double score = trustScoreCache
            .getCapabilityScore(event.workerId(), event.capabilityTag())
            .stream().boxed().findFirst().orElse(null);
    final UUID attestationSubject = attestationSubjectFor(event.caseId());

    final Object lock = subjectLocks.computeIfAbsent(attestationSubject, k -> new Object());

    boolean attestationWritten = false;
    try {
        final AmlTrustRoutingAttestation entry = new AmlTrustRoutingAttestation();
        entry.id = UUID.randomUUID();
        entry.subjectId = attestationSubject;
        entry.investigationCaseId = event.caseId();
        entry.capabilityTag = event.capabilityTag();
        entry.selectedWorkerId = event.workerId();
        entry.trustScoreAtRouting = score;
        entry.thresholdApplied = threshold;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = ACTOR_ID;
        entry.actorType = ActorType.SYSTEM;
        entry.actorRole = ACTOR_ROLE;
        entry.occurredAt = Instant.now();
        entry.reconstructed = false;
        entry.observerFailed = false;

        synchronized (lock) {
            attestationRepo.saveWithSequence(entry);
        }
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
}
```

Also add a logger field (if not already present):

```java
private static final org.jboss.logging.Logger LOG =
    org.jboss.logging.Logger.getLogger(AmlTrustRoutingObserver.class);
```

And add missing imports:

```java
import io.casehub.ledger.api.model.LedgerEntryType;
import java.time.Instant;
```

- [ ] **Step 2: Run existing attestation test to confirm still passes**

```bash
mvn -pl app -am -Dsurefire.failIfNoSpecifiedTests=false -Dtest=AmlTrustRoutingAttestationTest test
```

Expected: both tests PASS (the happy path still writes attestations correctly).

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/trust/AmlTrustRoutingObserver.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-44): AmlTrustRoutingObserver — double-try/catch hardening per PP-20260530-49856c

Observer failures now write an observerFailed=true attestation entry rather than disappearing.

Refs #44"
```

---

## Task 9: `AmlWorkItemLifecycleObserver` + Unit Tests

**Files:**
- Create: `app/src/main/java/io/casehub/aml/compliance/AmlWorkItemLifecycleObserver.java`
- Create: `app/src/test/java/io/casehub/aml/compliance/AmlWorkItemLifecycleObserverTest.java`

- [ ] **Step 1: Write failing unit tests**

Create `app/src/test/java/io/casehub/aml/compliance/AmlWorkItemLifecycleObserverTest.java`:

```java
package io.casehub.aml.compliance;

import io.casehub.aml.ledger.AmlLedgerService;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.UUID;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class AmlWorkItemLifecycleObserverTest {

    @Mock AmlLedgerService ledgerService;
    AmlWorkItemLifecycleObserver observer;

    UUID caseId = UUID.randomUUID();

    @BeforeEach
    void setUp() {
        observer = new AmlWorkItemLifecycleObserver(ledgerService);
    }

    @Test
    void completed_validCallerRef_writesApproved() {
        observer.onWorkItemLifecycle(event(WorkItemStatus.COMPLETED, "aml:investigation:" + caseId,
                "compliance-officer-001"));

        verify(ledgerService).writeSarOfficerReviewed(eq(caseId), eq("compliance-officer-001"),
                eq("APPROVED"));
        verify(ledgerService, never()).writeSarOfficerReviewedFailure(any(), any());
    }

    @Test
    void rejected_validCallerRef_writesRejected() {
        observer.onWorkItemLifecycle(event(WorkItemStatus.REJECTED, "aml:investigation:" + caseId,
                "compliance-officer-001"));

        verify(ledgerService).writeSarOfficerReviewed(eq(caseId), eq("compliance-officer-001"),
                eq("REJECTED"));
    }

    @Test
    void inProgress_noWrite() {
        observer.onWorkItemLifecycle(event(WorkItemStatus.IN_PROGRESS, "aml:investigation:" + caseId,
                "officer"));

        verify(ledgerService, never()).writeSarOfficerReviewed(any(), any(), any());
        verify(ledgerService, never()).writeSarOfficerReviewedFailure(any(), any());
    }

    @Test
    void nullWorkItem_noWrite() {
        WorkItemLifecycleEvent event = WorkItemLifecycleEvent.of(
                "COMPLETED", null, "officer", null);
        observer.onWorkItemLifecycle(event);

        verify(ledgerService, never()).writeSarOfficerReviewed(any(), any(), any());
    }

    @Test
    void oldFormatCallerRef_noWrite() {
        // Old format used slash: "aml:investigation/{txId}" — hard cutover
        observer.onWorkItemLifecycle(event(WorkItemStatus.COMPLETED,
                "aml:investigation/TXN-001", "officer"));

        verify(ledgerService, never()).writeSarOfficerReviewed(any(), any(), any());
    }

    @Test
    void differentDomain_noWrite() {
        observer.onWorkItemLifecycle(event(WorkItemStatus.COMPLETED,
                "devtown:pr-review:" + UUID.randomUUID(), "actor"));

        verify(ledgerService, never()).writeSarOfficerReviewed(any(), any(), any());
    }

    @Test
    void invalidUuidInCallerRef_noWrite() {
        observer.onWorkItemLifecycle(event(WorkItemStatus.COMPLETED,
                "aml:investigation:not-a-uuid", "officer"));

        verify(ledgerService, never()).writeSarOfficerReviewed(any(), any(), any());
    }

    @Test
    void nullActor_fallsBackToUnknown() {
        observer.onWorkItemLifecycle(event(WorkItemStatus.COMPLETED,
                "aml:investigation:" + caseId, null));

        verify(ledgerService).writeSarOfficerReviewed(eq(caseId), eq("unknown-officer"),
                eq("APPROVED"));
    }

    @Test
    void ledgerWriteFails_writesFailureEntry() {
        doThrow(new RuntimeException("DB error"))
            .when(ledgerService).writeSarOfficerReviewed(any(), any(), any());

        observer.onWorkItemLifecycle(event(WorkItemStatus.COMPLETED,
                "aml:investigation:" + caseId, "officer-X"));

        verify(ledgerService).writeSarOfficerReviewedFailure(eq(caseId), eq("officer-X"));
    }

    // -- Helpers --

    private WorkItemLifecycleEvent event(WorkItemStatus status, String callerRef, String actor) {
        WorkItem wi = new WorkItem();
        wi.id = UUID.randomUUID();
        wi.status = status;
        wi.callerRef = callerRef;
        return WorkItemLifecycleEvent.of(status.name(), wi, actor, null);
    }
}
```

- [ ] **Step 2: Run to confirm tests fail**

```bash
mvn -pl app -am -Dsurefire.failIfNoSpecifiedTests=false -Dtest=AmlWorkItemLifecycleObserverTest test 2>&1 | tail -20
```

Expected: compilation failure — `AmlWorkItemLifecycleObserver` does not exist.

- [ ] **Step 3: Create `AmlWorkItemLifecycleObserver`**

Create `app/src/main/java/io/casehub/aml/compliance/AmlWorkItemLifecycleObserver.java`:

```java
package io.casehub.aml.compliance;

import io.casehub.aml.ledger.AmlLedgerService;
import io.casehub.work.runtime.event.WorkItemLifecycleEvent;
import io.casehub.work.runtime.model.WorkItemStatus;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.UUID;

/**
 * Layer 9: writes an AML_SAR_OFFICER_REVIEWED ledger entry when the compliance officer
 * completes or rejects the SAR review WorkItem.
 *
 * <p>Observes {@link WorkItemLifecycleEvent} asynchronously. Only handles WorkItems whose
 * {@code callerRef} matches the {@code "aml:investigation:<UUID>"} prefix — set by
 * {@link io.casehub.aml.ComplianceReviewLifecycle} on WorkItem creation.
 *
 * <p>Both {@link WorkItemStatus#COMPLETED} (approved) and {@link WorkItemStatus#REJECTED}
 * produce a ledger entry with the officer's actorId. This officer identity is the human PII
 * that GDPR Art.17 erasure acts on.
 *
 * <p>Applies the PP-20260530-49856c double-try/catch pattern: failure writes an
 * observer-failure record via REQUIRES_NEW so it commits independently.
 */
@ApplicationScoped
public class AmlWorkItemLifecycleObserver {

    private static final Logger LOG = Logger.getLogger(AmlWorkItemLifecycleObserver.class);
    private static final String CALLER_REF_PREFIX = "aml:investigation:";

    private final AmlLedgerService ledgerService;

    @Inject
    public AmlWorkItemLifecycleObserver(AmlLedgerService ledgerService) {
        this.ledgerService = ledgerService;
    }

    public void onWorkItemLifecycle(@ObservesAsync WorkItemLifecycleEvent event) {
        // Guard 1: only handle terminal officer decisions
        if (event.status() != WorkItemStatus.COMPLETED
                && event.status() != WorkItemStatus.REJECTED) {
            return;
        }

        // Guard 2: workItem snapshot must be present (null in cross-cluster wire events)
        if (event.workItem() == null) {
            LOG.warnf("WorkItemLifecycleEvent has no WorkItem snapshot — cannot write SAR_OFFICER_REVIEWED");
            return;
        }

        // Guard 3: only handle AML compliance review WorkItems
        final String callerRef = event.workItem().callerRef;
        if (callerRef == null || !callerRef.startsWith(CALLER_REF_PREFIX)) {
            return;
        }

        // Guard 4: parse caseId — invalid UUID means data corruption, log and skip
        final UUID caseId;
        try {
            caseId = UUID.fromString(callerRef.substring(CALLER_REF_PREFIX.length()));
        } catch (IllegalArgumentException e) {
            LOG.warnf("Invalid caseId in callerRef '%s' — skipping SAR_OFFICER_REVIEWED write",
                    callerRef);
            return;
        }

        final String officerId = event.actor() != null ? event.actor() : "unknown-officer";
        final String reviewDecision = event.status() == WorkItemStatus.COMPLETED
                ? "APPROVED" : "REJECTED";

        boolean written = false;
        try {
            ledgerService.writeSarOfficerReviewed(caseId, officerId, reviewDecision);
            written = true;
        } catch (Exception e) {
            LOG.warnf(e, "Failed to write SAR_OFFICER_REVIEWED for caseId=%s officer=%s",
                    caseId, officerId);
            if (!written) {
                try {
                    ledgerService.writeSarOfficerReviewedFailure(caseId, officerId);
                } catch (Exception inner) {
                    LOG.errorf(inner,
                            "AUDIT GAP: SAR_OFFICER_REVIEWED failure entry also failed caseId=%s",
                            caseId);
                }
            }
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
mvn -pl app -am -Dsurefire.failIfNoSpecifiedTests=false -Dtest=AmlWorkItemLifecycleObserverTest test
```

Expected: all 9 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/compliance/AmlWorkItemLifecycleObserver.java
git -C /Users/mdproctor/claude/casehub/aml add app/src/test/java/io/casehub/aml/compliance/AmlWorkItemLifecycleObserverTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-55): AmlWorkItemLifecycleObserver — writes officer-reviewed ledger entry for GDPR erasure

Refs #55"
```

---

## Task 10: `AmlComplianceEvidenceService` Major Update + Unit Test Update

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/compliance/AmlComplianceEvidenceService.java`
- Modify: `app/src/test/java/io/casehub/aml/compliance/AmlComplianceEvidenceServiceTest.java`

This task fixes the `RoutingDecisionRecord` compile errors and adds all officer-review support.

- [ ] **Step 1: Update `AmlComplianceEvidenceService`**

Apply these changes to `app/src/main/java/io/casehub/aml/compliance/AmlComplianceEvidenceService.java`:

**a) Add import:**
```java
import io.casehub.aml.ledger.AmlSarOfficerReviewedLedgerEntry;
import io.casehub.aml.trust.AmlAttestationReconciler;
```

**b) Update constructor to 6-param:**
```java
private final AmlAttestationReconciler reconciler;

@Inject
public AmlComplianceEvidenceService(
        LedgerEntryRepository ledgerRepo,
        LedgerVerificationService verificationService,
        AmlTrustAttestationRepository attestationRepo,
        AmlWorkerDecisionRepository workerDecisionRepo,
        EntityManager em,
        AmlAttestationReconciler reconciler) {
    this.ledgerRepo = ledgerRepo;
    this.verificationService = verificationService;
    this.attestationRepo = attestationRepo;
    this.workerDecisionRepo = workerDecisionRepo;
    this.em = em;
    this.reconciler = reconciler;
}
```

**c) Update `findEvidence()` — add officer review entries and extend empty guard:**
```java
public Optional<ComplianceEvidence> findEvidence(UUID caseId) {
    List<LedgerEntry> all = ledgerRepo.findBySubjectId(caseId);
    List<AmlCaseOpenedLedgerEntry> caseEntries = filterCaseOpened(all);
    List<AmlComplianceReviewLedgerEntry> reviewEntries = filterComplianceReview(all);
    List<AmlSarOfficerReviewedLedgerEntry> officerReviewEntries = filterSarOfficerReviewed(all);
    if (caseEntries.isEmpty() && reviewEntries.isEmpty() && officerReviewEntries.isEmpty())
        return Optional.empty();
    return Optional.of(build(caseId, caseEntries, reviewEntries, officerReviewEntries));
}
```

**d) Update `assembleEvidence()` — add officer review filter:**
```java
ComplianceEvidence assembleEvidence(UUID caseId) {
    List<LedgerEntry> all = ledgerRepo.findBySubjectId(caseId);
    return build(caseId,
        filterCaseOpened(all), filterComplianceReview(all),
        filterSarOfficerReviewed(all));
}
```

**e) Update `build()` — 4-param signature:**
```java
private ComplianceEvidence build(UUID caseId,
        List<AmlCaseOpenedLedgerEntry> caseEntries,
        List<AmlComplianceReviewLedgerEntry> reviewEntries,
        List<AmlSarOfficerReviewedLedgerEntry> officerReviewEntries) {
    return new ComplianceEvidence(
            caseId,
            Instant.now(),
            buildAuditChain(caseId, caseEntries, reviewEntries, officerReviewEntries),
            buildSla(reviewEntries),
            buildTrustRouting(caseId),
            buildGdprErasure(),
            null);
}
```

**f) Update `buildAuditChain()` — 4-param signature + officer entries + extended allLinked:**
```java
private AuditChainRequirement buildAuditChain(UUID caseId,
        List<AmlCaseOpenedLedgerEntry> caseEntries,
        List<AmlComplianceReviewLedgerEntry> reviewEntries,
        List<AmlSarOfficerReviewedLedgerEntry> officerReviewEntries) {
    if (caseEntries.isEmpty() && reviewEntries.isEmpty() && officerReviewEntries.isEmpty()) {
        return new AuditChainRequirement(
                AuditChainRequirement.REQUIREMENT_ID,
                AuditChainRequirement.CITATION,
                AuditChainRequirement.MECHANISM,
                RequirementStatus.GAP, null, false, List.of());
    }

    boolean chainVerified = false;
    String treeRoot = null;
    try {
        chainVerified = verificationService.verify(caseId);
        treeRoot = verificationService.treeRoot(caseId);
    } catch (IllegalStateException ignored) {
        // No Merkle frontier — hash-chain disabled (tests) or not yet built
    }

    List<LedgerEventRecord> events = new ArrayList<>();
    for (AmlCaseOpenedLedgerEntry entry : caseEntries) {
        events.add(new LedgerEventRecord(entry.id, "CASE_OPENED", entry.actorId, entry.actorRole,
                entry.occurredAt, entry.causedByEntryId, entry.digest, buildInclusionProof(entry.id)));
    }
    for (AmlComplianceReviewLedgerEntry entry : reviewEntries) {
        events.add(new LedgerEventRecord(entry.id, "COMPLIANCE_REVIEW_OPENED", entry.actorId,
                entry.actorRole, entry.occurredAt, entry.causedByEntryId, entry.digest,
                buildInclusionProof(entry.id)));
    }
    for (AmlSarOfficerReviewedLedgerEntry entry : officerReviewEntries) {
        events.add(new LedgerEventRecord(entry.id, "SAR_OFFICER_REVIEWED", entry.actorId,
                entry.actorRole, entry.occurredAt, entry.causedByEntryId, entry.digest,
                buildInclusionProof(entry.id)));
    }

    boolean allLinked = reviewEntries.stream().allMatch(e -> e.causedByEntryId != null)
            && officerReviewEntries.stream().allMatch(e -> e.causedByEntryId != null);

    RequirementStatus status;
    if (chainVerified && allLinked && !officerReviewEntries.isEmpty()) {
        status = RequirementStatus.CLOSED;
    } else if (!caseEntries.isEmpty() || !reviewEntries.isEmpty() || !officerReviewEntries.isEmpty()) {
        status = RequirementStatus.PARTIAL;
    } else {
        status = RequirementStatus.GAP;
    }

    return new AuditChainRequirement(
            AuditChainRequirement.REQUIREMENT_ID,
            AuditChainRequirement.CITATION,
            AuditChainRequirement.MECHANISM,
            status, treeRoot, chainVerified, events);
}
```

**g) Update `buildTrustRouting()` — call reconciler + use 7-param `RoutingDecisionRecord`:**
```java
private TrustRoutingRequirement buildTrustRouting(UUID caseId) {
    List<WorkerDecisionEntry> dispatched = workerDecisionRepo.findAllByCaseId(caseId);
    Set<String> dispatchedCaps = dispatched.stream()
            .map(w -> w.capabilityTag)
            .collect(Collectors.toSet());

    List<AmlTrustRoutingAttestation> raw = attestationRepo.findByInvestigationCaseId(caseId);
    List<AmlTrustRoutingAttestation> attestations = reconciler.reconcileIfNeeded(caseId, dispatched, raw);
    Set<String> attestedCaps = attestations.stream()
            .map(a -> a.capabilityTag)
            .collect(Collectors.toSet());

    List<RoutingDecisionRecord> decisions = attestations.stream()
            .map(a -> new RoutingDecisionRecord(
                    a.capabilityTag, a.selectedWorkerId,
                    a.trustScoreAtRouting, a.thresholdApplied, a.id,
                    a.reconstructed, a.observerFailed))
            .toList();

    RequirementStatus status;
    if (dispatchedCaps.isEmpty()) {
        status = RequirementStatus.GAP;
    } else if (attestations.stream().allMatch(a -> !a.observerFailed && !a.reconstructed)
            && attestedCaps.containsAll(dispatchedCaps)) {
        status = RequirementStatus.CLOSED;
    } else {
        status = RequirementStatus.PARTIAL;
    }

    return new TrustRoutingRequirement(
            TrustRoutingRequirement.REQUIREMENT_ID,
            TrustRoutingRequirement.CITATION,
            TrustRoutingRequirement.MECHANISM,
            status, decisions);
}
```

**h) Add `filterSarOfficerReviewed()` helper:**
```java
private List<AmlSarOfficerReviewedLedgerEntry> filterSarOfficerReviewed(List<LedgerEntry> entries) {
    return entries.stream()
            .filter(AmlSarOfficerReviewedLedgerEntry.class::isInstance)
            .map(AmlSarOfficerReviewedLedgerEntry.class::cast)
            .toList();
}
```

- [ ] **Step 2: Update `AmlComplianceEvidenceServiceTest`**

Replace full content of `app/src/test/java/io/casehub/aml/compliance/AmlComplianceEvidenceServiceTest.java`. Key changes:

**a) Add reconciler mock and update setUp:**
```java
@Mock AmlAttestationReconciler mockReconciler;

// In setUp():
service = new AmlComplianceEvidenceService(
    ledgerRepo, verificationService, attestationRepo,
    workerDecisionRepo, em, mockReconciler);

// In each test that calls assembleEvidence (all 5 existing):
// Add this when() stub — reconciler returns the raw list unchanged
List<AmlTrustRoutingAttestation> attestationList = /* the list for that test */;
when(mockReconciler.reconcileIfNeeded(eq(caseId), any(), any())).thenReturn(attestationList);
```

**b) Update `assembleEvidence_happyPath_allRequirementsClosed` — auditChain now PARTIAL (no officer review entry):**
```java
// Before: assertEquals(RequirementStatus.CLOSED, evidence.auditChain().status())
// After — CLOSED requires ≥1 officer review entry; this test has none:
assertEquals(RequirementStatus.PARTIAL, evidence.auditChain().status());
```

**c) Add new test for full CLOSED audit chain with officer review:**
```java
@Test
void assembleEvidence_withOfficerReview_auditChainClosed() {
    var opened = caseOpenedEntry(caseId, caseOpenedId);
    var review = reviewOpenedEntry(caseId, reviewOpenedId, taskId, caseOpenedId);
    var officerReview = officerReviewEntry(caseId, UUID.randomUUID(), reviewOpenedId);
    when(ledgerRepo.findBySubjectId(caseId)).thenReturn(List.of(opened, review, officerReview));
    when(verificationService.verify(caseId)).thenReturn(true);
    when(verificationService.treeRoot(caseId)).thenReturn(treeRoot);
    when(verificationService.inclusionProof(any())).thenReturn(stubProof());
    when(em.find(eq(WorkItem.class), eq(taskId)))
        .thenReturn(workItem(taskId, Instant.now().plus(30, ChronoUnit.DAYS), null));
    when(attestationRepo.findByInvestigationCaseId(caseId)).thenReturn(List.of());
    when(workerDecisionRepo.findAllByCaseId(caseId)).thenReturn(List.of());
    when(mockReconciler.reconcileIfNeeded(eq(caseId), any(), any())).thenReturn(List.of());

    ComplianceEvidence evidence = service.assembleEvidence(caseId);

    assertEquals(RequirementStatus.CLOSED, evidence.auditChain().status());
    assertEquals(3, evidence.auditChain().events().size());
    assertEquals("SAR_OFFICER_REVIEWED", evidence.auditChain().events().get(2).eventType());
    assertEquals("compliance-officer-001", evidence.auditChain().events().get(2).actorId());
}
```

**d) Add new test for observerFailed and reconstructed trust routing status:**
```java
@Test
void assembleEvidence_observerFailedAttestation_trustRoutingPartial() {
    var opened = caseOpenedEntry(caseId, caseOpenedId);
    var review = reviewOpenedEntry(caseId, reviewOpenedId, taskId, caseOpenedId);
    when(ledgerRepo.findBySubjectId(caseId)).thenReturn(List.of(opened, review));
    when(verificationService.verify(caseId)).thenReturn(false);
    when(verificationService.treeRoot(caseId)).thenReturn(treeRoot);
    when(verificationService.inclusionProof(any())).thenReturn(stubProof());
    when(em.find(eq(WorkItem.class), eq(taskId)))
        .thenReturn(workItem(taskId, Instant.now().plus(30, ChronoUnit.DAYS), null));
    var failureAtt = attestation(caseId, "entity-resolution", "agent-A", 0.8, 0.70);
    failureAtt.observerFailed = true;
    when(mockReconciler.reconcileIfNeeded(eq(caseId), any(), any())).thenReturn(List.of(failureAtt));
    when(workerDecisionRepo.findAllByCaseId(caseId)).thenReturn(List.of(
        workerDecision(caseId, "entity-resolution", "agent-A")));

    ComplianceEvidence evidence = service.assembleEvidence(caseId);

    assertEquals(RequirementStatus.PARTIAL, evidence.trustRouting().status());
    assertTrue(evidence.trustRouting().decisions().get(0).observerFailed());
}

@Test
void assembleEvidence_reconstructedAttestation_trustRoutingPartial() {
    var opened = caseOpenedEntry(caseId, caseOpenedId);
    when(ledgerRepo.findBySubjectId(caseId)).thenReturn(List.of(opened));
    when(verificationService.verify(caseId)).thenReturn(false);
    when(verificationService.treeRoot(caseId)).thenReturn(treeRoot);
    when(verificationService.inclusionProof(any())).thenReturn(stubProof());
    when(em.find(eq(WorkItem.class), any())).thenReturn(null);
    var recon = attestation(caseId, "entity-resolution", "agent-A", 0.8, 0.70);
    recon.reconstructed = true;
    when(mockReconciler.reconcileIfNeeded(eq(caseId), any(), any())).thenReturn(List.of(recon));
    when(workerDecisionRepo.findAllByCaseId(caseId)).thenReturn(List.of(
        workerDecision(caseId, "entity-resolution", "agent-A")));

    ComplianceEvidence evidence = service.assembleEvidence(caseId);

    assertEquals(RequirementStatus.PARTIAL, evidence.trustRouting().status());
    assertTrue(evidence.trustRouting().decisions().get(0).reconstructed());
}
```

**e) Add `officerReviewEntry()` helper method:**
```java
private AmlSarOfficerReviewedLedgerEntry officerReviewEntry(
        UUID caseId, UUID entryId, UUID causedBy) {
    AmlSarOfficerReviewedLedgerEntry e = new AmlSarOfficerReviewedLedgerEntry();
    e.id = entryId;
    e.subjectId = caseId;
    e.sequenceNumber = 3;
    e.actorId = "compliance-officer-001";
    e.actorRole = "ComplianceOfficer";
    e.actorType = ActorType.HUMAN;
    e.occurredAt = Instant.now();
    e.causedByEntryId = causedBy;
    e.reviewDecision = "APPROVED";
    e.entryType = LedgerEntryType.EVENT;
    return e;
}
```

- [ ] **Step 3: Run all unit tests**

```bash
mvn -pl app -am -Dsurefire.failIfNoSpecifiedTests=false -Dtest=AmlComplianceEvidenceServiceTest test
```

Expected: all tests PASS (including the updated happy-path assertion).

- [ ] **Step 4: Run full compile check**

```bash
mvn -pl app -am compile -q
```

Expected: NO errors. All `RoutingDecisionRecord` call sites are now fixed.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/compliance/AmlComplianceEvidenceService.java
git -C /Users/mdproctor/claude/casehub/aml add app/src/test/java/io/casehub/aml/compliance/AmlComplianceEvidenceServiceTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-44,55): AmlComplianceEvidenceService — reconciler, SAR_OFFICER_REVIEWED, updated status logic

Refs #44 #55"
```

---

## Task 11: @QuarkusTest — Update Existing + Add GDPR + Reconciliation Tests

**Files:**
- Modify: `app/src/test/java/io/casehub/aml/compliance/AmlLayer7ResourceTest.java`

- [ ] **Step 1: Update existing `getComplianceEvidence_afterInvestigation_returnsAllRequirements` test**

The existing test expects `sla.status = GAP` and `auditChain.events` with size ≥ 1. After our changes:
- `sla.status` → `PARTIAL` (WorkItem now exists; officer hasn't acted)
- `auditChain.events` → size ≥ 2 (CASE_OPENED + COMPLIANCE_REVIEW_OPENED)
- The Javadoc comment about the engine path not writing COMPLIANCE_REVIEW_OPENED must be removed/updated

Update the relevant assertions:
```java
// Replace: .body("sla.status", equalTo("GAP"))
// With:
.body("sla.status", anyOf(equalTo("PARTIAL"), equalTo("CLOSED"), equalTo("BREACHED")))
.body("sla.workItemId", notNullValue())  // WorkItem now linked via ledger entry

// Replace: .body("auditChain.events", hasSize(greaterThanOrEqualTo(1)))
// With:
.body("auditChain.events", hasSize(greaterThanOrEqualTo(2)))
.body("auditChain.events[0].eventType", equalTo("CASE_OPENED"))
.body("auditChain.events[1].eventType", equalTo("COMPLIANCE_REVIEW_OPENED"))
```

Also update the class Javadoc to remove the outdated "engine path does not write COMPLIANCE_REVIEW_OPENED" note.

- [ ] **Step 2: Add imports needed for new tests**

Add to the imports in `AmlLayer7ResourceTest.java`:

```java
import io.casehub.work.runtime.service.WorkItemService;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.aml.trust.AmlTrustAttestationRepository;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import jakarta.transaction.Transactional;
import org.awaitility.Awaitility;
import java.util.concurrent.TimeUnit;
```

Add injections to the test class:
```java
@Inject WorkItemService workItemService;

@PersistenceContext(unitName = "qhorus")
EntityManager qhorusEm;
```

- [ ] **Step 3: Add GDPR integration test**

```java
@Test
void gdprDemoFlow_officerReview_erasure() {
    // Start investigation
    String caseId = given().contentType(ContentType.JSON)
        .body(pepTransaction("TXN-GDPR-" + UUID.randomUUID()))
        .when().post("/api/layer6/investigations")
        .then().statusCode(202)
        .extract().path("caseId");

    UUID caseUUID = UUID.fromString(caseId);

    // Wait for sar-drafting to complete
    Awaitility.await().atMost(30, TimeUnit.SECONDS).pollInterval(500, TimeUnit.MILLISECONDS)
        .until(() -> attestationRepo.findByInvestigationCaseId(caseUUID).stream()
            .anyMatch(a -> "sar-drafting".equals(a.capabilityTag)));

    // Drain to completed
    Awaitility.await().atMost(20, TimeUnit.SECONDS).pollInterval(200, TimeUnit.MILLISECONDS)
        .until(() -> "completed".equals(
            given().when().get("/api/layer6/investigations/" + caseId)
                .then().extract().path("status")));

    // Verify sla.workItemId is present (COMPLIANCE_REVIEW_OPENED now written on engine path)
    UUID taskId = UUID.fromString(
        given().when().get("/api/investigations/{caseId}/compliance-evidence", caseId)
            .then().statusCode(200)
            .body("sla.workItemId", notNullValue())
            .extract().path("sla.workItemId"));

    // Complete the WorkItem as compliance officer
    // WorkItem lifecycle: PENDING → claim → ASSIGNED → start → IN_PROGRESS → complete → COMPLETED
    workItemService.claim(taskId, "compliance-officer-001");
    workItemService.start(taskId, "compliance-officer-001");
    workItemService.complete(taskId, "compliance-officer-001", "SAR approved", "APPROVED");
    // 4-param: id, actorId, resolution, outcome

    // Await @ObservesAsync delivery — poll until SAR_OFFICER_REVIEWED appears in audit chain
    Awaitility.await().atMost(5, TimeUnit.SECONDS).pollInterval(200, TimeUnit.MILLISECONDS)
        .until(() -> {
            io.restassured.response.Response r = given().when()
                .get("/api/investigations/{caseId}/compliance-evidence", caseId);
            if (r.statusCode() != 200) return false;
            java.util.List<?> events = r.path("auditChain.events");
            return events != null && events.stream().anyMatch(e ->
                "SAR_OFFICER_REVIEWED".equals(((java.util.Map<?, ?>) e).get("eventType")));
        });

    // Assert officer review event fields
    given().when().get("/api/investigations/{caseId}/compliance-evidence", caseId)
        .then().statusCode(200)
        // auditChain: CLOSED requires chainVerified=true — disabled in H2; accept PARTIAL too
        .body("auditChain.status", anyOf(equalTo("CLOSED"), equalTo("PARTIAL")))
        .body("auditChain.events.eventType", hasItem("SAR_OFFICER_REVIEWED"))
        .body("auditChain.events.find { it.eventType == 'SAR_OFFICER_REVIEWED' }.actorId",
              equalTo("compliance-officer-001"))
        .body("auditChain.events.find { it.eventType == 'SAR_OFFICER_REVIEWED' }.actorRole",
              equalTo("ComplianceOfficer"));

    // GDPR erasure: erase the officer's actorId from the ledger
    given().when().post("/api/actors/compliance-officer-001/erasure")
        .then().statusCode(200);

    // Verify the officer's actorId is pseudonymized in the next evidence read
    given().when().get("/api/investigations/{caseId}/compliance-evidence", caseId)
        .then().statusCode(200)
        .body("auditChain.events.find { it.eventType == 'SAR_OFFICER_REVIEWED' }.actorId",
              not(equalTo("compliance-officer-001")));
}
```

- [ ] **Step 4: Add reconciliation path test**

```java
@Test
void reconciliationPath_deletedAttestation_rebuiltOnRead() {
    // Start investigation and drain
    String caseId = given().contentType(ContentType.JSON)
        .body(pepTransaction("TXN-RECON-" + UUID.randomUUID()))
        .when().post("/api/layer6/investigations")
        .then().statusCode(202)
        .extract().path("caseId");

    UUID caseUUID = UUID.fromString(caseId);

    Awaitility.await().atMost(30, TimeUnit.SECONDS).pollInterval(500, TimeUnit.MILLISECONDS)
        .until(() -> attestationRepo.findByInvestigationCaseId(caseUUID).stream()
            .anyMatch(a -> "sar-drafting".equals(a.capabilityTag)));
    Awaitility.await().atMost(20, TimeUnit.SECONDS).pollInterval(200, TimeUnit.MILLISECONDS)
        .until(() -> "completed".equals(
            given().when().get("/api/layer6/investigations/" + caseId)
                .then().extract().path("status")));

    // Verify baseline: trustRouting.status = CLOSED
    given().when().get("/api/investigations/{caseId}/compliance-evidence", caseId)
        .then().statusCode(200)
        .body("trustRouting.status", equalTo("CLOSED"));

    // Simulate observer failure: delete one attestation row
    UUID deletedAttId = attestationRepo.findByInvestigationCaseId(caseUUID)
        .stream().findFirst().orElseThrow().id;
    deleteAttestation(deletedAttId);

    // First read after deletion: reconciler fills the gap
    given().when().get("/api/investigations/{caseId}/compliance-evidence", caseId)
        .then().statusCode(200)
        .body("trustRouting.status", equalTo("PARTIAL"))
        .body("trustRouting.decisions.find { it.reconstructed == true }.reconstructed", is(true));

    // Second read: no duplicate — reconstructed entry already in DB
    long countBefore = attestationRepo.findByInvestigationCaseId(caseUUID).stream()
        .filter(a -> a.reconstructed).count();
    given().when().get("/api/investigations/{caseId}/compliance-evidence", caseId);
    long countAfter = attestationRepo.findByInvestigationCaseId(caseUUID).stream()
        .filter(a -> a.reconstructed).count();
    assertEquals(countBefore, countAfter, "No duplicate reconstructed entries on second call");
}

@Transactional
void deleteAttestation(UUID id) {
    qhorusEm.createQuery("DELETE FROM AmlTrustRoutingAttestation a WHERE a.id = :id")
        .setParameter("id", id)
        .executeUpdate();
    qhorusEm.clear();
}
```

- [ ] **Step 5: Run all tests**

```bash
mvn -pl app -am test
```

Expected: all tests pass. The `AmlTrustRoutingAttestationTest` tests may also need the reconciler context — those are @QuarkusTest and the reconciler is CDI-discovered; no changes needed there.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/test/java/io/casehub/aml/compliance/AmlLayer7ResourceTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "test(issue-44,55): GDPR integration test + reconciliation path test + update existing Layer7 test

Refs #44 #55"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| V2009: `reconstructed`, `observer_failed` columns + partial unique index | Task 1 |
| V2010: `aml_sar_officer_reviewed_ledger_entry` | Task 1 |
| `AmlTrustRoutingAttestation` entity fields | Task 2 |
| `AmlSarOfficerReviewedLedgerEntry` entity | Task 2 |
| `RoutingDecisionRecord` 7-param update | Task 2 |
| `AmlLedgerService.writeSarOfficerReviewed()` + `writeSarOfficerReviewedFailure()` | Task 3 |
| `AmlTrustAttestationRepository.saveObserverFailureEntry()` | Task 4 |
| `AmlAttestationReconciler` with `PersistenceException` catch | Task 5 |
| `AmlTrustRoutingObserver` double-try/catch hardening | Task 8 |
| `ComplianceReviewLifecycle` caseId param + consolidation (closes #56) | Task 6 |
| `AmlInvestigationCoordinator` updated call + removed redundant line | Task 6 |
| `AmlEngineCoordinator.startInvestigation()` — `caseHub.signal()` | Task 7 |
| Sar-drafting workers — read caseId + null guard | Task 7 |
| `AmlWorkItemLifecycleObserver` — COMPLETED + REJECTED + hardening | Task 9 |
| `AmlComplianceEvidenceService` — 6-param constructor, all method updates | Task 10 |
| `AmlComplianceEvidenceServiceTest` — reconciler mock, updated assertions | Task 10 |
| `ComplianceReviewLifecycleTest` — mock ledger + verify | Task 6 |
| `AmlInvestigationCoordinatorTest` — 2-arg ComplianceReviewLifecycle | Task 6 |
| GDPR integration test | Task 11 |
| Reconciliation path test | Task 11 |
| Update existing Layer7 test (sla PARTIAL, audit 2 events) | Task 11 |

All spec requirements covered. ✅

**Type consistency:**
- `attestationSubjectFor()` is referenced in both `AmlAttestationReconciler` and `AmlTrustRoutingAttestationTest` helper — method is package-private on `AmlTrustRoutingObserver`; both are in `io.casehub.aml.trust` ✅
- `WorkerDecisionEntry` import: `io.casehub.ledger.model.WorkerDecisionEntry` ✅
- `AmlSarOfficerReviewedLedgerEntry` in `io.casehub.aml.ledger` — auto-discovered by qhorus PU ✅
- `AmlWorkItemLifecycleObserver` in `io.casehub.aml.compliance` — CDI-discovered ✅

**Placeholder scan:** No TBDs or TODOs. All code blocks are complete. ✅
