# casehub-aml Design

## Architecture

### Coordinator + Specialist Composer Pattern

Layer 2 introduced `WorkItemAmlInvestigationService` wrapping `NaiveAmlInvestigationService`. The concrete injection (by type, not interface) made CDI `@DefaultBean` displacement impossible — `QhorusAmlInvestigator` could not substitute. Layer 3 replaced this chain with a composer.

**Key decomposition:**

- `AmlInvestigator` — inner interface (investigation concern only; swappable via CDI `@DefaultBean`)
- `ComplianceReviewLifecycle` — WorkItem concern (stable from Layer 2 onwards; unchanged as specialists evolve)
- `AmlInvestigationCoordinator` — outer coordinator composing both; stable through Layers 3–4; replaced by engine in Layer 5

This separation means the specialist implementation (`NaiveAmlInvestigationService` → `QhorusAmlInvestigator` → engine-driven in Layer 5) can evolve without touching the compliance lifecycle.

### SpecialistOutcome sealed interface

`SpecialistOutcome<T>` with `Completed<T>`, `Declined`, and `Failed` replaces concrete result types throughout `InvestigationSummary` and `SarDraftingService`. JSON serialisation uses a Jackson mixin in `app/` that adds a `"type"` discriminator without importing Jackson annotations into `api/`.

### Agent dispatch (Layer 3)

`QhorusAmlInvestigator` sends typed COMMAND messages and receives DONE/DECLINE responses via qhorus. Dispatch is currently in-process — `QhorusAmlInvestigator` calls `AgentBehaviour.handle()` after sending COMMAND. `channelGateway.fanOut()` did not trigger `PushAgentDispatch.post()` in testing; root cause tracked in issue #22. COMMAND/DONE/DECLINE messages are persisted to the qhorus DB regardless.

**SPIs:**
- `AgentBehaviour` — one implementation per specialist (entity resolution, pattern analysis, OSINT screening); `OsintScreeningBehaviour` always DECLINEs (clearance gate)
- `AgentDispatchMechanism` — `PushAgentDispatch` (qhorus channel) vs `NoOpAgentDispatch` (test double)
