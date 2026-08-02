# SAR Narrative Seeding — Design Spec

**Issue:** #98 (SAR narrative seeding from similar past cases)
**Epic:** #92 (Case-Based Reasoning)
**Date:** 2026-07-23

## Context

CBR Retain (#97) stores `PlanCbrCase` entries on case completion, including
`sar_narrative` as a `FeatureField.semanticText()` feature in `AmlCbrSchema`.
CBR Retrieve (#95) injects similar past cases into `CaseContext` as
`cbrExperiences` at case startup. CBR Reuse (#96) analyses experiences via the
`cbr-path-advisor` worker to produce path recommendations.

The gap: past SAR narratives are stored and retrieved but never surfaced to the
sar-drafting worker. The worker generates narratives from scratch every time,
ignoring templates from similar past cases.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Extraction point | cbr-path-advisor (Approach B) | Sanitisation insertion point upstream of worker input — when real sanitisation replaces the pass-through, LLM-powered workers won't see raw PII from past cases |
| Not worker-side extraction (Approach A) | Rejected | Raw `cbrExperiences` in sar-drafting input projection would put unsanitized PII in the LLM context window when workers become LLM-powered |
| Not new binding (Approach C) | Rejected | Over-engineered — another async worker dispatch for what is a 10-line extension of an existing loop |
| Outcome filter | `SAR_WARRANTED` only | `FALSE_POSITIVE`/`INCONCLUSIVE` cases never produce a `sarNarrative` (triage gate exits before sar-drafting fires); filter is defensive |
| Worker adaptation | Plumbing only | Workers are stubs — real narrative adaptation requires LLM integration (separate follow-up) |
| Quality metric | `narrativeSeeded` + `seedCount` on context and ledger | Enables UPHELD-rate segmentation; no A/B flag needed — unseeded cases occur naturally with empty/mismatched case base |

## §1 — Domain Types

### `SeedNarrative` record

New record in `api/src/main/java/io/casehub/aml/domain/`:

```java
public record SeedNarrative(
    String narrative,
    double similarityScore,
    String flagReason,
    String entityType
) {}
```

- `narrative` — passed through `DecisionContextSanitiser.sanitise()` before construction (currently pass-through; real PII redaction is a follow-up)
- `similarityScore` — source case similarity, for ranking. CBR retrieval pre-filters at `CbrConfig.minSimilarity`; the seeder defensively excludes `≤ 0`
- `flagReason` — source case flag reason (categorical label, not PII)
- `entityType` — source case entity type (categorical label, not PII)

## §2 — `SarNarrativeSeeder` Service

Plain class in `app/src/main/java/io/casehub/aml/cbr/`. Not a CDI bean —
`DecisionContextSanitiser` has no CDI producer in the project, so `@Inject`
would fail at startup. Constructed directly in `AmlInvestigationCaseHub.augment()`
with `PassThroughDecisionContextSanitiser`.

```java
public class SarNarrativeSeeder {

    private final DecisionContextSanitiser sanitiser;

    public SarNarrativeSeeder(DecisionContextSanitiser sanitiser) {
        this.sanitiser = sanitiser;
    }

    public List<SeedNarrative> extract(List<Map<String, Object>> experiences) {
        if (experiences == null || experiences.isEmpty()) {
            return List.of();
        }
        return experiences.stream()
            .filter(e -> "SAR_WARRANTED".equals(e.get("outcome")))
            .filter(e -> extractFeatureString(e, "sar_narrative") != null)
            .filter(e -> ((Number) e.get("similarityScore")).doubleValue() > 0)
            .map(e -> {
                String raw = extractFeatureString(e, "sar_narrative");
                double score = ((Number) e.get("similarityScore")).doubleValue();
                String flagReason = extractFeatureString(e, "flag_reason");
                String entityType = extractFeatureString(e, "entity_type");
                return new SeedNarrative(sanitiser.sanitise(raw), score, flagReason, entityType);
            })
            .sorted(Comparator.comparingDouble(SeedNarrative::similarityScore).reversed())
            .toList();
    }
}
```

### `FeatureValue` deserialization

All features in `RetrievedExperience.features()` are stored as `FeatureValue`
objects (via `CaseProfile.toFeatures()` and `FeatureValue.string()`). Jackson
serialization may produce either shape:

1. Plain `String` — if Jackson resolves `FeatureValue` to its inner value
2. Wrapped object `{"type":"STRING","value":"..."}` — if Jackson serializes the `FeatureValue` wrapper

A shared `extractFeatureString()` method handles both shapes for all string
features (`sar_narrative`, `flag_reason`, `entity_type`):

```java
@SuppressWarnings("unchecked")
private static String extractFeatureString(Map<String, Object> experience, String featureName) {
    var features = (Map<String, Object>) experience.get("features");
    if (features == null) return null;
    Object val = features.get(featureName);
    if (val instanceof String s) return s;
    if (val instanceof Map<?,?> m) return m.get("value") instanceof String s ? s : null;
    return null;
}
```

## §3 — Advisor Integration

### Construction chain

`SarNarrativeSeeder` is constructed directly in `AmlInvestigationCaseHub.augment()`
and passed through the descriptor to the advisor worker factory:

1. `AmlInvestigationCaseHub.augment()` — `var seeder = new SarNarrativeSeeder(new PassThroughDecisionContextSanitiser())`
2. `new AmlInvestigationCaseDescriptor(..., seeder)` — new constructor parameter
3. `CbrPathAdvisorWorker.create(ledgerRepository, principal, seeder)` — new factory parameter
4. Seeder captured in the `FlowWorkerFunction` lambda closure within `doAdvise()`

### Advisor output change

After the per-experience statistics loop in `CbrPathAdvisorWorker.doAdvise()`,
a single call extracts seed narratives:

```java
List<SeedNarrative> narratives = seeder.extract(experiences);
```

The output map includes the narratives:

```json
{
  "caseCount": 8,
  "capabilities": { ... },
  "predominantOutcome": "SAR_WARRANTED",
  "predominantOutcomeFrequency": 0.875,
  "confidence": 0.82,
  "similarSarNarratives": [
    {
      "narrative": "SAR filed for transaction TX-789...",
      "similarityScore": 0.88,
      "flagReason": "STRUCTURING",
      "entityType": "SHELL_COMPANY"
    }
  ]
}
```

### Existing output projection unchanged

`outputProjection: "{ cbrPathAdvice: . }"` writes the entire output to
`cbrPathAdvice`. The narratives nest under `cbrPathAdvice.similarSarNarratives`.

### Failure handling

The seeder call has its own try/catch inside `doAdvise()`, independent of the
existing top-level catch-all in `create()`. This preserves valid statistics
(capabilities, confidence, predominantOutcome) when only narrative extraction
fails:

```java
List<SeedNarrative> narratives;
try {
    narratives = seeder.extract(experiences);
} catch (Exception e) {
    LOG.warnf(e, "Narrative seeding failed — proceeding without seeds");
    narratives = List.of();
}
result.put("similarSarNarratives", narratives);
```

The existing top-level catch-all remains unchanged — it handles failures in the
statistics loop itself (e.g., malformed experience maps). The advisor's existing
fallback gains `"similarSarNarratives": []` for that path.

## §4 — Sar-Drafting Capability Change

### Projection changes

The `sar-drafting` capability definition gains `similarSarNarratives` on input
and `narrativeSeeded`/`seedCount` on output:

```yaml
- name: sar-drafting
  description: "Synthesise specialist findings into SAR narrative"
  inputProjection: >-
    { transaction: .transaction,
      entityResolution: .entityResolution,
      patternAnalysis: .patternAnalysis,
      osintScreening: .osintScreening,
      similarSarNarratives: .cbrPathAdvice.similarSarNarratives }
  outputProjection: "{ sarNarrative: .sarNarrative, narrativeSeeded: .narrativeSeeded, seedCount: .seedCount }"
```

### Null safety

- `cbrPathAdvice` null (no CBR experiences, advisor never fired) →
  `.cbrPathAdvice.similarSarNarratives` evaluates to `null` in JQ →
  worker receives `similarSarNarratives: null` → treats as no seeds
- Advisor fired, no SAR_WARRANTED narratives → `similarSarNarratives: []` →
  worker treats empty list as no seeds

### Worker stub changes

Both `sarDraftingWorkerJunior` and `sarDraftingWorkerSenior` gain:

```java
@SuppressWarnings("unchecked")
List<Map<String, Object>> seeds =
    (List<Map<String, Object>>) input.get("similarSarNarratives");
boolean seeded = seeds != null && !seeds.isEmpty();

// ... existing stub narrative generation unchanged ...

Map<String, Object> result = new LinkedHashMap<>();
result.put("sarNarrative", sarNarrative);
result.put("narrativeSeeded", seeded);
result.put("seedCount", seeded ? seeds.size() : 0);
return WorkerResult.of(result, plannedAction);
```

The `narrativeSeeded` and `seedCount` fields flow to the case context —
observable in tests and available for the quality metric.

## §5 — Quality Metric

### Ledger-level signal (not CBR features)

`narrativeSeeded` and `seedCount` are quality/observability metrics, not case
similarity features. They are NOT added to `PlanCbrCase` features — whether a
past case was itself seeded has no bearing on similarity to a current case, and
`AmlCbrSchema` has no weights for them. They are populated on the ledger entry
only.

### Ledger columns

Two new columns on `AmlCaseProfileLedgerEntry`:

- `narrative_seeded` — `Boolean`, nullable (null for pre-existing entries)
- `seed_count` — `Integer`, nullable

Populated from the case file snapshot at retain time:

```java
if (snapshot.get("narrativeSeeded") instanceof Boolean b) {
    entry.narrativeSeeded = b;
}
if (snapshot.get("seedCount") instanceof Number n) {
    entry.seedCount = n.intValue();
}
```

Enables a simple analytical query: segment UPHELD/WITHDRAWN/FLAGGED rates by
`narrative_seeded = true` vs `false`.

### No A/B flag needed

Unseeded cases occur naturally: empty case base, no SAR_WARRANTED experiences
among retrieved cases, or cases where advisor didn't fire. As the case base
grows, the proportion of seeded cases increases organically. The quality
comparison is retrospective, not experimental.

### Flyway migration

```sql
-- V3008__narrative_seeding_columns.sql
ALTER TABLE aml_case_profile_ledger_entry ADD COLUMN narrative_seeded BOOLEAN;
ALTER TABLE aml_case_profile_ledger_entry ADD COLUMN seed_count INTEGER;
```

### `domainContentBytes()` update

Extends the existing `String.join` pattern:

```java
@Override
public byte[] domainContentBytes() {
    return String.join("|",
                       flagReason != null ? flagReason : "",
                       transactionAmount != null ? transactionAmount.toPlainString() : "",
                       String.valueOf(priorIncidentCount),
                       entityType != null ? entityType : "",
                       jurisdictionRisk != null ? jurisdictionRisk : "",
                       networkComplexity != null ? networkComplexity : "",
                       outcome != null ? outcome : "",
                       confidence != null ? String.valueOf(confidence) : "",
                       investigationPath != null ? investigationPath : "",
                       narrativeSeeded != null ? String.valueOf(narrativeSeeded) : "",
                       seedCount != null ? String.valueOf(seedCount) : ""
                      ).getBytes(StandardCharsets.UTF_8);
}
```

## §6 — Testing

### Unit tests (api module)

- `SeedNarrativeTest`: record construction, field access

### Unit tests (app module)

- `SarNarrativeSeederTest`:
  - Filters to `SAR_WARRANTED` only — `FALSE_POSITIVE`/`INCONCLUSIVE` excluded
  - Extracts narrative from plain `String` feature shape
  - Extracts narrative from wrapped `FeatureValue` map shape
  - Skips experiences with null/missing `sar_narrative`
  - Calls `DecisionContextSanitiser.sanitise()` on each extracted narrative (passed via constructor)
  - Extracts `flag_reason` and `entity_type` from both plain String and wrapped FeatureValue shapes
  - Excludes experiences with `similarityScore ≤ 0` (defensive filter)
  - Returns results sorted by `similarityScore` descending
  - Empty input → empty output
  - Null input → empty output
  - All experiences lack narratives → empty output

### @QuarkusTest integration (app module)

**Seeding surfaces to worker:** Pre-populate `CbrCaseMemoryStore` with
SAR_WARRANTED cases that have `sar_narrative` features → start investigation →
drain to completion → assert `narrativeSeeded: true` and `seedCount > 0` in
case context.

**No seeds when case base empty:** Empty store → start investigation → drain →
assert `narrativeSeeded: false` and `seedCount: 0`.

**No seeds when all experiences are FALSE_POSITIVE:** Pre-populate with
FALSE_POSITIVE cases only → start investigation → drain → assert
`narrativeSeeded: false`.

**Advisor failure doesn't block sar-drafting:** Inject malformed experiences →
advisor writes fallback with `similarSarNarratives: []` → sar-drafting fires
with null/empty seeds → investigation completes normally.

**Ledger entry records seeding metadata:** Run seeded investigation → assert
`AmlCaseProfileLedgerEntry` has `narrativeSeeded: true` and `seedCount`
populated.

**Cold start backward compatibility:** No CBR config, no experiences → all
bindings work exactly as before → `narrativeSeeded: false`.

### Test conventions

- Drain to `status=completed` before assertions
- `casehub.ledger.hash-chain.enabled=false`
- Ledger subject isolation: `UUID.nameUUIDFromBytes("aml-<concern>:" + caseId)`
- `CbrCaseRetainObserver` excluded from CDI
- Gate approval ordering for PlannedAction workers
- `CbrQuery.withNotBefore(Instant.now())` for test isolation

## §7 — Scope Boundaries

### Not in scope

- **LLM-powered narrative adaptation** — stubs acknowledge seeds, don't adapt.
  Separate follow-up issue.
- **`DecisionContextSanitiser` generalization** — existing
  `PassThroughDecisionContextSanitiser` remains. The `String → String` contract
  is correct for both JSON and prose; the interface name and future implementation
  need to support both modes (structured field masking + NER-based prose redaction).
  This is a cross-cutting ledger concern.
- **Changes to CBR retrieval or storage** — narratives are already stored and
  retrieved. No engine changes needed.
- **Advisor capability rename** — description stays as-is; cosmetic.

### Follow-up issues

1. **#114** — LLM-powered sar-drafting worker — uses seed narratives as actual templates, context-limit management for seed count/length
2. **#115** — Generalize `DecisionContextSanitiser` → `ContentSanitiser` — support both structured JSON field masking and prose entity redaction (NER-based). Current pass-through is the insertion point; real implementation is cross-cutting (ledger module)
3. **#116** — Quality dashboard — visualize UPHELD rates segmented by `narrative_seeded`

### Cross-repo impact

None. All types used (`DecisionContextSanitiser`, `RetrievedExperience`
serialization shape, `CbrCaseMemoryStore`) are existing API surface.

## §8 — Files Touched

| File | Change |
|------|--------|
| `api/.../domain/SeedNarrative.java` | **New** — record |
| `app/.../cbr/SarNarrativeSeeder.java` | **New** — plain class, constructor takes `DecisionContextSanitiser` |
| `app/.../engine/AmlInvestigationCaseHub.java` | **Modify** — construct `SarNarrativeSeeder` in `augment()`, pass to descriptor constructor |
| `app/.../engine/AmlInvestigationCaseDescriptor.java` | **Modify** — new constructor parameter for seeder, pass to `CbrPathAdvisorWorker.create()`; sar-drafting workers read seeds, write `narrativeSeeded`/`seedCount` |
| `app/.../cbr/CbrPathAdvisorWorker.java` | **Modify** — `create()` gains `SarNarrativeSeeder` parameter; `doAdvise()` calls `seeder.extract()` after statistics loop |
| `app/.../resources/aml/aml-investigation.yaml` | **Modify** — sar-drafting inputProjection adds `similarSarNarratives` |
| `app/.../cbr/AmlCaseProfileStoreObserver.java` | **Modify** — extract `narrativeSeeded`/`seedCount`, populate ledger entry |
| `app/.../ledger/AmlCaseProfileLedgerEntry.java` | **Modify** — add `narrativeSeeded`, `seedCount` columns; update `domainContentBytes()` |
| `V3008__narrative_seeding_columns.sql` | **New** — migration |
| `api/.../domain/SeedNarrativeTest.java` | **New** |
| `app/.../cbr/SarNarrativeSeederTest.java` | **New** |
| `app/.../cbr/AmlCaseProfileStoreObserverTest.java` | **Modify** |
| `app/.../ledger/AmlCaseProfileLedgerEntryTest.java` | **Modify** |
| Integration test(s) | **New** — per §6 |

## Garden Entries Referenced

- GE-20260720-6ea915 — CbrCaseRetainObserver CDI exclusion
- GE-20260716-986cd1 — InMemoryCbrCaseMemoryStore test isolation
