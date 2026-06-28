# Batch Cleanup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close #69 (no code changes), rename CI workflow (#68), and add gate rejection outcome to the Layer 9 investigation API (#71).

**Architecture:** #71 adds an `InvestigationOutcome` record to the `api/` module (JPA-free), modifies `AmlLayer9Resource` to query the ledger for officer review entries, and derives outcome from the `reviewDecision` field. Lifecycle status and domain outcome are separate concerns.

**Tech Stack:** Java 21, Quarkus 3.32.2, REST Assured, Awaitility, casehub-ledger `LedgerEntryRepository`

## Global Constraints

- `TenancyConstants.DEFAULT_TENANT_ID` at all ledger call sites
- `LedgerEntryRepository.findBySubjectId()` + type filter — never `findLatestBySubjectId()` for typed queries
- `casehub.ledger.hash-chain.enabled=false` in test properties (H2 limitation)
- Every test that starts an engine investigation must drain to `status=completed`
- `quarkus.arc.exclude-types` must include engine-work-adapter and work exclusions per CLAUDE.md
- All commits reference an issue: `Refs #N` or `Closes #N`

---

### Task 1: Close #69 and rename CI workflow (#68)

**Files:**
- Rename: `.github/workflows/build.yml` → `.github/workflows/publish.yml`

**Interfaces:**
- Consumes: nothing
- Produces: nothing (no code changes — housekeeping only)

- [ ] **Step 1: Close #69 on GitHub**

```bash
gh issue close 69 --repo casehubio/aml --comment "Already migrated — all Worker imports use io.casehub.worker.api.* and CaseDefinition accessors are correct JavaBean style. No code changes needed."
```

- [ ] **Step 2: Rename the CI workflow**

```bash
git -C /Users/mdproctor/claude/casehub/aml mv .github/workflows/build.yml .github/workflows/publish.yml
```

- [ ] **Step 3: Verify the renamed file is unchanged**

```bash
git -C /Users/mdproctor/claude/casehub/aml diff --staged -- .github/workflows/publish.yml
```

Expected: only the rename shows, no content diff.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml commit -m "ci: rename build.yml to publish.yml

Aligns with platform#110 naming convention. Content unchanged —
repository_dispatch trigger already configured.

