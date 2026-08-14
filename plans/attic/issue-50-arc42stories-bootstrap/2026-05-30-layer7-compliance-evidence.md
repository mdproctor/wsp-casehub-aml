# Layer 7 — Compliance Evidence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Expose a `GET /api/investigations/{caseId}/compliance-evidence` endpoint that returns requirement-scoped evidence with Merkle inclusion proofs for each FinCEN/FATF accountability property delivered by Layers 1–6.

**Architecture:** Four requirement records (audit chain, SLA, trust routing, GDPR) nested inside a `ComplianceEvidence` root. A new `AmlTrustRoutingAttestation` ledger entity captures trust scores at routing time via a CDI observer. A prerequisite fix wires `causedByEntryId` on `COMPLIANCE_REVIEW_OPENED` entries by querying for `CASE_OPENED` at write time.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/H2 (tests), casehub-ledger (`LedgerVerificationService`, `LedgerErasureService`), casehub-engine-ledger (`WorkerDecisionEntry`, `TrustScoreCache`), casehub-work (`WorkItemService`)

---

## Reference

**Spec:** `specs/2026-05-30-layer7-compliance-evidence-design.md`  
**Branch:** `issue-43-layer7-comparison`  
**Issues:** casehubio/aml#43 (this work), #44 (observer reconciliation), casehubio/engine#403, casehubio/work#241  
**Run all tests:** `mvn verify -pl api,app -am -Dsurefire.failIfNoSpecifiedTests=false`  
**Run one test class:** `mvn verify -pl app -am -Dtest=ClassName -Dsurefire.failIfNoSpecifiedTests=false`

---

## File Map

**New in `api/src/main/java/io/casehub/aml/compliance/`:**
- `ComplianceEvidence.java` — root response record
- `RequirementStatus.java` — enum: CLOSED, PARTIAL, BREACHED, GAP
- `AuditChainRequirement.java` — 31 CFR §1020.320(a) + FATF R.16
- `LedgerEventRecord.java` — one per ledger entry in the chain
- `AmlInclusionProof.java` — projection of ledger InclusionProof
- `AmlProofStep.java` — projection of ledger ProofStep
- `SlaRequirement.java` — 31 CFR §1020.320(b)(3)
- `TrustRoutingRequirement.java` — FATF R.20
- `RoutingDecisionRecord.java` — one per worker dispatch
- `GdprErasureRequirement.java` — GDPR Art.17 / FATF privacy

**New in `app/src/main/java/io/casehub/aml/trust/`:**
- `AmlTrustRoutingAttestation.java` — LedgerEntry subclass
- `AmlTrustAttestationRepository.java` — JPQL queries by investigationCaseId
- `AmlTrustRoutingObserver.java` — CDI observer for WorkerDecisionEvent

**New in `app/src/main/java/io/casehub/aml/compliance/`:**
- `AmlComplianceEvidenceService.java` — assembles ComplianceEvidence
- `AmlLayer7Resource.java` — REST: GET /api/investigations/{caseId}/compliance-evidence, POST /api/actors/{actorId}/erasure

**New in `app/src/main/resources/db/aml-trust-routing/migration/`:**
- `V2004__aml_trust_routing_attestation.sql`

**Modified:**
- `app/src/main/java/io/casehub/aml/ledger/AmlLedgerService.java` — writeComplianceReviewOpened derives causedByEntryId
- `app/src/main/java/io/casehub/aml/routing/AmlTrustRoutingPolicyProvider.java` — add capabilities() method
- `app/src/main/resources/application.properties` — add trust package scan + Flyway location
- `app/src/test/resources/application.properties` — same, plus JpaLedgerMerkleFrontierRepository in selected-alternatives + tokenisation enabled

**New tests:**
- `app/src/test/java/io/casehub/aml/ledger/AmlLedgerCausedByTest.java`
- `app/src/test/java/io/casehub/aml/compliance/AmlComplianceEvidenceServiceTest.java`
- `app/src/test/java/io/casehub/aml/trust/AmlTrustRoutingAttestationTest.java`
- `app/src/test/java/io/casehub/aml/compliance/AmlLayer7ResourceTest.java`
- `app/src/test/java/io/casehub/aml/compliance/AmlLayer7ErasureTest.java`

---

## Task 1: Fix causedByEntryId chain in AmlLedgerService

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/ledger/AmlLedgerService.java`
- Create: `app/src/test/java/io/casehub/aml/ledger/AmlLedgerCausedByTest.java`

- [ ] **Step 1.1: Write the failing test**

```java
// app/src/test/java/io/casehub/aml/ledger/AmlLedgerCausedByTest.java
package io.casehub.aml.ledger;

import io.casehub.aml.AmlInvestigationApplicationService;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class AmlLedgerCausedByTest {

    @Inject AmlInvestigationApplicationService service;
    @Inject LedgerEntryRepository ledgerRepo;

    @Test
    void complianceReviewEntry_hasCausedByEntryId_pointingToCaseOpened() {
        UUID caseId = service.investigate(tx("TXN-CB-001")).caseId();

        List<LedgerEntry> entries = ledgerRepo.findBySubjectId(caseId);
        AmlInvestigationLedgerEntry caseOpened = entries.stream()
            .filter(e -> e instanceof AmlInvestigationLedgerEntry ale
                         && "CASE_OPENED".equals(ale.eventType))
            .map(e -> (AmlInvestigationLedgerEntry) e)
            .findFirst().orElseThrow(() -> new AssertionError("CASE_OPENED not found"));

        AmlInvestigationLedgerEntry reviewOpened = entries.stream()
            .filter(e -> e instanceof AmlInvestigationLedgerEntry ale
                         && "COMPLIANCE_REVIEW_OPENED".equals(ale.eventType))
            .map(e -> (AmlInvestigationLedgerEntry) e)
            .findFirst().orElseThrow(() -> new AssertionError("COMPLIANCE_REVIEW_OPENED not found"));

        assertNotNull(reviewOpened.causedByEntryId,
            "COMPLIANCE_REVIEW_OPENED must have causedByEntryId set");
        assertEquals(caseOpened.id, reviewOpened.causedByEntryId,
            "causedByEntryId must point to the CASE_OPENED entry");
    }

    private SuspiciousTransaction tx(String id) {
        return new SuspiciousTransaction(id, "ACC-A", "ACC-B",
            new BigDecimal("50000"), "USD", Instant.now(), "Structuring");
    }
}
```

- [ ] **Step 1.2: Run to confirm it fails**

```bash
mvn verify -pl app -am -Dtest=AmlLedgerCausedByTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: FAIL — `causedByEntryId` is null on `COMPLIANCE_REVIEW_OPENED`.

- [ ] **Step 1.3: Fix AmlLedgerService.writeComplianceReviewOpened**

Replace the existing `writeComplianceReviewOpened` method body with:

```java
public void writeComplianceReviewOpened(final UUID caseId, final String taskId) {
    // Derive causedByEntryId by querying for the CASE_OPENED entry.
    // This works for both the synchronous Layer 3 path and the async Quartz
    // Layer 5 path — no parameter threading needed.
    final UUID caseOpenedEntryId = repository.findBySubjectId(caseId).stream()
            .filter(e -> e instanceof AmlInvestigationLedgerEntry ale
                         && "CASE_OPENED".equals(ale.eventType))
            .map(e -> e.id)
            .findFirst()
            .orElse(null);

    final int sequenceNumber = nextSequenceNumber(caseId);
    final AmlInvestigationLedgerEntry entry = new AmlInvestigationLedgerEntry();
    entry.id = UUID.randomUUID();
    entry.subjectId = caseId;
    entry.sequenceNumber = sequenceNumber;
    entry.entryType = LedgerEntryType.EVENT;
    entry.actorId = ACTOR_ID;
    entry.actorType = ActorType.SYSTEM;
    entry.actorRole = ACTOR_ROLE;
    entry.occurredAt = Instant.now();
    entry.transactionId = taskId;
    entry.eventType = "COMPLIANCE_REVIEW_OPENED";
    entry.causedByEntryId = caseOpenedEntryId;
    repository.save(entry);
}
```

- [ ] **Step 1.4: Run the test to confirm it passes**

```bash
mvn verify -pl app -am -Dtest=AmlLedgerCausedByTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: PASS.

- [ ] **Step 1.5: Verify existing ledger tests still pass**

```bash
mvn verify -pl app -am -Dtest=AmlLedgerChainTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all 6 tests PASS.

