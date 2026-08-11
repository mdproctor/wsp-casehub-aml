# Case Profile Store Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #94 — feat: case profile store — case-level indexing in CaseMemoryStore
**Issue group:** #94

**Goal:** Store completed AML investigations into the CBR case base on SAR outcome recording, with tamper-evident ledger audit.

**Architecture:** Observer pattern on `SarOutcomeRecordedEvent` → extract `CaseProfile` + investigation path from engine context → store via `CbrCaseMemoryStore` + write `AmlCaseProfileLedgerEntry`. Two independent try/catch blocks isolate CBR store from ledger write.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine (CaseInstanceCache, PlanItemStore), casehub-neocortex (CbrCaseMemoryStore, FeatureVectorCbrCase), casehub-ledger (JpaLedgerEntry, LedgerEntryRepository)

## Global Constraints

- Flyway migration at `db/aml-engine-ledger/migration/V3005__*.sql` — H2 MODE=PostgreSQL compatible
- LedgerEntry subclasses extend `JpaLedgerEntry` (not `LedgerEntry`) — GE-20260707-99de4f
- All LedgerEntry writes null-guard `tenancyId` — protocol PP-20260610-ae4535
- Memory/ledger failures must not propagate — established AML convention (try/catch, log, continue)
- `subjectId` on AML ledger entries = raw `caseId` UUID — matches `AmlCaseOpenedLedgerEntry` convention
- CBR store entity ID = `UUID.nameUUIDFromBytes(("aml-cbr:" + caseId).getBytes(UTF_8)).toString()` — namespaced isolation
- `@Observes` (not `@ObservesAsync`) — event dispatched via `.fire()` not `.fireAsync()`
- `@Transactional(TxType.REQUIRES_NEW)` — isolate from caller's request transaction
- IntelliJ MCP workspace: `/Users/mdproctor/claude/casehub/aml` + `/Users/mdproctor/claude/casehub/neocortex` + `/Users/mdproctor/claude/casehub/ledger`
- `project_path` for all ide_* calls: `/Users/mdproctor/claude/casehub/aml`

---

### Task 1: AmlCaseProfileLedgerEntry + Flyway V3005

**Files:**
- Create: `app/src/main/java/io/casehub/aml/ledger/AmlCaseProfileLedgerEntry.java`
- Create: `app/src/main/resources/db/aml-engine-ledger/migration/V3005__case_profile_ledger_entry.sql`
- Test: `app/src/test/java/io/casehub/aml/ledger/AmlCaseProfileLedgerEntryTest.java`

**Interfaces:**
- Consumes: `io.casehub.ledger.runtime.model.jpa.JpaLedgerEntry` (superclass)
- Produces: `AmlCaseProfileLedgerEntry` entity — used by Task 3 (`AmlCaseProfileStoreObserver`) to write the tamper-evident audit record

- [ ] **Step 1: Write the failing test for `domainContentBytes()`**

```java
package io.casehub.aml.ledger;

import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import static org.junit.jupiter.api.Assertions.*;

class AmlCaseProfileLedgerEntryTest {

    @Test
    void domainContentBytes_allFields() {
        var entry = new AmlCaseProfileLedgerEntry();
        entry.flagReason = "STRUCTURING";
        entry.transactionAmount = new BigDecimal("50000.0000");
        entry.priorIncidentCount = 3;
        entry.entityType = "SHELL_COMPANY";
        entry.jurisdictionRisk = "HIGH";
        entry.networkComplexity = "LARGE_NETWORK";
        entry.outcome = "UPHELD";
        entry.confidence = 0.92;
        entry.investigationPath = "entity-resolution → pattern-analysis → sar-drafting";

        byte[] bytes = entry.domainContentBytes();
        String content = new String(bytes, java.nio.charset.StandardCharsets.UTF_8);

        assertEquals("STRUCTURING|50000.0000|3|SHELL_COMPANY|HIGH|LARGE_NETWORK|UPHELD|0.92|entity-resolution → pattern-analysis → sar-drafting", content);
    }

    @Test
    void domainContentBytes_nullableFieldsEmpty() {
        var entry = new AmlCaseProfileLedgerEntry();
        entry.flagReason = "LAYERING";
        entry.transactionAmount = new BigDecimal("1000.5000");
        entry.priorIncidentCount = 0;
        entry.entityType = null;
        entry.jurisdictionRisk = null;
        entry.networkComplexity = null;
        entry.outcome = "FLAGGED";
        entry.confidence = 0.5;
        entry.investigationPath = "(direct-verdict)";

        byte[] bytes = entry.domainContentBytes();
        String content = new String(bytes, java.nio.charset.StandardCharsets.UTF_8);

        assertEquals("LAYERING|1000.5000|0||||FLAGGED|0.5|(direct-verdict)", content);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl app -am -Dtest=AmlCaseProfileLedgerEntryTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `AmlCaseProfileLedgerEntry` does not exist

- [ ] **Step 3: Create `AmlCaseProfileLedgerEntry` entity**

Use `ide_create_file` to create `app/src/main/java/io/casehub/aml/ledger/AmlCaseProfileLedgerEntry.java`:

```java
package io.casehub.aml.ledger;

