# AML Workbench Backend API — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the REST API surface that the AML workbench UI consumes — investigation list/detail/flow/findings/gates, metrics, suspend/resume, and simulation.

**Architecture:** CQRS-lite read model (`InvestigationSummaryView`) for the investigation list, thin REST resources delegating to `CaseHubRuntime` for case detail queries, denormalised metrics from the summary table, and a simulation API using CDI `@DefaultBean` displacement for deterministic specialist workers.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate (H2 test, PostgreSQL prod), Flyway, REST Assured

**Spec:** `specs/2026-06-30-aml-workbench-ui-design.md`

**Issue:** To be filed before first commit (casehubio/aml — epic for the workbench)

## Global Constraints

- **Flyway versions:** AML owns V1–V999 and V2000+. Next available: V2014. Reserve in `.meta` at branch start.
- **JPA entities:** Never in `api/` — only in `app/`. Entities go in `app/src/main/java/io/casehub/aml/`.
- **Test database:** H2 with `MODE=PostgreSQL;DB_CLOSE_DELAY=-1`. Hash chain disabled in tests.
- **Persistence units:** Default PU for work + AML entities. Qhorus PU for qhorus + ledger + trust entities.
- **CDI exclusions in test:** `TenantScopedPrincipal`, `JpaPlanItemStore`, `ExpiryTimerJob`, `ClaimDeadlineTimerJob`, `QhorusInboundCurrentPrincipal` (test only). See CLAUDE.md §Investigation @QuarkusTest conventions.
- **Maven scoping:** Always `-pl app -am`. With `-Dtest=ClassName`, add `-Dsurefire.failIfNoSpecifiedTests=false`.
- **Ledger subject isolation:** AML LedgerEntry subclasses use `UUID.nameUUIDFromBytes("aml-<concern>:" + caseId)` as subjectId.
- **TDD:** Use `superpowers:test-driven-development` before implementing. Use `java-dev` skill for all Java code.
- **Code review:** Use `superpowers:requesting-code-review` before committing.
- **Doc sync:** Use `implementation-doc-sync` after committing.
- **IntelliJ first:** Use IntelliJ MCP for all class/symbol lookups. Never use bash grep/find for code search.
- **Platform protocols:** Consult `../parent/docs/PLATFORM.md` and `../garden/docs/protocols/` before and during implementation. Final coherence review before completion.
- **Design quality:** Prefer fixing the design over protecting callers. Breaking changes cost nothing. No backwards-compat shims.

---

## File Structure

### New Files

```
app/src/main/java/io/casehub/aml/query/
├── InvestigationSummaryView.java          # JPA entity — denormalised read model
├── InvestigationSummaryRepository.java    # Panache repository
├── InvestigationSummaryService.java       # Write path: create + update summary rows
└── InvestigationSummaryObserver.java      # @ObservesAsync CaseLifecycleEvent → status updates

app/src/main/java/io/casehub/aml/engine/
├── AmlInvestigationQueryResource.java     # REST: list, prior-context, flow, findings, gates
├── AmlInvestigationFlowService.java       # Reconstructs DAG from CaseHubRuntime.eventLog()
├── AmlInvestigationFindingsService.java   # Assembles specialist outcomes from CaseContext
├── AmlInvestigationGatesService.java      # Gates from WorkItem data
└── AmlInvestigationPriorContextService.java # Prior context from CaseContext

app/src/main/java/io/casehub/aml/metrics/
├── AmlMetricsResource.java                # REST: throughput, trust-scores, gates
└── AmlMetricsService.java                 # Aggregation queries on InvestigationSummaryView

app/src/main/java/io/casehub/aml/simulation/
├── AmlSimulationResource.java             # REST: seed, investigate, reset
├── AmlSimulationService.java              # Orchestrates scenario execution
└── AmlScenarioTemplate.java               # Scenario definitions (enum or sealed)

app/src/main/resources/db/aml-query/migration/
└── V2014__investigation_summary_view.sql  # DDL for summary table

app/src/test/java/io/casehub/aml/query/
├── InvestigationSummaryViewTest.java      # Entity unit test
├── InvestigationSummaryServiceTest.java   # Service integration test
└── InvestigationSummaryObserverTest.java  # Observer integration test

app/src/test/java/io/casehub/aml/engine/
├── AmlInvestigationQueryResourceTest.java # REST endpoint tests
├── AmlInvestigationFlowServiceTest.java   # Flow reconstruction tests
├── AmlInvestigationFindingsServiceTest.java
├── AmlInvestigationGatesServiceTest.java
└── AmlInvestigationPriorContextServiceTest.java

app/src/test/java/io/casehub/aml/metrics/
├── AmlMetricsResourceTest.java
└── AmlMetricsServiceTest.java

app/src/test/java/io/casehub/aml/simulation/
├── AmlSimulationResourceTest.java
└── AmlSimulationServiceTest.java
```

