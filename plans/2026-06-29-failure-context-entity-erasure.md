# Failure Context + Entity Erasure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add failure context to terminal investigation states (#80) and entity-level GDPR memory erasure (#79).

**Architecture:** Two independent features on one branch. #80 enriches `InvestigationResolution` with failure chain data from the engine's `EventLog`. #79 adds `CaseMemoryStore.eraseEntity()` orchestration with a tamper-evident ledger receipt. Both follow the existing hexagonal pattern: domain types in `api/`, services and persistence in `app/`.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, Mockito, H2 (test)

## Global Constraints

- `api/` module has zero dependencies — pure Java only. No engine-api, no Jackson annotations.
- All ledger writes use `TenancyConstants.DEFAULT_TENANT_ID` (not `principal.tenancyId()`).
- Flyway AML ledger migrations use `db/aml-ledger/migration/V2NNN__*.sql`, column types match existing migrations (id is UUID, not BIGINT).
- `domainContentBytes()` is mandatory on all `LedgerEntry` subclasses with persistent fields.
- Ledger subject isolation: prefix `"aml-entity-erasure:"` for entity erasure entries.
- Test properties: `casehub.ledger.hash-chain.enabled=false`.
- Maven: `mvn -pl app -am test -Dsurefire.failIfNoSpecifiedTests=false`.

---

### Task 1: Domain types + failure context service (#80)

**Files:**
- Create: `api/src/main/java/io/casehub/aml/domain/FailureContext.java`
- Create: `api/src/main/java/io/casehub/aml/domain/FailureEvent.java`
- Modify: `api/src/main/java/io/casehub/aml/domain/InvestigationResolution.java`
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationOutcomeService.java`
- Modify: `app/src/test/java/io/casehub/aml/engine/AmlInvestigationOutcomeServiceTest.java`

**Interfaces:**
- Consumes: `EventLogRepository.findByCaseAndTypes(UUID, Collection<CaseHubEventType>, String)` → `Uni<List<EventLog>>`
- Produces: `FailureContext(String triggerGoalName, String triggerGoalKind, List<FailureEvent> failureEvents, Instant occurredAt)`, `FailureEvent(String eventType, String workerId, Instant timestamp, String detail)`, `InvestigationResolution(InvestigationStatus, InvestigationOutcome, FailureContext)` — used by Task 2

- [ ] **Step 1: Create FailureEvent record**

```java
// api/src/main/java/io/casehub/aml/domain/FailureEvent.java
package io.casehub.aml.domain;

import java.time.Instant;

public record FailureEvent(
        String eventType,
        String workerId,
        Instant timestamp,
        String detail) {}
```

- [ ] **Step 2: Create FailureContext record**

```java
// api/src/main/java/io/casehub/aml/domain/FailureContext.java
package io.casehub.aml.domain;

import java.time.Instant;
import java.util.List;

public record FailureContext(
        String triggerGoalName,
        String triggerGoalKind,
        List<FailureEvent> failureEvents,
        Instant occurredAt) {}
```

- [ ] **Step 3: Update InvestigationResolution to 3-field record**

Replace the entire file:

```java
// api/src/main/java/io/casehub/aml/domain/InvestigationResolution.java
package io.casehub.aml.domain;

public record InvestigationResolution(
        InvestigationStatus status,
        InvestigationOutcome outcome,
        FailureContext failureContext) {}
```

- [ ] **Step 4: Write failing tests for failure context extraction**

Add these tests to `AmlInvestigationOutcomeServiceTest.java`. They won't compile yet — the `serviceWith()` helper needs the 4th `EventLogRepository` parameter added in Step 6.

```java
// Add imports at top:
import io.casehub.aml.domain.FailureContext;
import io.casehub.aml.domain.FailureEvent;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.api.model.event.EventStreamType;
import io.casehub.engine.common.spi.EventLogRepository;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import java.util.Collection;

@Test
void resolveInvestigation_faulted_goal_triggered_returns_failure_context_with_goal() {
    final UUID caseId = UUID.randomUUID();
    final CaseInstance instance = instanceWithState(caseId, CaseStatus.FAULTED);
    final Instant faultTime = Instant.parse("2026-06-29T10:00:00Z");

    final EventLog faultedEvent = eventLog(caseId, CaseHubEventType.CASE_FAULTED,
            null, faultTime, goalMetadata("pattern-agent-failed", "failure"));
    final EventLog workerFailed = eventLog(caseId, CaseHubEventType.WORKER_EXECUTION_FAILED,
            "pattern-analysis", faultTime.minusSeconds(5), errorMetadata("Connection refused"));

    final AmlInvestigationOutcomeService service = serviceWith(
            List.of(), instance, null, List.of(faultedEvent, workerFailed));
    final Optional<InvestigationResolution> result = service.resolveInvestigation(caseId);

    assertTrue(result.isPresent());
    assertEquals(InvestigationStatus.FAILED, result.get().status());
    assertNull(result.get().outcome());
    assertNotNull(result.get().failureContext());

    final FailureContext ctx = result.get().failureContext();
    assertEquals("pattern-agent-failed", ctx.triggerGoalName());
    assertEquals("failure", ctx.triggerGoalKind());
    assertEquals(faultTime, ctx.occurredAt());
    assertEquals(1, ctx.failureEvents().size());
    assertEquals("WORKER_EXECUTION_FAILED", ctx.failureEvents().get(0).eventType());
    assertEquals("pattern-analysis", ctx.failureEvents().get(0).workerId());
    assertEquals("Connection refused", ctx.failureEvents().get(0).detail());
}

