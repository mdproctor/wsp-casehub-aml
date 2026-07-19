# Case Profile Store Design — #94

## Summary

Store completed AML investigation case profiles into the CBR case base for future similarity retrieval. On SAR outcome recording, extract the complete `CaseProfile`, investigation path, and SAR narrative, store them via `CbrCaseMemoryStore`, and write a tamper-evident `LedgerEntry` for compliance audit.

## Context

Issue #94 is the second issue in the CBR epic (#92). It depends on #93 (case similarity model) which delivered `CaseProfile`, `CaseProfileExtractor`, and `AmlCbrSchema`. This issue delivers the CBR **Retain** step — persisting completed investigations as retrievable cases.

### What Already Exists

| Component | Location | What it provides |
|-----------|----------|------------------|
| `CaseProfile` | `api/.../domain/` | 6-dimension record with `toFeatures()` → `Map<String, FeatureValue>` |
| `CaseProfileExtractor` | `app/.../cbr/` | Extracts `initial()` and `complete()` profiles from transaction + prior context |
| `AmlCbrSchema` | `app/.../cbr/` | `CbrFeatureSchema` with 6 fields, similarity specs, weights. Registered at startup |
| `AmlMemoryDomains.CBR` | `app/.../memory/` | `MemoryDomain("aml.cbr")` |
| `CbrCaseMemoryStore` | neocortex `memory-api` | Platform SPI: `store()`, `retrieveSimilar()`, `recordOutcome()`, `registerSchema()` |
| `FeatureVectorCbrCase` | neocortex `memory-api` | Generic `CbrCase` impl: problem/solution/outcome/confidence/features |
| `SarOutcomeRecordedEvent` | `app/.../engine/` | CDI event fired synchronously (`.fire()`) on SAR outcome recording |
| `AmlSarOutcomeMemoryObserver` | `app/.../memory/` | Existing observer writing entity-level memories on SAR outcome (uses `@Observes`, `@Transactional(REQUIRES_NEW)`) |
| `CaseInstanceCache` | engine-common SPI | Cache for retrieving `CaseInstance` by case UUID |
| `PlanItemStore` | engine-common SPI | Durable store for plan item records: `findByCaseId(UUID, String)` → `List<PlanItemRecord>` |
| `CbrCaseRetainObserver` | engine runtime | Platform-level generic CBR retain — fires on `CaseOutcomeEvent`, uses `CbrConfig` from case definition. **Not used for AML** — AML requires domain-specific feature extraction (`CaseProfile`), SAR-specific outcome labels, and a compliance ledger entry that the generic observer cannot provide |

### Platform Coherence

- **Boundary rule**: "Do not implement CBR retrieval logic in application repos." Storage (Retain) uses the platform SPI — no retrieval logic here.
- **CBR architecture** (platform/cbr.md §4): Retain = `CaseMemoryStore` + `MemoryEmitter`. This design uses `CbrCaseMemoryStore` (the CBR-specific store), not `CaseMemoryStore` (entity-level). Correct for case-level indexing.
- **Protocol**: `aml-ledger-entry-tenancy-id-non-null` — all LedgerEntry writes must null-guard `tenancyId`.
- **No `CbrConfig`**: The AML case definition must NOT configure `CbrConfig`, which would activate the platform's `CbrCaseRetainObserver` and produce duplicate CBR entries. AML handles retain independently via `AmlCaseProfileStoreObserver`.

## Design

### 1. `AmlCaseProfileStoreObserver`

**Location:** `app/src/main/java/io/casehub/aml/cbr/AmlCaseProfileStoreObserver.java`

**Trigger:** `@Observes SarOutcomeRecordedEvent`

The event is dispatched synchronously via `Event.fire()` in `AmlLayer6Resource.recordOutcome()`. CDI spec: `.fire()` dispatches to `@Observes` handlers only — `@ObservesAsync` handlers require `.fireAsync()` and will not execute.

Rationale for this lifecycle point: the SAR verdict (UPHELD/WITHDRAWN/FLAGGED) is the quality signal that makes the case base useful. Cases without a verdict don't contribute learning value.

**Transaction strategy:** `@Transactional(TxType.REQUIRES_NEW)` — identical to the established pattern in `AmlSarOutcomeMemoryObserver`. Isolates the observer's writes from the caller's request transaction. The CBR store and ledger write use independent datasources (CBR store uses the default datasource via `CbrCaseMemoryStore`; `LedgerEntryRepository` uses the qhorus datasource with its own `@PersistenceContext`), so a failure in one does not roll back the other. Each operation is wrapped in an independent try/catch — memory/ledger failures must not propagate (established AML convention).

