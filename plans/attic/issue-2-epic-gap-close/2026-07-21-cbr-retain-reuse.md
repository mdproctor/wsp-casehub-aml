# CBR Retain + Reuse Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #97 — feat: CBR Retain — outcome indexing on case completion
**Issue group:** #97, #96

**Goal:** Refactor CBR retention from SAR-only to all case outcomes using
`PlanCbrCase` with structured plan traces, add investigation triage flow
with non-SAR exit path, and implement CBR path advisor for reuse-informed
routing.

**Architecture:** Investigation triage inserts a decision point after specialist
analysis. Cases exit via SAR path (existing) or cleared path (new). A single
`CaseOutcomeObserver` retains all completed cases as `PlanCbrCase` with
`PlanTrace` data. A CBR path advisor worker analyses retrieved experiences
at case start and writes capability-oriented statistics that influence
downstream bindings.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine 0.2-SNAPSHOT,
casehub-neocortex-memory-api 0.2-SNAPSHOT

## Global Constraints

- `PlanCbrCase` (not `FeatureVectorCbrCase`) — carries `List<PlanTrace>` for structured path data
- `TriageDecision` enum (not `InvestigationOutcome`) — avoids collision with existing `InvestigationOutcome` record
- `CaseOutcomeObserver` SPI — `@ApplicationScoped` displaces `NoOpCaseOutcomeObserver @DefaultBean`
- `CbrCaseRetainObserver` stays excluded from CDI — AML owns domain-specific retention
- Flyway migrations in `db/aml-engine-ledger/migration/` — next versions V3006, V3007
- All `LedgerEntry` subclasses extend `JpaLedgerEntry` (not `LedgerEntry` directly)
- `CbrCaseMemoryStore.store()` parameter order: `(cbrCase, caseType, entityId, domain, tenantId, caseId, path)` — per GE-20260718-95e11e
- `CbrQuery` path scope parameter mandatory — per GE-20260717-0489d1
- Test isolation: `CbrQuery.withNotBefore(Instant.now())` — per GE-20260716-986cd1
- Spec discrepancy: `CaseOutcomeEvent.outcomeLabel` is `newState.name()` ("COMPLETED"), NOT the goal name. Goal name is in `metadata.goalName`. AML observer ignores `outcomeLabel` — reads `investigationTriage.decision` from `caseFileSnapshot`.

---

### Task 1: TriageDecision Enum

**Files:**
- Create: `api/src/main/java/io/casehub/aml/domain/TriageDecision.java`
- Test: `api/src/test/java/io/casehub/aml/domain/TriageDecisionTest.java`

**Interfaces:**
- Produces: `TriageDecision { SAR_WARRANTED, FALSE_POSITIVE, INCONCLUSIVE }` — used by Tasks 2, 3

- [ ] **Step 1: Write failing test**

```java
package io.casehub.aml.domain;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

import static org.junit.jupiter.api.Assertions.*;

class TriageDecisionTest {

    @ParameterizedTest
    @CsvSource({"SAR_WARRANTED", "FALSE_POSITIVE", "INCONCLUSIVE"})
    void valueOf_roundTrips(String name) {
        assertEquals(name, TriageDecision.valueOf(name).name());
    }

    @Test
    void values_hasExactlyThree() {
        assertEquals(3, TriageDecision.values().length);
    }

    @Test
    void valueOf_unknownThrows() {
        assertThrows(IllegalArgumentException.class,
                () -> TriageDecision.valueOf("SAR_FILED"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -am -Dtest=TriageDecisionTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `TriageDecision` class not found

- [ ] **Step 3: Create the enum**

Use `ide_create_file`:
```java
package io.casehub.aml.domain;

