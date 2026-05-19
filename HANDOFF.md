# Handoff — casehub-aml post-epic-close
2026-05-19

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## Current state

**Epic branch:** `epic-layer3-qhorus` (both repos — EPIC-CLOSED.md written, deletion due 2026-06-01)
**Project main:** fully merged — all Layer 3 code, specs promoted to `docs/specs/`, DESIGN.md created

Layers 1, 2, and 3 complete. Epic closed this session.

**Also done this session:**
- Agentic harness framing corrected in parent (11 files) — CaseHub foundation is the harness; domain apps build on it. aml CLAUDE.md updated to match.
- Protocol PP-20260519-0692ff: JOURNAL.md entries need `§SectionName` anchor in header or they're silently skipped at epic close merge
- Garden GE-20260519-c93fd8: `git add -A` after `git checkout <branch> -- <files>` sweeps all untracked files, not just checked-out ones

## Open issues

| Issue | What | Status |
|-------|------|--------|
| #13 | Remove test workarounds — Flyway V2 conflict + qhorus reactive suppression | **do next** |
| #22 | fanOut() not triggering PushAgentDispatch.post() | **skip** — qhorus/claudony refactoring in progress; fan-out API may change |
| #24 | Minor Layer 3 code quality | low priority |

## What to build next

**#13 first** — quick win. Check whether casehubio/qhorus#141, #142 and casehubio/work#162 are resolved, then remove the workaround properties from `app/src/test/resources/application.properties` and verify `mvn verify -pl app -am` passes.

**Then Layer 4** — add casehub-ledger as the explicit FinCEN audit trail. Before starting: check for a child issue under Epic #9; create one if absent (issue-workflow Phase 1).

## References

- Blog: `blog/2026-05-19-mdp01-naming-the-harness.md`
- DESIGN.md: `DESIGN.md` (workspace) — §Architecture
- Protocol: `docs/protocols/casehub/journal-section-anchor.md`
- Epic marker: `EPIC-CLOSED.md` on workspace epic branch
