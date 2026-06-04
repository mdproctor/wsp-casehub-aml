# Layer 8: CaseMemoryStore Integration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire `CaseMemoryStore` as a pre-case intelligence layer — store investigation facts as cases progress, inject prior entity context before each new case starts, and enrich the YAML binding so known-high-risk entities route to senior analysts immediately.

**Architecture:** Three platform memory adapters already exist (`memory-inmem`, `memory-jpa`, `memory-sqlite`) behind the `CaseMemoryStore` SPI. AML adds: a domain constants class (`AmlMemoryDomains`), a central service (`AmlMemoryService`), a context value record (`AmlPriorContext`), a CDI event for SAR outcomes (`SarOutcomeRecordedEvent`), and a SAR memory observer (`AmlSarOutcomeMemoryObserver`). The ledger subclass hierarchy is redesigned from dual-use to two dedicated siblings. `PushAgentDispatch` is fixed to pass the actual qhorus `Message` entity to behaviours (previously always null).

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-platform `CaseMemoryStore` SPI, `casehub-platform-memory-jpa` (PostgreSQL prod), `casehub-platform-memory-inmem` (test isolation), JPA JOINED inheritance, CDI events, Flyway V2007.

**Spec:** `specs/2026-06-03-case-memory-store-design.md`

**Pre-verified:** Merkle hash (`LedgerMerkleTree.canonicalBytes`) covers only six base `LedgerEntry` fields — JOIN table changes in subclasses do not affect hash integrity.

---

## File Map

### New files (app module)

| File | Responsibility |
|---|---|
| `app/src/main/java/io/casehub/aml/ledger/AmlCaseOpenedLedgerEntry.java` | CASE_OPENED ledger subclass — transactionId, originAccountId, destinationAccountId |
| `app/src/main/java/io/casehub/aml/ledger/AmlComplianceReviewLedgerEntry.java` | COMPLIANCE_REVIEW_OPENED ledger subclass — taskId |
| `app/src/main/resources/db/aml-ledger/migration/V2007__aml_ledger_subclass_split.sql` | Drop aml_investigation_ledger_entry; create two new join tables |
| `app/src/main/java/io/casehub/aml/engine/SarOutcomeRecordedEvent.java` | CDI event record — caseId + outcome |
| `app/src/main/java/io/casehub/aml/memory/AmlMemoryDomains.java` | MemoryDomain constants: ENTITY_RISK, NETWORK, PATTERN |
| `app/src/main/java/io/casehub/aml/memory/AmlMemoryPolicyKeys.java` | PreferenceKey for lookback window (default 365 days) |
| `app/src/main/java/io/casehub/aml/memory/AmlPriorContext.java` | Value record with hasHistory(), isKnownHighRisk(), toContextMap() |
| `app/src/main/java/io/casehub/aml/memory/AmlMemoryService.java` | Central service — all CaseMemoryStore interactions for AML |
| `app/src/main/java/io/casehub/aml/memory/AmlSarOutcomeMemoryObserver.java` | CDI observer — writes SAR outcome memories under both account IDs |
| `app/src/test/java/io/casehub/aml/memory/AmlMemoryDomainsTest.java` | Unit: domain name string constants |
| `app/src/test/java/io/casehub/aml/memory/AmlPriorContextTest.java` | Unit: isKnownHighRisk threshold, recency, reversal, toContextMap shape |
| `app/src/test/java/io/casehub/aml/memory/AmlMemoryServiceTest.java` | Unit: text formatting, attribute completeness, failure guard |
| `app/src/test/java/io/casehub/aml/memory/AmlMemoryRoundtripTest.java` | @QuarkusTest: store/query roundtrip, both-accounts, erasure |
| `app/src/test/java/io/casehub/aml/memory/AmlSarOutcomeObserverTest.java` | @QuarkusTest: CDI event fires both observers, WITHDRAWN reversal |
| `app/src/test/java/io/casehub/aml/memory/AmlLayer8RoutingTest.java` | @QuarkusTest: prior-context routing, dedup, no-history, low-confidence |

### Modified files

| File | Change |
|---|---|
| `app/src/main/java/io/casehub/aml/ledger/AmlLedgerService.java` | Use AmlCaseOpenedLedgerEntry + AmlComplianceReviewLedgerEntry |
| `app/src/main/java/io/casehub/aml/agents/PushAgentDispatch.java` | Pass the fetched Message entity to behaviour.handle() instead of null |
| `app/src/main/java/io/casehub/aml/agents/EntityResolutionBehaviour.java` | Inject AmlMemoryService; deserialize transaction from command; store entity risk + network |
| `app/src/main/java/io/casehub/aml/agents/PatternAnalysisBehaviour.java` | Inject AmlMemoryService; deserialize transaction; store pattern findings |
| `app/src/main/java/io/casehub/aml/engine/AmlEngineCoordinator.java` | Inject AmlMemoryService; query prior context; add to initialContext |
| `app/src/main/java/io/casehub/aml/engine/AmlLayer6Resource.java` | Remove direct SarOutcomeFeedbackService call; fire SarOutcomeRecordedEvent |
| `app/src/main/java/io/casehub/aml/trust/SarOutcomeFeedbackService.java` | Add @Observes onSarOutcome() — keep recordOutcome() for direct test calls |
| `app/src/main/resources/aml/aml-investigation.yaml` | Merge senior-analyst-required to evaluate priorEntityContext.knownHighRisk |
| `app/src/main/resources/application.properties` | Add db/memory/migration Flyway location; add memory.jpa Hibernate package |
| `app/src/test/resources/application.properties` | Same + add InMemoryMemoryStore to selected-alternatives + Jandex entries |
| `app/pom.xml` | Add casehub-platform-memory-jpa (compile) + casehub-platform-memory-inmem (test) |
| `app/src/test/java/io/casehub/aml/ledger/AmlLedgerChainTest.java` | Update instanceof casts to AmlCaseOpenedLedgerEntry / AmlComplianceReviewLedgerEntry |

### Deleted files

| File | Reason |
|---|---|
| `app/src/main/java/io/casehub/aml/ledger/AmlInvestigationLedgerEntry.java` | Replaced by two sibling subclasses |

---

## Task 1: Ledger subclass redesign

**Files:**
- Create: `app/src/main/java/io/casehub/aml/ledger/AmlCaseOpenedLedgerEntry.java`
- Create: `app/src/main/java/io/casehub/aml/ledger/AmlComplianceReviewLedgerEntry.java`
- Create: `app/src/main/resources/db/aml-ledger/migration/V2007__aml_ledger_subclass_split.sql`
- Modify: `app/src/main/java/io/casehub/aml/ledger/AmlLedgerService.java`
- Modify: `app/src/test/java/io/casehub/aml/ledger/AmlLedgerChainTest.java`
- Delete: `app/src/main/java/io/casehub/aml/ledger/AmlInvestigationLedgerEntry.java`

The existing `AmlInvestigationLedgerEntry` mixes CASE_OPENED and COMPLIANCE_REVIEW_OPENED in one table via an `eventType` string — a dual-use design its own Javadoc flagged. Replace with two dedicated sibling subclasses. Merkle hash only covers base `LedgerEntry` fields (confirmed in `LedgerMerkleTree.canonicalBytes`) — the JOIN table change is safe.

- [ ] **Step 1: Write the new ledger entry subclasses**

Create `app/src/main/java/io/casehub/aml/ledger/AmlCaseOpenedLedgerEntry.java`:

```java
package io.casehub.aml.ledger;

import io.casehub.ledger.runtime.model.LedgerEntry;
import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

@Entity
@Table(name = "aml_case_opened_ledger_entry")
@DiscriminatorValue("AML_CASE_OPENED")
public class AmlCaseOpenedLedgerEntry extends LedgerEntry {

    @Column(name = "transaction_id", nullable = false, length = 255)
    public String transactionId;

    @Column(name = "origin_account_id", nullable = false, length = 255)
    public String originAccountId;

    @Column(name = "destination_account_id", nullable = false, length = 255)
    public String destinationAccountId;
}
```

Create `app/src/main/java/io/casehub/aml/ledger/AmlComplianceReviewLedgerEntry.java`:

```java
package io.casehub.aml.ledger;

import io.casehub.ledger.runtime.model.LedgerEntry;
import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

@Entity
@Table(name = "aml_compliance_review_ledger_entry")
@DiscriminatorValue("AML_COMPLIANCE_REVIEW")
public class AmlComplianceReviewLedgerEntry extends LedgerEntry {

    @Column(name = "task_id", nullable = false, length = 255)
    public String taskId;
}
```

- [ ] **Step 2: Write Flyway migration V2007**

Create `app/src/main/resources/db/aml-ledger/migration/V2007__aml_ledger_subclass_split.sql`:

```sql
-- V2007: replace dual-use aml_investigation_ledger_entry with two dedicated sibling tables.
-- Discriminator values: AML_CASE_OPENED and AML_COMPLIANCE_REVIEW.
-- V2001 introduced the original table (dropped here). V2007 is the next available
-- number in the shared qhorus datasource version namespace (V2002–V2006 are in
-- aml-engine-ledger and aml-trust-routing migration paths).

DROP TABLE IF EXISTS aml_investigation_ledger_entry;

CREATE TABLE aml_case_opened_ledger_entry (
    id                     UUID         NOT NULL,
    transaction_id         VARCHAR(255) NOT NULL,
    origin_account_id      VARCHAR(255) NOT NULL,
    destination_account_id VARCHAR(255) NOT NULL,
    CONSTRAINT pk_aml_case_opened PRIMARY KEY (id),
    CONSTRAINT fk_aml_case_opened FOREIGN KEY (id) REFERENCES ledger_entry (id)
);

CREATE INDEX idx_aml_case_opened_id ON aml_case_opened_ledger_entry (id);

CREATE TABLE aml_compliance_review_ledger_entry (
    id      UUID         NOT NULL,
    task_id VARCHAR(255) NOT NULL,
    CONSTRAINT pk_aml_compliance_review PRIMARY KEY (id),
    CONSTRAINT fk_aml_compliance_review FOREIGN KEY (id) REFERENCES ledger_entry (id)
);
```

- [ ] **Step 3: Update AmlLedgerService to use the new subclasses**

Replace `app/src/main/java/io/casehub/aml/ledger/AmlLedgerService.java`:

```java
package io.casehub.aml.ledger;

import java.time.Instant;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;

@ApplicationScoped
public class AmlLedgerService {

    private static final String ACTOR_ID = "aml-orchestrator";
    private static final String ACTOR_ROLE = "AmlInvestigationOrchestrator";

    @Inject
    LedgerEntryRepository repository;

    public UUID writeCaseOpened(final SuspiciousTransaction transaction, final UUID caseId) {
        final int sequenceNumber = nextSequenceNumber(caseId);
        final AmlCaseOpenedLedgerEntry entry = new AmlCaseOpenedLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = caseId;
        entry.sequenceNumber = sequenceNumber;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = ACTOR_ID;
        entry.actorType = ActorType.SYSTEM;
        entry.actorRole = ACTOR_ROLE;
        entry.occurredAt = Instant.now();
        entry.transactionId = transaction.id();
        entry.originAccountId = transaction.originAccountId();
        entry.destinationAccountId = transaction.destinationAccountId();
        repository.save(entry);
        return entry.id;
    }

    public void writeComplianceReviewOpened(final UUID caseId, final String taskId) {
        final UUID caseOpenedEntryId = repository.findBySubjectId(caseId).stream()
                .filter(AmlCaseOpenedLedgerEntry.class::isInstance)
                .map(e -> e.id)
                .findFirst()
                .orElse(null);
        final int sequenceNumber = nextSequenceNumber(caseId);
        final AmlComplianceReviewLedgerEntry entry = new AmlComplianceReviewLedgerEntry();
        entry.id = UUID.randomUUID();
        entry.subjectId = caseId;
        entry.sequenceNumber = sequenceNumber;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = ACTOR_ID;
        entry.actorType = ActorType.SYSTEM;
        entry.actorRole = ACTOR_ROLE;
        entry.occurredAt = Instant.now();
        entry.taskId = taskId;
        entry.causedByEntryId = caseOpenedEntryId;
        repository.save(entry);
    }

    private int nextSequenceNumber(final UUID subjectId) {
        return repository.findLatestBySubjectId(subjectId)
                .map(e -> e.sequenceNumber + 1)
                .orElse(1);
    }

    public static AmlLedgerService noOp() {
        return new AmlLedgerService() {
            @Override public UUID writeCaseOpened(SuspiciousTransaction tx, UUID caseId) { return UUID.randomUUID(); }
            @Override public void writeComplianceReviewOpened(UUID caseId, String taskId) {}
        };
    }

    public static AmlLedgerService stub(final UUID entryId) {
        return new AmlLedgerService() {
            @Override public UUID writeCaseOpened(SuspiciousTransaction tx, UUID caseId) { return entryId; }
            @Override public void writeComplianceReviewOpened(UUID caseId, String taskId) {}
        };
    }
}
```

