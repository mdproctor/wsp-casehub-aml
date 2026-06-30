# AML Workbench UI — Design Spec

**Date:** 2026-06-30
**Status:** Draft
**Issue:** To be filed in casehub-aml
**Scope:** Production AML investigation workbench built on casehub-pages

---

## Purpose

A standalone web application for operating AML investigations end-to-end. Serves three roles simultaneously:

1. **Operational tool** — compliance officers, MLROs, and ops staff use it daily
2. **Demo vehicle** — step-through walkthroughs with pre-seeded data and live simulation
3. **Sales showcase** — the app itself demonstrates CaseHub's accountability model

The app lives in the casehub-aml repo (`app/src/main/webui/`), served by Quinoa from the Quarkus backend (`aml-app` is the Quarkus module; `aml-api` is pure Java). It consumes the AML REST API directly — dataset URLs resolve relative to the Quarkus server.

---

## Tech Stack

- **casehub-pages** — TypeScript-first DSL for composition, data binding, layout
- **Quinoa** — Quarkus extension serving the frontend from `app/src/main/webui/`
- **esbuild** — fast rebuild for dev mode (~50ms)
- **npm deps:** `@casehubio/pages-runtime`, `@casehubio/pages-ui`, `@casehubio/pages-data`

No routing framework, no state management library. casehub-pages owns navigation (sidebar), filter state, sort state, and pagination. Custom visualisations use iframe components via `@casehubio/pages-iframe-api`.

---

## App Structure

Single `loadSite()` call with sidebar navigation across four views:

```typescript
const app = page("AML Investigations",
  sidebar(
    ["Work Queue",      workQueueView()],
    ["Investigations",  investigationsView()],
    ["Accountability",  accountabilityView()],
    ["Operations",      operationsView()],
  ),
);

loadSite(document.getElementById("app")!, app);
```

The sidebar gives the demo walkthrough a natural story: what needs action → how investigations work → proof of accountability → system health.

---

## Component Ownership

Principle: if the component only needs foundation data (work items, ledger entries, trust scores), it belongs in a shared package. If it needs AML domain knowledge, it belongs here. Build here first, extract when the shape stabilises.

| Component | Data Source | Ownership | Extract Target |
|-----------|-----------|-----------|----------------|
| Work queue | casehub-work WorkItems | Shared | `@casehubio/pages-work` |
| Audit trail viewer | casehub-ledger entries | Shared | `@casehubio/pages-ledger` |
| Merkle proof viewer | casehub-ledger verification | Shared | `@casehubio/pages-ledger` |
| Trust score dashboard | casehub-ledger trust scores | Shared | `@casehubio/pages-ledger` |
| Compliance evidence viewer | Requirement model + ledger | Shared viewer, AML requirements | `@casehubio/pages-compliance` |
| Oversight gate panel | Foundation gate model | Shared base + app config | `@casehubio/pages-work` |
| Investigation flow viz | AML specialist outcomes | AML-specific | stays here |
| SAR review detail | AML investigation summary | AML-specific | stays here |
| Specialist findings panels | AML domain results | AML-specific | stays here |
| Simulation controls | AML scenario templates | AML-specific | stays here |

---

## View 1: Work Queue

The compliance officer / MLRO inbox. What needs human action right now.

### Dataset

```typescript
dataset("work-queue", {
  url: "/api/work-items",
  query: { status: "OPEN", category: "aml-compliance" },
  refreshTime: "30second",
  columns: [
    { id: "id", type: "LABEL" },
    { id: "title", type: "LABEL" },
    { id: "priority", type: "LABEL" },
    { id: "candidateGroups", type: "LABEL" },
    { id: "claimDeadline", type: "DATE" },
    { id: "createdAt", type: "DATE" },
    { id: "status", type: "LABEL" },
    { id: "callerRef", type: "LABEL" },
  ],
});
```

### Layout

**Summary metrics row** (4x metric cards):
- Total open items
- Approaching SLA (< 7 days to `claimDeadline`)
- Overdue (past `claimDeadline`)
- By group breakdown (compliance-officers / aml-mlro / aml-senior-compliance)

**Filterable table:**
- Sortable by priority, deadline, creation date
- Row styling via `RowStyleRule`: red for overdue, amber for < 7 days, green for healthy
- Filterable by candidate group and priority

