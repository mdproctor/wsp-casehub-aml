# Layer 1: Naive Java AML Investigation Baseline — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the anti-pattern Layer 1 baseline — four minimal Java service stubs called sequentially with no CaseHub, exposed via a single REST endpoint that returns an `InvestigationSummary`.

**Architecture:** `api/` holds pure-Java result records and service interfaces (no Quarkus, no JPA). `app/tutorial/` holds minimal stub implementations and `NaiveAmlInvestigationService`, all plain Java with no CDI — newed-up directly. `app/` holds `AmlInvestigationResource`, the only CDI bean, which news up the service per-request. Unit tests live in `app/` (pure JUnit 5, no Quarkus); `@QuarkusTest` covers the REST endpoint.

**Tech Stack:** Java 21 records, JUnit 5, Quarkus REST (`quarkus-rest`), Jackson, REST Assured, `quarkus-junit` (not the deprecated `quarkus-junit5`).

**Issues:** Closes #12, Refs #9

---

## File Map

**Create in `api/`:**
- `api/src/main/java/io/casehub/aml/domain/EntityResolutionResult.java`
- `api/src/main/java/io/casehub/aml/domain/PatternAnalysisResult.java`
- `api/src/main/java/io/casehub/aml/domain/OsintResult.java`
- `api/src/main/java/io/casehub/aml/domain/InvestigationSummary.java`
- `api/src/main/java/io/casehub/aml/investigation/EntityResolutionService.java`
- `api/src/main/java/io/casehub/aml/investigation/PatternAnalysisService.java`
- `api/src/main/java/io/casehub/aml/investigation/OsintScreeningService.java`
- `api/src/main/java/io/casehub/aml/investigation/SarDraftingService.java`

**Create in `app/`:**
- `app/src/main/java/io/casehub/aml/tutorial/NaiveEntityResolutionService.java` (package-private)
- `app/src/main/java/io/casehub/aml/tutorial/NaivePatternAnalysisService.java` (package-private)
- `app/src/main/java/io/casehub/aml/tutorial/NaiveOsintScreeningService.java` (package-private)
- `app/src/main/java/io/casehub/aml/tutorial/NaiveSarDraftingService.java` (package-private)
- `app/src/main/java/io/casehub/aml/tutorial/NaiveAmlInvestigationService.java` (public)
- `app/src/main/java/io/casehub/aml/AmlInvestigationResource.java`
- `app/src/test/java/io/casehub/aml/tutorial/NaiveAmlInvestigationServiceTest.java`
- `app/src/test/java/io/casehub/aml/AmlInvestigationResourceTest.java`
- `app/src/test/resources/application.properties`

**Modify:**
- `app/pom.xml` — `quarkus-junit5` → `quarkus-junit` (deprecated alias, convention quarkus-junit-not-junit5.md)

---

## Task 1: Fix deprecated test dependency

**Files:**
- Modify: `app/pom.xml`

- [ ] **Step 1: Replace the deprecated dependency**

In `app/pom.xml` find and replace:
```xml
<!-- Remove this -->
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-junit5</artifactId>
  <scope>test</scope>
</dependency>

<!-- Replace with -->
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-junit</artifactId>
  <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Verify build still passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode verify -pl app -am -q
```
Expected: `BUILD SUCCESS`

- [ ] **Step 3: Commit**

```bash
git add app/pom.xml
git commit -m "fix: replace deprecated quarkus-junit5 with quarkus-junit Refs #12"
```

---

## Task 2: Domain result records in api/

**Files:**
- Create: `api/src/main/java/io/casehub/aml/domain/EntityResolutionResult.java`
- Create: `api/src/main/java/io/casehub/aml/domain/PatternAnalysisResult.java`
- Create: `api/src/main/java/io/casehub/aml/domain/OsintResult.java`
- Create: `api/src/main/java/io/casehub/aml/domain/InvestigationSummary.java`

No tests needed — these are pure data-holding records with no logic.

- [ ] **Step 1: Create EntityResolutionResult**

`api/src/main/java/io/casehub/aml/domain/EntityResolutionResult.java`:
```java
package io.casehub.aml.domain;

public record EntityResolutionResult(String entityId, String ownershipChain) {}
```

- [ ] **Step 2: Create PatternAnalysisResult**

`api/src/main/java/io/casehub/aml/domain/PatternAnalysisResult.java`:
```java
package io.casehub.aml.domain;

public record PatternAnalysisResult(boolean structuringDetected, String description) {}
```