- [ ] **Step 4: Delete the old dual-use class**

Delete `app/src/main/java/io/casehub/aml/ledger/AmlInvestigationLedgerEntry.java`.

- [ ] **Step 5: Fix AmlLedgerChainTest — update casts**

In `app/src/test/java/io/casehub/aml/ledger/AmlLedgerChainTest.java`, update all references:

```java
// Change:
assertEquals("CASE_OPENED", ((AmlInvestigationLedgerEntry) entry).eventType);
// To:
assertTrue(entry instanceof AmlCaseOpenedLedgerEntry,
    "Entry must be AmlCaseOpenedLedgerEntry");

// Change:
assertEquals(result.caseId(), ((AmlInvestigationLedgerEntry) entry.get()).subjectId, ...);
// To:
assertEquals(result.caseId(), entry.get().subjectId, ...);

// Change the COMPLIANCE_REVIEW_OPENED check:
boolean hasReviewEntry = entries.stream()
        .anyMatch(e -> e instanceof AmlInvestigationLedgerEntry ale
                       && "COMPLIANCE_REVIEW_OPENED".equals(ale.eventType));
// To:
boolean hasReviewEntry = entries.stream()
        .anyMatch(e -> e instanceof AmlComplianceReviewLedgerEntry);
```

Also add imports for `AmlCaseOpenedLedgerEntry` and `AmlComplianceReviewLedgerEntry`; remove `AmlInvestigationLedgerEntry` import.

- [ ] **Step 6: Run the ledger tests**

```bash
mvn -pl app -am test -Dtest="AmlLedgerChainTest,AmlLedgerCausedByTest" -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all tests PASS. If CASE_OPENED entry is not found by `instanceof`, check the discriminator value matches the `@DiscriminatorValue` annotation.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/src/main/java/io/casehub/aml/ledger/AmlCaseOpenedLedgerEntry.java \
  app/src/main/java/io/casehub/aml/ledger/AmlComplianceReviewLedgerEntry.java \
  app/src/main/resources/db/aml-ledger/migration/V2007__aml_ledger_subclass_split.sql \
  app/src/main/java/io/casehub/aml/ledger/AmlLedgerService.java \
  app/src/test/java/io/casehub/aml/ledger/AmlLedgerChainTest.java
git -C /Users/mdproctor/claude/casehub/aml rm \
  app/src/main/java/io/casehub/aml/ledger/AmlInvestigationLedgerEntry.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "refactor(issue-32): split AmlInvestigationLedgerEntry into two sibling subclasses

Refs #32"
```

---

## Task 2: SarOutcomeRecordedEvent CDI infrastructure

**Files:**
- Create: `app/src/main/java/io/casehub/aml/engine/SarOutcomeRecordedEvent.java`
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlLayer6Resource.java`
- Modify: `app/src/main/java/io/casehub/aml/trust/SarOutcomeFeedbackService.java`
- Test: `app/src/test/java/io/casehub/aml/trust/SarOutcomeFeedbackServiceTest.java` (unchanged — calls recordOutcome() directly, still valid)
- Test: `app/src/test/java/io/casehub/aml/engine/AmlLayer6ResourceTest.java` (unchanged — fires POST /outcome, still returns 204)

- [ ] **Step 1: Write the CDI event record**

Create `app/src/main/java/io/casehub/aml/engine/SarOutcomeRecordedEvent.java`:

```java
package io.casehub.aml.engine;

import io.casehub.aml.domain.SarOutcome;
import java.util.UUID;

public record SarOutcomeRecordedEvent(UUID caseId, SarOutcome outcome) {}
```

- [ ] **Step 2: Add @Observes to SarOutcomeFeedbackService**

In `app/src/main/java/io/casehub/aml/trust/SarOutcomeFeedbackService.java`, add:

```java
import io.casehub.aml.engine.SarOutcomeRecordedEvent;
import jakarta.enterprise.event.Observes;

// Add this new method (keep existing recordOutcome() unchanged for direct testability):
public void onSarOutcome(@Observes SarOutcomeRecordedEvent event) {
    recordOutcome(event.caseId(), event.outcome());
}
```

The existing `recordOutcome(UUID, SarOutcome)` method stays exactly as-is — tests still call it directly.

- [ ] **Step 3: Update AmlLayer6Resource to fire the event**

In `app/src/main/java/io/casehub/aml/engine/AmlLayer6Resource.java`, replace the direct `feedbackService` injection and call:

```java
// Remove:
@Inject SarOutcomeFeedbackService feedbackService;

// Add:
@Inject jakarta.enterprise.event.Event<SarOutcomeRecordedEvent> sarOutcomeEvent;

// Replace recordOutcome() body:
@POST
@Path("/{caseId}/outcome")
public Response recordOutcome(
        @PathParam("caseId") UUID caseId,
        final SarOutcomeRequest request) {
    final SarOutcome outcome = new SarOutcome(
            request.verdict(), request.reason(), request.investigationAccuracyScore());
    sarOutcomeEvent.fire(new SarOutcomeRecordedEvent(caseId, outcome));
    return Response.noContent().build();
}
```

- [ ] **Step 4: Run affected tests**

```bash
mvn -pl app -am test -Dtest="SarOutcomeFeedbackServiceTest,AmlLayer6ResourceTest" -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: both test classes PASS. `SarOutcomeFeedbackServiceTest` calls `recordOutcome()` directly (unchanged). `AmlLayer6ResourceTest.post_outcome_returns_204` now fires the CDI event but still returns 204.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/src/main/java/io/casehub/aml/engine/SarOutcomeRecordedEvent.java \
  app/src/main/java/io/casehub/aml/engine/AmlLayer6Resource.java \
  app/src/main/java/io/casehub/aml/trust/SarOutcomeFeedbackService.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-32): SarOutcomeRecordedEvent CDI event — decouple resource from observers

Refs #32"
```

---

## Task 3: Memory adapter dependencies and configuration

**Files:**
- Modify: `app/pom.xml`
- Modify: `app/src/main/resources/application.properties`
- Modify: `app/src/test/resources/application.properties`

- [ ] **Step 1: Add Maven dependencies**

In `app/pom.xml`, inside `<dependencies>`, add after the existing platform deps:

```xml
<!-- Layer 8: JPA-backed CaseMemoryStore (prod) — @ApplicationScoped, displaces NoOpCaseMemoryStore -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-memory-jpa</artifactId>
  <version>${casehub.version}</version>
</dependency>

<!-- Layer 8: in-memory CaseMemoryStore (test isolation) — @Alternative @Priority(1) -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-memory-inmem</artifactId>
  <version>${casehub.version}</version>
  <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Update main application.properties**

In `app/src/main/resources/application.properties`:

```properties
# Change quarkus.flyway.locations — add db/memory/migration:
quarkus.flyway.locations=classpath:db/work/migration,classpath:db/memory/migration

# Change quarkus.hibernate-orm.packages — add io.casehub.platform.memory.jpa:
quarkus.hibernate-orm.packages=io.casehub.work.runtime.model,io.casehub.work.runtime.filter,io.casehub.aml.domain,io.casehub.platform.memory.jpa
```

- [ ] **Step 3: Update test application.properties**

In `app/src/test/resources/application.properties`:

```properties
# Change quarkus.flyway.locations — add db/memory/migration:
quarkus.flyway.locations=classpath:db/work/migration,classpath:db/memory/migration

# Change quarkus.hibernate-orm.packages — add io.casehub.platform.memory.jpa:
quarkus.hibernate-orm.packages=io.casehub.work.runtime.model,io.casehub.work.runtime.filter,io.casehub.aml.domain,io.casehub.platform.memory.jpa

# Change quarkus.arc.selected-alternatives — add InMemoryMemoryStore:
quarkus.arc.selected-alternatives=io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,io.casehub.ledger.runtime.repository.jpa.JpaLedgerMerkleFrontierRepository,io.casehub.platform.memory.inmem.InMemoryMemoryStore

# Add Jandex entries for memory modules (both have embedded indices, but explicit is safer):
quarkus.index-dependency.casehub-platform-memory-jpa.group-id=io.casehub
quarkus.index-dependency.casehub-platform-memory-jpa.artifact-id=casehub-platform-memory-jpa
quarkus.index-dependency.casehub-platform-memory-inmem.group-id=io.casehub
quarkus.index-dependency.casehub-platform-memory-inmem.artifact-id=casehub-platform-memory-inmem
```

- [ ] **Step 4: Verify build starts with new config**

```bash
mvn -pl app -am test -Dtest=AmlInvestigationResourceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: test PASSES. If Flyway fails (V1000 conflict), check that the default datasource only has `db/work/migration` and `db/memory/migration` — no ledger locations on this datasource. If CDI fails to resolve `CaseMemoryStore`, check that `InMemoryMemoryStore` is in `selected-alternatives`.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/pom.xml \
  app/src/main/resources/application.properties \
  app/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-32): add memory adapter dependencies and config

Refs #32"
```

---

## Task 4: AmlMemoryDomains + AmlMemoryPolicyKeys + unit tests

**Files:**
- Create: `app/src/main/java/io/casehub/aml/memory/AmlMemoryDomains.java`
- Create: `app/src/main/java/io/casehub/aml/memory/AmlMemoryPolicyKeys.java`
- Create: `app/src/test/java/io/casehub/aml/memory/AmlMemoryDomainsTest.java`

- [ ] **Step 1: Write the failing domain constants test**

Create `app/src/test/java/io/casehub/aml/memory/AmlMemoryDomainsTest.java`:

```java
package io.casehub.aml.memory;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class AmlMemoryDomainsTest {

    @Test
    void entityRiskDomainName() {
        assertEquals("aml.entity-risk", AmlMemoryDomains.ENTITY_RISK.name());
    }

    @Test
    void networkDomainName() {
        assertEquals("aml.network", AmlMemoryDomains.NETWORK.name());
    }

    @Test
    void patternDomainName() {
        assertEquals("aml.pattern", AmlMemoryDomains.PATTERN.name());
    }

    @Test
    void domainsAreDistinct() {
        assertNotEquals(AmlMemoryDomains.ENTITY_RISK, AmlMemoryDomains.NETWORK);
        assertNotEquals(AmlMemoryDomains.NETWORK, AmlMemoryDomains.PATTERN);
        assertNotEquals(AmlMemoryDomains.ENTITY_RISK, AmlMemoryDomains.PATTERN);
    }
}
```

- [ ] **Step 2: Run — expect compilation failure**

```bash
mvn -pl app test -Dtest=AmlMemoryDomainsTest 2>&1 | grep -E "FAIL|ERROR|cannot find"
```

Expected: compilation error — `AmlMemoryDomains` does not exist.

- [ ] **Step 3: Write AmlMemoryDomains**

Create `app/src/main/java/io/casehub/aml/memory/AmlMemoryDomains.java`:

```java
package io.casehub.aml.memory;

import io.casehub.platform.api.memory.MemoryDomain;

public final class AmlMemoryDomains {

    public static final MemoryDomain ENTITY_RISK = new MemoryDomain("aml.entity-risk");
    public static final MemoryDomain NETWORK     = new MemoryDomain("aml.network");
    public static final MemoryDomain PATTERN     = new MemoryDomain("aml.pattern");

    private AmlMemoryDomains() {}
}
```

- [ ] **Step 4: Write AmlMemoryPolicyKeys**

Create `app/src/main/java/io/casehub/aml/memory/AmlMemoryPolicyKeys.java`:

