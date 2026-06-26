# Handoff — #70 closed: upstream SNAPSHOT migration (engine worker-api, qhorus ChannelCreateRequest)

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-25/26)

Closed branch `issue-70-upstream-snapshot-migration` covering #70.

1. **GITHUB_TOKEN** — set up classic PAT with `read:packages` scope for GitHub Packages resolution. `mvn -U dependency:resolve` now works.

2. **#70** — migrated AML to upstream SNAPSHOT breaking changes:
   - **Engine worker-api (#543/#567):** Worker primitives moved to `io.casehub.worker.api`, `FlowWorkerFunction` to `io.casehub.engine.flow`, `Worker`/`Capability` became records, `ActionRiskClassifier.classify()` gained `ClassificationContext`, `PlannedAction.context()` → `parameters()`.
   - **Qhorus #218:** `ChannelService.create()` consolidated into `ChannelCreateRequest.builder()`.
   - 9 files changed, all 183 tests pass.

3. **Root cause investigation** — engine main (`37b2eea8`) is broken; installed engine jars are from `issue-570` branch. `casehub-worker-api` version coupling between engine#543 and #567 caused multiple build failures before correct coordinated state was identified. Three garden entries submitted (GE-20260626-4a4790, GE-20260626-9ce1c9, GE-20260626-2e4a0d).

## Known issue — engine main broken

Engine main HEAD (`37b2eea8`) does not compile. The installed engine jars are from `issue-570-expression-engine-output-schema` branch. `casehub-worker-api` must be built from `72fc1ee` (bare marker `WorkerFunction`) to match. See GE-20260626-2e4a0d.

## Immediate next step

Pick #58 (separate sar-drafting from compliance-review-opening, M · Med).

## What's left

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #58 | Separate sar-drafting from compliance-review-opening | M | Med | Open, not blocked |
| #14 | Layer 10 — LLM supervisor mode | L | High | Blocked on engine#101 |

## References

- Main: `b6404d8` (ARC42 stale scan post-close)
- Branch closed: `issue-70-upstream-snapshot-migration` — EPIC-CLOSED.md committed
- Garden: GE-20260626-4a4790, GE-20260626-9ce1c9, GE-20260626-2e4a0d
