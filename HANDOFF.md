# Handoff — Hygiene complete, #32 unblocked
2026-05-31

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-05-31)

Full hygiene audit — workspace and project repos. All branches correctly stamped. Confirmed all 14 blog entries published. Found and fixed promotion gaps: blog entry `2026-05-30-mdp01-two-mappers-one-exception.md` added to workspace main, Layer 6 and Layer 7 plans promoted to workspace `plans/`, Layer 7 compliance evidence spec promoted to project `docs/specs/`, `journal-section-anchor` protocol (PP-20260519-0692ff) filed in garden HARNESS-INDEX, `issue-43` project branch pushed to remote. All code confirmed in project main (squash-merge SHAs differ but classes verified via `git ls-tree`).

`casehubio/platform#27` (CaseMemoryStore SPI) closed — aml#32 is now unblocked.

## Immediate next step

Pick up `#32` (CaseMemoryStore) — platform#27 is now closed, blocker is gone. Or pick up `#42` (ActionRiskClassifier) — check casehubio/engine#402 status first (was still open). Run `/work` to start.

## What's left

- Deferred GDPR demo: `AML_SAR_OFFICER_REVIEWED` ledger event with human `actorId` — casehubio/aml#44 · S · Low
- Pre-existing `quarkus:build` CDI failure (26 unsatisfied engine SPI deps) — not a regression · S · Med
- Workspace branches local-only (intentional — no code changes): `issue-13` (due 2026-06-03), `issue-26` (2026-06-04), `issue-30` (2026-06-06)

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #32 | CaseMemoryStore — surface entity history before/during investigation | M | Med | **Unblocked** — casehubio/platform#27 closed |
| #42 | ActionRiskClassifier oversight gate for consequential AML actions | M | Med | Check casehubio/engine#402 status first |
| #44 | Observer failure reconciliation — detect missing trust attestations at case close | M | Med | Silent evidence gaps |
| #14 | Layer 8 — LLM supervisor mode (investigation triage) | L | High | Blocked on casehubio/engine#101 (`future`) |

## References

- Blog: `blog/2026-05-31-mdp02-closed-isnt-clean.md`
- LAYER-LOG Layer 7: `LAYER-LOG.md §Layer 7`
- Garden: GE-20260531-2f51fd (pre-push hook asymmetry — `tools/`)
- Protocol filed: PP-20260519-0692ff (`casehub/journal-section-anchor.md` in garden)
