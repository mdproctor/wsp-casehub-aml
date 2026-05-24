---
layout: post
title: "Layer 5 was never blocked"
date: 2026-05-24
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
---

Coming back after Layer 4, the docs said Layer 5 — adaptive investigation paths, PEP routing, parallel checks — was blocked on engine P1.3. I'd accepted that at face value when I wrote it.

P1.3 is trust-weighted routing. That's a Layer 6 concern.

Layer 5 needs the engine to dispatch work, evaluate a case plan based on context, and support parallel capability targets. That's engine P0 — specifically engine#186, which closed weeks ago: wire the engine's COMMAND dispatch into the Qhorus commitment lifecycle so every work assignment becomes a normative obligation. Done.

We went and checked. Engine#336 and #337 are open — both about trust-weighted selection, specifically getting trust scores into `SelectionContext` and fixing `WorkOrchestrator` to resolve `WorkerSelectionStrategy` via CDI priority rather than concrete injection. Real work, not done. But none of it is a Layer 5 gate. Adaptive paths don't need trust scores. They need case plan evaluation, context-aware routing, and parallel task dispatch.

The label just drifted. At some point during planning, P1.3 got attached to Layer 5 because it's in the same general area — trust infrastructure for the engine. But trust-weighted *selection* is not the same as adaptive path *execution*. The engine can evaluate a case plan and route based on entity type without knowing anything about which analyst has the better track record.

Layer 5 is unblocked. I'm out of excuses.

There's a lesson here for any multi-repo project: "blocked by X" labels in cross-repo docs can quietly point at the wrong milestone for months. The version I'm switching to: name the specific capability the layer needs, not the milestone. "Requires: `WorkOrchestrator` dispatches COMMAND on task assignment" is harder to misread than "blocked: engine P1.3."

Layer 5 is a case plan YAML for AML investigation, PEP routing that fires when the flagged entity is politically exposed, parallel OSINT and entity resolution running simultaneously. No trust scoring yet. No LLM triage. Those are Layer 6 problems, properly gated.
