# LLM Supervisor Mode — Design Spec

**Issue:** casehubio/aml#8
**Date:** 2026-08-14
**Status:** Draft

## Summary

Add an LLM-based `PlanningStrategy` implementation (`AmlInvestigationSupervisor`) that dynamically selects which investigation bindings to fire based on accumulated case context. The LLM controls investigation flow — which specialist to invoke, in what order, whether to skip or add steps, when evidence is sufficient for early termination. The deterministic `InvestigationTriageEvaluator` remains the quality gate for SAR/FALSE_POSITIVE/INCONCLUSIVE decisions.

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| LLM scope (D1) | Procedural binding selection only | Preserves accountability boundary — LLM controls investigation flow, deterministic evaluator controls regulatory judgment |
| LLM integration (D2) | `ChatModelProvider` from `casehub-engine-ai` | Engine's native LLM layer; direct Anthropic SDK; no subprocess overhead |
| Operating mode (D3) | Filter — LLM selects subset of already-eligible bindings | JQ conditions encode domain invariants; LLM adds within-eligible decisions |
| Invocation frequency (D4) | Selective — only when genuine choices exist | Single-eligible-binding cycles pass through without LLM overhead |
| Fallback (D5) | Degrade gracefully with context marker | Investigation never stalls; `supervisorDegraded` marker for audit trail |
| Audit (D6) | Context field AND ledger entry | Procedural decisions with substantive consequences need tamper-evident audit |
| Placement (D7) | In `app/` module | Single CDI bean alongside other case-definition beans |

## Architecture

### Extension Point

The engine's `PlanningStrategy` SPI (in `casehub-engine-planning`) is the integration point. No engine changes required.

```
PlanningStrategy.select(CasePlanModel, PlanExecutionContext, List<Binding>) → List<Binding>
```

`CompoundStrategyDispatcher` discovers all `PlanningStrategy` CDI beans via `Instance<PlanningStrategy>`, builds a `Map<String, PlanningStrategy>` keyed by `id()`. The YAML case definition's `planningStrategy:` field selects which strategy to use.

### Component Overview

```
┌──────────────────────────────────────────────────────┐
│        AmlInvestigationSupervisor                    │
│      implements PlanningStrategy (pure — no I/O)     │
│           id() = "aml-supervisor"                    │
├──────────────────────────────────────────────────────┤
│  1. Delegate to AmlSupervisorLlmAdapter              │
│  2. Validate response (binding names in eligible)    │
│  3. Record decision in pending store (in-memory)     │
│  4. Return selected bindings                         │
│  5. On failure → return all eligible (fallback)      │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│        AmlSupervisorLlmAdapter                       │
│      (prompt building, LLM call, JSON parsing)       │
├──────────────────────────────────────────────────────┤
│  - buildPrompt(plan, context, eligible)              │
│  - callLlm(prompt) → SupervisorDecision              │
│  - extractJsonBlock(response)                        │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│        AmlSupervisorAuditObserver                    │
│      @ObservesAsync WorkerDecisionEvent              │
├──────────────────────────────────────────────────────┤
│  - Reads pending decisions from supervisor store     │
│  - Writes AmlSupervisorDecisionLedgerEntry           │
│  - Runs outside the planning cycle transaction       │
└──────────────────────────────────────────────────────┘
```

### YAML Configuration — Compound-Scoped Strategy

The supervisor is scoped to a YAML compound containing only the bindings where genuine decisions exist. Free-floating bindings (entity-resolution, cbr-path-advisor, failure-handling) remain under `ChoreographyStrategy`. This moves scope policy into the case definition (where domain policy lives) rather than burying it in Java gate logic.

```yaml
spec:
  compounds:
    - name: supervised-investigation
      planningStrategy: aml-supervisor
      members:
        - pattern-analysis
        - osint-screening
        - investigation-triage
        - sar-drafting
        - senior-analyst-required-resolution

  capabilities:
    # ... unchanged
```

When a binding is inside the compound, `CompoundStrategyDispatcher` routes it to the `aml-supervisor` strategy. Bindings outside the compound (entity-resolution, cbr-path-advisor, compliance-review-opening, failure-handling) use the default `ChoreographyStrategy`. No Java invocation gate needed.