@Test
void resolveInvestigation_faulted_retries_exhausted_disambiguates_two_faulted_events() {
    final UUID caseId = UUID.randomUUID();
    final CaseInstance instance = instanceWithState(caseId, CaseStatus.FAULTED);
    final Instant t1 = Instant.parse("2026-06-29T10:00:00Z");
    final Instant t2 = t1.plusSeconds(1);

    final EventLog retriesFaulted = eventLog(caseId, CaseHubEventType.CASE_FAULTED,
            "pattern-analysis", t1, workerRetriesMetadata("pattern-analysis"));
    final EventLog statusChangeFaulted = eventLog(caseId, CaseHubEventType.CASE_FAULTED,
            null, t2, statusChangeMetadata("RUNNING", "FAULTED"));
    final EventLog workerFailed = eventLog(caseId, CaseHubEventType.WORKER_EXECUTION_FAILED,
            "pattern-analysis", t1.minusSeconds(5), errorMetadata("Timeout"));

    final AmlInvestigationOutcomeService service = serviceWith(
            List.of(), instance, null, List.of(retriesFaulted, statusChangeFaulted, workerFailed));
    final Optional<InvestigationResolution> result = service.resolveInvestigation(caseId);

    assertTrue(result.isPresent());
    final FailureContext ctx = result.get().failureContext();
    assertNull(ctx.triggerGoalName());
    assertNull(ctx.triggerGoalKind());
    assertEquals(t1, ctx.occurredAt());
    assertEquals(1, ctx.failureEvents().size());
}

@Test
void resolveInvestigation_cancelled_returns_failure_context() {
    final UUID caseId = UUID.randomUUID();
    final CaseInstance instance = instanceWithState(caseId, CaseStatus.CANCELLED);
    final Instant cancelTime = Instant.parse("2026-06-29T12:00:00Z");

    final EventLog cancelledEvent = eventLog(caseId, CaseHubEventType.CASE_CANCELLED,
            null, cancelTime, statusChangeMetadata("RUNNING", "CANCELLED"));

    final AmlInvestigationOutcomeService service = serviceWith(
            List.of(), instance, null, List.of(cancelledEvent));
    final Optional<InvestigationResolution> result = service.resolveInvestigation(caseId);

    assertTrue(result.isPresent());
    assertEquals(InvestigationStatus.CANCELLED, result.get().status());
    assertNull(result.get().outcome());
    assertNotNull(result.get().failureContext());
    assertEquals(cancelTime, result.get().failureContext().occurredAt());
    assertTrue(result.get().failureContext().failureEvents().isEmpty());
}

@Test
void resolveInvestigation_suspended_returns_no_failure_context() {
    final UUID caseId = UUID.randomUUID();
    final CaseInstance instance = instanceWithState(caseId, CaseStatus.SUSPENDED);
    final AmlInvestigationOutcomeService service = serviceWith(
            List.of(), instance, null, List.of());
    final Optional<InvestigationResolution> result = service.resolveInvestigation(caseId);

    assertTrue(result.isPresent());
    assertEquals(InvestigationStatus.SUSPENDED, result.get().status());
    assertNull(result.get().outcome());
    assertNull(result.get().failureContext());
}

@Test
void resolveInvestigation_completed_returns_no_failure_context() {
    final UUID caseId = UUID.randomUUID();
    final AmlSarOfficerReviewedLedgerEntry entry = officerEntry("APPROVED", ActorType.HUMAN);
    final CaseInstance instance = completedInstance(caseId);
    final AmlInvestigationOutcomeService service = serviceWith(
            List.of(entry), instance, null, List.of());
    final Optional<InvestigationResolution> result = service.resolveInvestigation(caseId);

    assertTrue(result.isPresent());
    assertEquals(InvestigationStatus.COMPLETED, result.get().status());
    assertNotNull(result.get().outcome());
    assertNull(result.get().failureContext());
}
```

Add these helper methods at the bottom of the test class:

```java
private static final ObjectMapper MAPPER = new ObjectMapper();

