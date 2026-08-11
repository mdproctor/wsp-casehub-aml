# CBR Retrieve Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #95 — feat: CBR Retrieve — similarity search against case base
**Issue group:** #95

**Goal:** Configure the engine's `CbrConfig` on the AML investigation case definition so similar past investigations are retrieved at case startup and injected into `CaseContext` under `"cbrExperiences"`.

**Architecture:** The engine's `CaseStartedEventHandler` (engine#761) already calls `CbrRetrievalService.retrieve()` when `CbrConfig` is present, extracting features via JQ, querying `CbrCaseMemoryStore.retrieveSimilar()`, and writing serialised `RetrievedExperience` objects into the case context. AML configures this via `CbrConfig` in the case definition's `augment()` method. The platform's generic retain observer is excluded since AML has domain-specific retain (#94).

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine 0.2-SNAPSHOT (with #761), casehub-neocortex-memory-cbr-inmem

## Global Constraints

- Engine SNAPSHOT already installed locally (20 Jul 01:11) with `CaseStartedEventHandler` CBR wiring
- `InMemoryCbrCaseMemoryStore` retains cases across `@QuarkusTest` methods — use `withNotBefore()` pattern (GE-20260716-986cd1)
- Hash chain disabled in tests (`casehub.ledger.hash-chain.enabled=false`)
- Drain engine to completion before assertions (PP-20260604-820c35)
- `CbrCaseMemoryStore.store()` 6th parameter is `caseType` not scope (GE-20260718-95e11e)
- All edits via IntelliJ MCP (`ide_edit_member`, `ide_insert_member`, `ide_replace_member`). Bash only for git/maven/config files.

---

### Task 1: CbrConfig on AmlInvestigationCaseHub + CbrCaseRetainObserver exclusion

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseHub.java` — add `setCbrConfig()` in `augment()`
- Modify: `app/src/main/resources/application.properties` — exclude `CbrCaseRetainObserver`
- Modify: `app/src/test/resources/application.properties` — exclude `CbrCaseRetainObserver`
- Modify: `app/src/test/java/io/casehub/aml/engine/AmlInvestigationCaseHubTest.java` — add CbrConfig assertion
- Test: `app/src/test/java/io/casehub/aml/engine/AmlInvestigationCaseHubTest.java`

**Interfaces:**
- Consumes: `CbrConfig.builder()` (engine-api), `AmlCbrSchema.CASE_TYPE` (app/cbr), `CbrRetrievalTiming` (engine-api)
- Produces: `CaseDefinition` with `getCbrConfig() != null` — engine's `CaseStartedEventHandler` reads this to trigger retrieval

- [ ] **Step 1: Write failing test for CbrConfig presence**

Add to `AmlInvestigationCaseHubTest`:

```java
@Test
void hasCbrConfig() {
    final var config = caseHub.getDefinition().getCbrConfig();
    assertNotNull(config, "CbrConfig must be set for CBR retrieval");
    assertEquals("aml-investigation", config.caseType());
    assertEquals("aml.cbr", config.domain());
    assertEquals(10, config.topK());
    assertEquals(0.5, config.minSimilarity(), 0.001);
    assertEquals(0.0, config.vectorWeight(), 0.001);
    assertEquals(CbrRetrievalTiming.CASE_LIFETIME, config.timing());
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
mvn test -pl app -am -Dtest=AmlInvestigationCaseHubTest#hasCbrConfig -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: FAIL — `getCbrConfig()` returns `null`

- [ ] **Step 3: Add CbrConfig to augment()**

In `AmlInvestigationCaseHub.augment()`, after `definition.getWorkers().addAll(descriptor.workers())`:

```java
definition.setCbrConfig(CbrConfig.builder()
        .feature("flag_reason", ".transaction.flagReason")
        .feature("transaction_amount", ".transaction.amount")
        .feature("prior_incident_count", ".priorEntityContext.entityRiskCount")
        .feature("entity_type", ".entityResolution.entityType")
        .domain("aml.cbr")
        .caseType(AmlCbrSchema.CASE_TYPE)
        .topK(10)
        .minSimilarity(0.5)
        .vectorWeight(0.0)
        .timing(CbrRetrievalTiming.CASE_LIFETIME)
        .cbrType("feature-vector")
        .weight("flag_reason", 0.30)
        .weight("transaction_amount", 0.15)
        .weight("prior_incident_count", 0.10)
        .weight("entity_type", 0.20)
        .build());
```

Imports needed: `io.casehub.api.model.cbr.CbrConfig`, `io.casehub.api.model.cbr.CbrConfig.CbrRetrievalTiming`, `io.casehub.aml.cbr.AmlCbrSchema`

- [ ] **Step 4: Exclude CbrCaseRetainObserver from both application.properties**

In `app/src/main/resources/application.properties`, append to `quarkus.arc.exclude-types`:

```
io.casehub.engine.internal.memory.CbrCaseRetainObserver
```

In `app/src/test/resources/application.properties`, append to `quarkus.arc.exclude-types`:

```
io.casehub.engine.internal.memory.CbrCaseRetainObserver
```

- [ ] **Step 5: Run test to verify it passes**

```bash
mvn test -pl app -am -Dtest=AmlInvestigationCaseHubTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: ALL PASS (including existing tests — no regression)

- [ ] **Step 6: Run full test suite**

```bash
mvn test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: ALL PASS — the CbrCaseRetainObserver exclusion must not break existing tests, and CbrConfig presence must not interfere with existing Layer 6/9 flows (empty case base → no experiences → no context change).

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseHub.java \
       app/src/main/resources/application.properties \
       app/src/test/resources/application.properties \
       app/src/test/java/io/casehub/aml/engine/AmlInvestigationCaseHubTest.java
git commit -m "feat(#95): configure CbrConfig for CBR retrieval at case startup

Add CbrConfig to AmlInvestigationCaseHub with 4 JQ features (flag_reason,
transaction_amount, prior_incident_count, entity_type) and matching weights.
CASE_LIFETIME timing retrieves once before binding evaluation.

Exclude CbrCaseRetainObserver — AML has domain-specific retain via
AmlCaseProfileStoreObserver (#94).

Refs #95"
```

---

### Task 2: End-to-end CBR retrieval integration test

**Files:**
- Create: `app/src/test/java/io/casehub/aml/cbr/AmlCbrRetrieveTest.java`

**Interfaces:**
- Consumes: `CbrCaseMemoryStore.store()` (neocortex), `AmlCbrSchema` (app/cbr), `AmlEngineCoordinator.startInvestigation()` (app/engine), `CaseInstanceCache.get()` (engine-common), `FeatureVectorCbrCase` (neocortex), `FeatureValue` (neocortex)
- Produces: Verified behaviour — stored CBR cases appear as `cbrExperiences` in the case context after investigation startup

- [ ] **Step 1: Write integration test**

```java
package io.casehub.aml.cbr;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.aml.cbr.AmlCbrSchema;
import io.casehub.aml.domain.FlagReason;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.engine.AmlEngineCoordinator;
import io.casehub.engine.common.spi.CaseInstanceCache;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.FeatureVectorCbrCase;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.neocortex.memory.api.MemoryDomain;
import io.casehub.aml.memory.AmlMemoryDomains;
import io.casehub.platform.api.path.Path;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.awaitility.Awaitility;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

@QuarkusTest
class AmlCbrRetrieveTest {

    @Inject CbrCaseMemoryStore cbrStore;
    @Inject AmlEngineCoordinator coordinator;
    @Inject CaseInstanceCache caseInstanceCache;

    @Test
    void similar_case_appears_in_context_after_investigation_start() {
        var features = new LinkedHashMap<String, FeatureValue>();
        features.put("flag_reason", FeatureValue.string("STRUCTURING"));
        features.put("transaction_amount", FeatureValue.number(50000.0));
        features.put("prior_incident_count", FeatureValue.number(2));
        features.put("entity_type", FeatureValue.string("CORPORATE"));

        var pastCase = new FeatureVectorCbrCase(
                "Past structuring case TX-PAST-001",
                "entity-resolution → pattern-analysis → sar-drafting",
                "UPHELD", 0.87, features);

        String entityId = UUID.nameUUIDFromBytes(
                "aml-cbr:test-past-case".getBytes()).toString();
        cbrStore.store(pastCase, AmlCbrSchema.CASE_TYPE, entityId,
                AmlMemoryDomains.CBR, "test-tenant",
                "test-past-case", Path.root());

        var tx = new SuspiciousTransaction(
                "TXN-CBR-" + UUID.randomUUID(),
                "ACC-CBR-A", "ACC-CBR-B",
                new BigDecimal("55000"), "USD",
                Instant.now(), FlagReason.STRUCTURING);

        UUID caseId = coordinator.startInvestigation(tx);

        Awaitility.await()
                .atMost(10, TimeUnit.SECONDS)
                .pollInterval(200, TimeUnit.MILLISECONDS)
                .untilAsserted(() -> {
                    var instance = caseInstanceCache.get(caseId);
                    assertThat(instance).isNotNull();
                    var ctx = instance.getCaseContext();
                    Object experiences = ctx.get("cbrExperiences");
                    assertThat(experiences).isNotNull()
                            .isInstanceOf(List.class);

                    @SuppressWarnings("unchecked")
                    var list = (List<Map<String, Object>>) experiences;
                    assertThat(list).isNotEmpty();
                    assertThat(list.get(0)).containsEntry("outcome", "UPHELD");
                    assertThat((double) list.get(0).get("similarityScore"))
                            .isGreaterThan(0.0);
                });
    }

    @Test
    void empty_case_base_produces_no_experiences() {
        Instant before = Instant.now();

        var tx = new SuspiciousTransaction(
                "TXN-EMPTY-" + UUID.randomUUID(),
                "ACC-EMPTY-A", "ACC-EMPTY-B",
                new BigDecimal("10000"), "USD",
                Instant.now(), FlagReason.VELOCITY_ANOMALY);

        UUID caseId = coordinator.startInvestigation(tx);

        Awaitility.await()
                .atMost(10, TimeUnit.SECONDS)
                .pollInterval(200, TimeUnit.MILLISECONDS)
                .untilAsserted(() -> {
                    var instance = caseInstanceCache.get(caseId);
                    assertThat(instance).isNotNull();
                    var ctx = instance.getCaseContext();
                    // Empty results → cbrExperiences not written (engine guard)
                    Object experiences = ctx.get("cbrExperiences");
                    assertThat(experiences).isNull();
                });
    }
}
```

- [ ] **Step 2: Run test to verify it passes**

```bash
mvn test -pl app -am -Dtest=AmlCbrRetrieveTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: PASS — both tests green. If `similar_case_appears_in_context_after_investigation_start` fails because the engine hasn't picked up the SNAPSHOT, rebuild the engine dependency first.

- [ ] **Step 3: Run full test suite for regression**

```bash
mvn test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false
```

Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git add app/src/test/java/io/casehub/aml/cbr/AmlCbrRetrieveTest.java
git commit -m "test(#95): end-to-end CBR retrieval integration test

Verify that stored CBR cases appear as cbrExperiences in the case
context after investigation startup. Covers: similarity match with
STRUCTURING past case, empty case base produces no experiences.

Refs #95"
```
