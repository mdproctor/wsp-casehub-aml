# Handoff — casehub-aml Layer 5 ready
2026-05-24

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## Current state

**Both repos on main.** Layer 5 confirmed unblocked this session.

**Completed this session:**
- parent#51 closed: casehub-aml.md updated — Layer 4 marked complete (2026-05-23),
  Layer 5 corrected from `blocked: engine P1.3` to `engine P0 ✅ complete — unblocked`,
  Layer 6 gate corrected to `engine#336, #337` (trust-weighted selection, genuinely open)
- GE-20260524-c66b05: garden entry — tutorial layer blocking labels can silently
  point at the wrong milestone

## Open issues

| Issue | What | Status |
|-------|------|--------|
| qhorus#190 | OutboundMessage needs inReplyTo + subjectId for PushAgentDispatch fan-out replies | open |
| aml#22 | fanOut() not triggering PushAgentDispatch.post() | skip — qhorus refactoring ongoing |
| casehub/parent#48 | casehub-qhorus.md MessageDispatch API docs | needs parent session |

## What to build next

**Layer 5** — add casehub-engine: adaptive investigation path (PEP routing, parallel
specialist checks). Engine P0 (engine#186) is complete — unblocked.

**Immediate next step:** check for a Layer 5 child issue under Epic #9 (`gh issue view 9 --repo casehubio/aml`). None existed at end of this session — create one via issue-workflow Phase 1 before writing any code.

## References

- Blog: `blog/2026-05-24-mdp01-layer-5-was-never-blocked.md`
- ADR: `adr/0001-case-id-as-investigation-aggregate-identifier.md`
