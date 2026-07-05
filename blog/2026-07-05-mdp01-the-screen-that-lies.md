---
layout: post
title: "The Screen That Lies, and the Reasoning That Doesn't Exist Yet"
date: 2026-07-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [workbench, casehub-pages, CBR, frontend]
series: issue-91-aml-workbench-ui
---

*Part of a series on [#91 — AML workbench UI](https://github.com/casehubio/aml/issues/91). Previous: [From Nine Layers to a Screen](2026-06-30-mdp01-from-nine-layers-to-a-screen.md).*

The backend was done. Thirteen endpoints, simulation API, metrics — all tested. The frontend plan said "six tasks, standard casehub-pages composition." Four sidebar views, inline datasets for anything the foundation doesn't serve yet, bar charts and donut charts from the pages DSL. Should have been mechanical.

It wasn't.

casehub-pages is a TypeScript DSL that compiles to Web Components. The DSL uses branded types — `ColumnId` is `string & { __brand: "ColumnId" }`, `ColumnType` is an enum with values `DATE`, `NUMBER`, `LABEL`, `TEXT` (not `STRING`), and `CoreFunctionType` uses `EQUALS_TO` (not `eq`). None of this is in the getting-started docs. The actual `ColumnSettings` interface has five fields: `id`, `name`, `expression`, `pattern`, `empty`. No `width`. No `hidden`. No `type`. I'd been writing column definitions based on what the API felt like it should accept rather than what it actually does.

The real discovery was about data shape. The metrics endpoints return nested JSON objects — `{"totalInvestigations": 142, "byStatus": {"IN_PROGRESS": 12}}` — but pages-data expects tabular data: arrays of rows with consistent columns. The extraction pipeline tries JSON parse first, falls through to CSV, and throws `EXTRACTION_ERROR` when neither produces rows. And the `dataPath` property extracts a nested array from a response — it doesn't reshape an object into rows.

The fix was to use `inlineDataset()` with CSV mock data everywhere. Every dataset — work queue, investigations, throughput metrics, trust scores, gate metrics, audit trail, compliance evidence, GDPR erasure log — is now inline. When the real endpoints ship (casehub-work#241 for work items, casehub-ledger#162 for the audit trail), swapping `inlineDataset()` for `dataset()` with a URL is a one-line change per dataset. But the metrics endpoints need a different fix: they need to return tabular data, not aggregate objects, or the pages data layer needs an extraction mode for single-object responses.

Quinoa had its own surprises. `package-manager-install=true` requires an explicit `node-version` property — omit it and the build fails with a message about "not required when using only bun-version" that sends you down the wrong path entirely. And in dev mode, Quinoa serves from `target/quinoa/build/`, not from `src/main/webui/dist/`. Manual `npm run build` updates dist/ but the running server doesn't see it. You have to restart `quarkus:dev` for Quinoa to copy the fresh output. Both are now in the garden.

The workbench runs. Four views, all rendering with mock data. Work queue shows SLA status with red/amber/green row styling. Investigations list with status colours and a detail accordion — prior context, specialist findings, oversight gates, compliance review, failure context. The investigation flow iframe component renders a directed graph of the adaptive path with parallel branches shown side-by-side. Accountability has the full Merkle chain audit trail and compliance evidence mapping. Operations shows throughput, trust scores, gate activity, and an SLA donut. Simulation and intervention panels are stubs waiting for form submission support.

The more interesting thing that happened was a question about Case-Based Reasoning. The AML system already has the pieces: `CaseMemoryStore` retrieves prior entity context, trust scoring retains outcomes, the adaptive `CasePlanModel` reuses context for routing. Full CBR would formalise this into a complete Retrieve-Reuse-Revise-Retain cycle operating at case level — "this transaction pattern + jurisdiction + entity type is similar to 14 past cases, 12 of which were SAR-filed." The epic is filed as #92 with seven child issues covering the similarity model, case profile store, retrieval, path adaptation, outcome retention, SAR narrative seeding, and cold-start bootstrapping. The cold-start problem is the interesting constraint — CBR adds value proportional to case base size, so you need either synthetic seed cases or a learning period where CBR observes without influencing routing.
