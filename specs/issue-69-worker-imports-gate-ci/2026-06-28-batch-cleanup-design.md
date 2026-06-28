# Batch Cleanup: #69 Worker Accessor Renames, #71 Gate Rejection Status, #68 CI Rename

**Date:** 2026-06-28
**Branch:** issue-69-worker-imports-gate-ci
**Covers:** #69, #71, #68

---

## #69 — Worker accessor renames

Imports already migrated to `io.casehub.worker.api.*`. Remaining work: accessor renames in two test files where `CaseDefinition` record accessors still use JavaBean style.

**Files:**
- `AmlInvestigationCaseHubTest.java` — `getName()` → `name()`, `getCapabilities()` → `capabilities()`
- `AmlOversightCaseHubTest.java` — `getName()` → `name()`

Scale: XS. Mechanical.

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
    "type": "gate-rejected",
    "action": "SAR_FILING",
    "reason": "Insufficient evidence of layering pattern"
  }
}
```

Outcome types: `gate-rejected`, `sar-filed`. Null while `status` is `in-progress`.

### Data flow

1. `AmlWorkItemLifecycleObserver` already writes `AML_SAR_OFFICER_REVIEWED` with `reviewDecision` — no change
2. `AmlLayer9Resource.getInvestigation()` adds: query ledger for officer review entry via `LedgerEntryRepository.findLatestBySubjectId()`
3. Derive `InvestigationOutcome` from the ledger entry's `reviewDecision` field

### New types

- `InvestigationOutcome` — record in `api/` module (JPA-free): `type`, `action`, `reason`

### Not in scope

- Downstream routing on rejection (re-open, close) — deferred to #72 / Layer 10
- New ledger entry types
- YAML binding changes

### Test coverage

- Happy path: SAR filed and approved → `outcome.type = "sar-filed"`
- Rejection: MLRO rejects → `outcome.type = "gate-rejected"` with reason
- In-progress: no officer review → `outcome` is null
