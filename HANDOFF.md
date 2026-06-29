# Handoff — #78/#62 closed (2026-06-29)

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-29)

Closed two issues on one branch: #78 (InvestigationStatus exhaustive projection — FAILED, CANCELLED, SUSPENDED replace the lossy if-check with a no-default switch expression) and #62 (GDPR Art.17 erasure receipt — AmlErasureService wraps LedgerErasureService, GdprErasureRequirement upgraded from static booleans to live config/receipt queries, erasure receipt enabled via foundation V1010 migration). Design review (7 rounds, 24 issues, $18.11) caught the actorId/entityId mismatch that would have made memory erasure a silent no-op — removed from the design. Filed ledger#160 (countByTenant SPI method, shipped same session). Filed 4 deferred issues: #79 (entity-level memory erasure), #80 (failure context on terminal status), #81 (retention expiry), #82 (Art.22 decision records).

## Immediate next step

No open branch. Pick next work from backlog — #79 (entity-level memory erasure) is the natural follow-on from #62. #80 (failure context) is a follow-on from #78.

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #79 | Entity-level memory erasure (GDPR by account ID) | M | Med | Follow-on from #62 |
| #80 | Failure context on terminal InvestigationStatus | S | Low | Follow-on from #78 |
| #81 | Automated retention expiry | M | Med | |
| #82 | GDPR Art.22 decision record compliance supplements | M | Med | |
| #72 | Gate rejection routing — re-open or close on MLRO rejection | M | High | Blocked on #14 / Layer 10 |
| #14 | Layer 10 — LLM supervisor mode | L | High | Blocked on engine#101 |

## References

- Spec: `docs/specs/issue-78-status-erasure/2026-06-29-status-erasure-design.md`
- Blog: `blog/2026-06-29-mdp02-the-projection-not-a-lifecycle.md`
- Design review: `~/adr/casehub-aml/status-erasure-20260629-*/tracker.md`
- Garden: GE-20260628-6599e6 (actor-scoped erasure receipt queries fail post-tokenisation)