### Modified Files

```
app/src/main/java/io/casehub/aml/engine/AmlEngineCoordinator.java
  — Capture prior context and specialist results in summary view on investigation start

app/src/main/java/io/casehub/aml/ComplianceReviewLifecycle.java
  — Add .scope("casehubio/aml/oversight") to openReview() WorkItem creation (#88)

app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java
  — Add suspend/resume endpoints

app/src/main/resources/application.properties
  — Add Flyway location for db/aml-query/migration
  — Add Quinoa config (Plan 2)

app/src/test/resources/application.properties
  — Add Flyway location for db/aml-query/migration

app/pom.xml
  — No new dependencies needed (all foundation deps already present)
```

---

### Task 1: InvestigationSummaryView entity + Flyway migration + repository

**Files:**
- Create: `app/src/main/java/io/casehub/aml/query/InvestigationSummaryView.java`
- Create: `app/src/main/java/io/casehub/aml/query/InvestigationSummaryRepository.java`
- Create: `app/src/main/resources/db/aml-query/migration/V2014__investigation_summary_view.sql`
- Modify: `app/src/main/resources/application.properties` — add Flyway location
- Modify: `app/src/test/resources/application.properties` — add Flyway location
- Test: `app/src/test/java/io/casehub/aml/query/InvestigationSummaryViewTest.java`

**Interfaces:**
- Consumes: Nothing — this is the foundational read model
- Produces: `InvestigationSummaryView` JPA entity with fields: `id` (UUID PK), `caseId` (UUID unique), `status` (String), `outcomeType` (String nullable), `transactionId` (String), `originAccount` (String), `destinationAccount` (String), `amount` (BigDecimal), `currency` (String), `flagReason` (String), `createdAt` (Instant). `InvestigationSummaryRepository extends PanacheRepositoryBase<InvestigationSummaryView, UUID>` with query methods: `findByCaseId(UUID)`, `listByStatus(String, Page)`, `listAll(Page)`.

**Context for implementer:**
- This entity lives on the **default** persistence unit (same as WorkItem), not the qhorus PU.
- The `InvestigationSummaryView` table is a CQRS-lite read model — populated by an observer (Task 2), queried by the list endpoint (Task 3).
- Use `@Table(name = "aml_investigation_summary")`. The entity is NOT a ledger entry — it's a denormalised query projection.
- Register the package in `quarkus.hibernate-orm.packages` in both main and test `application.properties`.
- Create a new Flyway location `db/aml-query/migration` and add it to `quarkus.flyway.locations`.
- The migration SQL must be H2-compatible (ANSI types only). Use `VARCHAR` not `TEXT`, `TIMESTAMP WITH TIME ZONE` for instants, `DECIMAL(19,4)` for amounts.
- V2014 is reserved for this migration. Verify no other branch has taken it.

- [ ] **Step 1: Write the Flyway migration**

Create `app/src/main/resources/db/aml-query/migration/V2014__investigation_summary_view.sql`:

```sql
CREATE TABLE aml_investigation_summary (
    id              UUID            NOT NULL,
    case_id         UUID            NOT NULL,
    status          VARCHAR(32)     NOT NULL,
    outcome_type    VARCHAR(64),
    transaction_id  VARCHAR(128)    NOT NULL,
    origin_account  VARCHAR(128)    NOT NULL,
    dest_account    VARCHAR(128)    NOT NULL,
    amount          DECIMAL(19,4)   NOT NULL,
    currency        VARCHAR(8)      NOT NULL,
    flag_reason     VARCHAR(128)    NOT NULL,
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL,

    CONSTRAINT pk_aml_investigation_summary PRIMARY KEY (id),
    CONSTRAINT uq_aml_investigation_summary_case UNIQUE (case_id)
);

CREATE INDEX idx_aml_investigation_summary_status ON aml_investigation_summary (status);
CREATE INDEX idx_aml_investigation_summary_created ON aml_investigation_summary (created_at);
```

- [ ] **Step 2: Add Flyway location to application.properties**

In both `app/src/main/resources/application.properties` and `app/src/test/resources/application.properties`, add `classpath:db/aml-query/migration` to the existing `quarkus.flyway.locations` value.

Also add `io.casehub.aml.query` to `quarkus.hibernate-orm.packages` in both files.

- [ ] **Step 3: Write the JPA entity**

Create `app/src/main/java/io/casehub/aml/query/InvestigationSummaryView.java`:

```java
package io.casehub.aml.query;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "aml_investigation_summary")
public class InvestigationSummaryView {

    @Id
    private UUID id;

    @Column(name = "case_id", nullable = false, unique = true)
    private UUID caseId;

    @Column(nullable = false, length = 32)
    private String status;

    @Column(name = "outcome_type", length = 64)
    private String outcomeType;

    @Column(name = "transaction_id", nullable = false, length = 128)
    private String transactionId;

    @Column(name = "origin_account", nullable = false, length = 128)
    private String originAccount;

    @Column(name = "dest_account", nullable = false, length = 128)
    private String destinationAccount;

    @Column(nullable = false, precision = 19, scale = 4)
    private BigDecimal amount;

    @Column(nullable = false, length = 8)
    private String currency;

    @Column(name = "flag_reason", nullable = false, length = 128)
    private String flagReason;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    protected InvestigationSummaryView() {}

    public InvestigationSummaryView(UUID caseId, String transactionId,
            String originAccount, String destinationAccount,
            BigDecimal amount, String currency, String flagReason) {
        this.id = UUID.randomUUID();
        this.caseId = caseId;
        this.transactionId = transactionId;
        this.originAccount = originAccount;
        this.destinationAccount = destinationAccount;
        this.amount = amount;
        this.currency = currency;
        this.flagReason = flagReason;
        this.status = "IN_PROGRESS";
        this.createdAt = Instant.now();
    }

    // Getters + status/outcome setters only (no full setters — write path is controlled)
    public UUID id() { return id; }
    public UUID caseId() { return caseId; }
    public String status() { return status; }
    public String outcomeType() { return outcomeType; }
    public String transactionId() { return transactionId; }
    public String originAccount() { return originAccount; }
    public String destinationAccount() { return destinationAccount; }
    public BigDecimal amount() { return amount; }
    public String currency() { return currency; }
    public String flagReason() { return flagReason; }
    public Instant createdAt() { return createdAt; }

    public void updateStatus(String status) { this.status = status; }
    public void updateOutcomeType(String outcomeType) { this.outcomeType = outcomeType; }
}
```

- [ ] **Step 4: Write the repository**

Create `app/src/main/java/io/casehub/aml/query/InvestigationSummaryRepository.java`:

```java
package io.casehub.aml.query;

import io.quarkus.hibernate.orm.panache.PanacheRepositoryBase;
import io.quarkus.panache.common.Page;
import io.quarkus.panache.common.Sort;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

@ApplicationScoped
public class InvestigationSummaryRepository
        implements PanacheRepositoryBase<InvestigationSummaryView, UUID> {

    public Optional<InvestigationSummaryView> findByCaseId(UUID caseId) {
        return find("caseId", caseId).firstResultOptional();
    }

    public List<InvestigationSummaryView> listByStatus(String status, Page page) {
        return find("status", Sort.descending("createdAt"), status).page(page).list();
    }

    public List<InvestigationSummaryView> listAll(Page page) {
        return findAll(Sort.descending("createdAt")).page(page).list();
    }

    public long countByStatus(String status) {
        return count("status", status);
    }
}
```

