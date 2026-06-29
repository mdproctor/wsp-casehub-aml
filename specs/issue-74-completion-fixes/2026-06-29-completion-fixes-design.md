# Completion Detection Consolidation, Rejection Reason Capture, and Test Gaps

**Issues:** #74, #75, #77
**Branch:** issue-74-completion-fixes
**Date:** 2026-06-29

---

## Problem

The investigation status model is split across two foundation modules (engine case lifecycle + ledger audit) with no domain-level abstraction bridging them. Both Layer 6 and Layer 9 resources independently bridge this gap by duplicating identical completion detection logic (cache → repo → CaseStatus.COMPLETED). Both return 200 with `"in-progress"` for nonexistent caseIds — a bug. The officer's rejection reason is captured in the WorkItem lifecycle event but discarded by the observer. Several test gaps exist in the recently committed review corrections.

## Approach

Create the missing domain abstraction (`InvestigationResolution`) that ties engine completion status to ledger audit outcome. Consolidate into `AmlInvestigationOutcomeService`. Add rejection reason capture end-to-end. Fill test gaps.

Implementation order: #74 (consolidation) → #75 (rejection reason) → #77 (tests).

---

## §1 Domain Types (api/ module)

### §1.1 `InvestigationStatus` enum (new)

Package: `io.casehub.aml.domain`

```java
public enum InvestigationStatus {
    IN_PROGRESS,
    COMPLETED;

    @JsonValue
    public String toJson() {
        return name().toLowerCase().replace('_', '-');
    }

    @JsonCreator
    public static InvestigationStatus fromJson(String value) {
        return valueOf(value.toUpperCase().replace('-', '_'));
    }
}
```

Replaces raw `String status` across all response types.

### §1.2 `InvestigationResolution` record (new)

Package: `io.casehub.aml.domain`

```java
public record InvestigationResolution(
        InvestigationStatus status,
        InvestigationOutcome outcome) {}
```

`outcome` is null when in-progress or when case is completed but officer hasn't reviewed yet.

### §1.3 `InvestigationOutcome` record (modified)

Add `reason` component:

```java
public record InvestigationOutcome(String type, String reason) {

    public static InvestigationOutcome fromReviewDecision(
            final String reviewDecision, final String rejectionReason) {
        Objects.requireNonNull(reviewDecision,
                "reviewDecision must not be null — column is NOT NULL");
        return switch (reviewDecision) {
            case "APPROVED" -> new InvestigationOutcome("sar-filed", null);
            case "REJECTED" -> new InvestigationOutcome("gate-rejected", rejectionReason);
            case "UNKNOWN" -> new InvestigationOutcome("decision-not-recorded", null);
            default -> throw new IllegalStateException(
                    "Unexpected reviewDecision: " + reviewDecision);
        };
    }
}
```

Changes from current:
- Added `String reason` component (nullable — only for gate-rejected)
- Factory takes two parameters instead of one
- Null `reviewDecision` throws NPE instead of returning null (column is NOT NULL — null is data corruption)

---

## §2 Service Consolidation

### §2.1 `AmlInvestigationOutcomeService` (modified)

New dependencies (injected):
- `CaseInstanceCache`
- `CaseInstanceRepository`

New public method:

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
    if (instance.getState() != CaseStatus.COMPLETED) {
        return Optional.of(new InvestigationResolution(InvestigationStatus.IN_PROGRESS, null));
    }
    final InvestigationOutcome outcome = resolveOutcome(caseId);
    return Optional.of(new InvestigationResolution(InvestigationStatus.COMPLETED, outcome));
}
```

Existing `resolve()` renamed to `resolveOutcome()`, made package-private. Internal logic unchanged except `fromReviewDecision` call gains `rejectionReason` parameter.

---

## §3 Rejection Reason Capture (#75)

### §3.1 `AmlSarOfficerReviewedLedgerEntry` (modified)

```java
@Column(name = "rejection_reason", length = 1000)
public String rejectionReason;
```

`domainContentBytes()` updated to pipe-delimited: `reviewDecision|rejectionReason`.

### §3.2 Flyway V2012

File: `app/src/main/resources/db/aml-ledger/migration/V2012__aml_sar_officer_reviewed_rejection_reason.sql`

```sql
ALTER TABLE aml_sar_officer_reviewed_ledger_entry
    ADD COLUMN rejection_reason VARCHAR(1000);
```

Nullable column — no backfill for existing rows.

### §3.3 `AmlWorkItemLifecycleObserver` (modified)

Capture `event.detail()` for REJECTED events:

```java
final String rejectionReason = event.status() == WorkItemStatus.REJECTED
        ? event.detail() : null;
ledgerService.writeSarOfficerReviewed(caseId, officerId, reviewDecision, rejectionReason);
```

### §3.4 `AmlLedgerService` (modified)

Both `writeSarOfficerReviewed()` and `writeSarOfficerReviewedFailure()` gain a `String rejectionReason` parameter. Set `entry.rejectionReason = rejectionReason` before save.

### §3.5 Upstream verification

Confirmed: `WorkItemService.reject()` passes `reason` as the `detail` parameter to `WorkItemLifecycleEvent.of()`. The officer's rejection reason arrives in `event.detail()`. No upstream changes needed.

---

## §4 Resource Changes

### §4.1 `Layer9InvestigationResponse` (new)

Package: `io.casehub.aml.engine`

```java
public record Layer9InvestigationResponse(
        UUID caseId,
        InvestigationStatus status,
        InvestigationOutcome outcome) {}
