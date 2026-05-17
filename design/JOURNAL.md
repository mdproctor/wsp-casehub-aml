# Design Journal — epic-layer3-qhorus

## 2026-05-17 — Composer pattern replaces WorkItemAmlInvestigationService §Architecture

The key architectural decision this epic: replace the Layer 2 chain (WorkItemAmlInvestigationService wrapping NaiveAmlInvestigationService) with a composer.

**Why the change was needed:** `WorkItemAmlInvestigationService` injected `NaiveAmlInvestigationService` by concrete type. CDI `@DefaultBean` displacement works at the interface level — injecting by concrete type prevented `QhorusAmlInvestigator` from substituting. The Layer 2 commit promised "Changing specialist implementations in Layer 3 will propagate automatically" — the concrete injection made this impossible.

**What the composer adds:**
- `AmlInvestigator` — inner interface (investigation concern, swappable via CDI)
- `ComplianceReviewLifecycle` — WorkItem concern (stable from Layer 2 onwards)
- `AmlInvestigationCoordinator` — outer coordinator composing both; stable through Layers 3–4; replaced by engine in Layer 5

**SpecialistOutcome<T>** sealed interface with Completed/Declined/Failed replaces concrete result types throughout `InvestigationSummary` and `SarDraftingService`. Jackson mixin in app/ adds type discriminator without polluting api/ with framework dependencies.

**Dispatch approach:** Direct in-process — `QhorusAmlInvestigator` calls `AgentBehaviour.handle()` after sending the COMMAND message. `channelGateway.fanOut()` did not trigger `PushAgentDispatch.post()` (root cause tracked in #22). COMMAND/DONE/DECLINE messages are persisted to qhorus DB regardless.

**`casehub.qhorus.reactive.enabled` removed** — upstream bug casehubio/qhorus#141 resolved in current version.
