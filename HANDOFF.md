# Handoff — casehub-aml Layer 4 complete
2026-05-23

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## Current state

**Both repos on main.** issue-30-layer4-ledger-audit-trail closed and merged.

**Completed this session:**
- qhorus#184 shipped: `MessageDispatch` builder replacing `send()`, `DispatchResult` return type,
  `subjectId`/`causedByEntryId` propagation, builder validation for all 9 message types.
  Full design review (3 rounds), 104 test protocol-violation fixes, pushed to qhorus main.
- AML Layer 4 (issue #30 closed): `AmlInvestigationLedgerEntry` JPA entity, `AmlLedgerService`,
  V2001 Flyway migration, `AmlInvestigationResult.caseId`/`ledgerCaseEntryId`, `AmlInvestigator`
  updated to accept `caseId`, `QhorusAmlInvestigator` migrated to `dispatch()` with subjectId.
  30 tests pass.
- ADR-0001 filed: caseId as fresh UUID per investigation (vs deterministic UUID from tx string)
- 2 garden entries: stale qhorus snapshot → TABLE NOT FOUND, `@TestTransaction` + REQUIRED
  write invisibility
- All AML blog entries published to mdproctor.github.io

## Open issues

| Issue | What | Status |
|-------|------|--------|
| qhorus#190 | OutboundMessage needs inReplyTo + subjectId for PushAgentDispatch fan-out replies | open |
| aml#22 | fanOut() not triggering PushAgentDispatch.post() | skip — qhorus refactoring ongoing |
| casehub/parent#51 | casehub-aml.md Layer 4 status update | needs parent session |
| casehub/parent#48 | casehub-qhorus.md MessageDispatch API docs | needs parent session |

## What to build next

**Layer 5** — add casehub-engine: adaptive investigation path (PEP routing, parallel
specialist checks). Before starting: check for a child issue under Epic #9; create one
if absent (issue-workflow Phase 1).

Foundation gate: casehub-engine P0 (engine#186) is complete.

## References

- Blog: `blog/2026-05-23-mdp01-channel-ids-terrible-audit-keys.md`
- ADR: `adr/0001-case-id-as-investigation-aggregate-identifier.md`
- DESIGN.md: `DESIGN.md` — §AML domain ledger entries, §QhorusAmlInvestigator MessageDispatch API
