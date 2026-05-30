# Handoff — Layer 7 reframed, new session ready
2026-05-30

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-05-30)

Tutorial/showcase framing stripped from `CLAUDE.md` and `LAYER-LOG.md` on branch
`issue-43-layer7-comparison` (commit 231d71e). Matching the cleanup already applied
to casehub-life.

- `CLAUDE.md` — Added "Agentic Harness Goals" (production-first goal + Arc42Stories
  migration note). Removed tutorial sentence from "What This Project Is". "Tutorial
  Structure" → "Foundation Layers". `tutorial-strategy.md` refs removed.
- `LAYER-LOG.md` — Header rewritten as architecture record. "What it shows" →
  "What it adds" in Layers 1, 2, 3, 5, 6. Java gap-comment code blocks replaced
  with accountability gap tables. Layer 7 stub added with new framing.
- **Layer 7 reframed**: "comparison table vs IBM AMLSim" → "compliance evidence —
  accountability properties mapped against FinCEN/FATF requirements". See
  `design/JOURNAL.md §Layer 7 framing pivot`.
- Issue #43 created. Epic #9 updated with all issue numbers (Layers 5, 6 were missing).

## Immediate next step

New session — brainstorm Layer 7 fresh from the production-first framing. The
question is: which FinCEN/FATF requirements does the architecture close, and which
layer closes each one? Not: how does CaseHub compare to IBM AMLSim as a tutorial
exercise.

Run `work-start` on `issue-43-layer7-comparison`, then `superpowers:brainstorming`.
Read `design/JOURNAL.md §Layer 7 framing pivot` first.

## What's left

*Unchanged — `git show HEAD~1:HANDOFF.md` §What's left*

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| L7 | Layer 7 — compliance evidence: FinCEN/FATF requirements mapped to layers | M | Med | Approach fresh; fetch IBM AMLSim README + AnChain/Sardine sources for accuracy |

## References

- Journal: `design/JOURNAL.md §Layer 7 framing pivot`
- Previous blog: `blog/2026-05-30-mdp01-two-mappers-one-exception.md`
- Garden: GE-20260530-3562b0 (from prior session)
