# AML Workbench v2 — Design Spec

**Date:** 2026-09-01
**Status:** Draft
**Epic:** casehubio/aml#111
**Scope:** Full AML investigation workbench — dockWorkbench migration, blocks-ui composition, scenario automation, domain worker workbench

---

## Purpose

A production AML investigation workbench built on casehub-pages and blocks-ui. Serves three roles:

1. **Operational tool** — compliance officers, MLROs, specialists, and ops staff use it daily
2. **Demo vehicle** — step-through walkthroughs with scenario automation and narrated tutorials
3. **Sales showcase** — the app demonstrates CaseHub's accountability model with zero custom demo code

This is a **rewrite** of the June 2026 spec. The original designed custom components from primitives. This spec **composes** from blocks-ui and pages components that now exist, identifies gaps that require shared library enhancements, and confines AML-specific code to genuinely domain-specific concerns.

### Design principles

1. **Compose, don't build** — if blocks-ui or pages has the component, use it
2. **Enhance shared, don't fork** — if a shared component almost fits, enhance it in blocks-ui/pages rather than building an AML workaround
3. **Domain code is the residual** — only code that requires knowledge of AML, FinCEN, or SAR filing belongs in this repo

---

## Tech Stack

- **casehub-pages** — dockWorkbench DSL, scenario orchestrator, push wire protocol, hostPanel registry
- **blocks-ui** — split-workbench, list-pane, detail-pane, work-item-inbox, work-item-workbench, trust-workbench, approval-gate, kpi-metric-row, sla-indicator, compliance-summary, gdpr-erasure-action, audit-trail-viewer, routing-rationale, channel-activity, diagram-workbench, blocks-timeline
- **Quinoa** — Quarkus extension serving the frontend from `app/src/main/webui/`
- **esbuild** — fast rebuild for dev mode

---

## Layout Architecture

IDE-style dockWorkbench with four zones:

```
┌─────────────────────────────────────────────────────┐
│                    Dock Bar (top)                    │
├────────┬──────────────────────────┬─────────────────┤
│        │                          │                 │
│  Left  │        Centre            │     Right       │
│  Dock  │   (split-workbench)      │     Dock        │
│        │                          │                 │
│  Nav   │  List  │  Detail/Diagram │   Contextual    │
│        │        │                 │                 │
├────────┴────────┴─────────────────┴─────────────────┤
│                   Bottom Dock                       │
│              (Operations metrics)                   │
└─────────────────────────────────────────────────────┘
```

### dockWorkbench configuration

```typescript
const workbench = dockWorkbench({
  storageKey: 'aml-workbench',
  centre: hostPanel('aml-centre'),
  left: [
    { key: 'investigations', label: 'Investigations', icon: 'search',
      defaultOpen: true, content: hostPanel('aml-investigation-nav') },
    { key: 'worker-tasks', label: 'My Tasks', icon: 'assignment',
      content: hostPanel('aml-worker-nav') },
    { key: 'work-queue', label: 'Work Queue', icon: 'inbox',
      content: hostPanel('aml-work-queue-nav') },
  ],
  right: [
    { key: 'findings', label: 'Findings', icon: 'biotech',
      content: hostPanel('aml-findings-dock') },
    { key: 'compliance', label: 'Compliance', icon: 'verified',
      content: hostPanel('aml-compliance-dock') },
    { key: 'audit', label: 'Audit', icon: 'history',
      defaultOpen: true, content: hostPanel('aml-audit-dock') },
    { key: 'routing', label: 'Routing', icon: 'route',
      content: hostPanel('aml-routing-dock') },
  ],
  bottom: [
    { key: 'operations', label: 'Operations', icon: 'monitoring',
      content: hostPanel('aml-operations-dock') },
    { key: 'scenarios', label: 'Scenarios', icon: 'play_circle',
      content: hostPanel('aml-scenario-dock') },
  ],
});

loadSite(document.getElementById('app')!, workbench, { layoutStore });
```

### Left dock — Navigation panels

Three switchable navigation panels that feed the centre split-workbench:

| Panel | Content | Selection topic |
|-------|---------|-----------------|
| **Investigations** | `blocks-list-pane` with `/api/investigations` endpoint, investigation columns | `case` |
| **My Tasks** | `blocks-list-pane` filtered by capability tag via `/api/worker-tasks` | `worker-task` |
| **Work Queue** | `blocks-work-item-inbox` with `/api/work-items?scope=casehubio/aml/oversight` | `work-item` |

The left dock uses `exclusive: true` — only one nav panel is visible at a time.

### Centre — Topic-switching workbench

The centre panel is an AML-specific component (`aml-centre`) registered via `hostPanel`. It does **not** use a single `blocks-split-workbench` listening to multiple topics — `blocks-split-workbench` accepts only one `selection-topic` at a time.

Instead, `aml-centre` subscribes to dockWorkbench left-panel activation events and dynamically renders the appropriate content:

| Active left dock | Selection topic | Centre renders |
|-----------------|-----------------|----------------|
| **Investigations** | `case` | `blocks-split-workbench` with investigation list + detail tabs (Overview, Flow Diagram, Findings, Compliance, Audit, Routing) |
| **My Tasks** | `worker-task` | `blocks-worker-task-pane` with specialist workspace |
| **Work Queue** | `work-item` | `blocks-work-item-workbench` with work item detail |

This mirrors the existing `aml-app.ts` pattern (`_renderActiveView()` switching on `ViewId`) adapted for the dockWorkbench architecture. The centre component listens for `pages-dock-toggle` events — the real Pages dock activation event dispatched with `{ bubbles: true, composed: true }` from dock bar buttons (`pages-runtime/src/activation.ts` line 147). Because the centre panel is a sibling of the dock bar (not an ancestor), it listens on `document` and filters by left dock panel IDs:

```typescript
const LEFT_DOCK_PANELS = new Set(['aml-investigation-nav', 'aml-worker-nav', 'aml-work-queue-nav']);
const PANEL_TO_MODE: Record<string, string> = {
  'aml-investigation-nav': 'investigations',
  'aml-worker-nav': 'worker-tasks',
  'aml-work-queue-nav': 'work-queue',
};

@customElement('aml-centre')
class AmlCentre extends LitElement {
  @state() private _activeDock = 'investigations';

  private _onDockToggle = (e: Event) => {
    const { panelId, visible } = (e as CustomEvent<{ panelId: string; visible: boolean }>).detail;
    if (visible && LEFT_DOCK_PANELS.has(panelId)) {
      this._activeDock = PANEL_TO_MODE[panelId] ?? 'investigations';
    }
  };

  override connectedCallback() {
    super.connectedCallback();
    document.addEventListener('pages-dock-toggle', this._onDockToggle);
  }

  override disconnectedCallback() {
    super.disconnectedCallback();
    document.removeEventListener('pages-dock-toggle', this._onDockToggle);
  }

  override render() {
    switch (this._activeDock) {
      case 'investigations':
        return html`<blocks-split-workbench selection-topic="case" ...>...</blocks-split-workbench>`;
      case 'worker-tasks':
        return html`<blocks-worker-task-pane ...></blocks-worker-task-pane>`;
      case 'work-queue':
        return html`<blocks-work-item-workbench ...></blocks-work-item-workbench>`;
    }
  }
}
```

#### Cross-view drill-down (callerRef → caseId)

When a work-item is selected from the Work Queue, the workbench must extract the linked `caseId` from the WorkItem's `callerRef` and navigate to the investigation detail. Two callerRef formats exist:

- Compliance review WorkItems: `aml:investigation:{caseId}` (from `ComplianceReviewLifecycle.openReview()`)
- Gate approval WorkItems: `case:{caseId}/gate:{gateId}` (from `GateCallerRef.encode()`)

The centre panel's `work-item` selection handler parses callerRef trying both patterns (`case:(.+)/gate:.+` then `aml:investigation:(.+)`), extracts the caseId, and triggers investigation detail rendering. This is workbench-specific wiring, not a blocks-ui built-in.

#### URL navigation

The dockWorkbench migration replaces the current `#investigations`/`#compliance`/`#operations` hash routing with Pages navigation. Existing bookmark URLs will no longer work — Pages owns the URL model. Deep links to specific investigations are supported via `site.navigate()` with caseId parameters.

### Right dock — Contextual panels

Contextual information driven by the active investigation. All panels subscribe to a shared `investigation-context` selection topic (not the mode-specific `case`/`worker-task`/`work-item` topics) and update when the caseId changes:

| Panel | blocks-ui component | Data source |
|-------|-------------------|-------------|
| **Findings** | Specialist findings (AML-specific) | `/api/investigations/{caseId}/findings` |
| **Compliance** | `blocks-compliance-summary` | `/api/investigations/{caseId}/compliance-evidence` |
| **Audit** | `blocks-audit-trail-viewer` | `/api/investigations/{caseId}/audit-trail` |
| **Routing** | `blocks-trust-workbench` + `blocks-routing-rationale` | `/api/investigations/{caseId}/routing` + `/api/metrics/trust-scores` |

#### caseId derivation across modes

All three centre modes publish to `investigation-context` when they determine a caseId:

| Centre mode | caseId source | When published |
|-------------|---------------|----------------|
| **Investigations** | Directly from `case` selection topic | On row selection |
| **My Tasks** | `WorkerTaskResponse.caseId` | On task selection |
| **Work Queue** | Parsed from `callerRef` (via `case:(.+)/gate:.+` or `aml:investigation:(.+)`) | On work item selection |

The `aml-centre` component emits a `pages-selection` event on the `investigation-context` topic whenever any of its sub-views resolves a caseId. Right dock panels listen on `investigation-context` — they work regardless of which left dock panel is active.

### Bottom dock — Operations

System-wide metrics and scenario automation. Not tied to a specific case.

| Panel | Content |
|-------|---------|
| **Operations** | `blocks-kpi-metric-row` (throughput), `blocks-sla-indicator` (SLA health), `blocks-approval-gate` summary (gate activity), `aml-sar-quality-tab` (SAR outcome analysis), intervention metrics |
| **Scenarios** | `PagesLibraryView` for scenario selection + transport controls |

The Operations panel replaces the existing tabbed `aml-operations-view` (which has 5 tabs: Throughput, Trust Scores, Gates, Intervention, SAR Quality). In the dock layout, these become sections within the Operations panel rather than separate tabs — the dock panel scrolls vertically to show all sections.

---

## Component Map — What Goes Where

### blocks-ui components (used directly)

Components marked ✓ exist in `.casehub-packages` today. Components marked ✦ exist as packages but the spec references an incorrect or not-yet-registered element name.

| Component | Status | Where used | Configuration |
|-----------|--------|-----------|---------------|
| `blocks-split-workbench` | ✓ | Centre (investigations mode) | `selection-topic` set by active nav |
| `blocks-list-pane` | ✓ | Left dock (Investigations, Worker Tasks) | Column configs, endpoint, selection topic |
| `blocks-detail-pane` | ✓ | Centre right half | Tab definitions, selection-driven |
| `blocks-work-item-inbox` | ✓ | Left dock (Work Queue) | Endpoint, identity |
| `blocks-work-item-workbench` | ✓ | Centre (when work-item selected) | Full work item lifecycle |
| `blocks-audit-trail-viewer` | ✓ | Right dock (Audit) | Endpoint, causedByEntryId chain, Merkle verification |
| `blocks-trust-workbench` | ✓ | Right dock (Routing) | Trust scores, routing rationale |
| `blocks-routing-rationale` | ✓ | Right dock (Routing) | Per-decision routing explanation (used internally by trust-workbench) |
| `blocks-compliance-summary` | ✓ | Right dock (Compliance) | Regulation grid with status badges |
| `blocks-gdpr-erasure-action` | ✓ | Right dock (Compliance, GDPR sub-tab) | Three-phase erasure form |
| `blocks-sla-indicator` | ✓ | Bottom dock (Operations) | SLA countdown, breach policy |
| `blocks-approval-gate` | ✓ | Bottom dock (Operations) | Gate activity summary |
| `blocks-timeline` | ✓ | Centre (Overview tab) | Chronological event timeline |
| `casehub-diagram` | ✦ | Centre (Flow Diagram tab) | Investigation DAG with `graph-stencil-case` stencils and runtime overlay. Package exists as `casehub-diagram` (not `blocks-diagram-workbench` as in the original spec). |
| `blocks-channel-feed` | ✦ | Centre (Worker workspace) | Qhorus COMMAND/RESPONSE message feed. Package `channel-activity` provides `blocks-channel-feed`, `blocks-channel-message`, etc. |
| `blocks-kpi-metric-row` | ✦ | Bottom dock (Operations) | Package `kpi-metric-row` exists but needs `@customElement` registration — cross-repo work item |