- [ ] **Step 5: Write the failing test**

Create `app/src/test/java/io/casehub/aml/query/InvestigationSummaryViewTest.java` — a `@QuarkusTest` that verifies entity persistence, query by status, and unique constraint on caseId. Follow existing test patterns in the codebase (see `AmlErasureServiceTest.java` for the QuarkusTest setup pattern with CDI exclusions).

- [ ] **Step 6: Run tests to verify they fail, then pass after entity + migration are in place**

Run: `mvn test -pl app -am -Dtest=InvestigationSummaryViewTest -Dsurefire.failIfNoSpecifiedTests=false`

- [ ] **Step 7: Commit**

```
feat(#WB): InvestigationSummaryView entity + V2014 migration + repository

Refs #<workbench-epic-issue>
```

---

### Task 2: InvestigationSummaryService + CaseLifecycleEvent observer

**Files:**
- Create: `app/src/main/java/io/casehub/aml/query/InvestigationSummaryService.java`
- Create: `app/src/main/java/io/casehub/aml/query/InvestigationSummaryObserver.java`
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlEngineCoordinator.java` — call service on investigation start
- Test: `app/src/test/java/io/casehub/aml/query/InvestigationSummaryServiceTest.java`
- Test: `app/src/test/java/io/casehub/aml/query/InvestigationSummaryObserverTest.java`

**Interfaces:**
- Consumes: `InvestigationSummaryRepository` (Task 1), `CaseLifecycleEvent` (from `io.casehub.engine.common.spi.event`), `AmlInvestigationOutcomeService.resolveOutcome(UUID)` (existing in `io.casehub.aml.engine`)
- Produces: `InvestigationSummaryService.createSummary(UUID caseId, SuspiciousTransaction txn)` — called by coordinator on investigation start. `InvestigationSummaryObserver` — CDI `@ObservesAsync CaseLifecycleEvent` that updates status and resolves outcomeType on COMPLETED.

**Context for implementer:**
- `CaseLifecycleEvent` is a record in `io.casehub.engine.common.spi.event` with fields: `caseId`, `tenancyId`, `commandType`, `eventType`, `caseStatus`, `actorId`, `actorRole`, `traceId`.
- The observer fires on ALL case lifecycle transitions. Filter by `eventType` containing status changes: look for `caseStatus` field being non-null.
- When `caseStatus` is `"COMPLETED"`, call `AmlInvestigationOutcomeService.resolveOutcome(caseId)` to get the `InvestigationOutcome`, then write `outcome.type()` to the summary row's `outcomeType` column.
- For `FAILED`, `CANCELLED`, `SUSPENDED` status transitions, update the status but leave `outcomeType` null.
- The observer MUST use `@Transactional(TxType.REQUIRES_NEW)` — it runs on a CDI async thread, not in the engine's transaction.
- Follow the double-try/catch pattern established by `AmlTrustRoutingObserver` and `AmlWorkItemLifecycleObserver` for resilience.
- The `createSummary()` call in `AmlEngineCoordinator` should happen AFTER the caseId is generated but BEFORE the investigation starts, so the summary row exists for the observer to update.

- [ ] **Step 1: Write InvestigationSummaryService**

```java
package io.casehub.aml.query;

import io.casehub.aml.domain.SuspiciousTransaction;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.util.UUID;

@ApplicationScoped
public class InvestigationSummaryService {

    @Inject InvestigationSummaryRepository repository;

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void createSummary(UUID caseId, SuspiciousTransaction txn) {
        var summary = new InvestigationSummaryView(
            caseId, txn.id(), txn.originAccountId(), txn.destinationAccountId(),
            txn.amount(), txn.currency(), txn.flagReason());
        repository.persist(summary);
    }

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void updateStatus(UUID caseId, String status) {
        repository.findByCaseId(caseId)
            .ifPresent(summary -> summary.updateStatus(status));
    }

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    public void updateOutcome(UUID caseId, String outcomeType) {
        repository.findByCaseId(caseId)
            .ifPresent(summary -> summary.updateOutcomeType(outcomeType));
    }
}
```

- [ ] **Step 2: Write InvestigationSummaryObserver**

```java
package io.casehub.aml.query;

