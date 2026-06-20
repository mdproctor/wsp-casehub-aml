*Updated: ledger#153, engine#533 closed — removed from backlog.*

# Handoff — CI fix: ledger SNAPSHOT adaptation (2026-06-17/18)

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-17 → 2026-06-18)

AML CI was red due to cascading ledger SNAPSHOT changes from the fix/#148 branch:

1. **Root cause diagnosed:** `ledger_subject_sequence` `PK` violated under concurrent Quartz workers — ledger's `LedgerSequenceAllocator.h2NextSequenceNumber()` used `MERGE INTO` which is not atomic for concurrent first-inserts in H2 2.x MVStore (MVCC). `LOCK_MODE=1` was tried and confirmed ineffective (MVStore ignores it). Fix required `INSERT ON CONFLICT DO NOTHING` + `UPDATE` in the ledger.

2. **Ledger fixed:** Filed `ledger#148`, ledger session applied fix at `5dc77ad`, pushed to `casehubio/ledger` at `19634f3` → `38d9f20`.

3. **Cascading SNAPSHOT breaks in AML:**
   - `TrustScoreCache` (removed) → `TrustScoreSource` SPI (`@DefaultBean`: `MaterializedTrustScoreSource`, reads DB fresh)
   - `LedgerErasureService.erase()` → now requires `ErasureReason` (second param)
   - `ActorIdentityProvider` moved to `api.spi` — affected qhorus too (filed `qhorus#285`, fixed at `c15807e`)
   - `LedgerHealthJob.checkSequenceGaps()` JPQL broken by Hibernate 6 stricter type checking (filed `ledger#153`, background timer, doesn't fail CI)
   - `BlackboardEventCodecRegistrar.onStart()` fires twice in some local `@QuarkusTest` contexts due to locally-installed engine WIP at `01:54` local build (filed `engine#533`, local-only, CI uses May-29 GitHub Packages blackboard)

4. **AML adapted and pushed:** `a3fcb83` — all five affected files updated. CI queued at `a3fcb83`.

## Immediate next step

Wait for CI at `a3fcb83` to complete — should go green. If it fails, check `gh run view` for the new failure category (unlikely with the published blackboard version).

Then: start **#14 (Layer 10 — LLM triage supervisor)** once engine#101 (LlmPlanningStrategy SPI) lands.

## What's left

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #14 | Layer 10 — LLM supervisor mode (investigation triage) | L | High | Blocked on engine#101 (LlmPlanningStrategy); CaseContextProvider → engine#419 |

## References

- Garden: GE-20260618-0ed34c (H2 LOCK_MODE=1 ineffective with MVStore)
- Filed: ledger#148 (sequence race fix — closed), ledger#153 (LedgerHealthJob JPQL), engine#533 (blackboard codec), qhorus#285 (ActorIdentityProvider — closed)
- AML commit: `a3fcb83` fix: adapt to casehub-ledger SNAPSHOT
