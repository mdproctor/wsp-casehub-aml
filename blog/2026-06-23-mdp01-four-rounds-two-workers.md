---
layout: post
title: "Four rounds to get a two-worker migration right"
date: 2026-06-23
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [worker-migration, funcworkflowbuilder, spec-review, engine-559]
series: issue-66-sar-workers-flow
---

*Part of a series on [#66 — migrate SAR drafting workers to FuncWorkflowBuilder](https://github.com/casehubio/aml/issues/66). Previous: [Cache gaps and workflow context](2026-06-22-mdp01-cache-gaps-and-workflow-context.md).*

The last entry left two SAR drafting workers stranded on `WorkerFunction.Sync` because `DefaultWorkerExecutor.executeFlow()` didn't set `WorkerExecutionContext`. Engine#559 fixed that — a one-line change on the engine side. The migration itself was straightforward: wrap the lambda body in `workflow("sar-drafting-junior").tasks(function(s -> { ... }, Map.class)).build()`, drop the `WorkerResult.of()` wrapper, done.

The code change took minutes. The spec took four review rounds.

## What the reviews caught

The first draft claimed Layer 9 tests covered the SAR drafting path. They don't. `AmlLayer9ActionGateTest` runs `AmlOversightCaseHub` — different case hub, zero SAR workers. Layer 6 is the regression net, but through implicit coverage: if `WorkerExecutionContext.current()` returns null, the engine case fails and Awaitility times out. I'd written "both Layer 6 and Layer 9 verify complianceTaskId" — wrong on both halves.

The second round caught scope. `AmlOversightCaseHub` has three workers still on raw lambdas — a PP-20260531 violation I'd buried in "Out of Scope" with no explanation. Two of the three (`entityResolutionWorker`, `investigationSummaryWorker`) use single-arg `WorkerResult.of(Map)` and can be migrated today. The third (`entityLinkProposalWorker`) returns `WorkerResult.of(Map, PlannedAction)` and is genuinely blocked — `executeFlow` calls `model.asMap()` on the output; there's no pathway for a `PlannedAction` through the FuncDSL chain. That distinction matters: "oversight workers can't migrate" is wrong; "one oversight worker can't migrate because of PlannedAction" is the real constraint.

The same round flagged the CLAUDE.md note — "Applies to all YamlCaseHub worker lambdas" was now false for seven workers. A developer adding a new Flow worker following that note would incorrectly wrap with `WorkerResult.of()`.

The third round caught the class-level Javadoc. Removing "SAR drafting workers remain as WorkerFunction.Sync..." without updating "Pure-computation workers use..." left the Javadoc implying a category that no longer exists.

The fourth round was clean. One minor: a comment in the code snippet that belonged in spec prose, not in production code.

## The PlannedAction gap

The `entityLinkProposalWorker` constraint goes deeper than "FlowWorkerExecutor doesn't support PlannedAction." The PlannedAction metadata needs `ownershipChain`, which is in the worker's input (`entityResolution.ownershipChain`) but not in its output map (`{proposedLink, entityType, riskScore}`). If the engine adds `withPlannedAction(actionFn)`, the `actionFn` must receive the task input — not the output — or the caller is forced to widen the output contract to include data that doesn't belong in the case context. Filed as engine#564 with that API direction specified.

## The SNAPSHOT lesson

`mvn -U dependency:resolve` hit a 401 from GitHub Packages and silently overwrote the working engine-api jar with a broken one. Unit tests passed — they check `assertInstanceOf(WorkerFunction.Flow.class, ...)` which resolves from a different classpath module. The integration test failed with `NoClassDefFoundError`. Recovery meant building the engine locally from source, which meant cleaning untracked files from a feature branch that had contaminated the working tree. Filed as garden entry GE-20260623-eb19c0.
