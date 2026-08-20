# Trust Attestation for TRIAGE_OVERRULED Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #128 — Trust attestation for investigation-closed-no-sar outcome (TRIAGE_OVERRULED)
**Issue group:** #128

**Goal:** Write a `LedgerAttestation` with `CHALLENGED` verdict against the triage worker's `WorkerDecisionEntry` when a case reaches `investigation-closed-no-sar`, providing trust-routing feedback that the triage evaluator's SAR_WARRANTED decision was overruled.

**Architecture:** Extend `AmlTrustRoutingObserver` to implement `CaseOutcomeObserver`. The new `onOutcome()` method guards on `outcomeLabel == "investigation-closed-no-sar"`, finds the triage worker's `WorkerDecisionEntry`, and writes a `LedgerAttestation` via the qhorus `EntityManager` in a `REQUIRES_NEW` transaction. Follows the `SarOutcomeFeedbackService` attestation pattern.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine-api (`CaseOutcomeEvent`, `CaseOutcomeObserver`), casehub-ledger (`LedgerAttestation`, `AttestationVerdict`)

## Global Constraints

- No new dependencies — all types already on classpath
- No Flyway migrations — uses existing `LedgerAttestation` entity
- `CaseOutcomeObserver` is an engine SPI — implementations are discovered via CDI
- `QuarkusTransaction.requiringNew()` isolates attestation write from engine's completion transaction
- Test inserts for `WorkerDecisionEntry` follow the exact pattern in `SarOutcomeFeedbackServiceTest`
- `HIGH_RISK_JURISDICTION` is required for SAR path tests (GE-20260726-00e4df)
- CBR store must be cleared in `@BeforeEach` (GE-20260716-986cd1)

---

## Batch 1: TRIAGE_OVERRULED Attestation

### Task 1: Observer extension with happy-path test

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/trust/AmlTrustRoutingObserver.java`
- Create: `app/src/test/java/io/casehub/aml/trust/TriageOverruledAttestationTest.java`

**Interfaces:**
- Consumes: `AmlWorkerDecisionRepository.findLatestByCaseIdAndCapability(UUID, String)` (existing), `CaseOutcomeEvent` record (engine-api), `LedgerAttestation` entity (ledger-runtime)
- Produces: `LedgerAttestation` persisted in qhorus datasource with `verdict=CHALLENGED`, `trustDimension="investigation-accuracy"`, `capabilityTag="investigation-triage"`

- [ ] **Step 1: Write the failing test — escalation NO_SAR writes CHALLENGED attestation**

Create `TriageOverruledAttestationTest.java`. This test constructs a `CaseOutcomeEvent` directly (no full engine flow) with `outcomeLabel="investigation-closed-no-sar"`, inserts a `WorkerDecisionEntry` for the `investigation-triage` capability, calls `onOutcome()`, and verifies the `LedgerAttestation`.

```java
package io.casehub.aml.trust;