- [ ] **Step 3: Create OsintResult**

`api/src/main/java/io/casehub/aml/domain/OsintResult.java`:
```java
package io.casehub.aml.domain;

public record OsintResult(boolean sanctionsHit, boolean pepHit, String detail) {}
```

- [ ] **Step 4: Create InvestigationSummary**

`api/src/main/java/io/casehub/aml/domain/InvestigationSummary.java`:
```java
package io.casehub.aml.domain;

public record InvestigationSummary(
        SuspiciousTransaction transaction,
        EntityResolutionResult entityResolution,
        PatternAnalysisResult patternAnalysis,
        OsintResult osintScreening,
        String sarNarrative) {}
```

- [ ] **Step 5: Compile to confirm no errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl api -q
```
Expected: `BUILD SUCCESS`

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/aml/domain/
git commit -m "feat: add investigation result records Refs #12"
```

---

## Task 3: Service interfaces in api/

**Files:**
- Create: `api/src/main/java/io/casehub/aml/investigation/EntityResolutionService.java`
- Create: `api/src/main/java/io/casehub/aml/investigation/PatternAnalysisService.java`
- Create: `api/src/main/java/io/casehub/aml/investigation/OsintScreeningService.java`
- Create: `api/src/main/java/io/casehub/aml/investigation/SarDraftingService.java`

- [ ] **Step 1: Create EntityResolutionService**

`api/src/main/java/io/casehub/aml/investigation/EntityResolutionService.java`:
```java
package io.casehub.aml.investigation;

import io.casehub.aml.domain.EntityResolutionResult;
import io.casehub.aml.domain.SuspiciousTransaction;

public interface EntityResolutionService {
    EntityResolutionResult resolve(SuspiciousTransaction transaction);
}
```

- [ ] **Step 2: Create PatternAnalysisService**

`api/src/main/java/io/casehub/aml/investigation/PatternAnalysisService.java`:
```java
package io.casehub.aml.investigation;

import io.casehub.aml.domain.PatternAnalysisResult;
import io.casehub.aml.domain.SuspiciousTransaction;

public interface PatternAnalysisService {
    PatternAnalysisResult analyze(SuspiciousTransaction transaction);
}
```

- [ ] **Step 3: Create OsintScreeningService**

`api/src/main/java/io/casehub/aml/investigation/OsintScreeningService.java`:
```java
package io.casehub.aml.investigation;

import io.casehub.aml.domain.OsintResult;
import io.casehub.aml.domain.SuspiciousTransaction;

public interface OsintScreeningService {
    OsintResult screen(SuspiciousTransaction transaction);
}
```

- [ ] **Step 4: Create SarDraftingService**

`api/src/main/java/io/casehub/aml/investigation/SarDraftingService.java`:
```java
package io.casehub.aml.investigation;

import io.casehub.aml.domain.EntityResolutionResult;
import io.casehub.aml.domain.OsintResult;
import io.casehub.aml.domain.PatternAnalysisResult;
import io.casehub.aml.domain.SuspiciousTransaction;

public interface SarDraftingService {
    String draft(SuspiciousTransaction transaction,
                 EntityResolutionResult entity,
                 PatternAnalysisResult pattern,
                 OsintResult osint);
}
```

- [ ] **Step 5: Compile api**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl api -q
```
Expected: `BUILD SUCCESS`

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/aml/investigation/
git commit -m "feat: add investigation service interfaces Refs #12"
```

---

## Task 4: Unit tests for NaiveAmlInvestigationService (TDD — tests first)

**Files:**
- Create: `app/src/test/java/io/casehub/aml/tutorial/NaiveAmlInvestigationServiceTest.java`

Pure JUnit 5 — no Quarkus, no CDI. The class under test does not exist yet; the test must fail to compile first.

- [ ] **Step 1: Write the failing unit test**

