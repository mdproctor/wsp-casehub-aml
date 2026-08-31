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

### Centre — Split workbench

`blocks-split-workbench` with `selection-topic` driven by the active left nav panel. The centre listens to all three selection topics and renders the appropriate detail view:

- **`case` selection** → investigation detail with tabs (Overview, Flow Diagram, Findings, Compliance, Audit, Routing)
- **`worker-task` selection** → specialist workspace (task context + response form)
- **`work-item` selection** → work item detail (claim, approve, reject, delegate)

The left half of the split-workbench mirrors the left dock's list as a compact inline list (for when the left dock is collapsed). The right half renders the detail.

### Right dock — Contextual panels

Contextual information driven by the centre's current selection. All panels update when the selected investigation changes:

| Panel | blocks-ui component | Data source |
|-------|-------------------|-------------|
| **Findings** | Specialist findings (AML-specific) | `/api/investigations/{caseId}/findings` |
| **Compliance** | `blocks-compliance-summary` | `/api/investigations/{caseId}/compliance-evidence` |
| **Audit** | `blocks-audit-trail-viewer` | `/api/investigations/{caseId}/audit-trail` |
| **Routing** | `blocks-trust-workbench` + `blocks-routing-rationale` | `/api/investigations/{caseId}/routing` + `/api/metrics/trust-scores` |

### Bottom dock — Operations

System-wide metrics and scenario automation. Not tied to a specific case.

| Panel | Content |
|-------|---------|
| **Operations** | `blocks-kpi-metric-row` (throughput), `blocks-sla-indicator` (SLA health), `blocks-approval-gate` summary (gate activity) |
| **Scenarios** | `PagesLibraryView` for scenario selection + transport controls |

---

## Component Map — What Goes Where

### blocks-ui components (used directly)

| Component | Where used | Configuration |
|-----------|-----------|---------------|
| `blocks-split-workbench` | Centre | `selection-topic` switches by active nav |
| `blocks-list-pane` | Left dock (Investigations, Worker Tasks) | Column configs, endpoint, selection topic |
| `blocks-detail-pane` | Centre right half | Tab definitions, selection-driven |
| `blocks-work-item-inbox` | Left dock (Work Queue) | Endpoint, identity |
| `blocks-work-item-workbench` | Centre (when work-item selected) | Full work item lifecycle |
| `blocks-audit-trail-viewer` | Right dock (Audit) | Endpoint, causedByEntryId chain, Merkle verification |
| `blocks-trust-workbench` | Right dock (Routing) | Trust scores, routing rationale |
| `blocks-routing-rationale` | Right dock (Routing) | Per-decision routing explanation |
| `blocks-compliance-summary` | Right dock (Compliance) | Regulation grid with status badges |
| `blocks-gdpr-erasure-action` | Right dock (Compliance, GDPR sub-tab) | Three-phase erasure form |
| `blocks-kpi-metric-row` | Bottom dock (Operations) | Throughput, completion rate, in-flight |
| `blocks-sla-indicator` | Bottom dock (Operations) | SLA countdown, breach policy |
| `blocks-approval-gate` | Bottom dock (Operations) | Gate activity summary |
| `blocks-diagram-workbench` | Centre (Flow Diagram tab) | Investigation DAG with runtime overlay |
| `blocks-timeline` | Centre (Overview tab) | Investigation timeline with amlInvestigationTimelineStrategy |
| `blocks-channel-activity` | Centre (Worker workspace) | Qhorus COMMAND/RESPONSE message feed |

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

### Response form

Generic form provided by `blocks-worker-task-pane`:
- **RESPONSE/DONE** — structured result fields (specialist-specific, passed through from workspace), confidence score
- **DECLINE** — reason dropdown (out of clearance, insufficient data, conflict of interest), free-text detail

Submission maps to qhorus RESPONSE/DONE/DECLINE messages via `POST /api/worker-tasks/{taskId}/respond`.

---

## Investigation Flow Diagram

Replace the old spec's custom iframe graph with `blocks-diagram-workbench`.

### Data source

**New API Required:** `GET /api/investigations/{caseId}/flow` — returns the investigation path as a CasePlanModel topology with runtime state.

**Response schema:**
```json
{
  "caseDefinitionId": "aml-investigation",
  "nodes": [
    {
      "id": "entity-resolution",
      "type": "worker",
      "capabilityTag": "entity-resolution",
      "selectedWorker": "entity-resolution-agent",
      "trustScoreAtRouting": 0.85,
      "status": "completed",
      "startedAt": "2026-09-01T10:00:00Z",
      "completedAt": "2026-09-01T10:02:00Z"
    }
  ],
  "edges": [
    { "from": "entity-resolution", "to": "pattern-analysis" },
    { "from": "entity-resolution", "to": "osint-screening" }
  ],
  "parallelGroups": [["pattern-analysis", "osint-screening"]],
  "adaptiveDecisions": [
    {
      "trigger": "senior-analyst-required",
      "condition": "entityType == PEP || riskScore > 0.8",
      "fired": true,
      "timestamp": "2026-09-01T10:02:05Z"
    }
  ]
}
```

### Rendering

The `blocks-diagram-workbench` renders this as a DAG using `graph-stencil-case` stencils:
- **Worker nodes** — show capability tag, selected worker ID, trust score badge, and runtime status (completed/in-progress/failed/declined)
- **Parallel groups** — rendered as parallel branches (pattern-analysis and osint-screening side by side)
- **Adaptive decisions** — highlighted nodes showing which binding fired and why (e.g., "PEP detected → senior-analyst-required")
- **Drill-down** — clicking a worker node can drill into the SWF internals if the worker is a Flow worker

### Runtime overlay

