# Design: Migrate SAR Drafting Workers to FuncWorkflowBuilder

**Issue:** casehubio/aml#66
**Branch:** issue-66-sar-workers-flow
**Date:** 2026-06-23
**Status:** Approved

## Context

`AmlInvestigationCaseDescriptor` contains 7 workers. Five were migrated to `FuncWorkflowBuilder`
(WorkerFunction.Flow) in casehubio/aml#46. The two SAR drafting workers were left as
`WorkerFunction.Sync` (raw lambda) because `DefaultWorkerExecutor.executeFlow()` did not call
`WorkerExecutionContext.set(context)`, causing `WorkerExecutionContext.current().caseId()` to
return null inside FuncDSL lambdas.

`casehubio/engine#559` is now closed. `executeFlow` now sets `WorkerExecutionContext` before
delegating to `FlowWorkerExecutor`, matching the behaviour of `executeSync`. The blocker is gone.

## Scope

Files changed in `app/`:

| File | Change |
|------|--------|
| `AmlInvestigationCaseDescriptor.java` | Migrate 2 workers; update Javadoc |
| `AmlInvestigationCaseDescriptorTest.java` | Change SAR worker assertion from `Sync` to `Flow`; update comments |

Project root:

| File | Change |
|------|--------|
| `ARC42STORIES.MD` | Update 3 stale references to SAR workers being `WorkerFunction.Sync` (lines 1050, 1060, 1462) |

Outside `app/`:

| Artifact | Change |
|----------|--------|
| Garden entry `GE-20260609-ddd4b8` | Revise stale ⚠️ caveat to resolved note |

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
    workflow("sar-drafting-junior")          // workflow name unique per worker
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
- `WorkerResult.of(...)` wrapper removed — `FlowWorkerExecutor` wraps the `Map` return internally
- `WorkerExecutionContext.current().caseId()` is now safe in the flow path (engine#559)
- Instance fields captured by closure; lambda body is otherwise identical

Workflow names: `"sar-drafting-junior"` and `"sar-drafting-senior"` — consistent with the
naming pattern used by other workers (e.g., `"entity-resolution"`, `"pattern-analysis"`).

## §2 — Javadoc

Remove the "blocked on engine" language from three locations:

1. **Class-level Javadoc** — remove the sentence:
   > "SAR drafting workers remain as `WorkerFunction.Sync` pending engine support for
   > `WorkerExecutionContext` in the flow execution path (see #66)."

2. **`sarDraftingWorkerJunior()` Javadoc** — remove the paragraph starting:
   > "Remains as WorkerFunction.Sync (raw lambda) because WorkerExecutionContext.current()
   > is only set in the executeSync path..."

3. **`sarDraftingWorkerSenior()` Javadoc** — remove the equivalent paragraph and its
   `@link #sarDraftingWorkerJunior()` cross-reference.

The "Runs on a Quartz worker thread; JPA calls via ComplianceReviewLifecycle are safe here"
note on the junior worker remains accurate — the Flow executor runs on the same Quartz thread.

## §3 — Garden Entry Update

File: `GE-20260609-ddd4b8.md` in `~/.hortora/garden/jvm/casehub-engine/`

The Fix section ends with a `⚠️ Caveat` block that documents the flow path gap. Revise to a
resolved note:

**Replace:**
> ⚠️ Caveat — `WorkerFunction.Flow` (FuncWorkflowBuilder) workers: `WorkerExecutionContext.current()`
> returns `null` inside FuncDSL lambdas. `DefaultWorkerExecutor.executeFlow()` does not call
> `WorkerExecutionContext.set()` before delegating to `FlowWorkerExecutor` — only `executeSync()`
> and `executeAgentSync()` set it. Workers migrated to `FuncWorkflowBuilder` that call
> `WorkerExecutionContext.current()` will get NPE at runtime. Either keep such workers as
> `WorkerFunction.Sync`, or request the engine team to add `WorkerExecutionContext.set(context)`
> in `executeFlow()`. (casehub-aml#66)

**With:**
> ✅ Resolved (engine#559, 2026-06-23) — `DefaultWorkerExecutor.executeFlow()` now calls
> `WorkerExecutionContext.set(context)` before delegating to `FlowWorkerExecutor`, matching
> the behaviour of `executeSync()`. `WorkerExecutionContext.current().caseId()` is safe inside
> FuncDSL lambdas. See casehubio/aml#66 for the AML migration that removed the Sync workaround.

## §4 — Tests

### Unit test (change in AmlInvestigationCaseDescriptorTest)

`worker_execution_model_classification_is_exhaustive()` already exists and currently asserts
SAR workers are `WorkerFunction.Sync`. This is the TDD red signal — the test fails after the
migration until updated.

Change the SAR worker branch from:
```java
} else if (sarWorkers.contains(w.getName())) {
    assertInstanceOf(WorkerFunction.Sync.class, w.getFunction(),
            "SAR worker " + w.getName() + " must remain Sync until engine"
                    + " provides WorkerExecutionContext in the flow path (#66).");
}
```

To (merge `sarWorkers` into `pureComputationWorkers` — all 7 workers now use Flow):
```java
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

Update the comment at the top of the test from the "SAR drafting workers remain Sync pending..."
note to reflect that all workers now use Flow per PP-20260531-worker-func-exec.

### Integration tests (existing, no changes)

The Layer 6 `@QuarkusTest` and Layer 9 `@QuarkusTest` both drain to `status=completed` and
verify `complianceTaskId` is present in the outcome. These tests exercise the full SAR
drafting path and will catch any behavioural regression from the migration.

Run with:
```bash
mvn test -pl app -am -Dtest=AmlInvestigationCaseDescriptorTest -Dsurefire.failIfNoSpecifiedTests=false
mvn test -pl app -am -Dtest=AmlLayer6InvestigationIT -Dsurefire.failIfNoSpecifiedTests=false
```

## §5 — ARC42STORIES.MD

Three stale references to update:

**Line ~1050** (§9.4 Layer 5 descriptor):
Change "...+ 2 SAR drafting workers as `WorkerFunction.Sync` (pending `WorkerExecutionContext`
support in `executeFlow` — aml#66)" to "...+ 2 SAR drafting workers migrated to
`WorkerFunction.Flow` in aml#66 (engine#559 resolved `WorkerExecutionContext` in flow path)".

**Line ~1060** (Layer 5 narrative):
Change "Two SAR drafting workers remain `WorkerFunction.Sync`..." to "All 7 workers now use
`WorkerFunction.Flow` (FuncWorkflowBuilder per PP-20260531). Engine#559 fixed
`WorkerExecutionContext` propagation in `executeFlow` (aml#66)."

**Line ~1462** (§12 risks/debt table):
Row for "Raw worker lambdas in AmlInvestigationCaseDescriptor":
Change status from "Partially resolved — 5/7 workers migrated..." to "✅ Resolved — all 7
workers use `WorkerFunction.Flow` (aml#46 + aml#66). Engine#559 fixed the flow-path
`WorkerExecutionContext` gap."

## Out of Scope

- CLAUDE.md note about `Worker.Builder.function()` requiring `WorkerResult.of(...)` —
  that note covers the raw lambda path (Sync), which still applies to other workers
  (e.g., in `AmlOversightCaseHub`). No change needed.
- LAYER-LOG.md — no layer opened or closed; this migration completes a known item within Layer 5.
  The ARC42STORIES.MD update (§5 above) covers the layer record sufficiently.