**Click-through:**
- Clicking a row navigates to the Investigations view with the linked caseId extracted from `callerRef` (which encodes `aml:investigation:{caseId}`). Implementation: custom JavaScript click handler parses the callerRef string, extracts the caseId via regex (`/aml:investigation:(.+)/`), and triggers sidebar navigation to the Investigations view with the caseId as a URL parameter. This is application-level code in the workbench, not a casehub-pages built-in — pages provides tree navigation and the `ComponentApi` event system, but cross-view drill-down with parameter extraction is workbench-specific logic.

**Actions:**
- Claim, approve, reject buttons — REST POST via form save adapter
- Rejection requires a reason (textarea input)

### New API Required

`GET /api/work-items` — query endpoint for WorkItems filtered by status, category, candidate group. Currently AML creates WorkItems via casehub-work but does not expose a query surface. This is a foundation concern — the endpoint belongs in casehub-work or an AML adapter.

### Shared Component Notes

Everything in this view is generic. The only AML-specific behaviour is the click-through target (Investigations view). Extract candidate: a `WorkQueuePage` component that accepts a dataset URL, filter config, and drill-down target.

---

## View 2: Investigations

Two levels: case list and case detail drill-down.

### Case List

**Dataset:**

```typescript
dataset("investigations", {
  url: "/api/layer9/investigations",
  refreshTime: "60second",
  columns: [
    { id: "caseId", type: "LABEL" },
    { id: "status", type: "LABEL" },
    { id: "outcomeType", type: "LABEL" },
    { id: "transactionId", type: "LABEL" },
    { id: "originAccount", type: "LABEL" },
    { id: "destinationAccount", type: "LABEL" },
    { id: "amount", type: "NUMBER" },
    { id: "currency", type: "LABEL" },
    { id: "flagReason", type: "LABEL" },
    { id: "createdAt", type: "DATE" },
  ],
});
```

**Layout:**
- Status filter selector (in-progress / completed / failed / cancelled / suspended / all)
- Sortable, paginated table
- Row styling: green completed, blue in-progress, red failed, amber suspended
- Click-through to case detail

**New API Required:**

`GET /api/layer9/investigations` (list endpoint) — the current API only supports `GET /api/layer9/investigations/{caseId}` for a single case. Need a list/search endpoint returning summary rows. Query params: status filter, date range, account ID, pagination.

**Backend design:** No investigation list or summary table exists today. `Layer9InvestigationResponse` only carries `caseId, status, outcome, failureContext` — no transaction details. The transaction metadata (`originAccountId`, `destinationAccountId`, `amount`, `currency`, `flagReason`) is captured in `AmlCaseOpenedLedgerEntry` (which has `originAccountId`/`destinationAccountId`) and originates from `SuspiciousTransaction`.

**Approach:** Build a denormalised `InvestigationSummaryView` populated on case open. When `AmlEngineCoordinator.startInvestigation()` fires, a new `AmlInvestigationSummaryService` persists a summary row (caseId, transaction fields, status=IN_PROGRESS, createdAt) to a dedicated table. Status updates via a `CaseLifecycleEvent` observer. The list endpoint reads this table with filter/sort/pagination — no cross-join between engine state and ledger required.

**outcomeType population:** `CaseLifecycleEvent` carries `caseStatus` but NOT outcome information. When the observer receives a `COMPLETED` status transition, it calls `AmlInvestigationOutcomeService.resolveOutcome(caseId)` to resolve the `InvestigationOutcome` from `AmlSarOfficerReviewedLedgerEntry` records, then writes `outcome.type()` to the summary row's `outcomeType` column. For non-COMPLETED terminal states (FAILED, CANCELLED), `outcomeType` remains null — these cases have no outcome, only a `FailureContext`.

This is the correct approach because it avoids runtime joins across persistence units (engine on default PU, ledger on qhorus PU), keeps the read path fast, and follows the CQRS-lite pattern already established in the architecture.

**Response schema:**
```json
{
  "items": [
    {
      "caseId": "uuid",
      "status": "IN_PROGRESS|COMPLETED|FAILED|CANCELLED|SUSPENDED",
      "outcomeType": "sar-filed|gate-rejected|null",
      "transactionId": "TXN-2024-001",
      "originAccount": "ACC-001",
      "destinationAccount": "ACC-002",
      "amount": 50000.00,
      "currency": "USD",
      "flagReason": "PEP",
      "createdAt": "2026-06-30T10:00:00Z"
    }
  ],
  "total": 42,
  "page": 0,
  "pageSize": 25
}
```