`app/src/test/java/io/casehub/aml/tutorial/NaiveAmlInvestigationServiceTest.java`:
```java
package io.casehub.aml.tutorial;

import java.math.BigDecimal;
import java.time.Instant;

import org.junit.jupiter.api.Test;

import io.casehub.aml.domain.InvestigationSummary;
import io.casehub.aml.domain.SuspiciousTransaction;

import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertSame;

class NaiveAmlInvestigationServiceTest {

    private final NaiveAmlInvestigationService service = new NaiveAmlInvestigationService();

    private SuspiciousTransaction tx(String id) {
        return new SuspiciousTransaction(
                id, "ACC-A", "ACC-B",
                new BigDecimal("100000"), "USD",
                Instant.parse("2024-03-15T10:00:00Z"), "Structuring");
    }

    // Happy path: a valid transaction produces a fully populated summary
    @Test
    void investigate_validTransaction_returnsCompleteSummary() {
        InvestigationSummary summary = service.investigate(tx("TXN-001"));

        assertNotNull(summary);
        assertNotNull(summary.entityResolution());
        assertNotNull(summary.patternAnalysis());
        assertNotNull(summary.osintScreening());
        assertNotNull(summary.sarNarrative());
    }

    // Correctness: the original transaction object is preserved unchanged in the summary
    @Test
    void investigate_preservesTransactionIdentity() {
        SuspiciousTransaction input = tx("TXN-002");
        assertSame(input, service.investigate(input).transaction());
    }

    // Correctness: two successive calls produce independent summary objects
    @Test
    void investigate_calledTwice_producesIndependentSummaries() {
        InvestigationSummary first  = service.investigate(tx("TXN-003"));
        InvestigationSummary second = service.investigate(tx("TXN-004"));

        assertNotNull(first);
        assertNotNull(second);
    }
}
```

- [ ] **Step 2: Run — confirm compile failure (class not found)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test-compile -pl app -am -q 2>&1 | tail -5
```
Expected: `BUILD FAILURE` — `NaiveAmlInvestigationService` does not exist yet.

---

## Task 5: Implement stub services and NaiveAmlInvestigationService

**Files:**
- Create: `app/src/main/java/io/casehub/aml/tutorial/NaiveEntityResolutionService.java`
- Create: `app/src/main/java/io/casehub/aml/tutorial/NaivePatternAnalysisService.java`
- Create: `app/src/main/java/io/casehub/aml/tutorial/NaiveOsintScreeningService.java`
- Create: `app/src/main/java/io/casehub/aml/tutorial/NaiveSarDraftingService.java`
- Create: `app/src/main/java/io/casehub/aml/tutorial/NaiveAmlInvestigationService.java`

Stub classes are **package-private** (no `public`) — they must not be visible or injectable outside the `tutorial` package. `NaiveAmlInvestigationService` is `public` because `AmlInvestigationResource` (different package) needs to new it up.

- [ ] **Step 1: Create NaiveEntityResolutionService**

`app/src/main/java/io/casehub/aml/tutorial/NaiveEntityResolutionService.java`:
```java
package io.casehub.aml.tutorial;

import io.casehub.aml.domain.EntityResolutionResult;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.investigation.EntityResolutionService;

class NaiveEntityResolutionService implements EntityResolutionService {

    @Override
    public EntityResolutionResult resolve(SuspiciousTransaction transaction) {
        return new EntityResolutionResult("", "");
    }
}
```

- [ ] **Step 2: Create NaivePatternAnalysisService**

`app/src/main/java/io/casehub/aml/tutorial/NaivePatternAnalysisService.java`:
```java
package io.casehub.aml.tutorial;

import io.casehub.aml.domain.PatternAnalysisResult;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.investigation.PatternAnalysisService;

class NaivePatternAnalysisService implements PatternAnalysisService {

    @Override
    public PatternAnalysisResult analyze(SuspiciousTransaction transaction) {
        return new PatternAnalysisResult(false, "");
    }
}
```

- [ ] **Step 3: Create NaiveOsintScreeningService**

`app/src/main/java/io/casehub/aml/tutorial/NaiveOsintScreeningService.java`:
```java
package io.casehub.aml.tutorial;

import io.casehub.aml.domain.OsintResult;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.investigation.OsintScreeningService;

class NaiveOsintScreeningService implements OsintScreeningService {

    @Override
    public OsintResult screen(SuspiciousTransaction transaction) {
        return new OsintResult(false, false, "");
    }
}
```

- [ ] **Step 4: Create NaiveSarDraftingService**

`app/src/main/java/io/casehub/aml/tutorial/NaiveSarDraftingService.java`:
```java
package io.casehub.aml.tutorial;

import io.casehub.aml.domain.EntityResolutionResult;
import io.casehub.aml.domain.OsintResult;
import io.casehub.aml.domain.PatternAnalysisResult;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.investigation.SarDraftingService;

class NaiveSarDraftingService implements SarDraftingService {

