# AML Workbench Frontend — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the AML workbench UI on casehub-pages — four views (Work Queue, Investigations, Accountability, Operations) served by Quinoa from the Quarkus backend.

**Architecture:** TypeScript-first casehub-pages DSL app. Single `loadSite()` entry point with sidebar navigation. Standard pages components for tables/charts/metrics, custom iframe component for investigation flow graph. Datasets bind directly to AML REST endpoints (same-origin, no CORS).

**Tech Stack:** TypeScript 5.x, casehub-pages (`@casehubio/pages-runtime`, `@casehubio/pages-ui`, `@casehubio/pages-data`), esbuild, Quinoa

**Spec:** `specs/2026-06-30-aml-workbench-ui-design.md`

**Prerequisite:** Plan 1 (backend API) must be complete — this plan consumes those endpoints.

## Global Constraints

- **TypeScript-first:** Use the casehub-pages DSL (`page()`, `sidebar()`, `table()`, `dataset()`, `lookup()`). YAML only for runtime-loaded pages.
- **No routing/state management:** casehub-pages owns navigation, filter state, sort state, pagination.
- **esbuild (not webpack):** Host apps use esbuild for fast builds. casehub-pages uses webpack internally.
- **Quinoa convention:** Frontend lives at `app/src/main/webui/`. See `../casehub/pages/docs/quinoa-convention.md`.
- **npm registry:** `@casehubio` packages are on GitHub Packages. `.npmrc` must point to the GitHub registry.
- **TDD:** Use `superpowers:test-driven-development` where applicable. Use `ts-dev` skill for all TypeScript.
- **Custom viz:** Investigation flow graph uses an iframe component via `@casehubio/pages-iframe-api`.

---

## File Structure

```
app/src/main/webui/
├── package.json
├── tsconfig.json
├── esbuild.config.mjs
├── .npmrc
└── src/
    ├── index.ts                    # App entry — loadSite() with sidebar
    ├── datasets.ts                 # All dataset definitions
    ├── views/
    │   ├── work-queue.ts           # Work queue view (metrics + table + actions)
    │   ├── investigations.ts       # Case list + case detail drill-down
    │   ├── accountability.ts       # Audit trail, Merkle, compliance, GDPR tabs
    │   └── operations.ts           # Throughput, trust, gates, SLA, intervention, simulation
    ├── components/
    │   └── investigation-flow/     # Custom iframe component
    │       ├── index.html
    │       ├── flow.ts
    │       └── flow.css
    └── types.ts                    # Shared response type definitions
```

---

### Task 12: Quinoa setup + app shell

**Files:**
- Create: all files in `app/src/main/webui/`
- Modify: `app/pom.xml` — add Quinoa extension dependency
- Modify: `app/src/main/resources/application.properties` — add Quinoa config

**Interfaces:**
- Consumes: Nothing
- Produces: A running web app at `http://localhost:8080/` with sidebar navigation showing four empty views

**Context for implementer:**
- Add Quinoa extension to `app/pom.xml`:
  ```xml
  <dependency>
      <groupId>io.quarkiverse.quinoa</groupId>
      <artifactId>quarkus-quinoa</artifactId>
  </dependency>
  ```
- Quinoa config in `application.properties`:
  ```properties
  quarkus.quinoa.build-dir=dist
  quarkus.quinoa.package-manager-install=true
  ```
- `.npmrc` must configure `@casehubio` scope to GitHub Packages:
  ```
  @casehubio:registry=https://npm.pkg.github.com
  //npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
  ```
- `package.json` deps: `@casehubio/pages-runtime`, `@casehubio/pages-ui`, `@casehubio/pages-data`
- `esbuild.config.mjs`: single entry point `src/index.ts`, output to `dist/`, bundle for browser
- `index.ts`:
  ```typescript
  import { loadSite } from "@casehubio/pages-runtime";
  import { page, sidebar } from "@casehubio/pages-ui";
  import { html } from "@casehubio/pages-ui";

  const app = page("AML Investigations",
    sidebar(
      ["Work Queue",      html("<p>Work Queue — coming soon</p>")],
      ["Investigations",  html("<p>Investigations — coming soon</p>")],
      ["Accountability",  html("<p>Accountability — coming soon</p>")],
      ["Operations",      html("<p>Operations — coming soon</p>")],
    ),
  );

  loadSite(document.getElementById("app")!, app);
  ```
