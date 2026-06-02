# 0002 — Case definition construction: YAML bindings with programmatic worker functions

Date: 2026-05-25
Status: Accepted

## Context and Problem Statement

Layer 5 adds casehub-engine to the AML investigation. The engine requires a `CaseDefinition` that declares capabilities, bindings (with JQ conditions), goals, and worker functions. Worker functions must be Java lambdas because they capture CDI proxies and execute domain logic. The YAML DSL cannot express Java lambdas.

## Decision Drivers

* Bindings and JQ conditions are declarative — they should be readable as data, not code
* Worker functions must capture injected CDI beans (ComplianceReviewLifecycle, ObjectMapper)
* Tutorial readers should see the binding structure without parsing Java builder chains
* The engine's `CaseDefinitionYamlMapper` is the authoritative path for YAML → `CaseDefinition`

## Considered Options

* **Option A** — YAML file for all structure; augment `getDefinition()` with programmatic workers
* **Option B** — Fully fluent Java DSL (no YAML); build the entire definition in Java
* **Option C** — YAML file only, with workers declared as agent references resolved via `WorkerProvisioner`

## Decision Outcome

Chosen option: **Option A**, because YAML keeps the binding topology visible and readable (the teaching artefact of Layer 5), while Java augmentation supplies the lambda functions that YAML cannot express.

### Positive Consequences

* Binding conditions, capabilities, goals, and completion are visible as configuration — not buried in Java builder chains
* Worker lambda functions can capture injected CDI beans without any indirection
* `AmlInvestigationCaseHub.getDefinition()` is the single assembly point; augmentation is called exactly once via double-checked locking inside `synchronized(this)`
* No new SPI implementation required — `WorkerProvisioner` no-op default is sufficient

### Negative Consequences / Tradeoffs

* `augment()` mutates the YAML-loaded `CaseDefinition` via `getWorkers().addAll()`. The parent's internal list is modified in place; safety depends on the `synchronized` block in `getDefinition()` being the only entry point
* Two artefacts (YAML + Java) must stay consistent — capability names in the YAML capabilities section must match the `Capability` names used in worker builders
* Workers added after YAML load are not visible to `CaseDefinitionYamlMapper` — round-tripping the definition to YAML would lose them

## Pros and Cons of the Options

### Option A — YAML bindings + programmatic workers

* ✅ Bindings and JQ conditions readable as configuration
* ✅ Worker lambdas can capture CDI proxies without ceremony
* ✅ Single assembly point — `getDefinition()` — with clear synchronization
* ❌ Two-artefact consistency requirement (YAML capabilities ↔ Java worker capability names)
* ❌ `augment()` mutates the shared YAML-loaded definition in place

### Option B — Fully fluent Java DSL

* ✅ Single artefact — no YAML/Java split
* ✅ Worker functions naturally co-located with binding declarations
* ❌ Binding topology invisible to readers unfamiliar with the engine builder API
* ❌ Verbose: 5 capabilities × 3–4 builder calls each is several hundred lines
* ❌ Breaks the tutorial pattern — every other domain app uses YAML

### Option C — YAML only with WorkerProvisioner

* ✅ No programmatic augmentation — all structure is declarative
* ❌ `WorkerProvisioner` spins up external compute — wrong abstraction for in-process tutorial stubs
* ❌ Worker function results must be sent back via the agent mesh (Qhorus), adding qhorus dispatch overhead that obscures the engine teaching point
* ❌ Requires implementing a non-trivial SPI contract for what is functionally a stub

## Links

* casehubio/aml#31 — Layer 5 implementation
* `docs/specs/2026-05-24-layer5-engine-design.md` — design spec