- [ ] **Step 1.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/src/main/java/io/casehub/aml/ledger/AmlLedgerService.java \
  app/src/test/java/io/casehub/aml/ledger/AmlLedgerCausedByTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "fix(#43): wire causedByEntryId on COMPLIANCE_REVIEW_OPENED ledger entry"
```

---

## Task 2: Add capabilities() to AmlTrustRoutingPolicyProvider

The `POLICIES` map is `private` — `AmlComplianceEvidenceService` needs to know which capability tags have policies (to assess attestation completeness).

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/routing/AmlTrustRoutingPolicyProvider.java`

- [ ] **Step 2.1: Add capabilities() method**

Add after the `forCapability` method:

```java
/** Returns the set of capability tags for which explicit routing policies are configured. */
public java.util.Set<String> capabilities() {
    return POLICIES.keySet();
}
```

- [ ] **Step 2.2: Verify existing routing tests still pass**

```bash
mvn verify -pl app -am -Dtest=AmlTrustRoutingPolicyProviderTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: PASS.

- [ ] **Step 2.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/src/main/java/io/casehub/aml/routing/AmlTrustRoutingPolicyProvider.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#43): expose capabilities() on AmlTrustRoutingPolicyProvider"
```

---

## Task 3: Flyway migration + AmlTrustRoutingAttestation entity

**Files:**
- Create: `app/src/main/resources/db/aml-trust-routing/migration/V2004__aml_trust_routing_attestation.sql`
- Create: `app/src/main/java/io/casehub/aml/trust/AmlTrustRoutingAttestation.java`

- [ ] **Step 3.1: Create the migration**

```sql
-- app/src/main/resources/db/aml-trust-routing/migration/V2004__aml_trust_routing_attestation.sql
-- Layer 7: join table for AmlTrustRoutingAttestation (extends LedgerEntry via JOINED inheritance).
-- Records trust score at routing time per capability per case, written by AmlTrustRoutingObserver.
-- H2 MODE=PostgreSQL compatible — no PostgreSQL-specific DDL.
CREATE TABLE aml_trust_routing_attestation (
    id                    UUID         NOT NULL,
    capability_tag        VARCHAR(100) NOT NULL,
    selected_worker_id    VARCHAR(255) NOT NULL,
    trust_score_at_routing DOUBLE PRECISION,          -- nullable: null = no trust data available at routing
    threshold_applied     DOUBLE PRECISION NOT NULL,
    investigation_case_id UUID         NOT NULL,
    CONSTRAINT pk_aml_trust_routing_attestation PRIMARY KEY (id),
    CONSTRAINT fk_aml_trust_routing_ledger FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

- [ ] **Step 3.2: Create the entity**

```java
// app/src/main/java/io/casehub/aml/trust/AmlTrustRoutingAttestation.java
package io.casehub.aml.trust;

import io.casehub.ledger.runtime.model.LedgerEntry;
import jakarta.persistence.*;
import java.util.UUID;

/**
 * Layer 7: captures trust score at routing time per capability.
 *
 * Written by AmlTrustRoutingObserver on each WorkerDecisionEvent.
 * trustScoreAtRouting is nullable — null means TrustScoreCache had no entry
 * for this worker at routing time (distinct from a score of 0.0).
 *
 * When casehubio/engine#403 lands (trustScoreAtRouting on WorkerDecisionEntry),
 * this entity becomes redundant and can be removed.
 */
@Entity
@Table(name = "aml_trust_routing_attestation")
@DiscriminatorValue("AML_TRUST_ROUTING")
public class AmlTrustRoutingAttestation extends LedgerEntry {

    @Column(name = "capability_tag", nullable = false, length = 100)
    public String capabilityTag;

    @Column(name = "selected_worker_id", nullable = false)
    public String selectedWorkerId;

    @Column(name = "trust_score_at_routing")
    public Double trustScoreAtRouting;

    @Column(name = "threshold_applied", nullable = false)
    public double thresholdApplied;

    @Column(name = "investigation_case_id", nullable = false)
    public UUID investigationCaseId;
}
```

- [ ] **Step 3.3: Register entity package and Flyway location in application.properties**

In `app/src/main/resources/application.properties`, append `,io.casehub.aml.trust` to the qhorus packages line:

```properties
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus.runtime,io.casehub.ledger.runtime.model,io.casehub.ledger.model,io.casehub.aml.ledger,io.casehub.aml.trust
```

Append `,classpath:db/aml-trust-routing/migration` to the qhorus Flyway locations line:

```properties
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration,classpath:db/aml-ledger/migration,classpath:db/engine-ledger/migration,classpath:db/aml-trust-routing/migration
```

Apply the same changes to `app/src/test/resources/application.properties`.

Additionally, in `app/src/test/resources/application.properties`, update `selected-alternatives` to include `JpaLedgerMerkleFrontierRepository` (required by `LedgerVerificationService`):

```properties
quarkus.arc.selected-alternatives=io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,io.casehub.ledger.runtime.repository.jpa.JpaLedgerMerkleFrontierRepository
```

And enable tokenisation so `LedgerErasureService.erase()` can create/delete `ActorIdentity` rows:

```properties
casehub.ledger.identity.tokenisation.enabled=true
```

- [ ] **Step 3.4: Verify migration applies and Quarkus starts**

```bash
mvn verify -pl app -am -Dtest=AmlLedgerChainTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: PASS — migration V2004 applies cleanly, no schema errors.

- [ ] **Step 3.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/src/main/resources/db/aml-trust-routing/migration/V2004__aml_trust_routing_attestation.sql \
  app/src/main/java/io/casehub/aml/trust/AmlTrustRoutingAttestation.java \
  app/src/main/resources/application.properties \
  app/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#43): AmlTrustRoutingAttestation entity and V2004 migration"
```

---

## Task 4: AmlTrustAttestationRepository

**Files:**
- Create: `app/src/main/java/io/casehub/aml/trust/AmlTrustAttestationRepository.java`

- [ ] **Step 4.1: Create the repository**

```java
// app/src/main/java/io/casehub/aml/trust/AmlTrustAttestationRepository.java
package io.casehub.aml.trust;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import jakarta.transaction.Transactional;
import jakarta.transaction.Transactional.TxType;
import java.util.List;
import java.util.UUID;

@ApplicationScoped
public class AmlTrustAttestationRepository {

    @PersistenceContext(unitName = "qhorus")
    EntityManager em;

    @Transactional(TxType.SUPPORTS)
    public List<AmlTrustRoutingAttestation> findByInvestigationCaseId(UUID caseId) {
        return em.createQuery(
                "SELECT a FROM AmlTrustRoutingAttestation a" +
                " WHERE a.investigationCaseId = :caseId ORDER BY a.occurredAt ASC",
                AmlTrustRoutingAttestation.class)
                .setParameter("caseId", caseId)
                .getResultList();
    }
}
```

- [ ] **Step 4.2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/src/main/java/io/casehub/aml/trust/AmlTrustAttestationRepository.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#43): AmlTrustAttestationRepository"
```

---

## Task 5: AmlTrustRoutingObserver

**Files:**
- Create: `app/src/main/java/io/casehub/aml/trust/AmlTrustRoutingObserver.java`
- Create: `app/src/test/java/io/casehub/aml/trust/AmlTrustRoutingAttestationTest.java`

- [ ] **Step 5.1: Write the failing test**

