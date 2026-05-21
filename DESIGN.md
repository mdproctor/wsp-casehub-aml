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

### Flyway locations must be pinned explicitly

`quarkus.flyway.locations` is always set explicitly in both `application.properties` files (test and main). Flyway's classpath scan is recursive — relying on the default (`classpath:db/migration`) means any future dependency jar adding a `db/migration/` entry is silently picked up, potentially causing V-number conflicts or unexpected schema changes.

**AML datasource configuration:**
- Default datasource: `classpath:db/migration` (casehub-work migrations V1–V27 only)
- Qhorus datasource: `classpath:db/qhorus/migration,classpath:db/ledger/migration` — the qhorus PU manages both qhorus entities and ledger entities (`casehub.ledger.datasource=qhorus`), so both migration paths are required. This overrides the qhorus extension default (`db/qhorus/migration` only).

This pattern is captured in the `quarkus-test-database.md` universal protocol as a rule for all CaseHub domain applications.
