# Design Journal — issue-63-xs-xm-batch-fixes

### 2026-06-21 · §9.4·Layer 5

`AmlInvestigationCaseHub` was refactored from a monolithic CDI bean carrying all seven worker lambdas to a thin CDI wrapper delegating to `AmlInvestigationCaseDescriptor` — a plain POJO testable without Quarkus. This follows protocol PP-20260518 (revised 2026-05-30), which superseded the `*CaseDefinitions` DSL companion pattern in favour of `*CaseDescriptor` POJOs that carry all business logic independently of CDI. The double-checked locking in `getDefinition()` was replaced with `@PostConstruct init()`, which guarantees single-thread execution before the bean is observable. Worker lambdas capture their dependencies but do not invoke them at construction, so the descriptor can be verified structurally with `null` deps and no Quarkus container.

### 2026-06-21 · §9.4·Layer 6

`AmlLayer6Resource.getInvestigation()` now checks `CaseStatus.COMPLETED` from `CaseInstanceCache` rather than scanning `WorkerDecisionEntry` for a `sar-drafting` entry. The previous approach had an async delivery race: `WorkerDecisionEntry` is written by an `@ObservesAsync` observer, which can lag behind the engine's COMPLETED state under H2 concurrency, causing the drain endpoint to report "in-progress" on a case that had already completed. Switching to `CaseInstanceCache` eliminates the race and aligns Layer 6 with the Layer 9 completion detection pattern established in the #63 fix. The `WorkerDecisionEntry` query is now deferred to the completed branch only, avoiding a spurious DB query on every in-progress poll.
