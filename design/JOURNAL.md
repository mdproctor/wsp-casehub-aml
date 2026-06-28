# Design Journal — issue-58-separate-sar-drafting

## §1 SAR-drafting / compliance-review-opening split

**Decision:** Separate `openReview()` side effect from sar-drafting workers. Gate SAR filing behind MLRO approval via `PlannedAction(SAR_FILING)`.

**Key constraint:** `PlannedAction` not supported in `FlowWorkerFunction` (engine#564). `FlowWorkerFunctionHandler` hardcodes `WorkerResult.of(model.asMap())` — single-argument overload, no PlannedAction. SAR-drafting workers must convert from Flow to `WorkerFunction.Sync`.

**Execution model after change:** 6 of 8 workers use FlowWorkerFunction; 2 (sar-drafting-agent-junior, sar-drafting-agent-senior) use WorkerFunction.Sync. New `compliance-review-opening-agent` is Flow (no PlannedAction needed — it runs post-gate).

**Sequencing:** sar-drafting → PlannedAction(SAR_FILING) → AmlActionRiskClassifier → GateRequired(ALWAYS) → MLRO gate WorkItem → approval → compliance-review-opening → openReview() → complianceTaskId → investigation-complete.

**Open issue:** `AmlLayer7ResourceTest` and `AmlTrustRoutingAttestationTest` fail with `CaseInstance not found or wrong tenant` specifically with #58's larger case definition. Root cause partially identified (implicit `@Alternative` activation — GE-20260531-1e51d4 variant). Explicit `selected-alternatives` fixed 4 of 6 failing classes; 2 remain under investigation.
