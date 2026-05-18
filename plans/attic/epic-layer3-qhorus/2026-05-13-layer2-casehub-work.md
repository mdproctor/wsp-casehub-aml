# Layer 2: casehub-work — Compliance Officer WorkItem Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire casehub-work into the AML investigation flow so that completing an investigation creates a compliance officer WorkItem with a 30-day FinCEN claim deadline.

**Architecture:** Hexagonal — `api/` stays pure domain (new `AmlInvestigationResult` record only); `app/` owns the use-case interface `AmlInvestigationApplicationService` and both implementations (`NaiveAmlInvestigationService` as `@DefaultBean`, `WorkItemAmlInvestigationService` as active `@ApplicationScoped`). CDI resolves to the Layer 2 implementation automatically with no config switch.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-work `WorkItemService`, JUnit 5, REST Assured, `@QuarkusTest` with H2 `MODE=PostgreSQL`.

**Issue:** casehubio/aml#15  
**Spec:** `specs/2026-05-13-layer2-casehub-work-design.md`

---

## File Map

| Action | File | Responsibility |
|--------|------|---------------|
| CREATE | `api/src/main/java/io/casehub/aml/domain/AmlInvestigationResult.java` | Result record crossing domain/application boundary |
| CREATE | `app/src/main/java/io/casehub/aml/AmlInvestigationApplicationService.java` | Use-case port interface |
| MODIFY | `app/src/main/java/io/casehub/aml/tutorial/NaiveAmlInvestigationService.java` | Add `@DefaultBean`, implement interface, return `AmlInvestigationResult` |
| CREATE | `app/src/main/java/io/casehub/aml/tutorial/WorkItemAmlInvestigationService.java` | Layer 2 — active CDI bean, creates compliance WorkItem |
| MODIFY | `app/src/main/java/io/casehub/aml/AmlInvestigationResource.java` | Inject interface, return `AmlInvestigationResult` |
| MODIFY | `app/src/test/java/io/casehub/aml/tutorial/NaiveAmlInvestigationServiceTest.java` | Add null `complianceReviewTaskId` assertion |
| CREATE | `app/src/test/java/io/casehub/aml/tutorial/WorkItemAmlInvestigationServiceTest.java` | `@QuarkusTest` — WorkItem fields, claimDeadline, callerRef |
| MODIFY | `app/src/test/java/io/casehub/aml/AmlInvestigationResourceTest.java` | Update paths to `summary.*`, add `complianceReviewTaskId` assertion |

---

## Task 1: Add `AmlInvestigationResult` to `api/`

**Files:**
- Create: `api/src/main/java/io/casehub/aml/domain/AmlInvestigationResult.java`

- [ ] **Step 1: Create the record**

```java
package io.casehub.aml.domain;

public record AmlInvestigationResult(
        InvestigationSummary summary,
        String complianceReviewTaskId) {
}
```

- [ ] **Step 2: Verify `api/` still builds with no new dependencies**

```bash
mvn verify -pl api -am -Dsurefire.failIfNoSpecifiedTests=false
```
Expected: BUILD SUCCESS. No new deps in `api/pom.xml` — this record uses only existing domain types.

- [ ] **Step 3: Commit**

```bash
cd /Users/mdproctor/claude/casehub/aml
git add api/src/main/java/io/casehub/aml/domain/AmlInvestigationResult.java
git commit -m "feat: add AmlInvestigationResult domain record

Pure Java record in api/ — InvestigationSummary + complianceReviewTaskId.
No new api/ dependencies (api/ stays foundation-free per PP-20260512-9b8847).

Refs #15"
```

---

## Task 2: Add `AmlInvestigationApplicationService` interface to `app/`

**Files:**
- Create: `app/src/main/java/io/casehub/aml/AmlInvestigationApplicationService.java`

- [ ] **Step 1: Create the use-case port interface**

```java
package io.casehub.aml;

import io.casehub.aml.domain.AmlInvestigationResult;
import io.casehub.aml.domain.SuspiciousTransaction;

public interface AmlInvestigationApplicationService {
    AmlInvestigationResult investigate(SuspiciousTransaction transaction);
}
```

- [ ] **Step 2: Compile to confirm — no tests yet**

```bash
mvn compile -pl app -am
```
Expected: BUILD SUCCESS. No implementations exist yet, so no CDI issues at compile time.

