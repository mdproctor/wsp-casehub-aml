# Layer 3 Test Coverage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the test coverage gap in `NaiveSarDraftingServiceTest` (entity/pattern Declined and Failed variants) and strengthen the end-to-end assertion in `NaiveAmlInvestigationServiceTest`, then delete three stale branches.

**Architecture:** Pure test additions — no production code changes. `NaiveSarDraftingService` already handles all nine outcome combinations via exhaustive switch expressions; the tests verify that existing behaviour is correct and pinned. The `NaiveAmlInvestigationServiceTest` gains one content assertion proving the transaction ID flows through to the SAR narrative.

**Tech Stack:** Java 21, JUnit 5, plain `new` instantiation (no Quarkus, no CDI)

**Refs:** casehubio/aml#24, spec: `specs/issue-24-layer3-code-quality/2026-05-22-layer3-test-coverage-design.md`

---

## File Map

| File | Change |
|------|--------|
| `app/src/test/java/io/casehub/aml/tutorial/NaiveSarDraftingServiceTest.java` | Add 4 fixtures + 4 test methods; extract inline `completedOsint` to field |
| `app/src/test/java/io/casehub/aml/tutorial/NaiveAmlInvestigationServiceTest.java` | Strengthen one `assertNotNull` to `assertTrue(...contains(...))` |

---

## Task 1: Delete stale branches

No source changes — pure git operations. Remote branches pending deletion per handover.

**Project repo:** `/Users/mdproctor/claude/casehub/aml`
**Workspace:** `/Users/mdproctor/claude/public/casehub/aml`

- [ ] **Step 1: Delete remote branches**

```bash
git -C /Users/mdproctor/claude/casehub/aml push origin --delete epic-layer3-qhorus issue-13 issue-26
```

Expected: three `deleted` lines, one per branch. If any branch is already gone the command will print a warning but not fail overall — that is fine.

- [ ] **Step 2: Delete local branches (if they exist)**

```bash
git -C /Users/mdproctor/claude/casehub/aml branch -d epic-layer3-qhorus issue-13 issue-26 2>/dev/null || true
```

Expected: each deleted or "not found" — both are fine.

- [ ] **Step 3: Verify**

```bash
git -C /Users/mdproctor/claude/casehub/aml branch -a | grep -E "epic-layer3-qhorus|issue-13|issue-26"
```

Expected: no output.

---

## Task 2: Strengthen `NaiveAmlInvestigationServiceTest` content assertion

**File:** `app/src/test/java/io/casehub/aml/tutorial/NaiveAmlInvestigationServiceTest.java`

The existing `investigate_validTransaction_returnsCompleteSummary` test calls `assertNotNull(summary.sarNarrative())` — this verifies presence but not that the transaction flows through to the narrative. `NaiveSarDraftingService.draft()` always opens with `"SAR narrative for " + transaction.id()`, so we can assert the transaction ID appears.

- [ ] **Step 1: Add `assertTrue` import and strengthen the assertion**

In `investigate_validTransaction_returnsCompleteSummary`, replace:
```java
assertNotNull(summary.sarNarrative());
```
with:
```java
assertTrue(summary.sarNarrative().contains("TXN-001"),
        "Narrative should reference transaction ID: " + summary.sarNarrative());
```

The import `import static org.junit.jupiter.api.Assertions.assertTrue;` is already present in the file.

The full method after the change:
```java
@Test
void investigate_validTransaction_returnsCompleteSummary() {
    InvestigationSummary summary = service.investigate(tx("TXN-001"));

    assertNotNull(summary);
    assertNotNull(summary.entityResolution());
    assertNotNull(summary.patternAnalysis());
    assertNotNull(summary.osintScreening());
    assertTrue(summary.sarNarrative().contains("TXN-001"),
            "Narrative should reference transaction ID: " + summary.sarNarrative());
}
```

- [ ] **Step 2: Run the test to confirm it passes**

```bash
mvn test -pl app -am -Dtest=NaiveAmlInvestigationServiceTest -Dsurefire.failIfNoSpecifiedTests=false -q
```

Expected: `BUILD SUCCESS`. The assertion passes because `NaiveSarDraftingService.draft()` opens with `"SAR narrative for TXN-001. ..."`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/test/java/io/casehub/aml/tutorial/NaiveAmlInvestigationServiceTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "test(issue-24): assert SAR narrative contains transaction ID