import io.casehub.api.spi.CaseOutcomeEvent;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.platform.api.identity.TenancyConstants;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class TriageOverruledAttestationTest {

    @Inject AmlTrustRoutingObserver observer;

    @PersistenceContext(unitName = "qhorus")
    EntityManager em;

    @Test
    void escalation_no_sar_writes_challenged_attestation() {
        final UUID caseId = UUID.randomUUID();
        final UUID entryId = insertTriageWorkerDecisionEntry(caseId);

        final CaseOutcomeEvent event = new CaseOutcomeEvent(
                "aml-investigation",
                TenancyConstants.DEFAULT_TENANT_ID,
                caseId,
                Map.of("rejectionEscalation", Map.of("decision", "NO_SAR"),
                       "investigationTriage", Map.of("decision", "SAR_WARRANTED")),
                "investigation-closed-no-sar",
                Instant.now(),
                Map.of());

        observer.onOutcome(event);

        final List<LedgerAttestation> attestations = QuarkusTransaction.requiringNew().call(() ->
                em.createQuery(
                        "SELECT a FROM LedgerAttestation a WHERE a.subjectId = :sid AND a.trustDimension = :dim",
                        LedgerAttestation.class)
                    .setParameter("sid", caseId)
                    .setParameter("dim", "investigation-accuracy")
                    .getResultList());

        assertEquals(1, attestations.size());
        final LedgerAttestation a = attestations.get(0);
        assertEquals(AttestationVerdict.CHALLENGED, a.verdict);
        assertEquals("investigation-triage", a.capabilityTag);
        assertEquals("investigation-accuracy", a.trustDimension);
        assertEquals(0.2, a.dimensionScore, 0.001);
        assertEquals(1.0, a.confidence, 0.001);
        assertEquals("aml-orchestrator", a.attestorId);
        assertEquals(ActorType.SYSTEM, a.attestorType);
        assertEquals("TriageOutcomeFeedback", a.attestorRole);
        assertEquals(entryId, a.ledgerEntryId);
        assertTrue(a.evidence.contains("TRIAGE_OVERRULED"));
        assertTrue(a.evidence.contains("ESCALATION_NO_SAR"));
    }

    private UUID insertTriageWorkerDecisionEntry(UUID caseId) {
        return QuarkusTransaction.requiringNew().call(() -> {
            final UUID entryId = UUID.randomUUID();
            em.createNativeQuery(
                    "INSERT INTO ledger_entry (id, dtype, subject_id, sequence_number, entry_type, actor_id, actor_type, occurred_at, tenancy_id)" +
                    " VALUES (:id, 'WORKER_DECISION', :sid, 1, 'EVENT', :wid, 'SYSTEM', CURRENT_TIMESTAMP, :tid)")
                .setParameter("id", entryId)
                .setParameter("sid", caseId)
                .setParameter("wid", "investigation-triage-agent")
                .setParameter("tid", TenancyConstants.DEFAULT_TENANT_ID)
                .executeUpdate();
            em.createNativeQuery(
                    "INSERT INTO worker_decision_entry (id, worker_id, capability_tag, case_id)" +
                    " VALUES (:id, :wid, :cap, :cid)")
                .setParameter("id", entryId)
                .setParameter("wid", "investigation-triage-agent")
                .setParameter("cap", "investigation-triage")
                .setParameter("cid", caseId)
                .executeUpdate();
            return entryId;
        });
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl app -am -Dtest=TriageOverruledAttestationTest#escalation_no_sar_writes_challenged_attestation -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `AmlTrustRoutingObserver` does not implement `CaseOutcomeObserver`, so `onOutcome()` does not exist.

- [ ] **Step 3: Implement `onOutcome` and `writeTriageOverruledAttestation` in `AmlTrustRoutingObserver`**

Add `implements CaseOutcomeObserver` to the class declaration. Add new inject fields and the two new methods. The existing `onWorkerDecision` method and all other code remains unchanged.

Add these imports:
```java
import io.casehub.api.spi.CaseOutcomeEvent;
import io.casehub.api.spi.CaseOutcomeObserver;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.quarkus.narayana.jta.QuarkusTransaction;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import java.util.Map;
```

Add these fields after the existing `@Inject` fields:
```java
@Inject AmlWorkerDecisionRepository workerDecisionRepo;
@PersistenceContext(unitName = "qhorus") EntityManager em;
```

Change class declaration to:
```java
public class AmlTrustRoutingObserver implements CaseOutcomeObserver {
```

Add these methods after the existing `attestationSubjectFor` method:
```java
@Override
public void onOutcome(CaseOutcomeEvent event) {
    if (!"aml-investigation".equals(event.caseType())) return;
    if (!"investigation-closed-no-sar".equals(event.outcomeLabel())) return;

    try {
        writeTriageOverruledAttestation(event);
    } catch (Exception e) {
        LOG.warnf(e, "Triage overruled attestation failed caseId=%s", event.caseId());
    }
}

@SuppressWarnings("unchecked")
private void writeTriageOverruledAttestation(CaseOutcomeEvent event) {
    var triageEntry = workerDecisionRepo.findLatestByCaseIdAndCapability(
            event.caseId(), "investigation-triage");
    if (triageEntry.isEmpty()) {
        LOG.warnf("No WorkerDecisionEntry for investigation-triage caseId=%s — skipping overruled attestation",
                event.caseId());
        return;
    }

    String overruleSource = determineOverruleSource(event.caseFileSnapshot());

    QuarkusTransaction.requiringNew().run(() -> {
        var attestation = new LedgerAttestation();
        attestation.id = UUID.randomUUID();
        attestation.ledgerEntryId = triageEntry.get().id;
        attestation.subjectId = event.caseId();
        attestation.attestorId = ACTOR_ID;
        attestation.attestorType = ActorType.SYSTEM;
        attestation.attestorRole = "TriageOutcomeFeedback";
        attestation.verdict = AttestationVerdict.CHALLENGED;
        attestation.capabilityTag = "investigation-triage";
        attestation.trustDimension = "investigation-accuracy";
        attestation.dimensionScore = 0.2;
        attestation.confidence = 1.0;
        attestation.occurredAt = event.closedAt() != null ? event.closedAt() : Instant.now();
        attestation.evidence = "TRIAGE_OVERRULED: originalDecision=SAR_WARRANTED, overruleSource=" + overruleSource;
        em.persist(attestation);
    });
}

@SuppressWarnings("unchecked")
private String determineOverruleSource(Map<String, Object> snapshot) {
    if (snapshot.get("rejectionEscalation") instanceof Map<?, ?> esc
            && "NO_SAR".equals(esc.get("decision"))) {
        return "ESCALATION_NO_SAR";
    }
    if (snapshot.get("postRejectionTriage") instanceof Map<?, ?> triage
            && "FALSE_POSITIVE".equals(triage.get("decision"))) {
        return "RE_TRIAGE_DROP";
    }
    return "UNKNOWN";
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl app -am -Dtest=TriageOverruledAttestationTest#escalation_no_sar_writes_challenged_attestation -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/aml/trust/AmlTrustRoutingObserver.java app/src/test/java/io/casehub/aml/trust/TriageOverruledAttestationTest.java
git commit -m "feat(#128): TRIAGE_OVERRULED attestation — observer + happy path test

Extend AmlTrustRoutingObserver with CaseOutcomeObserver to write
LedgerAttestation(CHALLENGED) against investigation-triage worker
when case reaches investigation-closed-no-sar.

Refs #128"
```

---

### Task 2: Guard conditions, evidence format, and negative tests

**Files:**
- Modify: `app/src/test/java/io/casehub/aml/trust/TriageOverruledAttestationTest.java`

**Interfaces:**
- Consumes: `AmlTrustRoutingObserver.onOutcome(CaseOutcomeEvent)` from Task 1
- Produces: Test coverage for guard conditions and evidence format variants

- [ ] **Step 1: Write tests for guard conditions and evidence format**

Add these test methods to `TriageOverruledAttestationTest`:

```java
@Test
void re_triage_drop_writes_attestation_with_re_triage_evidence() {
    final UUID caseId = UUID.randomUUID();
    insertTriageWorkerDecisionEntry(caseId);

    final CaseOutcomeEvent event = new CaseOutcomeEvent(
            "aml-investigation",
            TenancyConstants.DEFAULT_TENANT_ID,
            caseId,
            Map.of("postRejectionTriage", Map.of("decision", "FALSE_POSITIVE"),
                   "actionGateRejected", Map.of("actionType", "sar.filing"),
                   "investigationTriage", Map.of("decision", "SAR_WARRANTED")),
            "investigation-closed-no-sar",
            Instant.now(),
            Map.of());

    observer.onOutcome(event);

    final List<LedgerAttestation> attestations = QuarkusTransaction.requiringNew().call(() ->
            em.createQuery(
                    "SELECT a FROM LedgerAttestation a WHERE a.subjectId = :sid AND a.trustDimension = :dim",
                    LedgerAttestation.class)
                .setParameter("sid", caseId)
                .setParameter("dim", "investigation-accuracy")
                .getResultList());

    assertEquals(1, attestations.size());
    assertEquals(AttestationVerdict.CHALLENGED, attestations.get(0).verdict);
    assertTrue(attestations.get(0).evidence.contains("RE_TRIAGE_DROP"));
}

@Test
void investigation_complete_outcome_does_not_write_attestation() {
    final UUID caseId = UUID.randomUUID();
    insertTriageWorkerDecisionEntry(caseId);

    final CaseOutcomeEvent event = new CaseOutcomeEvent(
            "aml-investigation",
            TenancyConstants.DEFAULT_TENANT_ID,
            caseId,
            Map.of("investigationTriage", Map.of("decision", "SAR_WARRANTED")),
            "investigation-complete",
            Instant.now(),
            Map.of());

    observer.onOutcome(event);

    final List<LedgerAttestation> attestations = QuarkusTransaction.requiringNew().call(() ->
            em.createQuery(
                    "SELECT a FROM LedgerAttestation a WHERE a.subjectId = :sid AND a.trustDimension = :dim",
                    LedgerAttestation.class)
                .setParameter("sid", caseId)
                .setParameter("dim", "investigation-accuracy")
                .getResultList());

    assertEquals(0, attestations.size());
}

@Test
void investigation_cleared_outcome_does_not_write_attestation() {
    final UUID caseId = UUID.randomUUID();
    insertTriageWorkerDecisionEntry(caseId);

    final CaseOutcomeEvent event = new CaseOutcomeEvent(
            "aml-investigation",
            TenancyConstants.DEFAULT_TENANT_ID,
            caseId,
            Map.of("investigationTriage", Map.of("decision", "FALSE_POSITIVE")),
            "investigation-cleared",
            Instant.now(),
            Map.of());

    observer.onOutcome(event);

    final List<LedgerAttestation> attestations = QuarkusTransaction.requiringNew().call(() ->
            em.createQuery(
                    "SELECT a FROM LedgerAttestation a WHERE a.subjectId = :sid AND a.trustDimension = :dim",
                    LedgerAttestation.class)
                .setParameter("sid", caseId)
                .setParameter("dim", "investigation-accuracy")
                .getResultList());

    assertEquals(0, attestations.size());
}

@Test
void non_aml_case_type_does_not_write_attestation() {
    final UUID caseId = UUID.randomUUID();
    insertTriageWorkerDecisionEntry(caseId);

    final CaseOutcomeEvent event = new CaseOutcomeEvent(
            "other-case-type",
            TenancyConstants.DEFAULT_TENANT_ID,
            caseId,
            Map.of(),
            "investigation-closed-no-sar",
            Instant.now(),
            Map.of());

    observer.onOutcome(event);

    final List<LedgerAttestation> attestations = QuarkusTransaction.requiringNew().call(() ->
            em.createQuery(
                    "SELECT a FROM LedgerAttestation a WHERE a.subjectId = :sid AND a.trustDimension = :dim",
                    LedgerAttestation.class)
                .setParameter("sid", caseId)
                .setParameter("dim", "investigation-accuracy")
                .getResultList());

    assertEquals(0, attestations.size());
}

@Test
void missing_triage_entry_does_not_throw() {
    final UUID caseId = UUID.randomUUID();
    // No WorkerDecisionEntry inserted

    final CaseOutcomeEvent event = new CaseOutcomeEvent(
            "aml-investigation",
            TenancyConstants.DEFAULT_TENANT_ID,
            caseId,
            Map.of("rejectionEscalation", Map.of("decision", "NO_SAR")),
            "investigation-closed-no-sar",
            Instant.now(),
            Map.of());

    assertDoesNotThrow(() -> observer.onOutcome(event));
}
```

- [ ] **Step 2: Run all tests to verify they pass**

Run: `mvn test -pl app -am -Dtest=TriageOverruledAttestationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All 6 tests PASS

- [ ] **Step 3: Run the full test suite to check for regressions**

Run: `mvn test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS — no regressions

- [ ] **Step 4: Commit**

```bash
git add app/src/test/java/io/casehub/aml/trust/TriageOverruledAttestationTest.java
git commit -m "test(#128): guard conditions, evidence format, and negative tests

- RE_TRIAGE_DROP evidence path
- investigation-complete → no attestation
- investigation-cleared → no attestation
- non-AML case type → no attestation
- missing triage entry → graceful no-op

Refs #128"
```

## References

- `2026-08-19-trust-attestation-triage-overruled-design.md` — design spec this plan implements
- `AmlTrustRoutingObserver.java:38-114` — existing observer, target for extension
- `SarOutcomeFeedbackService.java:70-88` — `LedgerAttestation` write pattern
- `AmlCaseProfileStoreObserver.java:72-78` — `CaseOutcomeObserver` implementation pattern
- `SarOutcomeFeedbackServiceTest.java:164-187` — `WorkerDecisionEntry` test insert pattern
- `SarFilingRejectionRoutingTest.java:64-89` — full integration test for escalation NO_SAR path
- `AmlWorkerDecisionRepository.java:28-39` — `findLatestByCaseIdAndCapability` query
- GE-20260628-dbc656 — gate approval before attestation wait ordering
- GE-20260726-00e4df — HIGH_RISK_JURISDICTION flag reason for SAR path tests
- GE-20260716-986cd1 — CBR store isolation in tests
- casehubio/aml#128 — focal issue