- Verify with `mvn quarkus:dev` — the app should load at `http://localhost:8080/` showing the sidebar with four placeholder views.

- [ ] **Step 1: Add Quinoa dependency to pom.xml**
- [ ] **Step 2: Create webui directory structure with package.json, tsconfig, esbuild config, .npmrc**
- [ ] **Step 3: Write index.ts with sidebar shell**
- [ ] **Step 4: Create index.html** (minimal HTML with `<div id="app"></div>` and script tag)
- [ ] **Step 5: Add Quinoa config to application.properties**
- [ ] **Step 6: Run `mvn quarkus:dev` and verify the app loads in browser**
- [ ] **Step 7: Commit**

---

### Task 13: Datasets + Work Queue view

**Files:**
- Create: `app/src/main/webui/src/datasets.ts`
- Create: `app/src/main/webui/src/views/work-queue.ts`
- Modify: `app/src/main/webui/src/index.ts` — import work queue view

**Interfaces:**
- Consumes: `GET /api/work-items?status=OPEN&scope=casehubio/aml/oversight` (from casehub-work, may not exist yet — use mock data initially)
- Produces: `workQueueView()` function returning a casehub-pages Component

**Context for implementer:**
- The work queue dataset may not be available yet (depends on casehub-work#241). Design the view with the dataset definition pointing at the correct URL — when the endpoint exists, it will work. For development, use `inlineDataset()` with mock data.
- View layout (see spec §View 1):
  - 4x metric cards: total open, approaching SLA, overdue, by group
  - Filterable table with RowStyleRule for SLA status (red/amber/green)
  - Click-through handler parsing callerRef to extract caseId
- Use `lookup()` with `groupBy()` for the metric cards (count by status).
- Use `filterBy()` for the status/group filters.
- The click-through is application-level code: a custom click handler that parses `callerRef` (two formats: `aml:investigation:{caseId}` and `case:{caseId}/gate:{gateId}`), extracts caseId, and navigates via `LiveSite.navigate()`.

- [ ] Steps: Create datasets.ts → create work-queue.ts → wire into index.ts → verify in browser → commit

---

### Task 14: Investigations view (list + detail)

**Files:**
- Create: `app/src/main/webui/src/views/investigations.ts`
- Modify: `app/src/main/webui/src/index.ts`
- Modify: `app/src/main/webui/src/datasets.ts`

**Interfaces:**
- Consumes: `GET /api/investigations` (Task 3), `GET /api/investigations/{caseId}/prior-context` (Task 4), `GET /api/investigations/{caseId}/findings` (Task 6), `GET /api/investigations/{caseId}/gates` (Task 7), `GET /api/layer9/investigations/{caseId}` (existing)
- Produces: `investigationsView()` function with case list table + case detail drill-down

**Context for implementer:**
- Two levels: case list (table) and case detail (accordion/tabs with 7 sections).
- **Case list:** Status filter selector, sortable/paginated table, row styling by status.
- **Case detail drill-down:** Dynamic dataset creation on click — see spec §Parameterised Dataset URLs. When a row is clicked, construct datasets with the specific caseId and render the detail view.
- Detail sections: Transaction, Prior Context, Investigation Flow (Task 17 iframe component — placeholder for now), Specialist Findings (expandable panels), Oversight Gates (table), Compliance Review, Failure Context (conditional).
- For the investigation flow section, use `html("<p>Investigation flow visualization — coming in Task 17</p>")` as placeholder until the iframe component is built.

- [ ] Steps: Add datasets → create investigations.ts with list view → add detail drill-down → wire into index.ts → verify in browser → commit

---

### Task 15: Accountability view

**Files:**
- Create: `app/src/main/webui/src/views/accountability.ts`
- Modify: `app/src/main/webui/src/index.ts`
- Modify: `app/src/main/webui/src/datasets.ts`

**Interfaces:**
- Consumes: `GET /api/ledger/entries?subjectId={caseId}` (foundation — may not exist yet), `GET /api/ledger/entries/{id}/proof` (foundation — may not exist yet), `GET /api/investigations/{caseId}/compliance-evidence` (existing), `POST /api/actors/{actorId}/erasure` (existing), `POST /api/entities/{entityId}/erasure` (existing)
- Produces: `accountabilityView()` with 4 tabs: Audit Trail, Merkle Verification, Compliance Evidence, GDPR Erasure

**Context for implementer:**
- **Audit Trail:** Table of ledger entries for a case. Requires a case selector (shared with Investigations view). If the ledger query endpoint doesn't exist yet, use inline mock data.
- **Merkle Verification:** Entry selector + "Verify" button. Shows proof path (leaf hash → siblings → root). If the proof endpoint doesn't exist yet, placeholder.
- **Compliance Evidence:** Uses existing `GET /api/investigations/{caseId}/compliance-evidence`. Table with expandable detail per requirement.
- **GDPR Erasure:** Table of completed erasures + action forms for actor/entity erasure. Uses existing POST endpoints.

- [ ] Steps: Create accountability.ts with 4 tabs → add datasets → wire into index.ts → verify in browser → commit

---

### Task 16: Operations view + simulation panel

**Files:**
- Create: `app/src/main/webui/src/views/operations.ts`
- Modify: `app/src/main/webui/src/index.ts`
- Modify: `app/src/main/webui/src/datasets.ts`

**Interfaces:**
- Consumes: `GET /api/metrics/throughput` (Task 9), `GET /api/metrics/trust-scores` (Task 9), `GET /api/metrics/gates` (Task 9), `POST /api/simulation/seed` (Task 11), `POST /api/simulation/investigate` (Task 11)
- Produces: `operationsView()` with sections: Throughput, Trust Scores, Gate Activity, SLA Health, Intervention, Simulation

**Context for implementer:**
- **Throughput:** Timeseries chart (investigations over time), metric cards (in-flight, avg completion, rate), breakdown by flag reason (bar chart) and outcome (pie chart).
- **Trust Scores:** Table (agent × capability) + bar chart. Use `refreshTime: "1minute"`.
- **Gate Activity:** Approval rate bar chart, pending count metric, recent rejections table.
- **SLA Health:** Donut chart (healthy/approaching/overdue). Uses work queue dataset with different groupBy.
- **Intervention:** Suspend/resume buttons (POST calls), escalate, gate override. Each needs a case/workitem selector and confirmation.
- **Simulation:** Scenario dropdown + "Run" button + "Seed All" button + "Reset" button. Links to investigation view after run.

- [ ] Steps: Create operations.ts with all sections → add datasets → wire into index.ts → verify in browser → commit

---

### Task 17: Investigation flow iframe component

**Files:**
- Create: `app/src/main/webui/src/components/investigation-flow/index.html`
- Create: `app/src/main/webui/src/components/investigation-flow/flow.ts`
- Create: `app/src/main/webui/src/components/investigation-flow/flow.css`
- Modify: `app/src/main/webui/src/views/investigations.ts` — replace placeholder with iframePlugin
- Modify: esbuild config — add component bundle entry point
- Modify: package.json — add `@casehubio/pages-iframe-api` dependency

**Interfaces:**
- Consumes: `GET /api/investigations/{caseId}/flow` (Task 5) via `ComponentApi` data binding
- Produces: Custom iframe component rendering a directed graph of the investigation path

**Context for implementer:**
- The iframe component is isolated — it runs in its own iframe, communicates via `window.postMessage`.
- Use `ComponentApi` from `@casehubio/pages-iframe-api` to receive the flow data.
- Render a directed graph: nodes (specialist stages), edges (execution order), parallel groups.
- Each node shows: capability tag, worker ID, trust score, status (completed/declined/failed), colour-coded.
- Use a simple SVG layout (dagre.js optional for auto-layout, or a manual left-to-right flow). Start simple — a vertical list with parallel branches shown side-by-side. Upgrade to dagre later if needed.
- Register the component with `iframePlugin({ componentId: "investigation-flow", ... })` in the investigations view.
- The component bundle is a separate esbuild entry point output to a subdirectory.

- [ ] Steps: Create iframe component files → register in investigations view → add esbuild entry → verify in browser → commit

---

## Execution Notes

- **Task ordering:** Task 12 first (Quinoa setup). Tasks 13–16 can be parallel after 12 but are more natural sequential. Task 17 last (depends on investigations view from Task 14).
- **Foundation endpoint dependencies:** Tasks 13 (work queue) and 15 (accountability audit trail + Merkle) depend on foundation endpoints (casehub-work#241, casehub-ledger#162) that may not exist yet. Use inline mock data for development, with dataset URLs pointing at the correct endpoints — when the foundation ships, the views will work immediately.
- **Browser testing:** Every task must be verified in a browser via `mvn quarkus:dev`. Screenshots or visual confirmation that the view renders correctly.
- **Code review:** Run `superpowers:requesting-code-review` after each task's commit.