### blocks-ui enhancements needed

| Component | Enhancement | Rationale |
|-----------|------------|-----------|
| `blocks-list-pane` | Push update support via EventConnection | Live investigation status changes without polling |
| `blocks-work-item-inbox` | Push update support via EventConnection | New work items appear without polling |
| `blocks-kpi-metric-row` | Push update support via EventConnection | Live metric updates |
| **New: `blocks-worker-task-pane`** | Generic worker task pane: task queue (filtered by capability), investigation context, specialist workspace slot | Reusable across CaseHub applications — any domain with specialist workers |

### AML-specific components (remain in this repo)

| Component | Purpose |
|-----------|---------|
| `aml-investigation-overview` | Transaction card, prior context, failure context — requires AML domain knowledge |
| `aml-specialist-workspace` | Per-specialist-type workspace content (entity resolution, pattern analysis, OSINT, SAR drafting, senior analyst) |
| `aml-sar-quality-tab` | SAR outcome analysis — AML-specific metric |
| `aml-scenario-actions` | `@ScenarioAction` backend methods for AML investigation lifecycle |
| AML scenario YAML scripts | Investigation walkthrough scenarios |

### Panel registration

All panels must be registered before `loadSite()`:

```typescript
import { registerPanel } from '@casehubio/pages-runtime';

// Register all AML panels
registerPanel('aml-centre', 'aml-centre');
registerPanel('aml-investigation-nav', 'aml-investigation-nav');
registerPanel('aml-worker-nav', 'aml-worker-nav');
registerPanel('aml-work-queue-nav', 'aml-work-queue-nav');
registerPanel('aml-findings-dock', 'aml-findings-dock');
registerPanel('aml-compliance-dock', 'aml-compliance-dock');
registerPanel('aml-audit-dock', 'aml-audit-dock');
registerPanel('aml-routing-dock', 'aml-routing-dock');
registerPanel('aml-operations-dock', 'aml-operations-dock');
registerPanel('aml-scenario-dock', 'aml-scenario-dock');
```

---

## Domain Worker Workbench

The specialist view — where entity resolution analysts, pattern analysts, OSINT screeners, SAR drafters, and senior analysts do investigation work. This is the "workbench" in AML Workbench.

### Architecture

A new `blocks-worker-task-pane` in blocks-ui provides the generic shell:

```
┌─────────────────────────────────────────┐
│  Task Queue (filtered by capability)    │
│  ┌─────────────────────────────────────┐│
│  │ INV-003 · PEP · entity-resolution  ││
│  │ INV-007 · STRUCTURING · pattern    ││
│  │ INV-012 · PEP · osint-screening    ││
│  └─────────────────────────────────────┘│
├─────────────────────────────────────────┤
│  Investigation Context                  │
│  Transaction | Prior Context | Findings │
├─────────────────────────────────────────┤
│  Specialist Workspace (slot)            │
│  [Domain-specific content goes here]    │
├─────────────────────────────────────────┤
│  Response Form                          │
│  [Submit RESPONSE/DONE/DECLINE]         │
└─────────────────────────────────────────┘
```

### Task queue

Data source: qhorus COMMAND messages filtered by the worker's capability tag.

**New API Required:** `GET /api/worker-tasks?capability={tag}` — returns pending COMMAND messages for a capability, with investigation context summary (transaction ID, flag reason, risk score, dispatch time). This is a thin adapter over qhorus message query + investigation summary join.

### Investigation context

When a worker selects a task, the context panel shows:
- Transaction details (from `InvestigationSummaryView`)
- Prior context (from `/api/investigations/{caseId}/prior-context`)
- What other specialists have already found (from `/api/investigations/{caseId}/findings`)
- The specific COMMAND parameters (from the qhorus message payload)

This reuses existing blocks-ui components (`blocks-detail-pane` with tabs) and existing AML endpoints.

### Specialist workspaces

Each specialist type has a workspace panel provided as slot content by the AML app:

| Specialist | Workspace content | Key UI elements |
|-----------|------------------|-----------------|
| **Entity Resolution** | Entity graph summary, ownership chain, risk scoring | Read-only data cards from findings endpoint |
| **Pattern Analysis** | Transaction pattern summary, related transactions | Pattern description, structuring detection result |
| **OSINT Screening** | Sanctions/PEP check results, adverse media | Hit/miss indicators, detail expandable |
| **SAR Drafting** | Narrative editor with auto-populated findings | Text area with pre-filled narrative from specialist findings |
| **Senior Analyst** | Consolidated findings with approve/escalate/decline | All specialist findings aggregated, decision buttons |

Each workspace is a Lit custom element registered in the AML app. The `blocks-worker-task-pane` renders the appropriate workspace based on the task's capability tag via a registry pattern:

```typescript
registerSpecialistWorkspace('entity-resolution', 'aml-entity-resolution-workspace');
registerSpecialistWorkspace('pattern-analysis', 'aml-pattern-analysis-workspace');
registerSpecialistWorkspace('osint-screening', 'aml-osint-workspace');
registerSpecialistWorkspace('sar-drafting', 'aml-sar-drafting-workspace');
registerSpecialistWorkspace('senior-analyst', 'aml-senior-analyst-workspace');
```

### `blocks-worker-task-pane` SPI contract

The registry is **component-scoped** (not window-global) — each `blocks-worker-task-pane` instance maintains its own workspace registry. This allows multiple CaseHub applications to register specialist workspaces independently.

