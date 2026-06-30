---
layout: post
title: "From Nine Layers to a Screen"
date: 2026-06-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [workbench, casehub-pages, design-review, SNAPSHOT]
---

I wanted to know what it would feel like to operate the AML system we'd built. Nine layers of investigation infrastructure — specialist dispatch, trust routing, Merkle-chained audit, oversight gates, case memory — and no way to see any of it except through curl and test assertions.

The answer turned out to be a casehub-pages workbench. Not a dashboard bolted on after the fact, but a real operational tool: four views covering the full investigation lifecycle. A compliance officer's work queue with SLA countdown. An investigation viewer that reconstructs the adaptive path the engine took — which specialists fired, which ran in parallel, what each found, what trust score drove the routing decision. An accountability view where you can trace every decision back through the Merkle chain to the case-opened event. And an operations view showing the system running: throughput, trust scores, gate approval rates.

The design review was the most productive part. Two independent Claude sessions — one reviewing, one implementing — went nine rounds. The reviewer caught a genuine cross-component interaction gap: the work queue's callerRef filter didn't account for two different WorkItem creation paths (compliance review vs oversight gates) using different callerRef formats. It also caught that the investigation list endpoint needed a denormalised read model — you can't cross-join engine state and ledger data at query time because they live on different persistence units. Both were the kind of design error that would have surfaced much later as a bug.

The backend API is thirteen new endpoints across four resource classes, a denormalised `InvestigationSummaryView` table with a `CaseLifecycleEvent` observer, three metrics aggregations, and a simulation API with seven scenario templates. The simulation is the demo enabler — seed a PEP investigation, a structuring ring, a gate rejection, a GDPR-erased case, and you can walk someone through the workbench without waiting for real alerts.

Two engine SNAPSHOT changes bit during implementation. `YamlCaseHub.getDefinition()` went final overnight with no changelog — the replacement is a protected `augment()` hook, discoverable only by decompiling the jar. And `Worker.Builder` dropped the `Capability` record entirely in favour of plain capability name strings. Both are simplifications — the new API is better — but the migration path was "compilation error, then read bytecode."

The qhorus SNAPSHOT was worse. A new `persistence-memory` module added in-memory store beans at `@Default` without priority annotations, creating a three-way CDI ambiguity with the existing testing and JPA stores. Excluding the blocking stores seemed like the fix — until the reactive wrappers, which inject the blocking stores by concrete type rather than interface, produced a second wave of unsatisfied dependencies. Two rounds of "fix one CDI error, get a different one" before the full exclusion list stabilised.

The frontend is next. The casehub-pages DSL makes this surprisingly tractable — `page()`, `sidebar()`, `dataset()`, `table()`, and the data layer handles filter state, pagination, and REST binding. The investigation flow visualisation needs a custom iframe component (directed graph of the adaptive path), but the other three views are standard compositions. The spec, the backend API, and the plan are all ready.
