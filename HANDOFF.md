# Handoff — #80/#79 closed (2026-06-29)

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-29)

Closed #80 (failure context on terminal InvestigationStatus — FailureContext/FailureEvent domain types, resolveFailureContext() querying EventLog with multi-CASE_FAULTED disambiguation, Layer6/Layer9 REST responses) and #79 (entity-level GDPR memory erasure — AmlErasureResult→ActorErasureResult rename, AmlEntityErasureLedgerEntry + V2013 migration, AmlErasureService.eraseEntity() + CaseMemoryStore.eraseEntity() orchestration with tamper-evident receipt, POST /api/entities/{entityId}/erasure endpoint). Design review ran 10 rounds ($32.42), caught BIGINT→UUID migration type error, SUSPENDED-is-not-a-failure semantic correction, multi-CASE_FAULTED disambiguation need. Filed deferred issues #83 (ledger content erasure) and #84 (cross-tenant erasure). Garden: revised GE-20260427-cc77a7 (resolved — .workItem() now exists), new GE-20260629-670471 (engine double-write of CASE_FAULTED).

## Immediate next step

No open branch. Pick next work from backlog — #81 (automated retention expiry) or #82 (GDPR Art.22 decision records) are natural follow-ons. #83 and #84 are deferred architectural concerns.

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #81 | Automated retention expiry (ErasureReason.RETENTION_EXPIRED) | M | Med | |
| #82 | GDPR Art.22 decision record compliance supplements | M | Med | |
| #83 | Entity data erasure in tamper-evident ledger content | L | High | Architectural limitation — deferred from #79 |
| #84 | Cross-tenant entity memory erasure | M | Med | Only when multi-tenancy activates |
| #72 | Gate rejection routing — re-open or close on MLRO rejection | M | High | Blocked on #14 / Layer 10 |
| #14 | Layer 10 — LLM supervisor mode | L | High | Blocked on engine#101 |

## References

- Spec: `docs/specs/issue-80-failure-context-erasure/2026-06-29-failure-context-entity-erasure-design.md`
- Blog: `blog/2026-06-29-mdp02-the-type-that-lies.md`
- Design review: `~/adr/casehub-aml/failure-context-entity-erasure-20260629-200629/tracker.md`
- Garden: GE-20260629-670471 (engine double-write of CASE_FAULTED), GE-20260427-cc77a7 (resolved)
