# Handoff — Build fixed (#49), engine#409 is the remaining blocker
2026-06-01

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-01)

Build fix only. Two `@DefaultBean` CDI conflicts resolved via `quarkus.arc.exclude-types`: `DefaultTestPrincipal` (from `casehub-engine-persistence-memory` conflicting with `MockCurrentPrincipal`) and `NoOpGroupMembershipProvider` (from `casehub-work` conflicting with `MockGroupMembershipProvider`). Both conflicts were masked by stale Quarkus augmentation cache — only surfaced after `mvn clean`.

Third fix: `AmlLayer6Resource.recordOutcome()` now accepts `SarOutcomeRequest` DTO instead of `SarOutcome` directly. JAX-RS spec §4.2.4 converts `IOException` from message body readers to 400 without invoking any exception mapper — the compact constructor validation was unreachable. Moving validation into the resource method body makes `IllegalArgumentExceptionMapper` fire correctly.

Build result: 83 tests, 3 failures. All three are `AmlLayer5InvestigationTest` — `SchedulerService.registerScheduledTriggers()` calling `getCaseDefinition(caseInstance.getCaseMetaModel())` and getting null. Tracked as casehubio/engine#409. Not fixable here.

Arc42stories migration (LAYER-LOG.md → ARC42STORIES.MD) discussed, deferred. Spec and CaseHub profile are written and read. Decision: do it before starting Layer 8.

## Immediate next step

Clean up orphan `design/JOURNAL.md` from workspace main (leftover from issue-43-layer7-comparison), then pick up `#32` (CaseMemoryStore) — run `/work`.

## What's left

- Orphan `design/JOURNAL.md` on workspace main — `git rm design/JOURNAL.md` and commit · XS · Low
- `issue-13-remove-test-workarounds` workspace branch: 12 days stale, no project counterpart, **due 2026-06-03** · XS · Low
- Arc42stories migration: LAYER-LOG.md → ARC42STORIES.MD before Layer 8 begins · M · Low (reformatting, not rewriting)
- Deferred GDPR demo: `AML_SAR_OFFICER_REVIEWED` ledger event with human `actorId` — casehubio/aml#44 · S · Low
- `AmlLayer5InvestigationTest` (3 tests): blocked on casehubio/engine#409 — not fixable here · XS · Low

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #32 | CaseMemoryStore — surface entity history before/during investigation | M | Med | **Unblocked** — platform#27 closed |
| #42 | ActionRiskClassifier oversight gate for consequential AML actions | M | Med | Check casehubio/engine#402 status first |
| #44 | Observer failure reconciliation — detect missing trust attestations at case close | M | Med | Silent evidence gaps |
| #14 | Layer 8 — LLM supervisor mode (investigation triage) | L | High | Blocked on casehubio/engine#101 (`future`) |

## References

- Blog: `blog/2026-06-01-mdp01-two-conflicts-one-spec.md`
- Garden: GE-20260601-13fc26 (JAX-RS §4.2.4 MBR bypass), GE-20260601-0eb1b6 (Quarkus augmentation cache), GE-20260601-3dbc80 (quarkus.arc.exclude-types technique)
- Issue: casehubio/aml#49 (build fix), casehubio/engine#409 (remaining test failures)
