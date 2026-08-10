---
layout: post
title: "The DSL you don't use"
date: 2026-08-10
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [casehub-pages, blocks-ui, architecture, documentation-drift]
---

The AML workbench spec, written six weeks ago, assumed casehub-pages would handle everything — `loadSite()`, `page()`, `sidebar()`, `dataset()`. Four views, parameterised drill-down, the full component-tree model. It described a standalone `dataset()` function you'd call at runtime when a user clicked a case:

```typescript
function openCaseDetail(caseId: string) {
  dataset("case-detail", { url: `/api/investigations/${caseId}` });
}
```

That function doesn't exist.

`buildDataSetScope()` walks the component tree once, at `loadSite()` time, and builds a static `Map<string, Map<DataSetId, DataSetEntry>>`. The `LiveSite` interface that comes back exposes `navigate()`, `setTheme()`, `dispose()`, and `layout`. No dataset mutation. The scope is frozen.

I only discovered this by reading `dataset-scope.ts` directly. There's no documentation saying "datasets are static." The exported names — `buildDataSetScope`, `resolveDataSetDef`, `extendDataSetScope` — suggest a management layer that invites runtime use. `extendDataSetScope` even sounds like what you'd want. It's for lazy page resolution — adding pages to the tree — not for application-driven data loading.

The workbench that was actually built ignored all of this. It uses Lit custom elements and blocks-ui components — `blocks-split-workbench`, `blocks-list-pane`, `blocks-detail-pane` — with direct REST endpoint attributes. Click a case, fire a selection-topic event, the detail pane fetches the data. No DSL, no component tree, no dataset scope. The parameterised URL problem that the spec agonised over (with a fallback composite endpoint design) simply doesn't arise.

The interesting part: the implementation was right and the spec was wrong, but the spec was never updated. Four closed issues later, the deferred concerns table still described a world where `TrustScoreCache` existed and `ComplianceReviewLifecycle` had no scope. The drift was invisible because the code worked fine — nobody needed the spec to build anything new.

This is the failure mode for living documentation: it rots fastest when the implementation succeeds. A broken spec that blocks work gets fixed immediately. A stale spec that nobody reads survives indefinitely.
