# AML Workbench v2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #111 — epic: AML workbench UI — showcase UX gaps and domain worker workflow
**Issue group:** #111, #110, #101 (close), #89 (close)

**Goal:** Rewrite the AML workbench from a custom Lit app with sidebar navigation to a Pages dockWorkbench composition using blocks-ui components, with scenario automation, push updates, and a domain worker workbench.

**Architecture:** Pages `dockWorkbench` DSL provides the IDE-style layout (left nav, centre split-workbench, right contextual docks, bottom operations). The centre panel (`aml-centre`) switches content based on which left dock is active via `pages-dock-toggle` events. Right dock panels subscribe to a shared `investigation-context` topic for caseId-driven updates. Blocks-ui components replace hand-rolled operations, audit, compliance, and routing views. The Pages scenario orchestrator replaces the custom mock-fetch showcase.

**Tech Stack:** casehub-pages (dockWorkbench, scenario orchestrator, EventConnection), blocks-ui (split-workbench, list-pane, detail-pane, work-item-inbox, trust-workbench, audit-trail-viewer, compliance-summary, kpi-metric-row, sla-indicator, approval-gate, casehub-diagram), Lit, TypeScript, Quarkus (SSE, @ScenarioAction), vitest

## Global Constraints

- All custom elements use the `blocks-` or `aml-` prefix — unprefixed tags silently fail (GE-20260804-24d409)
- All panels must be registered via `registerPanel()` before `loadSite()` (GE-20260805-e3211c)
- Datasets are static at `loadSite()` — use blocks-ui endpoint attributes for dynamic data (GE-20260810-cfc53d)
- After `loadSite()`, call `site.navigate()` to the default view to work around first-tab "No data" (GE-20260814-0d4123)
- Use `ide-tooling` for all code navigation, editing, and refactoring — not bash grep/sed
- Java 21 source, Quarkus 3.32.2, esbuild for frontend builds
- Every commit references #111

---

## Batch 1: dockWorkbench shell + investigation navigation

**After this batch:** The dock layout renders with a working investigation list in the left dock. Selecting an investigation shows the existing detail tabs in the centre. The old `aml-app.ts` sidebar is replaced.

### Task 1: Layout configuration + panel registration + entry point rewrite

**Files:**
- Create: `app/src/main/webui/src/layout.ts`
- Create: `app/src/main/webui/src/panels/centre.ts`
- Create: `app/src/main/webui/src/panels/investigation-nav.ts`
- Modify: `app/src/main/webui/src/index.ts`
- Delete: `app/src/main/webui/src/aml-app.ts` (replaced by layout.ts + panels)

**Interfaces:**
- Produces: `dockWorkbench` config exported from `layout.ts`, `aml-centre` custom element, `aml-investigation-nav` custom element
- Consumes: `@casehubio/pages-runtime` (`dockWorkbench`, `hostPanel`, `loadSite`, `registerPanel`), `@casehubio/blocks-ui-split-workbench`, `@casehubio/blocks-ui-list-pane`, `@casehubio/blocks-ui-detail-pane`

- [ ] **Step 1: Write test for aml-centre topic switching**

```typescript
// app/src/main/webui/src/panels/centre.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import './centre.js';

describe('aml-centre', () => {
  let el: HTMLElement;

  beforeEach(() => {
    el = document.createElement('aml-centre');
    document.body.appendChild(el);
  });

  afterEach(() => el.remove());

  it('defaults to investigations mode', async () => {
    await el.updateComplete;
    const split = el.shadowRoot?.querySelector('blocks-split-workbench');
    expect(split).toBeTruthy();
    expect(split?.getAttribute('selection-topic')).toBe('case');
  });

  it('switches to work-queue mode on dock toggle event', async () => {
    await el.updateComplete;
    document.dispatchEvent(new CustomEvent('pages-dock-toggle', {
      detail: { panelId: 'aml-work-queue-nav', visible: true },
      bubbles: true, composed: true,
    }));
    await el.updateComplete;
    const workbench = el.shadowRoot?.querySelector('blocks-work-item-workbench');
    expect(workbench).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd app/src/main/webui vitest run src/panels/centre.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement layout.ts**

```typescript
// app/src/main/webui/src/layout.ts
import { dockWorkbench, hostPanel } from '@casehubio/pages-ui/dist/dsl/builders.js';