```typescript
/**
 * Context provided to specialist workspace slot elements.
 * Set as a property on the workspace element when it is instantiated.
 */
interface WorkerTaskContext {
  taskId: string;
  capabilityTag: string;
  caseId: string;
  commandParams: Record<string, unknown>;
  investigationSummary: InvestigationSummaryResponse;
}

/**
 * Event emitted by workspace slot elements to submit structured results.
 * The generic response form listens for this to populate result fields.
 */
interface WorkspaceResultEvent extends CustomEvent {
  detail: {
    fields: Record<string, unknown>;   // specialist-specific structured result
    confidence: number;                 // 0.0 to 1.0
  };
}

/**
 * Properties set on the workspace slot element by blocks-worker-task-pane.
 */
interface SpecialistWorkspaceElement extends HTMLElement {
  taskContext: WorkerTaskContext;       // set by parent when task is selected
}
```

The `blocks-worker-task-pane` sets `taskContext` on the specialist element when a task is selected. The specialist element emits `workspace-result` events; the generic response form collects the `fields` and `confidence` from the event detail and includes them in the `POST /api/worker-tasks/{taskId}/respond` payload.

### Response form

Generic form provided by `blocks-worker-task-pane`:
- **RESPONSE/DONE** — structured result fields (specialist-specific, passed through from workspace), confidence score
- **DECLINE** — reason dropdown (out of clearance, insufficient data, conflict of interest), free-text detail

Submission maps to qhorus RESPONSE/DONE/DECLINE messages via `POST /api/worker-tasks/{taskId}/respond`.

---

## Investigation Flow Diagram

Replace the old spec's custom iframe graph with `casehub-diagram` (from the `casehub-diagram` package) using `graph-stencil-case` stencils for CasePlanModel visualization.

### Data source

`GET /api/investigations/{caseId}/flow` — already exists in `AmlInvestigationQueryResource.getInvestigationFlow()`. Returns the investigation path as a DAG with runtime state.

**Existing response schema** (from `InvestigationFlowResponse.java`):
```json
{
  "nodes": [
    {
      "capabilityTag": "entity-resolution",
      "workerId": "entity-resolution-agent",
      "trustScoreAtRouting": 0.85,
      "status": "completed",
      "timestamp": "2026-09-01T10:00:00Z"
    }
  ],
  "edges": [
    { "from": 0, "to": 1 },
    { "from": 0, "to": 2 }
  ],
  "parallelGroups": [[1, 2]]
}
```

Edges use **integer indices** into the nodes list (not string IDs). `parallelGroups` contains lists of node indices scheduled in parallel.

**Schema enhancements needed for v2 diagram:** The existing schema supports the basic DAG visualization. Two enhancements are needed for the v2 runtime overlay and adaptive decision display:

1. **`FlowNode` additions:** Add `startedAt` and `completedAt` timestamps (optional `Instant`) alongside the existing `timestamp` (dispatch time). The diagram overlay needs duration information for completed/in-progress display.
2. **`InvestigationFlowResponse` addition:** Add `adaptiveDecisions` list to show which adaptive bindings fired during the investigation. Each entry: `{ trigger: string, condition: string, fired: boolean, timestamp: Instant }`. This enables the "PEP detected → senior-analyst-required" highlights in the flow diagram.

These are additive changes to the existing Java records — no breaking changes to the existing frontend consumers.

### Rendering

The `casehub-diagram` renders this as a DAG using `graph-stencil-case` stencils:
- **Worker nodes** — show capability tag, selected worker ID, trust score badge, and runtime status (completed/in-progress/failed/declined)
- **Parallel groups** — rendered as parallel branches (pattern-analysis and osint-screening side by side)
- **Adaptive decisions** — highlighted nodes showing which binding fired and why (e.g., "PEP detected → senior-analyst-required")
- **Drill-down** — clicking a worker node can drill into the SWF internals if the worker is a Flow worker

### Runtime overlay

`CaseRuntimeState.toDecorations()` maps the flow response to `casehub-diagram` decorations:
- Green badge for completed
- Blue spinner for in-progress
- Red badge for failed
- Amber badge for declined
- Trust score shown as a pill on each worker node

---

## Scenario Automation

Replace custom simulation with Pages scenario orchestrator.

### Existing showcase system — migration path

The current webui has a `showcase/` directory with `mock-fetch.ts` (intercepting `/api/*` calls with mock data), `mock-data.ts`, and `element-guard.ts`. This provides a frontend-only demo mode without a running backend.

**Migration plan:**
1. **During implementation:** The showcase mock system coexists with scenario automation. Scenario automation requires a running backend; showcase mocks do not.
2. **After scenario automation is complete:** Remove `showcase/` directory. The scenario system fulfills the demo/sales showcase purpose with real backend execution, which is more convincing and eliminates mock/production divergence.
3. **Development without backend:** Developers can use Quarkus dev mode with `@QuarkusTestResource` for local backend execution. No frontend-only mock mode is needed — the platform is backend-driven.

### Backend SPI

AML registers `@ScenarioAction` methods for backend-driven scenario steps:

```java
@ApplicationScoped
public class AmlScenarioActions {

    @ScenarioAction("start-investigation")
    public Map<String, Object> startInvestigation(Map<String, Object> params) {
        // Create SuspiciousTransaction from params, start investigation
        // Return { caseId: "..." }
    }

    @ScenarioAction("approve-gate")
    public void approveGate(Map<String, Object> params) {
        // Find gate WorkItem by caseId + actionType, approve it
    }

    @ScenarioAction("seed-trust-scores")
    public void seedTrustScores(Map<String, Object> params) {
        // Seed trust attestations for deterministic routing
    }

    @ScenarioAction("wait-for-completion")
    public Map<String, Object> waitForCompletion(Map<String, Object> params) {
        // Poll investigation status until completed/failed
        // Return { status, outcomeType }
    }
}
```

### Scenario scripts

YAML scripts that interleave backend actions and UI walkthrough:

```yaml
name: pep-investigation
description: PEP investigation with senior analyst routing and SAR filing
labels: [demo, compliance, pep]
params:
  transactionAmount: 150000
  flagReason: HIGH_RISK_JURISDICTION

chapters:
  - title: Start Investigation
    sections:
      - title: Flag the transaction
        narrative: |
          A suspicious transaction has been flagged by the monitoring system.
          The transaction involves a Politically Exposed Person (PEP) with
          a high-risk jurisdiction flag.
        steps:
          - action: start-investigation
            delivery: graphql
            params:
              amount: ${params.transactionAmount}
              flagReason: ${params.flagReason}
              originAccount: ACC-PEP-001
              destinationAccount: ACC-SHELL-042
          - action: navigate
            delivery: aria
            target: { role: grid, name: Investigations }
          - action: spotlight
            delivery: aria
            target: { role: row, name: "${result.caseId}", index: 0 }
            message: New investigation created — click to view details

  - title: Investigation Flow
    sections:
      - title: Specialist routing
        narrative: |
          The engine evaluates the case bindings. Because the entity type
          is PEP, the `senior-analyst-required` binding fires — routing
          to the senior analyst instead of a junior.
        steps:
          - action: wait-for-completion
            delivery: graphql
            params:
              caseId: ${result.caseId}
          - action: click
            delivery: aria
            target: { role: row, name: "${result.caseId}" }
          - action: spotlight
            delivery: aria
            target: { role: tab, name: Flow Diagram }
            message: The investigation flow shows parallel specialist checks

  - title: Gate Approval
    sections:
      - title: SAR filing gate
        narrative: |
          The SAR filing action triggers the oversight gate. An MLRO
          must approve before the SAR is submitted to the regulator.
        steps:
          - action: click
            delivery: aria
            target: { role: button, name: Work Queue }
          - action: spotlight
            delivery: aria
            target: { role: row, name: "SAR Filing" }
            message: The MLRO sees the gate approval request in their inbox
          - action: approve-gate
            delivery: graphql
            params:
              caseId: ${result.caseId}
              actionType: sar.filing
```

### Scenario library

`PagesLibraryView` in the bottom dock (Scenarios panel) shows available scripts with:
- Script name, description, labels
- Readiness probe (checks if required ARIA targets exist)
- Run button, transport controls (play/pause/step/speed)
- Parameter editor for customizable scenarios

### Tutorial integration

Scenarios with `narrative` content become step-through tutorials:
- Markdown slides between automation steps
- Section-by-section execution with pause at section boundaries
- Progress tracking via outline tree
- Ideal for sales demos and onboarding

### Seed scenarios

| Scenario | Flag Reason | Key behaviour demonstrated |
|----------|------------|---------------------------|
| PEP investigation | HIGH_RISK_JURISDICTION | Senior-analyst routing, high trust scores, SAR filed, gate approval |
| Structuring ring | STRUCTURING | Pattern detection, parallel specialists, cleared (no SAR) |
| Clean transaction | LOW_RISK | Investigation completes with no SAR — false positive handling |
| Gate rejection | HIGH_RISK_JURISDICTION | MLRO rejects SAR filing — rejection routing flow |
| GDPR erasure | HIGH_RISK_JURISDICTION | Post-investigation erasure of actor PII and entity memories |
| Trust evolution | HIGH_RISK_JURISDICTION | Multiple investigations showing trust score changes over time |

---

## Push Updates

Wire Pages `EventConnection` for live updates. The Quarkus backend broadcasts state changes via SSE.

### Event sources

| Event | SSE endpoint | Consumers |
|-------|-------------|-----------|
| Investigation status change | `/api/events/investigations` | Investigation list, centre detail |
| New work item | `/api/events/work-items` | Work queue, left dock badge count |
| Trust score update | `/api/events/trust-scores` | Routing dock, operations trust panel |
| Gate decision | `/api/events/gates` | Operations gate activity |
| Worker task dispatch | `/api/events/worker-tasks` | Worker task queue |

### Backend implementation

Each SSE endpoint is a Quarkus `@RestStreamElement` (or `Multi<SseEvent>`) resource that broadcasts events via CDI `@ObservesAsync`:

```java
@Path("/api/events/investigations")
public class InvestigationEventResource {
    @GET
    @Produces(MediaType.SERVER_SENT_EVENTS)
    @RestStreamElement
    public Multi<InvestigationStatusEvent> stream() {
        return broadcastProcessor.stream();
    }
}
```

Investigation status events fire from the existing `CaseLifecycleEvent` observer. Work item events fire from `WorkItemLifecycleEvent`. Trust score events fire from `AmlTrustScoreSeeder` attestation writes.

### Frontend wiring

```typescript
import { EventConnection } from '@casehubio/pages-runtime';

const investigationEvents = new EventConnection('/api/events/investigations');
investigationEvents.onMessage((event) => {
  // Update investigation list, centre detail if matching caseId
});
```

### Event payload schemas

Each SSE event carries the complete set of mutable fields for the entity (not an incremental delta). Immutable fields established at creation (e.g., `transactionId`, `originAccount`, `amount`) are not repeated in events — only fields that change post-creation (`status`, `outcomeType`, `riskScore`) are included. The consumer merges the event's mutable fields into the entity it already holds from the initial list fetch.

```typescript
interface InvestigationStatusEvent {
  caseId: string;
  status: string;        // IN_PROGRESS | COMPLETED | FAILED | CANCELLED | SUSPENDED
  outcomeType: string | null;
  updatedAt: string;     // ISO 8601
}

interface WorkItemStatusEvent {
  workItemId: string;
  status: string;        // PENDING | ASSIGNED | IN_PROGRESS | COMPLETED | EXPIRED
  caseId: string | null; // from callerRef parsing, if applicable
  updatedAt: string;
}

interface TrustScoreUpdateEvent {
  agentId: string;
  capabilityTag: string;
  score: number;
  updatedAt: string;
}

interface GateDecisionEvent {
  workItemId: string;
  actionType: string;
  decision: string;      // APPROVED | REJECTED | EXPIRED
  decidedBy: string | null;
  decidedAt: string;
}

interface WorkerTaskEvent {
  taskId: string;
  capabilityTag: string;
  caseId: string;
  status: string;        // DISPATCHED | CLAIMED | COMPLETED | DECLINED
  updatedAt: string;
}
```