import io.casehub.aml.engine.AmlInvestigationOutcomeService;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class InvestigationSummaryObserver {

    private static final Logger LOG = Logger.getLogger(InvestigationSummaryObserver.class);

    @Inject InvestigationSummaryService summaryService;
    @Inject AmlInvestigationOutcomeService outcomeService;

    void onCaseLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        if (event.caseStatus() == null) return;

        try {
            summaryService.updateStatus(event.caseId(), event.caseStatus());

            if ("COMPLETED".equals(event.caseStatus())) {
                try {
                    var outcome = outcomeService.resolveOutcome(event.caseId());
                    if (outcome != null) {
                        summaryService.updateOutcome(event.caseId(), outcome.type());
                    }
                } catch (Exception e) {
                    LOG.warnf("Failed to resolve outcome for case %s: %s",
                        event.caseId(), e.getMessage());
                }
            }
        } catch (Exception e) {
            LOG.errorf(e, "Failed to update investigation summary for case %s", event.caseId());
        }
    }
}
```

- [ ] **Step 3: Modify AmlEngineCoordinator to call createSummary on investigation start**

Find the method in `AmlEngineCoordinator` that generates the `caseId` and starts the investigation. Inject `InvestigationSummaryService` and call `createSummary(caseId, transaction)` after caseId generation, before engine start. Use IntelliJ MCP to navigate to the class and understand its current structure.

- [ ] **Step 4: Write failing tests, then implement until green**

Tests should cover:
- `createSummary` persists a row with status=IN_PROGRESS
- Observer updates status on CaseLifecycleEvent with non-null caseStatus
- Observer resolves outcomeType on COMPLETED
- Observer ignores events with null caseStatus
- Observer handles missing summary gracefully (case not yet created)

- [ ] **Step 5: Commit**

---

### Task 3: Investigation list endpoint

**Files:**
- Create: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationQueryResource.java`
- Test: `app/src/test/java/io/casehub/aml/engine/AmlInvestigationQueryResourceTest.java`

**Interfaces:**
- Consumes: `InvestigationSummaryRepository` (Task 1)
- Produces: `GET /api/investigations` — returns paginated list of `InvestigationSummaryResponse` records. Query params: `status` (optional filter), `page` (default 0), `pageSize` (default 25).

**Context for implementer:**
- Response record: `InvestigationSummaryResponse(UUID caseId, String status, String outcomeType, String transactionId, String originAccount, String destinationAccount, BigDecimal amount, String currency, String flagReason, Instant createdAt)`.
- Map from `InvestigationSummaryView` entity fields. Return as `PagedResponse<InvestigationSummaryResponse>` with `items`, `total`, `page`, `pageSize`.
- Path is `/api/investigations` (not `/api/layer9/investigations`) — this is the query surface, separate from the Layer 9 operational endpoint.
- The existing `AmlInvestigationResource` at `/api/investigations` only has a POST. Adding a GET to the same path is fine.
- Actually — check if adding GET to the existing `AmlInvestigationResource` is cleaner than a new resource class. If the existing resource is thin enough, add the GET there. Otherwise create a new query resource.

- [ ] **Step 1: Create the response records**

```java
public record InvestigationSummaryResponse(
    UUID caseId, String status, String outcomeType,
    String transactionId, String originAccount, String destinationAccount,
    BigDecimal amount, String currency, String flagReason, Instant createdAt) {}

public record PagedResponse<T>(List<T> items, long total, int page, int pageSize) {}
```

- [ ] **Step 2: Write failing REST test**

Test: `GET /api/investigations` returns 200 with empty list. `GET /api/investigations?status=IN_PROGRESS` filters correctly. Pagination works.

- [ ] **Step 3: Implement the endpoint**

- [ ] **Step 4: Run tests, verify green, commit**

---

### Task 4: Prior context endpoint