public enum TriageDecision {
    SAR_WARRANTED,
    FALSE_POSITIVE,
    INCONCLUSIVE
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl api -am -Dtest=TriageDecisionTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add api/src/main/java/io/casehub/aml/domain/TriageDecision.java api/src/test/java/io/casehub/aml/domain/TriageDecisionTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#97): TriageDecision enum — SAR_WARRANTED, FALSE_POSITIVE, INCONCLUSIVE

Refs #97"
```

---

### Task 2: Investigation Triage Flow

**Files:**
- Modify: `app/src/main/resources/aml/aml-investigation.yaml` — add triage capability/binding, cleared goal, anyOf completion, sar-drafting gate
- Create: `app/src/main/java/io/casehub/aml/cbr/InvestigationTriageWorker.java` — stub worker
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptor.java` — register triage worker
- Test: `app/src/test/java/io/casehub/aml/cbr/InvestigationTriageFlowTest.java`

**Interfaces:**
- Consumes: `TriageDecision` (Task 1)
- Produces: `investigationTriage` context variable `{ decision: "SAR_WARRANTED", reason: "..." }`; `investigation-cleared` goal; triage worker registered in descriptor

- [ ] **Step 1: Write the triage worker**

Use `ide_create_file` for `app/src/main/java/io/casehub/aml/cbr/InvestigationTriageWorker.java`:
```java
package io.casehub.aml.cbr;

import io.casehub.aml.domain.TriageDecision;
import io.casehub.engine.flow.FlowWorkerFunction;
import io.casehub.worker.api.Worker;

import java.util.Map;

import static io.serverlessworkflow.fluent.func.FuncWorkflowBuilder.workflow;
import static io.serverlessworkflow.fluent.func.dsl.FuncDSL.function;

public final class InvestigationTriageWorker {

    private InvestigationTriageWorker() {}

    public static Worker create() {
        return Worker.builder()
                     .name("investigation-triage-agent")
                     .capabilityName("investigation-triage")
                     .function(new FlowWorkerFunction(
                             workflow("investigation-triage")
                                     .tasks(
                                             function(s -> Map.of(
                                                     "decision", TriageDecision.SAR_WARRANTED.name(),
                                                     "reason", "stub — real triage logic pending #112"
                                             ), Map.class))
                                     .build()))
                     .build();
    }
}
```

- [ ] **Step 2: Register triage worker in descriptor**

Use `ide_edit_member` on `AmlInvestigationCaseDescriptor.workers()` to add `InvestigationTriageWorker.create()` to the list:

```java
List<Worker> workers() {
    return List.of(
            entityResolutionWorker(),
            patternAnalysisWorker(),
            osintScreeningWorker(),
            osintScreeningWorkerSenior(),
            seniorAnalystWorker(),
            InvestigationTriageWorker.create(),
            sarDraftingWorkerJunior(),
            sarDraftingWorkerSenior(),
            complianceReviewOpeningWorker()
    );
}
```

- [ ] **Step 3: Update YAML — add triage capability, binding, cleared goal, anyOf completion, sar-drafting gate**

Use `ide_read_file` then `ide_replace_text_in_file` for `app/src/main/resources/aml/aml-investigation.yaml`.

Add capability after `compliance-review-opening`:
```yaml
    - name: investigation-triage
      description: "Evaluate specialist findings and decide whether SAR is warranted"
      inputProjection: "{ entityResolution: .entityResolution, patternAnalysis: .patternAnalysis, osintScreening: .osintScreening, cbrPathAdvice: .cbrPathAdvice }"
      outputProjection: "{ investigationTriage: . }"
```

Replace goals section:
```yaml
  goals:
    - name: investigation-complete
      kind: success
      condition: ".complianceTaskId != null"
    - name: investigation-cleared
      kind: success
      condition: '.investigationTriage.decision == "FALSE_POSITIVE" or .investigationTriage.decision == "INCONCLUSIVE"'
```

Replace completion section:
```yaml
  completion:
    success:
      anyOf:
        - investigation-complete
        - investigation-cleared
```

Add triage binding before sar-drafting:
```yaml
    - name: investigation-triage
      on: { contextChange: {} }
      when: >-
        .entityResolution != null and
        .patternAnalysis != null and
        .osintScreening != null and
        .investigationTriage == null and
        (.cbrPathAdvice != null or .cbrExperiences == null or (.cbrExperiences | length) == 0)
      capability: investigation-triage
```

Modify sar-drafting binding condition to gate on triage:
```yaml
    - name: sar-drafting
      on: { contextChange: {} }
      when: >-
        .entityResolution != null and
        .patternAnalysis != null and
        .osintScreening != null and
        .investigationTriage.decision == "SAR_WARRANTED" and
        .sarNarrative == null
      capability: sar-drafting
```

- [ ] **Step 4: Write integration test — SAR path still works**

Use `ide_create_file` for `app/src/test/java/io/casehub/aml/cbr/InvestigationTriageFlowTest.java`:
```java
package io.casehub.aml.cbr;

import io.casehub.aml.domain.FlagReason;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.engine.AmlEngineCoordinator;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.service.WorkItemService;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.awaitility.Awaitility;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class InvestigationTriageFlowTest {

    @Inject AmlEngineCoordinator coordinator;
    @Inject CaseInstanceCache   caseInstanceCache;
    @Inject WorkItemService     workItemService;
    @PersistenceContext EntityManager defaultEm;

    private List<WorkItem> findGateWorkItems(UUID caseId) {
        return QuarkusTransaction.requiringNew().call(() ->
                defaultEm.createQuery(
                                 "SELECT w FROM WorkItem w WHERE w.callerRef LIKE :pattern",
                                 WorkItem.class)
                         .setParameter("pattern", "case:" + caseId + "/gate:%")
                         .getResultList());
    }

    private void awaitAndApproveGate(UUID caseId) {
        Awaitility.await()
                  .atMost(15, TimeUnit.SECONDS)
                  .pollInterval(300, TimeUnit.MILLISECONDS)
                  .until(() -> !findGateWorkItems(caseId).isEmpty());
        WorkItem gate = findGateWorkItems(caseId).get(0);
        workItemService.completeFromSystem(gate.id, "test-mlro", "approved");
    }

    private void drain(UUID caseId) {
        Awaitility.await().atMost(Duration.ofSeconds(20)).pollInterval(Duration.ofMillis(100))
                  .until(() -> "completed".equals(
                          given().when().get("/api/layer6/investigations/" + caseId)
                                 .then().extract().path("status")));
    }

    @Test
    void sarPath_triageStubReturnsSarWarranted_investigationCompletes() {
        SuspiciousTransaction tx = new SuspiciousTransaction(
                "TXN-TRIAGE-SAR-" + UUID.randomUUID(),
                "ACC-TRIAGE-O-" + UUID.randomUUID(),
                "ACC-TRIAGE-D-" + UUID.randomUUID(),
                new BigDecimal("80000"), "USD", Instant.now(), FlagReason.STRUCTURING);

        UUID caseId = coordinator.startInvestigation(tx);
        awaitAndApproveGate(caseId);
        drain(caseId);

        var instance = caseInstanceCache.get(caseId);
        var ctx = instance.getCaseContext();
        assertThat(ctx.get("investigationTriage")).isNotNull();
        @SuppressWarnings("unchecked")
        var triage = (java.util.Map<String, Object>) ctx.get("investigationTriage");
        assertThat(triage.get("decision")).isEqualTo("SAR_WARRANTED");
        assertThat(ctx.get("complianceTaskId")).isNotNull();
    }
}
```

- [ ] **Step 5: Run test**

Run: `mvn test -pl app -am -Dtest=InvestigationTriageFlowTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS — investigation completes through triage stub

- [ ] **Step 6: Verify with `ide_diagnostics`**

Run `ide_diagnostics` on all modified files to confirm no compilation errors.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/cbr/InvestigationTriageWorker.java app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptor.java app/src/main/resources/aml/aml-investigation.yaml app/src/test/java/io/casehub/aml/cbr/InvestigationTriageFlowTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#97): investigation triage flow — stub worker, SAR gate, cleared goal

Adds investigation-triage capability and binding that fires after all
specialist findings. Stub always returns SAR_WARRANTED. sar-drafting
now gated on triage decision. investigation-cleared goal enables
non-SAR case completion (FALSE_POSITIVE / INCONCLUSIVE).

Refs #97"
```

---

### Task 3: Retention Refactor — CaseOutcomeObserver + PlanCbrCase

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/cbr/AmlCaseProfileStoreObserver.java` — implement `CaseOutcomeObserver`, produce `PlanCbrCase`
- Modify: `app/src/main/java/io/casehub/aml/ledger/AmlCaseProfileLedgerEntry.java` — confidence `double` → `Double`
- Create: `app/src/main/resources/db/aml-engine-ledger/migration/V3006__case_profile_confidence_nullable.sql`
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseHub.java` — `cbrType("plan")`
- Modify: `app/src/test/java/io/casehub/aml/cbr/AmlCaseProfileStoreObserverTest.java` — adapt to CaseOutcomeEvent trigger
- Test: existing test class refactored

**Interfaces:**
- Consumes: `TriageDecision` (Task 1), `investigationTriage` context variable (Task 2)
- Produces: `PlanCbrCase` stored in `CbrCaseMemoryStore`, `AmlCaseProfileLedgerEntry` with `TriageDecision` outcome

- [ ] **Step 1: Create Flyway migration for nullable confidence**

Use `ide_create_file` for `app/src/main/resources/db/aml-engine-ledger/migration/V3006__case_profile_confidence_nullable.sql`:
```sql
ALTER TABLE aml_case_profile_ledger_entry ALTER COLUMN confidence SET DATA TYPE DOUBLE PRECISION;
ALTER TABLE aml_case_profile_ledger_entry ALTER COLUMN confidence DROP NOT NULL;
```

- [ ] **Step 2: Update AmlCaseProfileLedgerEntry — confidence nullable**

Use `ide_edit_member` on `AmlCaseProfileLedgerEntry.confidence`:
```java
@Column(name = "confidence", nullable = true)
public Double confidence;
```

Use `ide_edit_member` on `AmlCaseProfileLedgerEntry.domainContentBytes`:
```java
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
            confidence != null ? String.valueOf(confidence) : "",
            investigationPath != null ? investigationPath : ""
    ).getBytes(java.nio.charset.StandardCharsets.UTF_8);
}
```

- [ ] **Step 3: Update AmlInvestigationCaseHub — cbrType("plan")**

Use `ide_replace_text_in_file` on `AmlInvestigationCaseHub.java`:
Replace `.cbrType("feature-vector")` with `.cbrType("plan")`

- [ ] **Step 4: Refactor AmlCaseProfileStoreObserver — CaseOutcomeObserver + PlanCbrCase**

Use `ide_edit_member` to replace the full class. The observer now:
- Implements `CaseOutcomeObserver` instead of observing `SarOutcomeRecordedEvent`
- Reads `investigationTriage.decision` from `caseFileSnapshot` to derive `TriageDecision`
- Builds `PlanTrace` objects from `PlanItemStore` records
- Produces `PlanCbrCase` instead of `FeatureVectorCbrCase`
- Sets `confidence` to `null` (no officer accuracy score at completion time)

Full replacement for the class:
```java
package io.casehub.aml.cbr;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.aml.domain.CaseProfile;
import io.casehub.aml.domain.EntityType;
import io.casehub.aml.domain.JurisdictionRisk;
import io.casehub.aml.domain.NetworkComplexity;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.domain.TriageDecision;
import io.casehub.aml.ledger.AmlCaseProfileLedgerEntry;
import io.casehub.aml.memory.AmlMemoryDomains;
import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.CapabilityTarget;
import io.casehub.api.model.TaskStatus;
import io.casehub.api.spi.CaseOutcomeEvent;
import io.casehub.api.spi.CaseOutcomeObserver;
import io.casehub.engine.common.internal.model.PlanItemRecord;
import io.casehub.engine.common.spi.CaseDefinitionRegistry;
import io.casehub.engine.common.spi.PlanItemStore;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.neocortex.memory.cbr.PlanCbrCase;
import io.casehub.neocortex.memory.cbr.PlanTrace;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.platform.api.path.Path;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import jakarta.transaction.Transactional.TxType;
import org.jboss.logging.Logger;

import java.math.BigDecimal;
import java.nio.charset.StandardCharsets;
import java.util.Comparator;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.UUID;
import java.util.stream.Collectors;

@ApplicationScoped
public class AmlCaseProfileStoreObserver implements CaseOutcomeObserver {

    private static final Logger LOG = Logger.getLogger(AmlCaseProfileStoreObserver.class);

    private static final Map<TaskStatus, String> OUTCOME_MAP = Map.of(
            TaskStatus.COMPLETED, "SUCCESS",
            TaskStatus.FAULTED,   "FAILURE",
            TaskStatus.REJECTED,  "DECLINED",
            TaskStatus.CANCELLED, "CANCELLED",
            TaskStatus.OBSOLETE,  "OBSOLETE");

    @Inject CbrCaseMemoryStore    cbrStore;
    @Inject LedgerEntryRepository ledgerRepository;
    @Inject PlanItemStore         planItemStore;
    @Inject CaseDefinitionRegistry registry;
    @Inject ObjectMapper          objectMapper;
    @Inject CurrentPrincipal      principal;

    @Override
    @Transactional(TxType.REQUIRES_NEW)
    public void onOutcome(CaseOutcomeEvent event) {
        try {
            doRetain(event);
        } catch (Exception e) {
            LOG.warnf(e, "CBR retain failed for caseId=%s — skipping", event.caseId());
        }
    }

    @SuppressWarnings("unchecked")
    private void doRetain(CaseOutcomeEvent event) {
        var caseId   = event.caseId();
        var snapshot = event.caseFileSnapshot();
        var tenantId = event.tenancyId() != null
                       ? event.tenancyId()
                       : TenancyConstants.DEFAULT_TENANT_ID;

        var triageMap = (Map<String, Object>) snapshot.get("investigationTriage");
        if (triageMap == null) {
            LOG.warnf("No investigationTriage in snapshot for caseId=%s — skipping CBR retain", caseId);
            return;
        }
        TriageDecision triageDecision;
        try {
            triageDecision = TriageDecision.valueOf((String) triageMap.get("decision"));
        } catch (Exception e) {
            LOG.warnf("Invalid triage decision in snapshot for caseId=%s — skipping", caseId);
            return;
        }

        SuspiciousTransaction tx;
        try {
            tx = objectMapper.convertValue(snapshot.get("transaction"), SuspiciousTransaction.class);
        } catch (Exception e) {
            LOG.warnf(e, "Failed to deserialize transaction for caseId=%s — skipping", caseId);
            return;
        }

        var entityTypeStr      = snapshot.get("entityType") instanceof String s ? s : null;
        var jurisdictionStr    = snapshot.get("jurisdictionRisk") instanceof String s ? s : null;
        var networkStr         = snapshot.get("networkComplexity") instanceof String s ? s : null;
        var priorCtx           = snapshot.get("priorEntityContext");
        int priorIncidentCount = 0;
        if (priorCtx instanceof Map<?,?> m && m.get("entityRiskCount") instanceof Number n) {
            priorIncidentCount = n.intValue();
        }

        EntityType        entityType   = safeValueOf(EntityType.class, entityTypeStr);
        JurisdictionRisk  jurisdiction = safeValueOf(JurisdictionRisk.class, jurisdictionStr);
        NetworkComplexity network      = safeValueOf(NetworkComplexity.class, networkStr);

        CaseProfile profile;
        if (entityType != null && jurisdiction != null && network != null) {
            profile = CaseProfile.complete(
                    tx.flagReason(), tx.amount(), priorIncidentCount,
                    entityType, jurisdiction, network);
        } else {
            profile = CaseProfile.initial(tx.flagReason(), tx.amount(), priorIncidentCount);
        }

        var definition = registry.findByName(event.caseType()).orElse(null);
        var capabilityNameMap = definition != null ? buildRoutingKeyMap(definition) : Map.<String, String>of();

        var records = planItemStore.findByCaseId(caseId, tenantId);
        int[] index = {0};
        var traces = records.stream()
                .filter(r -> r.status().isTerminal())
                .filter(r -> capabilityNameMap.isEmpty() || capabilityNameMap.containsKey(r.bindingName()))
                .filter(r -> r.executorName() != null)
                .sorted(Comparator.comparing(PlanItemRecord::createdAt))
                .map(r -> new PlanTrace(r.bindingName(),
                        capabilityNameMap.getOrDefault(r.bindingName(), r.bindingName()),
                        r.executorName(),
                        OUTCOME_MAP.getOrDefault(r.status(), r.status().name()),
                        index[0]++, Map.of()))
                .toList();

        String solution = traces.stream()
                .map(t -> t.bindingName() + "→" + t.workerName() + "(" + t.stepOutcome() + ")")
                .collect(Collectors.joining(", "));
        if (solution.isBlank()) {
            solution = "(direct-verdict)";
        }

        String problem = String.format("Flagged transaction %s: %s from %s to %s, amount %s %s",
                tx.id(), tx.flagReason().name(),
                tx.originAccountId(), tx.destinationAccountId(),
                tx.amount().toPlainString(), tx.currency());

        var features = new LinkedHashMap<>(profile.toFeatures());
        var sarNarrative = snapshot.get("sarNarrative");
        if (sarNarrative instanceof String s) {
            features.put("sar_narrative", FeatureValue.string(s));
        }

        var cbrCase = new PlanCbrCase(problem, solution,
                triageDecision.name(), null, features, traces);

        String entityId = UUID.nameUUIDFromBytes(
                ("aml-cbr:" + caseId).getBytes(StandardCharsets.UTF_8)).toString();

        try {
            cbrStore.store(cbrCase, AmlCbrSchema.CASE_TYPE, entityId,
                    AmlMemoryDomains.CBR, tenantId, caseId.toString(), Path.root());
            LOG.infof("CBR case profile stored: caseId=%s outcome=%s traces=%d",
                    caseId, triageDecision, traces.size());
        } catch (Exception e) {
            LOG.warnf(e, "CBR store failed for caseId=%s — skipping", caseId);
        }

        try {
            var entry = new AmlCaseProfileLedgerEntry();
            entry.flagReason         = tx.flagReason().name();
            entry.transactionAmount  = tx.amount();
            entry.priorIncidentCount = priorIncidentCount;
            entry.entityType         = entityTypeStr;
            entry.jurisdictionRisk   = jurisdictionStr;
            entry.networkComplexity  = networkStr;
            entry.outcome            = triageDecision.name();
            entry.confidence         = null;
            entry.investigationPath  = solution;
            entry.subjectId          = caseId;
            entry.actorId            = "aml-system";
            entry.tenancyId          = tenantId;
            entry.entryType          = LedgerEntryType.ATTESTATION;
            ledgerRepository.save(entry, tenantId);
            LOG.infof("CBR profile ledger entry written: caseId=%s", caseId);
        } catch (Exception e) {
            LOG.warnf(e, "Ledger write failed for CBR profile caseId=%s — skipping", caseId);
        }
    }

    private Map<String, String> buildRoutingKeyMap(CaseDefinition definition) {
        var map = new LinkedHashMap<String, String>();
        for (Binding binding : definition.getBindings()) {
            if (binding.target() instanceof CapabilityTarget ct) {
                map.put(binding.getName(), ct.capability().name());
            }
        }
        return map;
    }

    private static <E extends Enum<E>> E safeValueOf(Class<E> enumType, String value) {
        if (value == null) return null;
        try {
            return Enum.valueOf(enumType, value);
        } catch (IllegalArgumentException e) {
            return null;
        }
    }
}
```

- [ ] **Step 5: Refactor existing tests for CaseOutcomeEvent trigger**

The existing `AmlCaseProfileStoreObserverTest` fires `SarOutcomeRecordedEvent` manually.
After the refactor, the observer fires automatically via the engine's `CaseOutcomeObserver`
mechanism when the case completes. Refactor tests to assert retention after `drain(caseId)`
instead of manually firing the event.

Update test methods:
- Remove `@Inject Event<SarOutcomeRecordedEvent> sarOutcomeEvent`
- `onSarOutcome_storesProfileAndLedgerEntry` → assert `AmlCaseProfileLedgerEntry` written after `drain()` with outcome `SAR_WARRANTED`
- `onSarOutcome_cbrStoreContainsCase` → assert `PlanCbrCase` (not `FeatureVectorCbrCase`) with `SAR_WARRANTED` outcome
- `onSarOutcome_noCaseInstance_doesNotThrow` → remove (no longer applicable — observer fires from engine, not manually)
- `onSarOutcome_partialProfile_missingEnrichment` → assert partial profile with `SAR_WARRANTED`
- Change all `FeatureVectorCbrCase.class` to `PlanCbrCase.class` in `retrieveSimilar` calls

- [ ] **Step 6: Run tests**

Run: `mvn test -pl app -am -Dtest=AmlCaseProfileStoreObserverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 7: Verify with `ide_diagnostics`**

Run `ide_diagnostics` on all modified files.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add -A
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#97): retention refactor — CaseOutcomeObserver + PlanCbrCase

Refactors AmlCaseProfileStoreObserver from SarOutcomeRecordedEvent to
CaseOutcomeObserver SPI. Produces PlanCbrCase with structured PlanTrace
entries. Derives TriageDecision from caseFileSnapshot. Confidence becomes
nullable (no officer score at completion time).

Refs #97"
```

---

### Task 4: CBR Path Advisor

**Files:**
- Create: `app/src/main/java/io/casehub/aml/cbr/CbrPathAdvisorWorker.java`
- Create: `app/src/main/java/io/casehub/aml/ledger/AmlCbrAdvisoryLedgerEntry.java`
- Create: `app/src/main/resources/db/aml-engine-ledger/migration/V3007__cbr_advisory_ledger_entry.sql`
- Modify: `app/src/main/resources/aml/aml-investigation.yaml` — add cbr-path-advisor capability/binding
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptor.java` — register advisor worker
- Test: `app/src/test/java/io/casehub/aml/cbr/CbrPathAdvisorTest.java`

**Interfaces:**
- Consumes: `cbrExperiences` context variable (engine-injected `List<Map<String, Object>>`)
- Produces: `cbrPathAdvice` context variable; `AmlCbrAdvisoryLedgerEntry`

- [ ] **Step 1: Create advisory ledger entry + migration**

Use `ide_create_file` for `app/src/main/java/io/casehub/aml/ledger/AmlCbrAdvisoryLedgerEntry.java`:
```java
package io.casehub.aml.ledger;

import io.casehub.ledger.runtime.model.jpa.JpaLedgerEntry;
import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

import java.nio.charset.StandardCharsets;

@Entity
@Table(name = "aml_cbr_advisory_ledger_entry")
@DiscriminatorValue("AML_CBR_ADVISORY")
public class AmlCbrAdvisoryLedgerEntry extends JpaLedgerEntry {