Note: `outcomeType` is flattened from `InvestigationOutcome.type()` — the nested `outcome.type` field is projected to a top-level string for flat dataset consumption. Null when the investigation has no outcome yet.

### Case Detail

Drill-down for a single investigation. Uses cross-filtering from the case list or direct URL navigation.

**Datasets:**

```typescript
dataset("case-detail", "/api/layer9/investigations/{caseId}");
dataset("case-compliance", "/api/investigations/{caseId}/compliance-evidence");
dataset("case-ledger", "/api/ledger/entries?subjectId={caseId}");
```

**Layout — 7 sections via accordion or tabs:**

#### 1. Transaction
Panel showing the `SuspiciousTransaction` that triggered the investigation. Fields: ID, origin account, destination account, amount, currency, timestamp, flag reason. Standard metric/table display.

#### 2. Prior Context
What the system knew about these entities before the investigation started. Shows AmlPriorContext data: entity risk history, network relationships, pattern history. Highlights `isKnownHighRisk` if applicable — this explains why senior-analyst routing was triggered.

**New API Required:** `GET /api/investigations/{caseId}/prior-context` — expose the AmlPriorContext that was used at investigation start.

**Backend design:** `AmlEngineCoordinator.startInvestigation()` already injects the prior context into the engine's `CaseContext` under key `"priorEntityContext"` via `AmlPriorContext.toContextMap()`. Retrieval: `CaseHubRuntime.query(caseId, "priorEntityContext")` returns the structured map containing `hasHistory` (boolean), `knownHighRisk` (boolean), `entityRiskCount`, `networkCount`, `patternCount`, and `facts[]` (each with `domain`, `text`, `createdAt`, `confidence`). The endpoint returns this map directly — no reconstruction needed since `toContextMap()` already serialises to a well-structured JSON-compatible shape.

**Response schema:**
```json
{
  "hasHistory": true,
  "knownHighRisk": true,
  "entityRiskCount": 3,
  "networkCount": 1,
  "patternCount": 2,
  "facts": [
    { "domain": "ENTITY_RISK", "text": "Prior SAR filed — entity linked to PEP network", "createdAt": "2026-05-15T10:00:00Z", "confidence": "0.9" }
  ]
}
```

#### 3. Investigation Flow (custom iframe component)
Visual directed graph of the adaptive investigation path. Nodes represent specialist stages (entity-resolution → pattern-analysis + osint-screening → sar-drafting → compliance-review). Each node shows:
- Specialist capability tag
- Selected worker ID
- Trust score at routing time
- SpecialistOutcome (Completed / Declined / Failed)
- Result summary on hover/click

Parallel stages shown as parallel branches. Adaptive routing decisions highlighted (e.g., "PEP detected → senior-analyst-required fired"). This is the key differentiator visualisation — no competitor shows the decision chain this way.

**Implementation:** Custom iframe component registered with casehub-pages. Uses a graph rendering library (e.g., dagre, elkjs, or a simple custom SVG layout). Receives data via `ComponentApi` from the case detail dataset.

**New API Required:** `GET /api/investigations/{caseId}/flow` — the investigation path with routing decisions, specialist outcomes, and trust scores in a structure suitable for graph rendering. The Layer 6 response has `routingDecisions` but not the full flow topology.

**Backend design:** This data must be reconstructed from the engine's `EventLog`. `CaseHubRuntime.eventLog(caseId)` returns `List<CaseEventLogRecord>`. `EventLog` has a monotonic `seq` field (Long) providing causal ordering independent of wall-clock timestamps. Reconstruct the DAG:
1. Query `EventLog` entries for the caseId via `CaseHubRuntime.eventLog(caseId, Set.of(WORKER_SCHEDULED, WORKER_EXECUTION_COMPLETED, WORKER_EXECUTION_FAILED))`
2. Build a node per `WORKER_SCHEDULED` event, ordered by `seq`
3. **Parallel detection via `seq` ordering:** two `WORKER_SCHEDULED` events dispatched in the same binding evaluation cycle will have consecutive `seq` values with no intervening `WORKER_EXECUTION_COMPLETED` event between them. Group consecutive `WORKER_SCHEDULED` events (in `seq` order) that are not separated by a `WORKER_EXECUTION_COMPLETED` — these are parallel. This uses the engine's causal ordering, not wall-clock timestamp proximity.
4. Join with `AmlTrustRoutingAttestation` for trust scores at routing time (already persisted per capability)
5. Specialist outcomes come from `CaseContext` — query via `CaseHubRuntime.query(caseId, path)` for each specialist key
6. Return a graph structure: `{ nodes: [...], edges: [...] }` where edges represent execution order and parallel groups are marked