**Files:**
- Create: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationPriorContextService.java`
- Add endpoint to: `AmlInvestigationQueryResource.java` (Task 3)
- Test: `app/src/test/java/io/casehub/aml/engine/AmlInvestigationPriorContextServiceTest.java`

**Interfaces:**
- Consumes: `CaseHubRuntime.query(caseId, "priorEntityContext")` (from `io.casehub.api.engine.CaseHubRuntime`)
- Produces: `GET /api/investigations/{caseId}/prior-context` — returns the `AmlPriorContext.toContextMap()` structure.

**Context for implementer:**
- `CaseHubRuntime` is an interface at `io.casehub.api.engine.CaseHubRuntime`. Its `query(UUID caseId, String path)` returns `CompletionStage<Object>`. Call with path `"priorEntityContext"`.
- The returned object is the Map written by `AmlPriorContext.toContextMap()` at investigation start. It contains: `hasHistory` (boolean), `knownHighRisk` (boolean), `entityRiskCount`, `networkCount`, `patternCount`, `facts[]`.
- If the case doesn't exist or has no prior context, return 404.
- The `query()` method returns a `CompletionStage` — use `.toCompletableFuture().join()` in the synchronous REST handler, or use `@Blocking` + Uni.

- [ ] Steps: Write failing test → implement service + endpoint → verify green → commit

---

### Task 5: Investigation flow endpoint

**Files:**
- Create: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationFlowService.java`
- Add endpoint to: `AmlInvestigationQueryResource.java`
- Test: `app/src/test/java/io/casehub/aml/engine/AmlInvestigationFlowServiceTest.java`

**Interfaces:**
- Consumes: `CaseHubRuntime.eventLog(caseId, Set.of(WORKER_SCHEDULED, WORKER_EXECUTION_COMPLETED, WORKER_EXECUTION_FAILED))`, `AmlTrustRoutingAttestation` entries via `AmlTrustAttestationRepository`
- Produces: `GET /api/investigations/{caseId}/flow` → `InvestigationFlowResponse`

**Context for implementer:**
- `CaseEventLogRecord` is a record with: `eventType` (CaseHubEventType enum), `streamType` (EventStreamType enum), `timestamp` (Instant), `payload` (JsonNode), `metadata` (JsonNode).
- Use IntelliJ MCP to find `CaseHubEventType` enum values — look for `WORKER_SCHEDULED`, `WORKER_EXECUTION_COMPLETED`, `WORKER_EXECUTION_FAILED`.
- **Parallel detection algorithm:** Two consecutive `WORKER_SCHEDULED` events in the list with no intervening `WORKER_EXECUTION_COMPLETED` are parallel. Group them.
- Build response as `{ nodes: [...], edges: [...], parallelGroups: [...] }`.
- Each node: `{ capabilityTag, workerId, trustScoreAtRouting, status (scheduled/completed/failed), timestamp }`.
- Trust scores come from `AmlTrustRoutingAttestation` entries queried by a caseId-derived subject UUID.
- Worker name / capability tag is in the event's `payload` or `metadata` JsonNode — use IntelliJ to examine `CaseEventLogRecord` payload structure by looking at where events are created in the engine.

- [ ] Steps: Write failing test → implement flow reconstruction → verify green → commit

---

### Task 6: Specialist findings endpoint

**Files:**
- Create: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationFindingsService.java`
- Add endpoint to: `AmlInvestigationQueryResource.java`
- Test: `app/src/test/java/io/casehub/aml/engine/AmlInvestigationFindingsServiceTest.java`

**Interfaces:**
- Consumes: `CaseHubRuntime.query(caseId, key)` for keys: `"entityResolution"`, `"patternAnalysis"`, `"osintScreening"`, `"sarDraft"`
- Produces: `GET /api/investigations/{caseId}/findings` → `InvestigationFindingsResponse`

**Context for implementer:**
- Each `query()` call returns a `CompletionStage<Object>`. The returned object is a Map written by the worker function.
- Use IntelliJ MCP to find the worker functions in `AmlInvestigationCaseDescriptor` — verify the exact context keys each worker writes to.
- Wrap each result in a `SpecialistFindingResponse` with `status` ("COMPLETED" / "DECLINED" / "FAILED"), `result` (the data map), and optionally `agentId`, `capability`, `reason` for declined/failed.
- A null query result means the specialist hasn't executed yet — return `{ status: "PENDING" }`.

- [ ] Steps: Write failing test → implement → verify green → commit

---

### Task 7: Gates endpoint

**Files:**
- Create: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationGatesService.java`
- Add endpoint to: `AmlInvestigationQueryResource.java`
- Test: `app/src/test/java/io/casehub/aml/engine/AmlInvestigationGatesServiceTest.java`