```java
// app/src/test/java/io/casehub/aml/trust/AmlTrustRoutingAttestationTest.java
package io.casehub.aml.trust;

import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.engine.AmlEngineCoordinator;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.awaitility.Awaitility;
import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.TimeUnit;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class AmlTrustRoutingAttestationTest {

    @Inject AmlEngineCoordinator coordinator;
    @Inject AmlTrustAttestationRepository attestationRepo;

    @Test
    void workerDispatch_writesAttestationPerCapability() {
        UUID caseId = coordinator.startInvestigation(pep("TXN-ATT-001"));

        // Wait for sar-drafting worker to be scheduled (last in the chain)
        Awaitility.await().atMost(10, TimeUnit.SECONDS).until(() ->
            !attestationRepo.findByInvestigationCaseId(caseId).isEmpty()
        );

        List<AmlTrustRoutingAttestation> attestations =
            attestationRepo.findByInvestigationCaseId(caseId);

        assertFalse(attestations.isEmpty(), "At least one attestation must be written");

        for (AmlTrustRoutingAttestation a : attestations) {
            assertEquals(caseId, a.investigationCaseId);
            assertNotNull(a.capabilityTag, "capabilityTag must be set");
            assertNotNull(a.selectedWorkerId, "selectedWorkerId must be set");
            assertTrue(a.thresholdApplied > 0.0, "thresholdApplied must be positive");
            // trustScoreAtRouting may be null if cache had no entry — that is valid
        }
    }

    @Test
    void workerDispatch_attestation_hasNonNullScore_whenCacheSeeded() {
        UUID caseId = coordinator.startInvestigation(pep("TXN-ATT-002"));

        Awaitility.await().atMost(10, TimeUnit.SECONDS).until(() ->
            attestationRepo.findByInvestigationCaseId(caseId).stream()
                .anyMatch(a -> "sar-drafting".equals(a.capabilityTag))
        );

        AmlTrustRoutingAttestation sarAttestation = attestationRepo
            .findByInvestigationCaseId(caseId).stream()
            .filter(a -> "sar-drafting".equals(a.capabilityTag))
            .findFirst().orElseThrow(() -> new AssertionError("No sar-drafting attestation"));

        // AmlTrustScoreSeeder seeds sar-drafting agents at startup with Beta(9,1) ≈ 0.9
        assertNotNull(sarAttestation.trustScoreAtRouting,
            "trustScoreAtRouting must be non-null when cache is seeded");
        assertTrue(sarAttestation.trustScoreAtRouting > 0.0,
            "trustScoreAtRouting must be positive when seeded");
    }

    private SuspiciousTransaction pep(String id) {
        return new SuspiciousTransaction(id, "ACC-A", "ACC-B",
            new BigDecimal("200000"), "USD", Instant.now(), "PEP — high risk transfer");
    }
}
```

- [ ] **Step 5.2: Run to confirm it fails**

```bash
mvn verify -pl app -am -Dtest=AmlTrustRoutingAttestationTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: FAIL — `AmlTrustRoutingObserver` not yet created, attestation table is empty.

- [ ] **Step 5.3: Implement AmlTrustRoutingObserver**

```java
// app/src/main/java/io/casehub/aml/trust/AmlTrustRoutingObserver.java
package io.casehub.aml.trust;

import io.casehub.aml.routing.AmlTrustRoutingPolicyProvider;
import io.casehub.engine.common.spi.event.WorkerDecisionEvent;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.routing.TrustScoreCache;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.transaction.Transactional.TxType;
import java.time.Instant;
import java.util.UUID;

/**
 * Layer 7: writes AmlTrustRoutingAttestation on each WorkerDecisionEvent,
 * capturing the trust score from TrustScoreCache before it can drift.
 *
 * REQUIRES_NEW gives each observer call its own transaction, preventing a shared
 * transaction with the engine worker from interfering with the attestation write.
 * Sequence number collisions are unlikely in tutorial (workers dispatch near-sequentially)
 * but possible in production — tracked as casehubio/aml#44.
 */
@ApplicationScoped
public class AmlTrustRoutingObserver {

    private static final String ACTOR_ID = "aml-orchestrator";
    private static final String ACTOR_ROLE = "AmlInvestigationOrchestrator";

    @Inject TrustScoreCache trustScoreCache;
    @Inject AmlTrustRoutingPolicyProvider policyProvider;
    @Inject LedgerEntryRepository ledgerRepo;

    @Transactional(TxType.REQUIRES_NEW)
    public void onWorkerDecision(@Observes WorkerDecisionEvent event) {
        final Double score = trustScoreCache
                .getCapabilityScore(event.workerId(), event.capabilityTag())
                .stream().boxed().findFirst().orElse(null);

        final double threshold = policyProvider.forCapability(event.capabilityTag()).threshold();
        final int seq = nextSequenceNumber(event.caseId());

        final AmlTrustRoutingAttestation entry = new AmlTrustRoutingAttestation();
        entry.id = UUID.randomUUID();
        entry.subjectId = event.caseId();
        entry.investigationCaseId = event.caseId();
        entry.capabilityTag = event.capabilityTag();
        entry.selectedWorkerId = event.workerId();
        entry.trustScoreAtRouting = score;
        entry.thresholdApplied = threshold;
        entry.sequenceNumber = seq;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = ACTOR_ID;
        entry.actorType = ActorType.SYSTEM;
        entry.actorRole = ACTOR_ROLE;
        entry.occurredAt = Instant.now();
        ledgerRepo.save(entry);
    }

    private int nextSequenceNumber(UUID subjectId) {
        return ledgerRepo.findLatestBySubjectId(subjectId)
                .map(e -> e.sequenceNumber + 1)
                .orElse(1);
    }
}
```

- [ ] **Step 5.4: Run the test to confirm it passes**

```bash
mvn verify -pl app -am -Dtest=AmlTrustRoutingAttestationTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: both tests PASS.

- [ ] **Step 5.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/src/main/java/io/casehub/aml/trust/AmlTrustRoutingObserver.java \
  app/src/test/java/io/casehub/aml/trust/AmlTrustRoutingAttestationTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#43): AmlTrustRoutingObserver captures trust score at routing time"
```

---

## Task 6: API types in api/

These are pure Java records with no logic — no tests needed. Create all in one step.

**Files:** Create 10 files in `api/src/main/java/io/casehub/aml/compliance/`

- [ ] **Step 6.1: Create RequirementStatus**

```java
// api/src/main/java/io/casehub/aml/compliance/RequirementStatus.java
package io.casehub.aml.compliance;

public enum RequirementStatus {
    /** Requirement demonstrably met with evidence. */
    CLOSED,
    /** Mechanism present but evidence incomplete (chain unverified; some capabilities missing). */
    PARTIAL,
    /** Mechanism present but obligation not met (SLA deadline passed without completion). */
    BREACHED,
    /** Architectural gap; requirement not addressed. */
    GAP
}
```

- [ ] **Step 6.2: Create AmlProofStep and AmlInclusionProof**

```java
// api/src/main/java/io/casehub/aml/compliance/AmlProofStep.java
package io.casehub.aml.compliance;

public record AmlProofStep(String hash, String position) {} // position: "LEFT" | "RIGHT"
```

```java
// api/src/main/java/io/casehub/aml/compliance/AmlInclusionProof.java
package io.casehub.aml.compliance;

import java.util.List;

public record AmlInclusionProof(
    int entryIndex,
    int treeSize,
    String leafHash,
    List<AmlProofStep> siblings,
    String treeRoot
) {}
```

- [ ] **Step 6.3: Create LedgerEventRecord**

```java
// api/src/main/java/io/casehub/aml/compliance/LedgerEventRecord.java
package io.casehub.aml.compliance;

import java.time.Instant;
import java.util.UUID;

public record LedgerEventRecord(
    UUID entryId,
    String eventType,
    String actorId,
    String actorRole,
    Instant occurredAt,
    UUID causedByEntryId,
    String digest,
    AmlInclusionProof inclusionProof
) {}
```

- [ ] **Step 6.4: Create AuditChainRequirement**

```java
// api/src/main/java/io/casehub/aml/compliance/AuditChainRequirement.java
package io.casehub.aml.compliance;

import java.util.List;

public record AuditChainRequirement(
    String id,
    String citation,
    String mechanism,
    RequirementStatus status,
    String treeRoot,
    boolean chainVerified,
    List<LedgerEventRecord> events
) {
    public static final String REQUIREMENT_ID = "FINCEN-31CFR1020.320-AUDIT-CHAIN";
    public static final String CITATION =
        "31 CFR § 1020.320(a) / FATF R.16 — Auditable evidence chain with tamper-evident record";
    public static final String MECHANISM =
        "Layers 3+4: qhorus COMMAND/DONE/DECLINE per specialist + Merkle-chained LedgerEntry " +
        "(causedByEntryId links COMPLIANCE_REVIEW_OPENED to CASE_OPENED). " +
        "SCOPE: covers AML domain lifecycle events only. " +
        "Specialist dispatch audit lives in the qhorus ledger chain (subjectId = caseId).";
}
```

- [ ] **Step 6.5: Create SlaRequirement**

```java
// api/src/main/java/io/casehub/aml/compliance/SlaRequirement.java
package io.casehub.aml.compliance;

import java.time.Instant;
import java.util.List;
import java.util.UUID;