#### 4. Specialist Findings
Expandable panels for each specialist result:
- **Entity Resolution** — entity ID, ownership chain, entity type, risk score
- **Pattern Analysis** — structuring detected (boolean), description
- **OSINT Screening** — sanctions hit, PEP hit, detail
- **SAR Narrative** — the synthesised SAR text

Each panel shows the `SpecialistOutcome` variant — Completed with full results, or Declined/Failed with agent ID, capability, and reason.

**New API Required:** `GET /api/investigations/{caseId}/findings` — structured specialist outcomes. Currently embedded in `InvestigationSummary` but only available through the Layer 1 synchronous endpoint, not the Layer 9 async path.

**Backend design:** Specialist results are stored in `CaseContext` as individual entries, written by worker functions in `AmlInvestigationCaseDescriptor`. Each worker writes its result map under the capability key. Retrieval uses multiple `CaseHubRuntime.query()` calls:
1. `query(caseId, "entityResolution")` → `EntityResolutionResult` fields: `entityId`, `ownershipChain`, `entityType`, `riskScore`
2. `query(caseId, "patternAnalysis")` → `PatternAnalysisResult` fields: `structuringDetected`, `description`
3. `query(caseId, "osintScreening")` → `OsintResult` fields: `sanctionsHit`, `pepHit`, `detail` (plus `declined`, `reason` if the worker declined)
4. `query(caseId, "sarDraft")` → SAR narrative text

Each result is wrapped in the `SpecialistOutcome<T>` sealed hierarchy — `Completed(result)`, `Declined(agentId, capability, reason)`, or `Failed(agentId, capability, reason)`. The endpoint assembles all four into a single response. A null query result means the specialist hasn't executed yet (investigation still in progress).

**Response schema:**
```json
{
  "entityResolution": { "status": "COMPLETED", "result": { "entityId": "...", "ownershipChain": "...", "entityType": "PEP", "riskScore": 0.87 } },
  "patternAnalysis": { "status": "COMPLETED", "result": { "structuringDetected": false, "description": "..." } },
  "osintScreening": { "status": "DECLINED", "agentId": "osint-screening-agent", "capability": "osint-screening", "reason": "insufficient clearance" },
  "sarDraft": { "status": "COMPLETED", "result": { "narrative": "..." } }
}
```

#### 5. Oversight Gates
Table of gated actions for this investigation:
- Action type (SAR_FILING, ACCOUNT_RESTRICTION, etc.)
- Risk classification and gate policy that triggered
- Approval status (pending / approved / rejected)
- Approver identity and timestamp
- Candidate group

**New API Required:** `GET /api/investigations/{caseId}/gates` — gate decisions for a case. Currently internal to the oversight coordinator.

#### 6. Compliance Review
The WorkItem for this investigation:
- Status (open / claimed / completed / rejected)
- Assigned officer
- SLA deadline and countdown
- Decision and rejection reason (if applicable)

Data comes from the work queue dataset filtered by `callerRef`.

#### 7. Failure Context
Only rendered when `status` is FAILED, CANCELLED, or SUSPENDED. Shows:
- Trigger goal name and kind
- Failure events: event type, worker ID, timestamp, detail
- Timeline of failure progression

Data comes directly from the `failureContext` field in the Layer 9 response.

---

## View 3: Accountability

Three sub-views via tabs.

### Audit Trail
Ledger entry chain for a selected case in causal order. Table columns: entry type, actor, role, timestamp, `causedByEntryId`, digest. The `causedByEntryId` links are the differentiator — click any entry to see what caused it, trace backwards to the original CASE_OPENED event.

**Dataset:**

```typescript
dataset("audit-trail", {
  url: "/api/ledger/entries",
  query: { subjectId: "{caseId}" },
  columns: [
    { id: "id", type: "LABEL" },
    { id: "entryType", type: "LABEL" },
    { id: "actorId", type: "LABEL" },
    { id: "actorRole", type: "LABEL" },
    { id: "occurredAt", type: "DATE" },
    { id: "causedByEntryId", type: "LABEL" },
    { id: "digest", type: "LABEL" },
  ],
});
```

**New API Required:** `GET /api/ledger/entries?subjectId={id}` — query ledger entries by subject. This is a foundation concern (casehub-ledger).