private static EventLog eventLog(UUID caseId, CaseHubEventType type,
        String workerId, Instant timestamp, ObjectNode metadata) {
    final EventLog log = new EventLog();
    log.setCaseId(caseId);
    log.setEventType(type);
    log.setStreamType(EventStreamType.CASE);
    log.setWorkerId(workerId);
    log.setTimestamp(timestamp);
    log.setMetadata(metadata);
    return log;
}

private static ObjectNode goalMetadata(String goalName, String goalKind) {
    return MAPPER.createObjectNode()
            .put("oldStatus", "RUNNING").put("newStatus", "FAULTED")
            .put("goalName", goalName).put("goalKind", goalKind);
}

private static ObjectNode statusChangeMetadata(String oldStatus, String newStatus) {
    return MAPPER.createObjectNode()
            .put("oldStatus", oldStatus).put("newStatus", newStatus);
}

private static ObjectNode workerRetriesMetadata(String workerId) {
    return MAPPER.createObjectNode()
            .put("workerId", workerId).put("inputDataHash", "abc123");
}

private static ObjectNode errorMetadata(String errorMessage) {
    return MAPPER.createObjectNode()
            .put("inputDataHash", "abc123").put("errorMessage", errorMessage);
}
```

- [ ] **Step 5: Fix existing tests — add null failureContext to InvestigationResolution constructors**

Every existing test that asserts on `result.get().outcome()` now needs to account for the 3-field record. The existing `resolveInvestigation()` still returns 2-arg constructors — those will fail to compile. Update the existing tests that construct `InvestigationResolution` directly (none do — they call `service.resolveInvestigation()` which returns it). The compilation break is in `AmlInvestigationOutcomeService.resolveInvestigation()` itself — fixed in Step 6.

Existing tests that assert `assertNull(result.get().outcome())` for FAULTED/CANCELLED/SUSPENDED need to also assert `assertNull(result.get().failureContext())` until Step 6 is complete. **Remove** these three existing tests (they are replaced by the new tests in Step 4):
- `resolveInvestigation_faulted_returns_failed`
- `resolveInvestigation_cancelled_returns_cancelled`
- `resolveInvestigation_suspended_returns_suspended`

The new tests from Step 4 cover the same assertions plus failure context.

- [ ] **Step 6: Update serviceWith() helper — add EventLogRepository**

Replace the `serviceWith(List<LedgerEntry>, CaseInstance, CaseInstance)` method and add a 4-arg overload:

```java
private static AmlInvestigationOutcomeService serviceWith(final List<LedgerEntry> entries) {
    return serviceWith(entries, null, null, List.of());
}

private static AmlInvestigationOutcomeService serviceWith(
        final List<LedgerEntry> entries,
        final CaseInstance cacheResult,
        final CaseInstance repoResult) {
    return serviceWith(entries, cacheResult, repoResult, List.of());
}

private static AmlInvestigationOutcomeService serviceWith(
        final List<LedgerEntry> entries,
        final CaseInstance cacheResult,
        final CaseInstance repoResult,
        final List<EventLog> eventLogs) {
    // ... existing LedgerEntryRepository, CaseInstanceCache, CaseInstanceRepository stubs ...

    final EventLogRepository eventLogRepo = new EventLogRepository() {
        @Override
        public Uni<Void> append(EventLog e, String t) { return Uni.createFrom().voidItem(); }

        @Override
        public Uni<Long> appendAndReturnId(EventLog e, String t) { return Uni.createFrom().item(1L); }

        @Override
        public Uni<EventLog> findById(Long id, String t) { return Uni.createFrom().nullItem(); }

        @Override
        public Uni<List<EventLog>> findSchedulingEvents(UUID c, String w, Instant a, String t) {
            return Uni.createFrom().item(List.of());
        }

        @Override
        public Uni<List<EventLog>> findByCaseAndTypes(UUID c, Collection<CaseHubEventType> types, String t) {
            return Uni.createFrom().item(eventLogs.stream()
                    .filter(e -> types.contains(e.getEventType()))
                    .toList());
        }

        @Override
        public Uni<List<EventLog>> findByCaseAndWorkerAndType(UUID c, String w, CaseHubEventType t, String ten) {
            return Uni.createFrom().item(List.of());
        }

        @Override
        public Uni<List<EventLog>> findByWorkerAndType(String w, CaseHubEventType t, String ten) {
            return Uni.createFrom().item(List.of());
        }

        @Override
        public Uni<List<EventLog>> findByCaseWithFilters(UUID c, Collection<CaseHubEventType> et,
                Collection<EventStreamType> st, String t) {
            return Uni.createFrom().item(List.of());
        }
    };
    return new AmlInvestigationOutcomeService(repo, cache, caseRepo, eventLogRepo);
}
```

- [ ] **Step 7: Implement resolveFailureContext() and update resolveInvestigation()**

In `AmlInvestigationOutcomeService.java`:

Add field and update constructor:

```java
private final EventLogRepository eventLogRepository;

