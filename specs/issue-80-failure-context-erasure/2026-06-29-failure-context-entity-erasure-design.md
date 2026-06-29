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

**EventLog metadata schemas by fault path:**

The engine writes `CASE_FAULTED` events from three distinct code paths, each with a different metadata schema:

| Path | Handler | Metadata fields |
|------|---------|----------------|
| Goal-triggered fault | `CaseStatusChangedHandler` | `oldStatus`, `newStatus`, `goalName`, `goalKind` |
| Worker retries exhausted | `WorkerRetriesExhaustedEventHandler` | `workerId`, `inputDataHash` |
| Case timeout | `CaseTimeoutEnforcer` | `reason: "timeout"` |

`resolveFailureContext()` extracts `goalName`/`goalKind` from the terminal event when present (goal-triggered path). For the other two paths, `triggerGoalName` and `triggerGoalKind` are null — the failure chain events (`WORKER_EXECUTION_FAILED`, etc.) provide the diagnostic detail.

**Multiple terminal events:** For the worker-retries-exhausted and timeout paths, the engine writes two `CASE_FAULTED` entries: one from the originating handler (`WorkerRetriesExhaustedEventHandler` / `CaseTimeoutEnforcer`) and a second from `CaseStatusChangedHandler` (triggered by the `CASE_STATUS_CHANGED` event bus message). The goal-triggered path produces only one. Disambiguation rule: iterate all CASE_FAULTED/CASE_CANCELLED entries; use the earliest timestamp for `occurredAt`; extract `goalName`/`goalKind` from whichever entry has them (at most one will).

For `CASE_CANCELLED`, metadata always contains `oldStatus` and `newStatus` (written by `CaseStatusChangedHandler`).

**Failure chain event metadata schemas:**

| Event type | Handler | Metadata fields | `FailureEvent.detail` extraction |
|---|---|---|---|
| `WORKER_EXECUTION_FAILED` | `QuartzRetryService` | `inputDataHash`, `errorMessage` | `errorMessage` |
| `WORKER_OUTCOME_FAILED` | `WorkflowExecutionCompletedHandler` | `bindingName`, `reason`, `attempts`, `disposition` | `reason` |
| `ACTION_GATE_REJECTED` | `ActionGateRejectedHandler` | *(none)* | null |
| `ACTION_GATE_EXPIRED` | `ActionGateExpiredHandler` | *(none)* | null |

For all failure chain events, `workerId` is set directly on the `EventLog` entity (not in metadata) and maps to `FailureEvent.workerId`.

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
- COMPLETED: `outcome` from `resolveOutcome()` (null when no SAR review entry exists — a data-integrity anomaly, not a normal case), `failureContext` null
- FAILED/CANCELLED: `outcome` null, `failureContext` non-null
- SUSPENDED/IN_PROGRESS: both null

