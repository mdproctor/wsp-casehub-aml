# Handoff — S-batch closed: #65 (cache fallback) + #46 (FuncWorkflowBuilder, partial)

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-22)

Closed branch `issue-65-s-batch-fixes` covering #65 and #46.

1. **#65** — `AmlLayer6Resource` and `AmlLayer9Resource` now fall back to `CaseInstanceRepository` when `CaseInstanceCache.get(caseId)` returns null (TTL eviction, JVM restart). Tests verify completed status survives `caseInstanceCache.clear()`.

2. **#46** — 5 of 7 workers in `AmlInvestigationCaseDescriptor` migrated to `WorkerFunction.Flow` (FuncWorkflowBuilder, PP-20260531). `casehub-engine-flow` added to pom. 2 SAR drafting workers remain Sync — `DefaultWorkerExecutor.executeFlow` doesn't call `WorkerExecutionContext.set()`, so `WorkerExecutionContext.current().caseId()` returns null inside FuncDSL lambdas. Tracked in #66. Garden entry `GE-20260609-ddd4b8` revised to document this gap.

## Immediate next step

Pick an open issue. Options: #66 (SAR worker migration — blocked on engine; file engine issue first), #58 (split SAR drafting from compliance-review-opening, M), or any other open item.

## What's left

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #14 | Layer 10 — LLM supervisor mode | L | High | Blocked on engine#101 (LlmPlanningStrategy) |
| #58 | Separate sar-drafting from compliance-review-opening | M | Med | Open, not blocked |
| #66 | Migrate SAR workers to FuncWorkflowBuilder | S | Low | Blocked on engine WorkerExecutionContext support in executeFlow |

## References

- Main: `b441b51` (ARC42 stale scan)
- Branch closed: `issue-65-s-batch-fixes` — EPIC-CLOSED.md committed