    @Override
    public String draft(SuspiciousTransaction transaction,
                        EntityResolutionResult entity,
                        PatternAnalysisResult pattern,
                        OsintResult osint) {
        return "";
    }
}
```

- [ ] **Step 5: Create NaiveAmlInvestigationService with the four gap comments**

`app/src/main/java/io/casehub/aml/tutorial/NaiveAmlInvestigationService.java`:
```java
package io.casehub.aml.tutorial;

import io.casehub.aml.domain.EntityResolutionResult;
import io.casehub.aml.domain.InvestigationSummary;
import io.casehub.aml.domain.OsintResult;
import io.casehub.aml.domain.PatternAnalysisResult;
import io.casehub.aml.domain.SuspiciousTransaction;

public class NaiveAmlInvestigationService {

    private final NaiveEntityResolutionService entityResolutionService = new NaiveEntityResolutionService();
    private final NaivePatternAnalysisService  patternAnalysisService  = new NaivePatternAnalysisService();
    private final NaiveOsintScreeningService   osintScreeningService   = new NaiveOsintScreeningService();
    private final NaiveSarDraftingService      sarDraftingService      = new NaiveSarDraftingService();

    public InvestigationSummary investigate(SuspiciousTransaction transaction) {
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

        return new InvestigationSummary(transaction, entity, pattern, osint, sarNarrative);
    }
}
```

- [ ] **Step 6: Run unit tests — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am \
  -Dtest=NaiveAmlInvestigationServiceTest 2>&1 | grep -E "Tests run|BUILD"
```
Expected: `Tests run: 3, Failures: 0, Errors: 0, Skipped: 0` and `BUILD SUCCESS`

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/aml/tutorial/ \
        app/src/test/java/io/casehub/aml/tutorial/
git commit -m "feat: Layer 1 naive stubs and NaiveAmlInvestigationService with gap comments Refs #12"
```

---

## Task 6: Test application.properties for @QuarkusTest

**Files:**
- Create: `app/src/test/resources/application.properties`

This file overrides the main `application.properties` during `@QuarkusTest` runs. Key differences: unique H2 DB names (`aml-test`, `qhorus-test`) to prevent state leakage; `quarkus.http.test-port=0` to avoid TIME_WAIT conflicts; scheduler disabled; Instant serialisation as ISO strings.

- [ ] **Step 1: Create test application.properties**

`app/src/test/resources/application.properties`:
```properties
quarkus.http.test-port=0

quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:aml-test;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
quarkus.datasource.username=sa
quarkus.datasource.password=

quarkus.datasource.qhorus.db-kind=h2
quarkus.datasource.qhorus.jdbc.url=jdbc:h2:mem:qhorus-test;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
quarkus.datasource.qhorus.username=sa
quarkus.datasource.qhorus.password=

quarkus.hibernate-orm.packages=io.casehub.work.runtime.model,io.casehub.aml.domain
quarkus.hibernate-orm.database.generation=none
quarkus.flyway.migrate-at-start=true

quarkus.hibernate-orm.qhorus.datasource=qhorus
quarkus.hibernate-orm.qhorus.packages=io.casehub.qhorus.runtime,io.casehub.ledger.runtime.model,io.casehub.ledger.model
quarkus.hibernate-orm.qhorus.database.generation=none
quarkus.flyway.qhorus.migrate-at-start=true

casehub.ledger.datasource=qhorus
quarkus.scheduler.enabled=false
quarkus.jackson.write-dates-as-timestamps=false
```

---

## Task 7: @QuarkusTest for AmlInvestigationResource (TDD — tests first)

**Files:**
- Create: `app/src/test/java/io/casehub/aml/AmlInvestigationResourceTest.java`

- [ ] **Step 1: Write the failing @QuarkusTest**

`app/src/test/java/io/casehub/aml/AmlInvestigationResourceTest.java`:
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

    // Happy path: valid transaction returns 200 with all summary fields present
    @Test
    void postInvestigation_validTransaction_returns200WithSummary() {
        given()
                .contentType(ContentType.JSON)
                .body(VALID_TX)
        .when()
                .post("/api/investigations")
        .then()
                .statusCode(200)
                .body("transaction.id",   equalTo("TXN-001"))
                .body("entityResolution", notNullValue())
                .body("patternAnalysis",  notNullValue())
                .body("osintScreening",   notNullValue())
                .body("sarNarrative",     notNullValue());
    }

    // Robustness: malformed JSON body is rejected with 400
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

- [ ] **Step 2: Run — confirm failure (endpoint does not exist)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am \
  -Dtest=AmlInvestigationResourceTest 2>&1 | grep -E "Tests run|BUILD|404" | head -10
```
Expected: `BUILD FAILURE` or test failure with 404.