public record SlaRequirement(
    String id,
    String citation,
    String mechanism,
    RequirementStatus status,
    UUID workItemId,
    Instant claimDeadline,
    Instant completedAt,
    boolean slaMet,
    List<String> candidateGroups,
    String escalationPolicy
) {
    public static final String REQUIREMENT_ID = "FINCEN-SAR-30DAY-SLA";
    public static final String CITATION =
        "31 CFR § 1020.320(b)(3) — SAR human sign-off with 30-day filing deadline";
    public static final String MECHANISM =
        "Layer 2: casehub-work WorkItem with claimDeadline + candidateGroups=compliance-officers. " +
        "Layer 5: engine auto-escalation to senior-compliance-officers on deadline breach.";
    public static final String ESCALATION_POLICY =
        "senior-compliance-officers after claimDeadline breach";
}
```

- [ ] **Step 6.6: Create RoutingDecisionRecord and TrustRoutingRequirement**

```java
// api/src/main/java/io/casehub/aml/compliance/RoutingDecisionRecord.java
package io.casehub.aml.compliance;

import java.util.UUID;

public record RoutingDecisionRecord(
    String capabilityTag,
    String selectedWorker,
    Double trustScoreAtRouting,   // null = no trust data was available at routing time
    double thresholdApplied,
    UUID attestationEntryId
) {}
```

```java
// api/src/main/java/io/casehub/aml/compliance/TrustRoutingRequirement.java
package io.casehub.aml.compliance;

import java.util.List;

public record TrustRoutingRequirement(
    String id,
    String citation,
    String mechanism,
    RequirementStatus status,
    List<RoutingDecisionRecord> decisions
) {
    public static final String REQUIREMENT_ID = "FATF-R20-TRUST-ROUTING";
    public static final String CITATION =
        "FATF Recommendation 20 — Experienced analysts on complex cases";
    public static final String MECHANISM =
        "Layer 6: TrustWeightedAgentStrategy reads AmlTrustRoutingPolicyProvider thresholds. " +
        "Layer 7: AmlTrustRoutingAttestation captures trustScoreAtRouting at WorkerDecisionEvent " +
        "time before TrustScoreCache can drift. Workaround for casehubio/engine#403.";
}
```

- [ ] **Step 6.7: Create GdprErasureRequirement**

```java
// api/src/main/java/io/casehub/aml/compliance/GdprErasureRequirement.java
package io.casehub.aml.compliance;

public record GdprErasureRequirement(
    String id,
    String citation,
    String mechanism,
    boolean erasureCapabilityWired,
    boolean pseudonymizationActive,
    String erasureEndpoint
) {
    public static final String REQUIREMENT_ID = "GDPR-ART17-ERASURE";
    public static final String CITATION =
        "GDPR Art. 17 / FATF privacy obligation — PII erasure preserving audit structure";
    public static final String MECHANISM =
        "LedgerErasureService pseudonymizes actorId in ledger_entry rows via ActorIdentity token. " +
        "Audit entries remain intact; actor identity is replaced with an opaque token. " +
        "NOTE: current tutorial erases system actor; proper data subject erasure (analyst identity) " +
        "requires AML_SAR_OFFICER_REVIEWED ledger event — tracked as casehubio/aml#44.";
    public static final String ERASURE_ENDPOINT = "POST /api/actors/{actorId}/erasure";
}
```

- [ ] **Step 6.8: Create ComplianceEvidence**

```java
// api/src/main/java/io/casehub/aml/compliance/ComplianceEvidence.java
package io.casehub.aml.compliance;

import java.time.Instant;
import java.util.UUID;

public record ComplianceEvidence(
    UUID caseId,
    Instant generatedAt,
    AuditChainRequirement auditChain,
    SlaRequirement sla,
    TrustRoutingRequirement trustRouting,
    GdprErasureRequirement gdprErasure,
    String signature   // null — reserved for future offline signing (casehubio/engine#403 follow-on)
) {}
```

- [ ] **Step 6.9: Compile to verify no errors**

```bash
mvn compile -pl api -am -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 6.10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add api/src/main/java/io/casehub/aml/compliance/
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#43): compliance evidence API types in api/"
```

---

## Task 7: AmlComplianceEvidenceService (unit-tested)

**Files:**
- Create: `app/src/main/java/io/casehub/aml/compliance/AmlComplianceEvidenceService.java`
- Create: `app/src/test/java/io/casehub/aml/compliance/AmlComplianceEvidenceServiceTest.java`

- [ ] **Step 7.1: Write the unit test**

```java
// app/src/test/java/io/casehub/aml/compliance/AmlComplianceEvidenceServiceTest.java
package io.casehub.aml.compliance;

