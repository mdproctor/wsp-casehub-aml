---
layout: post
title: "Five layers of red"
date: 2026-08-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [ci, testing, triage, cbr, dependency-management]
---

AML's CI had been red since July 19. Ten consecutive failures, each masking the next. I sat down to fix what looked like a single yarn authentication error and ended up peeling five layers.

The first three were infrastructure. An earlier commit had added sibling checkout steps for casehub-pages and blocks-ui — yarn install, yarn build, symlink as siblings. AML is a pure Java project with zero npm dependencies. The checkout was unnecessary. Removing it revealed a Maven enforcer convergence failure: `json-schema-validator` 1.5.4 vs 2.0.0, pulled in through quarkus-flow's serverlessworkflow dependency. Pinning to 1.5.4 (the version casehub-work compiles against) fixed the enforcer. Below that, `AmlLayer5ResourceTest` was sending free-text strings where the `FlagReason` enum now expects constants — missed during the String-to-enum migration.

The fourth layer was an engine regression. `DefaultCaseDefinitionRegistry.getCaseDefinition()` returned null for every case started by a consumer app. We traced it to a vestigial Mutiny pipeline left behind by the #770 virtual thread migration — the Uni chain swallowed registration failures silently. The engine's own tests passed because they use in-memory stores without `@Transactional` interceptors. Filed as engine#787; two fix attempts before it resolved.

The fifth layer was the hardest to see. The investigation triage logic from #112 replaced a stub that always returned `SAR_WARRANTED` with real rule-based evaluation. Every integration test assumed every investigation produces a gate WorkItem. Low-risk cases now correctly return `FALSE_POSITIVE` and exit via `investigation-cleared` — no SAR, no gate. The symptom was Awaitility timeouts on `awaitAndApproveGate()`, which looked like a gate creation bug rather than a correct triage decision. Fourteen test classes needed updating: tests for the low-risk path remove the gate step and drain directly; tests requiring the SAR path use `HIGH_RISK_JURISDICTION` which triggers the `SHELL_COMPANY` hard gate deterministically.

Along the way we removed the junior osint-screening and sar-drafting workers. The engine's `PlanningStrategyLoopControl` can only track one PlanItem per capability — two workers for `osint-screening` left one PlanItem stuck RUNNING indefinitely, blocking case completion. The trust routing always selected the senior anyway (score 0.818 above the 0.80 threshold), so the juniors were dead code causing live failures.

The last fix was the most satisfying to root-cause. `SarNarrativeSeedingIntegrationTest.coldStart` passed in isolation (8 seconds) but timed out after 60 seconds in the full suite. I assumed Quartz thread starvation — increased the pool to 25, which fixed most intermittent failures but not this one. The diagnostic trace showed the gate WorkItem was actually being created and approved — the test was succeeding on one surefire retry. The first run was a FAILURE, not an ERROR. Reading the assertion: `narrativeSeeded must be false with empty case base — expected: false but was: true`. The CBR store from `AmlCaseProfileStoreObserverTest` (which runs first alphabetically) had contaminated the "cold start" assumption. The triage evaluator found similar past cases, the sar-drafting worker seeded narratives from them, and the assertion failed. One line: `cbrStore.eraseByScope(Path.root(), TENANT)`.

322 tests pass in 51 seconds. CI remains red until engine and work publish their latest SNAPSHOTs to GitHub Packages — the `GateRequired` constructor gained a `QuorumConfig` parameter that the published version doesn't have yet, and `casehub-work-engine-adapter` needs to publish with its blackboard→planning migration. No AML code changes needed for that.

I mapped the remaining backlog into a wave plan. Four S-scale quick wins first (#113, #115, #81, #87 — all independent, one session each), then M-scale parallel pairs across CBR, UI, and GDPR epics. Twelve issues across four epics before anything XL-scale needs a brainstorm.