**Shared Component:** Yes — any ledger consumer needs an audit trail table. Extract candidate.

### Merkle Verification
For any ledger entry, show the inclusion proof: leaf hash, sibling path to root, tree root. "Verify" button calls the verification endpoint and shows pass/fail with visual proof path.

**New API Required:** Already exists — `LedgerVerificationService.inclusionProof()`. Needs a REST endpoint: `GET /api/ledger/entries/{entryId}/proof`.

**Shared Component:** Yes — extract to `@casehubio/pages-ledger`.

### Compliance Evidence
FinCEN/FATF requirements mapped to evidence artifacts. Table: requirement ID, citation, mechanism, status (CLOSED / PARTIAL / BREACHED / GAP). Expandable detail per requirement showing linked ledger entries, WorkItems, and attestations.

**Dataset:** Uses existing `GET /api/investigations/{caseId}/compliance-evidence`.

**Shared Component:** The viewer component (table + expandable detail) is generic. The requirement definitions are AML-specific. Extract the viewer; keep the requirement config here.

### GDPR Erasure
Erasure status for actors and entities. Table showing completed erasures: actor/entity ID, erasure reason, memories erased, receipt entry ID, timestamp. Action buttons:
- "Erase Actor" — `POST /api/actors/{actorId}/erasure`
- "Erase Entity" — `POST /api/entities/{entityId}/erasure`

Both endpoints already exist.

---

## View 4: Operations

System-wide monitoring and intervention.

### Throughput Metrics

**Dataset:**

```typescript
dataset("throughput", {
  url: "/api/metrics/throughput",
  refreshTime: "30second",
});
```

- Timeseries chart: investigations started, completed, failed over time
- Metric cards: current in-flight, average completion time, completion rate
- Breakdown by flag reason (bar chart) and outcome type (pie chart)

**New API Required:** `GET /api/metrics/throughput` — aggregate investigation metrics with time bucketing.

**Response schema:** `{ started: number, completed: number, failed: number, avgCompletionTimeMs: number, completionRate: number, byFlagReason: { [reason]: number }, byOutcome: { [type]: number }, timeBuckets: [{ bucket: "2026-06-30T00:00Z", started: number, completed: number, failed: number }] }`. Time range and granularity via query params: `?since=2026-06-01&granularity=day`. Built from the `InvestigationSummaryView` table (same denormalised table used by the list endpoint).

### Trust Scores

**Dataset:**

```typescript
dataset("trust-scores", {
  url: "/api/metrics/trust-scores",
  refreshTime: "1minute",
});
```

- Table: agent ID, capability dimensions, current scores, threshold
- Bar chart: scores by capability across agents
- Trend timeseries: score evolution over time
- Highlight agents below threshold

**New API Required:** `GET /api/metrics/trust-scores` — current trust scores for all agents across all capability dimensions.

**Response schema:** `{ agents: [{ agentId: string, capabilities: [{ tag: string, score: number, threshold: number, observations: number }] }] }`. Current scores from `TrustScoreCache.getCapabilityScore()`. Historical trend data is NOT currently persisted — `TrustScoreCache` only holds current scores. Trend timeseries requires a new persistence layer (score snapshots on `TrustScoreJob` completion). This is deferred to a follow-up issue.

### Gate Activity

- Approval rate by action type (bar chart)
- Pending gates count (metric card)
- Average approval time (metric card)
- Recent rejections table with reasons

**New API Required:** `GET /api/metrics/gates` — aggregate gate activity metrics.

**Response schema:** `{ approvalRate: number, pending: number, avgApprovalTimeMs: number, byActionType: [{ actionType: string, total: number, approved: number, rejected: number }], recentRejections: [{ workItemId: string, actionType: string, reason: string, rejectedAt: string }] }`. Built from WorkItem queries filtered by gate callerRef pattern (`case:*/gate:*`). Requires the `GET /api/work-items` query endpoint in casehub-work.

### SLA Health

- Work items by SLA status: healthy / approaching / overdue (pie or donut chart)
- Timeline: items created vs completed per day

Uses the work queue dataset with different groupBy operations.

### Intervention

