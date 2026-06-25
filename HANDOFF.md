# Handoff — #67 closed: oversight workers migrated to FuncWorkflowBuilder

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-25)

Closed branch `issue-67-oversight-worker-flow` covering #67.

1. **#67** — `entityResolutionWorker` and `investigationSummaryWorker` in `AmlOversightCaseHub` migrated from `WorkerFunction.Sync` to `FuncWorkflowBuilder` (`WorkerFunction.Flow`). `entityLinkProposalWorker` remains Sync (blocked on engine#564 — PlannedAction not supported in Flow). Added `AmlOversightCaseHubTest` with exhaustive execution model classification guard.

2. **Engine SNAPSHOT fix** — root-caused "CaseInstance not found or wrong tenant" from previous session. Cause: local `mvn install` from engine `issue-543` branch polluted `.m2` with incompatible SNAPSHOTs. Fix: rebuild engine from correct commit. Garden GE-20260624-e3ffa7 submitted.

3. **ARC42STORIES.MD + CLAUDE.md synced** — worker migration status updated (2/3 oversight workers now Flow). qhorus#190 stale reference resolved.

## Known issue — GitHub Packages 401 + .m2 pollution

`GITHUB_TOKEN` environment variable is not set, causing all GitHub Packages requests to return 401. The `.m2` local SNAPSHOT cache was cleaned during this session (549 files removed) to fix engine branch pollution, but this also removed modules that had no GitHub-cached fallback (casehub-work, casehub-engine-common, etc.). Full `mvn install` on AML currently fails with dependency resolution errors.

**Fix:** Set `GITHUB_TOKEN` in shell profile, then `mvn -U -f /Users/mdproctor/claude/casehub/aml/pom.xml dependency:resolve` to pull all published SNAPSHOTs. Alternatively, rebuild dependency chain bottom-up from local clones: parent → platform → eidos → connectors → ledger → qhorus → work → engine → aml. All repos must be on commits matching the published SNAPSHOTs (pre-June breaking changes on engine, qhorus main).

## Immediate next step

Fix `GITHUB_TOKEN` so `mvn -U` resolves published SNAPSHOTs. Verify `mvn install` passes. Then pick #58 (separate sar-drafting from compliance-review-opening, M · Med).

## What's left

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #58 | Separate sar-drafting from compliance-review-opening | M | Med | Open, not blocked |
| #14 | Layer 10 — LLM supervisor mode | L | High | Blocked on engine#101 |

## References

- Main: `a9365c1` (squashed feat commit for #67)
- Branch closed: `issue-67-oversight-worker-flow` — EPIC-CLOSED.md committed
- Garden: `GE-20260624-e3ffa7` (local mvn install from feature branch overrides published SNAPSHOT)
