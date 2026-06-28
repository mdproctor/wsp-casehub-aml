# Batch Cleanup: #69 Worker Accessor Renames, #71 Gate Rejection Status, #68 CI Rename

**Date:** 2026-06-28
**Branch:** issue-69-worker-imports-gate-ci
**Covers:** #69, #71, #68

---

## #69 — Worker import migration

Imports already migrated to `io.casehub.worker.api.*` (ledger#88, parent#251). No remaining work.

`CaseDefinition` is a regular class with JavaBean-style getters (`getName()`, `getCapabilities()`),
not a Java record. The test files already use the correct accessor style for each type:
`CaseDefinition` → JavaBean getters, `Worker`/`Capability` → record accessors. No renames needed.

Scale: XS. Done — close #69.

---

## #68 — CI workflow rename

Rename `.github/workflows/build.yml` → `publish.yml`. Content unchanged — `repository_dispatch: types: [upstream-published]` is already configured. Aligns with platform#110 naming convention.

Scale: XS. File rename only.

---

## #71 — Gate rejection status surfacing

### Problem

When the MLRO rejects a SAR filing, `AmlWorkItemLifecycleObserver` writes a `REJECTED` ledger entry, but the investigation status API (`GET /api/layer9/investigations/{caseId}`) has no way to surface what happened. It returns only `status: "in-progress" | "completed"`.

### Design principle

Separate lifecycle status from domain outcome. `status` answers "is it done?" `outcome` answers "what happened?" These are orthogonal concerns — a gate rejection is a domain event that triggers a lifecycle transition, not a lifecycle state itself.

### API response shape

```json
{
  "caseId": "uuid",
  "status": "completed",
  "outcome": {
    "type": "gate-rejected"
  }
}
```

Outcome types:
- `gate-rejected` — MLRO rejected the SAR filing (from `reviewDecision = "REJECTED"`)
- `sar-filed` — MLRO approved the SAR filing (from `reviewDecision = "APPROVED"`)
- `review-inconclusive` — officer completed review but the observer failed to record the
  decision (from `reviewDecision = "UNKNOWN"`, written by the PP-20260530-49856c failure path)

`outcome` is null while `status` is `in-progress`.

**Eventual consistency:** `outcome` may also be null for a recently-completed case. The
`AmlWorkItemLifecycleObserver` fires via `@ObservesAsync` — the engine sets `CaseStatus.COMPLETED`
synchronously before the async ledger write commits. During this window (typically sub-second),
the API returns `status: "completed"` with `outcome: null`. Consumers should treat this as
transient and retry.

### Data flow

1. `AmlWorkItemLifecycleObserver` already writes `AML_SAR_OFFICER_REVIEWED` with `reviewDecision` — no change
2. `AmlLayer9Resource.getInvestigation()` adds: query ledger via `LedgerEntryRepository.findBySubjectId(caseId)`
   filtered to `AmlSarOfficerReviewedLedgerEntry.class::isInstance` (consistent with existing pattern in
   `AmlLedgerService.writeSarOfficerReviewed()` and `writeComplianceReviewOpened()`)
3. Derive `InvestigationOutcome` from the ledger entry's `reviewDecision` field:
   `"APPROVED"` → `sar-filed`, `"REJECTED"` → `gate-rejected`, `"UNKNOWN"` → `review-inconclusive`
4. If no matching entry exists, return `outcome: null` (in-progress or eventual-consistency window)

### New types

- `InvestigationOutcome` — record in `api/` module (JPA-free): `type` (String)

### Not in scope

- Downstream routing on rejection (re-open, close) — deferred to #72 / Layer 10
- New ledger entry types
- YAML binding changes

### Test coverage

- Happy path: SAR filed and approved → `outcome.type = "sar-filed"`
- Rejection: MLRO rejects → `outcome.type = "gate-rejected"`
- Observer failure: ledger write fails, fallback writes UNKNOWN → `outcome.type = "review-inconclusive"`
- In-progress: no officer review entry in ledger → `outcome` is null
- Eventual consistency: case completed but async observer not yet fired → `outcome` is null
