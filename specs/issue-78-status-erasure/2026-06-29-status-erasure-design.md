# InvestigationStatus Exhaustive Projection + GDPR Erasure Receipt

Covers: #78 (InvestigationStatus FAULTED/CANCELLED), #62 (GDPR Art.17 erasure receipt)

## Problem

**#78:** `AmlInvestigationOutcomeService.resolveInvestigation()` maps any `CaseStatus != COMPLETED` to `InvestigationStatus.IN_PROGRESS`. Cases that are `FAULTED`, `CANCELLED`, or `SUSPENDED` report as "in-progress" — hiding compliance-significant distinctions. A faulted investigation is a coverage gap requiring remediation; a cancelled investigation is a deliberate stop requiring audit justification. These are fundamentally different regulatory scenarios.

**#62:** The GDPR erasure endpoint (`AmlGdprErasureResource`) delegates to `LedgerErasureService.erase()` without enabling erasure receipts. The compliance evidence endpoint reports static booleans for GDPR readiness rather than querying actual erasure state.

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

Replace the `if (instance.getState() != CaseStatus.COMPLETED)` branch with an exhaustive switch. Full updated method body:

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

    // No default arm — intentionally causes a compilation failure when
    // CaseStatus gains new values, forcing explicit mapping.
    InvestigationStatus status = switch (instance.getState()) {
        case STARTING, RUNNING, WAITING -> InvestigationStatus.IN_PROGRESS;
        case COMPLETED -> InvestigationStatus.COMPLETED;
        case FAULTED -> InvestigationStatus.FAILED;
        case CANCELLED -> InvestigationStatus.CANCELLED;
        case SUSPENDED -> InvestigationStatus.SUSPENDED;
    };

    if (status != InvestigationStatus.COMPLETED) {
        return Optional.of(new InvestigationResolution(status, null));
    }
    final InvestigationOutcome outcome = resolveOutcome(caseId);
    return Optional.of(new InvestigationResolution(InvestigationStatus.COMPLETED, outcome));
}
```

The no-default constraint must be preserved; adding `default -> IN_PROGRESS` would restore the silent-mapping problem this spec eliminates.

For non-COMPLETED terminal states (FAILED, CANCELLED), outcome is null — no SAR officer review occurred. The if/else structure after the switch preserves the existing conditional: only COMPLETED cases call `resolveOutcome()` to look up SAR officer review entries.

**Status semantics:** `InvestigationStatus` values divide into terminal and non-terminal:
- **Terminal:** `COMPLETED`, `FAILED`, `CANCELLED` — the investigation will not advance further
- **Non-terminal:** `IN_PROGRESS`, `SUSPENDED` — the investigation may still reach a terminal state

Callers must always inspect the `status` field to determine finality. The response shape (empty routing decisions, null outcome) is identical for all non-COMPLETED statuses, but the status value distinguishes "keep polling" (IN_PROGRESS, SUSPENDED) from "stop, this is dead" (FAILED, CANCELLED).

**Wire format:** `toWireFormat()` already does `name().toLowerCase().replace('_', '-')`, so new values serialize as `"failed"`, `"cancelled"`, `"suspended"`. No changes to `AmlJacksonConfig.InvestigationStatusMixin`.

**Layer 6/9 resources:** No changes needed — both already delegate to `AmlInvestigationOutcomeService.resolveInvestigation()` and return `InvestigationStatus` in their response records. The Layer 6 resource's `if (r.status() != InvestigationStatus.COMPLETED)` guard correctly skips routing decision queries for non-completed cases regardless of the specific non-completed status.

### Part 2: GDPR Erasure Receipt (#62)

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

    public AmlErasureResult erase(String actorId, ErasureReason reason) {
        ErasureResult ledgerResult = ledgerErasureService.erase(actorId, reason);
        return new AmlErasureResult(
            ledgerResult.rawActorId(),
            ledgerResult.mappingFound(),
            ledgerResult.affectedEntryCount(),
            ledgerResult.receiptEntryId().orElse(null));
    }
}
```

No memory erasure — `CaseMemoryStore.eraseEntity()` erases by entity ID (account IDs in AML), not by actor ID (ledger identities). Calling `eraseEntity(actorId, tenantId)` would silently return 0 for all cases, giving false assurance that memory cleanup occurred. Entity-level memory erasure (by account ID) is a distinct use case filed as a separate issue (see Not in Scope).

#### Domain result (api/ module)

