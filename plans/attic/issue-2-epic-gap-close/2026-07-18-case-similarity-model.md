# Case Similarity Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #93 — case similarity model
**Issue group:** #93

**Goal:** Deliver the AML-specific domain model for CBR case similarity — typed enums, CaseProfile record, feature extraction, schema registration — wiring into neocortex's existing CBR infrastructure.

**Architecture:** Domain enums and CaseProfile record in `api/` (pure domain, minimal dependency). CBR schema constants, feature extractor, and startup registrar in `app/`. All similarity scoring delegated to neocortex's `CbrSimilarityScorer` — no custom scorer.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-neocortex-memory-api (CBR types), casehub-neocortex-memory-cbr-inmem (test scope)

## Global Constraints

- IntelliJ MCP mandatory for all `.java` edits — `ide_create_file` for new files, `ide_edit_member`/`ide_replace_member` for modifications, `ide_refactor_rename` for renames
- Pre-release: breaking changes to `SuspiciousTransaction` are acceptable
- `CbrFeatureSchema`, `FeatureValue`, `CbrSimilarityScorer`, `SimilaritySpec` are in `casehub-neocortex-memory-api` (Tier 1, pure Java)
- `InMemoryCbrCaseMemoryStore` is in `casehub-neocortex-memory-cbr-inmem`
- Existing `AmlMemoryDomains` in `app/src/main/java/io/casehub/aml/memory/AmlMemoryDomains.java` — add `CBR` constant there
- GE-20260718-95e11e: `CbrCaseMemoryStore.store()` 6th param is `caseType` not `scope`
- GE-20260717-0489d1: `CbrQuery.of()` requires mandatory `Path scope` param
- Run `mvn -pl api -am test` for api/ tests, `mvn -pl app -am test -Dsurefire.failIfNoSpecifiedTests=false` for app/ tests

---

### Task 1: FlagReason Enum + SuspiciousTransaction Migration

**Files:**
- Create: `api/src/main/java/io/casehub/aml/domain/FlagReason.java`
- Create: `api/src/test/java/io/casehub/aml/domain/FlagReasonTest.java`
- Modify: `api/src/main/java/io/casehub/aml/domain/SuspiciousTransaction.java` — `String flagReason` → `FlagReason flagReason`
- Modify: `app/src/main/java/io/casehub/aml/simulation/AmlScenarioTemplate.java` — string literals → enum values
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptor.java` — PEP check from `contains("PEP")` → enum name equality
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlOversightCaseHub.java` — same PEP check
- Modify: `app/src/main/java/io/casehub/aml/memory/AmlSarOutcomeMemoryObserver.java` — string-to-enum in constructor
- Modify: ~20 test files — `new SuspiciousTransaction(...)` calls with string → `FlagReason.VALUE`
- Modify: `api/src/main/java/io/casehub/aml/domain/InvestigationSummaryResponse.java` — if it carries flagReason

**Interfaces:**
- Produces: `FlagReason` enum with values `STRUCTURING`, `LAYERING`, `SMURFING`, `ROUND_TRIP`, `PEP_MATCH`, `HIGH_RISK_JURISDICTION`, `VELOCITY_ANOMALY`, `LARGE_VOLUME`
- Produces: `SuspiciousTransaction.flagReason()` returns `FlagReason` (was `String`)

- [ ] **Step 1: Write FlagReason enum test**

```java
package io.casehub.aml.domain;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class FlagReasonTest {

    @Test
    void allValues_roundTrip() {
        for (FlagReason reason : FlagReason.values()) {
            assertEquals(reason, FlagReason.valueOf(reason.name()));
        }
    }

    @Test
    void expectedValues_present() {
        assertNotNull(FlagReason.valueOf("STRUCTURING"));
        assertNotNull(FlagReason.valueOf("LAYERING"));
        assertNotNull(FlagReason.valueOf("SMURFING"));
        assertNotNull(FlagReason.valueOf("ROUND_TRIP"));
        assertNotNull(FlagReason.valueOf("PEP_MATCH"));
        assertNotNull(FlagReason.valueOf("HIGH_RISK_JURISDICTION"));
        assertNotNull(FlagReason.valueOf("VELOCITY_ANOMALY"));
        assertNotNull(FlagReason.valueOf("LARGE_VOLUME"));
    }

    @Test
    void count_isEight() {
        assertEquals(8, FlagReason.values().length);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -pl api -am test -Dtest=FlagReasonTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `FlagReason` does not exist

- [ ] **Step 3: Create FlagReason enum**

Use `ide_create_file`:

```java
package io.casehub.aml.domain;

