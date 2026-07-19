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
| `SarOutcomeRecordedEvent` | `app/.../engine/` | CDI event fired on SAR outcome recording |
| `AmlSarOutcomeMemoryObserver` | `app/.../memory/` | Existing observer writing entity-level memories on SAR outcome |

### Platform Coherence

- **Boundary rule**: "Do not implement CBR retrieval logic in application repos." Storage (Retain) uses the platform SPI — no retrieval logic here.
- **CBR architecture** (platform/cbr.md §4): Retain = `CaseMemoryStore` + `MemoryEmitter`. This design uses `CbrCaseMemoryStore` (the CBR-specific store), not `CaseMemoryStore` (entity-level). Correct for case-level indexing.
- **Protocol**: `aml-ledger-entry-tenancy-id-non-null` — all LedgerEntry writes must null-guard `tenancyId`.

## Design

### 1. `AmlCaseProfileStoreObserver`

**Location:** `app/src/main/java/io/casehub/aml/cbr/AmlCaseProfileStoreObserver.java`

**Trigger:** `@ObservesAsync SarOutcomeRecordedEvent`

Rationale for this lifecycle point: the SAR verdict (UPHELD/WITHDRAWN/FLAGGED) is the quality signal that makes the case base useful. Cases without a verdict don't contribute learning value.

**Dependencies:**
- `CbrCaseMemoryStore` — store the case
- `LedgerEntryRepository` — write the tamper-evident entry
- `CaseProfileExtractor` — extract the complete profile
- `CurrentPrincipal` — tenancy context (with null-guard per protocol)

**Flow:**
1. Extract `CaseProfile.complete()` from the event's transaction, prior context, and specialist findings
2. Build investigation path string from the engine case execution history (worker names in execution order)
3. Build `FeatureVectorCbrCase`:
   - `problem` = flagged transaction description (transaction type, accounts, amount)
   - `solution` = investigation path (e.g. `"entity-resolution → pattern-analysis → osint-screening → sar-drafting → compliance-review"`)
   - `outcome` = `SarVerdict.name()` (UPHELD / WITHDRAWN / FLAGGED)
   - `confidence` = `SarOutcome.investigationAccuracyScore()`
   - `features` = `CaseProfile.toFeatures()` merged with `{"sar_narrative": FeatureValue.string(sarNarrative)}`
4. Call `cbrCaseMemoryStore.store(cbrCase, AmlCbrSchema.CASE_TYPE, entityId, AmlMemoryDomains.CBR, tenantId, caseId, scope)`
5. Write `AmlCaseProfileLedgerEntry` with `causedByEntryId` linking to the SAR outcome ledger entry
6. Both calls wrapped in independent try/catch — memory failures must not propagate (established AML convention)

**Entity ID for the CBR store:** `UUID.nameUUIDFromBytes("aml-cbr:" + caseId)` — follows the AML ledger subject isolation convention (distinct from engine entries for the same case).

### 2. `AmlCbrSchema` Update

Add `FeatureField.text("sar_narrative")` to the existing schema definition. This enables semantic similarity matching on the SAR narrative during CBR Retrieve (#95).

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
| `entity_type` | `VARCHAR(50)` | NULL | May be null for `initial()` profiles |
| `jurisdiction_risk` | `VARCHAR(50)` | NULL | May be null for `initial()` profiles |
| `outcome` | `VARCHAR(50)` | NOT NULL | `SarVerdict` name |
| `confidence` | `DOUBLE` | NOT NULL | Investigation accuracy score |
| `investigation_path` | `VARCHAR(1000)` | NOT NULL | Worker execution sequence |

**`domainContentBytes()`:** Pipe-delimited UTF-8 of all non-transient fields (per ledger SNAPSHOT `domainContentBytes()` enforcement).

**`subjectId`:** `UUID.nameUUIDFromBytes("aml-cbr:" + caseId)` — same as the CBR store entity ID, maintaining the AML ledger subject isolation convention.

**`causedByEntryId`:** Links to the `AmlSarOfficerReviewedLedgerEntry` that recorded the SAR outcome, closing the evidence chain.

### 4. Flyway V3005

**Location:** `app/src/main/resources/db/aml-engine-ledger/migration/V3005__case_profile_ledger_entry.sql`

```sql
CREATE TABLE aml_case_profile_ledger_entry (
    id         UUID NOT NULL,
    flag_reason       VARCHAR(50)   NOT NULL,
    entity_type       VARCHAR(50),
    jurisdiction_risk VARCHAR(50),
    outcome           VARCHAR(50)   NOT NULL,
    confidence        DOUBLE        NOT NULL,
    investigation_path VARCHAR(1000) NOT NULL,
    PRIMARY KEY (id),
    FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

### 5. Investigation Path Extraction

The observer injects `CaseInstanceCache` (established pattern — used by `AmlLayer6Resource` and `AmlLayer9Resource`) to retrieve the case's completed plan items. The investigation path is built from the completed workers in execution order:

```
CaseInstance instance = caseInstanceCache.get(caseId);
String path = instance.getCompletedPlanItems().stream()
    .sorted(Comparator.comparing(PlanItem::getCompletedAt))
    .map(PlanItem::getWorkerId)
    .collect(Collectors.joining(" → "));
```

### 6. Enrichment Data Sources

The `CaseProfileExtractor.extractComplete()` method requires specialist worker outputs (`EntityType`, `JurisdictionRisk`, `NetworkComplexity`) that are stored in the case context during investigation execution. The observer reads these from the `CaseInstance` context map:

- `EntityType` — set by the entity-resolution worker
- `JurisdictionRisk` — set by the osint-screening worker
- `NetworkComplexity` — set by the pattern-analysis worker

The SAR narrative text comes from the sar-drafting worker's output in the case context.

If any enrichment field is unavailable (worker was skipped or failed), the observer stores a partial profile using `CaseProfile.initial()` rather than failing entirely. Partial profiles are still valuable — they record that a case with these initial dimensions resulted in this outcome.

## Testing

### Unit Tests

| Test | Scope |
|------|-------|
| `AmlCaseProfileLedgerEntryTest` | `domainContentBytes()` — all fields, nullable fields |
| `AmlCbrSchema` update test | Schema still valid after adding `sar_narrative` field |

### `@QuarkusTest`

| Test | Scope |
|------|-------|
| `AmlCaseProfileStoreObserverTest` | Full lifecycle: fire `SarOutcomeRecordedEvent` → verify `CbrCaseMemoryStore.store()` called with correct features, solution, outcome → verify `LedgerEntry` written with correct fields and `causedByEntryId` chain |
| Error isolation | CBR store failure → ledger entry still written; ledger failure → CBR store still written |
| Tenancy null-guard | Event with null `tenancyId` → falls back to `DEFAULT_TENANT_ID` |

Test conventions per CLAUDE.md: hash chain disabled, drain engine to completion before assertions, subject isolation via `UUID.nameUUIDFromBytes("aml-cbr:" + caseId)`.

## Scope Boundary

**In scope:** CBR Retain — store completed investigations as retrievable cases.

**Out of scope (later issues):**
- #95 — CBR Retrieve (similarity search against the case base)
- #96 — CBR Reuse (investigation path adaptation from similar cases)
- #97 — CBR Retain outcome update (`CbrCaseMemoryStore.recordOutcome()` after post-submission SAR re-evaluation)
- #98 — SAR narrative seeding from similar past cases
- #99 — Cold-start case base seeding
