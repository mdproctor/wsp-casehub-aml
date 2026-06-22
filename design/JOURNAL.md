# Design Journal — issue-65-s-batch-fixes

### 2026-06-22 · §9.4·casehub-engine

Added `casehub-engine-flow` as a compile dependency, activating `FlowWorkerExecutor` which displaces `NoOpWorkflowExecutor @DefaultBean`. This enables the `WorkerFunction.Flow` execution path for workers using `FuncWorkflowBuilder`, satisfying protocol PP-20260531-worker-func-exec (workers in production case definitions must use the quarkus-flow workflow pipeline, not raw `Function<Map, WorkerResult>` lambdas). Five of seven workers in `AmlInvestigationCaseDescriptor` are now `WorkerFunction.Flow`. Two SAR drafting workers remain `WorkerFunction.Sync` because `DefaultWorkerExecutor.executeFlow` does not set `WorkerExecutionContext` before calling `FlowWorkerExecutor.execute()` — the SAR workers call `WorkerExecutionContext.current().caseId()` to pass the case ID to `ComplianceReviewLifecycle.openReview()`. This gap is tracked in engine-side and AML-side issues (#66).