```java
package io.casehub.aml.memory;

import io.casehub.aml.routing.IntPreference;
import io.casehub.platform.api.preferences.PreferenceKey;

public final class AmlMemoryPolicyKeys {

    /**
     * Lookback window for entity-risk memory queries. Memories older than this are
     * excluded from isKnownHighRisk() evaluation to prevent stale classifications
     * from driving senior analyst routing. Default: 365 days.
     */
    public static final PreferenceKey<IntPreference> ENTITY_RISK_LOOKBACK_DAYS =
        new PreferenceKey<>("casehubio.aml.memory", "entity-risk-lookback-days",
            IntPreference.of(365), IntPreference::parse);

    private AmlMemoryPolicyKeys() {}
}
```

- [ ] **Step 5: Run domains test**

```bash
mvn -pl app -am test -Dtest=AmlMemoryDomainsTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all 4 tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/src/main/java/io/casehub/aml/memory/AmlMemoryDomains.java \
  app/src/main/java/io/casehub/aml/memory/AmlMemoryPolicyKeys.java \
  app/src/test/java/io/casehub/aml/memory/AmlMemoryDomainsTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-32): AmlMemoryDomains + AmlMemoryPolicyKeys constants

Refs #32"
```

---

## Task 5: AmlPriorContext + unit tests

**Files:**
- Create: `app/src/main/java/io/casehub/aml/memory/AmlPriorContext.java`
- Create: `app/src/test/java/io/casehub/aml/memory/AmlPriorContextTest.java`

- [ ] **Step 1: Write failing AmlPriorContext tests**

Create `app/src/test/java/io/casehub/aml/memory/AmlPriorContextTest.java`:

```java
package io.casehub.aml.memory;

import io.casehub.platform.api.memory.Memory;
import io.casehub.platform.api.memory.MemoryAttributeKeys;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class AmlPriorContextTest {

    private Memory memory(String entityId, double confidence, Instant createdAt) {
        return new Memory(UUID.randomUUID().toString(), entityId, AmlMemoryDomains.ENTITY_RISK,
            "tenant-1", null, "Entity " + entityId + " is high risk.",
            Map.of(MemoryAttributeKeys.CONFIDENCE, MemoryAttributeKeys.formatConfidence(confidence)),
            createdAt);
    }

    @Test
    void empty_hasHistoryFalse() {
        AmlPriorContext ctx = AmlPriorContext.empty();
        assertFalse(ctx.hasHistory());
    }

    @Test
    void nonEmptyEntityRisk_hasHistoryTrue() {
        AmlPriorContext ctx = new AmlPriorContext(
            List.of(memory("ACC-A", 0.5, Instant.now())),
            List.of(), List.of());
        assertTrue(ctx.hasHistory());
    }

    @Test
    void confidence_aboveThreshold_isKnownHighRisk() {
        AmlPriorContext ctx = new AmlPriorContext(
            List.of(memory("ACC-A", 0.8, Instant.now())),
            List.of(), List.of());
        assertTrue(ctx.isKnownHighRisk());
    }

    @Test
    void confidence_belowThreshold_notKnownHighRisk() {
        AmlPriorContext ctx = new AmlPriorContext(
            List.of(memory("ACC-A", 0.79, Instant.now())),
            List.of(), List.of());
        assertFalse(ctx.isKnownHighRisk());
    }

    @Test
    void isKnownHighRisk_usesOnlyMostRecentPerEntity() {
        // Older UPHELD (0.9) followed by newer WITHDRAWN reversal (0.0) → false
        Instant older = Instant.now().minusSeconds(100);
        Instant newer = Instant.now();
        Memory upheld    = memory("ACC-A", 0.9, older);
        Memory withdrawn = memory("ACC-A", 0.0, newer);
        AmlPriorContext ctx = new AmlPriorContext(
            List.of(upheld, withdrawn), List.of(), List.of());
        assertFalse(ctx.isKnownHighRisk(),
            "WITHDRAWN reversal is most recent — must return false");
    }

    @Test
    void isKnownHighRisk_usesOnlyMostRecentPerEntity_oldReversalDoesNotBlock() {
        // Older WITHDRAWN (0.0) followed by newer UPHELD (0.9) → true
        Instant older = Instant.now().minusSeconds(100);
        Instant newer = Instant.now();
        Memory withdrawn = memory("ACC-A", 0.0, older);
        Memory upheld    = memory("ACC-A", 0.9, newer);
        AmlPriorContext ctx = new AmlPriorContext(
            List.of(withdrawn, upheld), List.of(), List.of());
        assertTrue(ctx.isKnownHighRisk(),
            "UPHELD is most recent — must return true");
    }

    @Test
    void toContextMap_emptyContext_correctShape() {
        Map<String, Object> map = AmlPriorContext.empty().toContextMap();
        assertEquals(false, map.get("hasHistory"));
        assertEquals(false, map.get("knownHighRisk"));
        assertEquals(0, map.get("entityRiskCount"));
        assertEquals(0, map.get("networkCount"));
        assertEquals(0, map.get("patternCount"));
        assertTrue(((List<?>) map.get("facts")).isEmpty());
    }

    @Test
    void toContextMap_factHasRequiredFields() {
        AmlPriorContext ctx = new AmlPriorContext(
            List.of(memory("ACC-A", 0.9, Instant.now())),
            List.of(), List.of());
        @SuppressWarnings("unchecked")
        List<Map<String, Object>> facts = (List<Map<String, Object>>) ctx.toContextMap().get("facts");
        assertEquals(1, facts.size());
        Map<String, Object> fact = facts.get(0);
        assertEquals("aml.entity-risk", fact.get("domain"));
        assertNotNull(fact.get("text"));
        assertNotNull(fact.get("createdAt"));
        // confidence may be null for entries without the attribute — not null here
        assertEquals("0.9000", fact.get("confidence"));
    }

    @Test
    void toContextMap_limitsTo10Facts() {
        // 12 memories across all three domains — expect exactly 10 in facts
        List<Memory> entityRisk = List.of(
            memory("E1", 0.9, Instant.now().minusSeconds(1)),
            memory("E2", 0.8, Instant.now().minusSeconds(2)),
            memory("E3", 0.7, Instant.now().minusSeconds(3)),
            memory("E4", 0.6, Instant.now().minusSeconds(4)),
            memory("E5", 0.5, Instant.now().minusSeconds(5)));
        Memory networkMem = new Memory(UUID.randomUUID().toString(), "ACC-A",
            AmlMemoryDomains.NETWORK, "tenant-1", null, "network fact", Map.of(),
            Instant.now().minusSeconds(6));
        Memory patternMem = new Memory(UUID.randomUUID().toString(), "ACC-A",
            AmlMemoryDomains.PATTERN, "tenant-1", null, "pattern fact", Map.of(),
            Instant.now().minusSeconds(7));
        // 5 entity + 3 network + 4 pattern = 12 total
        List<Memory> network = List.of(networkMem,
            new Memory(UUID.randomUUID().toString(), "ACC-B", AmlMemoryDomains.NETWORK,
                "tenant-1", null, "net2", Map.of(), Instant.now().minusSeconds(8)),
            new Memory(UUID.randomUUID().toString(), "ACC-C", AmlMemoryDomains.NETWORK,
                "tenant-1", null, "net3", Map.of(), Instant.now().minusSeconds(9)));
        List<Memory> pattern = List.of(patternMem,
            new Memory(UUID.randomUUID().toString(), "ACC-A", AmlMemoryDomains.PATTERN,
                "tenant-1", null, "pat2", Map.of(), Instant.now().minusSeconds(10)),
            new Memory(UUID.randomUUID().toString(), "ACC-A", AmlMemoryDomains.PATTERN,
                "tenant-1", null, "pat3", Map.of(), Instant.now().minusSeconds(11)),
            new Memory(UUID.randomUUID().toString(), "ACC-A", AmlMemoryDomains.PATTERN,
                "tenant-1", null, "pat4", Map.of(), Instant.now().minusSeconds(12)));
        AmlPriorContext ctx = new AmlPriorContext(entityRisk, network, pattern);
        @SuppressWarnings("unchecked")
        List<Map<String, Object>> facts = (List<Map<String, Object>>) ctx.toContextMap().get("facts");
        assertEquals(10, facts.size(), "Must cap at 10 facts");
    }

    @Test
    void toContextMap_guaranteesAtLeastOnePerNonEmptyDomain() {
        // 9 entity-risk memories and 1 each of network and pattern
        // The 1 network and 1 pattern must appear in the 10-entry facts list
        List<Memory> entityRisk = List.of(
            memory("E1", 0.9, Instant.now().minusSeconds(1)),
            memory("E2", 0.8, Instant.now().minusSeconds(2)),
            memory("E3", 0.7, Instant.now().minusSeconds(3)),
            memory("E4", 0.6, Instant.now().minusSeconds(4)),
            memory("E5", 0.5, Instant.now().minusSeconds(5)),
            memory("E6", 0.4, Instant.now().minusSeconds(6)),
            memory("E7", 0.3, Instant.now().minusSeconds(7)),
            memory("E8", 0.2, Instant.now().minusSeconds(8)),
            memory("E9", 0.1, Instant.now().minusSeconds(9)));
        Memory networkMem = new Memory(UUID.randomUUID().toString(), "ACC-A",
            AmlMemoryDomains.NETWORK, "tenant-1", null, "network oldest",
            Map.of(), Instant.now().minusSeconds(100));
        Memory patternMem = new Memory(UUID.randomUUID().toString(), "ACC-A",
            AmlMemoryDomains.PATTERN, "tenant-1", null, "pattern oldest",
            Map.of(), Instant.now().minusSeconds(200));
        AmlPriorContext ctx = new AmlPriorContext(entityRisk, List.of(networkMem), List.of(patternMem));
        @SuppressWarnings("unchecked")
        List<Map<String, Object>> facts = (List<Map<String, Object>>) ctx.toContextMap().get("facts");
        assertEquals(10, facts.size());
        long networkCount = facts.stream().filter(f -> "aml.network".equals(f.get("domain"))).count();
        long patternCount = facts.stream().filter(f -> "aml.pattern".equals(f.get("domain"))).count();
        assertTrue(networkCount >= 1, "Must include at least 1 network fact");
        assertTrue(patternCount >= 1, "Must include at least 1 pattern fact");
    }
}
```

- [ ] **Step 2: Run — expect compilation failure**

```bash
mvn -pl app test -Dtest=AmlPriorContextTest 2>&1 | grep -E "FAIL|ERROR|cannot find"
```

Expected: compilation error — `AmlPriorContext` does not exist.

- [ ] **Step 3: Write AmlPriorContext**

Create `app/src/main/java/io/casehub/aml/memory/AmlPriorContext.java`:

```java
package io.casehub.aml.memory;

import io.casehub.platform.api.memory.Memory;
import io.casehub.platform.api.memory.MemoryAttributeKeys;

import java.util.*;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public record AmlPriorContext(
    List<Memory> entityRisk,
    List<Memory> network,
    List<Memory> pattern
) {
    public static AmlPriorContext empty() {
        return new AmlPriorContext(List.of(), List.of(), List.of());
    }

    public boolean hasHistory() {
        return !entityRisk.isEmpty() || !network.isEmpty() || !pattern.isEmpty();
    }

    /**
     * Groups entity-risk memories by entityId, takes the most recent per entity,
     * returns true if any has confidence ≥ 0.8. Uses most-recent-per-entity so
     * a WITHDRAWN reversal (confidence 0.0) supersedes an older UPHELD (confidence 0.9).
     */
    public boolean isKnownHighRisk() {
        return entityRisk.stream()
            .collect(Collectors.groupingBy(
                Memory::entityId,
                Collectors.maxBy(Comparator.comparing(Memory::createdAt))))
            .values().stream()
            .filter(Optional::isPresent)
            .map(Optional::get)
            .anyMatch(m -> {
                String conf = m.attributes().get(MemoryAttributeKeys.CONFIDENCE);
                if (conf == null) return false;
                return MemoryAttributeKeys.parseConfidence(conf) >= 0.8;
            });
    }

    /**
     * Produces a Map<String, Object> for injection into the engine's initialContext
     * under the key "priorEntityContext". Each fact is a structured map (not plain text)
     * to support future LLM agent consumption.
     *
     * <p>Selection: guarantee at least one entry per non-empty domain, fill to 10 by recency.
     */
    public Map<String, Object> toContextMap() {
        List<Memory> selected = selectFacts(10);

        List<Map<String, Object>> facts = selected.stream()
            .map(m -> {
                Map<String, Object> fact = new LinkedHashMap<>();
                fact.put("domain", m.domain().name());
                fact.put("text", m.text());
                fact.put("createdAt", m.createdAt().toString());
                fact.put("confidence", m.attributes().get(MemoryAttributeKeys.CONFIDENCE));
                return (Map<String, Object>) fact;
            })
            .toList();

        return Map.of(
            "hasHistory",      hasHistory(),
            "knownHighRisk",   isKnownHighRisk(),
            "entityRiskCount", entityRisk.size(),
            "networkCount",    network.size(),
            "patternCount",    pattern.size(),
            "facts",           facts
        );
    }

    private List<Memory> selectFacts(int maxTotal) {
        List<Memory> result = new ArrayList<>();
        Set<String> selectedIds = new HashSet<>();

        // Guarantee at least one per non-empty domain
        Stream.of(entityRisk, network, pattern)
            .filter(domain -> !domain.isEmpty())
            .forEach(domain -> domain.stream()
                .max(Comparator.comparing(Memory::createdAt))
                .ifPresent(m -> {
                    result.add(m);
                    selectedIds.add(m.memoryId());
                }));

        // Fill remaining slots by recency across all domains
        List<Memory> all = new ArrayList<>(entityRisk);
        all.addAll(network);
        all.addAll(pattern);
        all.stream()
            .sorted(Comparator.comparing(Memory::createdAt).reversed())
            .filter(m -> !selectedIds.contains(m.memoryId()))
            .limit(Math.max(0L, (long) maxTotal - result.size()))
            .forEach(result::add);

        result.sort(Comparator.comparing(Memory::createdAt).reversed());
        return result.size() <= maxTotal ? result : result.subList(0, maxTotal);
    }
}
```

- [ ] **Step 4: Run AmlPriorContextTest**

```bash
mvn -pl app -am test -Dtest=AmlPriorContextTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all 9 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/src/main/java/io/casehub/aml/memory/AmlPriorContext.java \
  app/src/test/java/io/casehub/aml/memory/AmlPriorContextTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-32): AmlPriorContext value record with isKnownHighRisk and toContextMap

Refs #32"
```

---

## Task 6: AmlMemoryService — unit tests + implementation

**Files:**
- Create: `app/src/main/java/io/casehub/aml/memory/AmlMemoryService.java`
- Create: `app/src/test/java/io/casehub/aml/memory/AmlMemoryServiceTest.java`

- [ ] **Step 1: Write failing unit tests for AmlMemoryService**

Create `app/src/test/java/io/casehub/aml/memory/AmlMemoryServiceTest.java`:

```java
package io.casehub.aml.memory;

import io.casehub.aml.domain.EntityResolutionResult;
import io.casehub.aml.domain.PatternAnalysisResult;
import io.casehub.aml.domain.SarOutcome;
import io.casehub.aml.domain.SarVerdict;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.memory.*;
import io.casehub.platform.api.preferences.PreferenceProvider;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class AmlMemoryServiceTest {

    @Mock CaseMemoryStore memoryStore;
    @Mock CurrentPrincipal principal;
    @Mock PreferenceProvider preferenceProvider;

    private AmlMemoryService service;
    private static final String TENANT = "test-tenant";

    private static final SuspiciousTransaction TX = new SuspiciousTransaction(
        "TXN-001", "ACC-ORIGIN", "ACC-DEST",
        new BigDecimal("50000"), "USD", Instant.now(), "Structuring");

    @BeforeEach
    void setUp() {
        when(principal.tenancyId()).thenReturn(TENANT);
        service = new AmlMemoryService(memoryStore, principal, preferenceProvider);
    }

    // ── storeEntityRisk ────────────────────────────────────────────────────────

    @Test
    void storeEntityRisk_callsStoreWithEntityRiskDomain() {
        EntityResolutionResult result = new EntityResolutionResult("entity-1", "chain", "CORPORATE", 0.35);
        when(memoryStore.store(any())).thenReturn("mem-1");

        service.storeEntityRisk(null, "entity-1", result);

        verify(memoryStore).store(argThat(input ->
            AmlMemoryDomains.ENTITY_RISK.equals(input.domain())
            && "entity-1".equals(input.entityId())
            && TENANT.equals(input.tenantId())
            && input.caseId() == null));
    }

    @Test
    void storeEntityRisk_textContainsEntityTypeAndRiskScore() {
        EntityResolutionResult result = new EntityResolutionResult("e-1", "chain", "PEP", 0.92);
        when(memoryStore.store(any())).thenReturn("mem-1");
        List<MemoryInput> captured = new ArrayList<>();
        doAnswer(inv -> { captured.add(inv.getArgument(0)); return "mem-1"; })
            .when(memoryStore).store(any());

        service.storeEntityRisk(null, "e-1", result);

        assertEquals(1, captured.size());
        String text = captured.get(0).text();
        assertTrue(text.contains("PEP"), "text must mention entity type");
        assertTrue(text.contains("0.9200"), "text must include risk score");
    }

    @Test
    void storeEntityRisk_confidenceAttributeSetCorrectly() {
        EntityResolutionResult result = new EntityResolutionResult("e-1", "chain", "PEP", 0.92);
        List<MemoryInput> captured = new ArrayList<>();
        doAnswer(inv -> { captured.add(inv.getArgument(0)); return "mem-1"; })
            .when(memoryStore).store(any());

        service.storeEntityRisk(null, "e-1", result);

        String conf = captured.get(0).attributes().get(MemoryAttributeKeys.CONFIDENCE);
        assertEquals("0.9200", conf);
    }

    @Test
    void storeEntityRisk_doesNotThrowOnMemoryStoreFailure() {
        EntityResolutionResult result = new EntityResolutionResult("e-1", "chain", "CORPORATE", 0.3);
        when(memoryStore.store(any())).thenThrow(new RuntimeException("store failure"));

        assertDoesNotThrow(() -> service.storeEntityRisk(null, "e-1", result));
    }

    // ── storeNetworkRelationship ───────────────────────────────────────────────

    @Test
    void storeNetworkRelationship_callsStoreAllForBothAccounts() {
        EntityResolutionResult result = new EntityResolutionResult("e-1", "chain", "CORPORATE", 0.35);
        when(memoryStore.storeAll(any())).thenReturn(List.of("m1", "m2"));

        service.storeNetworkRelationship(null, TX, result);

        verify(memoryStore).storeAll(argThat(inputs ->
            inputs.size() == 2
            && inputs.stream().allMatch(i -> AmlMemoryDomains.NETWORK.equals(i.domain()))
            && inputs.stream().anyMatch(i -> "ACC-ORIGIN".equals(i.entityId()))
            && inputs.stream().anyMatch(i -> "ACC-DEST".equals(i.entityId()))));
    }

    @Test
    void storeNetworkRelationship_textMentionsBothAccounts() {
        EntityResolutionResult result = new EntityResolutionResult("e-1", "chain", "CORPORATE", 0.35);
        List<List<MemoryInput>> captured = new ArrayList<>();
        doAnswer(inv -> { captured.add(inv.getArgument(0)); return List.of("m1", "m2"); })
            .when(memoryStore).storeAll(any());

        service.storeNetworkRelationship(null, TX, result);

        String text = captured.get(0).get(0).text();
        assertTrue(text.contains("ACC-ORIGIN"), "text must mention origin account");
        assertTrue(text.contains("ACC-DEST"),   "text must mention destination account");
    }

    // ── storePatternFindings ───────────────────────────────────────────────────

    @Test
    void storePatternFindings_callsStoreAllForBothAccounts() {
        PatternAnalysisResult result = new PatternAnalysisResult(true, "Layering detected");
        when(memoryStore.storeAll(any())).thenReturn(List.of("m1", "m2"));

        service.storePatternFindings(null, TX, result);

        verify(memoryStore).storeAll(argThat(inputs ->
            inputs.size() == 2
            && inputs.stream().allMatch(i -> AmlMemoryDomains.PATTERN.equals(i.domain()))
            && inputs.stream().anyMatch(i -> "ACC-ORIGIN".equals(i.entityId()))
            && inputs.stream().anyMatch(i -> "ACC-DEST".equals(i.entityId()))));
    }

    @Test
    void storePatternFindings_textMentionsStructuringDetected() {
        PatternAnalysisResult result = new PatternAnalysisResult(true, "Smurfing pattern");
        List<List<MemoryInput>> captured = new ArrayList<>();
        doAnswer(inv -> { captured.add(inv.getArgument(0)); return List.of("m1", "m2"); })
            .when(memoryStore).storeAll(any());

        service.storePatternFindings(null, TX, result);

        String text = captured.get(0).get(0).text();
        assertTrue(text.contains("Smurfing pattern"), "text must include description");
        assertTrue(text.toLowerCase().contains("structuring"), "text must mention structuring");
    }

    // ── storeSarOutcome ────────────────────────────────────────────────────────

    @Test
    void storeSarOutcome_upheld_highConfidence_entityRiskDomain() {
        SarOutcome outcome = new SarOutcome(SarVerdict.UPHELD, "SAR upheld", 0.95);
        when(memoryStore.storeAll(any())).thenReturn(List.of("m1", "m2"));

        service.storeSarOutcome(java.util.UUID.randomUUID(), TX, outcome);

        verify(memoryStore).storeAll(argThat(inputs ->
            inputs.size() == 2
            && inputs.stream().allMatch(i -> AmlMemoryDomains.ENTITY_RISK.equals(i.domain()))
            && inputs.stream().anyMatch(i -> "ACC-ORIGIN".equals(i.entityId()))
            && inputs.stream().anyMatch(i -> "ACC-DEST".equals(i.entityId()))));
    }

    @Test
    void storeSarOutcome_withdrawn_writesZeroConfidenceReversal() {
        SarOutcome outcome = new SarOutcome(SarVerdict.WITHDRAWN, "SAR withdrawn", 0.10);
        List<List<MemoryInput>> captured = new ArrayList<>();
        doAnswer(inv -> { captured.add(inv.getArgument(0)); return List.of("m1", "m2"); })
            .when(memoryStore).storeAll(any());

        service.storeSarOutcome(java.util.UUID.randomUUID(), TX, outcome);

        String conf = captured.get(0).get(0).attributes().get(MemoryAttributeKeys.CONFIDENCE);
        assertEquals("0.0000", conf, "WITHDRAWN must write confidence 0.0");
    }

    @Test
    void storeSarOutcome_doesNotThrowOnMemoryStoreFailure() {
        SarOutcome outcome = new SarOutcome(SarVerdict.UPHELD, "reason", 0.9);
        when(memoryStore.storeAll(any())).thenThrow(new RuntimeException("store failure"));

        assertDoesNotThrow(() -> service.storeSarOutcome(java.util.UUID.randomUUID(), TX, outcome));
    }

    // ── queryPriorContext ──────────────────────────────────────────────────────

    @Test
    void queryPriorContext_executesThreeDomainQueries() {
        when(memoryStore.query(any())).thenReturn(List.of());
        when(preferenceProvider.resolve(any())).thenReturn(
            io.casehub.platform.api.preferences.MapPreferences.empty());

        service.queryPriorContext(TX);

        verify(memoryStore, times(3)).query(any());
    }

    @Test
    void queryPriorContext_partialFailure_returnsPartialContext() {
        // First domain query succeeds, second throws, third succeeds
        when(memoryStore.query(argThat(q -> AmlMemoryDomains.ENTITY_RISK.equals(q.domain()))))
            .thenReturn(List.of());
        when(memoryStore.query(argThat(q -> AmlMemoryDomains.NETWORK.equals(q.domain()))))
            .thenThrow(new RuntimeException("network query failed"));
        when(memoryStore.query(argThat(q -> AmlMemoryDomains.PATTERN.equals(q.domain()))))
            .thenReturn(List.of());
        when(preferenceProvider.resolve(any())).thenReturn(
            io.casehub.platform.api.preferences.MapPreferences.empty());

        // Must not throw — partial result is acceptable
        AmlPriorContext ctx = assertDoesNotThrow(() -> service.queryPriorContext(TX));
        assertNotNull(ctx, "Partial failure must return a non-null context");
    }
}
```

