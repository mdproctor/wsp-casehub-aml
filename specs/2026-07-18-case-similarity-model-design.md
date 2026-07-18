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

### Divergences from guide-aml.md

The neocortex guide (`neocortex/docs/cbr/guide-aml.md`) provides a reference schema. This spec diverges deliberately in several areas:

| Aspect | guide-aml.md | This spec | Rationale |
|--------|-------------|-----------|-----------|
| Flag pattern | `transaction_pattern` | `flag_reason` | Aligns with `SuspiciousTransaction.flagReason` field name — same concept, AML-domain naming |
| Entity | `entity_risk_tier` LOW/MED/HIGH/PEP | `entity_type` INDIVIDUAL/CORPORATE/SHELL_COMPANY/PEP | Different dimension: ownership classification drives investigation methodology (beneficial ownership chains vs source-of-wealth), not just risk level |
| Jurisdiction | ISO 3166-1 alpha-2 | 3-tier risk enum HIGH/MED/LOW | Risk-level similarity — two FATF grey-list countries have similar investigation patterns regardless of specific country. Country-level granularity deferred to #95 filter criteria |
| Amount | `amount_range` (text labels, categorical) | `transaction_amount` (numeric, GaussianDecay) | Smooth similarity decay across the continuous amount range. Guide's buckets suit triggering thresholds, not case-to-case similarity |
| Prior history | `prior_sars_on_entity` numeric 0-100 | `prior_incident_count` numeric 0-20 | Same numeric approach. Range tightened from guide's [0, 100] to [0, 20] — AML entity-risk counts cluster in 0-10; the wider range compresses this into non-discriminating normalized distances. Values above 20 still work (scorer doesn't clamp). Field name reflects AML domain (entity-risk memories, not just SARs) |
| Network | Not present | `network_complexity` (new) | AML-specific dimension: counterparty graph size distinguishes single-entity structuring from network-based laundering |
| Narrative | `investigation_narrative` text/semantic | Omitted (deferred) | Requires embedding infrastructure (`memory-cbr-embedding`). Tracked as deferred scope — see Not In Scope |

## Deliverables

### 1. Domain Enums (api/)

Package: `io.casehub.aml.domain`

| Enum | Values | Purpose |
|------|--------|---------|
| `FlagReason` | `STRUCTURING`, `LAYERING`, `SMURFING`, `ROUND_TRIP`, `PEP_MATCH`, `HIGH_RISK_JURISDICTION`, `VELOCITY_ANOMALY`, `LARGE_VOLUME` | Why the transaction was flagged |
| `EntityType` | `INDIVIDUAL`, `CORPORATE`, `SHELL_COMPANY`, `PEP` | Beneficial ownership classification |
| `JurisdictionRisk` | `HIGH`, `MEDIUM`, `LOW` | FATF grey/black list risk tier |
| `NetworkComplexity` | `SINGLE_ENTITY`, `SMALL_NETWORK`, `LARGE_NETWORK` | Counterparty graph size (1 / 2-5 / >5) |

### 2. SuspiciousTransaction Change (api/)

`SuspiciousTransaction.flagReason` changes from `String` to `FlagReason` enum. Pre-release — no backward compatibility concern. All ~27 call sites updated.

### 3. CaseProfile Record (api/)

Package: `io.casehub.aml.domain`

```java
public record CaseProfile(
    FlagReason flagReason,              // non-null — always available at case start
    BigDecimal transactionAmount,       // non-null — raw amount from SuspiciousTransaction
    int priorIncidentCount,             // always available — entityRisk().size() from AmlPriorContext
    EntityType entityType,              // nullable — known after entity-resolution
    JurisdictionRisk jurisdiction,      // nullable — known after jurisdiction lookup
    NetworkComplexity network           // nullable — known after entity-resolution
)
```

Two factory methods:
- `CaseProfile.initial(flagReason, transactionAmount, priorIncidentCount)` — for Retrieve at case start (3 dimensions)
- `CaseProfile.complete(flagReason, transactionAmount, priorIncidentCount, entityType, jurisdiction, network)` — for Retain after investigation (6 dimensions)

`toFeatures() → Map<String, FeatureValue>` bridges to neocortex. Emits categorical fields as `FeatureValue.StringVal`, numeric fields as `FeatureValue.NumberVal`. Skips null fields (nullable dimensions on initial profiles).

**Scorer behavior with partial features:** `CbrSimilarityScorer` iterates over *query* features. For each query feature, it looks up the corresponding case feature. Missing case features score 0.0, but their weight is still included in the total weight denominator. This means:

- **Initial query (3 features) → complete stored case (6 features):** Only 3 query features scored; the 3 extra case features are ignored. Score reflects 3-dimension similarity.
- **Complete query (6 features) → initial stored case (3 features):** All 6 query features scored; the 3 missing case features score 0.0 with full weight. The initial case receives a lower overall score.

This asymmetry is correct: a complete query against a partial case *should* score lower because the unknown dimensions reduce confidence in the match. The penalty is proportional to the missing weight — initial profiles (missing 45% of total weight) score at most ~0.55 even with perfect matches on known dimensions.

### 4. CbrFeatureSchema + Similarity Configuration (app/)

Package: `io.casehub.aml.cbr`

`AmlCbrSchema` defines:
- `CASE_TYPE = "aml-investigation"`
- `SCHEMA` — `CbrFeatureSchema.of()` with 4 categorical + 2 numeric fields
- `WEIGHTS` — per-field weights summing to 1.0

Domain constant: `AmlMemoryDomains.CBR = new MemoryDomain("aml.cbr")` — added to the existing `AmlMemoryDomains` class alongside `ENTITY_RISK`, `NETWORK`, `PATTERN`. Follows the established `aml.{subdomain}` naming convention.

Similarity specs:

**`flag_reason`** (weight 0.30) — `CategoricalTable`:

| Pair | Score | Rationale |
|------|-------|-----------|
| STRUCTURING ↔ SMURFING | 0.7 | Smurfing is a structuring technique |
| STRUCTURING ↔ LAYERING | 0.4 | Both involve transaction manipulation |
| STRUCTURING ↔ ROUND_TRIP | 0.5 | Round-trip is a structuring variant |
| SMURFING ↔ LAYERING | 0.3 | Both multi-step, different mechanisms |
| SMURFING ↔ ROUND_TRIP | 0.3 | Both involve routing/splitting |
| LAYERING ↔ ROUND_TRIP | 0.5 | Both multi-hop transaction chains |
| VELOCITY_ANOMALY ↔ LARGE_VOLUME | 0.6 | Both volume/velocity patterns |
| PEP_MATCH ↔ HIGH_RISK_JURISDICTION | 0.3 | Both political-exposure risk |

Cross-cluster pairs (e.g., STRUCTURING ↔ PEP_MATCH) default to 0.0 — these are fundamentally different investigation types with distinct playbooks, evidence chains, and escalation criteria.

**`entity_type`** (weight 0.20) — `CategoricalTable`:

| Pair | Score | Rationale |
|------|-------|-----------|
| SHELL_COMPANY ↔ CORPORATE | 0.4 | Both corporate structures; shell company investigations extend corporate KYC |
| PEP ↔ INDIVIDUAL | 0.3 | PEP is a special case of individual entity |
| SHELL_COMPANY ↔ PEP | 0.2 | Both high-risk entity types; shared enhanced-due-diligence requirements |
| CORPORATE ↔ PEP | 0.1 | PEPs may own or control corporations; limited investigative overlap |

INDIVIDUAL ↔ CORPORATE = 0.0 is intentional — these are genuinely different investigation types (personal vs business accounts, different KYC procedures, different escalation patterns). The dimension correctly provides zero signal for the most common pair, allowing `flag_reason` and other dimensions to dominate. The value of `entity_type` is its discriminating power when unusual entity types (SHELL_COMPANY, PEP) appear.

SHELL_COMPANY ↔ INDIVIDUAL = 0.0 — very different entities with no investigative overlap.

**`transaction_amount`** (weight 0.15) — `FeatureField.numeric("transaction_amount", 0, 10_000_000, new GaussianDecay(0.15))`:

Raw transaction amount as a numeric field with Gaussian decay. The amount range 0–10M covers the practical AML investigation spectrum. GaussianDecay with σ=0.15 gives tight similarity for nearby amounts and rapid decay across order-of-magnitude differences. Eliminates cliff-edge boundaries inherent in band-based bucketing.

**`jurisdiction_risk`** (weight 0.15) — `CategoricalTable`:

| Pair | Score | Rationale |
|------|-------|-----------|
| HIGH ↔ MEDIUM | 0.5 | Adjacent risk tiers; shared enhanced-monitoring procedures |
| MEDIUM ↔ LOW | 0.5 | Adjacent risk tiers |
| HIGH ↔ LOW | 0.2 | Non-adjacent; significantly different risk assessment approaches |

