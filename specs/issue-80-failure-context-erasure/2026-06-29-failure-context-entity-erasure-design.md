# Design: Failure Context on Terminal Status + Entity-Level Memory Erasure

**Issues:** #80 (failure context), #79 (entity erasure)  
**Branch:** `issue-80-failure-context-erasure`  
**Date:** 2026-06-29  
**Deferred:** #83 (ledger content erasure), #84 (cross-tenant erasure)

---

## Problem Statement

### #80 — Failure context

`AmlInvestigationOutcomeService.resolveInvestigation()` returns `InvestigationResolution(status, null)` for all non-COMPLETED terminal states. Compliance officers cannot determine why an investigation failed or was cancelled. The engine's `CaseInstance` has no fault-reason field — failure context lives in the **EventLog**, which records typed events (`CASE_FAULTED`, `WORKER_EXECUTION_FAILED`, `ACTION_GATE_REJECTED`, etc.) with JSON metadata including goal name, goal kind, worker ID, and timestamps.

`InvestigationOutcome(type, reason)` models review decisions (`"sar-filed"`, `"gate-rejected"`). Using it for failure context would be a semantic mismatch — review outcomes and failure reasons are different concepts.

### #79 — Entity-level memory erasure

AML memory entries are keyed by `entityId` (account numbers), not `actorId` (investigators/agents). The current erasure endpoint (`POST /api/actors/{actorId}/erasure`) pseudonymises actor identity in ledger entries via `LedgerErasureService` but does not touch memory entries. When a data subject (account holder) exercises GDPR Art.17 rights, their investigation memory persists.

`CaseMemoryStore.eraseEntity(entityId, tenantId)` exists in the platform and performs cross-domain hard deletion — AML doesn't call it anywhere.

---

## Design

### Domain model (api/ module)

The api module has zero dependencies (pure Java). Engine-specific values use strings; mapping happens in the app service layer.

**New types:**

```java
package io.casehub.aml.domain;

public record FailureContext(
    String triggerGoalName,
    String triggerGoalKind,
    List<FailureEvent> failureEvents,
    Instant occurredAt) {}

public record FailureEvent(
    String eventType,
    String workerId,
    Instant timestamp,
    String detail) {}
```

`FailureEvent.detail` is a human-readable string extracted from the EventLog entry's `metadata` JSON — e.g., `"Failure goal 'pattern-agent-failed' satisfied"` or `"Gate expired after 30-day SLA"`. Null when metadata contains no actionable detail.

```java
public record EntityErasureResult(
    String entityId,
    int memoriesErased,
    UUID receiptEntryId) {}
```

**Modified types:**

```java
// Add failureContext field
public record InvestigationResolution(
    InvestigationStatus status,
    InvestigationOutcome outcome,
    FailureContext failureContext) {}

// Rename AmlErasureResult → ActorErasureResult
public record ActorErasureResult(
    String erasedActorId,
    boolean mappingFound,
    long affectedEntryCount,
    UUID receiptEntryId) {}
```

`InvestigationOutcome` is unchanged — it retains its original review-decision semantics.

**Invariants:**
- COMPLETED: `outcome` non-null, `failureContext` null
- FAILED/CANCELLED/SUSPENDED: `outcome` null, `failureContext` non-null
- IN_PROGRESS: both null

Enforced by construction in `AmlInvestigationOutcomeService` (the single producer).

### Service layer (app/ module)

**`AmlInvestigationOutcomeService.resolveInvestigation()`:**

```java
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
    case FAILED, CANCELLED, SUSPENDED -> Optional.of(new InvestigationResolution(
        status, null, resolveFailureContext(caseId, status)));
    case IN_PROGRESS -> Optional.of(new InvestigationResolution(
        status, null, null));
};
```

**`resolveFailureContext()`:**

New method. Queries `EventLogRepository.findByCaseAndTypes()` for:
- Terminal events: `CASE_FAULTED`, `CASE_CANCELLED`
- Failure events: `WORKER_EXECUTION_FAILED`, `WORKER_OUTCOME_FAILED`, `ACTION_GATE_REJECTED`, `ACTION_GATE_EXPIRED`

Extracts `goalName`/`goalKind` from the terminal event's metadata JSON (`metadata.goalName`, `metadata.goalKind`). Maps each failure event to a `FailureEvent` record. Returns `FailureContext` with the chain ordered by timestamp.

For SUSPENDED: returns `FailureContext(null, null, List.of(), occurredAt)` — timestamp only, no failure chain.

New dependency: `EventLogRepository` (engine-common, already on app classpath).

**`AmlErasureService.eraseEntity()`:**