- [ ] **Step 2: Run — expect compilation failure**

```bash
mvn -pl app test -Dtest=AmlMemoryServiceTest 2>&1 | grep -E "FAIL|ERROR|cannot find symbol"
```

Expected: compilation errors — `AmlMemoryService` does not exist.

- [ ] **Step 3: Write AmlMemoryService**

Create `app/src/main/java/io/casehub/aml/memory/AmlMemoryService.java`:

```java
package io.casehub.aml.memory;

import io.casehub.aml.domain.EntityResolutionResult;
import io.casehub.aml.domain.PatternAnalysisResult;
import io.casehub.aml.domain.SarOutcome;
import io.casehub.aml.domain.SarVerdict;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.memory.*;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.Preferences;
import io.casehub.platform.api.preferences.SettingsScope;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * Central service for all CaseMemoryStore interactions in the AML domain.
 *
 * <p>No caller touches CaseMemoryStore directly — all domain semantics (text
 * formatting, domain selection, attribute conventions) live here.
 *
 * <p>Failure handling: every store and query call is guarded. Memory failures
 * MUST NOT propagate — investigation is the primary flow; memory is additive.
 */
@ApplicationScoped
public class AmlMemoryService {

    private static final Logger LOG = Logger.getLogger(AmlMemoryService.class);
    private static final String AML_SYSTEM_ACTOR = "aml-system";

    private final CaseMemoryStore memoryStore;
    private final CurrentPrincipal principal;
    private final PreferenceProvider preferenceProvider;

    @Inject
    public AmlMemoryService(
            final CaseMemoryStore memoryStore,
            final CurrentPrincipal principal,
            final PreferenceProvider preferenceProvider) {
        this.memoryStore = memoryStore;
        this.principal = principal;
        this.preferenceProvider = preferenceProvider;
    }

    // ── Read path ──────────────────────────────────────────────────────────────

    /**
     * Query prior entity context for a suspicious transaction.
     * Executes three domain queries (entity-risk, network, pattern) across both account IDs.
     * Each query has its own try/catch — partial failure returns partial context.
     */
    public AmlPriorContext queryPriorContext(final SuspiciousTransaction transaction) {
        final String tenantId = principal.tenancyId();
        final List<String> entityIds = List.of(
            transaction.originAccountId(), transaction.destinationAccountId());
        final Instant since = lookbackCutoff();

        List<Memory> entityRisk = queryDomain(entityIds, AmlMemoryDomains.ENTITY_RISK, tenantId, since);
        List<Memory> network    = queryDomain(entityIds, AmlMemoryDomains.NETWORK,      tenantId, null);
        List<Memory> pattern    = queryDomain(entityIds, AmlMemoryDomains.PATTERN,      tenantId, null);

        return new AmlPriorContext(entityRisk, network, pattern);
    }

    private List<Memory> queryDomain(
            final List<String> entityIds,
            final MemoryDomain domain,
            final String tenantId,
            final Instant since) {
        try {
            MemoryQuery query = MemoryQuery.forEntities(entityIds, domain, tenantId).withLimit(10);
            if (since != null) {
                query = query.withSince(since);
            }
            return memoryStore.query(query);
        } catch (Exception e) {
            LOG.warnf(e, "Memory query failed for domain %s — returning empty list", domain.name());
            return List.of();
        }
    }

    private Instant lookbackCutoff() {
        try {
            final Preferences prefs = preferenceProvider.resolve(
                SettingsScope.of("casehubio", "aml", "memory"));
            final io.casehub.aml.routing.IntPreference lookback =
                prefs.get(AmlMemoryPolicyKeys.ENTITY_RISK_LOOKBACK_DAYS);
            final int days = lookback != null ? lookback.value() : 365;
            return Instant.now().minus(Duration.ofDays(days));
        } catch (Exception e) {
            LOG.warnf(e, "Failed to resolve memory lookback preference — defaulting to 365 days");
            return Instant.now().minus(Duration.ofDays(365));
        }
    }

    // ── Write path ─────────────────────────────────────────────────────────────

    /**
     * Store entity risk facts established during entity resolution.
     * caseId is null until qhorus#190 ships (behaviours called with null command).
     */
    public void storeEntityRisk(
            final UUID caseId,
            final String entityId,
            final EntityResolutionResult result) {
        try {
            final String text = String.format(
                "Account %s resolved as entity type %s (risk score: %.4f). Ownership chain: %s.",
                entityId, result.entityType(), result.riskScore(), result.ownershipChain());
            memoryStore.store(new MemoryInput(
                entityId,
                AmlMemoryDomains.ENTITY_RISK,
                principal.tenancyId(),
                caseId != null ? caseId.toString() : null,
                text,
                Map.of(
                    MemoryAttributeKeys.ACTOR_ID,   AML_SYSTEM_ACTOR,
                    MemoryAttributeKeys.OUTCOME,    result.entityType(),
                    MemoryAttributeKeys.CONFIDENCE,
                        MemoryAttributeKeys.formatConfidence(result.riskScore()))));
        } catch (Exception e) {
            LOG.warnf(e, "storeEntityRisk failed for entity %s — skipping", entityId);
        }
    }

    /**
     * Store network relationship facts under both account IDs.
     * caseId is null until qhorus#190 ships.
     */
    public void storeNetworkRelationship(
            final UUID caseId,
            final SuspiciousTransaction transaction,
            final EntityResolutionResult result) {
        try {
            final String text = String.format(
                "Transaction %s between accounts %s and %s. Beneficial ownership: %s.",
                transaction.id(),
                transaction.originAccountId(),
                transaction.destinationAccountId(),
                result.ownershipChain());
            final String caseIdStr = caseId != null ? caseId.toString() : null;
            final String tenantId = principal.tenancyId();
            final Map<String, String> attrs = Map.of(MemoryAttributeKeys.ACTOR_ID, AML_SYSTEM_ACTOR);
            memoryStore.storeAll(List.of(
                new MemoryInput(transaction.originAccountId(),      AmlMemoryDomains.NETWORK, tenantId, caseIdStr, text, attrs),
                new MemoryInput(transaction.destinationAccountId(), AmlMemoryDomains.NETWORK, tenantId, caseIdStr, text, attrs)));
        } catch (Exception e) {
            LOG.warnf(e, "storeNetworkRelationship failed for transaction %s — skipping", transaction.id());
        }
    }

    /**
     * Store pattern analysis findings under both account IDs.
     * caseId is null until qhorus#190 ships.
     */
    public void storePatternFindings(
            final UUID caseId,
            final SuspiciousTransaction transaction,
            final PatternAnalysisResult result) {
        try {
            final String text = String.format(
                "Pattern analysis for transaction %s (accounts %s → %s): %s. Structuring detected: %b.",
                transaction.id(),
                transaction.originAccountId(),
                transaction.destinationAccountId(),
                result.description(),
                result.structuringDetected());
            final String outcome = result.structuringDetected() ? "STRUCTURING_DETECTED" : "NO_STRUCTURING";
            final String caseIdStr = caseId != null ? caseId.toString() : null;
            final String tenantId = principal.tenancyId();
            final Map<String, String> attrs = Map.of(
                MemoryAttributeKeys.ACTOR_ID, AML_SYSTEM_ACTOR,
                MemoryAttributeKeys.OUTCOME,  outcome);
            memoryStore.storeAll(List.of(
                new MemoryInput(transaction.originAccountId(),      AmlMemoryDomains.PATTERN, tenantId, caseIdStr, text, attrs),
                new MemoryInput(transaction.destinationAccountId(), AmlMemoryDomains.PATTERN, tenantId, caseIdStr, text, attrs)));
        } catch (Exception e) {
            LOG.warnf(e, "storePatternFindings failed for transaction %s — skipping", transaction.id());
        }
    }

    /**
     * Store SAR outcome under both account IDs.
     * caseId is the engine UUID (always available — comes from SarOutcomeRecordedEvent payload).
     * WITHDRAWN and FLAGGED verdicts write confidence=0.0 as a reversal signal.
     */
    public void storeSarOutcome(
            final UUID caseId,
            final SuspiciousTransaction transaction,
            final SarOutcome outcome) {
        try {
            final boolean isReversal =
                outcome.verdict() == SarVerdict.WITHDRAWN || outcome.verdict() == SarVerdict.FLAGGED;
            final double confidence = isReversal ? 0.0 : outcome.investigationAccuracyScore();
            final String text = String.format(
                "Transaction from %s to %s resulted in SAR %s (%s). Investigation accuracy: %.4f.",
                transaction.originAccountId(),
                transaction.destinationAccountId(),
                isReversal ? "reversal" : "filing",
                outcome.verdict().name(),
                outcome.investigationAccuracyScore());
            final String caseIdStr = caseId != null ? caseId.toString() : null;
            final String tenantId = principal.tenancyId();
            final Map<String, String> attrs = Map.of(
                MemoryAttributeKeys.ACTOR_ID,   AML_SYSTEM_ACTOR,
                MemoryAttributeKeys.OUTCOME,    outcome.verdict().name(),
                MemoryAttributeKeys.CONFIDENCE, MemoryAttributeKeys.formatConfidence(confidence));
            memoryStore.storeAll(List.of(
                new MemoryInput(transaction.originAccountId(),      AmlMemoryDomains.ENTITY_RISK, tenantId, caseIdStr, text, attrs),
                new MemoryInput(transaction.destinationAccountId(), AmlMemoryDomains.ENTITY_RISK, tenantId, caseIdStr, text, attrs)));
        } catch (Exception e) {
            LOG.warnf(e, "storeSarOutcome failed for caseId %s — skipping", caseId);
        }
    }
}
```

- [ ] **Step 4: Run AmlMemoryServiceTest**

```bash
mvn -pl app -am test -Dtest=AmlMemoryServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all tests PASS. If `MapPreferences.empty()` doesn't exist, check the platform-api version — use `preferenceProvider.resolve(any())` returning a mock `Preferences` instead.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/src/main/java/io/casehub/aml/memory/AmlMemoryService.java \
  app/src/test/java/io/casehub/aml/memory/AmlMemoryServiceTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-32): AmlMemoryService — central CaseMemoryStore adapter for AML domain

Refs #32"
```

---

## Task 7: AmlMemoryService @QuarkusTest roundtrip

**Files:**
- Create: `app/src/test/java/io/casehub/aml/memory/AmlMemoryRoundtripTest.java`

- [ ] **Step 1: Write failing @QuarkusTest roundtrip tests**

Create `app/src/test/java/io/casehub/aml/memory/AmlMemoryRoundtripTest.java`:

```java
package io.casehub.aml.memory;

import io.casehub.aml.domain.EntityResolutionResult;
import io.casehub.aml.domain.PatternAnalysisResult;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.platform.api.memory.CaseMemoryStore;
import io.casehub.platform.api.memory.EraseRequest;
import io.casehub.platform.api.identity.TenancyConstants;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.time.Instant;

import static org.junit.jupiter.api.Assertions.*;

/**
 * @QuarkusTest: verifies AmlMemoryService store/query roundtrip using InMemoryMemoryStore
 * (selected via quarkus.arc.selected-alternatives in test application.properties).
 *
 * Each test uses unique account IDs to avoid cross-test pollution — InMemoryMemoryStore
 * is ApplicationScoped and shared across tests in the same Quarkus instance.
 */
@QuarkusTest
class AmlMemoryRoundtripTest {

    @Inject AmlMemoryService memoryService;
    @Inject CaseMemoryStore memoryStore;

    private static final String TENANT = TenancyConstants.DEFAULT_TENANT_ID;

    private SuspiciousTransaction tx(String id, String origin, String dest) {
        return new SuspiciousTransaction(id, origin, dest,
            new BigDecimal("50000"), "USD", Instant.now(), "Structuring");
    }

    @Test
    void storeEntityRisk_thenQueryPriorContext_returnsStoredFact() {
        SuspiciousTransaction transaction = tx("TXN-RT-001", "ACC-RT-ORIGIN-1", "ACC-RT-DEST-1");
        EntityResolutionResult result = new EntityResolutionResult(
            "ACC-RT-ORIGIN-1", "direct-owner", "PEP", 0.92);

        memoryService.storeEntityRisk(null, "ACC-RT-ORIGIN-1", result);

        AmlPriorContext ctx = memoryService.queryPriorContext(transaction);
        assertFalse(ctx.entityRisk().isEmpty(),
            "queryPriorContext must return the stored entity-risk memory");
        assertTrue(ctx.entityRisk().stream()
            .anyMatch(m -> "ACC-RT-ORIGIN-1".equals(m.entityId())),
            "stored memory must be for ACC-RT-ORIGIN-1");
        assertTrue(ctx.isKnownHighRisk(),
            "PEP entity with confidence 0.92 must trigger isKnownHighRisk");
    }

    @Test
    void storeNetworkRelationship_bothAccountIdsQueryable() {
        SuspiciousTransaction transaction = tx("TXN-RT-002", "ACC-RT-NET-A", "ACC-RT-NET-B");
        EntityResolutionResult result = new EntityResolutionResult(
            "entity-stub", "direct-owner", "CORPORATE", 0.35);

        memoryService.storeNetworkRelationship(null, transaction, result);

        // Query using origin account
        AmlPriorContext ctxA = memoryService.queryPriorContext(
            tx("TXN-RT-002-A", "ACC-RT-NET-A", "unrelated"));
        assertFalse(ctxA.network().isEmpty(), "origin account must have network memory");

        // Query using destination account
        AmlPriorContext ctxB = memoryService.queryPriorContext(
            tx("TXN-RT-002-B", "ACC-RT-NET-B", "unrelated"));
        assertFalse(ctxB.network().isEmpty(), "destination account must have network memory");
    }

    @Test
    void storePatternFindings_bothAccountIdsQueryable() {
        SuspiciousTransaction transaction = tx("TXN-RT-003", "ACC-RT-PAT-A", "ACC-RT-PAT-B");
        PatternAnalysisResult result = new PatternAnalysisResult(true, "Smurfing detected");

        memoryService.storePatternFindings(null, transaction, result);

        AmlPriorContext ctxA = memoryService.queryPriorContext(
            tx("TXN-RT-003-A", "ACC-RT-PAT-A", "unrelated"));
        assertFalse(ctxA.pattern().isEmpty(), "origin account must have pattern memory");

        AmlPriorContext ctxB = memoryService.queryPriorContext(
            tx("TXN-RT-003-B", "ACC-RT-PAT-B", "unrelated"));
        assertFalse(ctxB.pattern().isEmpty(), "destination account must have pattern memory");
    }

    @Test
    void erasure_removesEntityRiskForAccount_leavesNetworkIntact() {
        SuspiciousTransaction transaction = tx("TXN-RT-004", "ACC-RT-ERASE-A", "ACC-RT-ERASE-B");
        EntityResolutionResult result = new EntityResolutionResult(
            "ACC-RT-ERASE-A", "chain", "CORPORATE", 0.85);

        memoryService.storeEntityRisk(null, "ACC-RT-ERASE-A", result);
        memoryService.storeNetworkRelationship(null, transaction, result);

        // Verify pre-erase state
        AmlPriorContext before = memoryService.queryPriorContext(transaction);
        assertFalse(before.entityRisk().isEmpty(), "must have entity-risk before erase");
        assertFalse(before.network().isEmpty(),    "must have network before erase");

        // Erase entity-risk domain only
        memoryStore.erase(new EraseRequest("ACC-RT-ERASE-A", AmlMemoryDomains.ENTITY_RISK, TENANT, null));

        AmlPriorContext after = memoryService.queryPriorContext(transaction);
        assertTrue(after.entityRisk().isEmpty(),
            "entity-risk must be empty after erase");
        assertFalse(after.network().isEmpty(),
            "network domain must be unaffected by entity-risk erase");
    }

    @Test
    void hasHistory_false_whenNoMemoriesExist() {
        SuspiciousTransaction transaction = tx("TXN-RT-005", "ACC-RT-EMPTY-A", "ACC-RT-EMPTY-B");
        // No memories stored for these accounts
        AmlPriorContext ctx = memoryService.queryPriorContext(transaction);
        assertFalse(ctx.hasHistory());
        assertFalse(ctx.isKnownHighRisk());
    }
}
```

- [ ] **Step 2: Run the roundtrip tests**

```bash
mvn -pl app -am test -Dtest=AmlMemoryRoundtripTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all 5 tests PASS. If `TenancyConstants` is not found, import from `io.casehub.platform.api.identity.TenancyConstants`. If `EraseRequest` constructor args are wrong, check the record definition in casehub-platform-api.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/src/test/java/io/casehub/aml/memory/AmlMemoryRoundtripTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "test(issue-32): AmlMemoryService @QuarkusTest roundtrip — store/query/erasure

Refs #32"
```

---

## Task 8: AmlEngineCoordinator prior context injection

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlEngineCoordinator.java`

The coordinator queries prior entity context before starting the case and injects it into `initialContext` under key `"priorEntityContext"`.

- [ ] **Step 1: Write the failing test (checking initialContext shape)**

This is validated in Task 10's routing test. For now, verify the coordinator compiles and returns the correct caseId with the memory service wired.

- [ ] **Step 2: Modify AmlEngineCoordinator**

Replace the class body in `app/src/main/java/io/casehub/aml/engine/AmlEngineCoordinator.java`:

```java
package io.casehub.aml.engine;

import java.util.HashMap;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.ledger.AmlLedgerService;
import io.casehub.aml.memory.AmlMemoryService;
import io.casehub.aml.memory.AmlPriorContext;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class AmlEngineCoordinator {

    private static final Logger LOG = Logger.getLogger(AmlEngineCoordinator.class);
    private static final int CASE_START_TIMEOUT_SECONDS = 5;
    private static final TypeReference<Map<String, Object>> MAP_TYPE = new TypeReference<>() {};

    @Inject AmlInvestigationCaseHub caseHub;
    @Inject AmlLedgerService ledgerService;
    @Inject ObjectMapper objectMapper;
    @Inject AmlMemoryService memoryService;

    public UUID startInvestigation(final SuspiciousTransaction transaction) {
        final Map<String, Object> txMap = objectMapper.convertValue(transaction, MAP_TYPE);
        final AmlPriorContext priorContext = queryPriorContext(transaction);

        final Map<String, Object> initialContext = new HashMap<>();
        initialContext.put("transaction", txMap);
        initialContext.put("priorEntityContext", priorContext.toContextMap());

        final UUID caseId;
        try {
            caseId = caseHub.startCase(initialContext)
                    .toCompletableFuture()
                    .get(CASE_START_TIMEOUT_SECONDS, TimeUnit.SECONDS);
        } catch (Exception e) {
            LOG.errorf(e, "Failed to start AML investigation case for transaction %s",
                    transaction.id());
            throw new RuntimeException("Failed to start investigation case", e);
        }

        ledgerService.writeCaseOpened(transaction, caseId);
        LOG.infof("AML investigation started: caseId=%s txId=%s hasHistory=%b knownHighRisk=%b",
                caseId, transaction.id(), priorContext.hasHistory(), priorContext.isKnownHighRisk());
        return caseId;
    }

    private AmlPriorContext queryPriorContext(final SuspiciousTransaction transaction) {
        try {
            return memoryService.queryPriorContext(transaction);
        } catch (Exception e) {
            LOG.warnf(e, "Prior context query failed for transaction %s — starting cold",
                    transaction.id());
            return AmlPriorContext.empty();
        }
    }
}
```

- [ ] **Step 3: Run existing Layer 5 tests to verify coordinator still works**

```bash
mvn -pl app -am test -Dtest="AmlLayer5ResourceTest,AmlLayer5InvestigationTest" -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all tests PASS. If CDI unsatisfied for `AmlMemoryService`, check that the `io.casehub.aml.memory` package is scanned (it's in the same module — should be fine). If tests fail with prior-context-related errors, check that `queryPriorContext()` doesn't throw when the store is empty.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/src/main/java/io/casehub/aml/engine/AmlEngineCoordinator.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-32): AmlEngineCoordinator — inject priorEntityContext before case start

Refs #32"
```

---

## Task 9: PushAgentDispatch fix + behaviour emissions + AmlSarOutcomeMemoryObserver

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/agents/PushAgentDispatch.java`
- Modify: `app/src/main/java/io/casehub/aml/agents/EntityResolutionBehaviour.java`
- Modify: `app/src/main/java/io/casehub/aml/agents/PatternAnalysisBehaviour.java`
- Create: `app/src/main/java/io/casehub/aml/memory/AmlSarOutcomeMemoryObserver.java`
- Create: `app/src/test/java/io/casehub/aml/memory/AmlSarOutcomeObserverTest.java`

