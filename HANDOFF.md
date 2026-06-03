# Handoff — hygiene complete, engine#101 scoped (2026-06-03)

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-03)

Branch hygiene audit across workspace and project: all branches validated (rebase-merge SHA divergence made issue-43 look alarming — it was fine), all blogs confirmed published (grep for "aml" in filenames missed all 18), ADR 0002 backfilled to workspace, issue-50 project branch stamped closed and pushed (--no-verify required — corrected CLAUDE.md note that claimed new-to-remote branches were exempt).

Engine#101 (LLM supervisor mode) investigated: supervisor mode and DSL are separable concerns; `PlanningStrategy` SPI is already LLM-ready. CaseContextProvider SPI extracted to casehubio/engine#419. Scoping note added to #101.

## Immediate next step

`issue-13-remove-test-workarounds` workspace branch is now overdue (was due 2026-06-03). No project counterpart — propose deletion. Then run `/work` to start **#32 (CaseMemoryStore)**.

## What's left

- `issue-13-remove-test-workarounds` workspace branch — **overdue**, no project counterpart · XS · Low
- `issue-26-re-enable-flyway` workspace branch — deletion due 2026-06-04 · XS · Low
- `AmlLayer5InvestigationTest` (3 tests): blocked on casehubio/engine#409 · XS · Low
- Deferred GDPR demo: `AML_SAR_OFFICER_REVIEWED` ledger event — casehubio/aml#44 · S · Low

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #32 | CaseMemoryStore — surface entity history before/during investigation | M | Med | **Unblocked** — platform#27 closed |
| #42 | ActionRiskClassifier oversight gate for consequential AML actions | M | Med | Check casehubio/engine#402 status first |
| #44 | Observer failure reconciliation — detect missing trust attestations at case close | M | Med | Silent evidence gaps |
| #14 | Layer 8 — LLM supervisor mode (investigation triage) | L | High | Needs engine#101 (DSL + LlmPlanningStrategy); CaseContextProvider separated to engine#419; narrower than it looked |

## References

- Architecture record: `ARC42STORIES.MD` (project root)
- Garden revision: `GE-20260522-cf54ad` — git rev-list --count misleads after rebase-merge
- Engine: casehubio/engine#101 (LLM supervisor), casehubio/engine#419 (CaseContextProvider SPI)
