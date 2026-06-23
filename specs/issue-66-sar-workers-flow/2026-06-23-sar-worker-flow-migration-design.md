# Design: Migrate SAR Drafting Workers to FuncWorkflowBuilder

**Issue:** casehubio/aml#66
**Branch:** issue-66-sar-workers-flow
**Date:** 2026-06-23
**Status:** Approved (revised after review)

## Context

`AmlInvestigationCaseDescriptor` contains 7 workers. Five were migrated to `FuncWorkflowBuilder`
(WorkerFunction.Flow) in casehubio/aml#46. The two SAR drafting workers were left as
`WorkerFunction.Sync` (raw lambda) because `DefaultWorkerExecutor.executeFlow()` did not call
`WorkerExecutionContext.set(context)`, causing `WorkerExecutionContext.current().caseId()` to
return null inside FuncDSL lambdas. (Confirmed via bytecode inspection — see blog
`2026-06-22-mdp01-cache-gaps-and-workflow-context.md`.)

`casehubio/engine#559` is now closed. `executeFlow` now sets `WorkerExecutionContext` before
delegating to `FlowWorkerExecutor`, matching the behaviour of `executeSync`. The blocker is gone.

## Pre-requisite: Confirm engine SNAPSHOT includes engine#559

**This step gates integration test execution.** Always force-refresh the engine SNAPSHOT before
running `AmlLayer6InvestigationTest`:

```bash
mvn -U dependency:resolve -pl app -am -q
```

If the fix is not yet published (CI may still be running), the unit test will pass (it checks
only `WorkerFunction.Flow.class` instance type), but `AmlLayer6InvestigationTest` will fail with
an opaque Awaitility timeout — `WorkerExecutionContext.current()` still returns null, causing an
NPE inside the SAR lambda, causing the engine case to fail, causing the poll to never see
`status=completed`. In that case, wait for CI to publish the updated SNAPSHOT before proceeding
to the integration test.

Note: the spec was written at a point in time when the local jars predated engine#559; the
timestamp context in earlier drafts is no longer relevant. Force-refresh is always the right
action regardless of jar timestamps.

## Scope

Files changed in `app/`:

| File | Change |
|------|--------|
| `AmlInvestigationCaseDescriptor.java` | Migrate 2 workers; update Javadoc |
| `AmlInvestigationCaseDescriptorTest.java` | Change SAR worker assertion from `Sync` to `Flow`; update comments |

Project root (workspace):

| File | Change |
|------|--------|
| `CLAUDE.md` | Update misleading `Worker.Builder.function()` return type note |

Project root (`/Users/mdproctor/claude/casehub/aml`):

| File | Change |
|------|--------|
| `ARC42STORIES.MD` | Update 3 stale references to SAR workers being `WorkerFunction.Sync` (lines 1050, 1060, 1462) |

Outside `app/`:

| Artifact | Change |
|----------|--------|
| Garden entry `GE-20260609-ddd4b8` | Revise stale ⚠️ caveat to resolved note |

New issues to file (before executing §6 — see ordering constraint there):

| Repo | Description |
|------|-------------|
| `casehubio/aml` | Migrate `entityResolutionWorker` + `investigationSummaryWorker` in `AmlOversightCaseHub` to FuncWorkflowBuilder (PP-20260531 compliance — single-arg `WorkerResult.of(Map)`, not blocked; both are `private static Worker` — static qualifier retained since no instance state is captured) |
| `casehubio/engine` | Add PlannedAction support to FlowWorkerExecutor / FuncDSL — needed to unblock `entityLinkProposalWorker` migration in `AmlOversightCaseHub` (see §7 for constraint detail and API direction) |

## §1 — Migration

Both workers use the established FuncWorkflowBuilder pattern already present in the five
migrated workers. Because the SAR workers capture instance fields (`objectMapper`,
`complianceReviewLifecycle`), the methods remain `private Worker` (non-static).

**Structural change:**