Refs #24"
```

---

## Task 3: Fix fixture asymmetry and add entity/pattern Declined/Failed tests

**File:** `app/src/test/java/io/casehub/aml/tutorial/NaiveSarDraftingServiceTest.java`

Current state: `completedEntity` and `completedPattern` are class-level fixtures. `osint` is constructed inline in every test. The new tests need a symmetric pattern: all three specialists have class-level fixtures for all used variants.

- [ ] **Step 1: Extract `completedOsint` to a field and add Declined/Failed fixtures**

Add these four fields after the existing `completedPattern` field:

```java
private final SpecialistOutcome<OsintResult> completedOsint =
        new SpecialistOutcome.Completed<>(new OsintResult(false, false, "clean"));

private final SpecialistOutcome<EntityResolutionResult> declinedEntity =
        new SpecialistOutcome.Declined<>("entity-agent", "entity-resolution", "insufficient clearance");
private final SpecialistOutcome<EntityResolutionResult> failedEntity =
        new SpecialistOutcome.Failed<>("entity-agent", "entity-resolution", "timeout");
private final SpecialistOutcome<PatternAnalysisResult> declinedPattern =
        new SpecialistOutcome.Declined<>("pattern-agent", "pattern-analysis", "insufficient data");
private final SpecialistOutcome<PatternAnalysisResult> failedPattern =
        new SpecialistOutcome.Failed<>("pattern-agent", "pattern-analysis", "connection timeout");
```

- [ ] **Step 2: Update existing tests to use the new `completedOsint` field**

In `draft_withCompletedOsint_includesTransactionId`, replace:
```java
SpecialistOutcome<OsintResult> osint = new SpecialistOutcome.Completed<>(new OsintResult(false, false, "clean"));
```
with:
```java
SpecialistOutcome<OsintResult> osint = completedOsint;
```

In `draft_withDeclinedOsint_includesDeclineInNarrative`, the osint value is already a `Declined` — leave it as-is (it uses a distinct reason string specific to that test).

In `draft_withFailedOsint_includesFailureInNarrative`, same — leave as-is.

- [ ] **Step 3: Add the four new test methods**

Add these after `draft_withFailedOsint_includesFailureInNarrative`:

```java
@Test
void draft_withDeclinedEntity_includesDeclineInNarrative() {
    String narrative = service.draft(tx, declinedEntity, completedPattern, completedOsint);
    assertNotNull(narrative);
    assertTrue(narrative.contains("declined") || narrative.contains("clearance"),
            "Narrative should reference the entity decline: " + narrative);
}

@Test
void draft_withFailedEntity_includesFailureInNarrative() {
    String narrative = service.draft(tx, failedEntity, completedPattern, completedOsint);
    assertNotNull(narrative);
    assertTrue(narrative.contains("failed") || narrative.contains("timeout"),
            "Narrative should reference the entity failure: " + narrative);
}

@Test
void draft_withDeclinedPattern_includesDeclineInNarrative() {
    String narrative = service.draft(tx, completedEntity, declinedPattern, completedOsint);
    assertNotNull(narrative);
    assertTrue(narrative.contains("declined") || narrative.contains("data"),
            "Narrative should reference the pattern decline: " + narrative);
}

@Test
void draft_withFailedPattern_includesFailureInNarrative() {
    String narrative = service.draft(tx, completedEntity, failedPattern, completedOsint);
    assertNotNull(narrative);
    assertTrue(narrative.contains("failed") || narrative.contains("timeout"),
            "Narrative should reference the pattern failure: " + narrative);
}
```

- [ ] **Step 4: Run all seven tests in the class**

```bash
mvn test -pl app -am -Dtest=NaiveSarDraftingServiceTest -Dsurefire.failIfNoSpecifiedTests=false -q
```

Expected: `BUILD SUCCESS`, 7 tests run, 0 failures.

- [ ] **Step 5: Run the full app test suite to confirm no regressions**

```bash
mvn test -pl app -am -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/test/java/io/casehub/aml/tutorial/NaiveSarDraftingServiceTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "test(issue-24): cover entity and pattern Declined/Failed in NaiveSarDraftingServiceTest

All three SpecialistOutcome variants now tested for all three specialists.
Extract completedOsint to field for fixture symmetry.

Closes #24"
```
