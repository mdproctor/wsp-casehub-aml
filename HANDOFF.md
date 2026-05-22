# Handoff — casehub-aml Layer 3 test coverage parity
2026-05-22

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## Current state

**Both repos on main.** issue-24 closed and merged.

**Completed this session:**
- issue-24: SpecialistOutcome test coverage parity — all 9 combinations (3 variants × 3
  specialists) now tested in NaiveSarDraftingServiceTest
- NaiveAmlInvestigationServiceTest: sarNarrative now asserts transaction ID flows through
- Stale branches deleted (epic-layer3-qhorus, issue-13, issue-26 — remotes were already gone)
- Pre-existing AmlInvestigationResourceTest failure tracked as #29 (NoClassDefFoundError:
  ActorType — ledger-api not on @QuarkusTest classpath via qhorus transitive dep)
- #28 filed: OSINT Declined/Failed fixtures should be promoted to class-level fields (minor)

## Open issues

| Issue | What | Status |
|-------|------|--------|
| #22 | fanOut() not triggering PushAgentDispatch.post() | skip — qhorus/claudony refactoring in progress |
| #28 | OSINT Declined/Failed fixtures → class-level fields | low priority |
| #29 | AmlInvestigationResourceTest NoClassDefFoundError: ActorType | needs investigation |

## What to build next

**Layer 4** — add casehub-ledger as the explicit FinCEN audit trail. Ledger is already on
the classpath (transitively via qhorus). Before starting: check for a child issue under
Epic #9; create one if absent (issue-workflow Phase 1). Flyway is running real migrations —
Layer 4 ledger entries will land correctly in tests.

## References

- Blog: `blog/2026-05-22-mdp01-sealed-doesnt-mean-tested.md`
- DESIGN.md: `DESIGN.md` (workspace) — §Architecture