## Domain Model (app/)

### SupervisorDecision — structured output from LLM

```java
public record SupervisorDecision(
    List<String> selectedBindings,
    List<String> suppressedBindings,
    String rationale,
    boolean earlyTermination
) {
    public SupervisorDecision {
        Objects.requireNonNull(selectedBindings);
        Objects.requireNonNull(suppressedBindings);
        Objects.requireNonNull(rationale);
        if (selectedBindings.isEmpty()) {
            throw new IllegalArgumentException(
                "selectedBindings must not be empty — the LLM must always select at least one binding");
        }
    }
}
```

`earlyTermination` is **audit metadata, not a control signal**. The LLM expresses early termination by selecting triage (or a subset of specialists) and suppressing others — the control is which bindings are selected. Returning empty would stall the case (no context change → same bindings eligible → hot loop). The `earlyTermination` flag records the LLM's intent for compliance audit.
```

### AmlSupervisorDecisionLedgerEntry

```java
@Entity
@Table(name = "aml_supervisor_decision_ledger_entry")
@DiscriminatorValue("AML_SUPERVISOR_DECISION")
public class AmlSupervisorDecisionLedgerEntry extends JpaLedgerEntry {

    @Column(name = "selected_bindings", nullable = false, length = 500)
    public String selectedBindings;

    @Column(name = "suppressed_bindings", length = 500)
    public String suppressedBindings;

    @Column(name = "rationale", nullable = false, length = 2000)
    public String rationale;

    @Column(name = "early_termination", nullable = false)
    public boolean earlyTermination;

    @Column(name = "eligible_count", nullable = false)
    public int eligibleCount;

    @Column(name = "degraded", nullable = false)
    public boolean degraded;

    @Override
    protected byte[] domainContentBytes() {
        return String.join("|",
            selectedBindings,
            suppressedBindings != null ? suppressedBindings : "",
            rationale,
            String.valueOf(earlyTermination),
            String.valueOf(eligibleCount),
            String.valueOf(degraded)
        ).getBytes(java.nio.charset.StandardCharsets.UTF_8);
    }
}
```

### Flyway Migration — V3012

```sql
CREATE TABLE aml_supervisor_decision_ledger_entry (
    id UUID NOT NULL,
    selected_bindings VARCHAR(500) NOT NULL,
    suppressed_bindings VARCHAR(500),
    rationale VARCHAR(2000) NOT NULL,
    early_termination BOOLEAN NOT NULL,
    eligible_count INT NOT NULL,
    degraded BOOLEAN NOT NULL,
    CONSTRAINT fk_supervisor_decision_ledger_entry FOREIGN KEY (id) REFERENCES ledger_entry(id)
);
```

## AmlInvestigationSupervisor — Implementation

### Class Decomposition

Three classes with distinct responsibilities:

**`AmlInvestigationSupervisor`** — pure `PlanningStrategy`. No I/O, no side effects in `select()`. Delegates to the LLM adapter, validates the response, records the decision in a pending store, returns bindings.

```java
@ApplicationScoped
@io.quarkus.arc.Unremovable
public class AmlInvestigationSupervisor implements PlanningStrategy {

    private final AmlSupervisorLlmAdapter llmAdapter;
    private final AmlSupervisorPendingStore pendingStore;

    @Inject
    public AmlInvestigationSupervisor(
            AmlSupervisorLlmAdapter llmAdapter,
            AmlSupervisorPendingStore pendingStore) {
        this.llmAdapter = llmAdapter;
        this.pendingStore = pendingStore;
    }

    @Override
    public String id() { return "aml-supervisor"; }

    @Override
    public String getName() { return "AML Investigation LLM Supervisor"; }
}
```

**`AmlSupervisorLlmAdapter`** — owns prompt building, LLM call, JSON parsing. Injected into the strategy. Testable independently with mock `ChatModel`.

```java
@ApplicationScoped
public class AmlSupervisorLlmAdapter {

    private final ChatModelProvider chatModelProvider;
    private final ObjectMapper objectMapper;