**Dependencies:**
- `CbrCaseMemoryStore` — store the case
- `LedgerEntryRepository` — write the tamper-evident entry
- `CaseInstanceCache` — retrieve case context for enrichment data
- `PlanItemStore` — retrieve completed plan items for investigation path
- `ObjectMapper` — deserialize `SuspiciousTransaction` from case context
- `CurrentPrincipal` — tenancy context (with null-guard per protocol)

**Data access (from case context):**

The observer retrieves enrichment data from the engine's `CaseContext` via `CaseInstanceCache.get(caseId)`. The `AmlEngineCoordinator` sets the initial context at investigation start; specialist workers enrich it during execution.

| Data | Context path | Set by |
|------|-------------|--------|
| `SuspiciousTransaction` | `caseContext.get("transaction")` → deserialize via `ObjectMapper.convertValue(map, SuspiciousTransaction.class)` | `AmlEngineCoordinator.startInvestigation()` |
| Prior incident count | `caseContext.getPath("priorEntityContext.entityRiskCount")` → `int` | `AmlEngineCoordinator.startInvestigation()` (via `AmlPriorContext.toContextMap()`) |
| `EntityType` | `caseContext.getString("entityType")` → `EntityType.valueOf(...)` | Entity-resolution worker |
| `JurisdictionRisk` | `caseContext.getString("jurisdictionRisk")` → `JurisdictionRisk.valueOf(...)` | OSINT-screening worker |
| `NetworkComplexity` | `caseContext.getString("networkComplexity")` → `NetworkComplexity.valueOf(...)` | Pattern-analysis worker |
| SAR narrative | `caseContext.getString("sarNarrative")` | SAR-drafting worker |

**Flow:**
1. Retrieve `CaseInstance` via `caseInstanceCache.get(caseId)` and read `CaseContext`
2. Deserialize `SuspiciousTransaction` from `caseContext.get("transaction")`
3. Read `entityRiskCount` from `caseContext.getPath("priorEntityContext.entityRiskCount")`
4. Read enrichment dimensions from context keys. Build `CaseProfile` directly: if all three enrichment dimensions are available, call `CaseProfile.complete(tx.flagReason(), tx.amount(), entityRiskCount, entityType, jurisdiction, network)`; otherwise call `CaseProfile.initial(tx.flagReason(), tx.amount(), entityRiskCount)`. `CaseProfileExtractor` is not used — its methods require `AmlPriorContext` which is non-recoverable from the case context (see §6), and the observer already has the three values those methods extract (`flagReason`, `amount`, `entityRiskCount`)
5. Build investigation path string from `PlanItemStore` (see §5)
6. Build `FeatureVectorCbrCase`:
   - `problem` = flagged transaction description (transaction type, accounts, amount)
   - `solution` = investigation path (e.g. `"entity-resolution → pattern-analysis → osint-screening → sar-drafting → compliance-review"`). If no plan items are COMPLETED or FAULTED (edge case: direct manual verdict without worker execution), uses `"(direct-verdict)"` — `FeatureVectorCbrCase` rejects blank solutions via `isBlank()` check
   - `outcome` = `SarVerdict.name()` (UPHELD / WITHDRAWN / FLAGGED)
   - `confidence` = `SarOutcome.investigationAccuracyScore()`
   - `features` = `CaseProfile.toFeatures()`, merged with `{"sar_narrative": FeatureValue.string(sarNarrative)}` only if `sarNarrative` is non-null. When the sar-drafting worker was skipped or failed, the `sar_narrative` feature is omitted — `FeatureValue.string(null)` throws NPE (`StringVal` requires non-null), and embedding an empty string would produce a meaningless vector for semantic similarity. The schema field is present but the feature is sparse.
7. Call `cbrCaseMemoryStore.store(cbrCase, AmlCbrSchema.CASE_TYPE, entityId.toString(), AmlMemoryDomains.CBR, tenantId, caseId.toString(), Path.root())`
8. Write `AmlCaseProfileLedgerEntry` with `causedByEntryId` linking to the SAR officer review ledger entry (see §3)
9. Both calls (steps 7–8) wrapped in independent try/catch — memory failures must not propagate (established AML convention)

**Entity ID for the CBR store:** `UUID.nameUUIDFromBytes(("aml-cbr:" + caseId).getBytes(UTF_8)).toString()` — namespaced to isolate CBR memory entries from engine entries for the same case in the memory store. Explicit `.toString()` conversion since `CbrCaseMemoryStore.store()` takes `String` parameters. Note: this is the CBR **memory store** entity ID, not the ledger entry `subjectId` — the ledger entry uses raw `caseId` to match the established convention (see §3).

**Scope:** `Path.root()` — no hierarchical scoping needed for AML CBR cases. Consistent with the platform's `CbrCaseRetainObserver` convention.

### 2. `AmlCbrSchema` Update