**Interfaces:**
- Consumes: WorkItem query (by callerRef pattern `case:{caseId}/gate:*`), `AmlActionType.fromActionType()` for gate policy derivation
- Produces: `GET /api/investigations/{caseId}/gates` → `InvestigationGatesResponse`

**Context for implementer:**
- Gates ARE WorkItems created by `ActionGateWorkItemHandler` with `callerRef = GateCallerRef.encode(caseId, gateId)` producing format `case:{caseId}/gate:{gateId}`.
- Use IntelliJ MCP to find `ActionGateWorkItemHandler` and verify the payload structure — it stores `actionType`, `reversible`, `description`, `context` in the WorkItem's JSON payload.
- Query WorkItems by callerRef prefix `case:{caseId}/gate:`. This requires access to `WorkItemService` or direct JPA query on the WorkItem table.
- Derive `gatePolicy` at query time: `AmlActionType.fromActionType(actionTypeString).map(AmlActionType::gatePolicy)`.
- Response per gate: `{ workItemId, actionType, gatePolicy, reversible, description, candidateGroups, status, approvedBy, approvedAt, expiresAt }`.

- [ ] Steps: Write failing test → implement → verify green → commit

---

### Task 8: Suspend/resume endpoints

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java` — add suspend + resume
- Test: extend `app/src/test/java/io/casehub/aml/engine/AmlLayer9ResourceTest.java`

**Interfaces:**
- Consumes: `CaseHubRuntime.suspendCase(UUID)`, `CaseHubRuntime.resumeCase(UUID)`
- Produces: `POST /api/layer9/investigations/{caseId}/suspend` (204), `POST /api/layer9/investigations/{caseId}/resume` (204)

**Context for implementer:**
- Both methods exist on `CaseHubRuntime` (verified via IntelliJ MCP). They are void methods — fire and forget.
- Return 204 No Content on success. Return 404 if caseId doesn't exist (check via `InvestigationSummaryRepository.findByCaseId()` or catch engine exception).
- These are thin REST endpoints — one line of delegation each.

- [ ] Steps: Write failing test → implement → verify green → commit

---

### Task 9: Metrics endpoints

**Files:**
- Create: `app/src/main/java/io/casehub/aml/metrics/AmlMetricsService.java`
- Create: `app/src/main/java/io/casehub/aml/metrics/AmlMetricsResource.java`
- Test: `app/src/test/java/io/casehub/aml/metrics/AmlMetricsResourceTest.java`

**Interfaces:**
- Consumes: `InvestigationSummaryRepository` (aggregation queries), `TrustScoreSource` (current trust scores), WorkItem queries (gate metrics)
- Produces:
  - `GET /api/metrics/throughput` → `ThroughputMetrics`
  - `GET /api/metrics/trust-scores` → `TrustScoreMetrics`
  - `GET /api/metrics/gates` → `GateMetrics`

**Context for implementer:**
- **Throughput:** Count investigations by status from `InvestigationSummaryView`. Group by `flagReason` and `outcomeType`. Time bucketing via native SQL `DATE_TRUNC` (H2-compatible in PostgreSQL mode). Query params: `since` (ISO date), `granularity` (day/hour).
- **Trust scores:** Use `TrustScoreSource` (from `io.casehub.ledger.api.spi`) — `MaterializedTrustScoreSource` is `@DefaultBean` (reads DB fresh per call). Call `capabilityScore(agentId, capabilityTag)` for each known agent/capability pair. Agent IDs come from the trust routing policy YAML. Use IntelliJ MCP to find the policy file and extract agent IDs.
- **Gates:** Query WorkItems by callerRef pattern `case:*/gate:*`. Group by action type, count approved/rejected/pending. Calculate average approval time from `createdAt` to `completedAt`.

- [ ] Steps: Write failing tests for each endpoint → implement service + resource → verify green → commit

---

### Task 10: ComplianceReviewLifecycle scope update

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/ComplianceReviewLifecycle.java`
- Modify test: `app/src/test/java/io/casehub/aml/ComplianceReviewLifecycleTest.java`