Operational controls — actions that modify system state:
- **Suspend investigation** — `POST /api/investigations/{caseId}/suspend` delegates to `CaseHubRuntime.suspendCase(caseId)`. The engine handles in-flight worker cancellation and durable suspension (survives JVM restart via `PendingWorkRegistry`). Resume via `POST /api/investigations/{caseId}/resume` delegates to `CaseHubRuntime.resumeCase(caseId)`.
- **Escalate work item** — `POST /api/work-items/{id}/escalate`
- **Override gate** — gates are WorkItems created by `ActionGateWorkItemHandler` with a `callerRef` encoding `case:{caseId}/gate:{gateId}` (via `GateCallerRef.encode()`). Override is WorkItem completion with an override payload: `POST /api/work-items/{id}/complete` with `{ "decision": "OVERRIDE", "reason": "..." }`. The `WorkItemLifecycleAdapter` routes the approval back to the engine to resume the gated case. No standalone gate entity or gate REST resource exists.

Each action requires confirmation (modal or inline confirm) and writes to the audit trail.

**New API Required:** Suspend/resume endpoints are new thin REST layers over `CaseHubRuntime.suspendCase()`/`resumeCase()`. Escalate endpoint is new in casehub-work. Gate override uses the existing WorkItem completion lifecycle — no new gate-specific endpoint needed.

---

## Simulation & Seed Data

### Seed Scenarios

Pre-built investigation datasets covering the key narratives:

| Scenario | Flag Reason | Key Behaviour |
|----------|------------|---------------|
| PEP investigation | PEP | Senior-analyst routing, high trust scores, SAR filed |
| Structuring ring | STRUCTURING | Pattern detection, parallel specialist findings, multiple entities |
| Clean transaction | LOW_RISK | Investigation completes with no SAR — demonstrates false positive handling |
| Failed investigation | SYSTEM_ERROR | Specialist failure, failure context, fallback routing |
| Gate rejection | PEP | MLRO rejects SAR filing — shows gate rejection flow |
| GDPR-erased case | PEP | Post-erasure state — pseudonymised actors, erased entity memories |
| High-volume | STRUCTURING | Multiple concurrent investigations — shows throughput in ops view |

### Seed API

`POST /api/simulation/seed` — loads all scenarios by running canned `SuspiciousTransaction` records through the full Layer 9 pipeline. Idempotent — skips scenarios already present.

`POST /api/simulation/seed/{scenario}` — load a single scenario.

`DELETE /api/simulation/seed` — full data reset: truncates `InvestigationSummaryView`, engine `CaseInstance` records, `AmlTrustRoutingAttestation` entries, `CaseMemoryStore` entries, related `WorkItem` records, and ledger entries for seeded cases. This intentionally breaks Merkle chain integrity for the deleted entries — acceptable because simulation mode is mutually exclusive with production (see below).

### Live Simulation

`POST /api/simulation/investigate` — accepts a scenario template name, creates the `SuspiciousTransaction`, starts the investigation, returns the `caseId`. The UI can then poll the investigation detail endpoint to watch it flow through.

A simulation panel in the Operations view (or as a dedicated sidebar entry during demo mode) provides:
- Scenario selector (dropdown)
- "Run" button
- Link to watch the investigation in the Investigations view

### Simulation Mode

A configuration toggle (`casehub.aml.simulation.enabled=true`) that:
- Enables the simulation REST endpoints (disabled in production via `@IfBuildProperty`)
- Configures specialist workers to produce deterministic, scenario-appropriate results
- Adds the simulation panel to the UI

**Determinism mechanism:** Uses the existing CDI `@DefaultBean` displacement pattern established across all layers. Each specialist `AgentBehaviour` has a `@DefaultBean` stub that returns deterministic results. In production, real AI-backed implementations (without `@DefaultBean`) displace the stubs. In simulation mode, the stubs ARE the workers — they produce the canned results appropriate to the scenario.

The scenario template name is passed as part of the `SuspiciousTransaction` context (e.g., `flagReason: "PEP"` drives the PEP scenario path). The engine's binding evaluation then fires the correct adaptive path (PEP → senior-analyst-required, STRUCTURING → parallel pattern analysis). No interceptor or special extension point is needed — the existing stubs in `EntityResolutionBehaviour`, `PatternAnalysisBehaviour`, and `OsintScreeningBehaviour` already produce scenario-appropriate deterministic results because they read context from the `SuspiciousTransaction` and return structured `SpecialistOutcome` values.

The full Layer 9 pipeline runs (engine case, binding evaluation, parallel dispatch, gate WorkItems, ledger entries, trust attestations) — only the specialist execution is deterministic, not the orchestration.

