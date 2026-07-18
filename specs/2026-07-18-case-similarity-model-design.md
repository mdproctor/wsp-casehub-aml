# Case Similarity Model — Design Spec

**Issue:** #93 (parent: #92 CBR epic)
**Date:** 2026-07-18
**Status:** Approved

## Context

CaseHub AML introduces Case-Based Reasoning (CBR) so new investigations retrieve and adapt from similar past cases. The platform's neocortex module provides the complete CBR infrastructure — `CbrCaseMemoryStore`, `CbrSimilarityScorer`, `FeatureValue`, `CbrFeatureSchema`, `FeatureVectorCbrCase`, `CbrQuery`. Issue #93 delivers the AML-specific domain model that plugs into this infrastructure: the similarity dimensions, typed enums, case profile record, and feature extraction logic.

## Platform Coherence

No neocortex gaps. The `AmlInvestigationDemo` in neocortex `examples/example-cbr/` already demonstrates the wiring pattern. The original issue's `CaseSimilarityFunction` is entirely replaced by `CbrSimilarityScorer` — no custom scorer needed.

Relevant garden entries:
- GE-20260718-95e11e — `CbrCaseMemoryStore.store()` 6th parameter is `caseType` not `scope` (naming trap)
- GE-20260717-0489d1 — `CbrQuery.of()` gained mandatory `Path scope` parameter (recent SNAPSHOT break)

## Deliverables

### 1. Domain Enums (api/)

Package: `io.casehub.aml.domain`

| Enum | Values | Purpose |
|------|--------|---------|
| `FlagReason` | `STRUCTURING`, `LAYERING`, `SMURFING`, `ROUND_TRIP`, `PEP_MATCH`, `HIGH_RISK_JURISDICTION`, `VELOCITY_ANOMALY`, `LARGE_VOLUME` | Why the transaction was flagged |
| `EntityType` | `INDIVIDUAL`, `CORPORATE`, `SHELL_COMPANY`, `PEP` | Beneficial ownership classification |
| `JurisdictionRisk` | `HIGH`, `MEDIUM`, `LOW` | FATF grey/black list risk tier |
| `AmountBand` | `UNDER_10K`, `BAND_10K_50K`, `BAND_50K_200K`, `BAND_200K_1M`, `OVER_1M` | Transaction amount bracket |
| `NetworkComplexity` | `SINGLE_ENTITY`, `SMALL_NETWORK`, `LARGE_NETWORK` | Counterparty graph size (1 / 2-5 / >5) |

`AmountBand` has a static factory `AmountBand.of(BigDecimal amount)` — threshold logic lives on the enum.

### 2. SuspiciousTransaction Change (api/)

`SuspiciousTransaction.flagReason` changes from `String` to `FlagReason` enum. Pre-release — no backward compatibility concern. All ~27 call sites updated.

### 3. CaseProfile Record (api/)

Package: `io.casehub.aml.domain`

```java
public record CaseProfile(
    FlagReason flagReason,          // non-null — always available at case start
    AmountBand amountBand,          // non-null — derived from transaction amount
    boolean priorHistory,           // always available — from AmlPriorContext
    EntityType entityType,          // nullable — known after entity-resolution
    JurisdictionRisk jurisdiction,  // nullable — known after jurisdiction lookup
    NetworkComplexity network       // nullable — known after entity-resolution
)
```

Two factory methods:
- `CaseProfile.initial(flagReason, amountBand, priorHistory)` — for Retrieve at case start (3 dimensions)
- `CaseProfile.complete(flagReason, amountBand, priorHistory, entityType, jurisdiction, network)` — for Retain after investigation (6 dimensions)

`toFeatures() → Map<String, FeatureValue>` bridges to neocortex. Skips null fields — `CbrSimilarityScorer` scores only present dimensions.

### 4. CbrFeatureSchema + Similarity Configuration (app/)

Package: `io.casehub.aml.cbr`

`AmlCbrSchema` defines:
- `CASE_TYPE = "aml-investigation"`
- `DOMAIN = new MemoryDomain("aml")`
- `SCHEMA` — `CbrFeatureSchema.of()` with 6 categorical fields
- `WEIGHTS` — per-field weights summing to 1.0