    @Column(name = "case_count", nullable = false)
    public int caseCount;

    @Column(name = "avg_similarity", nullable = false)
    public double avgSimilarity;

    @Column(name = "confidence", nullable = false)
    public double confidence;

    @Column(name = "predominant_outcome", length = 50)
    public String predominantOutcome;

    @Column(name = "predominant_outcome_frequency")
    public Double predominantOutcomeFrequency;

    @Column(name = "recommended_capabilities", length = 1000)
    public String recommendedCapabilities;

    @Override
    protected byte[] domainContentBytes() {
        return String.join("|",
                String.valueOf(caseCount),
                String.valueOf(avgSimilarity),
                String.valueOf(confidence),
                predominantOutcome != null ? predominantOutcome : "",
                predominantOutcomeFrequency != null ? String.valueOf(predominantOutcomeFrequency) : "",
                recommendedCapabilities != null ? recommendedCapabilities : ""
        ).getBytes(StandardCharsets.UTF_8);
    }
}
```

Use `ide_create_file` for `app/src/main/resources/db/aml-engine-ledger/migration/V3007__cbr_advisory_ledger_entry.sql`:
```sql
CREATE TABLE aml_cbr_advisory_ledger_entry (
    id UUID NOT NULL,
    case_count INT NOT NULL,
    avg_similarity DOUBLE PRECISION NOT NULL,
    confidence DOUBLE PRECISION NOT NULL,
    predominant_outcome VARCHAR(50),
    predominant_outcome_frequency DOUBLE PRECISION,
    recommended_capabilities VARCHAR(1000),
    CONSTRAINT fk_cbr_advisory_ledger_entry FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

- [ ] **Step 2: Add entity to Hibernate packages**

In test `application.properties`, the `quarkus.hibernate-orm.qhorus.packages` must include the new entity's package. It already includes `io.casehub.aml.ledger` — no change needed.

- [ ] **Step 3: Create the advisor worker**

Use `ide_create_file` for `app/src/main/java/io/casehub/aml/cbr/CbrPathAdvisorWorker.java`:
```java
package io.casehub.aml.cbr;

import io.casehub.aml.ledger.AmlCbrAdvisoryLedgerEntry;
import io.casehub.engine.flow.FlowWorkerFunction;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.worker.api.Worker;
import io.casehub.api.model.WorkerExecutionContext;
import org.jboss.logging.Logger;

import java.nio.charset.StandardCharsets;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.stream.Collectors;

import static io.serverlessworkflow.fluent.func.FuncWorkflowBuilder.workflow;
import static io.serverlessworkflow.fluent.func.dsl.FuncDSL.function;

public final class CbrPathAdvisorWorker {

    private static final Logger LOG = Logger.getLogger(CbrPathAdvisorWorker.class);

    private CbrPathAdvisorWorker() {}

    public static Worker create(LedgerEntryRepository ledgerRepository, CurrentPrincipal principal) {
        return Worker.builder()
                .name("cbr-path-advisor-agent")
                .capabilityName("cbr-path-advisor")
                .function(new FlowWorkerFunction(
                        workflow("cbr-path-advisor")
                                .tasks(function(s -> {
                                    try {
                                        return doAdvise(s, ledgerRepository, principal);
                                    } catch (Exception e) {
                                        LOG.warnf(e, "CBR path advisor failed — returning fallback");
                                        return Map.of(
                                                "caseCount", 0,
                                                "error", true,
                                                "errorReason", "advisor failed — proceeding without CBR advice");
                                    }
                                }, Map.class))
                                .build()))
                .build();
    }

    @SuppressWarnings("unchecked")
    private static Map<String, Object> doAdvise(Object input,
                                                 LedgerEntryRepository ledgerRepository,
                                                 CurrentPrincipal principal) {
        var inputMap = (Map<String, Object>) input;
        var experiences = (List<Map<String, Object>>) inputMap.get("cbrExperiences");
        if (experiences == null || experiences.isEmpty()) {
            return Map.of("caseCount", 0);
        }

        var capabilityStats = new LinkedHashMap<String, CapabilityStats>();
        var outcomeCounts   = new HashMap<String, Integer>();
        double totalScore   = 0;
        double minScore     = Double.MAX_VALUE;
        int    count        = 0;

        for (var experience : experiences) {
            double score = experience.get("similarityScore") instanceof Number n
                           ? n.doubleValue() : 0.0;
            String outcome = (String) experience.get("outcome");
            var planTrace = (List<Map<String, Object>>) experience.get("planTrace");

            totalScore += score;
            if (score < minScore) minScore = score;
            count++;

            if (outcome != null) {
                outcomeCounts.merge(outcome, 1, Integer::sum);
            }

            if (planTrace != null) {
                for (var step : planTrace) {
                    String capabilityName = (String) step.get("capabilityName");
                    String stepOutcome    = (String) step.get("stepOutcome");
                    if (capabilityName != null) {
                        capabilityStats.computeIfAbsent(capabilityName, k -> new CapabilityStats())
                                       .record(stepOutcome);
                    }
                }
            }
        }

        double avgSimilarity = count > 0 ? totalScore / count : 0;
        double confidence    = avgSimilarity * Math.min(1.0, (double) count / 5.0);

        var capabilities = new LinkedHashMap<String, Object>();
        for (var entry : capabilityStats.entrySet()) {
            var stats = entry.getValue();
            capabilities.put(entry.getKey(), Map.of(
                    "frequency", (double) stats.count / count,
                    "outcomes", Map.copyOf(stats.outcomes)));
        }

        String predominantOutcome = null;
        double predominantFreq    = 0;
        for (var entry : outcomeCounts.entrySet()) {
            double freq = (double) entry.getValue() / count;
            if (freq > predominantFreq) {
                predominantFreq    = freq;
                predominantOutcome = entry.getKey();
            }
        }

        var result = new LinkedHashMap<String, Object>();
        result.put("caseCount", count);
        result.put("minSimilarity", count > 0 ? minScore : 0.0);
        result.put("avgSimilarity", avgSimilarity);
        result.put("capabilities", capabilities);
        if (predominantOutcome != null) {
            result.put("predominantOutcome", predominantOutcome);
            result.put("predominantOutcomeFrequency", predominantFreq);
        }
        result.put("confidence", confidence);

        writeLedgerEntry(result, ledgerRepository, principal, capabilityStats, count);

        return result;
    }

    private static void writeLedgerEntry(Map<String, Object> advice,
                                          LedgerEntryRepository ledgerRepository,
                                          CurrentPrincipal principal,
                                          Map<String, CapabilityStats> capabilityStats,
                                          int caseCount) {
        try {
            UUID caseId  = WorkerExecutionContext.current().caseId();
            String tenantId = principal.tenancyId() != null
                              ? principal.tenancyId()
                              : TenancyConstants.DEFAULT_TENANT_ID;

            var entry = new AmlCbrAdvisoryLedgerEntry();
            entry.caseCount                  = (int) advice.get("caseCount");
            entry.avgSimilarity              = (double) advice.get("avgSimilarity");
            entry.confidence                 = (double) advice.get("confidence");
            entry.predominantOutcome         = (String) advice.get("predominantOutcome");
            entry.predominantOutcomeFrequency = advice.get("predominantOutcomeFrequency") instanceof Number n
                                                ? n.doubleValue() : null;
            entry.recommendedCapabilities    = capabilityStats.entrySet().stream()
                    .filter(e -> (double) e.getValue().count / caseCount > 0.5)
                    .map(Map.Entry::getKey)
                    .collect(Collectors.joining(","));
            entry.subjectId = UUID.nameUUIDFromBytes(
                    ("aml-cbr-advisory:" + caseId).getBytes(java.nio.charset.StandardCharsets.UTF_8));
            entry.actorId   = "aml-cbr-advisor";
            entry.tenancyId = tenantId;
            entry.entryType = LedgerEntryType.ATTESTATION;
            ledgerRepository.save(entry, tenantId);
        } catch (Exception e) {
            LOG.warnf(e, "Advisory ledger entry write failed — non-fatal");
        }
    }

    private static final class CapabilityStats {
        int count;
        final Map<String, Integer> outcomes = new HashMap<>();

        void record(String outcome) {
            count++;
            if (outcome != null) {
                outcomes.merge(outcome, 1, Integer::sum);
            }
        }
    }
}
```

- [ ] **Step 4: Register advisor worker in descriptor**

Use `ide_edit_member` on `AmlInvestigationCaseDescriptor`. Add `LedgerEntryRepository` and `CurrentPrincipal` as constructor parameters. Add `CbrPathAdvisorWorker.create(ledgerRepository, principal)` to workers list.

Update constructor:
```java
private final ComplianceReviewLifecycle complianceReviewLifecycle;
private final ObjectMapper              objectMapper;
private final LedgerEntryRepository     ledgerRepository;
private final CurrentPrincipal          principal;

public AmlInvestigationCaseDescriptor(
        final ComplianceReviewLifecycle complianceReviewLifecycle,
        final ObjectMapper objectMapper,
        final LedgerEntryRepository ledgerRepository,
        final CurrentPrincipal principal) {
    this.complianceReviewLifecycle = complianceReviewLifecycle;
    this.objectMapper              = objectMapper;
    this.ledgerRepository          = ledgerRepository;
    this.principal                 = principal;
}
```

Update workers list:
```java
List<Worker> workers() {
    return List.of(
            entityResolutionWorker(),
            patternAnalysisWorker(),
            osintScreeningWorker(),
            osintScreeningWorkerSenior(),
            seniorAnalystWorker(),
            InvestigationTriageWorker.create(),
            CbrPathAdvisorWorker.create(ledgerRepository, principal),
            sarDraftingWorkerJunior(),
            sarDraftingWorkerSenior(),
            complianceReviewOpeningWorker()
    );
}
```

Update `AmlInvestigationCaseHub.augment()` to pass the new dependencies:
```java
final var descriptor = new AmlInvestigationCaseDescriptor(
        complianceReviewLifecycle, objectMapper, ledgerRepository, principal);
```

Add `@Inject LedgerEntryRepository ledgerRepository` and `@Inject CurrentPrincipal principal` fields to `AmlInvestigationCaseHub`.

- [ ] **Step 5: Update YAML — add cbr-path-advisor capability and binding**

Add capability:
```yaml
    - name: cbr-path-advisor
      description: "Analyse similar past cases and produce path recommendations"
      inputProjection: "{ cbrExperiences: .cbrExperiences }"
      outputProjection: "{ cbrPathAdvice: . }"
```

Add binding (before investigation-triage):
```yaml
    - name: cbr-path-advisor
      on: { contextChange: {} }
      when: ".cbrExperiences != null and (.cbrExperiences | length) > 0 and .cbrPathAdvice == null"
      capability: cbr-path-advisor
```

- [ ] **Step 6: Write integration test**

Use `ide_create_file` for `app/src/test/java/io/casehub/aml/cbr/CbrPathAdvisorTest.java`:

Test that the advisor fires when experiences exist by pre-populating the store,
then starting an investigation and asserting `cbrPathAdvice` appears in context.

```java
package io.casehub.aml.cbr;

import io.casehub.aml.domain.FlagReason;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.engine.AmlEngineCoordinator;
import io.casehub.aml.ledger.AmlCbrAdvisoryLedgerEntry;
import io.casehub.aml.memory.AmlMemoryDomains;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.neocortex.memory.cbr.PlanCbrCase;
import io.casehub.neocortex.memory.cbr.PlanTrace;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.platform.api.path.Path;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.service.WorkItemService;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.awaitility.Awaitility;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.time.Duration;
import java.time.Instant;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class CbrPathAdvisorTest {

    @Inject AmlEngineCoordinator    coordinator;
    @Inject CbrCaseMemoryStore      cbrStore;
    @Inject CaseInstanceCache       caseInstanceCache;
    @Inject LedgerEntryRepository   ledgerRepository;
    @Inject WorkItemService         workItemService;
    @PersistenceContext EntityManager defaultEm;

    private void storePastCase(String outcome, boolean hadSeniorAnalyst) {
        var features = new LinkedHashMap<String, FeatureValue>();
        features.put("flag_reason", FeatureValue.string("STRUCTURING"));
        features.put("transaction_amount", FeatureValue.number(60000.0));
        features.put("prior_incident_count", FeatureValue.number(1));
        features.put("entity_type", FeatureValue.string("CORPORATE"));

        var traces = new java.util.ArrayList<>(List.of(
                new PlanTrace("entity-resolution", "entity-resolution",
                        "entity-resolution-agent", "SUCCESS", 0, Map.of()),
                new PlanTrace("pattern-analysis", "pattern-analysis",
                        "pattern-analysis-agent", "SUCCESS", 1, Map.of()),
                new PlanTrace("osint-screening", "osint-screening",
                        "osint-screening-agent", "DECLINED", 2, Map.of())
        ));
        if (hadSeniorAnalyst) {
            traces.add(new PlanTrace("senior-analyst-required-resolution", "senior-analyst-review",
                    "senior-analyst-agent", "SUCCESS", 3, Map.of()));
        }

        var pastCase = new PlanCbrCase(
                "Past structuring case TX-PAST-" + UUID.randomUUID(),
                traces.stream().map(t -> t.bindingName() + "→" + t.workerName()).reduce((a, b) -> a + ", " + b).orElse(""),
                outcome, 0.85, features, traces);

        cbrStore.store(pastCase, AmlCbrSchema.CASE_TYPE,
                "past-case-" + UUID.randomUUID(),
                AmlMemoryDomains.CBR,
                TenancyConstants.DEFAULT_TENANT_ID,
                UUID.randomUUID().toString(),
                Path.root());
    }

    private List<WorkItem> findGateWorkItems(UUID caseId) {
        return QuarkusTransaction.requiringNew().call(() ->
                defaultEm.createQuery("SELECT w FROM WorkItem w WHERE w.callerRef LIKE :pattern", WorkItem.class)
                        .setParameter("pattern", "case:" + caseId + "/gate:%")
                        .getResultList());
    }

    private void awaitAndApproveGate(UUID caseId) {
        Awaitility.await().atMost(15, TimeUnit.SECONDS).pollInterval(300, TimeUnit.MILLISECONDS)
                .until(() -> !findGateWorkItems(caseId).isEmpty());
        workItemService.completeFromSystem(findGateWorkItems(caseId).get(0).id, "test-mlro", "approved");
    }

    private void drain(UUID caseId) {
        Awaitility.await().atMost(Duration.ofSeconds(20)).pollInterval(Duration.ofMillis(100))
                .until(() -> "completed".equals(
                        given().when().get("/api/layer6/investigations/" + caseId)
                                .then().extract().path("status")));
    }

    @Test
    void advisor_firesWithExperiences_writesCbrPathAdvice() {
        for (int i = 0; i < 5; i++) {
            storePastCase("SAR_WARRANTED", i < 4);
        }

        SuspiciousTransaction tx = new SuspiciousTransaction(
                "TXN-ADVISOR-" + UUID.randomUUID(),
                "ACC-ADV-O-" + UUID.randomUUID(),
                "ACC-ADV-D-" + UUID.randomUUID(),
                new BigDecimal("55000"), "USD", Instant.now(), FlagReason.STRUCTURING);

        UUID caseId = coordinator.startInvestigation(tx);
        awaitAndApproveGate(caseId);
        drain(caseId);

        var instance = caseInstanceCache.get(caseId);
        var ctx = instance.getCaseContext();

        @SuppressWarnings("unchecked")
        var advice = (Map<String, Object>) ctx.get("cbrPathAdvice");
        assertThat(advice).isNotNull();
        assertThat(((Number) advice.get("caseCount")).intValue()).isGreaterThan(0);
        assertThat(advice.get("capabilities")).isNotNull();
        assertThat(((Number) advice.get("confidence")).doubleValue()).isGreaterThan(0);
    }

    @Test
    void advisor_writesAdvisoryLedgerEntry() {
        storePastCase("SAR_WARRANTED", true);

        SuspiciousTransaction tx = new SuspiciousTransaction(
                "TXN-ADVLEG-" + UUID.randomUUID(),
                "ACC-ADVLEG-O-" + UUID.randomUUID(),
                "ACC-ADVLEG-D-" + UUID.randomUUID(),
                new BigDecimal("55000"), "USD", Instant.now(), FlagReason.STRUCTURING);

        UUID caseId = coordinator.startInvestigation(tx);
        awaitAndApproveGate(caseId);
        drain(caseId);

        UUID advisorySubjectId = UUID.nameUUIDFromBytes(
                ("aml-cbr-advisory:" + caseId).getBytes(java.nio.charset.StandardCharsets.UTF_8));
        var entries = ledgerRepository.findBySubjectId(advisorySubjectId, TenancyConstants.DEFAULT_TENANT_ID);
        var advisory = entries.stream()
                .filter(AmlCbrAdvisoryLedgerEntry.class::isInstance)
                .map(AmlCbrAdvisoryLedgerEntry.class::cast)
                .findFirst().orElse(null);

        assertThat(advisory).isNotNull();
        assertThat(advisory.caseCount).isGreaterThan(0);
    }
}
```

- [ ] **Step 7: Run tests**

Run: `mvn test -pl app -am -Dtest=CbrPathAdvisorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add -A
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#96): CBR path advisor — capability-oriented experience analysis

CbrPathAdvisorWorker analyses retrieved CBR experiences and writes
cbrPathAdvice with per-capability frequency/outcome stats. Includes
failure fallback (error: true) for triage liveness. Writes
AmlCbrAdvisoryLedgerEntry for audit trail.

Refs #96"
```

---

### Task 5: CBR-Influenced Binding + End-to-End Tests

**Files:**
- Modify: `app/src/main/resources/aml/aml-investigation.yaml` — senior-analyst CBR condition
- Test: `app/src/test/java/io/casehub/aml/cbr/CbrRoutingIntegrationTest.java`

**Interfaces:**
- Consumes: `cbrPathAdvice` (Task 4), all prior tasks

- [ ] **Step 1: Update senior-analyst-required-resolution binding**

Use `ide_replace_text_in_file` on `aml-investigation.yaml`. Replace the
`senior-analyst-required-resolution` binding condition:

```yaml
    - name: senior-analyst-required-resolution
      on: { contextChange: {} }
      when: >-
        .entityResolution != null and
        .priorEntityContext.knownHighRisk != true and
        (.entityResolution.entityType == "PEP" or
         .entityResolution.riskScore > 0.8 or
         (.cbrPathAdvice != null and
          .cbrPathAdvice.capabilities["senior-analyst-review"].frequency > 0.6)) and
        .seniorAnalystReview == null
      capability: senior-analyst-review
```

- [ ] **Step 2: Write cold-start test**

Use `ide_create_file` for `app/src/test/java/io/casehub/aml/cbr/CbrRoutingIntegrationTest.java`:

```java
package io.casehub.aml.cbr;

import io.casehub.aml.domain.FlagReason;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.engine.AmlEngineCoordinator;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.service.WorkItemService;
import io.quarkus.narayana.jta.QuarkusTransaction;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.awaitility.Awaitility;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import static io.restassured.RestAssured.given;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class CbrRoutingIntegrationTest {

    @Inject AmlEngineCoordinator coordinator;
    @Inject CaseInstanceCache   caseInstanceCache;
    @Inject WorkItemService     workItemService;
    @PersistenceContext EntityManager defaultEm;

    private List<WorkItem> findGateWorkItems(UUID caseId) {
        return QuarkusTransaction.requiringNew().call(() ->
                defaultEm.createQuery("SELECT w FROM WorkItem w WHERE w.callerRef LIKE :pattern", WorkItem.class)
                        .setParameter("pattern", "case:" + caseId + "/gate:%")
                        .getResultList());
    }

    private void awaitAndApproveGate(UUID caseId) {
        Awaitility.await().atMost(15, TimeUnit.SECONDS).pollInterval(300, TimeUnit.MILLISECONDS)
                .until(() -> !findGateWorkItems(caseId).isEmpty());
        workItemService.completeFromSystem(findGateWorkItems(caseId).get(0).id, "test-mlro", "approved");
    }

    private void drain(UUID caseId) {
        Awaitility.await().atMost(Duration.ofSeconds(20)).pollInterval(Duration.ofMillis(100))
                .until(() -> "completed".equals(
                        given().when().get("/api/layer6/investigations/" + caseId)
                                .then().extract().path("status")));
    }

    @Test
    void coldStart_emptyCaseBase_investigationCompletesNormally() {
        SuspiciousTransaction tx = new SuspiciousTransaction(
                "TXN-COLD-" + UUID.randomUUID(),
                "ACC-COLD-O-" + UUID.randomUUID(),
                "ACC-COLD-D-" + UUID.randomUUID(),
                new BigDecimal("40000"), "USD", Instant.now(), FlagReason.VELOCITY_ANOMALY);

        UUID caseId = coordinator.startInvestigation(tx);
        awaitAndApproveGate(caseId);
        drain(caseId);

        var instance = caseInstanceCache.get(caseId);
        var ctx = instance.getCaseContext();
        assertThat(ctx.get("cbrPathAdvice")).isNull();
        assertThat(ctx.get("investigationTriage")).isNotNull();
        assertThat(ctx.get("complianceTaskId")).isNotNull();
    }
}
```

- [ ] **Step 3: Run all tests**

Run: `mvn test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: ALL PASS

- [ ] **Step 4: Verify with `ide_diagnostics`**

Run `ide_diagnostics` on modified YAML and all test files.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add -A
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#96): CBR-influenced routing — senior-analyst binding + integration tests

senior-analyst-required-resolution now fires when CBR advice shows >60%
of similar cases needed senior analyst review. Cold start graceful
degradation verified.

Refs #96"
```

- [ ] **Step 6: Run full build**

Run: `mvn verify -pl app -am`
Expected: BUILD SUCCESS — all tests pass, Flyway migrations apply, CDI wiring correct
