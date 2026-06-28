# Separate sar-drafting from compliance-review-opening for SAR_FILING oversight gate

Refs #58. Splits the `openReview()` side effect out of the sar-drafting workers and gates
SAR filing behind MLRO approval via `PlannedAction(SAR_FILING)`.

## Problem

Both `sarDraftingWorkerJunior` and `sarDraftingWorkerSenior` in `AmlInvestigationCaseDescriptor`
call `complianceReviewLifecycle.openReview()` unconditionally inside the worker lambda. This
creates a compliance officer WorkItem BEFORE any MLRO gate can run.

The correct flow:
1. `sar-drafting` — pure analysis: synthesise SAR narrative, declare `PlannedAction(SAR_FILING)`
2. Engine fires `AmlActionRiskClassifier` → `GateRequired(ALWAYS)` for `SAR_FILING`
3. MLRO approves the gate
4. `compliance-review-opening` worker calls `openReview()`

## Constraint

`PlannedAction` is not supported in `FlowWorkerFunction` (engine#564). `FlowWorkerFunctionHandler`
hardcodes `WorkerResult.of(model.asMap()...)` — the single-argument overload with no `PlannedAction`
parameter. SAR-drafting workers must convert from Flow to Sync (raw lambda returning `WorkerResult`)
— same pattern as `entityLinkProposalWorker` in `AmlOversightCaseHub`.

## Worker changes

### SAR-drafting workers (junior + senior)

Convert from `FlowWorkerFunction` to `WorkerFunction.Sync` (raw lambda returning
`WorkerResult.of(map, plannedAction)`). `SyncAgentWorkerFunctionHandler` routes these
via `Worker.builder().function(Function<Map, WorkerResult>)`.

- **Remove**: `complianceReviewLifecycle.openReview()` call and `WorkerExecutionContext.current().caseId()` (no longer needed — workers don't call `openReview()`)
- **Add**: `PlannedAction.of("SAR filing for transaction <id>", AmlActionType.SAR_FILING.actionType(), params)`
- **Output**: `Map.of("sarNarrative", sarNarrative)` — no `complianceTaskId`
- **PlannedAction params**: `transactionId`, `amount`, `currency`, `entityType` (gives MLRO reviewer context)

The descriptor still needs `ComplianceReviewLifecycle` and `ObjectMapper` for the new
compliance-review-opening worker.

### New `compliance-review-opening-agent` worker

Worker name: `compliance-review-opening-agent` (follows `{capability}-agent[-qualifier]` convention).

Added to `AmlInvestigationCaseDescriptor`. Uses `FlowWorkerFunction` (no PlannedAction needed).

- **Input**: transaction, entityResolution, patternAnalysis, osintScreening, sarNarrative (all from context)
- **Builds** `InvestigationSummary` via existing `buildSummary()` method
- **caseId**: obtained via `WorkerExecutionContext.current().caseId()` — `FlowWorkerFunctionHandler` calls `WorkerExecutionContext.set(context)` before executing the workflow on a virtual thread; `FuncWorkflowBuilder` lambdas execute synchronously within `wfInstance.start()`, so the ThreadLocal is available
- **Calls** `complianceReviewLifecycle.openReview(tx, summary, caseId)`
- **Output**: `Map.of("complianceTaskId", complianceTaskId)`

## YAML changes (aml-investigation.yaml)

### Capability: sar-drafting (updated)

- description: "Synthesise specialist findings into SAR narrative"
- outputSchema: `"{ sarNarrative: .sarNarrative }"` (remove `complianceTaskId`)

### Capability: compliance-review-opening (new)

- description: "Open compliance officer review WorkItem after MLRO gate approval"
- inputSchema: `"{ transaction: .transaction, entityResolution: .entityResolution, patternAnalysis: .patternAnalysis, osintScreening: .osintScreening, sarNarrative: .sarNarrative }"`
- outputSchema: `"{ complianceTaskId: .complianceTaskId }"`

### Binding: sar-drafting (comment update)

The existing binding comment (lines 94–95) says "The SAR-drafting worker creates the compliance
WorkItem and writes complianceTaskId." Update to: "SAR-drafting worker synthesises the narrative
and declares PlannedAction(SAR_FILING); complianceTaskId is set by compliance-review-opening
after MLRO gate approval."

### Binding: compliance-review-opening (new)

```yaml
## Fires after MLRO gate approves SAR filing. The compliance-review-opening worker
## creates the compliance officer WorkItem (30-day FinCEN SLA) and writes complianceTaskId.
- name: compliance-review-opening
  on: { contextChange: {} }
  when: ".sarNarrative != null and .complianceTaskId == null"
  capability: compliance-review-opening
```

### Unchanged

- Goal `investigation-complete` condition `.complianceTaskId != null` — still correct
- All existing bindings — unchanged

## Sequencing

### Happy path (MLRO approves)

1. Specialist findings complete → sar-drafting binding fires
2. sar-drafting worker produces `sarNarrative` + `PlannedAction(SAR_FILING)`
3. `AmlActionRiskClassifier` classifies → `GateRequired(ALWAYS, reversible=false, ["aml-mlro"])`
4. Engine creates MLRO gate WorkItem, pauses case
5. MLRO approves → engine commits `sarNarrative` to case context
6. compliance-review-opening binding evaluates true → fires
7. Worker calls `openReview()`, writes `complianceTaskId` to context
8. Goal `investigation-complete` satisfied → case completes

### MLRO rejects (out of scope for #58)

If the MLRO rejects the SAR_FILING gate, the engine fires `ActionGateRejectedEvent`.
`sarNarrative` is never committed to case context. `complianceTaskId` never appears. The
`investigation-complete` goal is never satisfied. The case stalls.

Handling rejection requires design decisions beyond this issue:
- What does rejection mean for the investigation lifecycle? (close-without-filing? re-investigation? reassignment?)
- A new `ActionGateRejectedEvent` consumer in AML
- Potentially new goal conditions for alternative outcomes

File as a follow-up issue. This spec covers separation and the happy path only.

## Trust score seeding

### AmlTrustScoreSeeder

Add `compliance-review-opening-agent` to `SEEDS`. The existing pattern seeds all workers —
including solo workers with no competing worker for their capability (`entity-resolution-agent`,
`pattern-analysis-agent`, `senior-analyst-agent`). Trust scores serve dual purposes: routing
selection AND historical reliability tracking via attestations. Consistency requires seeding
the new worker.

`compliance-review-opening-agent` is deterministic (calls `openReview()`, returns a task ID).
High confidence: `alpha=9, beta=1` (score ≈ 0.90).

```java
new WorkerSeed("compliance-review-opening-agent", "compliance-review-opening", 9, 1)
```

### AmlTrustScoreSeederTest

Add `compliance_review_opening_seeded_with_high_trust()` — consistent with the existing
per-worker test methods.

## Test changes

### AmlInvestigationCaseDescriptorTest (unit — plain JUnit, no Quarkus)

Five methods break:

1. **`workers_returnsSevenDistinctWorkers()`** — count 7 → 8. Add `compliance-review-opening-agent` to the expected name set.

2. **`capability_names_match_expected_tags()`** — add mapping: `compliance-review-opening-agent` → `compliance-review-opening`.

3. **`worker_execution_model_classification_is_exhaustive()`** — this is the hardest break. Currently asserts every worker is `FlowWorkerFunction`. After the change, the descriptor has mixed execution models:
   - **Flow** (6): entity-resolution-agent, pattern-analysis-agent, osint-screening-agent, osint-screening-agent-senior, senior-analyst-agent, compliance-review-opening-agent
   - **Sync** (2): sar-drafting-agent-junior, sar-drafting-agent-senior

   Redesign: two explicit sets (`FLOW_WORKERS`, `SYNC_WORKERS`), assert each worker appears in exactly one set, assert the correct `instanceof` per set. This is a better test than the current "assert all are Flow" — it forces every new worker to be explicitly classified.

4. **`each_worker_declares_exactly_one_capability()`** — passes automatically (new worker has one capability).

5. **`each_worker_has_non_null_function()`** — passes automatically.

### AmlInvestigationCaseHubTest (@QuarkusTest — CDI integration)

Four assertions change:

- **`hasFiveCapabilities()`** → 6 capabilities. Add `compliance-review-opening` to expected list.
- **`hasSixBindings()`** → 7 bindings. Add `compliance-review-opening` to expected list.
- **`hasSevenWorkers()`** → 8 workers. Add `compliance-review-opening-agent` to expected name set.
- **`hasInvestigationCompleteGoal()`** — unchanged (goal condition still checks `complianceTaskId`).

### AmlLayer6InvestigationTest (@QuarkusTest — end-to-end integration)

New mid-flow step: await SAR_FILING gate WorkItem → assert gate properties
(`candidateGroups = "aml-mlro"`) → approve via `workItemService.completeFromSystem()` →
then poll for completion.

Needs: `@Inject WorkItemService`, `@PersistenceContext EntityManager` (no unitName — default
datasource for gate WorkItems), `findGateWorkItems(caseId)` helper using
`QuarkusTransaction.requiringNew()`.

Already has `@PersistenceContext(unitName = "qhorus")` for attestation queries — needs a second
`EntityManager` with no unitName for gate WorkItem queries.

### AmlLayer6ResourceTest (@QuarkusTest — REST endpoint integration)

All 4 tests that start investigations and drain to completion need the gate-approval step
inserted: await gate → approve → await completion. The 5th test
(`post_outcome_with_invalid_score_returns_400`) does not start an investigation — unchanged.

Needs same infrastructure as `AmlLayer6InvestigationTest`: `@Inject WorkItemService`,
`@PersistenceContext EntityManager`, `findGateWorkItems()` helper.

### AmlTrustScoreSeederTest (@QuarkusTest)

Add `compliance_review_opening_seeded_with_high_trust()` — asserts seeded score ≈ 0.90.

### Gate WorkItem query pattern (both Layer 6 test classes)

```java
private List<WorkItem> findGateWorkItems(UUID caseId) {
    return QuarkusTransaction.requiringNew().call(() ->
        em.createQuery("SELECT w FROM WorkItem w WHERE w.callerRef LIKE :pattern", WorkItem.class)
            .setParameter("pattern", "case:" + caseId + "/gate:%")
            .getResultList());
}
```

Same pattern as `AmlLayer9ActionGateTest`.

## Doc changes (ARC42STORIES.MD)

Five lines need updating:

| Line | Current | After |
|------|---------|-------|
| 203 | "5 capabilities · 5 contextChange bindings · 1 goal" | "6 capabilities · 7 contextChange bindings · 1 goal" |
| 204 | "delegates 7 worker lambdas to AmlInvestigationCaseDescriptor" | "delegates 8 worker lambdas to AmlInvestigationCaseDescriptor" |
| 1462 | "✅ Resolved — all 7 workers use FlowWorkerFunction" | 6 of 8 workers use FlowWorkerFunction; sar-drafting workers (junior + senior) use WorkerFunction.Sync for PlannedAction support (engine#564 blocks Flow support) |
| 1464 | SLA GAP — "sar-drafting worker does not write COMPLIANCE_REVIEW_OPENED ledger entry" (Deferred) | ✅ Resolved — `ComplianceReviewLifecycle.openReview()` consolidated both operations (aml#56); `compliance-review-opening` worker calls `openReview()` post-MLRO gate (aml#58) |
| 1504 | "sar-drafting workers still call openReview() unconditionally" | Remove (resolved by aml#58) |
| 1522 | "all 7 AML investigation worker lambdas" | "all 8 AML investigation worker lambdas" |

## Files changed

| File | Change |
|------|--------|
| `AmlInvestigationCaseDescriptor.java` | Convert sar-drafting workers to Sync + PlannedAction; add `compliance-review-opening-agent` worker (Flow); worker count 7→8 |
| `aml-investigation.yaml` | Update sar-drafting outputSchema and binding comment; add compliance-review-opening capability + binding; capability count 5→6, binding count 6→7 |
| `AmlTrustScoreSeeder.java` | Add `compliance-review-opening-agent` seed (alpha=9, beta=1) |
| `AmlInvestigationCaseDescriptorTest.java` | Update worker count (7→8), name set, capability map; redesign execution model classification for mixed Flow/Sync |
| `AmlInvestigationCaseHubTest.java` | Update capability count (5→6), binding count (6→7), worker count (7→8) |
| `AmlLayer6InvestigationTest.java` | Add gate-approval step; add WorkItemService + default EntityManager |
| `AmlLayer6ResourceTest.java` | Add gate-approval step to all 4 draining tests; add WorkItemService + default EntityManager |
| `AmlTrustScoreSeederTest.java` | Add `compliance_review_opening_seeded_with_high_trust()` |
| `ARC42STORIES.MD` | Update lines 203, 204, 1462, 1464, 1504, 1522 (see Doc changes table) |