import io.casehub.aml.ledger.AmlInvestigationLedgerEntry;
import io.casehub.aml.routing.AmlTrustRoutingPolicyProvider;
import io.casehub.aml.trust.AmlTrustAttestationRepository;
import io.casehub.aml.trust.AmlTrustRoutingAttestation;
import io.casehub.aml.trust.AmlWorkerDecisionRepository;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.model.WorkerDecisionEntry;
import io.casehub.ledger.runtime.privacy.LedgerErasureService;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.LedgerVerificationService;
import io.casehub.ledger.runtime.service.model.InclusionProof;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import jakarta.persistence.EntityManager;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class AmlComplianceEvidenceServiceTest {

    @Mock LedgerEntryRepository ledgerRepo;
    @Mock LedgerVerificationService verificationService;
    @Mock AmlTrustAttestationRepository attestationRepo;
    @Mock AmlWorkerDecisionRepository workerDecisionRepo;
    @Mock AmlTrustRoutingPolicyProvider policyProvider;
    @Mock LedgerErasureService erasureService;
    @Mock EntityManager em;

    AmlComplianceEvidenceService service;

    UUID caseId = UUID.randomUUID();
    UUID caseOpenedId = UUID.randomUUID();
    UUID reviewOpenedId = UUID.randomUUID();
    UUID taskId = UUID.randomUUID();
    String treeRoot = "sha256:abc123";

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        service = new AmlComplianceEvidenceService(
            ledgerRepo, verificationService, attestationRepo,
            workerDecisionRepo, policyProvider, em);

        when(policyProvider.capabilities()).thenReturn(Set.of("entity-resolution", "sar-drafting"));
    }

    @Test
    void assembleEvidence_happyPath_allRequirementsClosed() {
        // Ledger entries
        AmlInvestigationLedgerEntry opened = caseOpenedEntry(caseId, caseOpenedId);
        AmlInvestigationLedgerEntry review = reviewOpenedEntry(caseId, reviewOpenedId, taskId, caseOpenedId);
        when(ledgerRepo.findBySubjectId(caseId)).thenReturn(List.of(opened, review));

        // Merkle verification
        when(verificationService.verify(caseId)).thenReturn(true);
        when(verificationService.treeRoot(caseId)).thenReturn(treeRoot);
        when(verificationService.inclusionProof(any())).thenReturn(stubProof());

        // WorkItem lookup
        WorkItem wi = workItem(taskId, Instant.now().plus(30, ChronoUnit.DAYS), null);
        when(em.find(eq(WorkItem.class), eq(taskId))).thenReturn(wi);

        // Trust routing
        AmlTrustRoutingAttestation att1 = attestation(caseId, "entity-resolution", "agent-A", 0.8, 0.70);
        AmlTrustRoutingAttestation att2 = attestation(caseId, "sar-drafting", "agent-B", 0.9, 0.75);
        when(attestationRepo.findByInvestigationCaseId(caseId)).thenReturn(List.of(att1, att2));

        // Worker decisions (same capability set as attestations)
        WorkerDecisionEntry wd1 = workerDecision(caseId, "entity-resolution", "agent-A");
        WorkerDecisionEntry wd2 = workerDecision(caseId, "sar-drafting", "agent-B");
        when(workerDecisionRepo.findAllByCaseId(caseId)).thenReturn(List.of(wd1, wd2));

        ComplianceEvidence evidence = service.assembleEvidence(caseId);

        assertEquals(caseId, evidence.caseId());
        assertNotNull(evidence.generatedAt());
        assertNull(evidence.signature());

        assertEquals(RequirementStatus.CLOSED, evidence.auditChain().status());
        assertTrue(evidence.auditChain().chainVerified());
        assertEquals(treeRoot, evidence.auditChain().treeRoot());
        assertEquals(2, evidence.auditChain().events().size());
        assertNull(evidence.auditChain().events().get(0).causedByEntryId());      // CASE_OPENED
        assertEquals(caseOpenedId, evidence.auditChain().events().get(1).causedByEntryId()); // COMPLIANCE_REVIEW_OPENED

        assertEquals(RequirementStatus.CLOSED, evidence.sla().status());
        assertTrue(evidence.sla().slaMet());
        assertNotNull(evidence.sla().workItemId());

        assertEquals(RequirementStatus.CLOSED, evidence.trustRouting().status());
        assertEquals(2, evidence.trustRouting().decisions().size());

        assertTrue(evidence.gdprErasure().erasureCapabilityWired());
    }

    @Test
    void assembleEvidence_chainVerifiedFalse_auditChainPartial() {
        AmlInvestigationLedgerEntry opened = caseOpenedEntry(caseId, caseOpenedId);
        AmlInvestigationLedgerEntry review = reviewOpenedEntry(caseId, reviewOpenedId, taskId, caseOpenedId);
        when(ledgerRepo.findBySubjectId(caseId)).thenReturn(List.of(opened, review));
        when(verificationService.verify(caseId)).thenReturn(false);
        when(verificationService.treeRoot(caseId)).thenReturn(treeRoot);
        when(verificationService.inclusionProof(any())).thenReturn(stubProof());
        when(em.find(eq(WorkItem.class), eq(taskId))).thenReturn(workItem(taskId,
            Instant.now().plus(30, ChronoUnit.DAYS), null));
        when(attestationRepo.findByInvestigationCaseId(caseId)).thenReturn(List.of());
        when(workerDecisionRepo.findAllByCaseId(caseId)).thenReturn(List.of());

        ComplianceEvidence evidence = service.assembleEvidence(caseId);
        assertEquals(RequirementStatus.PARTIAL, evidence.auditChain().status());
    }

    @Test
    void assembleEvidence_slaBreached_statusBreached() {
        AmlInvestigationLedgerEntry opened = caseOpenedEntry(caseId, caseOpenedId);
        AmlInvestigationLedgerEntry review = reviewOpenedEntry(caseId, reviewOpenedId, taskId, caseOpenedId);
        when(ledgerRepo.findBySubjectId(caseId)).thenReturn(List.of(opened, review));
        when(verificationService.verify(caseId)).thenReturn(true);
        when(verificationService.treeRoot(caseId)).thenReturn(treeRoot);
        when(verificationService.inclusionProof(any())).thenReturn(stubProof());
        when(attestationRepo.findByInvestigationCaseId(caseId)).thenReturn(List.of());
        when(workerDecisionRepo.findAllByCaseId(caseId)).thenReturn(List.of());

        // WorkItem not completed and past deadline
        Instant deadline = Instant.now().minus(1, ChronoUnit.DAYS);
        when(em.find(eq(WorkItem.class), eq(taskId))).thenReturn(workItem(taskId, deadline, null));

        ComplianceEvidence evidence = service.assembleEvidence(caseId);
        assertEquals(RequirementStatus.BREACHED, evidence.sla().status());
        assertFalse(evidence.sla().slaMet());
    }

    @Test
    void assembleEvidence_partialAttestations_trustRoutingPartial() {
        AmlInvestigationLedgerEntry opened = caseOpenedEntry(caseId, caseOpenedId);
        AmlInvestigationLedgerEntry review = reviewOpenedEntry(caseId, reviewOpenedId, taskId, caseOpenedId);
        when(ledgerRepo.findBySubjectId(caseId)).thenReturn(List.of(opened, review));
        when(verificationService.verify(caseId)).thenReturn(true);
        when(verificationService.treeRoot(caseId)).thenReturn(treeRoot);
        when(verificationService.inclusionProof(any())).thenReturn(stubProof());
        when(em.find(eq(WorkItem.class), eq(taskId))).thenReturn(workItem(taskId,
            Instant.now().plus(30, ChronoUnit.DAYS), null));

        // Only one attestation, but two worker decisions
        AmlTrustRoutingAttestation att1 = attestation(caseId, "entity-resolution", "agent-A", 0.8, 0.70);
        when(attestationRepo.findByInvestigationCaseId(caseId)).thenReturn(List.of(att1));
        WorkerDecisionEntry wd1 = workerDecision(caseId, "entity-resolution", "agent-A");
        WorkerDecisionEntry wd2 = workerDecision(caseId, "sar-drafting", "agent-B");
        when(workerDecisionRepo.findAllByCaseId(caseId)).thenReturn(List.of(wd1, wd2));

        ComplianceEvidence evidence = service.assembleEvidence(caseId);
        assertEquals(RequirementStatus.PARTIAL, evidence.trustRouting().status());
    }

    @Test
    void assembleEvidence_noLedgerEntries_returnsEmpty() {
        when(ledgerRepo.findBySubjectId(caseId)).thenReturn(List.of());
        assertTrue(service.findEvidence(caseId).isEmpty());
    }

    // ── Helpers ────────────────────────────────────────────────────────────────

    private AmlInvestigationLedgerEntry caseOpenedEntry(UUID caseId, UUID entryId) {
        AmlInvestigationLedgerEntry e = new AmlInvestigationLedgerEntry();
        e.id = entryId; e.subjectId = caseId; e.sequenceNumber = 1;
        e.eventType = "CASE_OPENED"; e.transactionId = "TXN-001";
        e.entryType = LedgerEntryType.EVENT; e.actorId = "aml-orchestrator";
        e.actorType = ActorType.SYSTEM; e.actorRole = "AmlInvestigationOrchestrator";
        e.occurredAt = Instant.now().minus(5, ChronoUnit.MINUTES);
        e.digest = "sha256:abc"; e.causedByEntryId = null;
        return e;
    }

    private AmlInvestigationLedgerEntry reviewOpenedEntry(UUID caseId, UUID entryId,
            UUID taskId, UUID causedBy) {
        AmlInvestigationLedgerEntry e = new AmlInvestigationLedgerEntry();
        e.id = entryId; e.subjectId = caseId; e.sequenceNumber = 2;
        e.eventType = "COMPLIANCE_REVIEW_OPENED"; e.transactionId = taskId.toString();
        e.entryType = LedgerEntryType.EVENT; e.actorId = "aml-orchestrator";
        e.actorType = ActorType.SYSTEM; e.actorRole = "AmlInvestigationOrchestrator";
        e.occurredAt = Instant.now(); e.digest = "sha256:def"; e.causedByEntryId = causedBy;
        return e;
    }

    private WorkItem workItem(UUID taskId, Instant deadline, Instant completedAt) {
        WorkItem wi = new WorkItem();
        wi.id = taskId;
        wi.status = completedAt != null ? WorkItemStatus.COMPLETED : WorkItemStatus.PENDING;
        wi.claimDeadline = deadline;
        wi.completedAt = completedAt;
        wi.candidateGroups = "compliance-officers";
        return wi;
    }

    private AmlTrustRoutingAttestation attestation(UUID caseId, String cap, String worker,
            double score, double threshold) {
        AmlTrustRoutingAttestation a = new AmlTrustRoutingAttestation();
        a.id = UUID.randomUUID(); a.investigationCaseId = caseId;
        a.capabilityTag = cap; a.selectedWorkerId = worker;
        a.trustScoreAtRouting = score; a.thresholdApplied = threshold;
        a.occurredAt = Instant.now();
        return a;
    }

    private WorkerDecisionEntry workerDecision(UUID caseId, String cap, String worker) {
        WorkerDecisionEntry wd = new WorkerDecisionEntry();
        wd.caseId = caseId; wd.capabilityTag = cap; wd.workerId = worker;
        return wd;
    }

    private InclusionProof stubProof() {
        return new InclusionProof(UUID.randomUUID(), 0, 2, "sha256:leaf", List.of(), "sha256:root");
    }
}
```

- [ ] **Step 7.2: Run to confirm it fails to compile**

```bash
mvn test -pl app -am -Dtest=AmlComplianceEvidenceServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: compilation error — `AmlComplianceEvidenceService` doesn't exist yet.

- [ ] **Step 7.3: Implement AmlComplianceEvidenceService**