    @Inject
    public AmlSupervisorLlmAdapter(
            Instance<ChatModelProvider> chatModelProviders,
            ObjectMapper objectMapper) {
        this.chatModelProvider = chatModelProviders.stream()
            .filter(p -> p.type() == ModelType.ANTHROPIC)
            .findFirst()
            .orElse(null);
        this.objectMapper = objectMapper;
    }

    public boolean isAvailable() { return chatModelProvider != null; }

    public SupervisorDecision consult(
            CasePlanModel plan, PlanExecutionContext ctx, List<Binding> eligible) {
        String prompt = buildPrompt(plan, ctx, eligible);
        return callLlm(prompt);
    }
}
```

**`AmlSupervisorPendingStore`** — `@ApplicationScoped` in-memory store for decisions pending ledger write. Thread-safe `ConcurrentHashMap<UUID, SupervisorDecision>` keyed by `caseId`.

**`AmlSupervisorAuditObserver`** — `@ObservesAsync WorkerDecisionEvent`. Reads pending decisions from the store and writes `AmlSupervisorDecisionLedgerEntry` entries. Runs outside the planning cycle transaction, avoiding the dual-datasource `@Transactional` anti-pattern.

### SPI Contract — `select()` is Pure

`select()` performs no I/O:
1. Check if LLM adapter is available — if not, return `eligible` unchanged
2. Call `llmAdapter.consult()` — this is the LLM call (blocking, acceptable with virtual threads)
3. Validate response — reject hallucinated binding names
4. Store decision in `pendingStore` (in-memory only)
5. Return selected bindings

On any failure in steps 2-3, fall back to returning `eligible` unchanged and store a degraded decision in the pending store.

The only side effect is the in-memory pending store write, which is a `ConcurrentHashMap.put()` — no transaction, no I/O, no risk of re-entrant planning cycles.

**Note on LLM call in `select()`:** The LLM call itself (step 2) is I/O, which makes `select()` not strictly pure. However, the `PlanningStrategy` SPI is synchronous and the call is essential to the strategy's function. The distinction is: the LLM call is the strategy's *computation* (deciding which bindings to select), not a *side effect* (writing state that affects other components). Audit and context writes — which ARE side effects — are deferred to the observer.

### Scope via YAML Compound (replaces Java invocation gate)

No Java gate logic. The compound definition in the YAML scopes the supervisor to exactly the bindings where decisions matter. `CompoundStrategyDispatcher` handles the routing — the supervisor's `select()` is only called with bindings from the `supervised-investigation` compound.

### Context Projection

The LLM receives a structured projection of the case context, not the raw JSON. This controls token usage and focuses the LLM on decision-relevant information.

```java
String buildPrompt(CasePlanModel plan, PlanExecutionContext ctx, List<Binding> eligible) {
    var caseContext = ctx.caseContext();
    return """
        You are an AML investigation supervisor. Your role is to decide which
        investigation steps should fire next based on accumulated evidence.

        ## Case Context
        %s

        ## Investigation Progress
        %s

        ## Eligible Bindings (you may select a subset)
        %s

        ## Instructions
        - Select which bindings to fire NOW. You may suppress bindings that
          are not yet useful given the current evidence.
        - If evidence is sufficient for triage without all specialist checks
          completing, you may allow triage to proceed (early termination).
        - Hard regulatory gates (sanctions, PEP, shell company) are handled
          by the deterministic evaluator — do not reason about the triage
          outcome itself.
        - Respond with JSON matching the schema below.

        ## Response Schema
        {"selectedBindings": ["name1", "name2"],
         "suppressedBindings": ["name3"],
         "rationale": "brief explanation",
         "earlyTermination": false}
        """.formatted(
            projectCaseContext(caseContext),
            projectPlanProgress(plan),
            projectEligibleBindings(eligible, ctx.definition())
        );
}
```

**Context projection fields:**
- `entityResolution` — entity type, risk score, ownership chain summary
- `patternAnalysis` — structuring detected, description
- `osintScreening` — sanctions hit, PEP hit, declined status
- `cbrPathAdvice` — predominant outcome, confidence, recommended capabilities
- `investigationTriage` — if present, the triage decision and score
- `priorEntityContext` — known high-risk flag

**Plan progress projection:**
- Completed bindings and their outcomes
- Currently running bindings
- Failed/retried bindings

**Eligible binding projection:**
- Binding name and capability description (from YAML `description` field)
- Input projection summary (what evidence the specialist will receive)

### Step 3 — LLM Call

```java
SupervisorDecision callLlm(String prompt) {
    ChatModel chatModel = chatModelProvider.get();
    var response = chatModel.chat(prompt);
    return objectMapper.readValue(
        extractJsonBlock(response.aiMessage().text()),
        SupervisorDecision.class);
}
```

Uses LangChain4j `ChatModel.chat()` with structured JSON output. The `extractJsonBlock()` utility finds the JSON block in the response (handles markdown code fences).

### Step 4 — Response Validation

```java
List<Binding> validateAndResolve(SupervisorDecision decision, List<Binding> eligible) {
    Set<String> eligibleNames = eligible.stream()
        .map(Binding::getName)
        .collect(Collectors.toSet());

    // Reject hallucinated binding names
    for (String name : decision.selectedBindings()) {
        if (!eligibleNames.contains(name)) {
            throw new InvalidSupervisorResponseException(
                "LLM selected non-eligible binding: " + name);
        }
    }

    return eligible.stream()
        .filter(b -> decision.selectedBindings().contains(b.getName()))
        .toList();
}
```

Any hallucinated binding name (not in the eligible set) triggers fallback (D5). This is the critical safety check — the LLM cannot add bindings that JQ conditions didn't authorize.

### Audit — Deferred to Observer

`select()` writes decisions to `AmlSupervisorPendingStore` (in-memory `ConcurrentHashMap`). `AmlSupervisorAuditObserver` flushes pending decisions to the ledger:

```java
@ApplicationScoped
public class AmlSupervisorAuditObserver {

