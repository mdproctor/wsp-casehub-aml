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
- `decision-not-recorded` — officer completed review but the observer failed to record the
  decision (from `reviewDecision = "UNKNOWN"`, written by the PP-20260530-49856c failure path).
  This is a system recording failure, not a domain outcome — the officer made a definite
  decision but the ledger write failed.

`outcome` is null while `status` is `in-progress`.

**Entry selection rule:** The PP-20260530-49856c failure path can produce multiple
`AmlSarOfficerReviewedLedgerEntry` records for the same case (e.g., a successful write followed
by a spurious failure entry when the method throws after DB commit). Selection: prefer entries
with `actorType = ActorType.HUMAN` (the officer's actual decision, written by
`writeSarOfficerReviewed`). Fall back to `ActorType.SYSTEM` (the failure marker, written by
`writeSarOfficerReviewedFailure`) only when no HUMAN-attributed entry exists.

**Eventual consistency:** `outcome` may also be null for a recently-completed case. The
`AmlWorkItemLifecycleObserver` fires via `@ObservesAsync` — the engine sets `CaseStatus.COMPLETED`
synchronously before the async ledger write commits. During this window (typically sub-second),
the API returns `status: "completed"` with `outcome: null`.

**Bounded retry guidance:** Consumers should retry `completed + null outcome` for at most
5 seconds. After that window, treat `null` outcome as a permanent recording failure — the async
event was never delivered (e.g., JVM crash between engine commit and CDI event delivery). This is
a pre-existing gap in the `@ObservesAsync` pattern, not introduced by this spec. The 5-second
bound provides comfortable margin over the sub-second typical delivery window.

### Data flow

1. `AmlWorkItemLifecycleObserver` already writes `AML_SAR_OFFICER_REVIEWED` with `reviewDecision` — no change
2. `AmlInvestigationOutcomeService` (new `@ApplicationScoped` service) encapsulates outcome derivation:
   a. Query ledger via `LedgerEntryRepository.findBySubjectId(caseId)`,
      filter to `AmlSarOfficerReviewedLedgerEntry.class::isInstance`
   b. Apply entry selection rule: prefer `actorType = HUMAN`; fall back to `SYSTEM`
   c. Derive `InvestigationOutcome` from the selected entry's `reviewDecision` field:
      `"APPROVED"` → `sar-filed`, `"REJECTED"` → `gate-rejected`, `"UNKNOWN"` → `decision-not-recorded`
   d. If no matching entry exists, return `null` (in-progress or eventual-consistency window)
3. `AmlLayer9Resource.getInvestigation()` delegates to `AmlInvestigationOutcomeService` — thin
   dispatcher only, consistent with `AmlLayer7Resource` → `AmlComplianceEvidenceService` pattern

### New types

- `InvestigationOutcome` — record in `api/` module (JPA-free): `type` (String)
- `AmlInvestigationOutcomeService` — `@ApplicationScoped` in `app/` module: outcome derivation
  logic (ledger query, entry selection, reviewDecision mapping)

### Not in scope

- Downstream routing on rejection (re-open, close) — deferred to #72 / Layer 10
- New ledger entry types
- YAML binding changes

### Test coverage

- Happy path: SAR filed and approved → `outcome.type = "sar-filed"`
- Rejection: MLRO rejects → `outcome.type = "gate-rejected"`
- Observer failure: ledger write fails, fallback writes UNKNOWN → `outcome.type = "decision-not-recorded"`
- In-progress: no officer review entry in ledger → `outcome` is null
- Eventual consistency: case completed but async observer not yet fired → `outcome` is null
- Multiple entries (race): both HUMAN and SYSTEM entries exist → HUMAN wins → `outcome.type` from officer's decision
- Multiple entries (failure only): only SYSTEM entry exists → `outcome.type = "decision-not-recorded"`
- Service isolation: `AmlInvestigationOutcomeService` tested independently of JAX-RS
