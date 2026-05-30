# Design Journal — issue-43-layer7-comparison

## §Layer 7 framing pivot — 2026-05-30

**Decision:** Layer 7 reframed from "comparison table vs IBM AMLSim and industry
whitepapers" (tutorial-contrast framing) to "compliance evidence — accountability
properties mapped against FinCEN/FATF requirements" (production-first framing).

**Why:** The tutorial/showcase framing was stripped from CLAUDE.md and LAYER-LOG.md
this session (commit 231d71e on issue-43-layer7-comparison). Layer 7 now describes
what the architecture *delivers* against regulatory requirements, not what it teaches
compared to alternatives. The comparison to IBM AMLSim and industry approaches may
still appear in the Layer 7 content, but as supporting evidence for the accountability
claims — not as the primary framing device.

**What this means for implementation:** The Layer 7 LAYER-LOG.md entry (stub already
added in 231d71e) needs to be completed with:
- The FinCEN/FATF requirements and how each layer closes them
- Accurate characterisation of what alternatives (AMLSim, naive agentic AML) do and
  don't provide — sourced from actual docs, not invented weaknesses
- No "tutorial contrast" framing; document the architecture's properties on their own terms

**Open:** The next session should approach Layer 7 fresh from the production-first
framing, not from the tutorial-strategy.md §6.1 comparison table as the primary input.