export const workbench = dockWorkbench({
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
```

- [ ] **Step 4: Implement centre.ts**

The `aml-centre` component from the spec — listens for `pages-dock-toggle` on `document`, switches between `blocks-split-workbench` (investigations), `blocks-worker-task-pane` (worker tasks), and `blocks-work-item-workbench` (work queue). Emits `pages-selection` on `investigation-context` topic when any mode resolves a caseId. See spec lines 118-157 for full implementation.

Move the `investigationTabs` definition from the existing `aml-app.ts` into `centre.ts`. Move the `_investigationColumns`, `_columnRenderers`, and `_getRowClass` into `investigation-nav.ts` (next step).

- [ ] **Step 5: Implement investigation-nav.ts**

```typescript
// app/src/main/webui/src/panels/investigation-nav.ts
import { LitElement, html } from 'lit';
import { customElement } from 'lit/decorators.js';
import '@casehubio/blocks-ui-list-pane';
// ... column config and renderers moved from aml-app.ts

@customElement('aml-investigation-nav')
export class AmlInvestigationNav extends LitElement {
  override render() {
    return html`
      <blocks-list-pane
        selection-topic="case"
        endpoint="/api/investigations"
        .columnConfig=${investigationColumns}
        .columnRenderers=${columnRenderers}
        .getRowKey=${(row) => row.text('caseId')}
        .getRowClass=${getRowClass}>
      </blocks-list-pane>
    `;
  }
}
```

- [ ] **Step 6: Rewrite index.ts**

```typescript
// app/src/main/webui/src/index.ts
import './panels/centre.js';
import './panels/investigation-nav.js';
// Panel imports for existing detail tabs
import './panels/aml-investigation-overview.js';
import './panels/aml-findings-panel.js';
import './panels/aml-routing-panel.js';
import './panels/aml-compliance-panel.js';
import './panels/aml-audit-trail.js';
import '@casehubio/blocks-ui-blocks-timeline';

import { registerPanel } from '@casehubio/pages-runtime';
import { loadSite } from '@casehubio/pages-runtime';
import { workbench } from './layout.js';

// Register panels BEFORE loadSite
registerPanel('aml-centre', 'aml-centre');
registerPanel('aml-investigation-nav', 'aml-investigation-nav');
// Stub registrations for panels not yet implemented
registerPanel('aml-worker-nav', 'aml-investigation-nav');      // temporary
registerPanel('aml-work-queue-nav', 'aml-investigation-nav');   // temporary
registerPanel('aml-findings-dock', 'aml-investigation-nav');    // temporary
registerPanel('aml-compliance-dock', 'aml-investigation-nav');  // temporary
registerPanel('aml-audit-dock', 'aml-investigation-nav');       // temporary
registerPanel('aml-routing-dock', 'aml-investigation-nav');     // temporary
registerPanel('aml-operations-dock', 'aml-investigation-nav');  // temporary
registerPanel('aml-scenario-dock', 'aml-investigation-nav');    // temporary

const container = document.getElementById('app')!;
const site = await loadSite(container, workbench);
site.navigate('Investigations'); // GE-20260814-0d4123 workaround
```

- [ ] **Step 7: Run tests**

Run: `yarn --cwd app/src/main/webui vitest run`
Expected: All tests pass

- [ ] **Step 8: Visual verification in browser**

Run: `yarn --cwd app/src/main/webui dev`
Verify: dockWorkbench renders with left dock showing investigation list, centre showing split workbench, dock bars visible on all edges. Clicking investigation row shows detail tabs in centre right half.

- [ ] **Step 9: Commit**

```bash
git add app/src/main/webui/src/
git commit -m "feat(#111): dockWorkbench layout shell + investigation navigation

Replace custom sidebar/hash routing with Pages dockWorkbench DSL.
Centre panel switches content based on left dock activation.
Investigation list and detail tabs work in the new layout.

Refs #111"
```

---

## Batch 2: Right dock panels + investigation-context wiring

**After this batch:** Selecting an investigation updates all right dock panels (audit trail, compliance, routing, findings). The `investigation-context` topic bridges all three centre modes to the right dock.

### Task 2: Right dock panels — audit, compliance, routing, findings

**Files:**
- Create: `app/src/main/webui/src/panels/audit-dock.ts`
- Create: `app/src/main/webui/src/panels/compliance-dock.ts`
- Create: `app/src/main/webui/src/panels/routing-dock.ts`
- Create: `app/src/main/webui/src/panels/findings-dock.ts`
- Modify: `app/src/main/webui/src/index.ts` (update registrations)
- Modify: `app/src/main/webui/src/panels/centre.ts` (emit investigation-context events)

**Interfaces:**
- Consumes: `investigation-context` selection topic (CustomEvent with `{ caseId: string }` detail)
- Produces: Right dock panels that self-fetch data when caseId changes
- Consumes: `@casehubio/blocks-ui-audit-trail-viewer`, `@casehubio/blocks-ui-compliance-summary`, `@casehubio/blocks-ui-trust-workbench`

- [ ] **Step 1: Write test for investigation-context event emission**

```typescript
// app/src/main/webui/src/panels/centre.test.ts (extend)
it('emits investigation-context when case is selected', async () => {
  await el.updateComplete;
  let receivedCaseId: string | null = null;
  document.addEventListener('pages-selection', (e: Event) => {
    const detail = (e as CustomEvent).detail;
    if (detail.topic === 'investigation-context') {
      receivedCaseId = detail.caseId;
    }
  });
  // Simulate case selection from split-workbench
  el.shadowRoot?.querySelector('blocks-split-workbench')
    ?.dispatchEvent(new CustomEvent('selection-changed', {
      detail: { row: { text: (col: string) => col === 'caseId' ? 'test-case-123' : '' } },
      bubbles: true, composed: true,
    }));
  expect(receivedCaseId).toBe('test-case-123');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd app/src/main/webui vitest run src/panels/centre.test.ts`
Expected: FAIL

- [ ] **Step 3: Update centre.ts to emit investigation-context events**

Add a selection listener on each sub-view that extracts caseId and emits a `pages-selection` CustomEvent with `{ topic: 'investigation-context', caseId }` on `document`. For the investigations mode, listen to `blocks-split-workbench` `selection-changed`. For work-queue mode, parse callerRef from the WorkItem (spec lines 160-167).

- [ ] **Step 4: Implement audit-dock.ts**

```typescript
// app/src/main/webui/src/panels/audit-dock.ts
import { LitElement, html, css } from 'lit';
import { customElement, state } from 'lit/decorators.js';
import '@casehubio/blocks-ui-audit-trail-viewer';

@customElement('aml-audit-dock')
export class AmlAuditDock extends LitElement {
  @state() private _caseId: string | null = null;

  private _onContext = (e: Event) => {
    const detail = (e as CustomEvent).detail;
    if (detail.topic === 'investigation-context') {
      this._caseId = detail.caseId;
    }
  };

  override connectedCallback() {
    super.connectedCallback();
    document.addEventListener('pages-selection', this._onContext);
  }

  override disconnectedCallback() {
    super.disconnectedCallback();
    document.removeEventListener('pages-selection', this._onContext);
  }

  static override styles = css`:host { display: block; height: 100%; }`;

  override render() {
    if (!this._caseId) return html`<div>Select an investigation</div>`;
    return html`
      <blocks-audit-trail-viewer
        endpoint="/api/investigations/${this._caseId}/audit-trail">
      </blocks-audit-trail-viewer>
    `;
  }
}
```

- [ ] **Step 5: Implement compliance-dock.ts, routing-dock.ts, findings-dock.ts**

Same pattern as audit-dock: listen for `investigation-context`, render the appropriate blocks-ui component with the caseId-parameterised endpoint.

- `compliance-dock.ts` → `blocks-compliance-summary` + `blocks-gdpr-erasure-action`
- `routing-dock.ts` → `blocks-trust-workbench` with routing endpoint
- `findings-dock.ts` → Keep existing `aml-findings-panel` (AML-specific), wrapped to receive caseId from `investigation-context`

- [ ] **Step 6: Update index.ts registrations**

Replace stub registrations with real panel registrations for all 4 right dock panels.

- [ ] **Step 7: Run tests + visual verification**

Run: `yarn --cwd app/src/main/webui vitest run`
Then: `yarn --cwd app/src/main/webui dev`
Verify: Selecting investigation updates all right dock panels.

- [ ] **Step 8: Commit**

```bash
git add app/src/main/webui/src/
git commit -m "feat(#111): right dock panels — audit, compliance, routing, findings

Wire investigation-context topic for caseId-driven updates.
Replace custom audit/compliance/routing panels with blocks-ui components.

Refs #111"
```

---

## Batch 3: Operations dock + work queue

**After this batch:** Operations metrics use blocks-ui components (replacing 950 lines of hand-rolled code). Work queue shows work-item-inbox. All three left dock panels have real content.

### Task 3: Operations dock — replace hand-rolled operations view

**Files:**
- Create: `app/src/main/webui/src/panels/operations-dock.ts`
- Modify: `app/src/main/webui/src/index.ts` (update registration)
- Delete: `app/src/main/webui/src/views/operations.ts` (replaced)
- Keep: `app/src/main/webui/src/views/aml-sar-quality-tab.ts` (AML-specific, moved into operations dock)

**Interfaces:**
- Consumes: `@casehubio/blocks-ui-kpi-metric-row`, `@casehubio/blocks-ui-sla-indicator`, `@casehubio/blocks-ui-approval-gate`
- Produces: `aml-operations-dock` custom element

- [ ] **Step 1: Write test for operations dock rendering**

Test that the component renders KPI, SLA, gate, intervention, and SAR quality sections.

- [ ] **Step 2: Implement operations-dock.ts**

Compose blocks-ui components for throughput (kpi-metric-row with `/api/metrics/throughput`), SLA health (sla-indicator), gate activity (approval-gate), and include the existing `aml-sar-quality-tab` as a section. For intervention metrics, keep a minimal custom section until the blocks-ui intervention component exists.

- [ ] **Step 3: Delete old operations view**

Use `ide_refactor_safe_delete` to remove `views/operations.ts`. Remove its import from `index.ts`.

- [ ] **Step 4: Run tests + verify**

- [ ] **Step 5: Commit**

### Task 4: Work queue nav + scenario dock stub

**Files:**
- Create: `app/src/main/webui/src/panels/work-queue-nav.ts`
- Create: `app/src/main/webui/src/panels/worker-nav.ts` (stub — blocks-ui dependency)
- Create: `app/src/main/webui/src/panels/scenario-dock.ts` (stub — PagesLibraryView)
- Modify: `app/src/main/webui/src/index.ts` (update registrations)

**Interfaces:**
- Consumes: `@casehubio/blocks-ui-work-item-inbox`, `@casehubio/blocks-ui-work-item-workbench`
- Produces: `aml-work-queue-nav`, `aml-worker-nav`, `aml-scenario-dock` custom elements

- [ ] **Step 1: Implement work-queue-nav.ts**

```typescript
@customElement('aml-work-queue-nav')
export class AmlWorkQueueNav extends LitElement {
  override render() {
    return html`
      <blocks-work-item-inbox
        endpoint="/api/work-items?scope=casehubio/aml/oversight"
        selection-topic="work-item"
        .identity=${{ userId: 'current-user', groups: ['compliance-officers'] }}>
      </blocks-work-item-inbox>
    `;
  }
}
```

- [ ] **Step 2: Create worker-nav.ts stub**

Placeholder: "Worker task queue — requires blocks-ui worker-task-pane (cross-repo dependency)". Uses `blocks-list-pane` with `/api/worker-tasks` endpoint as interim.

- [ ] **Step 3: Create scenario-dock.ts stub**

Placeholder: "Scenario library — requires Pages ScenarioOrchestrator integration". Shows a message indicating the scenario system is not yet wired.

- [ ] **Step 4: Update index.ts — replace all stub registrations**

- [ ] **Step 5: Run tests + visual verification**

- [ ] **Step 6: Commit**

```bash
git commit -m "feat(#111): operations dock + work queue + panel stubs

Replace 950-line hand-rolled operations view with blocks-ui components.
Wire work-item-inbox in left dock. Add stubs for worker-nav and scenarios.

Refs #111"
```

---

## Batch 4: Backend — riskScore persistence + new endpoints

**After this batch:** The riskScore gap is closed. The routing and intervention metrics endpoints exist. The flow response includes timing and adaptive decision data.

### Task 5: riskScore persistence in InvestigationSummaryView

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/query/InvestigationSummaryView.java`
- Modify: `app/src/main/java/io/casehub/aml/query/InvestigationSummaryService.java`
- Modify: `app/src/main/java/io/casehub/aml/query/InvestigationSummaryObserver.java` (or create new observer)
- Modify: Java `InvestigationSummaryResponse` record — add `riskScore`
- Create: Flyway migration for `risk_score` column
- Test: `app/src/test/java/io/casehub/aml/query/InvestigationSummaryRiskScoreTest.java`

**Interfaces:**
- Consumes: `InvestigationTriageWorker` completion event, `TriageResult.riskScore()`
- Produces: `riskScore` field on `InvestigationSummaryView` and `InvestigationSummaryResponse`

- [ ] **Step 1: Write failing test**

```java
@QuarkusTest
class InvestigationSummaryRiskScoreTest {
    @Inject InvestigationSummaryService summaryService;

    @Test
    void riskScore_populated_after_triage_completion() {
        // Start investigation → wait for triage → verify riskScore is set
        // riskScore should be non-null after triage completes
    }
}
```

- [ ] **Step 2: Add riskScore column — Flyway migration**

Determine next version number from existing migrations. Add `risk_score DOUBLE` column to `investigation_summary_view` table.

- [ ] **Step 3: Add riskScore to InvestigationSummaryView entity**

Use `ide_insert_member` to add `@Column(name = "risk_score") private Double riskScore;` plus getter.

- [ ] **Step 4: Add updateRiskScore to InvestigationSummaryService**

- [ ] **Step 5: Create observer for triage completion**

Observer watches for triage worker completion event, extracts `TriageResult.riskScore()` from case context, calls `summaryService.updateRiskScore()`.

- [ ] **Step 6: Add riskScore to Java InvestigationSummaryResponse**

- [ ] **Step 7: Run test to verify it passes**

Run: `mvn test -pl app -am -Dtest=InvestigationSummaryRiskScoreTest -Dsurefire.failIfNoSpecifiedTests=false`

- [ ] **Step 8: Commit**

### Task 6: Routing endpoint + intervention metrics + flow enhancements

**Files:**
- Create: `app/src/main/java/io/casehub/aml/rest/AmlInvestigationRoutingResource.java`
- Create: `app/src/main/java/io/casehub/aml/rest/dto/InvestigationRoutingResponse.java`
- Create: `app/src/main/java/io/casehub/aml/rest/AmlInterventionMetricsResource.java` (or add to AmlMetricsResource)
- Modify: `app/src/main/java/io/casehub/aml/rest/dto/FlowNode.java` — add `startedAt`, `completedAt`
- Modify: `app/src/main/java/io/casehub/aml/rest/dto/InvestigationFlowResponse.java` — add `adaptiveDecisions`
- Test: Corresponding `@QuarkusTest` classes

**Interfaces:**
- Produces: `GET /api/investigations/{caseId}/routing` → `InvestigationRoutingResponse`
- Produces: `GET /api/metrics/interventions` → `InterventionMetrics`
- Produces: Enhanced `InvestigationFlowResponse` with timing and adaptive decisions

- [ ] **Step 1: Write failing test for routing endpoint**

- [ ] **Step 2: Implement routing endpoint**

Query `AmlWorkerDecisionRepository` entries for the caseId, join with routing context for `alternativesConsidered` and `rationale`. Return `InvestigationRoutingResponse` with the trust score at routing time (distinct from Layer 6 current-time scores).

- [ ] **Step 3: Write failing test for intervention metrics**

- [ ] **Step 4: Implement intervention metrics endpoint**

The TypeScript `InterventionMetrics` type already defines the contract. Implement the backend: query escalation events, decline routing events, gate rejections, and manual overrides from the event log and work item history.

- [ ] **Step 5: Add startedAt/completedAt to FlowNode, adaptiveDecisions to InvestigationFlowResponse**

Additive changes to existing records. Populate from engine event log timestamps and binding evaluation records.

- [ ] **Step 6: Run all tests**

Run: `mvn test -pl app -am`

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(#111): routing endpoint, intervention metrics, flow enhancements

Add /api/investigations/{caseId}/routing with routing-time trust scores.
Add /api/metrics/interventions for escalation/decline/rejection counts.
Enhance flow response with timing and adaptive decision data.

Refs #111"
```

---

## Batch 5: Investigation flow diagram

**After this batch:** The Flow Diagram tab in the investigation detail shows the adaptive investigation path as a DAG with runtime state overlay (status badges, trust scores).

### Task 7: casehub-diagram integration for investigation flow

**Files:**
- Create: `app/src/main/webui/src/detail/investigation-flow.ts`
- Modify: `app/src/main/webui/src/panels/centre.ts` (add Flow Diagram tab to investigationTabs)

**Interfaces:**
- Consumes: `GET /api/investigations/{caseId}/flow` (existing endpoint, enhanced in Batch 4)
- Consumes: `@casehubio/casehub-diagram` package (graph-stencil-case stencils, CaseRuntimeState)
- Produces: `aml-investigation-flow` custom element

- [ ] **Step 1: Write test for flow component data fetch**

- [ ] **Step 2: Implement investigation-flow.ts**

Fetch `/api/investigations/{caseId}/flow` on caseId change. Transform `InvestigationFlowResponse` (nodes + edges + parallelGroups) into the `casehub-diagram` data format with `graph-stencil-case` Worker stencils. Map node status to `CaseRuntimeState.toDecorations()` for runtime overlay (green completed, blue in-progress, red failed, amber declined). Show trust score as a pill on each worker node.

- [ ] **Step 3: Add Flow Diagram tab to investigationTabs**

```typescript
{ id: 'flow', label: 'Flow Diagram', tagName: 'aml-investigation-flow', order: 5 },
```

- [ ] **Step 4: Run tests + visual verification**

Verify the diagram renders with correct stencils and runtime overlay for a completed investigation.

- [ ] **Step 5: Commit**

---

## Batch 6: Domain worker workbench

**After this batch:** The "My Tasks" left dock shows pending worker tasks. Selecting a task shows the specialist workspace with investigation context and a response form. All 5 specialist workspaces render appropriate content.

**Dependency:** `blocks-worker-task-pane` in blocks-ui. If not yet available, this batch uses `blocks-list-pane` + custom composition as interim.

### Task 8: Worker tasks backend endpoint

**Files:**
- Create: `app/src/main/java/io/casehub/aml/rest/AmlWorkerTaskResource.java`
- Create: `app/src/main/java/io/casehub/aml/rest/dto/WorkerTaskResponse.java`
- Test: `app/src/test/java/io/casehub/aml/rest/AmlWorkerTaskResourceTest.java`

**Interfaces:**
- Produces: `GET /api/worker-tasks?capability={tag}` → `PagedResponse<WorkerTaskResponse>`
- Produces: `POST /api/worker-tasks/{taskId}/respond` → void
- Consumes: qhorus message query API, `InvestigationSummaryService`

- [ ] **Step 1: Write failing test for worker tasks list**

- [ ] **Step 2: Implement WorkerTaskResponse record**

```java
public record WorkerTaskResponse(
    String taskId,
    String capabilityTag,
    String caseId,
    String dispatchedAt,
    Map<String, Object> commandParams,
    InvestigationSummaryResponse investigationSummary
) {}
```

- [ ] **Step 3: Implement GET /api/worker-tasks**

Query qhorus COMMAND messages filtered by capability tag. Join with `InvestigationSummaryService` to include the summary in each response.

- [ ] **Step 4: Write failing test for respond endpoint**

- [ ] **Step 5: Implement POST /api/worker-tasks/{taskId}/respond**

Accept structured result (RESPONSE/DONE) or decline reason (DECLINE). Map to qhorus RESPONSE/DONE/DECLINE messages.

- [ ] **Step 6: Run tests**

- [ ] **Step 7: Commit**

### Task 9: Worker nav panel + specialist workspaces

**Files:**
- Modify: `app/src/main/webui/src/panels/worker-nav.ts` (replace stub)
- Create: `app/src/main/webui/src/worker/entity-resolution.ts`
- Create: `app/src/main/webui/src/worker/pattern-analysis.ts`
- Create: `app/src/main/webui/src/worker/osint-screening.ts`
- Create: `app/src/main/webui/src/worker/sar-drafting.ts`
- Create: `app/src/main/webui/src/worker/senior-analyst.ts`
- Modify: `app/src/main/webui/src/index.ts` (register specialist workspaces)

**Interfaces:**
- Consumes: `GET /api/worker-tasks` (from Task 8)
- Consumes: `WorkerTaskContext` interface (spec lines 349-356)
- Produces: 5 specialist workspace custom elements, `aml-worker-nav` panel

- [ ] **Step 1: Write tests for specialist workspace rendering**

- [ ] **Step 2: Implement worker-nav.ts**

Replace stub with real `blocks-list-pane` (or `blocks-worker-task-pane` if available) with `/api/worker-tasks` endpoint and `worker-task` selection topic.

- [ ] **Step 3: Implement 5 specialist workspaces**

Each is a Lit component implementing `SpecialistWorkspaceElement` (accepts `taskContext` property). Content per spec lines 320-329:
- `entity-resolution.ts` — data cards showing entity graph, ownership chain, risk score
- `pattern-analysis.ts` — pattern description, structuring detection result
- `osint-screening.ts` — sanctions/PEP hit/miss indicators
- `sar-drafting.ts` — text area with pre-filled narrative from findings
- `senior-analyst.ts` — consolidated findings with decision buttons

- [ ] **Step 4: Register specialist workspaces in index.ts**

- [ ] **Step 5: Update aml-centre to render worker-task-pane for My Tasks mode**

- [ ] **Step 6: Run tests + visual verification**

- [ ] **Step 7: Commit**

```bash
git commit -m "feat(#111): domain worker workbench — task endpoint + specialist workspaces

Add worker-tasks REST endpoint (list + respond).
Implement 5 specialist workspace components.
Wire My Tasks left dock panel.

Refs #111, Refs #110"
```

---

## Batch 7: Push updates (SSE)

**After this batch:** Investigation status changes, new work items, trust score updates, gate decisions, and worker task dispatches push to the frontend in real-time via SSE.

### Task 10: SSE backend endpoints

**Files:**
- Create: `app/src/main/java/io/casehub/aml/events/InvestigationEventResource.java`
- Create: `app/src/main/java/io/casehub/aml/events/WorkItemEventResource.java`
- Create: `app/src/main/java/io/casehub/aml/events/TrustScoreEventResource.java`
- Create: `app/src/main/java/io/casehub/aml/events/GateEventResource.java`
- Create: `app/src/main/java/io/casehub/aml/events/WorkerTaskEventResource.java`
- Create: Event DTO records for each stream
- Test: SSE integration tests

- [ ] **Step 1: Write failing test for investigation SSE stream**

- [ ] **Step 2: Implement investigation event resource**

Quarkus `Multi<SseEvent>` backed by a `BroadcastProcessor`. Observer on `CaseLifecycleEvent` feeds events.

- [ ] **Step 3: Implement remaining 4 SSE resources**

Same pattern for work items (`WorkItemLifecycleEvent`), trust scores (attestation writes), gates (WorkItem completion for gate callerRefs), worker tasks (qhorus COMMAND dispatch).

- [ ] **Step 4: Run tests**

- [ ] **Step 5: Commit**

### Task 11: Frontend EventConnection wiring

**Files:**
- Create: `app/src/main/webui/src/events/connections.ts`
- Modify: panels that consume push updates (investigation-nav, work-queue-nav, operations-dock, routing-dock)

**Interfaces:**
- Consumes: `EventConnection` from `@casehubio/pages-runtime`
- Consumes: SSE endpoints from Task 10

- [ ] **Step 1: Create connections.ts**

```typescript
import { EventConnection } from '@casehubio/pages-runtime';

export const investigationEvents = new EventConnection('/api/events/investigations');
export const workItemEvents = new EventConnection('/api/events/work-items');
export const trustScoreEvents = new EventConnection('/api/events/trust-scores');
export const gateEvents = new EventConnection('/api/events/gates');
export const workerTaskEvents = new EventConnection('/api/events/worker-tasks');
```

- [ ] **Step 2: Wire into panels**

Each panel that consumes push updates imports the relevant EventConnection and subscribes via `onMessage()`. The handler merges the event's mutable fields into the component's current state.

Note: If blocks-ui components don't yet accept EventConnection natively (cross-repo enhancement needed), wrap them in the panel and manually trigger refresh on event receipt.

- [ ] **Step 3: Run tests + visual verification**

Start the backend, trigger an investigation, verify the investigation list updates in real-time without page refresh.

- [ ] **Step 4: Commit**

```bash
git commit -m "feat(#111): push updates — SSE endpoints + EventConnection wiring

5 SSE streams for investigations, work items, trust scores, gates, worker tasks.
Frontend panels subscribe via Pages EventConnection.

Refs #111, Closes #89"
```

---

## Batch 8: Scenario automation

**After this batch:** The scenario dock shows available scripts. Running a scenario drives both backend (start investigation, approve gate) and frontend (ARIA targeting for UI walkthrough). Tutorials provide narrated walkthroughs. The mock-fetch showcase is removed.

### Task 12: @ScenarioAction backend SPI + scenario scripts

**Files:**
- Create: `app/src/main/java/io/casehub/aml/scenario/AmlScenarioActions.java`
- Create: `app/src/main/webui/src/scenarios/pep-investigation.yaml`
- Create: `app/src/main/webui/src/scenarios/structuring-ring.yaml`
- Create: `app/src/main/webui/src/scenarios/clean-transaction.yaml`
- Create: `app/src/main/webui/src/scenarios/gate-rejection.yaml`
- Create: `app/src/main/webui/src/scenarios/gdpr-erasure.yaml`
- Create: `app/src/main/webui/src/scenarios/trust-evolution.yaml`
- Modify: `app/src/main/webui/src/panels/scenario-dock.ts` (replace stub)
- Delete: `app/src/main/webui/src/showcase/` (mock-fetch system replaced)
- Modify: `app/src/main/webui/showcase.html` (remove or redirect to scenario system)

**Interfaces:**
- Consumes: `@ScenarioAction` annotation from `casehub-pages` scenario-client
- Consumes: `PagesLibraryView` from `@casehubio/pages-runtime`
- Consumes: `ScenarioExecutorClient` for backend action registration
- Produces: 4 backend scenario actions + 6 YAML scenario scripts + working scenario dock

- [ ] **Step 1: Implement AmlScenarioActions.java**

```java
@ApplicationScoped
public class AmlScenarioActions {
    @Inject AmlEngineCoordinator coordinator;
    @Inject WorkItemStore workItemStore;
    @Inject AmlTrustScoreSeeder trustSeeder;

    @ScenarioAction("start-investigation")
    public Map<String, Object> startInvestigation(Map<String, Object> params) { ... }

    @ScenarioAction("approve-gate")
    public void approveGate(Map<String, Object> params) { ... }

    @ScenarioAction("seed-trust-scores")
    public void seedTrustScores(Map<String, Object> params) { ... }

    @ScenarioAction("wait-for-completion")
    public Map<String, Object> waitForCompletion(Map<String, Object> params) { ... }
}
```

- [ ] **Step 2: Write PEP investigation scenario YAML**

Full YAML from spec lines 494-564 — interleaves backend `graphql` actions with ARIA browser automation.

- [ ] **Step 3: Write remaining 5 scenario YAML files**

Per spec seed scenario table (line 585-592). Each demonstrates a distinct AML workflow path.

- [ ] **Step 4: Implement scenario-dock.ts**

Replace stub with `PagesLibraryView` for scenario browsing + transport controls.

- [ ] **Step 5: Delete showcase/ directory**

Use `ide_refactor_safe_delete` to remove `showcase/mock-fetch.ts`, `showcase/mock-data.ts`, `showcase/element-guard.ts`, `showcase/index.ts`. Update `index.html` to remove showcase entry point reference if needed.

- [ ] **Step 6: Run tests**

- [ ] **Step 7: Visual verification — run PEP scenario end to end**

Start Quarkus in dev mode. Open workbench. Click Scenarios dock → select PEP investigation → Run. Verify: investigation starts, specialist routing fires, investigation appears in list, flow diagram renders, gate approval completes, SAR filed.

- [ ] **Step 8: Commit**

```bash
git commit -m "feat(#111): scenario automation — @ScenarioAction SPI + 6 YAML scenarios

Replace mock-fetch showcase with Pages scenario orchestrator.
4 backend scenario actions: start-investigation, approve-gate,
seed-trust-scores, wait-for-completion.
6 scenario scripts covering PEP, structuring, clean, gate rejection,
GDPR erasure, and trust evolution.

Refs #111, Closes #101"
```

---

## Post-implementation

After all batches are complete:

1. **Close issues:** #101 (dual-mode datasets — scenario orchestrator replaces), #89 (WebSocket/SSE — EventConnection covers)
2. **Revise issues:** #110 (update description to reflect blocks-ui worker-task-pane approach), #111 (update epic description)
3. **File blocks-ui issues:** worker-task-pane component, push update support on list-pane/work-item-inbox, kpi-metric-row registration
4. **Update CLAUDE.md:** Remove any references to the old workbench spec or showcase system
5. **Update ARC42STORIES.MD:** Document the workbench architecture as a new layer entry

---

## References

- [2026-09-01-aml-workbench-v2-design.md] — design spec this plan implements
- [app/src/main/webui/src/aml-app.ts] — existing implementation being replaced
- [app/src/main/webui/src/views/operations.ts] — 950-line operations view being replaced
- [app/src/main/webui/src/types.ts] — existing API response types
- [app/src/main/webui/src/panels/aml-investigation-overview.ts] — kept (AML-specific)
- [app/src/main/webui/src/strategies/aml-investigation-timeline.ts] — kept (timeline strategy)
- [GE-20260810-cfc53d] — datasets static at loadSite; use endpoint attributes
- [GE-20260804-befd45] — dockWorkbench decomposition
- [GE-20260805-e3211c] — registerPanel before loadSite
- [GE-20260804-24d409] — blocks- prefix required
- [GE-20260814-0d4123] — first tab No data workaround
- [GE-20260804-0959d2] — work-item-inbox no compact mode
- [GitHub #111] — parent epic
- [GitHub #110] — domain worker workbench
- [GitHub #101] — dual-mode datasets (close)
- [GitHub #89] — WebSocket/SSE (close)
