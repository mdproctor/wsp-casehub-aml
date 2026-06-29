# Completion Detection Consolidation, Rejection Reason Capture, and Test Gaps

**Issues:** #74, #73/#75, #77
**Branch:** issue-74-completion-fixes
**Date:** 2026-06-29

> **Note:** #73 and #75 are duplicates (both capture rejection reason from #71 deferred work). This spec covers the full scope of both. Close #73 as duplicate when #75 is delivered.

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

    public String toWireFormat() {
        return name().toLowerCase().replace('_', '-');
    }

    public static InvestigationStatus fromWireFormat(String value) {
        return valueOf(value.toUpperCase().replace('-', '_'));
    }
}
```

No Jackson annotations — the `api/` module has zero framework dependencies (ARC42STORIES §5, §8). JSON binding is handled by a mixin registered in `AmlJacksonConfig` (same registration mechanism as `SpecialistOutcome` — `mapper.addMixIn()` — but different annotations: `@JsonValue`/`@JsonCreator` for enum value mapping vs `@JsonTypeInfo`/`@JsonSubTypes` for polymorphic type discrimination):

```java
// In AmlJacksonConfig:
abstract class InvestigationStatusMixin {
    @JsonValue abstract String toWireFormat();
    @JsonCreator static InvestigationStatus fromWireFormat(String value) { return null; }
}

