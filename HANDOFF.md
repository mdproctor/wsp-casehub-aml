# Handoff — casehub-aml Flyway re-enabled
2026-05-21

## What this project is

*Unchanged — `git show HEAD~5:HANDOFF.md` §What this project is*

## Current state

**Both repos on main.** All three issue branches closed (epic-layer3-qhorus, issue-13, issue-26).

**Completed this session:**
- Agentic harness framing corrected in 11 parent docs — CaseHub foundation is the harness, domain apps build on it
- issue-13: reactive workarounds removed (ledger#92 fixed root cause)
- issue-26: Flyway re-enabled on both datasources after three upstream fixes (qhorus#174, ledger#95, qhorus#180)
- Flyway locations pinned explicitly in test and main config; `quarkus-test-database.md` protocol updated
- WorkItemCreateRequest adapted to Builder pattern (issue #27, casehub-work SNAPSHOT)
- All 5 blog entries published to mdproctor.github.io

## Open issues

| Issue | What | Status |
|-------|------|--------|
| #22 | fanOut() not triggering PushAgentDispatch.post() | skip — qhorus/claudony refactoring in progress |
| #24 | Minor Layer 3 code quality | low priority |

## What to build next

**Layer 4** — add casehub-ledger as the explicit FinCEN audit trail. Ledger is already on the classpath (transitively via qhorus). Before starting: check for a child issue under Epic #9; create one if absent (issue-workflow Phase 1). Flyway is now running real migrations — Layer 4 ledger entries will land correctly in tests.

## References

- Blog: `blog/2026-05-21-mdp01-what-flyway-was-hiding.md`
- DESIGN.md: `DESIGN.md` (workspace) — §Architecture (Layer 3 + Flyway locations pinning)
- Branch deletion due: epic-layer3-qhorus 2026-06-01, issue-13 2026-06-03, issue-26 2026-06-04