blocks-ui components that need enhancement for push support:
- `blocks-list-pane` — accept an `EventConnection` and merge incoming events into the displayed data
- `blocks-work-item-inbox` — accept an `EventConnection` for new work item notifications
- `blocks-kpi-metric-row` — accept an `EventConnection` for live metric updates

---

## Backend API Contracts

### Prerequisite code changes

All prerequisites are complete:

| Change | Where | Status |
|--------|-------|--------|
| Add `.scope("casehubio/aml/oversight")` to `ComplianceReviewLifecycle.openReview()` | AML app | **Done** — #88 closed, `.scope("casehubio/aml/oversight")` present at line 73 |

### Backend persistence

The `InvestigationSummaryView` CQRS pattern already exists (`io.casehub.aml.query`):
- `InvestigationSummaryView` — JPA entity with denormalised investigation fields
- `InvestigationSummaryObserver` — `CaseLifecycleEvent` observer that updates status; resolves `outcomeType` via `AmlInvestigationOutcomeService` on completion
- `InvestigationSummaryService` — query service with filter/sort/pagination
- `InvestigationSummaryRepository` — JPA repository

The `GET /api/investigations` list endpoint reads from this table.

**riskScore persistence gap:** The TypeScript `InvestigationSummaryResponse` includes `riskScore` and the investigation list renders a Risk column, but `InvestigationSummaryView` has no `riskScore` field and the Java `InvestigationSummaryResponse` record does not include it. To close this gap:

1. Add `riskScore` (Double, nullable) to `InvestigationSummaryView`
2. Add `updateRiskScore(UUID caseId, double riskScore)` to `InvestigationSummaryService`
3. Add `riskScore` to the Java `InvestigationSummaryResponse` record and `toResponse()` mapping

**Capture timing:** riskScore cannot be captured at summary creation time — `summaryService.createSummary()` runs synchronously in `AmlEngineCoordinator.startInvestigation()` (line 74) before the engine dispatches workers. The triage worker produces `TriageResult.riskScore()` asynchronously. The capture point is the triage worker's completion path: `InvestigationTriageWorker` already writes `riskScore` to the engine context (line 37). A new observer on triage worker completion (analogous to how `InvestigationSummaryObserver` captures `outcomeType` on case completion) calls `summaryService.updateRiskScore()` to persist the score. The summary row exists by this point because it was created before `startCase()` returned.

### Existing endpoints (no changes needed)

| Endpoint | Method | Purpose | Resource class |
|----------|--------|---------|----------------|
| `/api/investigations` | GET | List/search investigations with filters | `AmlInvestigationQueryResource.listInvestigations()` |
| `/api/investigations/{caseId}/flow` | GET | Investigation path topology for diagram | `AmlInvestigationQueryResource.getInvestigationFlow()` |
| `/api/investigations/{caseId}/findings` | GET | Structured specialist outcomes | `AmlInvestigationQueryResource.getFindings()` |
| `/api/investigations/{caseId}/gates` | GET | Gate decisions for a case | `AmlInvestigationQueryResource.getGates()` |
| `/api/investigations/{caseId}/prior-context` | GET | Prior entity context | `AmlInvestigationQueryResource.getPriorContext()` |
| `/api/investigations/{caseId}/compliance-evidence` | GET | Compliance evidence for a case | `AmlComplianceResource` |
| `/api/investigations/{caseId}/audit-trail` | GET | Audit trail entries for a case | `AmlAuditResource` |
| `/api/layer6/investigations/{caseId}` | GET | Single investigation detail (Layer 6) — includes routing decisions with current-time trust scores | `AmlLayer6Resource.getInvestigation()` |
| `/api/layer9/investigations/{caseId}` | GET | Single investigation detail (Layer 9) | `AmlLayer9Resource.getInvestigation()` |
| `/api/layer9/investigations/{caseId}/suspend` | POST | Suspend investigation | `AmlLayer9Resource.suspendInvestigation()` |
| `/api/layer9/investigations/{caseId}/resume` | POST | Resume investigation | `AmlLayer9Resource.resumeInvestigation()` |
| `/api/actors/{actorId}/erasure` | POST | GDPR actor erasure | `AmlGdprResource` |
| `/api/entities/{entityId}/erasure` | POST | GDPR entity erasure | `AmlGdprResource` |
| `/api/metrics/throughput` | GET | Throughput metrics | `AmlMetricsResource.getThroughputMetrics()` |
| `/api/metrics/trust-scores` | GET | Trust score metrics | `AmlMetricsResource.getTrustScoreMetrics()` |
| `/api/metrics/trust-scores/history` | GET | Historical trust score snapshots | `AmlMetricsResource.getTrustScoreHistory()` |
| `/api/metrics/gates` | GET | Gate activity metrics | `AmlMetricsResource.getGateMetrics()` |
| `/api/metrics/sar-quality` | GET | SAR quality report | `AmlMetricsResource.getSarQualityMetrics()` |

### New endpoints required

| Endpoint | Method | Purpose | Response type |
|----------|--------|---------|--------------|
| `/api/investigations/{caseId}/routing` | GET | Routing decisions with routing-time trust context (distinct from Layer 6 which returns current-time scores) — adds `alternativesConsidered` and `rationale` | `InvestigationRoutingResponse` |
| `/api/metrics/interventions` | GET | Intervention metrics — escalations, declines, gate rejections, manual overrides (frontend TypeScript type exists; backend implementation missing) | `InterventionMetrics` |
| `/api/worker-tasks` | GET | Pending worker tasks by capability | `PagedResponse<WorkerTaskResponse>` |
| `/api/worker-tasks/{taskId}/respond` | POST | Submit specialist response | `void` |
| `/api/events/investigations` | GET (SSE) | Investigation status change stream | SSE `InvestigationStatusEvent` |
| `/api/events/work-items` | GET (SSE) | Work item lifecycle stream | SSE `WorkItemStatusEvent` |
| `/api/events/trust-scores` | GET (SSE) | Trust score update stream | SSE `TrustScoreUpdateEvent` |
| `/api/events/gates` | GET (SSE) | Gate decision stream | SSE `GateDecisionEvent` |
| `/api/events/worker-tasks` | GET (SSE) | Worker task dispatch stream | SSE `WorkerTaskEvent` |
| `/api/scenario/library` | GET | Available scenario scripts | `ScriptDescriptor[]` |
| `/api/scenario/start` | POST | Start a scenario | `ScenarioState` |
| `/api/scenario/pause` | POST | Pause running scenario | `ScenarioState` |
| `/api/scenario/resume` | POST | Resume paused scenario | `ScenarioState` |