public enum FlagReason {
    STRUCTURING,
    LAYERING,
    SMURFING,
    ROUND_TRIP,
    PEP_MATCH,
    HIGH_RISK_JURISDICTION,
    VELOCITY_ANOMALY,
    LARGE_VOLUME
}
```

- [ ] **Step 4: Run FlagReasonTest — verify it passes**

Run: `mvn -pl api -am test -Dtest=FlagReasonTest`
Expected: 3 tests PASS

- [ ] **Step 5: Change SuspiciousTransaction.flagReason from String to FlagReason**

Use `ide_edit_member` on `SuspiciousTransaction` (member = `SuspiciousTransaction`):

```java
public record SuspiciousTransaction(
        String id,
        String originAccountId,
        String destinationAccountId,
        BigDecimal amount,
        String currency,
        Instant timestamp,
        FlagReason flagReason) {
}
```

- [ ] **Step 6: Update AmlScenarioTemplate enum constants**

Use `ide_edit_member` to update each enum constant's flagReason parameter. Mapping:

| Constant | Old string | New value |
|---|---|---|
| `PEP` | `"PEP_DETECTED"` | `FlagReason.PEP_MATCH` |
| `STRUCTURING` | `"STRUCTURING_PATTERN"` | `FlagReason.STRUCTURING` |
| `LOW_RISK` | `"ROUTINE_REVIEW"` | `FlagReason.VELOCITY_ANOMALY` |
| `SYSTEM_ERROR` | `"SYSTEM_FAILURE_TEST"` | `FlagReason.STRUCTURING` |
| `GATE_REJECTION` | `"HIGH_RISK_JURISDICTION"` | `FlagReason.HIGH_RISK_JURISDICTION` |
| `GDPR_ERASED` | `"PRIVACY_ERASURE_TEST"` | `FlagReason.STRUCTURING` |
| `HIGH_VOLUME` | `"LARGE_VOLUME_WIRE"` | `FlagReason.LARGE_VOLUME` |

Also change the constructor parameter type from `String` to `FlagReason`, field type, and the `toTransaction()`/`toTransactionWithUniqueId()` methods.

- [ ] **Step 7: Update AmlInvestigationCaseDescriptor PEP check**

The descriptor reads `flagReason` from the case context map as a `String`. Update the PEP check:

Old: `final boolean isPep = flagReason != null && flagReason.contains("PEP");`
New: `final boolean isPep = FlagReason.PEP_MATCH.name().equals(flagReason);`

Apply at both locations in the file (lines ~82 and anywhere else `isPep` is derived).

- [ ] **Step 8: Update AmlOversightCaseHub PEP check**

Same pattern as Step 7:

Old: `final boolean isPep = flagReason != null && flagReason.contains("PEP");`
New: `final boolean isPep = FlagReason.PEP_MATCH.name().equals(flagReason);`

- [ ] **Step 9: Update AmlSarOutcomeMemoryObserver**

This class constructs `SuspiciousTransaction` from event data. The `flagReason` field comes from a `String` in the event. Add `FlagReason.valueOf(...)` conversion:

Find the constructor call in `onSarOutcome` and update the last parameter from a raw string to `FlagReason.valueOf(flagReasonStr)`.

- [ ] **Step 10: Update InvestigationSummaryResponse**

Check if `InvestigationSummaryResponse` has `String flagReason` — if so, keep it as `String` (it's a REST response DTO; the enum name serializes to string via Jackson on the `SuspiciousTransaction` side, but the response may be a plain string field populated from the database view).

- [ ] **Step 11: Migrate all test constructor calls**

28 `new SuspiciousTransaction(...)` calls across test files. Each one's last parameter changes from a string literal to a `FlagReason` value. Use `ide_search_text` to find each, then `ide_replace_text_in_file` or `ide_edit_member` to update.

Common mappings in tests:
- `"high-velocity"` → `FlagReason.VELOCITY_ANOMALY`
- `"Structuring"` / `"structuring"` → `FlagReason.STRUCTURING`
- `"PEP_DETECTED"` → `FlagReason.PEP_MATCH`
- `"LAYERING_DETECTED"` → `FlagReason.LAYERING`
- `"HIGH_RISK_JURISDICTION"` → `FlagReason.HIGH_RISK_JURISDICTION`

- [ ] **Step 12: Build to verify compilation**

Run: `mvn -pl api,app -am compile`
Expected: BUILD SUCCESS — all call sites updated

- [ ] **Step 13: Run full test suite**

Run: `mvn -pl api,app -am test -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All existing tests pass with FlagReason enum