**Production safety:** Simulation mode is mutually exclusive with production deployment. `@IfBuildProperty("casehub.aml.simulation.enabled", stringValue = "true")` is a build-time property — simulation endpoints do not exist in production builds. `DELETE /api/simulation/seed` performs a full data reset including ledger entries, which breaks Merkle chain integrity. This is acceptable because simulation runs against a dev/demo database, never production. The Quarkus dev profile uses `quarkus.datasource.devservices` (Testcontainers) by default, providing an isolated database per run.

---

## New API Endpoints Summary

The current API is case-centric (start investigation, get single case). The UI needs system-wide query and metrics endpoints.

### Foundation (push to shared repos)

These endpoints serve genuinely foundational needs — they are not AML accommodations. Any casehub-pages dashboard in any domain needs work item queries and ledger entry access. `work#241` already tracks the WorkItem read API gap as technical debt.

**Boundary justification:** PLATFORM.md's boundary rule ("Do not modify foundation repos to accommodate application needs") applies to domain-specific features. Query surfaces for generic platform primitives (work items, ledger entries) are foundational capabilities that multiple applications need. The correct flow: file issues in the relevant repos, propose the API design in those repos' specs, implement there.

| Endpoint | Method | Owner | Purpose | Issue |
|----------|--------|-------|---------|-------|
| `/api/work-items` | GET | casehub-work | Query WorkItems by status, category, group | casehubio/work#241 (existing) |
| `/api/work-items/{id}/escalate` | POST | casehub-work | Escalate a work item | To be filed |
| `/api/work-items/{id}/complete` | POST | casehub-work | Complete (approve/reject/override) a work item | To be filed |
| `/api/ledger/entries` | GET | casehub-ledger | Query entries by subjectId | To be filed |
| `/api/ledger/entries/{id}/proof` | GET | casehub-ledger | Merkle inclusion proof | To be filed |

Note: `GET /api/work-items` is a significant API design decision for casehub-work — WorkItems have a rich lifecycle (open → claimed → delegated → completed/expired/cancelled) and need filter semantics, pagination, tenant scoping, and access control. This deserves its own spec in casehub-work.

### AML-specific

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/layer9/investigations` | GET (list) | List/search investigations with filters |
| `/api/investigations/{caseId}/prior-context` | GET | Prior entity context used at investigation start |
| `/api/investigations/{caseId}/flow` | GET | Investigation path topology for graph rendering |
| `/api/investigations/{caseId}/findings` | GET | Structured specialist outcomes |
| `/api/investigations/{caseId}/gates` | GET | Gate decisions for a case |
| `/api/investigations/{caseId}/suspend` | POST | Suspend via `CaseHubRuntime.suspendCase()` |
| `/api/investigations/{caseId}/resume` | POST | Resume via `CaseHubRuntime.resumeCase()` |
| `/api/metrics/throughput` | GET | Aggregate investigation metrics |
| `/api/metrics/trust-scores` | GET | Current trust scores across agents |
| `/api/metrics/gates` | GET | Aggregate gate activity |
| `/api/simulation/seed` | POST/DELETE | Seed/clear demo scenarios |
| `/api/simulation/seed/{scenario}` | POST | Seed single scenario |
| `/api/simulation/investigate` | POST | Run live simulation |

---

## File Structure

```
app/src/main/webui/
├── package.json
├── tsconfig.json
├── esbuild.config.mjs
├── .npmrc
└── src/
    ├── index.ts              # App entry — loadSite() call
    ├── views/
    │   ├── work-queue.ts     # Work queue view builder
    │   ├── investigations.ts # Case list + detail view builder
    │   ├── accountability.ts # Audit trail, Merkle, compliance, GDPR
    │   └── operations.ts     # Metrics, trust, gates, intervention, simulation
    ├── datasets/
    │   └── index.ts          # All dataset definitions
    ├── components/
    │   └── investigation-flow/ # Custom iframe component for flow viz
    │       ├── index.html
    │       ├── flow.ts
    │       └── flow.css
    └── simulation/
        └── scenarios.ts      # Scenario template definitions
