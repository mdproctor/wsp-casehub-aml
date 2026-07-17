# Handoff — #102 showcase page in progress (2026-07-17)

## What this project is

CaseHub AML — anti-money laundering investigation application built on CaseHub platform. 9 foundation layers complete. Frontend migrated from pages-DSL to Lit + blocks-ui web components.

## This session (2026-07-14 → 2026-07-17)

Continued #102 showcase page work. Fixed TypedRow extraction across all 5 detail panels — `detail-pane` passes `TypedRow` (from pages-data `extractDataSet`) as the `item` property, but panels were accessing `.caseId` directly instead of via `.text('caseId')`. Overview panel had a deeper issue: template accesses `.amount`, `.currency` etc. as plain properties, but `TypedRow` requires `.text()` / `.number()` accessors. Fixed by converting TypedRow to plain `InvestigationSummaryResponse` object in the `caseData` getter. Sidebar narrowed from 240px to 160px.

**Commits on main:**
- `b75b685` fix(#102): showcase TypedRow extraction + sidebar width
- `2298170` fix(#102): showcase element-guard + mock route fixes
- `df0dc6b` feat(#102): showcase page + SNAPSHOT alignment + blocks-ui migration

## Immediate next step

Rebuild the frontend (`npm run build` in `app/src/main/webui/`) and verify showcase in browser. The TypedRow fix was committed but not verified — the `caseData` getter conversion may have new issues (e.g. `toLocaleString` on `number()` return if the column doesn't exist).

## Showcase status — what works, what doesn't

### Working
- **Investigations list** — 6 mock investigations render in pages-table with column headers, sorting, pagination
- **Sidebar nav** — Investigations / Compliance / Operations with hash routing
- **Mock fetch interceptor** — all `/api/*` calls return realistic mock JSON with 50-200ms latency
- **Element guard** — prevents duplicate custom element registration from esbuild bundling conflicts
- **Detail tabs** appear on row selection (Overview, Findings, Routing & Trust, Compliance, Audit)

### Broken or untested (continue here)
1. **Overview panel** — `_renderTransactionCard()` hit `toLocaleString()` on undefined. Fix committed (TypedRow→object conversion in `caseData` getter) but not verified. Check: click an investigation row, Overview tab should show transaction card with amount, currency, flag reason
2. **Findings panel** — untested with TypedRow fix. Each section (entity-resolution, pattern-analysis, osint-screening, sar-narrative) should expand/collapse with mock data
3. **Routing & Trust panel** — untested with TypedRow fix. Should show routing decisions table, trust score placeholder, and gate decisions
4. **Compliance panel** — untested with TypedRow fix. Should show SAR status card, FinCEN requirements table, officer review, GDPR erasure section
5. **Audit Trail panel** — untested with TypedRow fix. Should show ledger entries table with verify buttons
6. **Compliance nav view** — `work-item-inbox` component had `assigneeId` undefined error. Mock data shape at `/workitems/inbox` may not match what blocks-ui `work-item-inbox` expects
7. **Operations nav view** — `kpi-metric-row` components render metrics. Untested
8. **Triple-dot column visibility button** — doesn't work. Likely a pages-table bug or missing event wiring. Low priority
9. **esbuild rebuild** — run `npm run build` after any source change; `npm run build:showcase` builds the showcase entry point

### Architecture notes for continuing

**Entry points:**
- `src/aml-app.ts` — root app element, sidebar nav, hash routing
- `src/showcase/index.ts` → imports `element-guard.ts` (must be first), `mock-fetch.ts`, then `../aml-app.ts`
- `esbuild.config.mjs` — `casehubResolvePlugin` resolves all `@casehubio/*` imports via AML's node_modules

**TypedRow dual-path pattern (all panels):**
```typescript
get caseId(): string { return this.item?.text?.('caseId') ?? this.item?.caseId ?? ''; }
```
Overview panel goes further — converts entire TypedRow to plain object for template rendering.

**Dev server:** `npx http-server app/src/main/webui/dist -p 8099` then open `http://localhost:8099/showcase.html`

**Mock data:** `src/showcase/mock-data.ts` — 6 AML investigation scenarios (structured deposits, PEP, smurfing, cross-border, nominee, cleared)

## What's left (backlog)

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Showcase verification & remaining UI fixes | S | Low | Verify all 5 detail tabs render, fix compliance nav |
| 101 | Dual-mode datasets — inline mocks + live endpoints | M | Med | |
| 92 | CBR epic — case-based reasoning for AML investigation | XL | High | Start with #93 |
| 89 | WebSocket/SSE real-time updates for workbench | M | Med | |
| 86 | Authentication and role-based access | L | High | |
| 81 | Automated retention expiry | M | Med | |

## Cross-repo state

- **blocks-ui** — casehubio/blocks-ui#51 resolved (import re-export fix). AML uses blocks-ui components: split-workbench, list-pane, detail-pane, data-table, work-item-inbox, approval-gate, kpi-metric-row, sla-indicator
- **pages** — casehubio/pages#73 filed (DataSource abstraction for pages-data). TypedRow / TypedDataSet / extractDataSet pipeline is the data layer
- **engine** — casehubio/engine#743 filed (document inputSchema→inputProjection rename)

## References

- Blog: `blog/2026-07-05-mdp01-the-screen-that-lies.md`
- Spec: `specs/2026-06-30-aml-workbench-ui-design.md`
- Plans: `plans/2026-06-30-aml-workbench-frontend.md`
- Garden: GE-20260707-99de4f (LedgerEntry @MappedSuperclass vs JpaLedgerEntry)
- Garden: GE-20260707-dda82d (Quarkus test selected-alternatives reactive repos)