### New response types

```typescript
/**
 * Routing decisions with routing-time context.
 *
 * Distinct from Layer6InvestigationResponse.routingDecisions which returns
 * current-time trust scores from TrustScoreSource. This endpoint captures
 * the trust score at the moment of routing, plus alternatives and rationale
 * that are not available from the Layer 6 endpoint.
 *
 * Data source: AmlWorkerDecisionRepository entries joined with routing context.
 */
interface InvestigationRoutingResponse {
  decisions: {
    capabilityTag: string;
    selectedWorker: string;
    trustScoreAtRouting: number | null;
    alternativesConsidered: { workerId: string; score: number }[];
    rationale: string;
  }[];
}

interface WorkerTaskResponse {
  taskId: string;
  capabilityTag: string;
  caseId: string;
  dispatchedAt: string;
  commandParams: Record<string, unknown>;
  investigationSummary: InvestigationSummaryResponse;
}

/**
 * Intervention metrics — already defined in frontend types.ts but
 * backend implementation in AmlMetricsResource is missing.
 * Must be implemented to match existing frontend contract.
 */
interface InterventionMetrics {
  escalationCount: number;
  manualOverrideCount: number;
  declineRoutingCount: number;
  gateRejectionCount: number;
  averageResponseTimeSeconds: number;
  recentInterventions: RecentIntervention[];
}

interface RecentIntervention {
  type: string;   // ESCALATION | DECLINE_REROUTE | GATE_REJECTION | MANUAL_OVERRIDE
  caseId: string;
  reason: string;
  actor: string;
  occurredAt: string;
}
```

### Foundation endpoints (shared repos — all implemented)

All foundation endpoint issues are closed and implemented:

| Endpoint | Repo | Issue | Status |
|----------|------|-------|--------|
| `GET /api/work-items` (query) | casehub-work | work#241 | **Closed** — implemented |
| `POST /api/work-items/{id}/escalate` | casehub-work | work#284 | **Closed** — implemented |
| `POST /api/work-items/{id}/complete` | casehub-work | work#284 | **Closed** — implemented |
| `GET /api/ledger/entries?subjectId={id}` | casehub-ledger | ledger#162 | **Closed** — implemented |
| `GET /api/ledger/entries/{id}/proof` | casehub-ledger | ledger#162 | **Closed** — implemented |

---

## File Structure

```
app/src/main/webui/src/
├── index.ts                    # loadSite() + registerPanel() + imports
├── layout.ts                   # dockWorkbench configuration
├── types.ts                    # API response types (existing, extended)
├── panels/
│   ├── centre.ts               # Split-workbench centre panel
│   ├── investigation-nav.ts    # Left dock: investigation list
│   ├── worker-nav.ts           # Left dock: worker task list
│   ├── work-queue-nav.ts       # Left dock: work item inbox
│   ├── findings-dock.ts        # Right dock: specialist findings
│   ├── compliance-dock.ts      # Right dock: compliance-summary + gdpr
│   ├── audit-dock.ts           # Right dock: audit-trail-viewer
│   ├── routing-dock.ts         # Right dock: trust-workbench + rationale
│   ├── operations-dock.ts      # Bottom dock: KPIs, SLA, gates
│   └── scenario-dock.ts        # Bottom dock: PagesLibraryView
├── detail/
│   ├── investigation-detail.ts # Detail pane for selected investigation
│   ├── investigation-overview.ts  # Transaction card, prior context (keep existing)
│   ├── investigation-flow.ts   # diagram-workbench integration
│   └── sar-quality-tab.ts      # SAR outcome analysis (keep existing)
├── worker/
│   ├── entity-resolution.ts    # Specialist workspace
│   ├── pattern-analysis.ts     # Specialist workspace
│   ├── osint-screening.ts      # Specialist workspace
│   ├── sar-drafting.ts         # Specialist workspace
│   └── senior-analyst.ts       # Specialist workspace
├── events/
│   └── connections.ts          # EventConnection instances for push updates
├── strategies/
│   └── investigation-timeline.ts  # blocks-timeline strategy (existing)
└── scenarios/
    ├── pep-investigation.yaml
    ├── structuring-ring.yaml
    ├── clean-transaction.yaml
    ├── gate-rejection.yaml
    ├── gdpr-erasure.yaml
    └── trust-evolution.yaml
```

---

## Cross-Repo Work Required