```

Replaces the HashMap serialization in Layer 9.

### §4.2 `Layer6InvestigationResponse` (modified)

`String status` → `InvestigationStatus status`. Breaking record signature change — all construction sites update.

### §4.3 `AmlLayer6Resource.getInvestigation()` (simplified)

```java
final Optional<InvestigationResolution> resolution =
        outcomeService.resolveInvestigation(caseId);
if (resolution.isEmpty()) {
    return Response.status(Response.Status.NOT_FOUND).build();
}
// ... build Layer6InvestigationResponse from resolution + routing decisions
```

Removes duplicated completion detection. Layer 6-specific logic (worker decisions, trust scores) remains in the resource.

### §4.4 `AmlLayer9Resource.getInvestigation()` (simplified)

```java
final Optional<InvestigationResolution> resolution =
        outcomeService.resolveInvestigation(caseId);
if (resolution.isEmpty()) {
    return Response.status(Response.Status.NOT_FOUND).build();
}
final InvestigationResolution r = resolution.get();
return Response.ok(new Layer9InvestigationResponse(caseId, r.status(), r.outcome())).build();
```

Removes duplicated completion detection and HashMap.

### §4.5 Behavioral change

Nonexistent caseIds now return 404 instead of 200 with `"in-progress"`. This is a bug fix, not a breaking change — the old behavior was misleading.

---

## §5 Test Changes (#77)

### §5.1 `AmlInvestigationOutcomeServiceTest` (unit test)

1. **Move package:** `io.casehub.aml.engine` → `io.casehub.aml.compliance` (matches class under test; enables package-private `resolveOutcome()` access)

2. **sequenceNumber tiebreaker test:** Two HUMAN entries with different `sequenceNumber` values — verify highest wins. Fix `officerEntry()` helper to accept `int sequenceNumber`.

3. **Failure-only scenario test:** Single SYSTEM entry with `reviewDecision = "REJECTED"` → outcome type `gate-rejected`. Exercises correction 4 path.

4. **Update `officerEntry()` helper:** Accept `String rejectionReason` parameter for the two-arg factory.

### §5.2 `InvestigationOutcomeTest` (api/ unit test)

- Update for two-arg factory
- Add test: `reason` populated for `gate-rejected`
- Add test: `reason` null for `sar-filed`
- Add test: null `reviewDecision` → `NullPointerException`

### §5.3 `AmlLayer9ResourceTest` (integration test)

Two new tests:

1. **`officer_approval_surfaces_sar_filed_outcome`:** Start via Layer 9, await gate, complete officer review, assert `outcome.type == "sar-filed"`.

2. **`officer_rejection_surfaces_gate_rejected_outcome`:** Same flow, reject review with reason text, assert `outcome.type == "gate-rejected"` and `outcome.reason` carries the rejection text.

Both use `awaitAndApproveGate()` pattern per GE-20260628-dbc656 (gate approval before attestation wait).

### §5.4 `AmlWorkItemLifecycleObserverTest` (unit test)

Update existing tests for the 4-arg `writeSarOfficerReviewed()` signature. Add test verifying rejection reason is captured from event detail.

### §5.5 Test ordering (GE-20260628-dbc656)

Layer 9 tests with oversight gates must call `awaitAndApproveGate()` BEFORE waiting for attestations. `WorkerDecisionEvent` fires at worker completion, not dispatch — gated workers are not "complete" until the gate is approved.

---

## Files Changed

| File | Change | Issue |
|------|--------|-------|
| `api/.../InvestigationStatus.java` | New enum | #74 |
| `api/.../InvestigationResolution.java` | New record | #74 |
| `api/.../InvestigationOutcome.java` | Add reason, 2-arg factory, drop null handling | #74, #75 |
| `app/.../AmlInvestigationOutcomeService.java` | Add resolveInvestigation(), new deps | #74 |
| `app/.../AmlSarOfficerReviewedLedgerEntry.java` | Add rejectionReason field | #75 |
| `app/.../V2012__*.sql` | Add rejection_reason column | #75 |
| `app/.../AmlWorkItemLifecycleObserver.java` | Capture event.detail() | #75 |
| `app/.../AmlLedgerService.java` | 4-arg write methods | #75 |
| `app/.../Layer6InvestigationResponse.java` | String → InvestigationStatus | #74 |
| `app/.../Layer9InvestigationResponse.java` | New record | #74 |
| `app/.../AmlLayer6Resource.java` | Use resolveInvestigation(), 404 | #74 |
| `app/.../AmlLayer9Resource.java` | Use resolveInvestigation(), typed response, 404 | #74 |
| `app/test/.../AmlInvestigationOutcomeServiceTest.java` | Move package, add 3 tests | #77 |
| `api/test/.../InvestigationOutcomeTest.java` | Update for 2-arg factory | #77 |
| `app/test/.../AmlLayer9ResourceTest.java` | Add 2 outcome integration tests | #77 |
| `app/test/.../AmlWorkItemLifecycleObserverTest.java` | Update for 4-arg signature | #77 |

## Out of Scope

- Refactoring `AmlLedgerService` internal duplication between `writeSarOfficerReviewed` and `writeSarOfficerReviewedFailure`
- `AmlLedgerService` no-op/stub helpers — update their signatures mechanically
- Sealed interface for `InvestigationOutcome` — evaluated and rejected; factory prevents invalid construction, no callers pattern-match in Java