```java
// Before (Sync)
.function((final Map<String, Object> input) -> {
    ...
    return WorkerResult.of(Map.of("sarNarrative", sarNarrative, "complianceTaskId", complianceTaskId));
})

// After (Flow)
.function(
    workflow("sar-drafting-junior")
        .tasks(
            function(s -> {
                @SuppressWarnings("unchecked")
                final Map<String, Object> input = (Map<String, Object>) s;
                // identical body
                return Map.of("sarNarrative", sarNarrative, "complianceTaskId", complianceTaskId);
            }, Map.class))
        .build())
```

Key differences from Sync:
- `WorkerResult.of(...)` wrapper removed — `executeFlow` calls `model.asMap()` on the workflow
  output and wraps it as `WorkerResult` internally. The flow path owns the wrapping.
- `WorkerExecutionContext.current().caseId()` is now safe in the flow path (engine#559)
- Instance fields captured by closure; lambda body is otherwise identical

Workflow names: `"sar-drafting-junior"` and `"sar-drafting-senior"` — consistent with the
naming pattern used by other workers (e.g., `"entity-resolution"`, `"pattern-analysis"`).

## §2 — Javadoc

Three locations require changes:

1. **Class-level Javadoc** — replace the entire `<p>Pure-computation workers use...` paragraph:

   Current:
   > `<p>Pure-computation workers use {@code FuncWorkflowBuilder.workflow().tasks(function(...)).build()}`
   > `per protocol PP-20260531-worker-func-exec. SAR drafting workers remain as`
   > `{@code WorkerFunction.Sync} pending engine support for {@link WorkerExecutionContext} in`
   > `the flow execution path (see #66).`

   Replace with:
   > `<p>All workers use {@code FuncWorkflowBuilder.workflow().tasks(function(...)).build()}`
   > `per protocol PP-20260531-worker-func-exec.`

   Removing only the SAR sentence would leave "Pure-computation workers use..." implying a
   non-pure-computation category still exists. After this migration that category is gone.

2. **`sarDraftingWorkerJunior()` Javadoc** — remove the paragraph starting:
   > "Remains as WorkerFunction.Sync (raw lambda) because WorkerExecutionContext.current()
   > is only set in the executeSync path..."

3. **`sarDraftingWorkerSenior()` Javadoc** — remove the equivalent paragraph and its
   `@link #sarDraftingWorkerJunior()` cross-reference.

The "Runs on a Quartz worker thread; JPA calls via ComplianceReviewLifecycle are safe here"
note on the junior worker remains accurate — the Flow executor runs on the same Quartz thread.

## §3 — CLAUDE.md Note

The current note reads:

> "Engine worker return type: `Worker.Builder.function()` requires
> `Function<Map<String, Object>, WorkerResult>`. Return `WorkerResult.of(Map.of(...))` — not
> `Map.of(...)` directly. Applies to all `YamlCaseHub` worker lambdas (casehubio/aml#54)."

After this migration, `AmlInvestigationCaseDescriptor` workers return `Map.of(...)` directly in
the flow path. "Applies to all YamlCaseHub worker lambdas" is now false for seven workers (five
already migrated, two migrated in this issue). A developer adding a new worker following this
note would incorrectly wrap with `WorkerResult.of()` in a Flow worker.

Update to:

> "Return type by execution model: `WorkerFunction.Sync` (raw lambda) workers — e.g.
> `AmlOversightCaseHub` — must return `WorkerResult.of(Map.of(...))`. `WorkerFunction.Flow`
> (FuncWorkflowBuilder) workers return `Map<String, Object>` directly; `executeFlow` calls
> `model.asMap()` and wraps internally. Do not wrap Flow worker returns."

## §4 — Garden Entry Update

File: `GE-20260609-ddd4b8.md` in `~/.hortora/garden/jvm/casehub-engine/`

The Fix section ends with a `⚠️ Caveat` block that documents the flow path gap. Revise to a
resolved note:

**Replace:**
> ⚠️ Caveat — `WorkerFunction.Flow` (FuncWorkflowBuilder) workers: `WorkerExecutionContext.current()`
> returns `null` inside FuncDSL lambdas...

**With:**
> ✅ Resolved (engine#559, 2026-06-23) — `DefaultWorkerExecutor.executeFlow()` now calls
> `WorkerExecutionContext.set(context)` before delegating to `FlowWorkerExecutor`, matching
> the behaviour of `executeSync()`. `WorkerExecutionContext.current().caseId()` is safe inside
> FuncDSL lambdas. See casehubio/aml#66 for the AML migration that removed the Sync workaround.

## §5 — Tests

### Unit test (change in AmlInvestigationCaseDescriptorTest)

`worker_execution_model_classification_is_exhaustive()` already exists and currently asserts
SAR workers are `WorkerFunction.Sync`. This is the TDD red signal — the test fails after the
migration until updated.

Collapse the two-set structure into one unified set (all 7 workers are now Flow):

```java
// All 7 workers now use WorkerFunction.Flow per PP-20260531-worker-func-exec
final Set<String> allWorkers = Set.of(
        "entity-resolution-agent",
        "pattern-analysis-agent",
        "osint-screening-agent",
        "osint-screening-agent-senior",
        "senior-analyst-agent",
        "sar-drafting-agent-junior",
        "sar-drafting-agent-senior");

for (final Worker w : descriptor.workers()) {
    if (allWorkers.contains(w.getName())) {
        assertInstanceOf(WorkerFunction.Flow.class, w.getFunction(),
                "Worker " + w.getName() + " must use WorkerFunction.Flow (FuncWorkflowBuilder).");
    } else {
        fail("Worker " + w.getName() + " is unclassified — add it to allWorkers.");
    }
}
```

Remove the comment "SAR drafting workers remain Sync pending engine WorkerExecutionContext
support (#66)" — it is no longer true.

### Integration test (existing, no changes)

`AmlLayer6InvestigationTest` exercises the full SAR drafting path, including
`WorkerExecutionContext.current().caseId()`. It does not explicitly assert `complianceTaskId` —
coverage is implicit: if `WorkerExecutionContext.current()` returns null, `caseId()` throws NPE,
the engine case fails, and the test times out at the `status=completed` poll. This is the
regression net for the migration — a passing Layer 6 test confirms the flow path is wired
correctly end-to-end.

Layer 9 (`AmlLayer9ActionGateTest`, `AmlLayer9ResourceTest`) tests `AmlOversightCaseHub` — no
SAR workers, no `complianceTaskId`, no coverage of this migration. Do not include Layer 9 tests
in migration validation.

Run validation:
```bash
# Unit test first (fast, no SNAPSHOT dependency)
mvn test -pl app -am -Dtest=AmlInvestigationCaseDescriptorTest -Dsurefire.failIfNoSpecifiedTests=false

# Integration test — only after confirming engine SNAPSHOT includes engine#559 (see Pre-requisite)
mvn test -pl app -am -Dtest=AmlLayer6InvestigationTest -Dsurefire.failIfNoSpecifiedTests=false
```

## §6 — ARC42STORIES.MD

**Ordering constraint:** File the two issues in the "New issues to file" table first. Write their
real numbers directly into the two `(aml#NNN, engine#NNN)` placeholders below before committing.
This produces one ARC42STORIES.MD commit, not two.

LAYER-LOG.md: no update required — this migration corrects known debt within Layer 5, not a layer
extension or new layer. The ARC42STORIES.MD update below is sufficient.

Three stale references to update:

**Line ~1050** (§9.4 Layer 5 descriptor bullet):
Collapse "5 Flow + 2 Sync" framing — the distinction was meaningful only while the migration was
blocked. Now all 7 use Flow:
> "...carries all 7 workers as `WorkerFunction.Flow` (FuncWorkflowBuilder per PP-20260531; SAR
> workers migrated in aml#66 after engine#559 fixed `WorkerExecutionContext` in `executeFlow`);
> testable without Quarkus; per protocol PP-20260518"

**Line ~1060** (Layer 5 narrative):
> "All 7 workers use `WorkerFunction.Flow` (FuncWorkflowBuilder per PP-20260531). SAR drafting
> workers migrated in aml#66 — engine#559 added `WorkerExecutionContext.set(context)` in
> `DefaultWorkerExecutor.executeFlow()`. `AmlOversightCaseHub` workers remain `WorkerFunction.Sync`
> pending engine PlannedAction support in flow path (aml#NNN, engine#NNN)."

**Line ~1462** (§12 risks/debt table):
Change status from "Partially resolved..." to:
> "✅ Resolved — all 7 workers in `AmlInvestigationCaseDescriptor` use `WorkerFunction.Flow`
> (aml#46 + aml#66). `AmlOversightCaseHub` workers tracked separately (aml#NNN, engine#NNN)."

## §7 — AmlOversightCaseHub: Scope Boundary and Constraint

`AmlOversightCaseHub` has three workers, all currently `WorkerFunction.Sync` — a PP-20260531
violation. Two are NOT blocked by this issue and should be tracked separately:

| Worker | Return type | Status |
|--------|-------------|--------|
| `oversight-entity-resolution-agent` | `WorkerResult.of(Map)` — single arg | **Migratable now** — `private static Worker`, static qualifier retained |
| `oversight-investigation-summary-agent` | `WorkerResult.of(Map)` — single arg | **Migratable now** — `private static Worker`, static qualifier retained |
| `oversight-entity-link-proposal-agent` | `WorkerResult.of(Map, PlannedAction)` — two args | **Blocked** |

`entityLinkProposalWorker` is blocked because `executeFlow` calls `model.asMap()` on the
workflow output — only the Map is surfaced, the `PlannedAction` has no pathway through the
FuncDSL chain. If migrated as-is, the PlannedAction is silently lost and the oversight gate
never fires.

The right fix is an engine-side change: FuncDSL needs a mechanism to attach a PlannedAction
to a function task — `function(lambda, Map.class).withPlannedAction(actionFn)` — so
`FlowWorkerExecutor` can include it in the wrapped `WorkerResult`. This requires a new engine
issue.

**API direction for the engine issue — `actionFn` input type:**

`actionFn` must be `Function<Map<String, Object>, PlannedAction>` where the argument is the
**task input** (the same `s` that `function(s -> ..., Map.class)` receives — the case context
at the point the worker is invoked). It must NOT receive the output.

Why: the current `entityLinkProposalWorker` output map is
`{proposedLink, entityType, riskScore}`. The PlannedAction metadata requires `ownershipChain`,
which is in the input (`entityResolution.ownershipChain`) but not in the output. An
output-typed `actionFn` cannot access `ownershipChain` and would require the output contract
to be widened — conflating what gets committed to the case context with what the PlannedAction
needs. The task input contains the complete context.

For single-task workflows (the current pattern), task input equals workflow input. For
multi-step workflows, `withPlannedAction` attaches to a specific task and receives that task's
input (the output of the prior step), not the original workflow input. This is the simpler
executor design: no special bookkeeping needed to thread the original input through the chain.
If a future use case needs the original workflow input in a later task's `withPlannedAction`,
restructure the workflow (e.g., pass the needed data through as a named output) rather than
adding a BiFunction variant.

**Actions:**
1. File `casehubio/aml` issue: migrate `entityResolutionWorker` + `investigationSummaryWorker`
   in `AmlOversightCaseHub` to FuncWorkflowBuilder (unblocked, PP-20260531 compliance)
2. File `casehubio/engine` issue: add PlannedAction support to FlowWorkerExecutor / FuncDSL
   (blocks `entityLinkProposalWorker` migration)

These are out of scope for aml#66 — tracked in the "New issues to file" table above.
