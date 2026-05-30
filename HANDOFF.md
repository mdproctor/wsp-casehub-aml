# Handoff — casehub-aml backlog cleared, Layer 7 next
2026-05-30

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-05-30)

Closed all remaining S/XS backlog items for this repo on branch `issue-40-layer6-log-entry`.

- **#40** — Full Layer 6 entry in LAYER-LOG.md; replaced the stub with all sections following the Layer 5 pattern. Code review caught Beta seed values were wrong for 5 of 7 workers (corrected to exact per-worker values). Blog: `blog/2026-05-30-mdp01-two-mappers-one-exception.md`
- **#41** — `IllegalArgumentExceptionMapper` + `JsonMappingExceptionMapper` in `io.casehub.aml.rest`. Jackson wraps `IllegalArgumentException` from record compact constructors as `ValueInstantiationException` — requires two mappers in tandem. 6 unit tests. Garden entry GE-20260530-3562b0 filed.
- **Protocols**: PP-20260530-5354d0 (two-mapper pattern, application scope) + PP-20260530-2fd788 (`@Provider @ApplicationScoped`, universal)

Both issues closed. Project branch and fork on main. Upstream casehubio/aml on main.

## Immediate next step

Layer 7 — comparison table vs IBM AMLSim. No issue exists yet under epic #9:
`gh issue view 9 --repo casehubio/aml` — create via issue-workflow Phase 1, then `work-start`.

## What's left

- engine#396 — CDI fix for CaseLedgerEntryRepository; once resolved, Layer 5 tests go green · S · Med
- engine#395 — Flyway scoping fix; once resolved, remove local V2002/V2003 copies · XS · Low
- parent#101, parent#102 — peer repo doc updates · XS · Low (needs parent session)

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| L7 | Layer 7 — comparison table vs IBM AMLSim and industry whitepapers | M | Med | No issue yet — create first |

## References

- Blog: `blog/2026-05-30-mdp01-two-mappers-one-exception.md`
- Garden: GE-20260530-3562b0 (ExceptionMapper<IllegalArgumentException> doesn't catch Jackson-wrapped ValueInstantiationException)
