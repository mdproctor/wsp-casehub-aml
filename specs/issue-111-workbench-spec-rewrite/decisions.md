# AML Workbench Spec Rewrite — Decisions

## D1: Layout architecture

**Choice:** dockWorkbench migration (Pages DSL)
**Alternatives:**
- Extend current Lit app — faster but no scenario automation, tutorials, or state persistence
- Hybrid (dockWorkbench shell, existing panels) — gets DSL benefits but doesn't address internal panel composition
**Rationale:** Full dockWorkbench migration unlocks scenario automation, tutorial system, state persistence, and deferred rendering — all required for the demo vehicle use case.
**Trade-offs:** Requires rewriting the app entry point and navigation; existing hash routing is replaced by Pages navigation.
**Sources:** GE-20260804-befd45 (dockWorkbench decomposition), GE-20260804-84ac70 (Pages DSL convention), GE-20260805-e3211c (hostPanel + registerPanel pattern)
**Exploration:** quick
**Status:** captured

## D2: Operations view

**Choice:** Replace with blocks-ui components (trust-workbench, approval-gate, kpi-metric-row, sla-indicator)
**Alternatives:**
- Keep existing + add alongside — no regression risk but duplicate implementations
- Keep as-is — functional but ~950 lines of hand-rolled code with blocks-ui equivalents available
**Rationale:** Drops ~700 lines of custom CSS/fetch/render. SAR Quality tab stays as custom panel (AML-specific). Shared components get tested across multiple harnesses.
**Trade-offs:** Requires learning blocks-ui component APIs; any missing features need blocks-ui enhancement rather than local workaround.
**Sources:** blocks-ui survey (trust-workbench, approval-gate, kpi-metric-row, sla-indicator)
**Exploration:** quick
**Status:** captured

## D3: Simulation / demo mode

**Choice:** Pages scenario orchestrator (YAML scripts + @ScenarioAction backend SPI + ARIA browser automation)
**Alternatives:**
- Hybrid (orchestrator + lightweight seed endpoint) — two mechanisms for different use cases
- Custom simulation API — independent of Pages lifecycle but misses demo/tutorial integration
**Rationale:** Scenario orchestrator drives both frontend (ARIA targeting) and backend (@ScenarioAction Java methods). YAML scripts replace custom seed APIs, @IfBuildProperty gating, and the determinism mechanism. Library view replaces scenario selector. Tutorials provide narrated walkthroughs for sales/demo.
**Trade-offs:** No custom `/api/simulation/seed` endpoint — dev data population requires running a scenario script rather than a curl command. Acceptable because the orchestrator has REST endpoints for programmatic scenario execution.
**Sources:** Pages survey (ScenarioOrchestrator, ScenarioCompiler, ScenarioHandler, ScenarioLibraryResource)
**Exploration:** quick
**Status:** captured

## D4: Investigation flow visualization

**Choice:** blocks-diagram-workbench with runtime overlay (CaseRuntimeState decorations)
**Alternatives:**
- Keep blocks-timeline (existing) — simpler but doesn't show parallel branches or adaptive routing
- Both timeline + diagram — two views of same data, more complete but more work
**Rationale:** Diagram workbench shows the adaptive investigation path as a DAG with per-specialist status badges, trust scores at routing time, and worker drill-down. graph-stencil-case already has Worker/Binding/Milestone/Goal stencils with runtime overlay support.
**Trade-offs:** More complex than the timeline view; requires a backend endpoint that reconstructs the flow DAG from engine event log.
**Sources:** blocks-ui survey (diagram-workbench, graph-stencil-case, CaseRuntimeState)
**Exploration:** quick
**Status:** captured

## D5: Domain worker workbench (#110) — component ownership

**Choice:** Build a generic `worker-task-pane` in blocks-ui that composes task queue + investigation context + specialist workspace slot
**Alternatives:**
- Pages-level worker abstraction (workerWorkbench() builder) — more opinionated, higher-level
- AML-specific composition only — no new shared component, composition is the domain logic
**Rationale:** The task queue (filtered by capability tag via qhorus COMMAND) and investigation context panel are generic across all CaseHub applications. The specialist workspace content is domain-specific. blocks-ui is the right level — it's where work-item-inbox and other workbench components live.
**Trade-offs:** Requires blocks-ui contribution; specialist workspace content is still AML-specific code.
**Sources:** Old spec #110 requirements, blocks-ui work-item-inbox/workbench pattern
**Exploration:** quick
**Status:** captured