Similarity specs:
- `flag_reason` (weight 0.30) — `CategoricalTable`: STRUCTURING ↔ SMURFING = 0.7, STRUCTURING ↔ LAYERING = 0.4, SMURFING ↔ LAYERING = 0.3, PEP_MATCH ↔ HIGH_RISK_JURISDICTION = 0.3
- `entity_type` (weight 0.20) — `CategoricalTable`: SHELL_COMPANY ↔ CORPORATE = 0.4, PEP ↔ INDIVIDUAL = 0.3
- `amount_band` (weight 0.15) — identity match (exact categorical)
- `jurisdiction_risk` (weight 0.15) — identity match
- `prior_history` (weight 0.10) — identity match
- `network_complexity` (weight 0.10) — identity match

Weights are constants. `PreferenceProvider`-backed configuration is deferred to later CBR issues.

### 5. CaseProfileExtractor (app/)

Package: `io.casehub.aml.cbr`

`@ApplicationScoped` CDI bean with two methods:
- `extractInitial(SuspiciousTransaction, AmlPriorContext) → CaseProfile` — partial profile for Retrieve
- `extractComplete(SuspiciousTransaction, AmlPriorContext, EntityType, JurisdictionRisk, NetworkComplexity) → CaseProfile` — full profile for Retain

Thin composition point — logic lives on domain types (`AmountBand.of()`, `AmlPriorContext.hasHistory()`).

### 6. AmlCbrSchemaRegistrar (app/)

Package: `io.casehub.aml.cbr`

`@ApplicationScoped` bean observing `StartupEvent` — calls `cbrCaseMemoryStore.registerSchema(AmlCbrSchema.SCHEMA)`.

## Not In Scope

| Concern | Deferred to |
|---------|-------------|
| Storing cases in `CbrCaseMemoryStore` | #94 |
| Querying similar cases via `CbrQuery` | #95 |
| Adapting investigation paths from retrieved cases | #96 |
| Recording outcomes at case completion | #97 |
| `PreferenceProvider`-backed weight configuration | #94 or #96 |
| Engine `LambdaFeatureExtractor` wiring into `CaseDefinition` | #95 |

## Testing

### Unit Tests (api/)

- `FlagReasonTest` — enum coverage, all values round-trip
- `AmountBandTest` — `AmountBand.of()` threshold boundaries: exact boundaries (10000 → BAND_10K_50K), edge cases (0, negative, null), currency-agnostic (pure BigDecimal)
- `CaseProfileTest`:
  - `initial()` factory — non-null fields set, nullable fields null
  - `complete()` factory — all fields populated
  - `toFeatures()` — partial profile skips nulls, full profile includes all 6 dimensions
  - `toFeatures()` values match enum `.name()` strings

### Unit Tests (app/)

- `CaseProfileExtractorTest`:
  - `extractInitial` — correct mapping from transaction + prior context
  - `extractInitial` with no prior history → `priorHistory = false`
  - `extractInitial` with known high-risk entity → `priorHistory = true`
  - `extractComplete` — all 6 dimensions populated
- `AmlCbrSchemaTest`:
  - Schema has 6 fields with correct names and types
  - Weights sum to 1.0
  - CategoricalTable similarities are symmetric (CbrSimilarityScorer validates this)

### @QuarkusTest (app/)

- `AmlCbrSchemaRegistrarTest` — schema registered on startup, `CbrCaseMemoryStore` accepts it without error

## Dependencies

### Compile (api/)

- `casehub-neocortex-memory-api` — `FeatureValue` (already on classpath via Layer 8 `CaseMemoryStore` integration)

### Compile (app/)

- `casehub-neocortex-memory-api` — `CbrFeatureSchema`, `FeatureField`, `SimilaritySpec`, `CbrCaseMemoryStore`, `MemoryDomain`
- `casehub-neocortex-memory-cbr-inmem` (test scope) — `InMemoryCbrCaseMemoryStore` for `@QuarkusTest`

### Maven

Check whether `casehub-neocortex-memory-api` is already in `api/pom.xml`. If not, add it. The CBR types (`CbrFeatureSchema`, `CbrSimilarityScorer`) are in `memory-api` — pure Java, Tier 1.