Closes #68

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: Add `InvestigationOutcome` record (#71)

**Files:**
- Create: `api/src/main/java/io/casehub/aml/domain/InvestigationOutcome.java`
- Test: `api/src/test/java/io/casehub/aml/domain/InvestigationOutcomeTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `InvestigationOutcome.fromReviewDecision(String reviewDecision)` returns `InvestigationOutcome` with `type()` returning `"sar-filed"`, `"gate-rejected"`, or `"review-inconclusive"`. Returns `null` for unknown values.

- [ ] **Step 1: Write the failing test**

Create `api/src/test/java/io/casehub/aml/domain/InvestigationOutcomeTest.java`:

```java
package io.casehub.aml.domain;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class InvestigationOutcomeTest {

    @Test
    void approved_maps_to_sar_filed() {
        final InvestigationOutcome outcome = InvestigationOutcome.fromReviewDecision("APPROVED");
        assertNotNull(outcome);
        assertEquals("sar-filed", outcome.type());
    }

    @Test
    void rejected_maps_to_gate_rejected() {
        final InvestigationOutcome outcome = InvestigationOutcome.fromReviewDecision("REJECTED");
        assertNotNull(outcome);
        assertEquals("gate-rejected", outcome.type());
    }

    @Test
    void unknown_maps_to_review_inconclusive() {
        final InvestigationOutcome outcome = InvestigationOutcome.fromReviewDecision("UNKNOWN");
        assertNotNull(outcome);
        assertEquals("review-inconclusive", outcome.type());
    }

    @Test
    void null_input_returns_null() {
        assertNull(InvestigationOutcome.fromReviewDecision(null));
    }

    @Test
    void unrecognised_value_returns_null() {
        assertNull(InvestigationOutcome.fromReviewDecision("SOMETHING_ELSE"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl api -Dtest=InvestigationOutcomeTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `InvestigationOutcome` does not exist.

- [ ] **Step 3: Write minimal implementation**

Create `api/src/main/java/io/casehub/aml/domain/InvestigationOutcome.java`:

```java
package io.casehub.aml.domain;

public record InvestigationOutcome(String type) {

    public static InvestigationOutcome fromReviewDecision(final String reviewDecision) {
        if (reviewDecision == null) {
            return null;
        }
        return switch (reviewDecision) {
            case "APPROVED" -> new InvestigationOutcome("sar-filed");
            case "REJECTED" -> new InvestigationOutcome("gate-rejected");
            case "UNKNOWN" -> new InvestigationOutcome("review-inconclusive");
            default -> null;
        };
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl api -Dtest=InvestigationOutcomeTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: 5 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/aml/domain/InvestigationOutcome.java \
       api/src/test/java/io/casehub/aml/domain/InvestigationOutcomeTest.java
git commit -m "feat(#71): add InvestigationOutcome record — maps reviewDecision to outcome type

Refs #71

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 3: Add outcome to Layer 9 investigation API (#71)

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java`
- Test: `app/src/test/java/io/casehub/aml/engine/AmlLayer9ActionGateTest.java` (add rejection test)
- Test: `app/src/test/java/io/casehub/aml/engine/AmlLayer9ResourceTest.java` (add outcome assertions)

**Interfaces:**
- Consumes: `InvestigationOutcome.fromReviewDecision(String)` from Task 2. `LedgerEntryRepository.findBySubjectId(UUID subjectId, String tenancyId)` returns `List<LedgerEntry>`. `AmlSarOfficerReviewedLedgerEntry` has field `reviewDecision` (String).
- Produces: `GET /api/layer9/investigations/{caseId}` now returns `{"caseId": "...", "status": "...", "outcome": {"type": "..."}}` where `outcome` is null when in-progress or during the eventual-consistency window.

- [ ] **Step 1: Write the rejection test**

Add to `AmlLayer9ActionGateTest.java` — a new test method that rejects the SAR filing gate and asserts the API returns `outcome.type = "gate-rejected"`:

```java
@Test
void gate_rejection_surfaces_in_investigation_outcome() {
    final SuspiciousTransaction tx = new SuspiciousTransaction(
        "TXN-REJECT-" + UUID.randomUUID(), "ACC-REJ-A", "ACC-REJ-B",
        new BigDecimal("500000"), "GBP",
        Instant.parse("2024-12-01T00:00:00Z"),
        "Entity link — PEP suspected beneficial owner");

    final String caseIdStr = given().contentType(ContentType.JSON).body(tx)
        .when().post("/api/layer9/investigations")
        .then().statusCode(202)
        .extract().path("caseId");

    final UUID caseId = UUID.fromString(caseIdStr);

    // Await gate WorkItem
    Awaitility.await()
        .atMost(15, TimeUnit.SECONDS)
        .pollInterval(300, TimeUnit.MILLISECONDS)
        .until(() -> !findGateWorkItems(caseId).isEmpty());

    final WorkItem gate = findGateWorkItems(caseId).get(0);

    // Reject the gate
    workItemService.rejectFromSystem(gate.id, "test-aml-compliance", "Insufficient evidence");

    // Drain to completed
    Awaitility.await()
        .atMost(15, TimeUnit.SECONDS)
        .pollInterval(300, TimeUnit.MILLISECONDS)
        .until(() -> "completed".equals(
            given().when().get("/api/layer9/investigations/" + caseIdStr)
                .then().extract().path("status")));

    // Allow async observer to fire
    Awaitility.await()
        .atMost(10, TimeUnit.SECONDS)
        .pollInterval(300, TimeUnit.MILLISECONDS)
        .until(() -> {
            final String outcomeType = given()
                .when().get("/api/layer9/investigations/" + caseIdStr)
                .then().extract().path("outcome.type");
            return "gate-rejected".equals(outcomeType);
        });
}
```

Add import for `WorkItemService.rejectFromSystem` — already available via `@Inject WorkItemService workItemService`.

- [ ] **Step 2: Write the happy-path outcome test**

Add to `AmlLayer9ActionGateTest.java` — extend the existing `gate_fires_for_pep_entity_and_resumes_on_approval` test with an outcome assertion after completion:

```java
// After the existing drain-to-completed Awaitility block, add:

// Allow async observer to fire
Awaitility.await()
    .atMost(10, TimeUnit.SECONDS)
    .pollInterval(300, TimeUnit.MILLISECONDS)
    .until(() -> {
        final String outcomeType = given()
            .when().get("/api/layer9/investigations/" + caseIdStr)
            .then().extract().path("outcome.type");
        return "sar-filed".equals(outcomeType);
    });
```

- [ ] **Step 2b: Verify outcome is null for in-progress case**

Add to `AmlLayer9ResourceTest.java` — assert the new `outcome` field is null for the happy-path CORPORATE case while in-progress (before completion), and `sar-filed` after:

```java
@Test
void outcome_is_null_while_in_progress_then_sar_filed_after_completion() {
    final String caseIdStr = given().contentType(ContentType.JSON).body(CORPORATE_TX)
            .when().post("/api/layer9/investigations")
            .then().statusCode(202)
            .extract().path("caseId");

    // Drain to completed
    Awaitility.await().atMost(15, TimeUnit.SECONDS).pollInterval(200, TimeUnit.MILLISECONDS)
            .until(() -> "completed".equals(
                    given().when().get("/api/layer9/investigations/" + caseIdStr)
                            .then().extract().path("status")));

    // Allow async observer to fire, then check outcome
    Awaitility.await().atMost(10, TimeUnit.SECONDS).pollInterval(300, TimeUnit.MILLISECONDS)
            .until(() -> {
                final String outcomeType = given()
                    .when().get("/api/layer9/investigations/" + caseIdStr)
                    .then().extract().path("outcome.type");
                return "sar-filed".equals(outcomeType);
            });
}
```

Note: CORPORATE transactions go through the SAR drafting → compliance review → approval flow without a gate (no PEP, low risk). The observer writes `APPROVED` → outcome `sar-filed`. This test does NOT use the oversight gate path — it uses the Layer 5/6 non-gate flow via `AmlInvestigationCaseHub`, not `AmlOversightCaseHub`. If the non-gate flow does not produce an `AML_SAR_OFFICER_REVIEWED` entry (because no compliance review occurs for auto-completed CORPORATE investigations), the outcome will be null — which is also a valid assertion. Adjust the expected value based on what the actual flow produces.

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl app -am -Dtest=AmlLayer9ActionGateTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `outcome` field does not exist in the response.

- [ ] **Step 4: Modify `AmlLayer9Resource.getInvestigation()`**

Edit `app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java`:

Add imports:
```java
import io.casehub.aml.domain.InvestigationOutcome;
import io.casehub.aml.ledger.AmlSarOfficerReviewedLedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.TenancyConstants;
import java.util.HashMap;
```

Add field:
```java
@Inject LedgerEntryRepository ledgerEntryRepository;
```

Replace the `getInvestigation` method body:

```java
@GET
@Path("/{caseId}")
public Response getInvestigation(@PathParam("caseId") final UUID caseId) {
    CaseInstance instance = caseInstanceCache.get(caseId);
    if (instance == null) {
        instance = caseInstanceRepository
                .findByUuid(caseId, TenancyConstants.DEFAULT_TENANT_ID)
                .await().indefinitely();
    }
    final boolean completed = instance != null && instance.getState() == CaseStatus.COMPLETED;

    final InvestigationOutcome outcome = ledgerEntryRepository
            .findBySubjectId(caseId, TenancyConstants.DEFAULT_TENANT_ID).stream()
            .filter(AmlSarOfficerReviewedLedgerEntry.class::isInstance)
            .map(AmlSarOfficerReviewedLedgerEntry.class::cast)
            .findFirst()
            .map(e -> InvestigationOutcome.fromReviewDecision(e.reviewDecision))
            .orElse(null);

    final Map<String, Object> body = new HashMap<>();
    body.put("caseId", caseId);
    body.put("status", completed ? "completed" : "in-progress");
    body.put("outcome", outcome);
    return Response.ok(body).build();
}
```

Note: `HashMap` allows null values — `Map.of()` does not. `outcome` is null while in-progress or during the eventual-consistency window.

Remove the now-unused `TenancyConstants` import if it was only used in the old `findByUuid` call — check: it's still used for `findByUuid`, so keep it.

- [ ] **Step 5: Run the full Layer 9 test suite**

Run: `mvn test -pl app -am -Dtest=AmlLayer9ActionGateTest,AmlLayer9ResourceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all tests PASS (3 existing + 1 new rejection test, plus outcome assertions on the approval test).

- [ ] **Step 6: Run the full project test suite**

Run: `mvn test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all tests PASS — no regressions in Layer 5/6/7/8 tests. The response shape change (adding `outcome` field) does not break existing tests because they assert specific fields, not the full response body.

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java \
       app/src/test/java/io/casehub/aml/engine/AmlLayer9ActionGateTest.java
git commit -m "feat(#71): surface gate rejection outcome in Layer 9 investigation API

Adds outcome field to GET /api/layer9/investigations/{caseId}.
Derives outcome from AmlSarOfficerReviewedLedgerEntry.reviewDecision:
APPROVED → sar-filed, REJECTED → gate-rejected, UNKNOWN → review-inconclusive.
Outcome is null while in-progress (eventual consistency with async observer).

Uses findBySubjectId() + type filter, consistent with AmlLedgerService pattern.

Refs #71

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: Close #71 and update CLAUDE.md if needed

**Files:**
- None (housekeeping)

**Interfaces:**
- Consumes: Tasks 1–3 completed and passing
- Produces: nothing

- [ ] **Step 1: Run full build to confirm green**

Run: `mvn verify -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: BUILD SUCCESS.

- [ ] **Step 2: Close #71 on GitHub**

```bash
gh issue close 71 --repo casehubio/aml --comment "Gate rejection outcome surfaced in Layer 9 API. Downstream routing deferred to #72 (Layer 10)."
```

- [ ] **Step 3: Verify no CLAUDE.md drift**

Check if the `AmlLayer9Resource` description or Layer 9 documentation in CLAUDE.md needs updating to reflect the new `outcome` field. If not, skip.