@Inject
public AmlInvestigationOutcomeService(
        final LedgerEntryRepository ledgerEntryRepository,
        final CaseInstanceCache caseInstanceCache,
        final CaseInstanceRepository caseInstanceRepository,
        final EventLogRepository eventLogRepository) {
    this.ledgerEntryRepository = ledgerEntryRepository;
    this.caseInstanceCache = caseInstanceCache;
    this.caseInstanceRepository = caseInstanceRepository;
    this.eventLogRepository = eventLogRepository;
}
```

Replace `resolveInvestigation()` body (lines 43-67):

```java
public Optional<InvestigationResolution> resolveInvestigation(final UUID caseId) {
    CaseInstance instance = caseInstanceCache.get(caseId);
    if (instance == null) {
        instance = caseInstanceRepository
                .findByUuid(caseId, TenancyConstants.DEFAULT_TENANT_ID)
                .await().indefinitely();
    }
    if (instance == null) {
        return Optional.empty();
    }

    InvestigationStatus status = switch (instance.getState()) {
        case STARTING, RUNNING, WAITING -> InvestigationStatus.IN_PROGRESS;
        case COMPLETED -> InvestigationStatus.COMPLETED;
        case FAULTED -> InvestigationStatus.FAILED;
        case CANCELLED -> InvestigationStatus.CANCELLED;
        case SUSPENDED -> InvestigationStatus.SUSPENDED;
    };

    return switch (status) {
        case COMPLETED -> Optional.of(new InvestigationResolution(
                status, resolveOutcome(caseId), null));
        case FAILED, CANCELLED -> Optional.of(new InvestigationResolution(
                status, null, resolveFailureContext(caseId)));
        case IN_PROGRESS, SUSPENDED -> Optional.of(new InvestigationResolution(
                status, null, null));
    };
}
```

Add `resolveFailureContext()`:

```java
FailureContext resolveFailureContext(final UUID caseId) {
    final List<EventLog> events = eventLogRepository.findByCaseAndTypes(
            caseId,
            List.of(CaseHubEventType.CASE_FAULTED, CaseHubEventType.CASE_CANCELLED,
                    CaseHubEventType.WORKER_EXECUTION_FAILED,
                    CaseHubEventType.WORKER_OUTCOME_FAILED,
                    CaseHubEventType.ACTION_GATE_REJECTED,
                    CaseHubEventType.ACTION_GATE_EXPIRED),
            TenancyConstants.DEFAULT_TENANT_ID)
            .await().indefinitely();

    String triggerGoalName = null;
    String triggerGoalKind = null;
    Instant occurredAt = null;

    for (final EventLog e : events) {
        if (e.getEventType() == CaseHubEventType.CASE_FAULTED
                || e.getEventType() == CaseHubEventType.CASE_CANCELLED) {
            if (occurredAt == null || e.getTimestamp().isBefore(occurredAt)) {
                occurredAt = e.getTimestamp();
            }
            if (triggerGoalName == null && e.getMetadata() != null
                    && e.getMetadata().has("goalName")) {
                triggerGoalName = e.getMetadata().get("goalName").asText();
                triggerGoalKind = e.getMetadata().has("goalKind")
                        ? e.getMetadata().get("goalKind").asText() : null;
            }
        }
    }

    final List<FailureEvent> failureEvents = events.stream()
            .filter(e -> e.getEventType() != CaseHubEventType.CASE_FAULTED
                    && e.getEventType() != CaseHubEventType.CASE_CANCELLED)
            .sorted(Comparator.comparing(EventLog::getTimestamp))
            .map(e -> new FailureEvent(
                    e.getEventType().name(),
                    e.getWorkerId(),
                    e.getTimestamp(),
                    extractDetail(e)))
            .toList();

    return new FailureContext(triggerGoalName, triggerGoalKind, failureEvents,
            occurredAt != null ? occurredAt : Instant.now());
}

private String extractDetail(final EventLog e) {
    if (e.getMetadata() == null) return null;
    if (e.getMetadata().has("errorMessage")) {
        return e.getMetadata().get("errorMessage").asText();
    }
    if (e.getMetadata().has("reason")) {
        return e.getMetadata().get("reason").asText();
    }
    return null;
}
```

Add imports:

```java
import io.casehub.aml.domain.FailureContext;
import io.casehub.aml.domain.FailureEvent;
import io.casehub.api.model.event.CaseHubEventType;
import io.casehub.engine.common.spi.EventLogRepository;
import java.util.List;
```

- [ ] **Step 8: Run tests**

Run: `mvn -pl api -am test && mvn -pl app -am test -Dtest=AmlInvestigationOutcomeServiceTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: All tests pass including 5 new failure context tests.

