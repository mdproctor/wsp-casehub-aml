# Layer 2: casehub-work — Compliance Officer WorkItem with 30-day FinCEN SLA

**Date:** 2026-05-13  
**Issue:** casehubio/aml#15  
**Epic:** casehubio/aml#9 (Tutorial layers 1–7)  
**Builds on:** Layer 1 (casehubio/aml#12, closed)

---

## Context

Layer 1 established a naive Java baseline with no accountability, no deadline tracking, and no audit trail. Four `LAYER 1 GAP` comments mark the missing capabilities. Layer 2 closes the deadline-tracking gap: after the SAR narrative is drafted, a compliance officer must review and file it within 30 days (FinCEN requirement). Layer 2 introduces `casehub-work` to create a formal `WorkItem` representing that review obligation.

This is a production-quality implementation. The tutorial is derived from the code, not the other way around.

---

## Architecture

Layer 2 follows hexagonal architecture as the module boundary rule:

- **Domain layer (`api/`)** — pure Java, zero external dependencies. Owns domain records and specialist service interfaces. Gains one new result record.
- **Application layer (`app/`)** — use-case orchestration. Owns the top-level service interface and both implementations. CDI `@DefaultBean` makes the naive implementation the fallback; the Layer 2 implementation displaces it automatically.
- **Infrastructure layer (`app/`)** — REST adapter. Injects the application service interface; CDI resolves to the Layer 2 implementation.

Protocol: `PP-20260512-9b8847` — use-case orchestration lives in `app/`, `api/` stays pure domain (raised to casehubio/parent#18).

---

## Module Changes

### `api/` — one addition, no new dependencies

**New:** `io.casehub.aml.domain.AmlInvestigationResult`

```java
public record AmlInvestigationResult(
    InvestigationSummary summary,
    String complianceReviewTaskId   // null when no WorkItem created
) {}
```

`api/pom.xml` — no changes. Zero new dependencies.

---

### `app/` — new interface, two implementations, updated resource

**New:** `io.casehub.aml.AmlInvestigationApplicationService` (interface)

```java
public interface AmlInvestigationApplicationService {
    AmlInvestigationResult investigate(SuspiciousTransaction transaction);
}
```

**Refactored:** `NaiveAmlInvestigationService`

- Now `@ApplicationScoped @DefaultBean`
- Implements `AmlInvestigationApplicationService`
- Returns `new AmlInvestigationResult(summary, null)` — no WorkItem
- Four specialist stubs unchanged

**New:** `io.casehub.aml.tutorial.WorkItemAmlInvestigationService`

- `@ApplicationScoped` — displaces the `@DefaultBean` automatically; no config switch
- Implements `AmlInvestigationApplicationService`
- Injects `WorkItemService` from casehub-work
- Calls the same four naive specialist stubs (no change to investigation logic)
- After SAR narrative is drafted, creates a WorkItem:
  - `title`: `"Compliance review — SAR for transaction {transactionId}"`
  - `candidateGroups`: `["compliance-officers"]`
  - `claimDeadline`: `Instant.now().plus(30, ChronoUnit.DAYS)`
  - `callerRef`: `"aml:investigation/{transactionId}"`
- Returns `new AmlInvestigationResult(summary, workItem.getId().toString())`

**Updated:** `AmlInvestigationResource`

- Inject `AmlInvestigationApplicationService` (CDI resolves to `WorkItemAmlInvestigationService`)
- Return type changes from `InvestigationSummary` to `AmlInvestigationResult`

---

## REST Contract

**Request:** unchanged — `POST /api/investigations` with `SuspiciousTransaction` body.

**Response** (Layer 2):
```json
{
  "summary": {
    "transaction": { "id": "TXN-001", ... },
    "entityResolution": { ... },
    "patternAnalysis": { ... },
    "osintScreening": { ... },
    "sarNarrative": "..."
  },
  "complianceReviewTaskId": "a1b2c3d4-e5f6-..."
}
```

This is a breaking change from Layer 1 (fields previously at root now under `summary`). Intentional — Layer 2 changes the contract.

---

## WorkItem Details

| Field | Value |
|---|---|
| `title` | `"Compliance review — SAR for transaction {transactionId}"` |
| `candidateGroups` | `["compliance-officers"]` |
| `claimDeadline` | `Instant.now().plus(30, ChronoUnit.DAYS)` |
| `callerRef` | `"aml:investigation/{transactionId}"` (opaque to casehub-work) |

No Flyway migration needed. casehub-work ships its own migrations; `io.casehub.work.runtime.model` is already in the Hibernate scan packages in both `application.properties` and test config.

---

## CDI Wiring

`@DefaultBean` (Quarkus extension to CDI): a `@DefaultBean` is only activated when no other bean of the same type exists. Since both implementations are always present, CDI always picks `WorkItemAmlInvestigationService`. `NaiveAmlInvestigationService` remains compilable and independently testable but is not the active path.

No configuration switches. No `@Alternative @Priority`. The Layer 2 bean simply exists and wins.

---

## Testing

### Unit tests (pure Java, no Quarkus)

**`WorkItemAmlInvestigationServiceTest`** (new):
- Stub `WorkItemService` with an in-memory implementation
- `investigate()` returns `AmlInvestigationResult` with non-null `complianceReviewTaskId`
- `WorkItemService.create()` called with `claimDeadline` within the 30-day window
- `candidateGroups` contains `"compliance-officers"`
- `callerRef` contains the transaction ID

**`NaiveAmlInvestigationServiceTest`** (update):
- Existing three tests pass unchanged
- Add: `investigate()` returns `complianceReviewTaskId == null`

### `@QuarkusTest` (full CDI + H2)

**`AmlInvestigationResourceTest`** (update):
- Update existing path assertions: `transaction.id` → `summary.transaction.id`, etc.
- Add: `complianceReviewTaskId` is non-null and a valid UUID string
- Add: `GET /workitems/{complianceReviewTaskId}` returns 200 with `claimDeadline` in the 30-day window and `candidateGroups` containing `"compliance-officers"`

### Maven scope

```bash
mvn verify -pl api,app -am -Dsurefire.failIfNoSpecifiedTests=false
```

---

## What This Does NOT Do

- No Flyway migration — casehub-work manages its own schema
- No Testcontainers — no new AML migrations to dialect-validate
- No integration-tests module — deferred to a later layer
- No change to specialist service implementations — investigation logic is identical to Layer 1
- No change to `api/` dependencies
