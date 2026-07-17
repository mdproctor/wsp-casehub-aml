---
layout: post
title: "Six columns that matter"
date: 2026-07-17
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [frontend, ux, showcase]
series: issue-103-showcase-ux-polish
---

*Part of a series on [#102 — standalone showcase page](https://github.com/casehubio/aml/issues/102). Previous: [The screen that lies](2026-07-05-mdp01-the-screen-that-lies.md).*

The investigation list had ten columns and showed nothing useful. Every cell was truncated — `c1a2b3c4-...`, `ACC-CORP-...`, `TXN-2024-...`. An analyst scanning 50 cases would see ten columns of ellipses.

I looked at what Focal, Sumsub, AMLYZE, and Unit21 actually show in their AML case lists. The answer is five or six columns: status, risk level, flag reason, amount, outcome, date. Everything else — case IDs, transaction IDs, account numbers — belongs in the detail view, not the list. The list answers "which case do I pick up next?" and for that, risk score and flag reason are the two columns that matter most.

The fix was straightforward. Six visible columns with renderers: status as a colour-coded badge, risk score as a percentage (red above 80%, amber 50–80%, green below), amount merged with currency, outcome, flag reason with room to breathe, and created date. The five low-value columns get `visible: false` in the column config — they're still in the data and accessible through the column picker (the `⋮` button that I'd assumed was broken — it was always implemented in pages-table, just needed columns with config entries to appear in the dropdown).

The interventions tab was the other gap. It had action cards (suspend, resume, escalate, override) but no metrics. An operations manager opening that tab should see counts first — how many escalations, how many DECLINE re-routes, how many gate rejections — before reaching for the action buttons. Five metric cards and a recent interventions feed with colour-coded type badges now sit above the action grid.

The mock data needed fixing too. The previous session's TypedRow work had left several shape mismatches between the mock JSON and the TypeScript types — `outcomeType`/`resolution` instead of `type`/`reason` on outcomes, flat prior context instead of the structured `PriorContextResponse`, work items missing the `WorkItemRootResponse` wrapper. The `createdAt` column was typed as DATE in the dataset but the overview panel was calling `.text()` on it — should have been `.date()`. All fixed, all verified in the browser across every tab and view.

The column picker discovery is the interesting one. I'd filed an issue for it, investigated the pages-table source, and found the full implementation already there — `_toggleColumnPicker`, checkbox dropdown, display mode switcher. The "bug" was that columns without explicit `columnConfig` entries don't appear in the picker. Once every column has a config entry (even just `{ id: 'caseId', visible: false }`), the picker shows all of them and toggling works.