- [ ] **Step 9: Commit**

```
feat(#80): failure context on terminal InvestigationStatus — domain types + service

Refs #80
```

---

### Task 2: REST layer — failure context in responses (#80)

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/engine/Layer6InvestigationResponse.java`
- Modify: `app/src/main/java/io/casehub/aml/engine/Layer9InvestigationResponse.java`
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlLayer6Resource.java`
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java`

**Interfaces:**
- Consumes: `InvestigationResolution(status, outcome, failureContext)` from Task 1
- Produces: JSON responses with `failureContext` field for non-COMPLETED terminal states

- [ ] **Step 1: Update Layer6InvestigationResponse**

Replace the record:

```java
public record Layer6InvestigationResponse(
        UUID caseId,
        InvestigationStatus status,
        List<WorkerRoutingDecision> routingDecisions,
        InvestigationOutcome outcome,
        FailureContext failureContext) {}
```

Add import: `import io.casehub.aml.domain.FailureContext;`

- [ ] **Step 2: Update Layer9InvestigationResponse**

Replace the record:

```java
public record Layer9InvestigationResponse(
        UUID caseId,
        InvestigationStatus status,
        InvestigationOutcome outcome,
        FailureContext failureContext) {}
```

Add import: `import io.casehub.aml.domain.FailureContext;`

- [ ] **Step 3: Update AmlLayer6Resource.getInvestigation()**

Replace lines 69-84 (the response construction):

```java
if (r.status() != InvestigationStatus.COMPLETED) {
    return Response.ok(new Layer6InvestigationResponse(
            caseId, r.status(), List.of(), null, r.failureContext())).build();
}
final List<WorkerDecisionEntry> entries = workerDecisionRepo.findAllByCaseId(caseId);
final List<WorkerRoutingDecision> decisions = entries.stream()
        .map(e -> {
            final OptionalDouble score =
                    trustScoreSource.capabilityScore(e.workerId, e.capabilityTag);
            return new WorkerRoutingDecision(
                    e.capabilityTag, e.workerId,
                    score.isPresent() ? score.getAsDouble() : null);
        })
        .toList();
return Response.ok(new Layer6InvestigationResponse(
        caseId, r.status(), decisions, r.outcome(), null)).build();
```

- [ ] **Step 4: Update AmlLayer9Resource.getInvestigation()**

Replace line 52:

```java
return Response.ok(new Layer9InvestigationResponse(
        caseId, r.status(), r.outcome(), r.failureContext())).build();