**`prior_incident_count`** (weight 0.10) — `FeatureField.numeric("prior_incident_count", 0, 20, new GaussianDecay(0.3))`:

Count of entity-risk memories from `AmlPriorContext.entityRisk().size()`. Captures recidivism severity — an entity with 3 prior incidents is more similar to one with 5 than to one with 15. Range [0, 20] concentrates discrimination where AML entity-risk counts cluster: 0 vs 5 incidents scores 0.707 similarity, 0 vs 10 scores 0.249. Values above 20 still produce valid (very low) similarity — `computeNormalizedDistance` does not clamp, so entities with 25 and 30 incidents score d=0.25, sim=0.707. GaussianDecay with σ=0.3 provides smooth similarity gradient.

**`network_complexity`** (weight 0.10) — `CategoricalTable`:

| Pair | Score | Rationale |
|------|-------|-----------|
| SINGLE_ENTITY ↔ SMALL_NETWORK | 0.3 | Both limited scope; small networks share some single-entity investigation steps |
| SMALL_NETWORK ↔ LARGE_NETWORK | 0.5 | Both multi-entity; differ in scale but share network-analysis methodology |
| SINGLE_ENTITY ↔ LARGE_NETWORK | 0.1 | Fundamentally different investigation approaches |

Weights are constants. `PreferenceProvider`-backed configuration is deferred to later CBR issues.

### 5. CaseProfileExtractor (app/)

Package: `io.casehub.aml.cbr`

`@ApplicationScoped` CDI bean with two methods:
- `extractInitial(SuspiciousTransaction, AmlPriorContext) → CaseProfile` — partial profile for Retrieve. Uses `tx.amount()` directly and `priorContext.entityRisk().size()` for incident count.
- `extractComplete(SuspiciousTransaction, AmlPriorContext, EntityType, JurisdictionRisk, NetworkComplexity) → CaseProfile` — full profile for Retain

Thin composition point — logic lives on domain types (`AmlPriorContext.entityRisk().size()`).

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
| `investigation_narrative` semantic text dimension | Deferred issue (requires `memory-cbr-embedding` infrastructure). Enables cross-category retrieval — "shell company in Cyprus" finds similar layering cases even with different transaction patterns. Without this, CBR retrieves only structurally similar cases. |

## Testing

### Unit Tests (api/)

- `FlagReasonTest` — enum coverage, all values round-trip
- `CaseProfileTest`:
  - `initial()` factory — non-null fields set, nullable fields null
  - `complete()` factory — all fields populated
  - `toFeatures()` — partial profile skips nulls, full profile includes all 6 dimensions
  - `toFeatures()` — categorical values match enum `.name()` strings
  - `toFeatures()` — `transaction_amount` emits `FeatureValue.NumberVal`
  - `toFeatures()` — `prior_incident_count` emits `FeatureValue.NumberVal`

### Unit Tests (app/)

- `CaseProfileExtractorTest`:
  - `extractInitial` — correct mapping from transaction + prior context
  - `extractInitial` with no prior history → `priorIncidentCount = 0`
  - `extractInitial` with 5 entity-risk memories → `priorIncidentCount = 5`
  - `extractComplete` — all 6 dimensions populated
- `AmlCbrSchemaTest`:
  - Schema has 6 fields with correct names and types (4 categorical, 2 numeric)
  - Weights sum to 1.0
  - CategoricalTable similarities are symmetric (`CategoricalTable` construction enforces bidirectional storage via `CategoricalTableBuilder.add()`)
  - Numeric fields have correct min/max ranges and GaussianDecay specs
  - `AmlMemoryDomains.CBR` domain constant follows `aml.{subdomain}` convention

### @QuarkusTest (app/)

- `AmlCbrSchemaRegistrarTest` — schema registered on startup, `CbrCaseMemoryStore` accepts it without error

## Dependencies

### Compile (api/)

- `casehub-neocortex-memory-api` — `FeatureValue` (NOT currently on `api/pom.xml` — must be added. The `memory-api` module is Tier 1: pure Java, no Quarkus, no JPA — architecturally appropriate for `api/`)

### Compile (app/)

- `casehub-neocortex-memory-api` — `CbrFeatureSchema`, `FeatureField`, `SimilaritySpec`, `CbrCaseMemoryStore`, `MemoryDomain`
- `casehub-neocortex-memory-cbr-inmem` (test scope) — `InMemoryCbrCaseMemoryStore` for `@QuarkusTest`
