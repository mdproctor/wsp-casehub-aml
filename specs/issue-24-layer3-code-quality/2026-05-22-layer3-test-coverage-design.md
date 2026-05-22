# Design: Layer 3 Test Coverage — Issue #24

**Date:** 2026-05-22  
**Issue:** casehubio/aml#24  
**Branch:** issue-24-layer3-code-quality  
**Scope:** XS — test additions only, no production code changes

---

## Problem

`NaiveSarDraftingServiceTest` covers all three `SpecialistOutcome` variants (Completed, Declined, Failed) for `osint` but only Completed fixtures for `entity` and `pattern`. The switch branches for entity/pattern Declined and Failed have no test coverage despite being implemented. `NaiveAmlInvestigationServiceTest` asserts structure but not that the SAR narrative carries the transaction ID through the full Layer 1 pipeline.

---

## Changes

### `NaiveSarDraftingServiceTest` — 4 new tests + 4 new fixtures

Add Declined and Failed fixtures for entity and pattern at field level:

```java
private final SpecialistOutcome<EntityResolutionResult> declinedEntity =
    new SpecialistOutcome.Declined<>("entity-agent", "entity-resolution", "insufficient clearance");
private final SpecialistOutcome<EntityResolutionResult> failedEntity =
    new SpecialistOutcome.Failed<>("entity-agent", "entity-resolution", "timeout");
private final SpecialistOutcome<PatternAnalysisResult> declinedPattern =
    new SpecialistOutcome.Declined<>("pattern-agent", "pattern-analysis", "insufficient data");
private final SpecialistOutcome<PatternAnalysisResult> failedPattern =
    new SpecialistOutcome.Failed<>("pattern-agent", "pattern-analysis", "connection timeout");
```

Four new `@Test` methods, each using `completedOsint` as the third arg. Assertion style matches existing osint tests: keyword presence, not exact string match.

| Test | Fixture | Assertion |
|------|---------|-----------|
| `draft_withDeclinedEntity_includesDeclineInNarrative` | `declinedEntity` | `contains("declined") \|\| contains("clearance")` |
| `draft_withFailedEntity_includesFailureInNarrative` | `failedEntity` | `contains("failed") \|\| contains("timeout")` |
| `draft_withDeclinedPattern_includesDeclineInNarrative` | `declinedPattern` | `contains("declined") \|\| contains("data")` |
| `draft_withFailedPattern_includesFailureInNarrative` | `failedPattern` | `contains("failed") \|\| contains("timeout")` |

### `NaiveAmlInvestigationServiceTest` — 1 strengthened assertion

In `investigate_validTransaction_returnsCompleteSummary`, replace:
```java
assertNotNull(summary.sarNarrative());
```
with:
```java
assertTrue(summary.sarNarrative().contains("TXN-001"),
    "Narrative should reference transaction ID");
```

### Branch deletions

Delete `epic-layer3-qhorus`, `issue-13`, `issue-26` from local and remote. No commits required.

---

## What this does NOT change

- Production code — no changes to any `main/` source
- `NaiveAmlInvestigationServiceTest` assertions for entity/pattern/osint structure — these are correct; the naive service always produces Completed outcomes by design
- Issue #22 (fanOut) — deferred, qhorus/claudony refactoring in progress
- Issue #23 (qhorus message persistence) — separate issue

---

## Testing

All new tests are pure JUnit 5 unit tests — no Quarkus, no CDI, instantiated with `new`. Run with:

```bash
mvn test -pl app -am -Dtest=NaiveSarDraftingServiceTest,NaiveAmlInvestigationServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```