| Repo | Work | Type | Status |
|------|------|------|--------|
| **blocks-ui** | New `blocks-worker-task-pane` component with SPI contract | New component | Needed |
| **blocks-ui** | Push update support on `list-pane`, `work-item-inbox` | Enhancement | Needed |
| **blocks-ui** | Register `@customElement` for `kpi-metric-row` package (package exists, element not registered) | Enhancement | Needed |
| **casehub-pages** | Verify `EventConnection` works with blocks-ui component refresh | Verification | Needed |
| ~~casehub-aml~~ | ~~`ComplianceReviewLifecycle` scope (#88)~~ | ~~Prerequisite~~ | **Done** |
| ~~casehub-work~~ | ~~`GET /api/work-items` query endpoint (work#241)~~ | ~~New endpoint~~ | **Done** |
| ~~casehub-work~~ | ~~WorkItem escalate + complete endpoints (work#284)~~ | ~~New endpoint~~ | **Done** |
| ~~casehub-ledger~~ | ~~Ledger entry query + proof endpoints (ledger#162)~~ | ~~New endpoint~~ | **Done** |

---

## Issue Triage

Based on this spec:

| Issue | Current state | Action | Reason |
|-------|--------------|--------|--------|
| **#101** (dual-mode datasets) | OPEN | **Close** | Scenario orchestrator replaces dual-mode; no custom simulation endpoints |
| **#89** (WebSocket/SSE) | OPEN | **Close** | Pages EventConnection covers push updates; spec includes SSE endpoints |
| **#110** (domain worker workbench) | OPEN | **Revise** | Rewrite as blocks-ui `worker-task-pane` + AML specialist workspaces |
| **#111** (epic: workbench UI) | OPEN | **Revise** | Update epic description to reflect composition-first approach |
| **#86** (auth/RBAC) | — | **Keep deferred** | Gate actions API-protected; UI role filtering is follow-up |
| **#10** (operational tooling) | — | **Keep** | MCP tools, OTel, PROV-DM — unaffected by UI |
| **#103–#109** (showcase UX children) | All CLOSED | **No action** | Already closed |

---

## Role Model (deferred — #86)

The workbench has a two-tier group model. Auth is deferred but the API layer enforces these groups now:

| Group | Role | Scope |
|-------|------|-------|
| `compliance-officers` | Compliance Officer (SAR review) | `ComplianceReviewLifecycle` WorkItems |
| `aml-compliance` | Compliance Officer (gates) | ACCOUNT_RESTRICTION, TRANSACTION_BLOCKING, ENTITY_LINK_CREATION gates |
| `aml-mlro` | MLRO | SAR_FILING gate (exclusive) |
| `aml-senior-compliance` | Senior Compliance Director | LAW_ENFORCEMENT_REFERRAL gate, GDPR erasure |

Non-gated consequential actions (suspend, resume, escalate) require `aml-compliance` or `aml-mlro` at the REST endpoint level. Gate actions are protected by WorkItem `candidateGroup` matching. UI role filtering via `withAccess()` is a follow-up (#86).

---

## Deferred Concerns

| Concern | Issue | Notes |
|---------|-------|-------|
| Authentication and role-based UI access | #86 | `withAccess()` available when needed; gate actions already API-protected |
| Mobile responsiveness | TBD | Desktop-first operational tool — file issue before implementation |
| Internationalisation | TBD | English-only for initial release — file issue before implementation |

Note: Trust score historical trend persistence (#87) was previously listed here but is now **implemented** — `AmlMetricsResource` has `GET /api/metrics/trust-scores/history` backed by `TrustScoreSnapshotService`. Issue #87 is closed.

---

## Requirements Checklist (vs. old spec)

Every requirement from the June 2026 spec mapped to this spec's coverage:

| Old spec section | Covered? | How |
|-----------------|----------|-----|
| §View 1: Work Queue | ✅ | Left dock Work Queue panel + blocks-work-item-inbox |
| §View 2: Case List | ✅ | Left dock Investigations panel + blocks-list-pane |
| §View 2: Case Detail (7 sections) | ✅ | Centre detail tabs + right dock panels |
| §View 2: Transaction | ✅ | aml-investigation-overview (kept) |
| §View 2: Prior Context | ✅ | aml-investigation-overview (kept) |
| §View 2: Investigation Flow | ✅ | casehub-diagram with runtime overlay |
| §View 2: Specialist Findings | ✅ | Right dock Findings panel |
| §View 2: Oversight Gates | ✅ | blocks-approval-gate in Operations dock |
| §View 2: Compliance Review | ✅ | Right dock Compliance panel + blocks-compliance-summary |
| §View 2: Failure Context | ✅ | aml-investigation-overview (kept) |
| §View 3: Audit Trail | ✅ | Right dock Audit panel + blocks-audit-trail-viewer |
| §View 3: Merkle Verification | ✅ | Built into blocks-audit-trail-viewer |
| §View 3: Compliance Evidence | ✅ | Right dock Compliance panel + blocks-compliance-summary |
| §View 3: GDPR Erasure | ✅ | blocks-gdpr-erasure-action in Compliance panel |
| §View 4: Throughput | ✅ | blocks-kpi-metric-row in Operations dock |
| §View 4: Trust Scores | ✅ | blocks-trust-workbench in Routing dock |
| §View 4: Gate Activity | ✅ | blocks-approval-gate in Operations dock |
| §View 4: SLA Health | ✅ | blocks-sla-indicator in Operations dock |
| §View 4: Intervention | ✅ | Operations dock + suspend/resume endpoints |
| §Simulation & Seed Data | ✅ | Pages scenario orchestrator + YAML scripts |
| §Parameterised Dataset URLs | ✅ | Resolved: blocks-ui endpoint attributes, not pages datasets (GE-20260810-cfc53d) |
| §Role Model | Deferred | #86 — gate actions already API-protected |
| §Component Ownership | ✅ | Component Map section above |
| §New API Endpoints | ✅ | Backend API Contracts section above |
| Domain worker workbench (#110) | ✅ | Domain Worker Workbench section above |

---

## References

- [GE-20260810-cfc53d] — datasets are static at loadSite(); use blocks-ui endpoint attributes
- [GE-20260804-befd45] — dockWorkbench decomposes into 3 primitives
- [GE-20260804-84ac70] — Pages DSL layout convention for CaseHub apps
- [GE-20260805-e3211c] — hostPanel requires registerPanel() before loadSite()
- [GE-20260804-24d409] — Lit @customElement tag mismatch silently degrades
- [GE-20260814-0d4123] — first tab "No data" workaround (navigate after loadSite)
- [GE-20260804-0959d2] — work-item-inbox has no compact mode; use list-pane for docks
- [GE-20260822-dd986e] — PagesElement render gate blocks standalone; use composition
- `app/src/main/webui/src/aml-app.ts` — existing implementation (split-workbench pattern)
- `app/src/main/webui/src/types.ts` — existing API response types
- `app/src/main/webui/src/panels/aml-investigation-overview.ts` — existing overview panel
- `app/src/main/webui/src/views/operations.ts` — existing operations view (to be replaced)
- Old spec: `specs/2026-06-30-aml-workbench-ui-design.md` — requirements source
- casehubio/aml#111 — parent epic
- casehubio/aml#110 — domain worker workbench
- casehubio/aml#101 — dual-mode datasets (close)
- casehubio/aml#89 — WebSocket/SSE (close)
- casehubio/work#241 — WorkItem query endpoint
- casehubio/work#284 — WorkItem escalate + complete endpoints
- casehubio/ledger#162 — Ledger entry query + proof endpoints