`PushAgentDispatch.post()` currently calls `behaviour.handle(null)`. Fix it to pass the fetched `Message` entity (it's already retrieved for the `inReplyTo` lookup). Behaviours can then deserialize the transaction from the COMMAND content for memory storage.

- [ ] **Step 1: Fix PushAgentDispatch to pass the Message entity**

In `app/src/main/java/io/casehub/aml/agents/PushAgentDispatch.java`, update `post()`:

```java
@Override
public void post(final ChannelRef ref, final OutboundMessage message) {
    if (message.type() != MessageType.COMMAND || behaviour == null) {
        return;
    }

    final String corrId = message.correlationId() != null
            ? message.correlationId().toString() : null;

    // Fetch the persisted COMMAND Message — used both for the inReplyTo id
    // and to pass to the behaviour (enabling transaction deserialization).
    final io.casehub.qhorus.runtime.message.Message commandMessage = corrId != null
            ? messageService.findAllByCorrelationId(corrId).stream()
                    .filter(m -> m.messageType == MessageType.COMMAND)
                    .findFirst()
                    .orElse(null)
            : null;

    final SpecialistOutcome<?> outcome = behaviour.handle(commandMessage);
    final Long commandId = commandMessage != null ? commandMessage.id : null;

    final MessageType replyType = switch (outcome) {
        case SpecialistOutcome.Completed<?> ignored -> MessageType.DONE;
        case SpecialistOutcome.Declined<?> ignored  -> MessageType.DECLINE;
        case SpecialistOutcome.Failed<?> ignored    -> MessageType.FAILURE;
    };

    final String content = switch (outcome) {
        case SpecialistOutcome.Completed<?> ignored -> null;
        case SpecialistOutcome.Declined<?> d        -> d.reason();
        case SpecialistOutcome.Failed<?> f          -> f.reason();
    };

    messageService.dispatch(MessageDispatch.builder()
            .channelId(ref.id())
            .sender(behaviour.capability())
            .type(replyType)
            .content(content)
            .correlationId(corrId)
            .inReplyTo(commandId)
            .actorType(ActorType.AGENT)
            .build());
}
```

- [ ] **Step 2: Update EntityResolutionBehaviour to store entity risk + network**

Replace `app/src/main/java/io/casehub/aml/agents/EntityResolutionBehaviour.java`:

```java
package io.casehub.aml.agents;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.aml.DefaultEntityResolutionService;
import io.casehub.aml.domain.EntityResolutionResult;
import io.casehub.aml.domain.SpecialistOutcome;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.memory.AmlMemoryService;
import io.casehub.qhorus.runtime.message.Message;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.Map;

@ApplicationScoped
@DefaultBean
public class EntityResolutionBehaviour implements AgentBehaviour {

    private static final Logger LOG = Logger.getLogger(EntityResolutionBehaviour.class);
    private static final String CAPABILITY = "entity-resolution";
    private static final TypeReference<Map<String, Object>> MAP_TYPE = new TypeReference<>() {};

    private final DefaultEntityResolutionService service = new DefaultEntityResolutionService();

    @Inject AmlMemoryService memoryService;
    @Inject ObjectMapper objectMapper;

    @Override
    public String capability() {
        return CAPABILITY;
    }

    @Override
    public SpecialistOutcome<EntityResolutionResult> handle(final Message command) {
        final SuspiciousTransaction transaction = deserializeTransaction(command);
        final EntityResolutionResult result = service.resolve(transaction);

        memoryService.storeEntityRisk(null, result.entityId(), result);
        if (transaction != null) {
            memoryService.storeNetworkRelationship(null, transaction, result);
        }

        return new SpecialistOutcome.Completed<>(result);
    }

    @SuppressWarnings("unchecked")
    private SuspiciousTransaction deserializeTransaction(final Message command) {
        if (command == null || command.content == null) return null;
        try {
            Map<String, Object> payload = objectMapper.readValue(command.content, MAP_TYPE);
            Object inputData = payload.get("inputData");
            if (!(inputData instanceof Map<?, ?> inputMap)) return null;
            Object txData = inputMap.get("transaction");
            if (!(txData instanceof Map<?, ?> txMap)) return null;
            return objectMapper.convertValue(txMap, SuspiciousTransaction.class);
        } catch (Exception e) {
            LOG.warnf(e, "Failed to deserialize transaction from COMMAND content — memory storage will be partial");
            return null;
        }
    }
}
```

- [ ] **Step 3: Update PatternAnalysisBehaviour to store pattern findings**

Replace `app/src/main/java/io/casehub/aml/agents/PatternAnalysisBehaviour.java`:

```java
package io.casehub.aml.agents;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.aml.DefaultPatternAnalysisService;
import io.casehub.aml.domain.PatternAnalysisResult;
import io.casehub.aml.domain.SpecialistOutcome;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.memory.AmlMemoryService;
import io.casehub.qhorus.runtime.message.Message;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.Map;

@ApplicationScoped
@DefaultBean
public class PatternAnalysisBehaviour implements AgentBehaviour {

    private static final Logger LOG = Logger.getLogger(PatternAnalysisBehaviour.class);
    private static final String CAPABILITY = "pattern-analysis";
    private static final TypeReference<Map<String, Object>> MAP_TYPE = new TypeReference<>() {};

    private final DefaultPatternAnalysisService service = new DefaultPatternAnalysisService();

    @Inject AmlMemoryService memoryService;
    @Inject ObjectMapper objectMapper;

    @Override
    public String capability() {
        return CAPABILITY;
    }

    @Override
    public SpecialistOutcome<PatternAnalysisResult> handle(final Message command) {
        final SuspiciousTransaction transaction = deserializeTransaction(command);
        final PatternAnalysisResult result = service.analyze(transaction);

        if (transaction != null) {
            memoryService.storePatternFindings(null, transaction, result);
        }

        return new SpecialistOutcome.Completed<>(result);
    }

    @SuppressWarnings("unchecked")
    private SuspiciousTransaction deserializeTransaction(final Message command) {
        if (command == null || command.content == null) return null;
        try {
            Map<String, Object> payload = objectMapper.readValue(command.content, MAP_TYPE);
            Object inputData = payload.get("inputData");
            if (!(inputData instanceof Map<?, ?> inputMap)) return null;
            Object txData = inputMap.get("transaction");
            if (!(txData instanceof Map<?, ?> txMap)) return null;
            return objectMapper.convertValue(txMap, SuspiciousTransaction.class);
        } catch (Exception e) {
            LOG.warnf(e, "Failed to deserialize transaction from COMMAND content — pattern memory skipped");
            return null;
        }
    }
}
```

- [ ] **Step 4: Write AmlSarOutcomeMemoryObserver**

Create `app/src/main/java/io/casehub/aml/memory/AmlSarOutcomeMemoryObserver.java`:

```java
package io.casehub.aml.memory;

import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.engine.SarOutcomeRecordedEvent;
import io.casehub.aml.ledger.AmlCaseOpenedLedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import org.jboss.logging.Logger;

import java.math.BigDecimal;
import java.time.Instant;

/**
 * CDI observer for SarOutcomeRecordedEvent. Writes SAR outcome memories under both
 * account IDs by looking up the AmlCaseOpenedLedgerEntry for the case.
 *
 * <p>Uses REQUIRES_NEW to own its own transaction on the default datasource (memory-jpa),
 * independent of the qhorus transaction used by SarOutcomeFeedbackService.
 */
@ApplicationScoped
public class AmlSarOutcomeMemoryObserver {

    private static final Logger LOG = Logger.getLogger(AmlSarOutcomeMemoryObserver.class);

    @Inject LedgerEntryRepository ledgerRepository;
    @Inject AmlMemoryService memoryService;

    @Observes
    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void onSarOutcome(final SarOutcomeRecordedEvent event) {
        final AmlCaseOpenedLedgerEntry caseEntry = ledgerRepository
                .findBySubjectId(event.caseId()).stream()
                .filter(AmlCaseOpenedLedgerEntry.class::isInstance)
                .map(AmlCaseOpenedLedgerEntry.class::cast)
                .findFirst()
                .orElse(null);

        if (caseEntry == null) {
            LOG.warnf("No AmlCaseOpenedLedgerEntry found for caseId=%s — skipping SAR memory write",
                    event.caseId());
            return;
        }

        final SuspiciousTransaction transaction = reconstructTransaction(caseEntry);
        memoryService.storeSarOutcome(event.caseId(), transaction, event.outcome());
        LOG.infof("SAR outcome memory stored: caseId=%s verdict=%s",
                event.caseId(), event.outcome().verdict());
    }

    private SuspiciousTransaction reconstructTransaction(final AmlCaseOpenedLedgerEntry entry) {
        // Reconstruct a minimal SuspiciousTransaction for memory storage.
        // We only need originAccountId and destinationAccountId for the store call.
        return new SuspiciousTransaction(
            entry.transactionId,
            entry.originAccountId,
            entry.destinationAccountId,
            BigDecimal.ZERO,      // amount not needed for memory storage
            "UNKNOWN",            // currency not needed
            Instant.EPOCH,        // timestamp not needed
            "SAR_OUTCOME");       // flagReason not needed
    }
}
```

- [ ] **Step 5: Write AmlSarOutcomeObserverTest**

Create `app/src/test/java/io/casehub/aml/memory/AmlSarOutcomeObserverTest.java`:

```java
package io.casehub.aml.memory;

import io.casehub.aml.domain.SarOutcome;
import io.casehub.aml.domain.SarVerdict;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.engine.AmlEngineCoordinator;
import io.casehub.aml.engine.SarOutcomeRecordedEvent;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.platform.api.memory.CaseMemoryStore;
import io.casehub.platform.api.memory.MemoryQuery;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Verifies that SarOutcomeRecordedEvent triggers both SarOutcomeFeedbackService
 * (trust attestation) and AmlSarOutcomeMemoryObserver (memory write).
 */
@QuarkusTest
class AmlSarOutcomeObserverTest {

    @Inject AmlEngineCoordinator coordinator;
    @Inject Event<SarOutcomeRecordedEvent> sarOutcomeEvent;
    @Inject CaseMemoryStore memoryStore;

    private static final String TENANT = TenancyConstants.DEFAULT_TENANT_ID;

    @Test
    void sarOutcomeEvent_writesMemoryForBothAccounts() {
        // Start a real investigation so we have an AmlCaseOpenedLedgerEntry
        SuspiciousTransaction tx = new SuspiciousTransaction(
            "TXN-SAR-MEM-001", "ACC-SAR-ORIGIN-1", "ACC-SAR-DEST-1",
            new BigDecimal("80000"), "USD", Instant.now(), "SAR outcome test");
        UUID caseId = coordinator.startInvestigation(tx);

        // Fire the SAR outcome event
        sarOutcomeEvent.fire(new SarOutcomeRecordedEvent(
            caseId, new SarOutcome(SarVerdict.UPHELD, "SAR upheld by FinCEN", 0.92)));

        // Both accounts must have entity-risk memory
        List<io.casehub.platform.api.memory.Memory> originMemories = memoryStore.query(
            MemoryQuery.forEntities(List.of("ACC-SAR-ORIGIN-1"), AmlMemoryDomains.ENTITY_RISK, TENANT));
        List<io.casehub.platform.api.memory.Memory> destMemories = memoryStore.query(
            MemoryQuery.forEntities(List.of("ACC-SAR-DEST-1"), AmlMemoryDomains.ENTITY_RISK, TENANT));

        assertFalse(originMemories.isEmpty(),
            "Origin account must have SAR outcome memory");
        assertFalse(destMemories.isEmpty(),
            "Destination account must have SAR outcome memory");
        assertTrue(originMemories.stream().anyMatch(m -> m.text().contains("UPHELD")),
            "SAR memory text must mention verdict");
    }

    @Test
    void withdrawn_sarOutcome_writesZeroConfidenceReversal() {
        SuspiciousTransaction tx = new SuspiciousTransaction(
            "TXN-SAR-MEM-002", "ACC-SAR-WITHDRAWN-1", "ACC-SAR-WITHDRAWN-2",
            new BigDecimal("30000"), "USD", Instant.now(), "SAR withdrawn test");
        UUID caseId = coordinator.startInvestigation(tx);

        sarOutcomeEvent.fire(new SarOutcomeRecordedEvent(
            caseId, new SarOutcome(SarVerdict.WITHDRAWN, "SAR withdrawn", 0.10)));

        List<io.casehub.platform.api.memory.Memory> memories = memoryStore.query(
            MemoryQuery.forEntities(List.of("ACC-SAR-WITHDRAWN-1"), AmlMemoryDomains.ENTITY_RISK, TENANT));

        assertFalse(memories.isEmpty(), "WITHDRAWN must still write a memory entry (reversal)");
        String confidence = memories.get(0).attributes()
            .get(io.casehub.platform.api.memory.MemoryAttributeKeys.CONFIDENCE);
        assertEquals("0.0000", confidence,
            "WITHDRAWN verdict must write confidence=0.0 reversal");
    }

    @Test
    void sarOutcomeEvent_withNoLedgerEntry_doesNotThrow() {
        // Fire event for a caseId with no ledger entry — observer must handle gracefully
        UUID nonexistentCaseId = UUID.randomUUID();
        assertDoesNotThrow(() ->
            sarOutcomeEvent.fire(new SarOutcomeRecordedEvent(
                nonexistentCaseId, new SarOutcome(SarVerdict.UPHELD, "test", 0.9))));
    }
}
```

- [ ] **Step 6: Run behaviour + observer tests**

```bash
mvn -pl app -am test -Dtest="AmlSarOutcomeObserverTest,AmlLayer5InvestigationTest,AmlLayer6InvestigationTest" -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all tests PASS. If `AmlSarOutcomeMemoryObserver.onSarOutcome` fails with a JTA transaction issue, verify `@Transactional(REQUIRES_NEW)` is on the method. If the ledger repo can't find the entry (wrong datasource), confirm `ledgerRepository` scans the qhorus PU (it's on the qhorus datasource, which has the ledger_entry table).

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/src/main/java/io/casehub/aml/agents/PushAgentDispatch.java \
  app/src/main/java/io/casehub/aml/agents/EntityResolutionBehaviour.java \
  app/src/main/java/io/casehub/aml/agents/PatternAnalysisBehaviour.java \
  app/src/main/java/io/casehub/aml/memory/AmlSarOutcomeMemoryObserver.java \
  app/src/test/java/io/casehub/aml/memory/AmlSarOutcomeObserverTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-32): behaviour emissions + AmlSarOutcomeMemoryObserver

Refs #32"
```

---

## Task 10: YAML binding merge + Layer 8 routing tests

**Files:**
- Modify: `app/src/main/resources/aml/aml-investigation.yaml`
- Create: `app/src/test/java/io/casehub/aml/memory/AmlLayer8RoutingTest.java`

The `senior-analyst-required` binding is extended to OR in `priorEntityContext.knownHighRisk`. No new separate binding — merging eliminates the duplicate dispatch race.

- [ ] **Step 1: Write the failing routing tests**

Create `app/src/test/java/io/casehub/aml/memory/AmlLayer8RoutingTest.java`:

```java
package io.casehub.aml.memory;

import io.casehub.api.engine.CaseHubRuntime;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.platform.api.memory.CaseMemoryStore;
import io.casehub.platform.api.memory.MemoryAttributeKeys;
import io.casehub.platform.api.memory.MemoryInput;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.atomic.AtomicReference;
import java.util.stream.Collectors;

import static io.restassured.RestAssured.given;
import static org.awaitility.Awaitility.await;
import static org.junit.jupiter.api.Assertions.*;

/**
 * Layer 8: verifies that prior entity context drives senior analyst routing.
 *
 * Uses unique account IDs per test to avoid cross-test pollution
 * (InMemoryMemoryStore is ApplicationScoped and shared within a test run).
 */
@QuarkusTest
class AmlLayer8RoutingTest {

    @Inject CaseMemoryStore memoryStore;
    @Inject CaseHubRuntime caseHubRuntime;

    private static final Duration TIMEOUT       = Duration.ofSeconds(15);
    private static final Duration POLL_INTERVAL = Duration.ofMillis(100);
    private static final String   TENANT        = TenancyConstants.DEFAULT_TENANT_ID;

    private UUID startInvestigation(final String txId, final String originAccountId) {
        final var body = """
                {
                  "id": "%s",
                  "originAccountId": "%s",
                  "destinationAccountId": "ACC-L8-DEST-SHARED",
                  "amount": 50000,
                  "currency": "USD",
                  "timestamp": "2024-01-01T00:00:00Z",
                  "flagReason": "Unusual pattern"
                }
                """.formatted(txId, originAccountId);

        return UUID.fromString(
            given().contentType("application/json").body(body)
                .when().post("/api/layer6/investigations")
                .then().statusCode(202)
                .extract().path("caseId"));
    }

    private Set<String> scheduledWorkerNames(final UUID caseId) {
        return caseHubRuntime.eventLog(caseId, Set.of(CaseHubEventType.WORKER_SCHEDULED))
                .toCompletableFuture().join()
                .stream()
                .filter(r -> r.metadata() != null && r.metadata().has("workerName"))
                .map(r -> r.metadata().get("workerName").asText())
                .collect(Collectors.toSet());
    }

    private void storeHighConfidenceEntityRisk(final String accountId) {
        memoryStore.store(new MemoryInput(
            accountId, AmlMemoryDomains.ENTITY_RISK, TENANT, null,
            "Account " + accountId + " appeared in 3 prior SAR filings — high risk.",
            Map.of(MemoryAttributeKeys.CONFIDENCE, MemoryAttributeKeys.formatConfidence(0.9),
                   MemoryAttributeKeys.OUTCOME, "UPHELD")));
    }

    @Test
    void knownHighRiskEntity_seniorAnalystScheduled() {
        final String account = "ACC-L8-HR-" + UUID.randomUUID();
        storeHighConfidenceEntityRisk(account);

        UUID caseId = startInvestigation("TXN-L8-HR-001-" + UUID.randomUUID(), account);

        await().atMost(TIMEOUT).pollInterval(POLL_INTERVAL)
            .until(() -> scheduledWorkerNames(caseId).contains("senior-analyst-agent"));
    }

    @Test
    void knownHighRiskEntity_seniorAnalystScheduledExactlyOnce() {
        // Entity stub returns riskScore=0.35 (non-PEP, non-high-risk) — only prior-context triggers.
        // So senior-analyst-required fires at most once (from prior context alone).
        final String account = "ACC-L8-DEDUP-" + UUID.randomUUID();
        storeHighConfidenceEntityRisk(account);

        UUID caseId = startInvestigation("TXN-L8-DEDUP-001-" + UUID.randomUUID(), account);

        // Wait for investigation to complete (SAR drafting scheduled)
        await().atMost(TIMEOUT).pollInterval(POLL_INTERVAL)
            .until(() -> scheduledWorkerNames(caseId).stream()
                .anyMatch(w -> w.startsWith("sar-drafting-agent")));

        long seniorCount = caseHubRuntime.eventLog(caseId, Set.of(CaseHubEventType.WORKER_SCHEDULED))
            .toCompletableFuture().join().stream()
            .filter(r -> r.metadata() != null && r.metadata().has("workerName"))
            .filter(r -> "senior-analyst-agent".equals(r.metadata().get("workerName").asText()))
            .count();
        assertEquals(1, seniorCount,
            "senior-analyst-agent must be scheduled exactly once, not " + seniorCount + " times");
    }

    @Test
    void noHistoryNonPepEntity_seniorAnalystNotScheduled() {
        // No memory pre-populated; entity stub returns CORPORATE with riskScore=0.35
        // Binding must NOT fire.
        final String account = "ACC-L8-NOHIST-" + UUID.randomUUID();

        UUID caseId = startInvestigation("TXN-L8-NOHIST-001-" + UUID.randomUUID(), account);

        final AtomicReference<Set<String>> snapshot = new AtomicReference<>();
        await().atMost(TIMEOUT).pollInterval(POLL_INTERVAL).until(() -> {
            Set<String> workers = scheduledWorkerNames(caseId);
            snapshot.set(workers);
            return workers.stream().anyMatch(w -> w.startsWith("sar-drafting-agent"));
        });

        assertFalse(snapshot.get().contains("senior-analyst-agent"),
            "Non-PEP entity with no history must not trigger senior analyst: " + snapshot.get());
    }

    @Test
    void lowConfidenceHistory_seniorAnalystNotScheduled() {
        // Confidence 0.7 < 0.8 threshold — must NOT trigger isKnownHighRisk
        final String account = "ACC-L8-LOWCONF-" + UUID.randomUUID();
        memoryStore.store(new MemoryInput(
            account, AmlMemoryDomains.ENTITY_RISK, TENANT, null,
            "Account " + account + " — moderate risk.",
            Map.of(MemoryAttributeKeys.CONFIDENCE, MemoryAttributeKeys.formatConfidence(0.7))));

        UUID caseId = startInvestigation("TXN-L8-LOWCONF-001-" + UUID.randomUUID(), account);

        final AtomicReference<Set<String>> snapshot = new AtomicReference<>();
        await().atMost(TIMEOUT).pollInterval(POLL_INTERVAL).until(() -> {
            Set<String> workers = scheduledWorkerNames(caseId);
            snapshot.set(workers);
            return workers.stream().anyMatch(w -> w.startsWith("sar-drafting-agent"));
        });

        assertFalse(snapshot.get().contains("senior-analyst-agent"),
            "Low-confidence history must not trigger senior analyst: " + snapshot.get());
    }
}
```

- [ ] **Step 2: Run routing tests — expect failures (binding not yet updated)**

```bash
mvn -pl app -am test -Dtest=AmlLayer8RoutingTest#knownHighRiskEntity_seniorAnalystScheduled -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: FAIL — `knownHighRiskEntity_seniorAnalystScheduled` times out because `priorEntityContext.knownHighRisk` isn't referenced in the YAML binding yet.

- [ ] **Step 3: Update aml-investigation.yaml — merge senior-analyst-required binding**

In `app/src/main/resources/aml/aml-investigation.yaml`, replace the `senior-analyst-required` binding:

```yaml
    ## Adaptive path — fires when any high-risk signal is present.
    ## Signal 1 (prior context): fires at case start when entity has established high-risk
    ##   history in CaseMemoryStore (priorEntityContext.knownHighRisk == true).
    ## Signal 2 (entity resolution): fires after entity resolution for first-time encounters
    ##   with PEP entities or risk score > 0.8.
    ## The .seniorAnalystReview == null guard prevents re-dispatch if both signals fire.
    - name: senior-analyst-required
      on: { contextChange: {} }
      when: >-
        ((.priorEntityContext.knownHighRisk == true) or
         (.entityResolution.entityType == "PEP") or
         (.entityResolution.riskScore > 0.8)) and
        .seniorAnalystReview == null
      capability: senior-analyst-review
```

The old binding was:
```yaml
    ## Adaptive path — fires only when entity is PEP or risk score exceeds threshold.
    - name: senior-analyst-required
      on: { contextChange: {} }
      when: >-
        .entityResolution != null and
        (.entityResolution.entityType == "PEP" or .entityResolution.riskScore > 0.8) and
        .seniorAnalystReview == null
      capability: senior-analyst-review
```

Note: The new binding removes the `.entityResolution != null` guard on the entity resolution conditions — JQ evaluates null-safe field access (`.entityResolution.entityType` returns null when `.entityResolution` is null, and `null == "PEP"` is false). Verify this JQ behaviour in the engine if tests show the binding fires unexpectedly at case start.

- [ ] **Step 4: Run all Layer 8 routing tests**

```bash
mvn -pl app -am test -Dtest=AmlLayer8RoutingTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all 4 tests PASS. If `knownHighRiskEntity_seniorAnalystScheduled` fails with timeout, check that `toContextMap()` produces `"knownHighRisk": true` for the pre-populated account and that the engine evaluates `.priorEntityContext.knownHighRisk == true` correctly (boolean comparison in JQ is strict — must not be a string `"true"`).

- [ ] **Step 5: Run the full test suite**

```bash
mvn -pl app -am test -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: all tests PASS. Pay attention to:
- `AmlLedgerChainTest` — may fail if casts aren't fully updated
- `AmlLayer5InvestigationTest.nonPepTransaction_seniorAnalystDoesNotFire` — still must pass (no history = no high-risk = no senior analyst)
- `AmlLayer6InvestigationTest` — uses the CDI event path now

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add \
  app/src/main/resources/aml/aml-investigation.yaml \
  app/src/test/java/io/casehub/aml/memory/AmlLayer8RoutingTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(issue-32): merge senior-analyst-required YAML binding to evaluate priorEntityContext

Refs #32"
```

---

## Self-Review Checklist

- [ ] **Spec coverage:**
  - [x] Entity ID strategy (account IDs) — Tasks 6, 7, 9
  - [x] Memory domain taxonomy (ENTITY_RISK, NETWORK, PATTERN) — Task 4
  - [x] AmlMemoryService read + write paths — Tasks 6, 7
  - [x] AmlPriorContext isKnownHighRisk + lookback window — Task 5
  - [x] AmlPriorContext toContextMap 10-item capped, structured facts — Task 5
  - [x] Emission: direct call from behaviours — Task 9
  - [x] Emission: CDI event for SAR outcomes — Tasks 2, 9
  - [x] SarOutcomeRecordedEvent + AmlLayer6Resource fires it — Task 2
  - [x] SarOutcomeFeedbackService observes event — Task 2
  - [x] AmlSarOutcomeMemoryObserver observes event — Task 9
  - [x] AmlEngineCoordinator injects priorEntityContext — Task 8
  - [x] YAML binding merged — Task 10
  - [x] YAML binding fires exactly once (dedup test) — Task 10
  - [x] Ledger subclass redesign (V2007 + two siblings) — Task 1
  - [x] Merkle hash verified — documented in plan header
  - [x] GDPR erasure + domain isolation — Tasks 7
  - [x] Partial query failure handling — Tasks 6, 7
  - [x] caseId=null in behaviour entries (qhorus#190) — Tasks 6, 9
  - [x] WITHDRAWN reversal writes confidence=0.0 — Tasks 6, 9
  - [x] pom.xml + application.properties — Task 3
  - [x] AmlSarOutcomeMemoryObserver reads AmlCaseOpenedLedgerEntry for account IDs — Task 9

- [ ] **Placeholder scan:** No TBDs found. All code blocks contain complete, runnable code.

- [ ] **Type consistency:**
  - `AmlMemoryDomains.ENTITY_RISK` — used in Tasks 4, 5, 6, 9, 10 ✓
  - `AmlPriorContext.empty()` — defined in Task 5, used in Task 8 ✓
  - `AmlMemoryService.storeEntityRisk(UUID, String, EntityResolutionResult)` — defined Task 6, called Task 9 ✓
  - `AmlMemoryService.storeNetworkRelationship(UUID, SuspiciousTransaction, EntityResolutionResult)` — defined Task 6, called Task 9 ✓
  - `AmlMemoryService.storePatternFindings(UUID, SuspiciousTransaction, PatternAnalysisResult)` — defined Task 6, called Task 9 ✓
  - `AmlMemoryService.storeSarOutcome(UUID, SuspiciousTransaction, SarOutcome)` — defined Task 6, called Task 9 ✓
  - `SarOutcomeRecordedEvent(UUID, SarOutcome)` — defined Task 2, fired Task 2, observed Tasks 2 and 9 ✓
  - `AmlCaseOpenedLedgerEntry.originAccountId/.destinationAccountId` — defined Task 1, read Task 9 ✓
