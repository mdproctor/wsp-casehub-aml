# Handoff — casehub-aml S/XS backlog cleared
2026-05-29

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-05-29)

Cleared all S/XS issues scoped to this repo.

- LAYER-LOG.md: build approach note (#36) + Vertical Slice Index S1–S5 + Participates in: on each layer + Layer 6 stub (#37)
- `SarOutcome` compact constructor: range validation + null guards for verdict/reason (#39)
- `AmlInvestigationCaseHub.buildSummary`: eliminated double `objectMapper.convertValue` on `SuspiciousTransaction`
- 9 tests in `SarOutcomeTest`; all Layer 6 tests pass (25/25)
- Blog: `blog/2026-05-29-mdp02-invariants-and-a-plan.md`

## Issues filed this session

| Issue | Repo | What | Status |
|-------|------|------|--------|
| aml#36 | casehubio/aml | build approach note in LAYER-LOG.md | ✅ closed |
| aml#37 | casehubio/aml | Vertical Slice Index in LAYER-LOG.md | ✅ closed |
| aml#39 | casehubio/aml | double deserialization + SarOutcome validation | ✅ closed |
| aml#40 | casehubio/aml | full Layer 6 entry in LAYER-LOG.md | open |
| aml#41 | casehubio/aml | IllegalArgumentExceptionMapper (REST 400 for invalid SarOutcome) | open |
| engine#395 | casehubio/engine | Flyway scoping violation — V2000/V2001 at wrong path | open |
| engine#396 | casehubio/engine | CaseLedgerEntryRepository CDI regression → breaks Layer 5 tests | open |
| parent#101, #102 | casehubio/parent | casehub-aml.md + PLATFORM.md updates | open |

## Immediate next step

Start Layer 7 — comparison table vs IBM AMLSim. No child issue exists under epic #9:
`gh issue view 9 --repo casehubio/aml` — create via issue-workflow Phase 1, then `work-start`.

## What's left

- aml#40 — full Layer 6 entry in LAYER-LOG.md (stub in place) · S · Low
- aml#41 — IllegalArgumentExceptionMapper for clean 400 on invalid SarOutcome · XS · Low
- engine#396 — CDI fix; once resolved, Layer 5 tests go green · S · Med
- engine#395 — Flyway scoping fix; once resolved, remove local V2002/V2003 copies · XS · Low
- parent#101, parent#102 — peer repo doc updates · XS · Low (needs parent session)

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| L7 | Layer 7 — comparison table vs IBM AMLSim and industry whitepapers | M | Med | No issue yet — create first |

## References

- Blog: `blog/2026-05-29-mdp02-invariants-and-a-plan.md`
- Garden: GE-20260529-d7b6f8 (TrustBootstrapSource SPI — from prior session)