```java
// app/src/main/java/io/casehub/aml/compliance/AmlComplianceEvidenceService.java
package io.casehub.aml.compliance;

import io.casehub.aml.ledger.AmlInvestigationLedgerEntry;
import io.casehub.aml.routing.AmlTrustRoutingPolicyProvider;
import io.casehub.aml.trust.AmlTrustAttestationRepository;
import io.casehub.aml.trust.AmlTrustRoutingAttestation;
import io.casehub.aml.trust.AmlWorkerDecisionRepository;
import io.casehub.ledger.model.WorkerDecisionEntry;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.LedgerVerificationService;
import io.casehub.ledger.runtime.service.model.InclusionProof;
import io.casehub.work.runtime.model.WorkItem;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import java.time.Instant;
import java.util.*;
import java.util.stream.Collectors;

@ApplicationScoped
public class AmlComplianceEvidenceService {

    private final LedgerEntryRepository ledgerRepo;
    private final LedgerVerificationService verificationService;
    private final AmlTrustAttestationRepository attestationRepo;
    private final AmlWorkerDecisionRepository workerDecisionRepo;
    private final AmlTrustRoutingPolicyProvider policyProvider;
    private final EntityManager em;

    @Inject
    public AmlComplianceEvidenceService(
            LedgerEntryRepository ledgerRepo,
            LedgerVerificationService verificationService,
            AmlTrustAttestationRepository attestationRepo,
            AmlWorkerDecisionRepository workerDecisionRepo,
            AmlTrustRoutingPolicyProvider policyProvider,
            EntityManager em) {
        this.ledgerRepo = ledgerRepo;
        this.verificationService = verificationService;
        this.attestationRepo = attestationRepo;
        this.workerDecisionRepo = workerDecisionRepo;
        this.policyProvider = policyProvider;
        this.em = em;
    }

    /** Returns empty if no ledger entries exist for this caseId (404 at REST layer). */
    public Optional<ComplianceEvidence> findEvidence(UUID caseId) {
        List<LedgerEntry> allEntries = ledgerRepo.findBySubjectId(caseId);
        if (allEntries.isEmpty()) return Optional.empty();
        return Optional.of(assembleEvidence(caseId, allEntries));
    }

    // Package-private for unit testing with pre-built entry lists
    ComplianceEvidence assembleEvidence(UUID caseId) {
        return assembleEvidence(caseId, ledgerRepo.findBySubjectId(caseId));
    }

    private ComplianceEvidence assembleEvidence(UUID caseId, List<LedgerEntry> allEntries) {
        List<AmlInvestigationLedgerEntry> amlEntries = allEntries.stream()
            .filter(e -> e instanceof AmlInvestigationLedgerEntry)
            .map(e -> (AmlInvestigationLedgerEntry) e)
            .sorted(Comparator.comparingInt(e -> e.sequenceNumber))
            .toList();

        return new ComplianceEvidence(
            caseId,
            Instant.now(),
            buildAuditChain(caseId, amlEntries),
            buildSla(caseId, amlEntries),
            buildTrustRouting(caseId),
            buildGdprErasure(),
            null
        );
    }

    private AuditChainRequirement buildAuditChain(UUID caseId,
            List<AmlInvestigationLedgerEntry> amlEntries) {
        boolean verified = false;
        String treeRoot = null;
        try {
            verified = verificationService.verify(caseId);
            treeRoot = verificationService.treeRoot(caseId);
        } catch (IllegalStateException ignored) {
            // No Merkle frontier yet (no entries in the frontier table)
        }

        List<LedgerEventRecord> events = amlEntries.stream()
            .map(e -> new LedgerEventRecord(
                e.id, e.eventType, e.actorId, e.actorRole, e.occurredAt,
                e.causedByEntryId, e.digest,
                buildInclusionProof(e.id)))
            .toList();

        boolean causedByPresent = amlEntries.stream()
            .filter(e -> "COMPLIANCE_REVIEW_OPENED".equals(e.eventType))
            .allMatch(e -> e.causedByEntryId != null);

        RequirementStatus status;
        if (amlEntries.isEmpty()) {
            status = RequirementStatus.GAP;
        } else if (verified && causedByPresent) {
            status = RequirementStatus.CLOSED;
        } else {
            status = RequirementStatus.PARTIAL;
        }

        return new AuditChainRequirement(
            AuditChainRequirement.REQUIREMENT_ID,
            AuditChainRequirement.CITATION,
            AuditChainRequirement.MECHANISM,
            status, treeRoot, verified, events);
    }

    private AmlInclusionProof buildInclusionProof(UUID entryId) {
        try {
            InclusionProof proof = verificationService.inclusionProof(entryId);
            List<AmlProofStep> siblings = proof.siblings().stream()
                .map(s -> new AmlProofStep(s.hash(), s.position()))
                .toList();
            return new AmlInclusionProof(
                proof.entryIndex(), proof.treeSize(),
                proof.leafHash(), siblings, proof.treeRoot());
        } catch (Exception e) {
            return null; // Entry not yet in Merkle tree
        }
    }

    private SlaRequirement buildSla(UUID caseId,
            List<AmlInvestigationLedgerEntry> amlEntries) {
        Optional<AmlInvestigationLedgerEntry> reviewEntry = amlEntries.stream()
            .filter(e -> "COMPLIANCE_REVIEW_OPENED".equals(e.eventType))
            .findFirst();

        if (reviewEntry.isEmpty()) {
            return new SlaRequirement(
                SlaRequirement.REQUIREMENT_ID, SlaRequirement.CITATION,
                SlaRequirement.MECHANISM, RequirementStatus.GAP,
                null, null, null, false, List.of(), SlaRequirement.ESCALATION_POLICY);
        }

        UUID taskId;
        try {
            taskId = UUID.fromString(reviewEntry.get().transactionId);
        } catch (IllegalArgumentException e) {
            // transactionId is not a valid UUID — treat as GAP
            return new SlaRequirement(
                SlaRequirement.REQUIREMENT_ID, SlaRequirement.CITATION,
                SlaRequirement.MECHANISM, RequirementStatus.GAP,
                null, null, null, false, List.of(), SlaRequirement.ESCALATION_POLICY);
        }

        WorkItem wi = em.find(WorkItem.class, taskId);
        if (wi == null) {
            return new SlaRequirement(
                SlaRequirement.REQUIREMENT_ID, SlaRequirement.CITATION,
                SlaRequirement.MECHANISM, RequirementStatus.GAP,
                taskId, null, null, false, List.of(), SlaRequirement.ESCALATION_POLICY);
        }

        List<String> groups = wi.candidateGroups != null
            ? List.of(wi.candidateGroups.split(",")) : List.of();
        boolean slaMet = wi.completedAt != null && wi.claimDeadline != null
            && wi.completedAt.isBefore(wi.claimDeadline);
        boolean breached = wi.claimDeadline != null
            && Instant.now().isAfter(wi.claimDeadline) && wi.completedAt == null;

        RequirementStatus status;
        if (wi.claimDeadline == null) {
            status = RequirementStatus.PARTIAL;
        } else if (breached) {
            status = RequirementStatus.BREACHED;
        } else {
            status = RequirementStatus.CLOSED;
        }

        return new SlaRequirement(
            SlaRequirement.REQUIREMENT_ID, SlaRequirement.CITATION,
            SlaRequirement.MECHANISM, status,
            taskId, wi.claimDeadline, wi.completedAt, slaMet,
            groups, SlaRequirement.ESCALATION_POLICY);
    }

    private TrustRoutingRequirement buildTrustRouting(UUID caseId) {
        List<AmlTrustRoutingAttestation> attestations =
            attestationRepo.findByInvestigationCaseId(caseId);
        Set<String> dispatchedCapabilities = workerDecisionRepo.findAllByCaseId(caseId)
            .stream().map(wd -> wd.capabilityTag).collect(Collectors.toSet());
        Set<String> attestedCapabilities = attestations.stream()
            .map(a -> a.capabilityTag).collect(Collectors.toSet());

        RequirementStatus status;
        if (dispatchedCapabilities.isEmpty()) {
            status = RequirementStatus.GAP;
        } else if (attestedCapabilities.containsAll(dispatchedCapabilities)) {
            status = RequirementStatus.CLOSED;
        } else {
            status = RequirementStatus.PARTIAL;
        }

        List<RoutingDecisionRecord> decisions = attestations.stream()
            .map(a -> new RoutingDecisionRecord(
                a.capabilityTag, a.selectedWorkerId,
                a.trustScoreAtRouting, a.thresholdApplied, a.id))
            .toList();

        return new TrustRoutingRequirement(
            TrustRoutingRequirement.REQUIREMENT_ID,
            TrustRoutingRequirement.CITATION,
            TrustRoutingRequirement.MECHANISM,
            status, decisions);
    }

    private GdprErasureRequirement buildGdprErasure() {
        return new GdprErasureRequirement(
            GdprErasureRequirement.REQUIREMENT_ID,
            GdprErasureRequirement.CITATION,
            GdprErasureRequirement.MECHANISM,
            true,  // erasureCapabilityWired — static at build time
            true,  // pseudonymizationActive — ActorIdentityProvider always registered
            GdprErasureRequirement.ERASURE_ENDPOINT);
    }
}
```

