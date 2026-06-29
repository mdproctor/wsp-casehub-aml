# Batch Cleanup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close #69 (no code changes), rename CI workflow (#68), and add gate rejection outcome to the Layer 9 investigation API (#71).

**Architecture:** #71 adds an `InvestigationOutcome` record to the `api/` module (JPA-free) and an `AmlInvestigationOutcomeService` to the `app/` module that encapsulates outcome derivation from the ledger. `AmlLayer9Resource` delegates to the service — thin dispatcher only, consistent with `AmlLayer7Resource` → `AmlComplianceEvidenceService`. Entry selection prefers `actorType=HUMAN` over `SYSTEM` to handle the double-try/catch race in the observer. Lifecycle status and domain outcome are separate concerns.

**Tech Stack:** Java 21, Quarkus 3.32.2, REST Assured, Awaitility, casehub-ledger `LedgerEntryRepository`

## Global Constraints

- `TenancyConstants.DEFAULT_TENANT_ID` at all ledger call sites
- `LedgerEntryRepository.findBySubjectId()` + type filter — never `findLatestBySubjectId()` for typed queries
- Entry selection rule: prefer `actorType = ActorType.HUMAN` over `ActorType.SYSTEM` when multiple `AmlSarOfficerReviewedLedgerEntry` entries exist for the same case
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
- Produces: `InvestigationOutcome.fromReviewDecision(String reviewDecision)` returns `InvestigationOutcome` with `type()` returning `"sar-filed"`, `"gate-rejected"`, or `"decision-not-recorded"`. Returns `null` for null or unrecognised values.

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
    void unknown_maps_to_decision_not_recorded() {
        final InvestigationOutcome outcome = InvestigationOutcome.fromReviewDecision("UNKNOWN");
        assertNotNull(outcome);
        assertEquals("decision-not-recorded", outcome.type());
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
            case "UNKNOWN" -> new InvestigationOutcome("decision-not-recorded");
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

### Task 3: Add `AmlInvestigationOutcomeService` with entry selection (#71)

**Files:**
- Create: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationOutcomeService.java`
- Test: `app/src/test/java/io/casehub/aml/engine/AmlInvestigationOutcomeServiceTest.java`

**Interfaces:**
- Consumes: `InvestigationOutcome.fromReviewDecision(String)` from Task 2. `LedgerEntryRepository.findBySubjectId(UUID subjectId, String tenancyId)` returns `List<LedgerEntry>`. `AmlSarOfficerReviewedLedgerEntry` has fields `reviewDecision` (String) and `actorType` (`ActorType`).
- Produces: `AmlInvestigationOutcomeService.resolve(UUID caseId)` returns `InvestigationOutcome` or `null`. Entry selection: prefer `actorType=HUMAN` over `SYSTEM`. Used by `AmlLayer9Resource` in Task 4.

- [ ] **Step 1: Write the service unit test**

Create `app/src/test/java/io/casehub/aml/engine/AmlInvestigationOutcomeServiceTest.java`:

```java
package io.casehub.aml.engine;

import io.casehub.aml.domain.InvestigationOutcome;
import io.casehub.aml.ledger.AmlSarOfficerReviewedLedgerEntry;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.platform.api.identity.TenancyConstants;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Optional;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class AmlInvestigationOutcomeServiceTest {

    @Test
    void returns_null_when_no_officer_review_entry() {
        final AmlInvestigationOutcomeService service = serviceWith(List.of());
        assertNull(service.resolve(UUID.randomUUID()));
    }

    @Test
    void returns_sar_filed_for_approved_human_entry() {
        final AmlSarOfficerReviewedLedgerEntry entry = officerEntry("APPROVED", ActorType.HUMAN);
        final AmlInvestigationOutcomeService service = serviceWith(List.of(entry));
        final InvestigationOutcome outcome = service.resolve(UUID.randomUUID());
        assertNotNull(outcome);
        assertEquals("sar-filed", outcome.type());
    }

    @Test
    void returns_gate_rejected_for_rejected_human_entry() {
        final AmlSarOfficerReviewedLedgerEntry entry = officerEntry("REJECTED", ActorType.HUMAN);
        final AmlInvestigationOutcomeService service = serviceWith(List.of(entry));
        final InvestigationOutcome outcome = service.resolve(UUID.randomUUID());
        assertNotNull(outcome);
        assertEquals("gate-rejected", outcome.type());
    }

    @Test
    void returns_decision_not_recorded_for_unknown_system_entry() {
        final AmlSarOfficerReviewedLedgerEntry entry = officerEntry("UNKNOWN", ActorType.SYSTEM);
        final AmlInvestigationOutcomeService service = serviceWith(List.of(entry));
        final InvestigationOutcome outcome = service.resolve(UUID.randomUUID());
        assertNotNull(outcome);
        assertEquals("decision-not-recorded", outcome.type());
    }

    @Test
    void prefers_human_entry_over_system_entry_in_race() {
        final AmlSarOfficerReviewedLedgerEntry humanEntry = officerEntry("APPROVED", ActorType.HUMAN);
        final AmlSarOfficerReviewedLedgerEntry systemEntry = officerEntry("UNKNOWN", ActorType.SYSTEM);
        final AmlInvestigationOutcomeService service = serviceWith(List.of(humanEntry, systemEntry));
        final InvestigationOutcome outcome = service.resolve(UUID.randomUUID());
        assertNotNull(outcome);
        assertEquals("sar-filed", outcome.type());
    }

    @Test
    void prefers_human_entry_regardless_of_list_order() {
        final AmlSarOfficerReviewedLedgerEntry humanEntry = officerEntry("REJECTED", ActorType.HUMAN);
        final AmlSarOfficerReviewedLedgerEntry systemEntry = officerEntry("UNKNOWN", ActorType.SYSTEM);
        // SYSTEM entry first in list
        final AmlInvestigationOutcomeService service = serviceWith(List.of(systemEntry, humanEntry));
        final InvestigationOutcome outcome = service.resolve(UUID.randomUUID());
        assertNotNull(outcome);
        assertEquals("gate-rejected", outcome.type());
    }

    private static AmlSarOfficerReviewedLedgerEntry officerEntry(final String decision,
            final ActorType actorType) {
        final AmlSarOfficerReviewedLedgerEntry entry = new AmlSarOfficerReviewedLedgerEntry();
        entry.reviewDecision = decision;
        entry.actorType = actorType;
        return entry;
    }

    private static AmlInvestigationOutcomeService serviceWith(
            final List<LedgerEntry> entries) {
        final LedgerEntryRepository repo = new LedgerEntryRepository() {
            @Override
            public List<LedgerEntry> findBySubjectId(UUID subjectId, String tenancyId) {
                return entries;
            }
            // Stub remaining methods — not needed for this test
            @Override public void save(LedgerEntry entry, String tenancyId) {}
            @Override public Optional<LedgerEntry> findLatestBySubjectId(UUID subjectId, String tenancyId) { return Optional.empty(); }
            @Override public Optional<LedgerEntry> findEntryById(UUID id, String tenancyId) { return Optional.empty(); }
        };
        return new AmlInvestigationOutcomeService(repo);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl app -am -Dtest=AmlInvestigationOutcomeServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `AmlInvestigationOutcomeService` does not exist.

- [ ] **Step 3: Write the service implementation**

Create `app/src/main/java/io/casehub/aml/engine/AmlInvestigationOutcomeService.java`:

```java
package io.casehub.aml.engine;

import io.casehub.aml.domain.InvestigationOutcome;
import io.casehub.aml.ledger.AmlSarOfficerReviewedLedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.platform.api.identity.TenancyConstants;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Comparator;
import java.util.UUID;

@ApplicationScoped
public class AmlInvestigationOutcomeService {

    private static final Comparator<AmlSarOfficerReviewedLedgerEntry> HUMAN_FIRST =
            Comparator.comparing(
                    (AmlSarOfficerReviewedLedgerEntry e) -> e.actorType == ActorType.HUMAN ? 0 : 1);

    private final LedgerEntryRepository ledgerEntryRepository;

    @Inject
    public AmlInvestigationOutcomeService(final LedgerEntryRepository ledgerEntryRepository) {
        this.ledgerEntryRepository = ledgerEntryRepository;
    }

    public InvestigationOutcome resolve(final UUID caseId) {
        return ledgerEntryRepository
                .findBySubjectId(caseId, TenancyConstants.DEFAULT_TENANT_ID).stream()
                .filter(AmlSarOfficerReviewedLedgerEntry.class::isInstance)
                .map(AmlSarOfficerReviewedLedgerEntry.class::cast)
                .min(HUMAN_FIRST)
                .map(e -> InvestigationOutcome.fromReviewDecision(e.reviewDecision))
                .orElse(null);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl app -am -Dtest=AmlInvestigationOutcomeServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: 6 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/aml/engine/AmlInvestigationOutcomeService.java \
       app/src/test/java/io/casehub/aml/engine/AmlInvestigationOutcomeServiceTest.java
git commit -m "feat(#71): add AmlInvestigationOutcomeService — entry selection prefers HUMAN over SYSTEM

Encapsulates outcome derivation from ledger: findBySubjectId + type filter +
actorType preference (HUMAN wins over SYSTEM in the double-try/catch race).

Refs #71

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: Wire outcome into Layer 9 investigation API (#71)

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java`
- Test: `app/src/test/java/io/casehub/aml/engine/AmlLayer9ActionGateTest.java` (add rejection test)
- Test: `app/src/test/java/io/casehub/aml/engine/AmlLayer9ResourceTest.java` (add outcome assertion)

**Interfaces:**
- Consumes: `AmlInvestigationOutcomeService.resolve(UUID caseId)` from Task 3. `WorkItemService.rejectFromSystem(UUID id, String actorId, String reason)` for the rejection test.
- Produces: `GET /api/layer9/investigations/{caseId}` now returns `{"caseId": "...", "status": "...", "outcome": {"type": "..."}}` where `outcome` is null when in-progress or during the eventual-consistency window (max 5 seconds).

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

- [ ] **Step 2: Write the happy-path outcome test**

Add to `AmlLayer9ActionGateTest.java` — extend the existing `gate_fires_for_pep_entity_and_resumes_on_approval` test. After the existing drain-to-completed Awaitility block, add:

```java
// Allow async observer to fire — assert outcome
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

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl app -am -Dtest=AmlLayer9ActionGateTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `outcome` field does not exist in the response.

- [ ] **Step 4: Modify `AmlLayer9Resource.getInvestigation()`**

Edit `app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java`:

Add import:
```java
import io.casehub.aml.domain.InvestigationOutcome;
import java.util.HashMap;
```

Add field:
```java
@Inject AmlInvestigationOutcomeService outcomeService;
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
    final InvestigationOutcome outcome = outcomeService.resolve(caseId);

    final Map<String, Object> body = new HashMap<>();
    body.put("caseId", caseId);
    body.put("status", completed ? "completed" : "in-progress");
    body.put("outcome", outcome);
    return Response.ok(body).build();
}
```

Note: `HashMap` allows null values — `Map.of()` does not.

- [ ] **Step 5: Run the full Layer 9 test suite**

Run: `mvn test -pl app -am -Dtest=AmlLayer9ActionGateTest,AmlLayer9ResourceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all tests PASS.

- [ ] **Step 6: Run the full project test suite**

Run: `mvn test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all tests PASS — no regressions.

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java \
       app/src/test/java/io/casehub/aml/engine/AmlLayer9ActionGateTest.java
git commit -m "feat(#71): wire outcome into Layer 9 investigation API

AmlLayer9Resource delegates to AmlInvestigationOutcomeService — thin
dispatcher pattern. Adds gate rejection test via rejectFromSystem().

Outcome types: sar-filed, gate-rejected, decision-not-recorded.
Null while in-progress or during eventual-consistency window (max 5s).

Closes #71

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 5: Final verification and close

**Files:**
- None (housekeeping)

**Interfaces:**
- Consumes: Tasks 1–4 completed and passing
- Produces: nothing

- [ ] **Step 1: Run full build to confirm green**

Run: `mvn verify -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: BUILD SUCCESS.

- [ ] **Step 2: Verify no CLAUDE.md drift**

Check if the `AmlLayer9Resource` description or Layer 9 documentation in CLAUDE.md needs updating to reflect the new `outcome` field and `AmlInvestigationOutcomeService`. If not, skip.
