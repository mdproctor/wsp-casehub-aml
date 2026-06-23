# Handoff — #66 closed: SAR workers migrated to FuncWorkflowBuilder

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-23)

Closed branch `issue-66-sar-workers-flow` covering #66.

1. **#66** — Both SAR drafting workers in `AmlInvestigationCaseDescriptor` migrated from `WorkerFunction.Sync` to `FuncWorkflowBuilder` (`WorkerFunction.Flow`). All 7 investigation workers now use Flow per PP-20260531. Engine#559 fixed the `WorkerExecutionContext` flow-path gap. Garden entry GE-20260609-ddd4b8 updated to resolved. CLAUDE.md return type note corrected (Sync vs Flow distinction). ARC42STORIES.MD synced at lines 1050, 1060, 1462.

2. **Filed aml#67** — migrate 2 of 3 `AmlOversightCaseHub` workers to Flow (unblocked).

3. **Filed engine#564** — add PlannedAction support to FlowWorkerExecutor / FuncDSL (blocks `entityLinkProposalWorker` migration). API direction: `withPlannedAction(Function<Map, PlannedAction>)` receives task input, not output.

4. **Garden GE-20260623-eb19c0** — `mvn -U` with GitHub Packages 401 silently corrupts local SNAPSHOT jars.

## Known issue — integration test failure (not caused by this branch)

`mvn install` (all tests) fails with "CaseInstance not found or wrong tenant" on all `@QuarkusTest` integration tests. Caused by the locally-rebuilt engine SNAPSHOT (from recovery after GitHub Packages 401 corrupted the jar). Engine main includes newer multi-tenancy enforcement not compatible with the AML test setup. Fix: restore correct published SNAPSHOT via `mvn -U` once GitHub Packages auth is resolved. Unit tests pass.

## Immediate next step

Fix GitHub Packages authentication so `mvn -U` resolves the correct published engine SNAPSHOT. Then re-run `mvn install` to confirm integration tests pass. After that, pick #67 (oversight worker migration, S · Low) or #58 (separate sar-drafting from compliance-review-opening, M · Med).

## What's left

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #67 | Migrate 2 oversight workers in AmlOversightCaseHub to Flow | S | Low | Unblocked, PP-20260531 compliance |
| #58 | Separate sar-drafting from compliance-review-opening | M | Med | Open, not blocked |
| #14 | Layer 10 — LLM supervisor mode | L | High | Blocked on engine#101 |

## References

- Main: `84a9ae2` (squashed feat commit)
- Branch closed: `issue-66-sar-workers-flow` — EPIC-CLOSED.md committed
- Blog: `2026-06-23-mdp01-four-rounds-two-workers.md`
- Garden: `GE-20260623-eb19c0` (SNAPSHOT corruption gotcha)
- Spec: `docs/specs/issue-66-sar-workers-flow/2026-06-23-sar-worker-flow-migration-design.md`