- [ ] **Step 7.4: Run the unit test**

```bash
mvn test -pl app -am -Dtest=AmlComplianceEvidenceServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all 5 tests PASS.

- [ ] **Step 7.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/src/main/java/io/casehub/aml/compliance/AmlComplianceEvidenceService.java \
  app/src/test/java/io/casehub/aml/compliance/AmlComplianceEvidenceServiceTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#43): AmlComplianceEvidenceService assembles requirement-scoped evidence"
```

---

## Task 8: AmlLayer7Resource + configuration

**Files:**
- Create: `app/src/main/java/io/casehub/aml/compliance/AmlLayer7Resource.java`
- Create: `app/src/test/java/io/casehub/aml/compliance/AmlLayer7ResourceTest.java`
- Create: `app/src/test/java/io/casehub/aml/compliance/AmlLayer7ErasureTest.java`

- [ ] **Step 8.1: Write the integration tests**

```java
// app/src/test/java/io/casehub/aml/compliance/AmlLayer7ResourceTest.java
package io.casehub.aml.compliance;

import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.engine.common.spi.event.WorkerDecisionEvent;
import io.casehub.ledger.runtime.service.eventlog.TestEventLogRepository;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import jakarta.inject.Inject;
import org.awaitility.Awaitility;
import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.Map;
import java.util.concurrent.TimeUnit;
import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class AmlLayer7ResourceTest {

    @Inject TestEventLogRepository eventLog;

    @Test
    void getComplianceEvidence_afterInvestigation_returnsAllRequirements() {
        // Start investigation
        String caseId = given().contentType(ContentType.JSON)
            .body(pepTransaction("TXN-L7-001"))
            .when().post("/api/layer6/investigations")
            .then().statusCode(202)
            .extract().path("caseId");

        // Wait for sar-drafting worker to be scheduled
        Awaitility.await().atMost(15, TimeUnit.SECONDS).until(() ->
            eventLog.findByCaseId(java.util.UUID.fromString(caseId)).stream()
                .anyMatch(e -> "WORKER_SCHEDULED".equals(e.eventType())
                    && "sar-drafting".equals(e.metadata().get("workerName")))
        );

        // Fetch compliance evidence
        given().when().get("/api/investigations/{caseId}/compliance-evidence", caseId)
            .then().statusCode(200)
            .body("caseId", equalTo(caseId))
            .body("generatedAt", notNullValue())
            .body("signature", nullValue())
            // Audit chain
            .body("auditChain.status", equalTo("CLOSED"))
            .body("auditChain.chainVerified", is(true))
            .body("auditChain.treeRoot", notNullValue())
            .body("auditChain.events", hasSize(2))
            .body("auditChain.events[0].eventType", equalTo("CASE_OPENED"))
            .body("auditChain.events[0].causedByEntryId", nullValue())
            .body("auditChain.events[1].eventType", equalTo("COMPLIANCE_REVIEW_OPENED"))
            .body("auditChain.events[1].causedByEntryId", notNullValue())
            .body("auditChain.events[0].inclusionProof.treeRoot",
                equalTo(given().when()
                    .get("/api/investigations/{caseId}/compliance-evidence", caseId)
                    .then().extract().path("auditChain.treeRoot")))
            // SLA
            .body("sla.status", equalTo("CLOSED"))
            .body("sla.workItemId", notNullValue())
            .body("sla.claimDeadline", notNullValue())
            .body("sla.slaMet", is(false)) // not completed yet — deadline in future
            // Trust routing
            .body("trustRouting.status", equalTo("CLOSED"))
            .body("trustRouting.decisions", not(empty()))
            .body("trustRouting.decisions[0].trustScoreAtRouting", notNullValue())
            // GDPR
            .body("gdprErasure.erasureCapabilityWired", is(true))
            .body("gdprErasure.erasureEndpoint",
                equalTo("POST /api/actors/{actorId}/erasure"));
    }

    @Test
    void getComplianceEvidence_unknownCase_returns404() {
        given().when()
            .get("/api/investigations/{caseId}/compliance-evidence",
                 java.util.UUID.randomUUID())
            .then().statusCode(404);
    }

    private SuspiciousTransaction pepTransaction(String id) {
        return new SuspiciousTransaction(id, "ACC-A", "ACC-B",
            new BigDecimal("200000"), "USD", Instant.now(), "PEP — high risk");
    }
}
```

```java
// app/src/test/java/io/casehub/aml/compliance/AmlLayer7ErasureTest.java
package io.casehub.aml.compliance;

import io.casehub.aml.domain.SuspiciousTransaction;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.awaitility.Awaitility;
import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.concurrent.TimeUnit;
import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

/**
 * Layer 7: verifies GDPR erasure endpoint wiring.
 *
 * Erases "aml-orchestrator" — the system actor that wrote CASE_OPENED and
 * COMPLIANCE_REVIEW_OPENED entries. This proves the erasure mechanism works
 * end-to-end (tokenisation enabled in test config). A proper human-actor
 * erasure scenario requires casehubio/aml#44 (AML_SAR_OFFICER_REVIEWED event).
 */
@QuarkusTest
class AmlLayer7ErasureTest {

    @Test
    void eraseActor_systemActor_returnsMappingFoundAndCount() {
        // Ensure at least one investigation has been run so aml-orchestrator has ledger entries
        String caseId = given().contentType(ContentType.JSON)
            .body(new SuspiciousTransaction("TXN-ERASE-001", "ACC-A", "ACC-B",
                new BigDecimal("50000"), "USD", Instant.now(), "Structuring"))
            .when().post("/api/layer6/investigations")
            .then().statusCode(202)
            .extract().path("caseId");

        // Wait for investigation to write ledger entries
        Awaitility.await().atMost(10, TimeUnit.SECONDS).until(() ->
            given().when()
                .get("/api/investigations/{caseId}/compliance-evidence", caseId)
                .then().extract().statusCode() == 200
        );

        // Erase — tokenisation is enabled in test config
        given().when()
            .post("/api/actors/{actorId}/erasure", "aml-orchestrator")
            .then().statusCode(200)
            .body("rawActorId", equalTo("aml-orchestrator"))
            .body("mappingFound", is(true))
            .body("affectedEntryCount", greaterThanOrEqualTo(2));
    }

    @Test
    void eraseActor_unknownActor_returnsMappingFalse() {
        given().when()
            .post("/api/actors/{actorId}/erasure", "actor-that-does-not-exist")
            .then().statusCode(200)
            .body("mappingFound", is(false))
            .body("affectedEntryCount", equalTo(0));
    }
}
```

- [ ] **Step 8.2: Run tests to confirm they fail**

```bash
mvn verify -pl app -am -Dtest="AmlLayer7ResourceTest,AmlLayer7ErasureTest" -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: FAIL — endpoints don't exist yet.

- [ ] **Step 8.3: Implement AmlLayer7Resource**

```java
// app/src/main/java/io/casehub/aml/compliance/AmlLayer7Resource.java
package io.casehub.aml.compliance;

import io.casehub.ledger.runtime.privacy.LedgerErasureService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import java.util.UUID;

@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class AmlLayer7Resource {

    @Inject AmlComplianceEvidenceService evidenceService;
    @Inject LedgerErasureService erasureService;

    @GET
    @Path("/api/investigations/{caseId}/compliance-evidence")
    public Response getComplianceEvidence(@PathParam("caseId") UUID caseId) {
        return evidenceService.findEvidence(caseId)
            .map(e -> Response.ok(e).build())
            .orElse(Response.status(Response.Status.NOT_FOUND).build());
    }

    @POST
    @Path("/api/actors/{actorId}/erasure")
    public LedgerErasureService.ErasureResult eraseActor(
            @PathParam("actorId") String actorId) {
        return erasureService.erase(actorId);
    }
}
```

- [ ] **Step 8.4: Run integration tests**

```bash
mvn verify -pl app -am -Dtest="AmlLayer7ResourceTest,AmlLayer7ErasureTest" -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all tests PASS.

