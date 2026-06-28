# Batch Cleanup: #69 Worker Accessor Renames, #71 Gate Rejection Status, #68 CI Rename

**Date:** 2026-06-28
**Branch:** issue-69-worker-imports-gate-ci
**Covers:** #69, #71, #68
**Status:** Post-implementation review — all changes are committed. This spec documents completed work for design completeness review.

---

## S1: #69 — Worker import migration

Imports already migrated to `io.casehub.worker.api.*` (ledger#88, parent#251). No remaining work.

`CaseDefinition` is a regular class with JavaBean-style getters (`getName()`, `getCapabilities()`),
not a Java record. The test files already use the correct accessor style for each type:
`CaseDefinition` → JavaBean getters, `Worker`/`Capability` → record accessors. No renames needed.

Scale: XS. Already closed.

---

## S2: #68 — CI workflow rename

Rename `.github/workflows/build.yml` → `publish.yml`. Content unchanged — `repository_dispatch: types: [upstream-published]` is already configured. Aligns with platform#110 naming convention.

Scale: XS. Already committed (7fec6a2); close #68.

---

## S3: #71 — Gate rejection status surfacing

**Status:** Implemented across commits 8abe996, 909cfdb, b0fffd7. This section documents the
completed design and identifies corrections discovered during review.

### S3.1: Problem

When the MLRO rejects a SAR filing, `AmlWorkItemLifecycleObserver` writes a `REJECTED` ledger entry, but the investigation status APIs have no way to surface what happened. Both `GET /api/layer9/investigations/{caseId}` and `GET /api/layer6/investigations/{caseId}` return only `status: "in-progress" | "completed"`.

### S3.2: Design principle

Separate lifecycle status from domain outcome. `status` answers "is it done?" `outcome` answers "what happened?" These are orthogonal concerns — a gate rejection is a domain event that triggers a lifecycle transition, not a lifecycle state itself.

### S3.3: API surfaces

Both Layer 6 and Layer 9 return outcome resolution for completed cases.

**Layer 9** — `GET /api/layer9/investigations/{caseId}`:
```json
{
  "caseId": "uuid",
  "status": "completed",
  "outcome": {
    "type": "gate-rejected"
  }
}
```

**Layer 6** — `GET /api/layer6/investigations/{caseId}`:
```json
{
  "caseId": "uuid",
  "status": "completed",
  "routingDecisions": [...],
  "outcome": {
    "type": "gate-rejected"
  }
}
```

Layer 6 adds `routingDecisions` (per-capability worker selections with trust scores from
`TrustScoreSource`). Outcome derivation is shared — both endpoints delegate to
`AmlInvestigationOutcomeService`. Two endpoints return the same outcome because Layer 6 callers
(trust routing consumers) and Layer 9 callers (oversight gate consumers) have different primary
concerns but both need the investigation result.

Outcome types:
- `gate-rejected` — MLRO rejected the SAR filing (from `reviewDecision = "REJECTED"`)
- `sar-filed` — MLRO approved the SAR filing (from `reviewDecision = "APPROVED"`)
- `decision-not-recorded` — legacy: the observer failure path previously wrote
  `reviewDecision = "UNKNOWN"`. Retained for backwards compatibility with existing ledger data.
  New failure entries preserve the actual decision (see failure path below).

`outcome` is null while `status` is `in-progress`.

**Entry selection rule:** The PP-20260530-49856c failure path can produce multiple
`AmlSarOfficerReviewedLedgerEntry` records for the same case (e.g., a successful write followed
by a spurious failure entry when the method throws after DB commit). Selection: prefer entries
with `actorType = ActorType.HUMAN` (the officer's actual decision, written by
`writeSarOfficerReviewed`). Fall back to `ActorType.SYSTEM` (the failure marker, written by
`writeSarOfficerReviewedFailure`) only when no HUMAN-attributed entry exists.
Secondary sort: by `sequenceNumber` descending (latest entry wins) ensures deterministic
selection when multiple entries share the same `actorType` (commit 41c7c13). Mutually exclusive write paths per PP-20260530-49856c ensure
monotonic `sequenceNumber` — the primary write either fully commits (preventing the failure
path via `written = true`) or fully rolls back, so the failure entry always receives the
correct next sequence number.

**Unexpected `reviewDecision` values:** `InvestigationOutcome.fromReviewDecision`
throws `IllegalStateException` for values other than `"APPROVED"`, `"REJECTED"`, `"UNKNOWN"`,
or `null` (commit 41c7c13). An unrecognised value indicates data corruption or an
unanticipated upstream enum expansion — fail fast rather than silently returning null, which
would be indistinguishable from the eventual-consistency window.

**Failure path preserves actual decision:** `writeSarOfficerReviewedFailure`
accepts the actual `reviewDecision` derived by the observer (`"APPROVED"` or `"REJECTED"`) and
writes it to the failure entry alongside `actorType = SYSTEM` (commit 41c7c13). This ensures
the audit trail captures the officer's decision even when the primary write fails. `"UNKNOWN"`
is no longer written by new failure entries. The `decision-not-recorded` outcome type is
retained for backwards compatibility with existing `"UNKNOWN"` entries from before this change.

**Consumer-visible states after completion:**

| `status` | `outcome` | Meaning | Consumer action |
|----------|-----------|---------|-----------------|
| `in-progress` | `null` | Investigation ongoing | Poll |
| `completed` | `{ type: "..." }` | Decision recorded | Use outcome |
| `completed` | `null` | Either: (a) async delivery delay (sub-second, will resolve) or (b) total observer failure (permanent — both `writeSarOfficerReviewed` and `writeSarOfficerReviewedFailure` threw) | Retry for 5s. After that, treat as total observer failure (AUDIT GAP). |

State (b) — total observer failure — is a pre-existing gap in the `@ObservesAsync` pattern.
The PP-20260530-49856c double-try/catch minimises its likelihood: the failure path must also
fail for the outcome to remain permanently null. A fully reliable solution would require
guaranteed delivery (e.g., outbox pattern), which is out of scope for this spec.

### S3.4: Data flow

`[DEFERRED: #74]` Steps 2–3 describe the consolidated design. Current code has completion
detection inlined in both resources and `resolve()` returns `InvestigationOutcome` only. See
§S3.9, correction 2.

1. `AmlWorkItemLifecycleObserver` writes `AML_SAR_OFFICER_REVIEWED` with `reviewDecision` — no
   change to the observer trigger. Failure path passes actual `reviewDecision` instead of
   hardcoding `"UNKNOWN"` (commit 41c7c13).
2. `AmlInvestigationOutcomeService` (`@ApplicationScoped`, in `compliance/` package — commit
   41c7c13) encapsulates both completion detection and outcome
   derivation. Method: `Optional<InvestigationResolution> resolveInvestigation(UUID caseId)` where
   `record InvestigationResolution(String status, InvestigationOutcome outcome)` is in `api/`.
   a. Check `CaseInstanceCache.get(caseId)`, fall back to
      `CaseInstanceRepository.findByUuid(caseId, ...)`
   b. If both return null, return `Optional.empty()` — case not found. Resources translate to
      HTTP 404. (Pre-existing bug: current code returns `{status: "in-progress"}` for
      nonexistent UUIDs, indistinguishable from a real in-progress investigation.)
   c. If case exists but not `CaseStatus.COMPLETED`, return
      `Optional.of(new InvestigationResolution("in-progress", null))`
   d. If completed, query ledger via `LedgerEntryRepository.findBySubjectId(caseId)`,
      filter to `AmlSarOfficerReviewedLedgerEntry.class::isInstance`
   e. Apply entry selection rule: prefer `actorType = HUMAN`, then `SYSTEM`; tiebreak by
      `sequenceNumber` descending
   f. Derive `InvestigationOutcome` from the selected entry's `reviewDecision` field:
      `"APPROVED"` → `sar-filed`, `"REJECTED"` → `gate-rejected`,
      `"UNKNOWN"` → `decision-not-recorded`. Unexpected values → `IllegalStateException`
   g. If no matching entry exists, return
      `Optional.of(new InvestigationResolution("completed", null))` — eventual-consistency window
      or total observer failure
3. `AmlLayer9Resource.getInvestigation()` delegates fully to
   `AmlInvestigationOutcomeService.resolveInvestigation()` — a thin dispatcher that unwraps
   the Optional (404 if empty) and wraps the result in a HashMap response.
   `AmlLayer6Resource.getInvestigation()` delegates outcome resolution to the service but
   retains routing-decision assembly: querying
   `AmlWorkerDecisionRepository.findAllByCaseId()`, mapping each `WorkerDecisionEntry` with
   live trust scores from `TrustScoreSource`, and building `Layer6InvestigationResponse`.
   Layer 6 is not a thin dispatcher — it delegates one sub-concern (status + outcome) while
   retaining ~15 lines of routing-decision logic.

### S3.5: Implemented types

Both types already exist in the codebase (commits 8abe996, 909cfdb). Committed corrections
reference their commit hash. Deferred items are marked `[DEFERRED: #74]`. See §S3.9 for
the correction status table.

- `InvestigationOutcome` — record in `api/` module (JPA-free): `type` (String).
  `fromReviewDecision` factory method throws `IllegalStateException` on unexpected values
  (commit 41c7c13).
- `InvestigationResolution` — `[DEFERRED: #74]` new record in `api/` module:
  `record InvestigationResolution(String status, InvestigationOutcome outcome)`. Return type for
  `AmlInvestigationOutcomeService.resolveInvestigation()` (see §S3.9, correction 2).
- `AmlInvestigationOutcomeService` — `@ApplicationScoped`, in `compliance/` package (moved from
  `engine/` in commit 41c7c13): completion detection + outcome derivation. The `compliance/` package currently has no `engine-common` imports —
  `AmlComplianceEvidenceService` imports `WorkItem` from `casehub-work` (a different module),
  not engine types. Adding `CaseInstanceCache`, `CaseInstanceRepository`, and `CaseStatus`
  from `engine-common` is a new cross-module dependency direction in this package. Accepted
  because: the service is semantically a compliance query ("what did the officer decide?");
  the engine dependency is incidental (completion-state scoping); and the `aml` app module
  already depends on `engine-common` at the Maven level — this is a package-level concern,
  not a new module dependency.

### S3.6: Not in scope

- Downstream routing on rejection (re-open, close) — deferred to #72 / Layer 10
- New ledger entry types
- YAML binding changes
- **Rejection reason capture** — issue #71 explicitly requests "rejection reason" in both the
  API response and the ledger entry. This spec does not deliver it.
  `AmlSarOfficerReviewedLedgerEntry` has no `rejectionReason` field; `InvestigationOutcome` has
  no `reason` field; `AmlWorkItemLifecycleObserver` ignores `event.detail()` and
  `event.rationale()`. Deferred: requires upstream verification that `casehub-work` populates
  `detail()`/`rationale()` on REJECTED transitions, plus schema changes to the ledger entry and
  outcome types. Tracked as casehubio/aml#73.

### S3.7: Test coverage

**Existing tests (passing against current code):**

- Happy path: `AmlInvestigationOutcomeServiceTest.returns_sar_filed_for_approved_human_entry`
- Rejection: `AmlInvestigationOutcomeServiceTest.returns_gate_rejected_for_rejected_human_entry`
- Observer failure (legacy): `AmlInvestigationOutcomeServiceTest.returns_decision_not_recorded_for_unknown_system_entry`
- In-progress: `AmlInvestigationOutcomeServiceTest.returns_null_when_no_officer_review_entry`
- Multiple entries (race): `AmlInvestigationOutcomeServiceTest.prefers_human_entry_over_system_entry_in_race`, `prefers_human_entry_regardless_of_list_order`
- Service isolation: `AmlInvestigationOutcomeService` tested independently of JAX-RS
- Observer writes failure entry: `AmlWorkItemLifecycleObserverTest.ledgerWriteFails_writesFailureEntry` — verifies 3-param `writeSarOfficerReviewedFailure(caseId, officerId, reviewDecision)` with `eq("APPROVED")` (updated in commit 41c7c13)
- Unrecognised reviewDecision: `InvestigationOutcomeTest.unrecognised_value_throws` — asserts `IllegalStateException` (updated in commit 41c7c13)
- Layer 6 outcome integration: `AmlLayer6ResourceTest.officer_approval_surfaces_sar_filed_outcome`, `officer_rejection_surfaces_gate_rejected_outcome`

**Test gaps for committed code (tracked as casehubio/aml#77):**

- sequenceNumber tiebreaker: two HUMAN entries with different `sequenceNumber` → highest wins.
  The `officerEntry()` helper does not set `sequenceNumber` — all entries default to 0, so the
  tiebreaker code path (correction 3, commit 41c7c13) is never exercised.
- Failure-only scenario with preserved actual decision: only SYSTEM entry with
  `reviewDecision = "REJECTED"` → outcome type `gate-rejected`. Tests only cover legacy
  `"UNKNOWN"` → `decision-not-recorded`.
- Layer 9 outcome integration test: trigger officer review, assert `outcome.type` after
  completion. Layer 6 has coverage; Layer 9 does not — the HashMap serialisation path is
  untested.
- Test package mismatch: `AmlInvestigationOutcomeServiceTest` is in `io.casehub.aml.engine` but
  the class under test is in `io.casehub.aml.compliance` (moved in commit 41c7c13). Move test
  to `io.casehub.aml.compliance`.

**Tests to add for correction 2 (casehubio/aml#74):**

- Nonexistent caseId → 404 response
- Eventual consistency — case completed, async observer not yet fired → `outcome` is null

### S3.8: ARC42STORIES update

**✅ Committed (0c90b2a):**

- Layer 9 (ActionRiskClassifier Oversight Gate) entry: `outcome` field on
  `GET /api/layer9/investigations/{caseId}` response, `AmlInvestigationOutcomeService`
  delegation from both Layer 6 and Layer 9 resources, `AmlSarOfficerReviewedLedgerEntry` as the
  data source for outcome derivation
- Layer 6 (Trust Routing) entry: `outcome` field in `Layer6InvestigationResponse` description,
  correct 4-field record shape including `InvestigationOutcome outcome`

**Pending (commit directly — one-line merge):**

- Fix `aml-app/` directory listing: merge duplicate `compliance/` entries (lines 253 and 262)
  into a single entry listing all compliance package members: `AmlInvestigationOutcomeService`,
  `AmlComplianceEvidenceService`, `AmlLayer7Resource`, `AmlWorkItemLifecycleObserver`
- `AmlWorkItemLifecycleObserver` is in `compliance/` but missing from the directory listing

### S3.9: Corrections

Design corrections identified during review. Corrections 1, 3, 4, 5 are committed (41c7c13).
Only correction 2 remains — tracked as casehubio/aml#74.

| # | Correction | Status |
|---|-----------|--------|
| 1 | `fromReviewDecision` throws on unexpected values | ✅ Committed (41c7c13) |
| 2 | Consolidate completion detection; add `InvestigationResolution`; handle not-found | Pending — casehubio/aml#74 |
| 3 | Add `sequenceNumber` tiebreaker | ✅ Committed (41c7c13) |
| 4 | Failure path preserves actual `reviewDecision` | ✅ Committed (41c7c13) |
| 5 | Move service to `compliance/` package | ✅ Committed (41c7c13) |

**Correction 2 scope (casehubio/aml#74):**

- Move cache→repo→`CaseStatus.COMPLETED` logic from both `AmlLayer9Resource` and
  `AmlLayer6Resource` into `AmlInvestigationOutcomeService`
- Rename `resolve()` to `resolveInvestigation()`, return `Optional<InvestigationResolution>`
- Add `record InvestigationResolution(String status, InvestigationOutcome outcome)` to `api/` module
- When both cache and repo return null → `Optional.empty()` → resources return HTTP 404
  (fixes API-level bug: nonexistent caseIds currently return misleading
  `{status: "in-progress"}`)
