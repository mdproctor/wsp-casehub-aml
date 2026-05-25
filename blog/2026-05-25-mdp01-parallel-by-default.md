---
layout: post
title: "Parallel by default"
date: 2026-05-25
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
---

I wanted to add casehub-engine to the AML investigation without rebuilding what Layers 3 and 4 already did. The engine should replace the fixed sequential pipeline in `QhorusAmlInvestigator` — not wrap it.

The case definition design took some thought. The engine needs capabilities, bindings with JQ conditions, goals, and worker functions. YAML handles the first three cleanly — bindings are declarative data and the YAML mapper is the right tool for them. Worker functions are Java lambdas that capture CDI proxies. YAML can't express that. The choice was: YAML for structure, Java augmentation for workers.

`AmlInvestigationCaseHub` extends `YamlCaseHub`, loads `aml/aml-investigation.yaml`, then augments the definition with five workers inside a synchronized `getDefinition()` call. The workers are simple lambdas. The OSINT worker always declines — it returns `{declined: true, ...}` to context, which satisfies the `osintScreening != null` condition and lets sar-drafting proceed. DECLINE is a valid outcome, not a failure.

The five bindings are all `contextChange` triggers with JQ `when:` conditions:

```yaml
- name: entity-resolution
  on: { contextChange: {} }
  when: ".transaction != null and .entityResolution == null"
  capability: entity-resolution

- name: pattern-analysis
  on: { contextChange: {} }
  when: ".entityResolution != null and .patternAnalysis == null"
  capability: pattern-analysis

- name: osint-screening
  on: { contextChange: {} }
  when: ".entityResolution != null and .osintScreening == null"
  capability: osint-screening

- name: senior-analyst-required
  on: { contextChange: {} }
  when: >-
    .entityResolution != null and
    (.entityResolution.entityType == "PEP" or .entityResolution.riskScore > 0.8) and
    .seniorAnalystReview == null
  capability: senior-analyst-review
```

The engine evaluates all four simultaneously on every context change. After entity-resolution completes and writes to context, the engine re-evaluates every binding in one pass — pattern, OSINT, and (for a PEP entity) senior-analyst all become eligible at the same moment and fire in parallel. There's no declaration like "run these concurrently." Parallel is just what happens when multiple conditions become true at once.

The test took longer to write than the implementation. We were polling `WORKER_EXECUTION_COMPLETED` events and checking for `workerName` in metadata. The workers were completing in under 50 milliseconds per the debug logs, but the Awaitility condition timed out at 10 seconds every time. The events existed. The metadata didn't have the key we were looking for.

It turned out `WorkflowExecutionCompletedHandler.buildMetadata()` puts `inputDataHash` and `contextChanges` into the metadata, not `workerName`. That lives in `WORKER_SCHEDULED` events, written by `WorkerScheduleEventHandler`. We switched to polling scheduled events — a worker is only scheduled when its binding condition is satisfied, so sar-drafting being scheduled is proof that entity, pattern, and OSINT all completed and wrote to context. It's actually the right check.

In code review, Claude flagged that `AmlEngineCoordinator` was generating its own UUID before calling `startCase()`, writing the ledger entry with that UUID, then discarding it and returning the engine's UUID. The spec says the case UUID must link the ledger, the engine event log, and the WorkItem. Two UUIDs means nothing is linked. We fixed the order: `startCase()` first, write the ledger with the returned UUID, return it.

The integration tests now pass cleanly. A PEP transaction routes to the senior analyst binding. A non-PEP transaction routes to pattern and OSINT in parallel. OSINT declining doesn't block SAR drafting. The `investigation-complete` goal fires when the SAR is drafted and the compliance review is open — the engine case reaches `COMPLETED` status.

Layer 5 is done. Layer 6 adds trust routing — experienced analysts routed to complex cases based on SAR outcome attestations. That does need engine#336 and #337.