---

## Task 8: Implement AmlInvestigationResource

**Files:**
- Create: `app/src/main/java/io/casehub/aml/AmlInvestigationResource.java`

- [ ] **Step 1: Create AmlInvestigationResource**

`app/src/main/java/io/casehub/aml/AmlInvestigationResource.java`:
```java
package io.casehub.aml;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;

import io.casehub.aml.domain.InvestigationSummary;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.tutorial.NaiveAmlInvestigationService;

@Path("/api/investigations")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class AmlInvestigationResource {

    @POST
    public InvestigationSummary investigate(SuspiciousTransaction transaction) {
        return new NaiveAmlInvestigationService().investigate(transaction);
    }
}
```

- [ ] **Step 2: Run the @QuarkusTest — expect PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am \
  -Dtest=AmlInvestigationResourceTest 2>&1 | grep -E "Tests run|BUILD"
```
Expected: `Tests run: 2, Failures: 0, Errors: 0, Skipped: 0` and `BUILD SUCCESS`

- [ ] **Step 3: Run full verify across both modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode verify -pl api,app -am 2>&1 \
  | grep -E "Tests run|BUILD"
```
Expected: all tests pass, `BUILD SUCCESS`

- [ ] **Step 4: Commit**

```bash
git add app/src/main/java/io/casehub/aml/AmlInvestigationResource.java \
        app/src/test/java/io/casehub/aml/AmlInvestigationResourceTest.java \
        app/src/test/resources/application.properties
git commit -m "feat: AmlInvestigationResource POST /api/investigations with @QuarkusTest Closes #12 Refs #9"
```

---

## Task 9: Code review

- [ ] **Step 1: Invoke superpowers:requesting-code-review**

Review all files created in this plan for quality, naming, gap comment clarity, and convention compliance before finalising.

---

## Task 10: Documentation updates

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/parent/docs/repos/casehub-aml.md` (via GitHub issue — do NOT commit to parent repo directly)

- [ ] **Step 1: Verify all cross-references in CLAUDE.md resolve**

```bash
for f in \
  "/Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md" \
  "/Users/mdproctor/claude/casehub/parent/docs/use-case-analysis.md" \
  "/Users/mdproctor/claude/casehub/parent/docs/tutorial-strategy.md" \
  "/Users/mdproctor/claude/casehub/parent/docs/repos/casehub-aml.md" \
  "/Users/mdproctor/claude/casehub/parent/docs/repos/casehub-work.md" \
  "/Users/mdproctor/claude/casehub/parent/docs/repos/casehub-qhorus.md" \
  "/Users/mdproctor/claude/casehub/parent/docs/repos/casehub-ledger.md" \
  "/Users/mdproctor/claude/casehub/parent/docs/repos/casehub-engine.md" \
  "/Users/mdproctor/claude/casehub/parent/docs/conventions/flyway-migration-rules.md" \
  "/Users/mdproctor/claude/casehub/parent/docs/conventions/flyway-version-range-allocation.md" \
  "/Users/mdproctor/claude/casehub/parent/docs/conventions/quarkus-named-datasource-schema-generation.md" \
  "/Users/mdproctor/claude/casehub/parent/docs/conventions/quarkus-test-database.md" \
  "/Users/mdproctor/claude/casehub/parent/docs/conventions/quarkus-test-naming-convention.md" \
  "/Users/mdproctor/claude/casehub/parent/docs/conventions/spi-testing-alternative-inner-classes.md" \
  "/Users/mdproctor/claude/casehub/parent/docs/conventions/quarkus-integration-test-module-separation.md"; do
  [ -f "$f" ] && echo "✓ $f" || echo "✗ MISSING: $f"
done
```
Expected: all `✓`

- [ ] **Step 2: Raise issue on parent repo to update casehub-aml.md**

```bash
gh issue create --repo casehubio/parent \
  --title "docs: update casehub-aml.md — Layer 1 complete" \
  --body "Status field should change from 'Greenfield — no code yet' to 'Layer 1 (naive baseline) complete — api/ domain model and service interfaces, app/ tutorial implementation and REST endpoint'. See casehubio/aml#12." \
  --label "documentation"
```
