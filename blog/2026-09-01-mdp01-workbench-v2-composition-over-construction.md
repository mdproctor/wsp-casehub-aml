---
layout: post
title: "Workbench v2 — Composition Over Construction"
date: 2026-09-01
entry_type: note
subtype: diary
projects: [casehubio/aml]
tags: [casehub-pages, blocks-ui, dockWorkbench, scenario-automation, design]
series: issue-111-workbench-spec-rewrite
---

# Workbench v2 — Composition Over Construction

The AML workbench spec was two months old and already wrong. Not in its requirements — those held up well. Wrong in its assumptions about what we'd need to build.

The original spec, written in late June, designed custom components from primitives. A hand-rolled sidebar with hash routing. Manual fetch calls and render loops for every operations metric. A custom iframe graph for the investigation flow. Custom simulation APIs with `@IfBuildProperty` gating. Nearly a thousand lines of operations view code doing what blocks-ui components now do out of the box.

I hadn't looked at pages or blocks-ui in weeks. When I did, the gap was striking. Pages had grown a full scenario orchestrator — YAML scripts driving both backend actions and browser automation via ARIA targeting. blocks-ui had shipped `trust-workbench`, `approval-gate`, `compliance-summary`, `audit-trail-viewer`, `gdpr-erasure-action`, and a dozen other components that the old spec was planning to build by hand. The `casehub-diagram` package meant the investigation flow visualisation — the most complex custom component in the original spec — was a composition exercise, not a rendering project.

The survey surfaced something more fundamental than a component inventory update. The entire architectural approach needed to change. The old spec was a construction project: build these views, write these renderers, wire these fetch loops. The new spec is a composition project: which existing components go where, what thin wiring connects them, and what's genuinely AML-specific.

Three design principles came out of the rewrite:

1. **Compose, don't build** — if blocks-ui or pages has the component, use it
2. **Enhance shared, don't fork** — if a shared component almost fits, enhance it in blocks-ui rather than building an AML-specific workaround
3. **Domain code is the residual** — only code that requires knowledge of AML regulation or SAR filing belongs in this repo

The layout migrated from a custom Lit sidebar to the Pages `dockWorkbench` DSL — an IDE-style layout with collapsible dock panels on all four edges. This wasn't just a visual change. `dockWorkbench` unlocks scenario automation (the orchestrator needs the Pages component tree), state persistence (panel positions survive page reloads), and deferred rendering (panels don't render until first opened). The old sidebar gave us none of that.

One non-obvious finding: `blocks-split-workbench` only accepts a single `selection-topic`. The centre panel needs to switch between investigation detail, worker task workspace, and work item detail depending on which left dock panel is active. The solution is a topic-switching `aml-centre` component that listens for `pages-dock-toggle` events — an undocumented CustomEvent dispatched by the dock bar buttons with `{ panelId, visible }` detail. The centre renders different content per mode and publishes a synthetic `investigation-context` bridge topic so the right dock panels update regardless of which navigation mode is active.

The domain worker workbench — issue #110, the biggest design gap from the original spec — falls out naturally from this composition approach. A generic `worker-task-pane` in blocks-ui provides the task queue, investigation context, and response form. AML provides only the specialist workspace content — five small Lit components, one per specialist type, that render domain-specific data cards and forms. The specialist workspace is the only genuinely AML-specific part.

The implementation plan has eight batches. Batch 1 landed in this session — the dockWorkbench shell with investigation list navigation. The old `aml-app.ts` sidebar is replaced by `layout.ts` + `centre.ts` + `investigation-nav.ts`, backed by stub panels for the remaining seven batches. The investigation list and detail tabs work in the new layout.

What opens up: scenario automation makes the demo vehicle use case real — narrated YAML walkthroughs of PEP investigations, gate approvals, and GDPR erasure, with no custom simulation code. The push update wiring (Pages `EventConnection` → SSE endpoints) means the operations dashboard updates in real time. And the composition-first approach means most of the "implementation" is wiring existing components — the creative work was the spec, not the code.