If `AmlLayer7ResourceTest` fails with Merkle chain errors, check that `JpaLedgerMerkleFrontierRepository` was added to `selected-alternatives` in Task 3. If `AmlLayer7ErasureTest` fails with `mappingFound = false`, check that `casehub.ledger.identity.tokenisation.enabled=true` was added to `application.properties` in Task 3.

- [ ] **Step 8.5: Run full test suite to check for regressions**

```bash
mvn verify -pl api,app -am -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all tests PASS.

- [ ] **Step 8.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/src/main/java/io/casehub/aml/compliance/AmlLayer7Resource.java \
  app/src/test/java/io/casehub/aml/compliance/AmlLayer7ResourceTest.java \
  app/src/test/java/io/casehub/aml/compliance/AmlLayer7ErasureTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#43): AmlLayer7Resource — compliance evidence and erasure endpoints"
```

---

## Task 9: Complete LAYER-LOG.md Layer 7 entry

**Files:**
- Modify: `LAYER-LOG.md` in the project repo (`/Users/mdproctor/claude/casehub/aml/LAYER-LOG.md`)

- [ ] **Step 9.1: Fill in the Layer 7 entry**

Replace the current Layer 7 stub section with:

```markdown
## Layer 7 — Compliance evidence (accountability properties mapped against FinCEN/FATF)

**Participates in:** S5 **Issue:** casehubio/aml#43 **Navigation:** `git log --grep="#43" --oneline`
**Spec:** workspace `specs/2026-05-30-layer7-compliance-evidence-design.md`
**Completed:** 2026-05-30

### What it adds

Structured evidence that each FinCEN/FATF accountability property is met, surfaced via
`GET /api/investigations/{caseId}/compliance-evidence`. Evidence is requirement-scoped (not layer-scoped) and includes
Merkle inclusion proofs per ledger event — independently verifiable without trusting the service.

Also wires GDPR erasure via `POST /api/actors/{actorId}/erasure` (`LedgerErasureService`)
and captures trust scores at routing time (`AmlTrustRoutingAttestation`) before cache drift.

**Prerequisite fix:** `writeComplianceReviewOpened` now derives `causedByEntryId` internally by querying for
`CASE_OPENED` — works for both the synchronous Layer 3 and async Layer 5 paths.

### Accountability properties delivered

| FinCEN/FATF requirement | Mechanism | Evidence artifact |
|---|---|---|
| 31 CFR §1020.320(a) — auditable evidence chain | Layers 3+4: qhorus COMMAND/DONE/DECLINE + Merkle-chained LedgerEntry | Inclusion proof per ledger event; `causedByEntryId` chain; `chainVerified` boolean |
| 31 CFR §1020.320(b)(3) — 30-day SAR human sign-off | Layer 2: casehub-work WorkItem with `claimDeadline` | WorkItem `claimDeadline`, `completedAt`, `candidateGroups`; `status: CLOSED/BREACHED` |
| FATF R.20 — experienced analysts on complex cases | Layer 6: `TrustWeightedAgentStrategy` + Layer 7: `AmlTrustRoutingAttestation` | `trustScoreAtRouting` captured at `WorkerDecisionEvent` time per capability |
| GDPR Art.17 / FATF privacy — PII erasure preserving audit structure | Layer 7: `LedgerErasureService` wired | Erasure endpoint; `ActorIdentity` pseudonymization; audit entries intact |

### Scope constraint

`AuditChainRequirement` covers `AmlInvestigationLedgerEntry` records only (case lifecycle events). Specialist dispatch
audit (COMMAND/DONE/DECLINE per agent) lives in the qhorus
`MessageLedgerEntry` chain — same `subjectId` but a separate Merkle tree. Stated explicitly in the `mechanism` field so
examiners have accurate expectations.

### Open items

- casehubio/engine#403: add `trustScoreAtRouting` to `WorkerDecisionEntry` (makes `AmlTrustRoutingAttestation`
  redundant)
- casehubio/work#241: public `WorkItemEntity` read API (removes `em.find` workaround)
- casehubio/aml#44: observer failure reconciliation (silent evidence gaps under DB errors)
- GDPR proper data-subject demo: add `AML_SAR_OFFICER_REVIEWED` event with human `actorId`

### Key wiring

**`LedgerVerificationService` in tests.** `JpaLedgerMerkleFrontierRepository` is `@Alternative`
and must be added to `quarkus.arc.selected-alternatives` in test `application.properties`
alongside `JpaLedgerEntryRepository`. Without it, CDI fails to inject
`LedgerMerkleFrontierRepository` into `LedgerVerificationService`.

**Tokenisation enabled in tests.** `casehub.ledger.identity.tokenisation.enabled=true` is required in test
`application.properties` for `LedgerErasureService.erase()` to create
`ActorIdentity` rows and return `mappingFound = true`.

**`WorkerDecisionEvent` observation.** `@Observes WorkerDecisionEvent` with
`@Transactional(REQUIRES_NEW)` gives each attestation its own transaction, decoupled from the engine worker's
transaction. Sequence number assignment uses `findLatestBySubjectId` which is not concurrent-safe under parallel worker
dispatch (pattern-analysis + osint-screening fire simultaneously in Layer 5). Acceptable for tutorial; tracked as
casehubio/aml#44.

**`WorkItemEntity` lookup.** `em.find(WorkItem.class, taskId)` using the default `EntityManager` —
`WorkItemEntity` is on the default datasource. `WorkItemStore` is an internal class; no public query API exists in
casehub-work-api (casehubio/work#241).

### Pattern to replicate (in another domain)

1. Write `XxxComplianceEvidenceService` that assembles one record per regulatory requirement
2. For each requirement with a Merkle-provable artifact: call `LedgerVerificationService.inclusionProof(entryId)` per
   entry, project to domain API types
3. For SLA: find the WorkItem task ID from the ledger entry, fetch via `em.find`
4. For trust routing: observe `WorkerDecisionEvent`, write a domain ledger attestation capturing score at routing time
5. Expose `GET /api/{domain}/compliance-evidence` and `POST /api/actors/{actorId}/erasure`
6. In test `application.properties`: enable `JpaLedgerMerkleFrontierRepository` as selected alternative and
   `casehub.ledger.identity.tokenisation.enabled=true`
```

- [ ] **Step 9.2: Commit LAYER-LOG**

```bash
git -C /Users/mdproctor/claude/casehub/aml add LAYER-LOG.md
git -C /Users/mdproctor/claude/casehub/aml commit -m "docs(#43): complete Layer 7 LAYER-LOG entry"
```

---

## Task 10: Final verification

- [ ] **Step 10.1: Run the complete test suite**

```bash
mvn verify -pl api,app -am -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: BUILD SUCCESS, all tests PASS.

- [ ] **Step 10.2: Check for any IDE diagnostics**

Use IntelliJ inspection or `get_file_problems` on the new files — resolve any errors before committing.

- [ ] **Step 10.3: Update vertical slice S5 in LAYER-LOG.md**

In the Vertical Slice Index table, change `S5` status from `🔲 pending` to `✅ complete`.

```bash
git -C /Users/mdproctor/claude/casehub/aml add LAYER-LOG.md
git -C /Users/mdproctor/claude/casehub/aml commit -m "docs(#43): mark S5 complete in vertical slice index"
```

---

## Self-Review Notes

**ProofStep fields:** `InclusionProof.siblings()` returns `List<ProofStep>`. `ProofStep` is referenced as having `.hash()` and `.position()` in `buildInclusionProof`. Confirm the exact method names on `ProofStep` from the decompiled ledger class before Task 7. If names differ, adjust the projection in `AmlComplianceEvidenceService.buildInclusionProof`.

**`slaMet` semantics:** Task 7 sets `slaMet = false` when the WorkItem is not yet completed even if the deadline is still in the future (tutorial state). The integration test asserts `slaMet: false` for this reason. This is correct — `slaMet` is only `true` when `completedAt != null && completedAt.before(claimDeadline)`.

**`TestEventLogRepository` import:** Used in `AmlLayer7ResourceTest` for Awaitility polling. If the import path differs from the Layer 5/6 tests, copy the import from `AmlLayer5InvestigationTest` or `AmlLayer6InvestigationTest`.

**`WorkItem.candidateGroups` type:** Assumed to be `String` (comma-separated) based on `WorkItemService.create()` which stores it directly. Adjust the `groups` parsing in `AmlComplianceEvidenceService.buildSla()` if the actual type is `List<String>`.