`CaseRuntimeState.toDecorations()` maps the flow response to `blocks-diagram-workbench` decorations:
- Green badge for completed
- Blue spinner for in-progress
- Red badge for failed
- Amber badge for declined
- Trust score shown as a pill on each worker node

---

## Scenario Automation

Replace custom simulation with Pages scenario orchestrator.

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

blocks-ui components that need enhancement for push support:
- `blocks-list-pane` — accept an `EventConnection` and merge incoming events into the displayed data
- `blocks-work-item-inbox` — accept an `EventConnection` for new work item notifications
- `blocks-kpi-metric-row` — accept an `EventConnection` for live metric updates

---

## Backend API Contracts

### Existing endpoints (no changes needed)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/layer6/investigations/{caseId}` | GET | Single investigation detail (Layer 6) |
| `/api/layer9/investigations/{caseId}` | GET | Single investigation detail (Layer 9) |
| `/api/investigations/{caseId}/compliance-evidence` | GET | Compliance evidence for a case |
| `/api/investigations/{caseId}/audit-trail` | GET | Audit trail entries for a case |
| `/api/investigations/{caseId}/prior-context` | GET | Prior entity context |
| `/api/actors/{actorId}/erasure` | POST | GDPR actor erasure |
| `/api/entities/{entityId}/erasure` | POST | GDPR entity erasure |
| `/api/metrics/throughput` | GET | Throughput metrics |
| `/api/metrics/trust-scores` | GET | Trust score metrics |
| `/api/metrics/gates` | GET | Gate activity metrics |
| `/api/metrics/interventions` | GET | Intervention metrics |

### New endpoints required

| Endpoint | Method | Purpose | Response type |
|----------|--------|---------|--------------|
| `/api/investigations` | GET | List/search investigations with filters | `PagedResponse<InvestigationSummaryResponse>` |
| `/api/investigations/{caseId}/flow` | GET | Investigation path topology for diagram | `InvestigationFlowResponse` |
| `/api/investigations/{caseId}/findings` | GET | Structured specialist outcomes | `InvestigationFindingsResponse` |
| `/api/investigations/{caseId}/gates` | GET | Gate decisions for a case | `InvestigationGatesResponse` |
| `/api/investigations/{caseId}/routing` | GET | Routing decisions with trust context | `InvestigationRoutingResponse` |
| `/api/worker-tasks` | GET | Pending worker tasks by capability | `PagedResponse<WorkerTaskResponse>` |
| `/api/worker-tasks/{taskId}/respond` | POST | Submit specialist response | `void` |
| `/api/layer9/investigations/{caseId}/suspend` | POST | Suspend investigation | `void` |
| `/api/layer9/investigations/{caseId}/resume` | POST | Resume investigation | `void` |
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
  transactionId: string;
  flagReason: string;
  riskScore: number;
  dispatchedAt: string;
  commandParams: Record<string, unknown>;
}
```

### Foundation endpoints (file issues in shared repos)

| Endpoint | Repo | Issue | Status |
|----------|------|-------|--------|
| `GET /api/work-items` (query) | casehub-work | work#241 | Existing issue |
| `POST /api/work-items/{id}/escalate` | casehub-work | work#284 | Existing issue |
| `POST /api/work-items/{id}/complete` | casehub-work | work#284 | Existing issue |
| `GET /api/ledger/entries?subjectId={id}` | casehub-ledger | ledger#162 | Existing issue |
| `GET /api/ledger/entries/{id}/proof` | casehub-ledger | ledger#162 | Existing issue |

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

| Repo | Work | Type |
|------|------|------|
| **blocks-ui** | New `blocks-worker-task-pane` component | New component |
| **blocks-ui** | Push update support on `list-pane`, `work-item-inbox`, `kpi-metric-row` | Enhancement |
| **blocks-ui** | Specialist workspace registry in `worker-task-pane` | New SPI |
| **casehub-pages** | Verify `EventConnection` works with blocks-ui component refresh | Verification |
| **casehub-work** | `GET /api/work-items` query endpoint (work#241) | New endpoint |
| **casehub-work** | WorkItem escalate + complete endpoints (work#284) | New endpoint |
| **casehub-ledger** | Ledger entry query + proof endpoints (ledger#162) | New endpoint |

---

## Issue Triage

Based on this spec:

| Issue | Action | Reason |
|-------|--------|--------|
| **#101** (dual-mode datasets) | **Close** | Scenario orchestrator replaces dual-mode; no custom simulation endpoints |
| **#89** (WebSocket/SSE) | **Close** | Pages EventConnection covers push updates; spec includes SSE endpoints |
| **#110** (domain worker workbench) | **Revise** | Rewrite as blocks-ui `worker-task-pane` + AML specialist workspaces |
| **#111** (epic: workbench UI) | **Revise** | Update epic description to reflect composition-first approach |
| **#86** (auth/RBAC) | **Keep deferred** | Gate actions API-protected; UI role filtering is follow-up |
| **#10** (operational tooling) | **Keep** | MCP tools, OTel, PROV-DM — unaffected by UI |
| **#103–#109** (showcase UX children) | **Close** | Subsumed by this spec's comprehensive redesign |

---

## Deferred Concerns

| Concern | Issue | Notes |
|---------|-------|-------|
| Authentication and role-based UI access | #86 | `withAccess()` available when needed; gate actions already API-protected |
| Trust score historical trend persistence | #87 | Current scores only; trend timeseries needs snapshot persistence |
| Mobile responsiveness | — | Desktop-first operational tool |
| Internationalisation | — | English-only for initial release |

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
| §View 2: Investigation Flow | ✅ | blocks-diagram-workbench with runtime overlay |
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