import io.casehub.ledger.runtime.model.jpa.JpaLedgerEntry;
import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

import java.math.BigDecimal;
import java.nio.charset.StandardCharsets;

@Entity
@Table(name = "aml_case_profile_ledger_entry")
@DiscriminatorValue("AML_CASE_PROFILE")
public class AmlCaseProfileLedgerEntry extends JpaLedgerEntry {

    @Column(name = "flag_reason", nullable = false, length = 50)
    public String flagReason;

    @Column(name = "transaction_amount", nullable = false, precision = 19, scale = 4)
    public BigDecimal transactionAmount;

    @Column(name = "prior_incident_count", nullable = false)
    public int priorIncidentCount;

    @Column(name = "entity_type", length = 50)
    public String entityType;

    @Column(name = "jurisdiction_risk", length = 50)
    public String jurisdictionRisk;

    @Column(name = "network_complexity", length = 50)
    public String networkComplexity;

    @Column(name = "outcome", nullable = false, length = 50)
    public String outcome;

    @Column(name = "confidence", nullable = false)
    public double confidence;

    @Column(name = "investigation_path", nullable = false, length = 1000)
    public String investigationPath;

    @Override
    protected byte[] domainContentBytes() {
        return String.join("|",
                flagReason != null ? flagReason : "",
                transactionAmount != null ? transactionAmount.toPlainString() : "",
                String.valueOf(priorIncidentCount),
                entityType != null ? entityType : "",
                jurisdictionRisk != null ? jurisdictionRisk : "",
                networkComplexity != null ? networkComplexity : "",
                outcome != null ? outcome : "",
                String.valueOf(confidence),
                investigationPath != null ? investigationPath : ""
        ).getBytes(StandardCharsets.UTF_8);
    }
}
```

- [ ] **Step 4: Create Flyway V3005 migration**

Use `Write` to create `app/src/main/resources/db/aml-engine-ledger/migration/V3005__case_profile_ledger_entry.sql`:

```sql
CREATE TABLE aml_case_profile_ledger_entry (
    id                   UUID NOT NULL,
    flag_reason          VARCHAR(50)    NOT NULL,
    transaction_amount   DECIMAL(19,4)  NOT NULL,
    prior_incident_count INTEGER        NOT NULL,
    entity_type          VARCHAR(50),
    jurisdiction_risk    VARCHAR(50),
    network_complexity   VARCHAR(50),
    outcome              VARCHAR(50)    NOT NULL,
    confidence           DOUBLE         NOT NULL,
    investigation_path   VARCHAR(1000)  NOT NULL,
    PRIMARY KEY (id),
    FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl app -am -Dtest=AmlCaseProfileLedgerEntryTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 6: Verify with `ide_diagnostics`**

Run `ide_diagnostics` on the new entity file. Fix any issues.

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/aml/ledger/AmlCaseProfileLedgerEntry.java \
       app/src/main/resources/db/aml-engine-ledger/migration/V3005__case_profile_ledger_entry.sql \
       app/src/test/java/io/casehub/aml/ledger/AmlCaseProfileLedgerEntryTest.java
git commit -m "feat(#94): AmlCaseProfileLedgerEntry + V3005 migration

Tamper-evident ledger entry for CBR case profile storage.
9 columns covering all 6 CaseProfile dimensions plus outcome,
confidence, and investigation path.

Refs #94"
```

---

### Task 2: AmlCbrSchema — add `sar_narrative` semantic text field

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/cbr/AmlCbrSchema.java`
- Modify: `app/src/test/java/io/casehub/aml/cbr/CaseProfileExtractorTest.java` (if schema validation tests exist there)

**Interfaces:**
- Consumes: `io.casehub.neocortex.memory.cbr.FeatureField.semanticText(String)` — creates `Text(name, true)`
- Produces: Updated `AmlCbrSchema.SCHEMA` with 7 fields (was 6) — used by Task 3 for schema registration and feature validation

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.aml.cbr;

import io.casehub.neocortex.memory.cbr.FeatureField;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class AmlCbrSchemaTest {

    @Test
    void schema_containsSevenFields() {
        assertEquals(7, AmlCbrSchema.SCHEMA.fields().size());
    }

    @Test
    void schema_sarNarrativeIsSemanticText() {
        var sarNarrative = AmlCbrSchema.SCHEMA.fields().stream()
                .filter(f -> "sar_narrative".equals(f.name()))
                .findFirst()
                .orElseThrow(() -> new AssertionError("sar_narrative field not found"));

        assertInstanceOf(FeatureField.Text.class, sarNarrative);
        assertTrue(((FeatureField.Text) sarNarrative).semantic());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl app -am -Dtest=AmlCbrSchemaTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — schema has 6 fields, not 7; no `sar_narrative` field

- [ ] **Step 3: Add `sar_narrative` field to `AmlCbrSchema`**

Use `ide_edit_member` to update the `SCHEMA` field in `AmlCbrSchema.java`. Add `FeatureField.semanticText("sar_narrative")` as the last field in `CbrFeatureSchema.of(...)`:

The current `SCHEMA` field ends with:
```java
new FeatureField.Categorical("network_complexity",
        new SimilaritySpec.CategoricalTable(Map.of(
                "SINGLE_ENTITY", Map.of("SMALL_NETWORK", 0.3, "LARGE_NETWORK", 0.1),
                "SMALL_NETWORK", Map.of("LARGE_NETWORK", 0.5)))));
```

Add after the last categorical field, before the closing `);`:
```java
FeatureField.semanticText("sar_narrative"));
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl app -am -Dtest=AmlCbrSchemaTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/aml/cbr/AmlCbrSchema.java \
       app/src/test/java/io/casehub/aml/cbr/AmlCbrSchemaTest.java
git commit -m "feat(#94): add sar_narrative semantic text field to AmlCbrSchema

Enables embedding-based similarity on SAR narratives during
CBR Retrieve (#95). Uses semanticText() (semantic=true) not
text() (semantic=false).

Refs #94"
```

---

### Task 3: AmlCaseProfileStoreObserver + QuarkusTest

**Files:**
- Create: `app/src/main/java/io/casehub/aml/cbr/AmlCaseProfileStoreObserver.java`
- Create: `app/src/test/java/io/casehub/aml/cbr/AmlCaseProfileStoreObserverTest.java`

**Interfaces:**
- Consumes:
  - `SarOutcomeRecordedEvent(UUID caseId, SarOutcome outcome)` — CDI event trigger
  - `CbrCaseMemoryStore.store(CbrCase, String caseType, String entityId, MemoryDomain, String tenantId, String caseId, Path scope)` — CBR storage
  - `LedgerEntryRepository.save(LedgerEntry, String tenancyId)` — ledger write
  - `CaseInstanceCache.get(UUID caseId)` → `CaseInstance` — case context access
  - `PlanItemStore.findByCaseId(UUID caseId, String tenancyId)` → `List<PlanItemRecord>` — investigation path
  - `CaseContext.get(String key)`, `.getString(String key)`, `.getPath(String path)` — context data access
  - `FeatureVectorCbrCase(String problem, String solution, String outcome, Double confidence, Map<String, FeatureValue> features)` — CBR case record
  - `AmlCaseProfileLedgerEntry` (from Task 1) — ledger entry entity
  - `AmlCbrSchema.CASE_TYPE` = `"aml-investigation"`, `AmlMemoryDomains.CBR` — constants
  - `CaseProfile.initial(FlagReason, BigDecimal, int)` and `CaseProfile.complete(FlagReason, BigDecimal, int, EntityType, JurisdictionRisk, NetworkComplexity)` — profile construction
  - `CaseProfile.toFeatures()` → `Map<String, FeatureValue>` — feature extraction
- Produces: CDI observer that fires on SAR outcome and writes to CBR store + ledger. No downstream consumers in this issue.

- [ ] **Step 1: Write the failing `@QuarkusTest`**

The test fires `SarOutcomeRecordedEvent` and verifies that `CbrCaseMemoryStore.store()` was called with the correct arguments. Since the observer uses `@Observes` (synchronous), the store call happens within the same request — no async drain needed.

```java
package io.casehub.aml.cbr;

import io.casehub.aml.domain.FlagReason;
import io.casehub.aml.domain.SarOutcome;
import io.casehub.aml.domain.SarVerdict;
import io.casehub.aml.engine.SarOutcomeRecordedEvent;
import io.casehub.aml.ledger.AmlCaseProfileLedgerEntry;
import io.casehub.api.model.TaskStatus;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.spi.PlanItemStore;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.FeatureVectorCbrCase;
import io.casehub.platform.api.identity.TenancyConstants;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.event.Event;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class AmlCaseProfileStoreObserverTest {

    @Inject Event<SarOutcomeRecordedEvent> sarOutcomeEvent;
    @Inject CaseInstanceCache caseInstanceCache;
    @Inject PlanItemStore planItemStore;
    @Inject CbrCaseMemoryStore cbrStore;
    @Inject LedgerEntryRepository ledgerRepository;

    @Test
    void onSarOutcome_storesProfileInCbrCaseBase() {
        UUID caseId = UUID.randomUUID();

        // Set up case context with transaction and enrichment data
        var instance = new CaseInstance();
        instance.setUuid(caseId);
        instance.tenancyId = TenancyConstants.DEFAULT_TENANT_ID;
        var ctx = /* build CaseContext with transaction, entityType, etc. */;
        instance.setCaseContext(ctx);
        caseInstanceCache.put(instance);

        // Set up plan items for investigation path
        // (use PlanItemStore or engine-testing fixtures)

        // Fire the event
        var outcome = new SarOutcome(SarVerdict.UPHELD, "Confirmed structuring pattern", 0.92);
        sarOutcomeEvent.fire(new SarOutcomeRecordedEvent(caseId, outcome));

        // Verify CBR store was called
        // (assert via InMemoryCbrCaseMemoryStore or a mock/spy)

        // Verify ledger entry was written
        var entries = ledgerRepository.findBySubjectId(caseId, TenancyConstants.DEFAULT_TENANT_ID);
        var profileEntry = entries.stream()
                .filter(AmlCaseProfileLedgerEntry.class::isInstance)
                .map(AmlCaseProfileLedgerEntry.class::cast)
                .findFirst()
                .orElseThrow(() -> new AssertionError("No AmlCaseProfileLedgerEntry found"));

        assertEquals("STRUCTURING", profileEntry.flagReason);
        assertEquals("UPHELD", profileEntry.outcome);
        assertEquals(0.92, profileEntry.confidence, 0.001);
    }
}
```

**Note:** The test above is a skeleton. The actual test requires:
1. Building a `CaseContext` with the right keys (`"transaction"`, `"entityType"`, `"jurisdictionRisk"`, `"networkComplexity"`, `"sarNarrative"`, `"priorEntityContext"`)
2. Creating `PlanItemRecord` entries via `PlanItemStore` — use `PlanItemSaveRequest`
3. Asserting on `InMemoryCbrCaseMemoryStore` state (the in-memory impl is `@Alternative @Priority(1)` — check if it's on the test classpath, otherwise use the `NoOpCbrCaseMemoryStore`)

Look at existing `@QuarkusTest` patterns in AML (e.g. `AmlSarOutcomeObserverTest`, `AmlLayer6IntegrationTest`) for the exact CaseContext construction and PlanItemStore setup patterns.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl app -am -Dtest=AmlCaseProfileStoreObserverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `AmlCaseProfileStoreObserver` does not exist

- [ ] **Step 3: Create `AmlCaseProfileStoreObserver`**

Use `ide_create_file` to create `app/src/main/java/io/casehub/aml/cbr/AmlCaseProfileStoreObserver.java`:

```java
package io.casehub.aml.cbr;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.aml.domain.CaseProfile;
import io.casehub.aml.domain.EntityType;
import io.casehub.aml.domain.JurisdictionRisk;
import io.casehub.aml.domain.NetworkComplexity;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.engine.SarOutcomeRecordedEvent;
import io.casehub.aml.ledger.AmlCaseProfileLedgerEntry;
import io.casehub.aml.memory.AmlMemoryDomains;
import io.casehub.api.model.TaskStatus;
import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.spi.PlanItemStore;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.neocortex.memory.cbr.FeatureVectorCbrCase;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.platform.api.path.Path;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.transaction.Transactional.TxType;
import org.jboss.logging.Logger;

import java.nio.charset.StandardCharsets;
import java.util.Comparator;
import java.util.LinkedHashMap;
import java.util.UUID;
import java.util.stream.Collectors;

@ApplicationScoped
public class AmlCaseProfileStoreObserver {

    private static final Logger LOG = Logger.getLogger(AmlCaseProfileStoreObserver.class);

    @Inject CbrCaseMemoryStore cbrStore;
    @Inject LedgerEntryRepository ledgerRepository;
    @Inject CaseInstanceCache caseInstanceCache;
    @Inject PlanItemStore planItemStore;
    @Inject ObjectMapper objectMapper;
    @Inject CurrentPrincipal principal;

    @Transactional(TxType.REQUIRES_NEW)
    public void onSarOutcome(@Observes final SarOutcomeRecordedEvent event) {
        var caseId = event.caseId();
        var outcome = event.outcome();
        var tenantId = principal.tenancyId() != null
                ? principal.tenancyId()
                : TenancyConstants.DEFAULT_TENANT_ID;

        var instance = caseInstanceCache.get(caseId);
        if (instance == null) {
            LOG.warnf("No CaseInstance found for caseId=%s — skipping CBR profile store", caseId);
            return;
        }
        var ctx = instance.getCaseContext();

        // Extract transaction from case context
        SuspiciousTransaction tx;
        try {
            tx = objectMapper.convertValue(ctx.get("transaction"), SuspiciousTransaction.class);
        } catch (Exception e) {
            LOG.warnf(e, "Failed to deserialize transaction from case context for caseId=%s — skipping", caseId);
            return;
        }

        // Build CaseProfile — complete if all enrichment data available, initial otherwise
        var entityTypeStr = ctx.getString("entityType");
        var jurisdictionStr = ctx.getString("jurisdictionRisk");
        var networkStr = ctx.getString("networkComplexity");
        var entityRiskCountObj = ctx.getPath("priorEntityContext.entityRiskCount");
        int priorIncidentCount = entityRiskCountObj instanceof Number n ? n.intValue() : 0;

        CaseProfile profile;
        if (entityTypeStr != null && jurisdictionStr != null && networkStr != null) {
            profile = CaseProfile.complete(
                    tx.flagReason(), tx.amount(), priorIncidentCount,
                    EntityType.valueOf(entityTypeStr),
                    JurisdictionRisk.valueOf(jurisdictionStr),
                    NetworkComplexity.valueOf(networkStr));
        } else {
            profile = CaseProfile.initial(tx.flagReason(), tx.amount(), priorIncidentCount);
        }

        // Build investigation path from plan items
        var records = planItemStore.findByCaseId(caseId, tenantId);
        String path = records.stream()
                .filter(r -> r.status() == TaskStatus.COMPLETED || r.status() == TaskStatus.FAULTED)
                .filter(r -> r.executorName() != null)
                .sorted(Comparator.comparing(PlanItemRecord::createdAt))
                .map(PlanItemRecord::bindingName)
                .collect(Collectors.joining(" → "));
        if (path.isBlank()) {
            path = "(direct-verdict)";
        }

        // Build problem description
        String problem = String.format("Flagged transaction %s: %s from %s to %s, amount %s %s",
                tx.id(), tx.flagReason().name(),
                tx.originAccountId(), tx.destinationAccountId(),
                tx.amount().toPlainString(), tx.currency());

        // Build features map — merge CaseProfile features + optional SAR narrative
        var features = new LinkedHashMap<>(profile.toFeatures());
        var sarNarrative = ctx.getString("sarNarrative");
        if (sarNarrative != null) {
            features.put("sar_narrative", FeatureValue.string(sarNarrative));
        }

        var cbrCase = new FeatureVectorCbrCase(
                problem, path, outcome.verdict().name(),
                outcome.investigationAccuracyScore(), features);

        String entityId = UUID.nameUUIDFromBytes(
                ("aml-cbr:" + caseId).getBytes(StandardCharsets.UTF_8)).toString();

        // Store in CBR case base
        try {
            cbrStore.store(cbrCase, AmlCbrSchema.CASE_TYPE, entityId,
                    AmlMemoryDomains.CBR, tenantId, caseId.toString(), Path.root());
            LOG.infof("CBR case profile stored: caseId=%s outcome=%s", caseId, outcome.verdict());
        } catch (Exception e) {
            LOG.warnf(e, "CBR store failed for caseId=%s — skipping", caseId);
        }

        // Write ledger entry
        try {
            var entry = new AmlCaseProfileLedgerEntry();
            entry.flagReason = tx.flagReason().name();
            entry.transactionAmount = tx.amount();
            entry.priorIncidentCount = priorIncidentCount;
            entry.entityType = entityTypeStr;
            entry.jurisdictionRisk = jurisdictionStr;
            entry.networkComplexity = networkStr;
            entry.outcome = outcome.verdict().name();
            entry.confidence = outcome.investigationAccuracyScore();
            entry.investigationPath = path;
            entry.setSubjectId(caseId);
            entry.setActorId("aml-system");
            entry.setTenancyId(tenantId);
            entry.setEntryType(LedgerEntryType.ATTESTATION);

            // Link to SAR officer review entry if it exists
            var existingEntries = ledgerRepository.findBySubjectId(caseId, tenantId);
            existingEntries.stream()
                    .filter(e -> e instanceof io.casehub.aml.ledger.AmlSarOfficerReviewedLedgerEntry)
                    .max(Comparator.comparing(e -> e.getCreatedAt() != null ? e.getCreatedAt() : java.time.Instant.EPOCH))
                    .ifPresent(sarEntry -> entry.setCausedByEntryId(sarEntry.getId()));

            ledgerRepository.save(entry, tenantId);
            LOG.infof("CBR profile ledger entry written: caseId=%s", caseId);
        } catch (Exception e) {
            LOG.warnf(e, "Ledger write failed for CBR profile caseId=%s — skipping", caseId);
        }
    }
}
```

- [ ] **Step 4: Refine the test with actual CaseContext construction**

Study `AmlSarOutcomeObserverTest` and `AmlLayer6IntegrationTest` for:
- How to build a `CaseContext` with typed entries
- How to set up `PlanItemRecord` entries
- CDI exclusions needed for this test

Flesh out the test skeleton from Step 1 with:
1. A properly constructed `CaseContext` containing `"transaction"` (as a Map), `"entityType"`, `"jurisdictionRisk"`, `"networkComplexity"`, `"sarNarrative"`, and `"priorEntityContext"` with nested `"entityRiskCount"`
2. `PlanItemSaveRequest` calls to seed completed plan items in `PlanItemStore`
3. Additional test methods:
   - `onSarOutcome_partialProfile_missingEnrichment` — entityType is null → `CaseProfile.initial()` used
   - `onSarOutcome_nullSarNarrative_featureOmitted` — no `sar_narrative` in features map
   - `onSarOutcome_emptyPlanItems_directVerdictSentinel` — investigation path = `"(direct-verdict)"`
   - `onSarOutcome_cbrStoreFailure_ledgerStillWritten` — mock CbrCaseMemoryStore to throw → ledger entry still present
   - `onSarOutcome_nullTenancyId_usesDefault` — principal returns null → DEFAULT_TENANT_ID used

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -pl app -am -Dtest=AmlCaseProfileStoreObserverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 6: Verify with `ide_diagnostics`**

Run `ide_diagnostics` on `AmlCaseProfileStoreObserver.java`. Fix any issues.

- [ ] **Step 7: Run full test suite**

Run: `mvn test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All existing tests still pass. No regressions.

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/aml/cbr/AmlCaseProfileStoreObserver.java \
       app/src/test/java/io/casehub/aml/cbr/AmlCaseProfileStoreObserverTest.java
git commit -m "feat(#94): AmlCaseProfileStoreObserver — CBR retain on SAR outcome

Observes SarOutcomeRecordedEvent, extracts CaseProfile + investigation
path from engine context, stores via CbrCaseMemoryStore, and writes
AmlCaseProfileLedgerEntry with causedByEntryId chain to officer review.

Refs #94"
```