**Interfaces:**
- Consumes: None new
- Produces: WorkItems created by `openReview()` now include `.scope("casehubio/aml/oversight")`

**Context for implementer:**
- Find the `openReview()` method in `ComplianceReviewLifecycle.java` using IntelliJ MCP.
- Add `.scope("casehubio/aml/oversight")` to the WorkItem builder chain. Use IntelliJ to verify the WorkItem builder has a `scope()` method.
- This is a one-line change + test update.
- Closes #88.

- [ ] Steps: Update test to assert scope → verify fail → add scope to builder → verify green → commit (`Closes #88`)

---

### Task 11: Simulation API

**Files:**
- Create: `app/src/main/java/io/casehub/aml/simulation/AmlScenarioTemplate.java`
- Create: `app/src/main/java/io/casehub/aml/simulation/AmlSimulationService.java`
- Create: `app/src/main/java/io/casehub/aml/simulation/AmlSimulationResource.java`
- Test: `app/src/test/java/io/casehub/aml/simulation/AmlSimulationResourceTest.java`

**Interfaces:**
- Consumes: `AmlLayer9Resource.startInvestigation()` (or the underlying coordinator), `InvestigationSummaryRepository` (for idempotency check and reset)
- Produces:
  - `POST /api/simulation/seed` — runs all scenarios (202 Accepted)
  - `POST /api/simulation/seed/{scenario}` — runs one scenario (202 Accepted)
  - `DELETE /api/simulation/seed` — full data reset (204)
  - `POST /api/simulation/investigate` — starts a live investigation from a scenario template (returns caseId)

**Context for implementer:**
- **Gate with `@IfBuildProperty`:** All simulation endpoints must be gated by `@io.quarkus.arc.properties.IfBuildProperty(name = "casehub.aml.simulation.enabled", stringValue = "true")` on the resource class. In production builds, the endpoints don't exist.
- Add `casehub.aml.simulation.enabled=true` to test `application.properties` and `%dev` profile in main properties.
- **Scenario templates:** Enum with fields for each scenario (PEP, STRUCTURING, LOW_RISK, SYSTEM_ERROR, GATE_REJECTION, GDPR_ERASED, HIGH_VOLUME). Each template produces a `SuspiciousTransaction` with the appropriate field values.
- **Seed mechanism:** For each scenario, create a `SuspiciousTransaction` from the template and call the Layer 9 start investigation flow. The existing `@DefaultBean` specialist workers produce deterministic results based on `flagReason` and the transaction context.
- **Idempotency:** Check if a summary with matching `transactionId` already exists before seeding.
- **Reset:** `DELETE` truncates `aml_investigation_summary` and related tables. This breaks Merkle chain integrity — acceptable in simulation mode only.
- **Live simulation:** `POST /api/simulation/investigate` accepts `{ "scenario": "PEP" }`, creates the transaction, starts the investigation, returns `{ "caseId": "uuid" }`.

- [ ] Steps: Write scenario enum → write service → write resource with tests → verify green → commit

---

## Execution Notes

- **Task ordering:** Tasks 1→2→3 are sequential (each depends on the previous). Tasks 4–7 can run in parallel after Task 3. Tasks 8–10 are independent. Task 11 depends on all others.
- **Issue filing:** Before the first commit, file an epic issue in casehubio/aml for the workbench backend API. Reference it in all commit messages.
- **Platform coherence review:** After all tasks complete, do a final review against `../parent/docs/PLATFORM.md` — verify boundary rules, persistence conventions, and naming patterns are followed.
- **Code review:** Run `superpowers:requesting-code-review` after each task's commit. Batch minor findings into a single follow-up issue at the end.
