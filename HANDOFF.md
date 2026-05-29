# Handoff — casehub-aml Layer 6 complete
2026-05-29

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-05-29)

Layer 6 (trust-weighted routing) shipped to `casehub-aml` main (casehubio/aml upstream).

- `AmlTrustRoutingPolicyProvider` — per-capability thresholds via Preferences API (osint=0.70, sar=0.75, senior-analyst=0.80)
- `AmlTrustScoreSeeder` — seeds Beta(α,β) at `@Observes @Priority(20) StartupEvent`; calls `trustScoreCache.hydrate()` after
- `SarOutcomeFeedbackService` — writes `LedgerAttestation` (SOUND/FLAGGED) on SAR outcome
- `AmlLayer6Resource` — `/api/layer6/investigations` POST/GET/outcome endpoints
- Senior/junior worker variants for sar-drafting and osint-screening
- 20/20 Layer 6 tests pass; 3/5 `AmlLayer5InvestigationTest` errors from engine#396 (CDI regression, not our code)
- Blog: `blog/2026-05-29-mdp01-trust-loop-complete.md`
- Garden: GE-20260529-d7b6f8 (TrustBootstrapSource SPI never fires on fresh deployment)

## Issues filed this session

| Issue | Repo | What | Status |
|-------|------|------|--------|
| aml#38 | casehubio/aml | Layer 6 trust routing | ✅ closed |
| aml#39 | casehubio/aml | Minor quality (double deserialization, score validation) | open |
| engine#395 | casehubio/engine | Flyway scoping violation — V2000/V2001 at wrong path | open |
| engine#396 | casehubio/engine | CaseLedgerEntryRepository @ApplicationScoped beating selected-alternatives → breaks Layer 5 tests | open |
| parent#101 | casehubio/parent | casehub-aml.md Layer 6 status + new components | open |
| parent#102 | casehubio/parent | PLATFORM.md cross-dep map: add casehub-engine-ledger → casehub-aml row | open |

## Immediate next step

Start Layer 7 — comparison table vs IBM AMLSim. Check epic #9 for a child issue: `gh issue view 9 --repo casehubio/aml`. None exists — create one via issue-workflow Phase 1.

## What's left

- aml#39 — double deserialization in SAR workers, range validation for investigationAccuracyScore · S · Low
- engine#396 — CDI fix for CaseLedgerEntryRepository; once resolved, Layer 5 tests go green · S · Med
- engine#395 — Flyway scoping fix; once resolved, remove local V2002/V2003 copies from aml · XS · Low
- parent#101, parent#102 — peer repo doc updates · XS · Low (needs parent session)

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| L7 | Layer 7 — comparison table vs IBM AMLSim and industry whitepapers | M | Med | Final tutorial layer; no issue yet |

## References

- Blog: `blog/2026-05-29-mdp01-trust-loop-complete.md`
- Spec: `docs/specs/2026-05-29-layer6-trust-routing-design.md` (in project)
- Garden: GE-20260529-d7b6f8 (TrustBootstrapSource SPI)