## D6: Detail panel replacement

**Choice:** Replace everything possible with blocks-ui equivalents; enhance shared components where they fall short
**Alternatives:**
- Replace only where blocks-ui has direct equivalents — less risk, some custom code remains
- Keep existing panels — working code, focus effort on new capabilities
**Rationale:** Maximum reuse of shared components. Where blocks-ui components don't quite fit, enhance them rather than maintaining parallel implementations. Only genuinely AML-specific rendering stays in the AML app.
**Trade-offs:** Requires blocks-ui enhancements for any gaps; more cross-repo work.
**Sources:** blocks-ui survey (audit-trail-viewer, compliance-summary, trust-workbench, routing-rationale, gdpr-erasure-action)
**Exploration:** quick
**Status:** captured

## D7: Dock layout structure

**Choice:** IDE-style: centre (split-workbench) + left dock (navigation) + right dock (contextual detail) + bottom dock (operations)
**Alternatives:**
- Tab-based centre with side panels — simpler mental model, each tab self-contained
- Full dockWorkbench with all panels dockable — maximum flexibility, harder tutorial guidance
**Rationale:** Mimics IDE-style layout familiar to enterprise Java developers. Centre always has list-to-detail flow. Left dock switches navigation context (investigations, worker tasks, work items). Right dock shows contextual information driven by centre selection. Bottom dock shows system-wide metrics.
**Trade-offs:** More structured than a fully flexible dock; guided walkthrough benefits from predictable layout.
**Sources:** GE-20260804-befd45 (dockWorkbench primitives), GE-20260804-84ac70 (Pages DSL layout convention)
**Exploration:** quick
**Status:** captured

## D8: Centre panel design

**Choice:** blocks-split-workbench in centre — left half shows list (investigations or worker tasks), right half shows selected item detail/diagram/workspace
**Alternatives:**
- Tabbed centre with dynamic content — more screen real estate per view, loses list-detail pairing
- Context-driven centre swap — each nav selection swaps entire centre content, maximum space
**Rationale:** Consistent interaction pattern: the centre always pairs a list with a detail view. Left nav dock selection changes which list feeds the split. Right dock provides additional context. Users always know where to find the list, where to find the detail.
**Trade-offs:** Less screen real estate for detail than a full-width view; mitigated by collapsible dock panels.
**Sources:** Existing aml-app.ts already uses blocks-split-workbench for investigations — extending the pattern.
**Exploration:** quick
**Status:** captured

## D9: Backend scope

**Choice:** Frontend + backend API contracts in the same spec
**Alternatives:**
- Frontend only, reference existing types — keeps spec focused but risks mismatch
- Frontend only, defer backend entirely — enables full UI dev with mock data, backend separate
**Rationale:** The existing types.ts already defines most response shapes. The spec validates and extends them. Having both frontend and backend in one spec ensures consistency and enables a single implementation plan.
**Trade-offs:** Larger spec; implementation touches both frontend and backend code.
**Sources:** Existing types.ts, old spec §New API Endpoints Summary
**Exploration:** quick
**Status:** captured

## D10: Push updates and auth

**Choice:** Include push updates (Pages EventConnection), defer auth/RBAC
**Alternatives:**
- Include both push and auth — full production readiness in one spec
- Defer both — focus on composition and scenario automation only
**Rationale:** Pages push wire protocol (WebSocket → SSE fallback) is available and natural to wire during the dockWorkbench migration. Auth is still a separate concern — gate actions are already API-protected by WorkItem candidateGroup matching. UI role filtering is a follow-up.
**Trade-offs:** No role-based UI filtering — all views visible to all users until auth is added.
**Sources:** Pages survey (EventConnection, PushSource, WebSocketSource)
**Exploration:** quick
**Status:** captured
