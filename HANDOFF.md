# Handoff — #74/#75/#77 closed (2026-06-29)

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-29)

Closed three issues on one branch: #74 (consolidate completion detection into `AmlInvestigationOutcomeService` with `InvestigationResolution` domain abstraction, 404 for nonexistent cases), #75 (rejection reason capture from `WorkItemLifecycleEvent.detail()` through ledger to `InvestigationOutcome.reason`), #77 (test gaps — sequenceNumber tiebreaker, failure-only scenario, Layer 9 integration tests). Adversarial design review ran 5 rounds (19 issues) — caught Merkle hash backward compat, Jackson mixin for api/ zero-framework constraint, and service package move from compliance/ to engine/. Also filed #78 (FAULTED/CANCELLED mapped to IN_PROGRESS — pre-existing, not a regression) and #73 closed as duplicate of #75.

## Immediate next step

No open branch. Pick next work from backlog — #62 (GDPR erasure receipt) is unblocked. #14 (Layer 10 LLM supervisor) still blocked on engine#101.

## What's left

- #78 — InvestigationStatus should distinguish FAULTED/CANCELLED from IN_PROGRESS · XS · Low

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #62 | GDPR Art.17 erasure receipt — tamper-evident record | M | Med | Unblocked |
| #78 | InvestigationStatus FAULTED/CANCELLED handling | XS | Low | |
| #72 | Gate rejection routing — re-open or close on MLRO rejection | M | High | Blocked on #14 / Layer 10 |
| #14 | Layer 10 — LLM supervisor mode | L | High | Blocked on engine#101 |

## References

- Spec: `docs/specs/issue-74-completion-fixes/2026-06-29-completion-fixes-design.md`
- Blog: `blog/2026-06-29-mdp02-the-missing-abstraction.md`
- Garden: GE-20260629-0d981d (Layer 9 has no SAR outcomes — oversight gate only)
- Adversarial review: `~/adr/casehub-aml/completion-fixes-20260629-034108/`