@Override
public void customize(ObjectMapper mapper) {
    mapper.addMixIn(SpecialistOutcome.class, SpecialistOutcomeMixin.class);
    mapper.addMixIn(InvestigationStatus.class, InvestigationStatusMixin.class);
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

**Package move:** `io.casehub.aml.compliance` → `io.casehub.aml.engine`. The consolidated service now composes engine state (cache/repo) with compliance outcome (ledger query). Both callers (`AmlLayer6Resource`, `AmlLayer9Resource`) are in `engine/`. The engine SPI deps (`CaseInstanceCache`, `CaseInstanceRepository`) are natural in this package. Keeping the service in `compliance/` would leak engine dependencies into a package that currently depends only on ledger.

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

Existing `resolve()` renamed to `resolveOutcome()`, made package-private. Updated to pass `rejectionReason` through to the two-arg factory:

```java
InvestigationOutcome resolveOutcome(final UUID caseId) {
    return ledgerEntryRepository
            .findBySubjectId(caseId, TenancyConstants.DEFAULT_TENANT_ID).stream()
            .filter(AmlSarOfficerReviewedLedgerEntry.class::isInstance)
            .map(AmlSarOfficerReviewedLedgerEntry.class::cast)
            .min(HUMAN_FIRST_LATEST_SEQ)
            .map(e -> InvestigationOutcome.fromReviewDecision(e.reviewDecision, e.rejectionReason))
            .orElse(null);
}
```

Data flow: `AmlSarOfficerReviewedLedgerEntry.rejectionReason` (§3.1) → `InvestigationOutcome.fromReviewDecision()` (§1.3) → `InvestigationOutcome.reason`.

---

## §3 Rejection Reason Capture (#75)

### §3.1 `AmlSarOfficerReviewedLedgerEntry` (modified)

```java
@Column(name = "rejection_reason", length = 1000)
public String rejectionReason;
```

`domainContentBytes()` updated with backward-compatible logic. Existing entries (where `rejectionReason` is null) must produce identical bytes to the pre-change implementation to preserve Merkle hash verification:

```java
@Override
protected byte[] domainContentBytes() {
    if (rejectionReason == null) {
        return (reviewDecision != null ? reviewDecision : "")
                .getBytes(java.nio.charset.StandardCharsets.UTF_8);
    }
    return String.join("|",
            reviewDecision != null ? reviewDecision : "",
            rejectionReason).getBytes(java.nio.charset.StandardCharsets.UTF_8);
}
```

The pipe-delimited format is used only for new entries that carry a rejection reason. Entries without one produce the same bytes as before the change.

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

**Return type change:** `Layer6InvestigationResponse` → `Response`. The current method signature `public Layer6InvestigationResponse getInvestigation(...)` cannot return a 404 `Response` object. The signature changes to `public Response getInvestigation(...)`. The happy path wraps the body in `Response.ok(...)`. All test assertions change from direct body assertions to `Response`-wrapped assertions.

**Removed injections:** `CaseInstanceCache` and `CaseInstanceRepository` are removed from this resource — completion detection is now in `AmlInvestigationOutcomeService`. Only `coordinator`, `outcomeService`, `workerDecisionRepo`, `sarOutcomeEvent`, and `trustScoreSource` remain.

```java
public Response getInvestigation(@PathParam("caseId") UUID caseId) {
    final Optional<InvestigationResolution> resolution =
            outcomeService.resolveInvestigation(caseId);
    if (resolution.isEmpty()) {
        return Response.status(Response.Status.NOT_FOUND).build();
    }
    final InvestigationResolution r = resolution.get();
    if (r.status() != InvestigationStatus.COMPLETED) {
        return Response.ok(new Layer6InvestigationResponse(
                caseId, r.status(), List.of(), null)).build();
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
            caseId, r.status(), decisions, r.outcome())).build();
}
```

Routing decisions and trust scores are only queried for completed cases — in-progress cases return `List.of()` for decisions and `null` for outcome, avoiding an unnecessary database hit to `AmlWorkerDecisionRepository`.

### §4.4 `AmlLayer9Resource.getInvestigation()` (simplified)

**Removed injections:** `CaseInstanceCache` and `CaseInstanceRepository` are removed — completion detection is now in `AmlInvestigationOutcomeService`. Only `coordinator` and `outcomeService` remain.

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

Test stays in `io.casehub.aml.engine` — matches the class under test after the §2.1 package move. `resolveOutcome()` is package-private and accessible from the test.

**Update `serviceWith()` helper:** The constructor gains `CaseInstanceCache` and `CaseInstanceRepository` parameters. The helper must be updated to pass all three dependencies. For `resolveOutcome()` tests, the cache and repo can be null (those tests only exercise the ledger query path). For `resolveInvestigation()` tests, stub implementations are needed.

**Existing test updates:**

1. **sequenceNumber tiebreaker test:** Two HUMAN entries with different `sequenceNumber` values — verify highest wins. Fix `officerEntry()` helper to accept `int sequenceNumber`.

2. **Failure-only scenario test:** Single SYSTEM entry with `reviewDecision = "REJECTED"` → outcome type `gate-rejected`. Exercises correction 4 path.

3. **Update `officerEntry()` helper:** Accept `String rejectionReason` parameter for the two-arg factory.

**New `resolveInvestigation()` unit tests:**

4. **Cache hit, completed → COMPLETED with outcome:** Stub cache to return a COMPLETED `CaseInstance`, verify `resolveInvestigation()` returns `InvestigationResolution(COMPLETED, outcome)`.

5. **Cache hit, not completed → IN_PROGRESS:** Stub cache to return a non-completed `CaseInstance`, verify returns `InvestigationResolution(IN_PROGRESS, null)`.

6. **Cache miss, repo hit → delegates to repo:** Stub cache to return null, stub repo to return a COMPLETED `CaseInstance`, verify returns COMPLETED.

7. **Cache miss, repo miss → empty (404):** Both cache and repo return null, verify `Optional.empty()`.

### §5.2 `InvestigationOutcomeTest` and `InvestigationStatusTest` (api/ unit tests)

**InvestigationOutcomeTest:**
- Update for two-arg factory
- Add test: `reason` populated for `gate-rejected`
- Add test: `reason` null for `sar-filed`
- **Replace** existing `null_input_returns_null` test with: null `reviewDecision` → `NullPointerException` (the existing test asserts `assertNull(fromReviewDecision(null))` — contradicts the new NPE behavior)

**InvestigationStatusTest (new, api/):**
- `IN_PROGRESS` → `toWireFormat()` returns `"in-progress"`
- `COMPLETED` → `toWireFormat()` returns `"completed"`
- `fromWireFormat("in-progress")` → `IN_PROGRESS`
- `fromWireFormat("completed")` → `COMPLETED`

**InvestigationStatusMixinTest (new, app/):**

Tests the actual Jackson mixin wiring, not just the Java methods the mixin delegates to. A mixin misconfiguration (wrong method signature, missing `addMixIn` call) would pass the api/ unit tests while silently serializing `IN_PROGRESS` as `"IN_PROGRESS"` in production.

```java
@Test
void jackson_serializes_via_mixin() throws Exception {
    ObjectMapper mapper = new ObjectMapper();
    new AmlJacksonConfig().customize(mapper);
    assertEquals("\"in-progress\"", mapper.writeValueAsString(InvestigationStatus.IN_PROGRESS));
    assertEquals("\"completed\"", mapper.writeValueAsString(InvestigationStatus.COMPLETED));
}

@Test
void jackson_deserializes_via_mixin() throws Exception {
    ObjectMapper mapper = new ObjectMapper();
    new AmlJacksonConfig().customize(mapper);
    assertEquals(InvestigationStatus.IN_PROGRESS, mapper.readValue("\"in-progress\"", InvestigationStatus.class));
    assertEquals(InvestigationStatus.COMPLETED, mapper.readValue("\"completed\"", InvestigationStatus.class));
}
```

### §5.3 `AmlLayer6ResourceTest` and `AmlLayer9ResourceTest` (integration tests)

**404 regression tests (both resources):**

The §4.5 behavioral change (nonexistent caseIds return 404 instead of 200 with `"in-progress"`) is the bug fix that motivated #74. Neither test class currently GETs a nonexistent caseId — all existing tests create a case first. Both resources need a regression test:

```java
@Test
void get_nonexistent_investigation_returns_404() {
    given().when().get("/api/layer6/investigations/" + UUID.randomUUID())
            .then().statusCode(404);
}
```

Same test in `AmlLayer9ResourceTest` with the `/api/layer9/investigations/` path.

**Layer 9 test infrastructure:** `AmlLayer9ResourceTest` currently has no gate-approval or work-item helpers. The following must be added (duplicated from `AmlLayer6ResourceTest` — not extracted to a shared utility, since the tests target different URL paths and response shapes):
- `@PersistenceContext EntityManager defaultEm`
- `@Inject WorkItemService workItemService`
- `awaitAndApproveGate(UUID caseId)` — waits for gate work items, approves the first
- `findGateWorkItems(UUID caseId)` — queries gate work items by callerRef pattern
- `findComplianceReviewWorkItem(UUID caseId)` — queries compliance review work item by callerRef

**Layer 9 new outcome tests:**

1. **`officer_approval_surfaces_sar_filed_outcome`:** Start via Layer 9, await gate, complete officer review, assert `outcome.type == "sar-filed"`.

2. **`officer_rejection_surfaces_gate_rejected_outcome`:** Same flow, reject review with reason text, assert `outcome.type == "gate-rejected"` and `outcome.reason` carries the rejection text.

Both use `awaitAndApproveGate()` pattern per GE-20260628-dbc656 (gate approval before attestation wait).

### §5.4 `AmlWorkItemLifecycleObserverTest` (unit test)

Update existing tests for the 4-arg `writeSarOfficerReviewed()` signature. Add test verifying rejection reason is captured from event detail.

**Test helper update:** The existing `event(WorkItemStatus, String, String)` helper always passes `null` as the `detail` parameter to `WorkItemLifecycleEvent.of()`. Add an overloaded helper:

```java
private WorkItemLifecycleEvent event(WorkItemStatus status, String callerRef,
        String actor, String detail) {
    WorkItem wi = new WorkItem();
    wi.id = UUID.randomUUID();
    wi.status = status;
    wi.callerRef = callerRef;
    return WorkItemLifecycleEvent.of(status.name(), wi, actor, detail);
}
```

The existing 3-arg helper delegates to the new 4-arg helper with `null` detail. The rejection reason test uses the 4-arg helper with a non-null detail string.

### §5.5 Test ordering (GE-20260628-dbc656)

Layer 9 tests with oversight gates must call `awaitAndApproveGate()` BEFORE waiting for attestations. `WorkerDecisionEvent` fires at worker completion, not dispatch — gated workers are not "complete" until the gate is approved.

---

## Files Changed

| File | Change | Issue |
|------|--------|-------|
| `api/.../InvestigationStatus.java` | New enum (annotation-free, wire format methods) | #74 |
| `api/.../InvestigationResolution.java` | New record | #74 |
| `api/.../InvestigationOutcome.java` | Add reason, 2-arg factory, drop null handling | #74, #75 |
| `app/.../AmlInvestigationOutcomeService.java` | Move compliance/ → engine/, add resolveInvestigation(), new deps | #74 |
| `app/.../AmlJacksonConfig.java` | Add `InvestigationStatusMixin` | #74 |
| `app/.../AmlSarOfficerReviewedLedgerEntry.java` | Add rejectionReason field, backward-compatible domainContentBytes() | #75 |
| `app/.../V2012__*.sql` | Add rejection_reason column | #75 |
| `app/.../AmlWorkItemLifecycleObserver.java` | Capture event.detail() | #75 |
| `app/.../AmlLedgerService.java` | 4-arg write methods, update noOp()/stub() overrides | #75 |
| `app/.../Layer6InvestigationResponse.java` | String → InvestigationStatus | #74 |
| `app/.../Layer9InvestigationResponse.java` | New record | #74 |
| `app/.../AmlLayer6Resource.java` | Return type → Response, use resolveInvestigation(), remove unused cache/repo injections, 404 | #74 |
| `app/.../AmlLayer9Resource.java` | Use resolveInvestigation(), typed response, remove unused cache/repo injections, 404 | #74 |
| `app/test/.../AmlInvestigationOutcomeServiceTest.java` | Update serviceWith() constructor, add 4 resolveInvestigation() tests, add 3 resolveOutcome() tests (stays in engine/) | #77 |
| `api/test/.../InvestigationOutcomeTest.java` | Update for 2-arg factory, replace null test | #77 |
| `api/test/.../InvestigationStatusTest.java` | New — wire format roundtrip | #77 |
| `app/test/.../InvestigationStatusMixinTest.java` | New — Jackson mixin wiring verification | #77 |
| `app/test/.../AmlLayer6ResourceTest.java` | Add 404 regression test for nonexistent caseId | #77 |
| `app/test/.../AmlLayer9ResourceTest.java` | Add gate/review helpers, add 2 outcome integration tests, add 404 regression test | #77 |
| `app/test/.../AmlWorkItemLifecycleObserverTest.java` | Update for 4-arg signature, add detail helper overload | #77 |

## Out of Scope

- Refactoring `AmlLedgerService` internal duplication between `writeSarOfficerReviewed` and `writeSarOfficerReviewedFailure`
- Sealed interface for `InvestigationOutcome` — evaluated and rejected; factory prevents invalid construction, no callers pattern-match in Java
