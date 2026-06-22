---
layout: post
title: "Cache gaps and workflow context — two fixes, one surprise"
date: 2026-06-22
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [casehub-engine, worker-execution, cache, FuncWorkflowBuilder]
---

The investigation status endpoint had a subtle defect. Both Layer 6 and Layer 9 used
`CaseInstanceCache.get(caseId)` to determine whether a case had completed. This was
introduced deliberately — the previous approach scanned `WorkerDecisionEntry` records,
which are written asynchronously by observers and proved unreliable. The cache felt
like a cleaner signal.

The problem is TTL. Cache eviction is an engine implementation detail, and the AML
resources had no fallback when `get()` returned null. A client that polled with any
delay after completion would see "in-progress" indefinitely. We added
`CaseInstanceRepository` as the fallback: try the cache first, miss → query the
repository. The repository is persistent; it doesn't evict. In the in-memory configuration
AML currently runs, this means two different backing stores, and the cache miss correctly
routes to the one that doesn't forget.

The code is six lines per resource — not interesting. What's interesting is that the
cache eviction scenario can't be reproduced in a test with normal Awaitility polling,
because the engine populates the cache before the test's GET fires. To verify the fallback
path actually runs, we inject `CaseInstanceCache` into the test and call `clear()` after
the drain. Then the next GET has to go to the repository. Clean and direct.

---

The worker protocol migration had a different character. The requirement — all workers in
production case definitions must use `FuncWorkflowBuilder.workflow().tasks(function(...)).build()`
rather than raw lambda — seemed straightforward. Seven workers, same wrapping pattern each time.

It wasn't quite that clean.

Five of the seven workers are pure computation: entity resolution, pattern analysis, the two OSINT
screening stubs, and the senior analyst reviewer. These converted without incident. The lambda
body stays the same; you wrap it in `workflow("name").tasks(function(lambda, Map.class)).build()`.
The one difference worth noting: the raw `Function<Map, WorkerResult>` path has the lambda return
`WorkerResult.of(Map.of(...))`, but the flow path expects `Map.of(...)` directly. The engine's
`executeFlow` calls `model.asMap()` on the workflow output and wraps the result itself.

The two SAR drafting workers were the problem. Both call `WorkerExecutionContext.current().caseId()`
to get the case ID for opening the compliance review WorkItem. That's the only way to get it —
the case ID isn't in the input data passed to the worker, and it's not returned through any other
channel.

`WorkerExecutionContext` is a ThreadLocal. The engine's `DefaultWorkerExecutor` sets it
before executing sync workers:

```java
WorkerExecutionContext.set(context);
try {
    return fn.apply(inputData);
} finally {
    WorkerExecutionContext.clear();
}
```

The flow execution path doesn't. I read the bytecode of `FlowWorkerExecutor` to confirm it —
no `WorkerExecutionContext.set()` call, and `WorkflowDefinition.instance(Object)` takes only
the input data. The workflow instance ID is auto-generated, not the case ID. So `current()`
returns null inside a `FuncWorkflowBuilder` lambda.

The SAR workers stay as `WorkerFunction.Sync` until the engine adds context setup in the flow
path. That's a one-line fix on the engine side — `WorkerExecutionContext.set(context)` before
delegating to `FlowWorkerExecutor.execute()` — but it's not there yet.

There's a broader pattern here. The flow execution model assumes workers are stateless and derive
everything from their input data. Workers that need runtime context — the case ID, the current
tenant, anything set on the Quartz thread before execution — don't fit that model. Something to
design for explicitly if the engine's worker contract evolves.