- [ ] **Step 3: Commit**

```bash
git add app/src/main/java/io/casehub/aml/AmlInvestigationApplicationService.java
git commit -m "feat: add AmlInvestigationApplicationService use-case interface

Application-layer port for the investigation use case. Implementations
live in app/tutorial/. Resource injects this interface; CDI picks the
active implementation at runtime.

Refs #15"
```

---

## Task 3: Refactor `NaiveAmlInvestigationService` — implement interface, add `@DefaultBean`

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/tutorial/NaiveAmlInvestigationService.java`
- Modify: `app/src/test/java/io/casehub/aml/tutorial/NaiveAmlInvestigationServiceTest.java`

- [ ] **Step 1: Write the failing test first — update `NaiveAmlInvestigationServiceTest`**

Replace the existing file content with:

```java
package io.casehub.aml.tutorial;

import java.math.BigDecimal;
import java.time.Instant;

import org.junit.jupiter.api.Test;

import io.casehub.aml.domain.AmlInvestigationResult;
import io.casehub.aml.domain.SuspiciousTransaction;

import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertNotSame;
import static org.junit.jupiter.api.Assertions.assertNull;
import static org.junit.jupiter.api.Assertions.assertSame;

class NaiveAmlInvestigationServiceTest {

    private final NaiveAmlInvestigationService service = new NaiveAmlInvestigationService();

    private SuspiciousTransaction tx(String id) {
        return new SuspiciousTransaction(
                id, "ACC-A", "ACC-B",
                new BigDecimal("100000"), "USD",
                Instant.parse("2024-03-15T10:00:00Z"), "Structuring");
    }

    @Test
    void investigate_validTransaction_returnsCompleteSummary() {
        AmlInvestigationResult result = service.investigate(tx("TXN-001"));

        assertNotNull(result);
        assertNotNull(result.summary());
        assertNotNull(result.summary().entityResolution());
        assertNotNull(result.summary().patternAnalysis());
        assertNotNull(result.summary().osintScreening());
        assertNotNull(result.summary().sarNarrative());
    }

    @Test
    void investigate_noWorkItem_complianceReviewTaskIdIsNull() {
        AmlInvestigationResult result = service.investigate(tx("TXN-001"));
        // Layer 1 naive implementation never creates a WorkItem
        assertNull(result.complianceReviewTaskId());
    }

    @Test
    void investigate_preservesTransactionIdentity() {
        SuspiciousTransaction input = tx("TXN-002");
        assertSame(input, service.investigate(input).summary().transaction());
    }

    @Test
    void investigate_calledTwice_producesIndependentResults() {
        AmlInvestigationResult first  = service.investigate(tx("TXN-003"));
        AmlInvestigationResult second = service.investigate(tx("TXN-004"));

        assertNotNull(first);
        assertNotNull(second);
        assertNotSame(first, second);
    }
}
```

- [ ] **Step 2: Run to confirm it fails**

```bash
mvn test -pl app -am -Dtest=NaiveAmlInvestigationServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```
Expected: FAIL — `investigate()` still returns `InvestigationSummary`, not `AmlInvestigationResult`.

- [ ] **Step 3: Refactor `NaiveAmlInvestigationService`**

Replace the existing file content with:

```java
package io.casehub.aml.tutorial;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.aml.AmlInvestigationApplicationService;
import io.casehub.aml.domain.AmlInvestigationResult;
import io.casehub.aml.domain.EntityResolutionResult;
import io.casehub.aml.domain.InvestigationSummary;
import io.casehub.aml.domain.OsintResult;
import io.casehub.aml.domain.PatternAnalysisResult;
import io.casehub.aml.domain.SuspiciousTransaction;

@ApplicationScoped
@DefaultBean
public class NaiveAmlInvestigationService implements AmlInvestigationApplicationService {

    private final NaiveEntityResolutionService entityResolutionService = new NaiveEntityResolutionService();
    private final NaivePatternAnalysisService  patternAnalysisService  = new NaivePatternAnalysisService();
    private final NaiveOsintScreeningService   osintScreeningService   = new NaiveOsintScreeningService();
    private final NaiveSarDraftingService      sarDraftingService      = new NaiveSarDraftingService();