- [ ] **Step 14: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add api/src/main/java/io/casehub/aml/domain/FlagReason.java api/src/test/java/io/casehub/aml/domain/FlagReasonTest.java api/src/main/java/io/casehub/aml/domain/SuspiciousTransaction.java app/
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#93): FlagReason enum + SuspiciousTransaction type migration

Replace String flagReason with typed FlagReason enum across the
entire codebase. Updates all 28 constructor call sites, PEP detection
checks, and scenario templates.

Refs #93"
```

---

### Task 2: EntityType, JurisdictionRisk, NetworkComplexity Enums

**Files:**
- Create: `api/src/main/java/io/casehub/aml/domain/EntityType.java`
- Create: `api/src/main/java/io/casehub/aml/domain/JurisdictionRisk.java`
- Create: `api/src/main/java/io/casehub/aml/domain/NetworkComplexity.java`
- Create: `api/src/test/java/io/casehub/aml/domain/CbrEnumsTest.java`

**Interfaces:**
- Produces: `EntityType` with `INDIVIDUAL`, `CORPORATE`, `SHELL_COMPANY`, `PEP`
- Produces: `JurisdictionRisk` with `HIGH`, `MEDIUM`, `LOW`
- Produces: `NetworkComplexity` with `SINGLE_ENTITY`, `SMALL_NETWORK`, `LARGE_NETWORK`

- [ ] **Step 1: Write tests for all three enums**

```java
package io.casehub.aml.domain;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CbrEnumsTest {

    @Test
    void entityType_hasFourValues() {
        assertEquals(4, EntityType.values().length);
        assertNotNull(EntityType.valueOf("INDIVIDUAL"));
        assertNotNull(EntityType.valueOf("CORPORATE"));
        assertNotNull(EntityType.valueOf("SHELL_COMPANY"));
        assertNotNull(EntityType.valueOf("PEP"));
    }

    @Test
    void jurisdictionRisk_hasThreeValues() {
        assertEquals(3, JurisdictionRisk.values().length);
        assertNotNull(JurisdictionRisk.valueOf("HIGH"));
        assertNotNull(JurisdictionRisk.valueOf("MEDIUM"));
        assertNotNull(JurisdictionRisk.valueOf("LOW"));
    }

    @Test
    void networkComplexity_hasThreeValues() {
        assertEquals(3, NetworkComplexity.values().length);
        assertNotNull(NetworkComplexity.valueOf("SINGLE_ENTITY"));
        assertNotNull(NetworkComplexity.valueOf("SMALL_NETWORK"));
        assertNotNull(NetworkComplexity.valueOf("LARGE_NETWORK"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -pl api -am test -Dtest=CbrEnumsTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure

- [ ] **Step 3: Create the three enum files**

Use `ide_create_file` for each:

`EntityType.java`:
```java
package io.casehub.aml.domain;

public enum EntityType {
    INDIVIDUAL,
    CORPORATE,
    SHELL_COMPANY,
    PEP
}
```

`JurisdictionRisk.java`:
```java
package io.casehub.aml.domain;

public enum JurisdictionRisk {
    HIGH,
    MEDIUM,
    LOW
}
```

`NetworkComplexity.java`:
```java
package io.casehub.aml.domain;

public enum NetworkComplexity {
    SINGLE_ENTITY,
    SMALL_NETWORK,
    LARGE_NETWORK
}
```

- [ ] **Step 4: Run tests — verify pass**

Run: `mvn -pl api -am test -Dtest=CbrEnumsTest`
Expected: 3 tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add api/src/main/java/io/casehub/aml/domain/EntityType.java api/src/main/java/io/casehub/aml/domain/JurisdictionRisk.java api/src/main/java/io/casehub/aml/domain/NetworkComplexity.java api/src/test/java/io/casehub/aml/domain/CbrEnumsTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#93): EntityType, JurisdictionRisk, NetworkComplexity enums

CBR similarity dimensions for entity classification, FATF risk tier,
and counterparty graph complexity.

Refs #93"
```

---

### Task 3: CaseProfile Record with toFeatures()

**Files:**
- Modify: `api/pom.xml` — add `casehub-neocortex-memory-api` compile dependency
- Create: `api/src/main/java/io/casehub/aml/domain/CaseProfile.java`
- Create: `api/src/test/java/io/casehub/aml/domain/CaseProfileTest.java`

**Interfaces:**
- Consumes: `FlagReason`, `EntityType`, `JurisdictionRisk`, `NetworkComplexity` (from Tasks 1-2)
- Consumes: `FeatureValue` from `io.casehub.neocortex.memory.cbr`
- Produces: `CaseProfile.initial(FlagReason, BigDecimal, int) → CaseProfile`
- Produces: `CaseProfile.complete(FlagReason, BigDecimal, int, EntityType, JurisdictionRisk, NetworkComplexity) → CaseProfile`
- Produces: `CaseProfile.toFeatures() → Map<String, FeatureValue>`

- [ ] **Step 1: Add neocortex-memory-api dependency to api/pom.xml**

Add to `api/pom.xml` `<dependencies>`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-memory-api</artifactId>
    <version>${casehub.version}</version>
</dependency>
```

- [ ] **Step 2: Write CaseProfile tests**

```java
package io.casehub.aml.domain;

import io.casehub.neocortex.memory.cbr.FeatureValue;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class CaseProfileTest {

    @Test
    void initial_setsRequiredFields_nullsOptional() {
        CaseProfile profile = CaseProfile.initial(
                FlagReason.STRUCTURING, new BigDecimal("75000.00"), 3);

        assertEquals(FlagReason.STRUCTURING, profile.flagReason());
        assertEquals(new BigDecimal("75000.00"), profile.transactionAmount());
        assertEquals(3, profile.priorIncidentCount());
        assertNull(profile.entityType());
        assertNull(profile.jurisdiction());
        assertNull(profile.network());
    }

    @Test
    void complete_setsAllFields() {
        CaseProfile profile = CaseProfile.complete(
                FlagReason.PEP_MATCH, new BigDecimal("500000"), 5,
                EntityType.PEP, JurisdictionRisk.HIGH, NetworkComplexity.LARGE_NETWORK);

        assertEquals(FlagReason.PEP_MATCH, profile.flagReason());
        assertEquals(EntityType.PEP, profile.entityType());
        assertEquals(JurisdictionRisk.HIGH, profile.jurisdiction());
        assertEquals(NetworkComplexity.LARGE_NETWORK, profile.network());
    }

    @Test
    void toFeatures_initialProfile_emitsThreeDimensions() {
        CaseProfile profile = CaseProfile.initial(
                FlagReason.LAYERING, new BigDecimal("9500"), 0);

        Map<String, FeatureValue> features = profile.toFeatures();

        assertEquals(3, features.size());
        assertEquals(FeatureValue.string("LAYERING"), features.get("flag_reason"));
        assertEquals(FeatureValue.number(9500.0), features.get("transaction_amount"));
        assertEquals(FeatureValue.number(0.0), features.get("prior_incident_count"));
        assertFalse(features.containsKey("entity_type"));
        assertFalse(features.containsKey("jurisdiction_risk"));
        assertFalse(features.containsKey("network_complexity"));
    }

    @Test
    void toFeatures_completeProfile_emitsSixDimensions() {
        CaseProfile profile = CaseProfile.complete(
                FlagReason.SMURFING, new BigDecimal("200000"), 2,
                EntityType.SHELL_COMPANY, JurisdictionRisk.HIGH,
                NetworkComplexity.SMALL_NETWORK);

        Map<String, FeatureValue> features = profile.toFeatures();

        assertEquals(6, features.size());
        assertEquals(FeatureValue.string("SMURFING"), features.get("flag_reason"));
        assertEquals(FeatureValue.number(200000.0), features.get("transaction_amount"));
        assertEquals(FeatureValue.number(2.0), features.get("prior_incident_count"));
        assertEquals(FeatureValue.string("SHELL_COMPANY"), features.get("entity_type"));
        assertEquals(FeatureValue.string("HIGH"), features.get("jurisdiction_risk"));
        assertEquals(FeatureValue.string("SMALL_NETWORK"), features.get("network_complexity"));
    }

    @Test
    void initial_rejectsNullFlagReason() {
        assertThrows(NullPointerException.class, () ->
                CaseProfile.initial(null, new BigDecimal("100"), 0));
    }

    @Test
    void initial_rejectsNullAmount() {
        assertThrows(NullPointerException.class, () ->
                CaseProfile.initial(FlagReason.STRUCTURING, null, 0));
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn -pl api -am test -Dtest=CaseProfileTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `CaseProfile` does not exist

- [ ] **Step 4: Create CaseProfile record**

Use `ide_create_file`:

```java
package io.casehub.aml.domain;

import io.casehub.neocortex.memory.cbr.FeatureValue;

import java.math.BigDecimal;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Objects;

public record CaseProfile(
        FlagReason flagReason,
        BigDecimal transactionAmount,
        int priorIncidentCount,
        EntityType entityType,
        JurisdictionRisk jurisdiction,
        NetworkComplexity network
) {
    public CaseProfile {
        Objects.requireNonNull(flagReason, "flagReason");
        Objects.requireNonNull(transactionAmount, "transactionAmount");
    }

    public static CaseProfile initial(FlagReason flagReason, BigDecimal transactionAmount,
                                      int priorIncidentCount) {
        return new CaseProfile(flagReason, transactionAmount, priorIncidentCount,
                null, null, null);
    }

    public static CaseProfile complete(FlagReason flagReason, BigDecimal transactionAmount,
                                       int priorIncidentCount, EntityType entityType,
                                       JurisdictionRisk jurisdiction, NetworkComplexity network) {
        return new CaseProfile(flagReason, transactionAmount, priorIncidentCount,
                entityType, jurisdiction, network);
    }

    public Map<String, FeatureValue> toFeatures() {
        var map = new LinkedHashMap<String, FeatureValue>();
        map.put("flag_reason", FeatureValue.string(flagReason.name()));
        map.put("transaction_amount", FeatureValue.number(transactionAmount.doubleValue()));
        map.put("prior_incident_count", FeatureValue.number(priorIncidentCount));
        if (entityType != null) map.put("entity_type", FeatureValue.string(entityType.name()));
        if (jurisdiction != null) map.put("jurisdiction_risk", FeatureValue.string(jurisdiction.name()));
        if (network != null) map.put("network_complexity", FeatureValue.string(network.name()));
        return Map.copyOf(map);
    }
}
```

- [ ] **Step 5: Run CaseProfileTest — verify pass**

Run: `mvn -pl api -am test -Dtest=CaseProfileTest`
Expected: 6 tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add api/pom.xml api/src/main/java/io/casehub/aml/domain/CaseProfile.java api/src/test/java/io/casehub/aml/domain/CaseProfileTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#93): CaseProfile record with toFeatures() bridge

Typed case similarity profile with initial (3 dimensions) and complete
(6 dimensions) factory methods. toFeatures() maps to neocortex
FeatureValue map, skipping null fields for partial profiles.

Refs #93"
```

---

### Task 4: AmlCbrSchema + AmlMemoryDomains.CBR + AmlCbrSchemaRegistrar

**Files:**
- Modify: `app/pom.xml` — add `casehub-neocortex-memory-cbr-inmem` test dependency
- Modify: `app/src/main/java/io/casehub/aml/memory/AmlMemoryDomains.java` — add `CBR` constant
- Create: `app/src/main/java/io/casehub/aml/cbr/AmlCbrSchema.java`
- Create: `app/src/main/java/io/casehub/aml/cbr/AmlCbrSchemaRegistrar.java`
- Create: `app/src/test/java/io/casehub/aml/cbr/AmlCbrSchemaTest.java`
- Create: `app/src/test/java/io/casehub/aml/cbr/AmlCbrSchemaRegistrarTest.java`

**Interfaces:**
- Consumes: `CbrFeatureSchema`, `FeatureField`, `SimilaritySpec`, `CbrCaseMemoryStore`, `MemoryDomain` from neocortex
- Produces: `AmlCbrSchema.CASE_TYPE` = `"aml-investigation"`
- Produces: `AmlCbrSchema.SCHEMA` — `CbrFeatureSchema` with 6 fields
- Produces: `AmlCbrSchema.WEIGHTS` — `Map<String, Double>` summing to 1.0
- Produces: `AmlMemoryDomains.CBR` — `MemoryDomain("aml.cbr")`
- Produces: `AmlCbrSchemaRegistrar` — `@ApplicationScoped`, registers schema on `StartupEvent`

- [ ] **Step 1: Add memory-cbr-inmem test dependency to app/pom.xml**

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-memory-cbr-inmem</artifactId>
    <version>${casehub.version}</version>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Write AmlCbrSchemaTest**

```java
package io.casehub.aml.cbr;

import io.casehub.neocortex.memory.cbr.CbrFeatureSchema;
import io.casehub.neocortex.memory.cbr.FeatureField;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class AmlCbrSchemaTest {

    @Test
    void schema_hasSixFields() {
        CbrFeatureSchema schema = AmlCbrSchema.SCHEMA;
        assertEquals(6, schema.fields().size());
    }

    @Test
    void schema_caseType() {
        assertEquals("aml-investigation", AmlCbrSchema.SCHEMA.caseType());
    }

    @Test
    void schema_fieldNames() {
        var names = AmlCbrSchema.SCHEMA.fields().stream()
                .map(FeatureField::name).toList();
        assertTrue(names.contains("flag_reason"));
        assertTrue(names.contains("transaction_amount"));
        assertTrue(names.contains("prior_incident_count"));
        assertTrue(names.contains("entity_type"));
        assertTrue(names.contains("jurisdiction_risk"));
        assertTrue(names.contains("network_complexity"));
    }

    @Test
    void schema_hasFourCategoricalTwoNumeric() {
        long categoricals = AmlCbrSchema.SCHEMA.fields().stream()
                .filter(f -> f instanceof FeatureField.Categorical).count();
        long numerics = AmlCbrSchema.SCHEMA.fields().stream()
                .filter(f -> f instanceof FeatureField.Numeric).count();
        assertEquals(4, categoricals);
        assertEquals(2, numerics);
    }

    @Test
    void weights_sumToOne() {
        double sum = AmlCbrSchema.WEIGHTS.values().stream()
                .mapToDouble(Double::doubleValue).sum();
        assertEquals(1.0, sum, 0.001);
    }

    @Test
    void weights_coverAllSchemaFields() {
        for (FeatureField field : AmlCbrSchema.SCHEMA.fields()) {
            assertTrue(AmlCbrSchema.WEIGHTS.containsKey(field.name()),
                    "Missing weight for field: " + field.name());
        }
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn -pl app -am test -Dtest=AmlCbrSchemaTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure

- [ ] **Step 4: Add AmlMemoryDomains.CBR constant**

Use `ide_insert_member` on `AmlMemoryDomains`, position `last`:

```java
/** Case-Based Reasoning case profiles — similarity dimensions for investigation retrieval. */
public static final MemoryDomain CBR = new MemoryDomain("aml.cbr");
```

- [ ] **Step 5: Create AmlCbrSchema**

Use `ide_create_file`:

```java
package io.casehub.aml.cbr;

import io.casehub.aml.memory.AmlMemoryDomains;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.CbrFeatureSchema;
import io.casehub.neocortex.memory.cbr.FeatureField;
import io.casehub.neocortex.memory.cbr.SimilaritySpec;

import java.util.Map;

public final class AmlCbrSchema {

    public static final String CASE_TYPE = "aml-investigation";

    public static final MemoryDomain DOMAIN = AmlMemoryDomains.CBR;

    public static final CbrFeatureSchema SCHEMA = CbrFeatureSchema.of(CASE_TYPE,
            new FeatureField.Categorical("flag_reason",
                    new SimilaritySpec.CategoricalTable(Map.of(
                            "STRUCTURING", Map.of("SMURFING", 0.7, "LAYERING", 0.4, "ROUND_TRIP", 0.5),
                            "SMURFING", Map.of("LAYERING", 0.3, "ROUND_TRIP", 0.3),
                            "LAYERING", Map.of("ROUND_TRIP", 0.5),
                            "VELOCITY_ANOMALY", Map.of("LARGE_VOLUME", 0.6),
                            "PEP_MATCH", Map.of("HIGH_RISK_JURISDICTION", 0.3)))),
            new FeatureField.Categorical("entity_type",
                    new SimilaritySpec.CategoricalTable(Map.of(
                            "SHELL_COMPANY", Map.of("CORPORATE", 0.4, "PEP", 0.2),
                            "PEP", Map.of("INDIVIDUAL", 0.3, "CORPORATE", 0.1)))),
            new FeatureField.Numeric("transaction_amount", 0, 10_000_000,
                    new SimilaritySpec.GaussianDecay(0.15)),
            new FeatureField.Categorical("jurisdiction_risk",
                    new SimilaritySpec.CategoricalTable(Map.of(
                            "HIGH", Map.of("MEDIUM", 0.5, "LOW", 0.2),
                            "MEDIUM", Map.of("LOW", 0.5)))),
            new FeatureField.Numeric("prior_incident_count", 0, 20,
                    new SimilaritySpec.GaussianDecay(0.3)),
            new FeatureField.Categorical("network_complexity",
                    new SimilaritySpec.CategoricalTable(Map.of(
                            "SINGLE_ENTITY", Map.of("SMALL_NETWORK", 0.3, "LARGE_NETWORK", 0.1),
                            "SMALL_NETWORK", Map.of("LARGE_NETWORK", 0.5)))));

    public static final Map<String, Double> WEIGHTS = Map.of(
            "flag_reason", 0.30,
            "entity_type", 0.20,
            "transaction_amount", 0.15,
            "jurisdiction_risk", 0.15,
            "prior_incident_count", 0.10,
            "network_complexity", 0.10);

    private AmlCbrSchema() {}
}
```

- [ ] **Step 6: Run AmlCbrSchemaTest — verify pass**

Run: `mvn -pl app -am test -Dtest=AmlCbrSchemaTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: 6 tests PASS

- [ ] **Step 7: Write AmlCbrSchemaRegistrarTest**

```java
package io.casehub.aml.cbr;

import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.inmem.InMemoryCbrCaseMemoryStore;
import io.quarkus.runtime.StartupEvent;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class AmlCbrSchemaRegistrarTest {

    @Test
    void onStart_registersSchema_noError() {
        CbrCaseMemoryStore store = new InMemoryCbrCaseMemoryStore();
        AmlCbrSchemaRegistrar registrar = new AmlCbrSchemaRegistrar();
        registrar.cbrStore = store;

        assertDoesNotThrow(() -> registrar.onStart(new StartupEvent()));
    }
}
```

- [ ] **Step 8: Create AmlCbrSchemaRegistrar**

Use `ide_create_file`:

```java
package io.casehub.aml.cbr;

import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;

@ApplicationScoped
public class AmlCbrSchemaRegistrar {

    @Inject
    CbrCaseMemoryStore cbrStore;

    void onStart(@Observes StartupEvent ev) {
        cbrStore.registerSchema(AmlCbrSchema.SCHEMA);
    }
}
```

- [ ] **Step 9: Run AmlCbrSchemaRegistrarTest — verify pass**

Run: `mvn -pl app -am test -Dtest=AmlCbrSchemaRegistrarTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: 1 test PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/pom.xml app/src/main/java/io/casehub/aml/cbr/ app/src/test/java/io/casehub/aml/cbr/ app/src/main/java/io/casehub/aml/memory/AmlMemoryDomains.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#93): AmlCbrSchema, registrar, and CBR memory domain

CbrFeatureSchema with 4 categorical + 2 numeric fields, weighted
similarity tables for flag patterns and entity types, GaussianDecay
for amount and prior incidents. Schema auto-registered at startup.

Refs #93"
```

---

### Task 5: CaseProfileExtractor

**Files:**
- Create: `app/src/main/java/io/casehub/aml/cbr/CaseProfileExtractor.java`
- Create: `app/src/test/java/io/casehub/aml/cbr/CaseProfileExtractorTest.java`

**Interfaces:**
- Consumes: `SuspiciousTransaction` (Task 1), `AmlPriorContext` (existing), `CaseProfile` (Task 3)
- Consumes: `EntityType`, `JurisdictionRisk`, `NetworkComplexity` (Task 2)
- Produces: `CaseProfileExtractor.extractInitial(SuspiciousTransaction, AmlPriorContext) → CaseProfile`
- Produces: `CaseProfileExtractor.extractComplete(SuspiciousTransaction, AmlPriorContext, EntityType, JurisdictionRisk, NetworkComplexity) → CaseProfile`

- [ ] **Step 1: Write CaseProfileExtractorTest**

```java
package io.casehub.aml.cbr;

import io.casehub.aml.domain.*;
import io.casehub.aml.memory.AmlPriorContext;
import io.casehub.neocortex.memory.Memory;
import io.casehub.neocortex.memory.MemoryDomain;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class CaseProfileExtractorTest {

    private final CaseProfileExtractor extractor = new CaseProfileExtractor();

    private SuspiciousTransaction tx(FlagReason reason, BigDecimal amount) {
        return new SuspiciousTransaction("TX-1", "ACC-A", "ACC-B",
                amount, "USD", Instant.now(), reason);
    }

    private Memory memory(String id) {
        return new Memory(id, "entity-1", new MemoryDomain("aml.entity-risk"),
                "risk note", Instant.now(), Map.of());
    }

    @Test
    void extractInitial_noPriorHistory() {
        CaseProfile profile = extractor.extractInitial(
                tx(FlagReason.STRUCTURING, new BigDecimal("9500")),
                AmlPriorContext.empty());

        assertEquals(FlagReason.STRUCTURING, profile.flagReason());
        assertEquals(new BigDecimal("9500"), profile.transactionAmount());
        assertEquals(0, profile.priorIncidentCount());
        assertNull(profile.entityType());
        assertNull(profile.jurisdiction());
        assertNull(profile.network());
    }

    @Test
    void extractInitial_withPriorHistory() {
        AmlPriorContext prior = new AmlPriorContext(
                List.of(memory("m1"), memory("m2"), memory("m3"), memory("m4"), memory("m5")),
                List.of(), List.of());

        CaseProfile profile = extractor.extractInitial(
                tx(FlagReason.PEP_MATCH, new BigDecimal("500000")),
                prior);

        assertEquals(5, profile.priorIncidentCount());
    }

    @Test
    void extractComplete_allDimensions() {
        CaseProfile profile = extractor.extractComplete(
                tx(FlagReason.LAYERING, new BigDecimal("1000000")),
                AmlPriorContext.empty(),
                EntityType.SHELL_COMPANY, JurisdictionRisk.HIGH,
                NetworkComplexity.LARGE_NETWORK);

        assertEquals(FlagReason.LAYERING, profile.flagReason());
        assertEquals(new BigDecimal("1000000"), profile.transactionAmount());
        assertEquals(0, profile.priorIncidentCount());
        assertEquals(EntityType.SHELL_COMPANY, profile.entityType());
        assertEquals(JurisdictionRisk.HIGH, profile.jurisdiction());
        assertEquals(NetworkComplexity.LARGE_NETWORK, profile.network());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -pl app -am test -Dtest=CaseProfileExtractorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure

- [ ] **Step 3: Create CaseProfileExtractor**

Use `ide_create_file`:

```java
package io.casehub.aml.cbr;

import io.casehub.aml.domain.*;
import io.casehub.aml.memory.AmlPriorContext;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class CaseProfileExtractor {

    public CaseProfile extractInitial(SuspiciousTransaction tx, AmlPriorContext priorContext) {
        return CaseProfile.initial(
                tx.flagReason(),
                tx.amount(),
                priorContext.entityRisk().size());
    }

    public CaseProfile extractComplete(SuspiciousTransaction tx, AmlPriorContext priorContext,
                                       EntityType entityType, JurisdictionRisk jurisdiction,
                                       NetworkComplexity network) {
        return CaseProfile.complete(
                tx.flagReason(),
                tx.amount(),
                priorContext.entityRisk().size(),
                entityType, jurisdiction, network);
    }
}
```

- [ ] **Step 4: Run CaseProfileExtractorTest — verify pass**

Run: `mvn -pl app -am test -Dtest=CaseProfileExtractorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: 3 tests PASS

- [ ] **Step 5: Run full test suite to verify nothing broken**

Run: `mvn -pl api,app -am test -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/cbr/CaseProfileExtractor.java app/src/test/java/io/casehub/aml/cbr/CaseProfileExtractorTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#93): CaseProfileExtractor — initial and complete extraction

Thin CDI bean composing SuspiciousTransaction + AmlPriorContext into
CaseProfile. extractInitial for Retrieve (3 dims), extractComplete
for Retain (6 dims).

Refs #93"
```