Enforced by construction in `AmlInvestigationOutcomeService` (the single producer). SUSPENDED is not a failure — it is an intermediate pause like IN_PROGRESS.

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
    case FAILED, CANCELLED -> Optional.of(new InvestigationResolution(
        status, null, resolveFailureContext(caseId, status)));
    case IN_PROGRESS, SUSPENDED -> Optional.of(new InvestigationResolution(
        status, null, null));
};
```

**`resolveFailureContext()`:**

New method. Queries `EventLogRepository.findByCaseAndTypes()` for:
- Terminal events: `CASE_FAULTED`, `CASE_CANCELLED`
- Failure events: `WORKER_EXECUTION_FAILED`, `WORKER_OUTCOME_FAILED`, `ACTION_GATE_REJECTED`, `ACTION_GATE_EXPIRED`

Extracts `goalName`/`goalKind` from the terminal event's metadata JSON when present (goal-triggered fault path only — see metadata schema table above). Maps each failure event to a `FailureEvent` record. Returns `FailureContext` with the chain ordered by timestamp.

`EventLogRepository.findByCaseAndTypes()` returns `Uni<List<EventLog>>` — call `.await().indefinitely()` to block, matching the existing pattern in `resolveInvestigation()` for `CaseInstanceRepository`.

New dependency: `EventLogRepository` (engine-common, already on app classpath). Constructor changes:

```java
@Inject
public AmlInvestigationOutcomeService(
        final LedgerEntryRepository ledgerEntryRepository,
        final CaseInstanceCache caseInstanceCache,
        final CaseInstanceRepository caseInstanceRepository,
        final EventLogRepository eventLogRepository) {
```

The existing `AmlInvestigationOutcomeServiceTest.serviceWith()` helper needs a 4th parameter — a mock `EventLogRepository` returning `Uni.createFrom().item(List.of(...))` with seeded `EventLog` entries for failure context tests.

**`AmlErasureService.eraseEntity()`:**

New method. Constructor changes — inject `CaseMemoryStore`, `CurrentPrincipal`, and `AmlLedgerService` alongside the existing `LedgerErasureService`:

```java
@Inject
public AmlErasureService(
        final LedgerErasureService ledgerErasureService,
        final CaseMemoryStore memoryStore,
        final CurrentPrincipal principal,
        final AmlLedgerService ledgerService) {
```

Orchestration:
1. `CaseMemoryStore.eraseEntity(entityId, TenancyConstants.DEFAULT_TENANT_ID)` → `memoriesErased`
2. `AmlLedgerService.writeEntityErasure(entityId, reason, memoriesErased, principal.actorId(), principal.actorType())` → `receiptEntryId`
3. Return `EntityErasureResult(entityId, memoriesErased, receiptEntryId)`

The ledger write uses `AmlLedgerService.writeEntityErasure()` (new method) following the established pattern in `writeCaseOpened()` / `writeSarOfficerReviewed()`: populate all `LedgerEntry` base fields and call `LedgerEntryRepository.save(entry, TenancyConstants.DEFAULT_TENANT_ID)`. Actor identity is passed as parameters from the caller (matching the `writeSarOfficerReviewed(caseId, officerId, ...)` pattern — `AmlLedgerService` does not inject `CurrentPrincipal`).

**Base field values for entity erasure:**

| Field | Value | Rationale |
|-------|-------|-----------|
| `id` | `UUID.randomUUID()` | Unique entry identifier |
| `subjectId` | `UUID.nameUUIDFromBytes(...)` | Deterministic per entity (see below) |
| `tenancyId` | `TenancyConstants.DEFAULT_TENANT_ID` | Matches all existing `AmlLedgerService` methods; multi-tenant deferred to #84 |
| `sequenceNumber` | `nextSequenceNumber(subjectId)` | Sequential per subject |
| `entryType` | `LedgerEntryType.EVENT` | Matches other AML entries |
| `actorId` | caller-provided `actorId` | The compliance officer processing the GDPR request |
| `actorType` | caller-provided `actorType` | HUMAN for manual request, SYSTEM for automated |
| `actorRole` | `"GdprComplianceOfficer"` | Distinguishes from other actor roles |
| `occurredAt` | `Instant.now()` | Timestamp of erasure |

**Deterministic `subjectId`** (design choice): `UUID.nameUUIDFromBytes(("aml-entity-erasure:" + entityId).getBytes(UTF_8))` produces a stable UUID per entity. All erasure records for the same entity share a `subjectId`, enabling `LedgerEntryRepository.findBySubjectId()` to return the complete erasure history for an entity. This is intentional — the `subjectId` provides queryability, not uniqueness (the entry's `id` provides uniqueness).

**Cross-datasource ordering:** Memory erasure (default datasource) and ledger write (qhorus datasource) cannot share a transaction. Memory erasure runs first because the ledger receipt records `memoriesErased`. If the ledger write fails after memory erasure, GDPR compliance is achieved (data deleted) but the audit receipt is missing — the exception propagates to the caller, who can retry. The retry will produce `memoriesErased = 0` and a valid receipt.

**Retry interpretation:** When multiple erasure entries share the same deterministic `subjectId`, the first entry with `memoriesErased > 0` is the authoritative count. Subsequent entries with `memoriesErased = 0` indicate a retry-after-partial-failure (ledger write failed on first attempt), not a no-op. This is queryable via `LedgerEntryRepository.findBySubjectId()` using the deterministic UUID.

**Error handling:** If `CaseMemoryStore.eraseEntity()` throws, the exception propagates — no ledger entry is written, no partial state. The `NoOpCaseMemoryStore` (`@DefaultBean`) overrides `eraseEntity()` and returns 0, so it does not throw.

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

**Resource handler changes:**

`AmlLayer6Resource.getInvestigation()` — the non-COMPLETED path currently returns:
```java
new Layer6InvestigationResponse(caseId, r.status(), List.of(), null)
```
Changes to:
```java
new Layer6InvestigationResponse(caseId, r.status(), List.of(), null, r.failureContext())
```
The COMPLETED path changes to:
```java
new Layer6InvestigationResponse(caseId, r.status(), decisions, r.outcome(), null)
```

`AmlLayer9Resource.getInvestigation()` — currently returns `r.outcome()` unconditionally. Changes to:
```java
new Layer9InvestigationResponse(caseId, r.status(), r.outcome(), r.failureContext())
```

Both Layer6 and Layer9 pass through whatever `InvestigationResolution` provides — the invariant enforcement is in the service, not the resource.

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
    id UUID NOT NULL,
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
  - FAULTED case (goal-triggered): seed EventLog with one `CASE_FAULTED` (with `goalName`/`goalKind`) + `WORKER_EXECUTION_FAILED` → verify failureContext has triggerGoalName
  - FAULTED case (retries exhausted): seed TWO `CASE_FAULTED` entries — first at T1 with `{workerId, inputDataHash}`, second at T2 (T2 > T1) with `{oldStatus, newStatus}` — + one `WORKER_EXECUTION_FAILED` → verify `occurredAt == T1`, `triggerGoalName == null`
  - CANCELLED case: seed EventLog with `CASE_CANCELLED` → verify failureContext with occurredAt
  - SUSPENDED case: verify both outcome and failureContext are null
  - COMPLETED case: verify failureContext is null (backward compat)
- `AmlErasureServiceTest` additions:
  - `eraseEntity()` with memories present → verify count and receipt
  - `eraseEntity()` with no memories → verify `memoriesErased=0`, receipt still written
- `AmlLedgerService.noOp()` and `AmlLedgerService.stub(UUID)` — add `writeEntityErasure()` override returning no-op / fixed UUID (prevents NPE in tests using these stubs)

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