    @Override
    public AmlInvestigationResult investigate(SuspiciousTransaction transaction) {
        // LAYER 1 GAP: no attribution — who resolved this entity graph?
        // No record of which agent made this decision or when.
        EntityResolutionResult entity = entityResolutionService.resolve(transaction);

        // LAYER 1 GAP: no failure resilience — if this call times out or throws,
        // the entire investigation is lost with no trace of partial work.
        PatternAnalysisResult pattern = patternAnalysisService.analyze(transaction);

        // LAYER 1 GAP: no deadline tracking — OSINT runs sequentially after pattern
        // analysis. No FinCEN 30-day SLA. No parallel execution. No formal obligation.
        OsintResult osint = osintScreeningService.screen(transaction);

        // LAYER 1 GAP: no audit trail — this narrative cannot be proven to FinCEN.
        // No tamper-evident record of the reasoning chain exists.
        String sarNarrative = sarDraftingService.draft(transaction, entity, pattern, osint);

        InvestigationSummary summary = new InvestigationSummary(transaction, entity, pattern, osint, sarNarrative);
        return new AmlInvestigationResult(summary, null);
    }
}
```

- [ ] **Step 4: Run tests — confirm they pass**

```bash
mvn test -pl app -am -Dtest=NaiveAmlInvestigationServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```
Expected: 4 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/aml/tutorial/NaiveAmlInvestigationService.java \
        app/src/test/java/io/casehub/aml/tutorial/NaiveAmlInvestigationServiceTest.java
git commit -m "refactor: NaiveAmlInvestigationService implements AmlInvestigationApplicationService

Adds @ApplicationScoped @DefaultBean — becomes the CDI fallback when no other
AmlInvestigationApplicationService bean is present. Returns AmlInvestigationResult
with null complianceReviewTaskId (Layer 1 has no WorkItem obligation).

Refs #15"
```

---

## Task 4: Create `WorkItemAmlInvestigationService`

**Files:**
- Create: `app/src/test/java/io/casehub/aml/tutorial/WorkItemAmlInvestigationServiceTest.java`
- Create: `app/src/main/java/io/casehub/aml/tutorial/WorkItemAmlInvestigationService.java`

- [ ] **Step 1: Write the failing `@QuarkusTest`**

```java
package io.casehub.aml.tutorial;

import java.math.BigDecimal;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.aml.AmlInvestigationApplicationService;
import io.casehub.aml.domain.AmlInvestigationResult;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.work.runtime.model.WorkItem;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertTrue;

@QuarkusTest
class WorkItemAmlInvestigationServiceTest {

    @Inject
    AmlInvestigationApplicationService service;

    private SuspiciousTransaction tx(String id) {
        return new SuspiciousTransaction(
                id, "ACC-A", "ACC-B",
                new BigDecimal("100000"), "USD",
                Instant.parse("2024-03-15T10:00:00Z"), "Structuring");
    }

    @Test
    @Transactional
    void investigate_createsWorkItemWithComplianceFields() {
        AmlInvestigationResult result = service.investigate(tx("TXN-LAYER2"));

        assertNotNull(result.complianceReviewTaskId(),
                "Layer 2 must return a complianceReviewTaskId");

        WorkItem item = WorkItem.findById(UUID.fromString(result.complianceReviewTaskId()));
        assertNotNull(item, "WorkItem must be persisted");
        assertTrue(item.candidateGroups.contains("compliance-officers"),
                "Must be routed to compliance-officers");
        assertTrue(item.claimDeadline.isAfter(Instant.now().plus(29, ChronoUnit.DAYS)),
                "claimDeadline must be ~30 days out");
        assertTrue(item.callerRef.contains("TXN-LAYER2"),
                "callerRef must reference the transaction ID");
    }

    @Test
    @Transactional
    void investigate_returnsSummaryWithAllFindings() {
        AmlInvestigationResult result = service.investigate(tx("TXN-LAYER2-B"));

        assertNotNull(result.summary());
        assertNotNull(result.summary().entityResolution());
        assertNotNull(result.summary().patternAnalysis());
        assertNotNull(result.summary().osintScreening());
        assertNotNull(result.summary().sarNarrative());
        assertEquals("TXN-LAYER2-B", result.summary().transaction().id());
    }
}
```

- [ ] **Step 2: Run to confirm it fails**

```bash
mvn test -pl app -am -Dtest=WorkItemAmlInvestigationServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```
Expected: FAIL — `AmlInvestigationApplicationService` resolves to `NaiveAmlInvestigationService` (`@DefaultBean`), which returns null `complianceReviewTaskId`.