Add `FeatureField.semanticText("sar_narrative")` to the existing schema definition. This enables semantic similarity matching on the SAR narrative during CBR Retrieve (#95). The `semanticText()` factory creates `new Text(name, true)` with `semantic=true`, enabling vector-based similarity. The plain `text()` factory creates `semantic=false` (keyword matching only), which would not support the stated goal.

No weight needed in `WEIGHTS` — text fields use vector similarity via the `vectorWeight` parameter in `CbrQuery`, not per-field weights.

### 3. `AmlCaseProfileLedgerEntry`

**Location:** `app/src/main/java/io/casehub/aml/ledger/AmlCaseProfileLedgerEntry.java`

**Extends:** `JpaLedgerEntry` (not `LedgerEntry` — per GE-20260707-99de4f)

**Table:** `aml_case_profile_ledger_entry`

**Discriminator:** `AML_CASE_PROFILE`

**Columns:**

| Column | Type | Nullable | Purpose |
|--------|------|----------|---------|
| `flag_reason` | `VARCHAR(50)` | NOT NULL | `FlagReason` enum name |
| `transaction_amount` | `DECIMAL(19,4)` | NOT NULL | Original flagged transaction amount |
| `prior_incident_count` | `INTEGER` | NOT NULL | Entity risk history count at investigation time |
| `entity_type` | `VARCHAR(50)` | NULL | May be null for `initial()` profiles |
| `jurisdiction_risk` | `VARCHAR(50)` | NULL | May be null for `initial()` profiles |
| `network_complexity` | `VARCHAR(50)` | NULL | May be null for `initial()` profiles |
| `outcome` | `VARCHAR(50)` | NOT NULL | `SarVerdict` name |
| `confidence` | `DOUBLE` | NOT NULL | Investigation accuracy score |
| `investigation_path` | `VARCHAR(1000)` | NOT NULL | Worker execution sequence |

**`domainContentBytes()`:** Pipe-delimited UTF-8 of all non-transient fields (per ledger SNAPSHOT `domainContentBytes()` enforcement).

**`subjectId`:** `caseId` (raw case UUID) — all AML investigation ledger entries share `subjectId = caseId` so that `findBySubjectId(caseId, tenancyId)` returns the complete audit trail for an investigation. This is the established convention documented in `AmlCaseOpenedLedgerEntry`: "subjectId on this entry equals the case UUID, shared with all other ledger entries for the same investigation." The CBR memory store entity ID (`UUID.nameUUIDFromBytes("aml-cbr:" + caseId)`) is a separate concern — it namespaces entries within the memory store, not the ledger.

**`causedByEntryId`:** Links to the `AmlSarOfficerReviewedLedgerEntry` for this case, closing the evidence chain from officer review → CBR profile storage. Lookup: query `LedgerEntryRepository.findBySubjectId(caseId, tenancyId)`, filter for `AmlSarOfficerReviewedLedgerEntry`, take the most recent by `createdAt`. If no officer review entry exists (e.g., SAR outcome recorded without compliance review), `causedByEntryId` is `null`.

### 4. Flyway V3005

**Location:** `app/src/main/resources/db/aml-engine-ledger/migration/V3005__case_profile_ledger_entry.sql`

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

### 5. Investigation Path Extraction

The observer injects `PlanItemStore` (engine-common SPI) to retrieve plan items by case ID. The investigation path is built from plan items that were actually executed (COMPLETED or FAULTED), sorted by creation time:

```java
PlanItemStore planItemStore = ...;
List<PlanItemRecord> records = planItemStore.findByCaseId(caseId, tenancyId);

String path = records.stream()
    .filter(r -> r.status() == TaskStatus.COMPLETED || r.status() == TaskStatus.FAULTED)
    .filter(r -> r.executorName() != null)
    .sorted(Comparator.comparing(PlanItemRecord::createdAt))
    .map(PlanItemRecord::bindingName)
    .collect(Collectors.joining(" → "));

if (path.isBlank()) {
    path = "(direct-verdict)";
}
```

**Empty path guard:** `FeatureVectorCbrCase` rejects blank solutions (`isBlank()` check in constructor). If no plan items are COMPLETED or FAULTED — e.g., a direct manual verdict recorded via `recordOutcome()` without worker execution — the sentinel `"(direct-verdict)"` is used. The case is still stored: it records that an investigation with these initial dimensions was resolved directly, which is a valid CBR data point for the Retrieve step.

**Why not `isTerminal()`:** `TaskStatus.isTerminal()` returns `true` for COMPLETED, FAULTED, REJECTED, OBSOLETE, and CANCELLED. OBSOLETE (superseded by another plan item — e.g., race pattern losers) and CANCELLED (case cancelled before execution) represent work that was never attempted. Including them would pollute the investigation path with phantom steps, degrading CBR similarity quality — two investigations with identical actual steps would appear different if one had more cancelled/obsolete items. FAULTED is included because it represents work that was attempted (useful signal: "this investigation tried X but it failed, and still reached this outcome").

`PlanItemRecord` provides `bindingName()` (the case definition binding name, e.g. `"entity-resolution"`), `executorName()` (the actual worker that executed), `status()`, and `createdAt()` (creation timestamp for ordering).

### 6. Enrichment Data Sources

The `CaseProfileExtractor.extractComplete()` method requires specialist worker outputs (`EntityType`, `JurisdictionRisk`, `NetworkComplexity`) that are stored in the case context during investigation execution. The observer reads these from the `CaseInstance` context map via `CaseInstanceCache`:

**Context key contracts:**

| Key | Type | Set by | Used for |
|-----|------|--------|----------|
| `"transaction"` | `Map<String, Object>` (serialized `SuspiciousTransaction`) | `AmlEngineCoordinator` | `flagReason`, `amount` for `CaseProfileExtractor` |
| `"priorEntityContext"` | `Map<String, Object>` (serialized `AmlPriorContext.toContextMap()`) | `AmlEngineCoordinator` | `entityRiskCount` → `priorIncidentCount` |
| `"entityType"` | `String` (`EntityType` enum name) | Entity-resolution worker | `EntityType` dimension |
| `"jurisdictionRisk"` | `String` (`JurisdictionRisk` enum name) | OSINT-screening worker | `JurisdictionRisk` dimension |
| `"networkComplexity"` | `String` (`NetworkComplexity` enum name) | Pattern-analysis worker | `NetworkComplexity` dimension |
| `"sarNarrative"` | `String` (free-text) | SAR-drafting worker | Semantic text feature for CBR |

**`SuspiciousTransaction` reconstruction:** The original transaction is in the case context as a serialized map under `"transaction"` (set by `AmlEngineCoordinator.startInvestigation()`). The observer deserializes it via `ObjectMapper.convertValue(caseContext.get("transaction"), SuspiciousTransaction.class)`, providing real values for `amount`, `currency`, `timestamp`, and `flagReason` — unlike the dummy-value pattern in `AmlSarOutcomeMemoryObserver` which is acceptable for entity-risk memory but not for CBR feature extraction where `transaction_amount` carries 15% weight.

**`AmlPriorContext` data:** The full `AmlPriorContext` object is not recoverable from the case context (it's serialized as `toContextMap()`). The observer only needs `entityRisk().size()` for `priorIncidentCount`, which is available as `entityRiskCount` in the serialized map: `caseContext.getPath("priorEntityContext.entityRiskCount")`.

**Partial profiles:** If any enrichment field is unavailable (worker was skipped or failed), the observer stores a partial profile using `CaseProfile.initial()` rather than failing entirely. Partial profiles are still valuable — they record that a case with these initial dimensions resulted in this outcome.

## Testing

### Unit Tests

| Test | Scope |
|------|-------|
| `AmlCaseProfileLedgerEntryTest` | `domainContentBytes()` — all fields including new `transactionAmount`, `priorIncidentCount`, `networkComplexity`; nullable fields |
| `AmlCbrSchema` update test | Schema still valid after adding `sar_narrative` field |

### `@QuarkusTest`

| Test | Scope |
|------|-------|
| `AmlCaseProfileStoreObserverTest` | Full lifecycle: fire `SarOutcomeRecordedEvent` → verify `CbrCaseMemoryStore.store()` called with correct features, solution, outcome → verify `LedgerEntry` written with correct fields and `causedByEntryId` chain |
| Error isolation | CBR store failure → ledger entry still written; ledger failure → CBR store still written |
| Tenancy null-guard | Event with null `tenancyId` → falls back to `DEFAULT_TENANT_ID` |
| Partial profile | Missing enrichment field (e.g., entity-resolution worker skipped) → `CaseProfile.initial()` used, CBR case still stored |

Test conventions per CLAUDE.md: hash chain disabled, drain engine to completion before assertions, ledger subject isolation via `caseId` (raw case UUID), CBR store entity isolation via `UUID.nameUUIDFromBytes(("aml-cbr:" + caseId).getBytes(UTF_8))`.

## Scope Boundary

**In scope:** CBR Retain — store completed investigations as retrievable cases.

**Out of scope (later issues):**
- #95 — CBR Retrieve (similarity search against the case base)
- #96 — CBR Reuse (investigation path adaptation from similar cases)
- #97 — CBR Retain outcome update (`CbrCaseMemoryStore.recordOutcome()` after post-submission SAR re-evaluation)
- #98 — SAR narrative seeding from similar past cases
- #99 — Cold-start case base seeding