New `AmlErasureResult` in `io.casehub.aml.compliance`:

```java
public record AmlErasureResult(
    String erasedActorId,
    boolean mappingFound,
    long affectedEntryCount,
    UUID receiptEntryId) {}
```

Flattened from the foundation `ErasureResult` — no nested records in the API. `receiptEntryId` is nullable (null when receipt feature is disabled, though it should always be non-null with AML's config). The `Optional<UUID>` from `ErasureResult.receiptEntryId()` is unwrapped via `.orElse(null)` in the mapping.

#### REST resource update

`AmlGdprErasureResource` delegates to `AmlErasureService` and returns `AmlErasureResult`:

```java
@POST
public AmlErasureResult eraseActor(@PathParam("actorId") String actorId) {
    return erasureService.erase(actorId, ErasureReason.GDPR_ART_17_REQUEST);
}
```

#### Repository extension

`ErasureReceiptRepository` gains a tenant-wide count method:

```java
long countByTenant(String tenancyId);
```

All three implementations:

`JpaErasureReceiptRepository`:

```java
@Override
public long countByTenant(String tenancyId) {
    return em.createQuery(
        "SELECT COUNT(e) FROM ErasureReceiptLedgerEntry e WHERE e.tenancyId = :tenancyId",
        Long.class)
        .setParameter("tenancyId", tenancyId)
        .getSingleResult();
}
```

`NoOpErasureReceiptRepository` (`@DefaultBean`):

```java
@Override
public long countByTenant(String tenancyId) {
    return 0L;
}
```

`InMemoryErasureReceiptRepository` (`@Alternative @Priority(1)`):

```java
@Override
public long countByTenant(String tenancyId) {
    return blocking.allEntries().stream()
            .filter(e -> e instanceof ErasureReceiptLedgerEntry)
            .filter(e -> tenancyId.equals(e.tenancyId))
            .count();
}
```

This keeps the count query in the SPI that owns erasure receipt reads, consistent with the existing `findByErasedActorId` method. All three implementations follow the same pattern as `findByErasedActorId`: JPA uses JPQL, NoOp returns empty/zero, InMemory filters the backing `allEntries()` store.

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

`MECHANISM` constant updated to reflect new capabilities:

```java
public static final String MECHANISM =
    "LedgerErasureService pseudonymizes actorId in ledger_entry rows via ActorIdentity token. " +
    "Audit entries remain intact; actor identity is replaced with an opaque token. " +
    "Tamper-evident ErasureReceiptLedgerEntry records each erasure in the Merkle chain.";
```

Status logic:
- `CLOSED` — tokenisation enabled AND receipt enabled (full capability stack active)
- `PARTIAL` — endpoint exists but tokenisation or receipts not enabled
- `GAP` — would only occur if LedgerErasureService is not on classpath (won't happen in AML)

`erasureReceiptCount` is the total number of `ErasureReceiptLedgerEntry` records in the tenant — queried via `ErasureReceiptRepository.countByTenant()`, not by actor (per GE-20260628-6599e6: post-tokenisation, actor-scoped queries return empty because the token-identity link is severed). Included for observability — does not affect CLOSED/PARTIAL/GAP logic.

`AmlComplianceEvidenceService` injection changes — add `LedgerConfig` and `ErasureReceiptRepository`:

```java
@Inject
public AmlComplianceEvidenceService(
        LedgerEntryRepository ledgerRepo,
        LedgerVerificationService verificationService,
        AmlTrustAttestationRepository attestationRepo,
        AmlWorkerDecisionRepository workerDecisionRepo,
        EntityManager em,
        AmlAttestationReconciler reconciler,
        LedgerConfig ledgerConfig,
        ErasureReceiptRepository erasureReceiptRepo) { ... }
```

`buildGdprErasure()` changes from returning static booleans to querying live config and receipt state:

```java
private GdprErasureRequirement buildGdprErasure() {
    boolean tokenisationEnabled = ledgerConfig.identity().tokenisation().enabled();
    boolean receiptEnabled = ledgerConfig.erasureReceipt().enabled();

    long receiptCount = 0L;
    try {
        receiptCount = erasureReceiptRepo.countByTenant(TenancyConstants.DEFAULT_TENANT_ID);
    } catch (Exception ignored) {
        // DB unreachable — degrade gracefully, count stays 0
    }

    RequirementStatus status;
    if (tokenisationEnabled && receiptEnabled) {
        status = RequirementStatus.CLOSED;
    } else if (tokenisationEnabled || receiptEnabled) {
        status = RequirementStatus.PARTIAL;
    } else {
        status = RequirementStatus.GAP;
    }

    return new GdprErasureRequirement(
            GdprErasureRequirement.REQUIREMENT_ID,
            GdprErasureRequirement.CITATION,
            GdprErasureRequirement.MECHANISM,
            status, tokenisationEnabled, receiptEnabled, receiptCount,
            GdprErasureRequirement.ERASURE_ENDPOINT);
}
```

The `countByTenant()` call is wrapped in try/catch to match the error resilience pattern of sibling methods (`buildAuditChain()` catches `IllegalStateException` from `verificationService.verify()`). Config flag reads (`LedgerConfig`) are local and infallible. Without this, a DB failure would take down the entire compliance evidence endpoint instead of degrading the GDPR section gracefully.

No PU conflict — `ErasureReceiptRepository` manages its own EntityManager injection (qhorus PU), same as `LedgerEntryRepository` which is already injected.

#### Flyway migration

`V1010__erasure_receipt_entry.sql` exists in `db/ledger/migration` (foundation-owned). AML's qhorus Flyway locations already include `classpath:db/ledger/migration`, so the `erasure_receipt_entry` join table is created automatically. No AML-side migration needed.

**Javadoc fix:** `LedgerConfig.ErasureReceiptConfig` javadoc incorrectly references "V1009 migration" — update to "V1010 migration" as part of this change.

## Not in Scope

- Entity data erasure (erasing memory about investigated subjects by account ID rather than actor ID) — different use case, different endpoint. **GitHub issue to be filed during implementation.**
- Automated retention expiry — `ErasureReason.RETENTION_EXPIRED` is available but no scheduled job exists. **GitHub issue to be filed during implementation.**
- Investigation failure context enrichment (fault reason, cancellation justification on `InvestigationResolution`) — available through engine's `CaseInstance` APIs; enrichment is a separate concern from status projection. **GitHub issue to be filed during implementation.**
- GDPR Art.22 decision record compliance supplements — separate Layer 7 concern. **GitHub issue to be filed during implementation.**

## Test Specification

### Part 1: InvestigationStatus projection

- **Exhaustive mapping:** test `resolveInvestigation()` for each `CaseStatus` value — verify correct `InvestigationStatus` mapping via the switch expression
- **Wire format roundtrip:** verify `toWireFormat()` / `fromWireFormat()` for FAILED, CANCELLED, SUSPENDED
- **Layer 6 response shape:** verify each non-COMPLETED status returns empty routing decisions and null outcome
- **Serialization mixin:** verify `InvestigationStatusMixin` correctly serializes all new enum values

### Part 2: GDPR erasure

- **AmlErasureService happy path:** ledger erasure succeeds — `AmlErasureResult` reflects all fields from `ErasureResult`
- **AmlErasureResult mapping:** verify `Optional<UUID>` to nullable `UUID` conversion for `receiptEntryId`
- **buildGdprErasure() config combinations:** test CLOSED (both enabled), PARTIAL (one enabled), GAP (neither)
- **buildGdprErasure() DB failure:** verify `countByTenant()` exception degrades gracefully — status still correct, `receiptCount` defaults to 0
- **Erasure receipt count:** verify `ErasureReceiptRepository.countByTenant()` returns correct count across all implementations (JPA, InMemory, NoOp)
- **REST endpoint:** integration test — POST returns `AmlErasureResult` with receipt ID when receipt enabled
- **MECHANISM constant:** verify updated text reflects new capabilities

## Garden Entries Applied

- **GE-20260531-46f8ab:** tokenisation flag for erasure tests — already globally enabled in AML test properties; no `@TestProfile` needed
- **GE-20260628-6599e6:** actor-scoped erasure receipt queries fail post-tokenisation — compliance evidence queries by tenant only

## Platform Coherence

- No new foundation types created — uses existing `ErasureReceiptLedgerEntry`, `LedgerErasureService`, `ErasureReceiptRepository`
- `InvestigationStatus` expansion follows the harness pattern (life: 3 values, clinical: 4, AML: 5)
- `AmlErasureService` follows the AML pattern of domain services coordinating foundation primitives
- `AmlErasureResult` follows the flattened-record API pattern used by other AML response types
- `ErasureReceiptRepository.countByTenant()` follows the SPI read-method pattern used by existing repository interfaces