- [ ] **Step 3: Create `WorkItemAmlInvestigationService`**

```java
package io.casehub.aml.tutorial;

import java.time.Instant;
import java.time.temporal.ChronoUnit;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.casehub.aml.AmlInvestigationApplicationService;
import io.casehub.aml.domain.AmlInvestigationResult;
import io.casehub.aml.domain.EntityResolutionResult;
import io.casehub.aml.domain.InvestigationSummary;
import io.casehub.aml.domain.OsintResult;
import io.casehub.aml.domain.PatternAnalysisResult;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemCreateRequest;
import io.casehub.work.runtime.model.WorkItemPriority;
import io.casehub.work.runtime.service.WorkItemService;

@ApplicationScoped
public class WorkItemAmlInvestigationService implements AmlInvestigationApplicationService {

    @Inject
    WorkItemService workItemService;

    private final NaiveEntityResolutionService entityResolutionService = new NaiveEntityResolutionService();
    private final NaivePatternAnalysisService  patternAnalysisService  = new NaivePatternAnalysisService();
    private final NaiveOsintScreeningService   osintScreeningService   = new NaiveOsintScreeningService();
    private final NaiveSarDraftingService      sarDraftingService      = new NaiveSarDraftingService();

    @Override
    public AmlInvestigationResult investigate(SuspiciousTransaction transaction) {
        EntityResolutionResult entity  = entityResolutionService.resolve(transaction);
        PatternAnalysisResult  pattern = patternAnalysisService.analyze(transaction);
        OsintResult            osint   = osintScreeningService.screen(transaction);
        String sarNarrative            = sarDraftingService.draft(transaction, entity, pattern, osint);

        InvestigationSummary summary = new InvestigationSummary(transaction, entity, pattern, osint, sarNarrative);

        // LAYER 2: create a compliance officer WorkItem with the FinCEN 30-day claim SLA.
        // The compliance officer has 30 days from investigation completion to review and file.
        WorkItem workItem = workItemService.create(new WorkItemCreateRequest(
                "Compliance review — SAR for transaction " + transaction.id(),
                null,
                "aml-compliance",
                null,
                WorkItemPriority.HIGH,
                null,
                "compliance-officers",
                null,
                null,
                "aml-system",
                null,
                Instant.now().plus(30, ChronoUnit.DAYS),
                null,
                null,
                null,
                null,
                "aml:investigation/" + transaction.id(),
                null,
                null
        ));

        return new AmlInvestigationResult(summary, workItem.id.toString());
    }
}
```

- [ ] **Step 4: Run tests — confirm they pass**

```bash
mvn test -pl app -am -Dtest=WorkItemAmlInvestigationServiceTest -Dsurefire.failIfNoSpecifiedTests=false
```
Expected: 2 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/aml/tutorial/WorkItemAmlInvestigationService.java \
        app/src/test/java/io/casehub/aml/tutorial/WorkItemAmlInvestigationServiceTest.java
git commit -m "feat: WorkItemAmlInvestigationService — Layer 2 compliance WorkItem

@ApplicationScoped displaces @DefaultBean (NaiveAmlInvestigationService).
Creates a compliance officer WorkItem with 30-day claimDeadline (FinCEN SLA)
and candidateGroups=compliance-officers after every investigation completes.

Refs #15"
```

---

## Task 5: Update `AmlInvestigationResource`

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/AmlInvestigationResource.java`

No new test needed — covered by `AmlInvestigationResourceTest` in Task 6.

- [ ] **Step 1: Update the resource to inject the interface and return `AmlInvestigationResult`**

Replace the existing file with:

```java
package io.casehub.aml;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;

import io.casehub.aml.domain.AmlInvestigationResult;
import io.casehub.aml.domain.SuspiciousTransaction;

@Path("/api/investigations")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class AmlInvestigationResource {

    @Inject
    AmlInvestigationApplicationService investigationService;

    @POST
    public AmlInvestigationResult investigate(SuspiciousTransaction transaction) {
        return investigationService.investigate(transaction);
    }
}
```

- [ ] **Step 2: Compile only — tests will be fixed in Task 6**

```bash
mvn compile -pl app -am
```
Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git add app/src/main/java/io/casehub/aml/AmlInvestigationResource.java
git commit -m "refactor: AmlInvestigationResource injects AmlInvestigationApplicationService

