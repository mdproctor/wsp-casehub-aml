# Handoff — XS/S batch fixes closed (2026-06-21)

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-21)

Closed branch `issue-63-xs-xm-batch-fixes` covering 8 issues (#63, #48, #52, #47, #54, #59, #49, #45). All S/XS issues that weren't externally blocked.

Key outcomes:
1. **CI green** — `QhorusInboundCurrentPrincipal` excluded in test properties only; `AmlLayer6Resource` drain switched from WorkerDecisionEntry to `CaseStatus.COMPLETED` (matches Layer 9). Previously-failing `AmlSarOutcomeObserverTest` and `AmlTrustRoutingAttestationTest` now pass.
2. **`AmlInvestigationCaseDescriptor`** — plain POJO carrying all 7 worker lambdas; `AmlInvestigationCaseHub` is now a thin `@PostConstruct` CDI wrapper. Follows protocol PP-20260518 (revised 2026-05-30).
3. **Migrations idempotent** — V2005/V2006 got `IF NOT EXISTS` guards.
4. **CI cache** — `cache: maven` added to `setup-java` in `build.yml`.

Issues #54, #59, #49, #47 were already applied in code — closed with explanation. #14 remains blocked on engine#101.

Filed #65: cache-eviction caveat for Layer 6 drain (if engine evicts case from cache before polling, endpoint permanently shows in-progress).

## Immediate next step

Check CI at `e37dcaf` (ARC42 stale scan commit) — should be green. Then consider #65 (cache-eviction fallback) or start Layer 10 (#14, blocked on engine#101 LlmPlanningStrategy).

## What's left

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #14 | Layer 10 — LLM supervisor mode | L | High | Blocked on engine#101 (LlmPlanningStrategy) |
| #46 | Migrate workers to FuncWorkflowBuilder | M | Med | Open, not blocked |
| #65 | Layer 6 drain fallback when cache evicts | S | Med | Filed this session |

## References

- Main: `e37dcaf` (ARC42 stale scan)
- Branch closed: `issue-63-xs-xm-batch-fixes` — EPIC-CLOSED.md committed