New method. Orchestrates:
1. `CaseMemoryStore.eraseEntity(entityId, tenantId)` → count
2. Write `AmlEntityErasureLedgerEntry` with `subjectId = UUID.nameUUIDFromBytes(("aml-entity-erasure:" + entityId).getBytes(UTF_8))`
3. Return `EntityErasureResult(entityId, memoriesErased, receipt.id)`

New dependencies: `CaseMemoryStore`, `CurrentPrincipal` (both already available in app context).

### REST API

**Modified responses:**

```java
public record Layer6InvestigationResponse(
    UUID caseId,
    InvestigationStatus status,
    List<WorkerRoutingDecision> routingDecisions,
    InvestigationOutcome outcome,
    FailureContext failureContext) {}

public record Layer9InvestigationResponse(
    UUID caseId,
    InvestigationStatus status,
    InvestigationOutcome outcome,
    FailureContext failureContext) {}
```

**New endpoint:**

```
POST /api/entities/{entityId}/erasure → EntityErasureResult
```

Defined as `AmlEntityErasureResource` in `AmlLayer7Resource.java` (alongside the existing `AmlGdprErasureResource`). Calls `AmlErasureService.eraseEntity(entityId, GDPR_ART_17_REQUEST)`.

**Existing endpoint return type rename:**

`AmlGdprErasureResource.eraseActor()` returns `ActorErasureResult` (was `AmlErasureResult`).

### Ledger entry and migration

**`AmlEntityErasureLedgerEntry`:**

```java
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
        return String.join("|",
            erasedEntityId != null ? erasedEntityId : "",
            erasureReason != null ? erasureReason.name() : "",
            String.valueOf(memoriesErased)
        ).getBytes(StandardCharsets.UTF_8);
    }
}
```

**V2013** — `db/aml-ledger/migration/V2013__aml_entity_erasure_entry.sql`:

```sql
CREATE TABLE aml_entity_erasure_entry (
    id BIGINT NOT NULL,
    erased_entity_id VARCHAR(255) NOT NULL,
    erasure_reason VARCHAR(50) NOT NULL,
    memories_erased INT NOT NULL,
    CONSTRAINT pk_aml_entity_erasure_entry PRIMARY KEY (id),
    CONSTRAINT fk_aml_entity_erasure_entry_ledger
        FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

### Testing

**Unit tests (api/ module):**
- `FailureContextTest` — construction, null goal for non-goal-triggered faults
- `FailureEventTest` — construction
- `InvestigationResolutionTest` — updated for three-field record
- `ActorErasureResultTest` — renamed from existing test
- `EntityErasureResultTest` — construction with receipt ID

**@QuarkusTest (app/ module):**
- `AmlInvestigationOutcomeServiceTest` additions:
  - FAULTED case: seed EventLog with `CASE_FAULTED` + `WORKER_EXECUTION_FAILED` → verify failureContext
  - CANCELLED case: seed EventLog with `CASE_CANCELLED` → verify failureContext with occurredAt
  - SUSPENDED case: verify failureContext with occurredAt, empty events
  - COMPLETED case: verify failureContext is null (backward compat)
- `AmlErasureServiceTest` additions:
  - `eraseEntity()` with memories present → verify count and receipt
  - `eraseEntity()` with no memories → verify `memoriesErased=0`, receipt still written

**Integration tests:**
- Layer6 GET on FAULTED caseId → JSON includes failureContext
- Layer6 GET on COMPLETED caseId → failureContext is null
- Entity erasure POST → EntityErasureResult in response

**Test conventions (from CLAUDE.md):**
- `casehub.ledger.hash-chain.enabled=false`
- Drain investigations to terminal status before asserting
- `UUID.nameUUIDFromBytes("aml-entity-erasure:" + entityId)` for subject isolation
- Awaitility with `QuarkusTransaction.requiringNew()` for default-datasource queries

---

## Protocol compliance

| Protocol | Status |
|----------|--------|
| `aml-ledger-entry-tenancy-id-non-null` | ✅ `eraseEntity()` is synchronous REST — CDI request context active, `principal.tenancyId()` guaranteed non-null |
| `tenant-principal-exclusion` | ✅ No new `CurrentPrincipal` implementations |
| `domainContentBytes()` enforcement | ✅ `AmlEntityErasureLedgerEntry` overrides with pipe-delimited fields |
| Ledger subject isolation | ✅ `"aml-entity-erasure:"` prefix |
| Flyway V2001+ for AML ledger | ✅ V2013, no conflict |
| api/ module purity | ✅ Strings for engine-specific values, no new dependencies |

## Deferred concerns

| # | Description | Why deferred |
|---|-------------|-------------|
| #83 | Entity data in ledger content and case context | Architectural limitation — ledger is tamper-evident, content can't be modified |
| #84 | Cross-tenant entity erasure | AML uses single tenant; only relevant when multi-tenancy activates |