Removes direct NaiveAmlInvestigationService instantiation. CDI now resolves
the active implementation (WorkItemAmlInvestigationService at runtime).

Refs #15"
```

---

## Task 6: Update `AmlInvestigationResourceTest`

**Files:**
- Modify: `app/src/test/java/io/casehub/aml/AmlInvestigationResourceTest.java`

- [ ] **Step 1: Run existing test to see what fails**

```bash
mvn test -pl app -am -Dtest=AmlInvestigationResourceTest -Dsurefire.failIfNoSpecifiedTests=false
```
Expected: FAIL — response paths `transaction.id`, `entityResolution`, etc. no longer exist at root; they are now under `summary`.

- [ ] **Step 2: Update the test**

Replace the existing file with:

```java
package io.casehub.aml;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;
import static org.hamcrest.Matchers.notNullValue;

@QuarkusTest
class AmlInvestigationResourceTest {

    private static final String VALID_TX = """
            {
              "id": "TXN-001",
              "originAccountId": "ACC-A",
              "destinationAccountId": "ACC-B",
              "amount": 100000,
              "currency": "USD",
              "timestamp": "2024-03-15T10:00:00Z",
              "flagReason": "Structuring"
            }
            """;

    @Test
    void postInvestigation_validTransaction_returns200WithSummaryAndWorkItem() {
        given()
                .contentType(ContentType.JSON)
                .body(VALID_TX)
        .when()
                .post("/api/investigations")
        .then()
                .statusCode(200)
                .body("summary.transaction.id",   equalTo("TXN-001"))
                .body("summary.entityResolution", notNullValue())
                .body("summary.patternAnalysis",  notNullValue())
                .body("summary.osintScreening",   notNullValue())
                .body("summary.sarNarrative",     notNullValue())
                .body("complianceReviewTaskId",   notNullValue());
    }

    @Test
    void postInvestigation_malformedJson_returns400() {
        given()
                .contentType(ContentType.JSON)
                .body("not-valid-json")
        .when()
                .post("/api/investigations")
        .then()
                .statusCode(400);
    }
}
```

- [ ] **Step 3: Run tests — confirm they pass**

```bash
mvn test -pl app -am -Dtest=AmlInvestigationResourceTest -Dsurefire.failIfNoSpecifiedTests=false
```
Expected: 2 tests PASS.

- [ ] **Step 4: Run the full test suite**

```bash
mvn verify -pl api,app -am -Dsurefire.failIfNoSpecifiedTests=false
```
Expected: BUILD SUCCESS. All tests pass: `NaiveAmlInvestigationServiceTest` (4), `WorkItemAmlInvestigationServiceTest` (2), `AmlInvestigationResourceTest` (2).

- [ ] **Step 5: Commit**

```bash
git add app/src/test/java/io/casehub/aml/AmlInvestigationResourceTest.java
git commit -m "test: update AmlInvestigationResourceTest for Layer 2 response shape

Response now returns {summary: {...}, complianceReviewTaskId: '...'}.
Paths updated from root to summary.*. Added complianceReviewTaskId assertion.

Refs #15"
```

---

## Self-Review Checklist

- **Spec coverage:**
  - `AmlInvestigationResult` in `api/` ✅ Task 1
  - `AmlInvestigationApplicationService` interface in `app/` ✅ Task 2
  - `NaiveAmlInvestigationService` → `@DefaultBean` + returns result ✅ Task 3
  - `WorkItemAmlInvestigationService` → `@ApplicationScoped` displaces default ✅ Task 4
  - `AmlInvestigationResource` injects interface ✅ Task 5
  - REST contract: `summary.*` + `complianceReviewTaskId` ✅ Task 6
  - WorkItem fields: `candidateGroups`, `claimDeadline`, `callerRef` ✅ Task 4 test + Task 6 test
  - `complianceReviewTaskId = null` in naive ✅ Task 3 test

- **Type consistency:** `AmlInvestigationResult` defined in Task 1, used in Tasks 2–6. `AmlInvestigationApplicationService` defined in Task 2, used in Tasks 3–6. `WorkItem.id` is `UUID`; `.toString()` used in Task 4. All consistent.

- **No placeholders:** All steps contain complete code. WorkItemCreateRequest uses all 19 positional args.
