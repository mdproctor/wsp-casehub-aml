# InvestigationStatus Exhaustive Projection + GDPR Erasure Receipt

Covers: #78 (InvestigationStatus FAULTED/CANCELLED), #62 (GDPR Art.17 erasure receipt)

## Problem

**#78:** `AmlInvestigationOutcomeService.resolveInvestigation()` maps any `CaseStatus != COMPLETED` to `InvestigationStatus.IN_PROGRESS`. Cases that are `FAULTED`, `CANCELLED`, or `SUSPENDED` report as "in-progress" — hiding compliance-significant distinctions. A faulted investigation is a coverage gap requiring remediation; a cancelled investigation is a deliberate stop requiring audit justification. These are fundamentally different regulatory scenarios.

**#62:** The GDPR erasure endpoint (`AmlGdprErasureResource`) delegates to `LedgerErasureService.erase()` without enabling erasure receipts or cleaning up CaseMemoryStore data. The compliance evidence endpoint reports static booleans for GDPR readiness rather than querying actual erasure state.

## Design

### Part 1: InvestigationStatus Exhaustive Projection (#78)

`InvestigationStatus` remains a query-time projection of `CaseStatus` — not a lifecycle with stored state. The projection becomes exhaustive: every `CaseStatus` value maps to exactly one `InvestigationStatus` value via a switch expression that fails to compile if `CaseStatus` gains new values.

**Enum expansion** (api/ module, `io.casehub.aml.domain`):

| InvestigationStatus | CaseStatus source | Regulatory meaning |
|---|---|---|
| `IN_PROGRESS` | STARTING, RUNNING, WAITING | SLA clock running |
| `COMPLETED` | COMPLETED | Outcome available |
| `FAILED` | FAULTED | System error — compliance gap |
| `CANCELLED` | CANCELLED | Deliberate stop — audit justification required |
| `SUSPENDED` | SUSPENDED | Administrative pause |

**Mapping** (app/ module, `AmlInvestigationOutcomeService.resolveInvestigation()`):

Replace the `if (instance.getState() != CaseStatus.COMPLETED)` check with:

```java
InvestigationStatus status = switch (instance.getState()) {
    case STARTING, RUNNING, WAITING -> InvestigationStatus.IN_PROGRESS;
    case COMPLETED -> InvestigationStatus.COMPLETED;
    case FAULTED -> InvestigationStatus.FAILED;
    case CANCELLED -> InvestigationStatus.CANCELLED;
    case SUSPENDED -> InvestigationStatus.SUSPENDED;
};
```

For non-COMPLETED terminal states (FAILED, CANCELLED), outcome is null — no SAR officer review occurred. `InvestigationResolution(status, outcome)` handles this correctly without changes.

**Wire format:** `toWireFormat()` already does `name().toLowerCase().replace('_', '-')`, so new values serialize as `"failed"`, `"cancelled"`, `"suspended"`. No changes to `AmlJacksonConfig.InvestigationStatusMixin`.

**Layer 6/9 resources:** No changes needed — both already delegate to `AmlInvestigationOutcomeService.resolveInvestigation()` and return `InvestigationStatus` in their response records. The Layer 6 resource's `if (r.status() != InvestigationStatus.COMPLETED)` guard correctly skips routing decision queries for non-completed cases regardless of the specific non-completed status.

### Part 2: GDPR Erasure Receipt + Memory Cleanup (#62)

#### Configuration

Enable foundation erasure receipts in both `application.properties` files:

```properties
casehub.ledger.erasure-receipt.enabled=true
```

Activate `JpaErasureReceiptRepository` in test `selected-alternatives`:

```
io.casehub.ledger.runtime.repository.jpa.JpaErasureReceiptRepository
```

No `@TestProfile` needed — tokenisation is already globally enabled (`casehub.ledger.identity.tokenisation.enabled=true`) in test properties.

#### Domain service (app/ module)

New `AmlErasureService` in `io.casehub.aml.compliance`:

```java
@ApplicationScoped
public class AmlErasureService {
    @Inject LedgerErasureService ledgerErasureService;
    @Inject CaseMemoryStore memoryStore;
    @Inject CurrentPrincipal principal;

    public AmlErasureResult erase(String actorId, ErasureReason reason) {
        ErasureResult ledgerResult = ledgerErasureService.erase(actorId, reason);
        int memoryCount = eraseMemory(actorId);
        return new AmlErasureResult(ledgerResult, memoryCount);
    }

    private int eraseMemory(String actorId) {
        try {
            return memoryStore.eraseEntity(actorId, principal.tenancyId());
        } catch (Exception e) {
            // Memory erasure failure must not block ledger erasure response
            return -1;
        }
    }
}
```

Memory erasure failures return -1 (attempted but failed) — the ledger erasure is the compliance-critical operation and must not be blocked by memory store issues. This follows AmlMemoryService's pattern where memory failures never propagate.

#### Domain result (api/ module)

New `AmlErasureResult` in `io.casehub.aml.compliance`:

```java
public record AmlErasureResult(
    String erasedActorId,
    boolean mappingFound,
    long affectedEntryCount,
    UUID receiptEntryId,
    int memoryEntriesErased) {}
```

Flattened from the foundation `ErasureResult` — no nested records in the API. `receiptEntryId` is nullable (null when receipt feature is disabled, though it should always be non-null with AML's config). `memoryEntriesErased` is -1 if memory erasure failed.

#### REST resource update

`AmlGdprErasureResource` delegates to `AmlErasureService` and returns `AmlErasureResult`:

```java
@POST
public AmlErasureResult eraseActor(@PathParam("actorId") String actorId) {
    return erasureService.erase(actorId, ErasureReason.GDPR_ART_17_REQUEST);
}
```

#### Compliance evidence enhancement

`GdprErasureRequirement` gains dynamic state:

```java
public record GdprErasureRequirement(
    String id,
    String citation,
    String mechanism,
    RequirementStatus status,
    boolean tokenisationEnabled,
    boolean erasureReceiptEnabled,
    long erasureReceiptCount,
    String erasureEndpoint) {}
```

Status logic:
- `CLOSED` — tokenisation enabled AND receipt enabled (full capability stack active)
- `PARTIAL` — endpoint exists but tokenisation or receipts not enabled
- `GAP` — would only occur if LedgerErasureService is not on classpath (won't happen in AML)

`erasureReceiptCount` is the total number of `ErasureReceiptLedgerEntry` records in the tenant — queried by tenant only, not by actor (per GE-20260628-6599e6: post-tokenisation, actor-scoped queries return empty because the token-identity link is severed).

`AmlComplianceEvidenceService.buildGdprErasure()` changes from returning static booleans to querying:
1. `LedgerConfig` for tokenisation and receipt enabled flags
2. `LedgerEntryRepository` for erasure receipt count by tenant (filter by dtype)

#### Flyway migration

`V1010__erasure_receipt_entry.sql` exists in `db/ledger/migration` (foundation-owned). AML's qhorus Flyway locations already include `classpath:db/ledger/migration`, so the `erasure_receipt_entry` join table is created automatically. No AML-side migration needed.

## Not in Scope

- Entity data erasure (erasing memory about investigated subjects by account ID rather than actor ID) — different use case, different endpoint, file as a future issue
- Automated retention expiry — `ErasureReason.RETENTION_EXPIRED` is available but no scheduled job exists
- GDPR Art.22 decision record compliance supplements — separate Layer 7 concern

## Garden Entries Applied

- **GE-20260531-46f8ab:** tokenisation flag for erasure tests — already globally enabled in AML test properties; no `@TestProfile` needed
- **GE-20260628-6599e6:** actor-scoped erasure receipt queries fail post-tokenisation — compliance evidence queries by tenant only

## Platform Coherence

- No new foundation types created — uses existing `ErasureReceiptLedgerEntry`, `LedgerErasureService`, `CaseMemoryStore.eraseEntity()`
- `InvestigationStatus` expansion follows the harness pattern (life: 3 values, clinical: 4, AML: 5)
- `AmlErasureService` follows the AML pattern of domain services coordinating foundation primitives
- `AmlErasureResult` follows the flattened-record API pattern used by other AML response types