```

---

## Parameterised Dataset URLs

The case detail view needs datasets that take `{caseId}` as a parameter (e.g., `/api/investigations/{caseId}/flow`).

**Design:** Case detail datasets are created dynamically when the user drills down from the case list. The click handler receives the selected `caseId` and constructs dataset instances with the parameterised URL resolved:

```typescript
function openCaseDetail(caseId: string) {
  dataset("case-detail", { url: `/api/layer9/investigations/${caseId}` });
  dataset("case-compliance", { url: `/api/investigations/${caseId}/compliance-evidence` });
  dataset("case-ledger", { url: `/api/ledger/entries?subjectId=${caseId}` });
  // ... remaining case-detail datasets
}
```

If casehub-pages does not support dynamic dataset creation (datasets must be declared at `loadSite()` time), the fallback is a single `case-detail` dataset that fetches a composite response from a new `GET /api/investigations/{caseId}/detail` endpoint aggregating all case-detail data into one response. The workbench sets the active caseId via `ComponentApi` state, and the dataset URL includes it as a query parameter.

Either approach works. The composite endpoint is less elegant (one fat response vs multiple focused datasets) but eliminates the parameterisation question entirely.

---

## Role Model

### Gated compliance actions

Three candidate groups govern access to gated actions, derived from `AmlGroups` and the `candidateGroups` on each `AmlActionType`:

| Group | Role | Permitted Actions |
|-------|------|-------------------|
| `aml-compliance` | Compliance Officer | ACCOUNT_RESTRICTION, TRANSACTION_BLOCKING, ENTITY_LINK_CREATION |
| `aml-mlro` | Money Laundering Reporting Officer | SAR_FILING (exclusive — tightest gate in the system) |
| `aml-senior-compliance` | Senior Compliance Director | LAW_ENFORCEMENT_REFERRAL |

**Work queue filtering:** The work queue filters by `candidateGroups` — a user in `aml-compliance` sees only gate WorkItems for actions assigned to that group. The MLRO sees only SAR filing gates.

### Non-gated consequential actions

These actions fall outside the `AmlActionType` gate model but are consequential and require access control:

| Action | Required Group | Rationale |
|--------|---------------|-----------|
| GDPR erasure (actor/entity) | `aml-senior-compliance` | Irreversible, compliance-consequential — erases memories and pseudonymises ledger entries. Requires senior sign-off. |
| Suspend investigation | `aml-compliance` or `aml-mlro` | Operational control — any compliance officer can pause an investigation for review. |
| Resume investigation | `aml-compliance` or `aml-mlro` | Same as suspend — resuming is equally consequential. |
| Escalate work item | `aml-compliance` or `aml-mlro` | Operational control — escalation changes SLA and routing. |
| Simulation seed/investigate | N/A (build-time gated) | Simulation endpoints only exist in simulation builds (`@IfBuildProperty`). No role-based access needed — the endpoints are absent in production. |

### UI enforcement

Role-based UI filtering (hiding views, disabling buttons) is deferred to the auth implementation. The spec's `withAccess()` reference in "What This Spec Does Not Cover" is the casehub-pages mechanism for this. For now, the workbench shows all views to all users — gate actions are protected at the API level by WorkItem candidateGroup matching, not by UI visibility. Non-gated consequential actions will be protected at the REST endpoint level using the group mappings above.

---

## Deferred Concerns — Issue Tracking

Items explicitly out of scope for this spec, tracked as GitHub issues:

| Concern | Issue | Notes |
|---------|-------|-------|
| Authentication and role-based UI access | To be filed in casehub-aml | `withAccess()` mechanism available in casehub-pages |
| Trust score historical trend persistence | To be filed in casehub-aml | `TrustScoreCache` holds current scores only; trend timeseries needs snapshot persistence |
| WorkItem query API design | casehubio/work#241 (existing) | Rich lifecycle, pagination, tenant scoping — deserves its own spec |
| Ledger entry query endpoint | To be filed in casehub-ledger | `GET /api/ledger/entries?subjectId={id}` |
| Ledger proof REST endpoint | To be filed in casehub-ledger | `GET /api/ledger/entries/{id}/proof` |
| WorkItem escalate endpoint | To be filed in casehub-work | `POST /api/work-items/{id}/escalate` |
| WorkItem complete endpoint | To be filed in casehub-work | `POST /api/work-items/{id}/complete` |
| WebSocket/SSE real-time updates | To be filed in casehub-aml | Replace polling with push when scale demands it |
| casehub-pages consumption model verification | To be filed in casehub-pages | Verify DSL API vs iframe embedding — see §Parameterised Dataset URLs |

---

## What This Spec Does Not Cover

- Authentication and role-based access (deferred — `withAccess()` is available when needed)
- Websocket/SSE for real-time updates (start with polling via `refreshTime`, upgrade later)
- Mobile responsiveness (desktop-first operational tool)
- Internationalisation
- Offline mode