```

- [ ] **Step 5: Run tests**

Run: `mvn -pl app -am test -Dsurefire.failIfNoSpecifiedTests=false`

Expected: All existing tests pass. The Layer6 and Layer9 resource tests compile with the new 5/4-field constructors.

- [ ] **Step 6: Commit**

```
feat(#80): wire failure context through Layer6 and Layer9 REST responses

Refs #80
```

---

### Task 3: Rename AmlErasureResult → ActorErasureResult (#79 prep)

**Files:**
- Rename: `app/src/main/java/io/casehub/aml/compliance/AmlErasureResult.java` → `ActorErasureResult.java`
- Modify: `app/src/main/java/io/casehub/aml/compliance/AmlErasureService.java`
- Modify: `app/src/main/java/io/casehub/aml/compliance/AmlLayer7Resource.java`
- Modify: `app/src/test/java/io/casehub/aml/compliance/AmlErasureServiceTest.java`

**Interfaces:**
- Produces: `ActorErasureResult(String erasedActorId, boolean mappingFound, long affectedEntryCount, UUID receiptEntryId)` — same shape, new name

- [ ] **Step 1: Rename the file and class via IntelliJ refactor**

Use `ide_refactor_rename` to rename `AmlErasureResult` → `ActorErasureResult`. This updates all references automatically.

- [ ] **Step 2: Verify compilation**

Run: `mvn -pl app -am test -Dtest=AmlErasureServiceTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: All tests pass with renamed type.

- [ ] **Step 3: Commit**

```
refactor(#79): rename AmlErasureResult → ActorErasureResult

Refs #79
```

---

### Task 4: Entity erasure ledger entry + migration + AmlLedgerService (#79)

**Files:**
- Create: `app/src/main/java/io/casehub/aml/ledger/AmlEntityErasureLedgerEntry.java`
- Create: `app/src/main/resources/db/aml-ledger/migration/V2013__aml_entity_erasure_entry.sql`
- Modify: `app/src/main/java/io/casehub/aml/ledger/AmlLedgerService.java`

**Interfaces:**
- Consumes: `LedgerEntryRepository.save(entry, tenancyId)`, `LedgerEntry` base class
- Produces: `AmlLedgerService.writeEntityErasure(String entityId, ErasureReason reason, int memoriesErased, String actorId, ActorType actorType)` → `UUID` — used by Task 5

- [ ] **Step 1: Create AmlEntityErasureLedgerEntry**

```java
// app/src/main/java/io/casehub/aml/ledger/AmlEntityErasureLedgerEntry.java
package io.casehub.aml.ledger;

import java.nio.charset.StandardCharsets;

import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Table;

import io.casehub.ledger.api.model.ErasureReason;
import io.casehub.ledger.runtime.model.LedgerEntry;

@Entity
@Table(name = "aml_entity_erasure_entry")
@DiscriminatorValue("AML_ENTITY_ERASURE")
public class AmlEntityErasureLedgerEntry extends LedgerEntry {

    @Column(name = "erased_entity_id", nullable = false)
    public String erasedEntityId;

    @Enumerated(EnumType.STRING)
    @Column(name = "erasure_reason", nullable = false)
    public ErasureReason erasureReason;

    @Column(name = "memories_erased", nullable = false)
    public int memoriesErased;

    @Override
    protected byte[] domainContentBytes() {
        final String content = String.join("|",
                erasedEntityId != null ? erasedEntityId : "",
                erasureReason != null ? erasureReason.name() : "",
                String.valueOf(memoriesErased));
        return content.getBytes(StandardCharsets.UTF_8);
    }
}
```

- [ ] **Step 2: Create V2013 migration**

```sql
-- app/src/main/resources/db/aml-ledger/migration/V2013__aml_entity_erasure_entry.sql
CREATE TABLE aml_entity_erasure_entry (
    id UUID NOT NULL,
    erased_entity_id VARCHAR(255) NOT NULL,
    erasure_reason VARCHAR(50) NOT NULL,
    memories_erased INT NOT NULL,
    CONSTRAINT pk_aml_entity_erasure_entry PRIMARY KEY (id),
    CONSTRAINT fk_aml_entity_erasure_entry_ledger
        FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

- [ ] **Step 3: Add writeEntityErasure() to AmlLedgerService**

Add this method after `writeSarOfficerReviewedFailure()`:

```java
public UUID writeEntityErasure(final String entityId, final ErasureReason reason,
        final int memoriesErased, final String actorId, final ActorType actorType) {
    final UUID subjectId = UUID.nameUUIDFromBytes(
            ("aml-entity-erasure:" + entityId).getBytes(java.nio.charset.StandardCharsets.UTF_8));
    final int sequenceNumber = nextSequenceNumber(subjectId);
    final AmlEntityErasureLedgerEntry entry = new AmlEntityErasureLedgerEntry();
    entry.id = UUID.randomUUID();
    entry.subjectId = subjectId;
    entry.sequenceNumber = sequenceNumber;
    entry.entryType = LedgerEntryType.EVENT;
    entry.actorId = actorId;
    entry.actorType = actorType;
    entry.actorRole = "GdprComplianceOfficer";
    entry.occurredAt = Instant.now();
    entry.erasedEntityId = entityId;
    entry.erasureReason = reason;
    entry.memoriesErased = memoriesErased;
    repository.save(entry, TenancyConstants.DEFAULT_TENANT_ID);
    return entry.id;
}
```

Add import: `import io.casehub.ledger.api.model.ErasureReason;`

- [ ] **Step 4: Update noOp() and stub() helpers**

Add `writeEntityErasure()` override to both anonymous subclasses in `noOp()` and `stub()`:

In `noOp()`:
```java
@Override public UUID writeEntityErasure(String entityId, ErasureReason reason,
        int memoriesErased, String actorId, ActorType actorType) { return UUID.randomUUID(); }
```

In `stub(final UUID entryId)`:
```java
@Override public UUID writeEntityErasure(String entityId, ErasureReason reason,
        int memoriesErased, String actorId, ActorType actorType) { return entryId; }
```

Add imports to both stubs: `import io.casehub.ledger.api.model.ErasureReason;` (already imported at class level after Step 3).

- [ ] **Step 5: Run compilation check**

Run: `mvn -pl app -am compile`

Expected: Compiles successfully. `AmlEntityErasureLedgerEntry` passes `LedgerProcessor` build-time guard.

- [ ] **Step 6: Commit**

```
feat(#79): AmlEntityErasureLedgerEntry, V2013 migration, AmlLedgerService.writeEntityErasure()

Refs #79
```

---

### Task 5: Entity erasure service + endpoint + tests (#79)

**Files:**
- Create: `app/src/main/java/io/casehub/aml/compliance/EntityErasureResult.java`
- Modify: `app/src/main/java/io/casehub/aml/compliance/AmlErasureService.java`
- Modify: `app/src/main/java/io/casehub/aml/compliance/AmlLayer7Resource.java`
- Modify: `app/src/test/java/io/casehub/aml/compliance/AmlErasureServiceTest.java`

**Interfaces:**
- Consumes: `CaseMemoryStore.eraseEntity(String, String)` → `int`, `AmlLedgerService.writeEntityErasure(...)` → `UUID` from Task 4
- Produces: `POST /api/entities/{entityId}/erasure` → `EntityErasureResult`

- [ ] **Step 1: Create EntityErasureResult**

```java
// app/src/main/java/io/casehub/aml/compliance/EntityErasureResult.java
package io.casehub.aml.compliance;

import java.util.UUID;

public record EntityErasureResult(
        String entityId,
        int memoriesErased,
        UUID receiptEntryId) {}
```

- [ ] **Step 2: Write failing tests for entity erasure**

Add to `AmlErasureServiceTest.java`:

```java
// Add imports:
import io.casehub.aml.ledger.AmlLedgerService;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.memory.CaseMemoryStore;
import io.casehub.platform.api.memory.MemoryInput;
import io.casehub.platform.api.memory.MemoryQuery;
import io.casehub.platform.api.memory.EraseRequest;
import io.casehub.platform.api.memory.Memory;

@Test
void eraseEntity_returns_count_and_receipt() {
    final UUID receiptId = UUID.randomUUID();
    final LedgerErasureService ledger = Mockito.mock(LedgerErasureService.class);
    final CaseMemoryStore memoryStore = Mockito.mock(CaseMemoryStore.class);
    final CurrentPrincipal principal = mockPrincipal("officer-1", ActorType.HUMAN);
    final AmlLedgerService ledgerService = AmlLedgerService.stub(receiptId);

    when(memoryStore.eraseEntity("ACCT-12345", "default")).thenReturn(3);

    final AmlErasureService service = new AmlErasureService(
            ledger, memoryStore, principal, ledgerService);
    final EntityErasureResult result = service.eraseEntity(
            "ACCT-12345", ErasureReason.GDPR_ART_17_REQUEST);

    assertEquals("ACCT-12345", result.entityId());
    assertEquals(3, result.memoriesErased());
    assertEquals(receiptId, result.receiptEntryId());
}

@Test
void eraseEntity_with_no_memories_still_writes_receipt() {
    final UUID receiptId = UUID.randomUUID();
    final LedgerErasureService ledger = Mockito.mock(LedgerErasureService.class);
    final CaseMemoryStore memoryStore = Mockito.mock(CaseMemoryStore.class);
    final CurrentPrincipal principal = mockPrincipal("officer-1", ActorType.HUMAN);
    final AmlLedgerService ledgerService = AmlLedgerService.stub(receiptId);

    when(memoryStore.eraseEntity("ACCT-99999", "default")).thenReturn(0);

    final AmlErasureService service = new AmlErasureService(
            ledger, memoryStore, principal, ledgerService);
    final EntityErasureResult result = service.eraseEntity(
            "ACCT-99999", ErasureReason.GDPR_ART_17_REQUEST);

    assertEquals(0, result.memoriesErased());
    assertEquals(receiptId, result.receiptEntryId());
}

private static CurrentPrincipal mockPrincipal(String actorId, ActorType actorType) {
    final CurrentPrincipal p = Mockito.mock(CurrentPrincipal.class);
    when(p.actorId()).thenReturn(actorId);
    when(p.actorType()).thenReturn(actorType);
    when(p.tenancyId()).thenReturn("default");
    return p;
}
```

- [ ] **Step 3: Update existing tests for new AmlErasureService constructor**

The existing tests create `new AmlErasureService(ledger)` — update to the 4-arg constructor. The existing `erase()` method doesn't use the new dependencies, so pass mocks:

```java
@Test
void erase_maps_ledger_result_with_receipt() {
    final UUID receiptId = UUID.randomUUID();
    final LedgerErasureService ledger = Mockito.mock(LedgerErasureService.class);
    when(ledger.erase(eq("officer-jane"), eq(ErasureReason.GDPR_ART_17_REQUEST)))
            .thenReturn(new ErasureResult("officer-jane", true, 5L, Optional.of(receiptId)));
    final AmlErasureService service = new AmlErasureService(
            ledger, Mockito.mock(CaseMemoryStore.class),
            mockPrincipal("x", ActorType.SYSTEM), AmlLedgerService.noOp());

    final ActorErasureResult result = service.erase("officer-jane", ErasureReason.GDPR_ART_17_REQUEST);

    assertEquals("officer-jane", result.erasedActorId());
    assertTrue(result.mappingFound());
    assertEquals(5L, result.affectedEntryCount());
    assertEquals(receiptId, result.receiptEntryId());
}

@Test
void erase_maps_empty_receipt_to_null() {
    final LedgerErasureService ledger = Mockito.mock(LedgerErasureService.class);
    when(ledger.erase(anyString(), any()))
            .thenReturn(new ErasureResult("officer-jane", false, 0L, Optional.empty()));
    final AmlErasureService service = new AmlErasureService(
            ledger, Mockito.mock(CaseMemoryStore.class),
            mockPrincipal("x", ActorType.SYSTEM), AmlLedgerService.noOp());

    final ActorErasureResult result = service.erase("officer-jane", ErasureReason.GDPR_ART_17_REQUEST);

    assertFalse(result.mappingFound());
    assertEquals(0L, result.affectedEntryCount());
    assertNull(result.receiptEntryId());
}
```

- [ ] **Step 4: Implement eraseEntity() in AmlErasureService**

Update the class:

```java
@ApplicationScoped
public class AmlErasureService {

    private final LedgerErasureService ledgerErasureService;
    private final CaseMemoryStore memoryStore;
    private final CurrentPrincipal principal;
    private final AmlLedgerService ledgerService;

    @Inject
    public AmlErasureService(
            final LedgerErasureService ledgerErasureService,
            final CaseMemoryStore memoryStore,
            final CurrentPrincipal principal,
            final AmlLedgerService ledgerService) {
        this.ledgerErasureService = ledgerErasureService;
        this.memoryStore = memoryStore;
        this.principal = principal;
        this.ledgerService = ledgerService;
    }

    public ActorErasureResult erase(final String actorId, final ErasureReason reason) {
        final LedgerErasureService.ErasureResult ledgerResult =
                ledgerErasureService.erase(actorId, reason);
        return new ActorErasureResult(
                ledgerResult.rawActorId(),
                ledgerResult.mappingFound(),
                ledgerResult.affectedEntryCount(),
                ledgerResult.receiptEntryId().orElse(null));
    }

    public EntityErasureResult eraseEntity(final String entityId, final ErasureReason reason) {
        final int memoriesErased = memoryStore.eraseEntity(
                entityId, TenancyConstants.DEFAULT_TENANT_ID);
        final UUID receiptEntryId = ledgerService.writeEntityErasure(
                entityId, reason, memoriesErased,
                principal.actorId(), principal.actorType());
        return new EntityErasureResult(entityId, memoriesErased, receiptEntryId);
    }
}
```

Add imports:

```java
import io.casehub.aml.ledger.AmlLedgerService;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.platform.api.memory.CaseMemoryStore;
```

- [ ] **Step 5: Add entity erasure endpoint to AmlLayer7Resource**

Add after the existing `AmlGdprErasureResource` class in the same file:

```java
@ApplicationScoped
@Path("/api/entities/{entityId}/erasure")
@Produces(MediaType.APPLICATION_JSON)
class AmlEntityErasureResource {

    @Inject
    AmlErasureService erasureService;

    @POST
    public EntityErasureResult eraseEntity(@PathParam("entityId") String entityId) {
        return erasureService.eraseEntity(entityId, ErasureReason.GDPR_ART_17_REQUEST);
    }
}
```

- [ ] **Step 6: Run all tests**

Run: `mvn -pl app -am test -Dsurefire.failIfNoSpecifiedTests=false`

Expected: All tests pass — existing actor erasure tests (updated for 4-arg constructor), new entity erasure tests, and all failure context tests from Tasks 1-2.

- [ ] **Step 7: Commit**

```
feat(#79): entity-level GDPR memory erasure — AmlErasureService.eraseEntity() + REST endpoint

Refs #79
```

---

## Self-Review Checklist

**Spec coverage:**
- ✅ FailureContext, FailureEvent domain types
- ✅ InvestigationResolution 3-field record
- ✅ resolveFailureContext() with EventLog query
- ✅ Multi-CASE_FAULTED disambiguation (earliest timestamp, scan for goalName)
- ✅ SUSPENDED returns both null (not FailureContext)
- ✅ Layer6/Layer9 response updates
- ✅ ActorErasureResult rename
- ✅ AmlEntityErasureLedgerEntry + V2013 migration
- ✅ AmlLedgerService.writeEntityErasure() + stub updates
- ✅ AmlErasureService.eraseEntity() with CaseMemoryStore
- ✅ Entity erasure REST endpoint
- ✅ All test cases from spec
- ✅ deterministic subjectId with "aml-entity-erasure:" prefix
- ✅ TenancyConstants.DEFAULT_TENANT_ID (not principal.tenancyId())
- ✅ Actor fields: principal.actorId(), principal.actorType(), role "GdprComplianceOfficer"

**Placeholder scan:** None found.

**Type consistency:** `FailureContext`, `FailureEvent`, `EntityErasureResult`, `ActorErasureResult` — names and signatures consistent across all tasks.