    @Inject AmlSupervisorPendingStore pendingStore;
    @Inject LedgerEntryRepository ledgerEntryRepository;

    void onWorkerDecision(@ObservesAsync WorkerDecisionEvent event) {
        UUID caseId = event.caseId();
        SupervisorDecision decision = pendingStore.take(caseId);
        if (decision == null) return;

        var entry = new AmlSupervisorDecisionLedgerEntry();
        entry.selectedBindings = String.join(",", decision.selectedBindings());
        entry.suppressedBindings = String.join(",", decision.suppressedBindings());
        entry.rationale = decision.rationale();
        entry.earlyTermination = decision.earlyTermination();
        entry.eligibleCount = decision.eligibleCount();
        entry.degraded = decision.degraded();
        entry.setSubjectId(UUID.nameUUIDFromBytes(
            ("aml-supervisor:" + caseId).getBytes(StandardCharsets.UTF_8)));
        entry.setTenancyId(event.tenancyId());
        entry.setActorId("aml-supervisor");
        entry.setActorType(ActorType.SYSTEM);
        ledgerEntryRepository.save(entry, event.tenancyId());
    }
}
```

Subject ID uses the `"aml-supervisor:" + caseId` pattern per the ledger subject isolation protocol.

This runs outside the planning cycle transaction — no dual-datasource risk. The observer follows the same pattern as `AmlTrustRoutingObserver` writing attestations on `WorkerDecisionEvent`.

### Fallback

On any failure in the LLM adapter (unavailable, timeout, unparseable response) or validation (hallucinated bindings):

1. Record a degraded decision in the pending store (all eligible selected, `degraded: true`)
2. Return `eligible` unchanged (fire all — ChoreographyStrategy behavior)

Degradation is **per-call, not sticky**. Each planning cycle independently attempts the LLM. If the LLM recovers mid-investigation, subsequent cycles use it.

### CBR Interaction

The YAML case definition already uses `cbrPathAdvice` in binding conditions (e.g., `senior-analyst-review` fires when `cbrPathAdvice.capabilities["senior-analyst-review"].frequency > 0.6`). The LLM supervisor also receives CBR advice in its context projection.

These are **complementary, not competing** channels:
- JQ conditions use CBR for eligibility (can this binding fire?)
- LLM uses CBR for selection (should this binding fire now?)

The LLM sees the same CBR data that JQ conditions evaluate, but reasons about it holistically — e.g., "CBR says 80% of similar cases filed SARs, and we already have strong entity resolution evidence — proceed to triage without waiting for pattern analysis."

No special deconfliction logic is needed. The JQ conditions gate eligibility; the LLM filters within eligibility. CBR informs both independently.

## Testing Strategy

### Unit tests (app/ test) — pure domain, no Quarkus

**`SupervisorDecisionTest`:**
- Valid construction with selected and suppressed bindings
- Empty selectedBindings always throws (earlyTermination is metadata, not a control signal)
- Null fields throw

**`AmlInvestigationSupervisorTest`:**
- LLM adapter available → delegates and returns validated subset
- Validate: hallucinated binding name → fallback (all eligible returned)
- Validate: valid subset → correct bindings returned
- Fallback: LLM adapter throws → all eligible returned + degraded decision in pending store
- Fallback: LLM adapter unavailable → permanent pass-through, no pending store write
- Pending store: decision recorded on successful LLM call
- Pending store: degraded decision recorded on failure

**`AmlSupervisorLlmAdapterTest`:**
- Prompt contains case context projection, plan progress, eligible binding descriptions
- Correct fields included for each specialist finding state (entity resolution present, pattern analysis absent, etc.)
- JSON response parsed to SupervisorDecision
- Malformed JSON throws
- Missing JSON block in response throws

**`AmlSupervisorDecisionLedgerEntryTest`:**
- `domainContentBytes()` produces correct pipe-delimited output
- All fields serialized

### @QuarkusTest integration tests (app/)

**`SupervisorModeInvestigationTest`:**
- Full investigation with mock `ChatModelProvider` returning structured decisions
- Verify LLM is consulted only for compound-scoped bindings (not entity-resolution, cbr-path-advisor, failure-handling)
- Verify fallback: mock ChatModelProvider.get() throws → investigation completes via choreography
- Verify ledger entries: `AmlSupervisorDecisionLedgerEntry` written by audit observer after worker dispatch
- Verify early termination: LLM selects triage + suppresses remaining specialists → triage fires with partial evidence → deterministic triage evaluates correctly with available findings
- Drain to `status=completed` per protocol PP-20260604-820c35

**`SupervisorDegradedInvestigationTest`:**
- Full investigation with always-failing ChatModelProvider
- Verify all cycles degrade → investigation completes identically to non-supervisor mode
- Verify degraded ledger entries written by audit observer

## File Inventory

### New files

| File | Module | Description |
|------|--------|-------------|
| `SupervisorDecision.java` | app | Structured LLM output record |
| `AmlInvestigationSupervisor.java` | app | Pure `PlanningStrategy` — delegates, validates, returns |
| `AmlSupervisorLlmAdapter.java` | app | Prompt building, LLM call, JSON parsing |
| `AmlSupervisorPendingStore.java` | app | In-memory store for decisions pending ledger write |
| `AmlSupervisorAuditObserver.java` | app | `@ObservesAsync WorkerDecisionEvent` — writes ledger entries |
| `AmlSupervisorDecisionLedgerEntry.java` | app | Tamper-evident audit ledger entry |
| `InvalidSupervisorResponseException.java` | app | Thrown on hallucinated binding names |
| `V3012__supervisor_decision_ledger_entry.sql` | app (migration) | Join table for ledger subclass |
| `SupervisorDecisionTest.java` | app (test) | Unit tests for record |
| `AmlInvestigationSupervisorTest.java` | app (test) | Unit tests for strategy |
| `AmlSupervisorLlmAdapterTest.java` | app (test) | Unit tests for prompt + parsing |
| `AmlSupervisorDecisionLedgerEntryTest.java` | app (test) | Unit tests for ledger entry |
| `SupervisorModeInvestigationTest.java` | app (test) | Integration test — LLM-guided investigation |
| `SupervisorDegradedInvestigationTest.java` | app (test) | Integration test — degraded mode |

### Modified files

| File | Change |
|------|--------|
| `aml-investigation.yaml` | Add `planningStrategy: aml-supervisor` |
| `app/pom.xml` | Add `casehub-engine-ai` dependency |
| `application.properties` (test) | Add CDI exclusions for engine-ai beans if needed |

### Dependencies added

| Artifact | Purpose |
|----------|---------|
| `casehub-engine-ai` | `ChatModelProvider` SPI + `AnthropicChatModelProvider` |
