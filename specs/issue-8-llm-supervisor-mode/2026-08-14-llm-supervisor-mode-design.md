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
┌─────────────────────────────────────────────┐
│           AmlInvestigationSupervisor        │
│         implements PlanningStrategy         │
│              id() = "aml-supervisor"        │
├─────────────────────────────────────────────┤
│  1. Invocation gate — should LLM be called? │
│  2. Context projection — what does LLM see? │
│  3. LLM call — ChatModel.chat(prompt)       │
│  4. Response validation — binding names ok? │
│  5. Audit — context field + ledger entry    │
│  6. Fallback — degrade on any failure       │
└─────────────────────────────────────────────┘
```

### YAML Configuration

`aml-investigation.yaml` gains the `planningStrategy` field:

```yaml
spec:
  planningStrategy: aml-supervisor
  capabilities:
    # ... unchanged
```

When `planningStrategy` is absent or `"default"`, the `ChoreographyStrategy` applies (current behavior). Setting `"aml-supervisor"` activates the LLM supervisor for all investigations using this case definition.

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
        if (selectedBindings.isEmpty() && !earlyTermination) {
            throw new IllegalArgumentException(
                "Must select at least one binding unless earlyTermination is true");
        }
    }
}
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

### Class Structure

```java
@ApplicationScoped
@io.quarkus.arc.Unremovable
public class AmlInvestigationSupervisor implements PlanningStrategy {

    private final ChatModelProvider chatModelProvider;
    private final LedgerEntryRepository ledgerEntryRepository;
    private final ObjectMapper objectMapper;

    @Inject
    public AmlInvestigationSupervisor(
            Instance<ChatModelProvider> chatModelProviders,
            LedgerEntryRepository ledgerEntryRepository,
            ObjectMapper objectMapper) {
        this.chatModelProvider = chatModelProviders.stream()
            .filter(p -> p.type() == ModelType.ANTHROPIC)
            .findFirst()
            .orElse(null);
        this.ledgerEntryRepository = ledgerEntryRepository;
        this.objectMapper = objectMapper;
    }

    @Override
    public String id() { return "aml-supervisor"; }

    @Override
    public String getName() { return "AML Investigation LLM Supervisor"; }
}
```

### Step 1 — Invocation Gate

The LLM is consulted only at genuine decision points. The gate logic lives inside `select()`:

```java
boolean shouldConsultLlm(List<Binding> eligible, PlanExecutionContext ctx) {
    if (chatModelProvider == null) return false;
    if (eligible.isEmpty()) return false;
    if (hasTriageBinding(eligible)) return true;
    if (eligible.size() >= 2) return true;
    return false;
}
```

When the gate is not met, return `eligible` unchanged (pass-through, same as `ChoreographyStrategy`). No LLM call, no audit entry.

The gate fires when:
- **Multiple bindings are eligible** — a genuine sequencing/prioritization choice
- **Triage is among the eligible** — the decision point where early termination is most impactful (even as the only eligible binding — the LLM may judge that triage should wait for more evidence)

The gate does NOT fire when:
- `chatModelProvider` is null (no LLM configured — permanent pass-through)
- Only one non-triage binding is eligible (obvious choice, no decision to make)

### Step 2 — Context Projection

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

### Step 5 — Audit

On every LLM-consulted cycle, write both:

**Context field** — for downstream worker visibility:
```java
// Written via PlanExecutionContext — the supervisor returns bindings,
// and the context field is written by a companion observer
Map<String, Object> supervisorDecision = Map.of(
    "selected", decision.selectedBindings(),
    "suppressed", decision.suppressedBindings(),
    "rationale", decision.rationale(),
    "earlyTermination", decision.earlyTermination(),
    "degraded", false
);
```

**Ledger entry** — tamper-evident:
```java
var entry = new AmlSupervisorDecisionLedgerEntry();
entry.selectedBindings = String.join(",", decision.selectedBindings());
entry.suppressedBindings = String.join(",", decision.suppressedBindings());
entry.rationale = decision.rationale();
entry.earlyTermination = decision.earlyTermination();
entry.eligibleCount = eligible.size();
entry.degraded = false;
entry.setSubjectId(UUID.nameUUIDFromBytes(
    ("aml-supervisor:" + ctx.caseId()).getBytes(StandardCharsets.UTF_8)));
entry.setTenancyId(ctx.tenancyId());
entry.setActorId("aml-supervisor");
entry.setActorType(ActorType.SYSTEM);
ledgerEntryRepository.save(entry, ctx.tenancyId());
```

Subject ID uses the `"aml-supervisor:" + caseId` pattern per the ledger subject isolation protocol.

### Step 6 — Fallback

On any failure in steps 3-4 (LLM unavailable, timeout, unparseable response, hallucinated bindings):

```java
List<Binding> degradedFallback(List<Binding> eligible, PlanExecutionContext ctx) {
    // Write degradation marker to context
    // (handled by writing to the context via the return value —
    //  the supervisor cannot write context directly from select(),
    //  so the degradation marker is written as a ledger entry only)

    var entry = new AmlSupervisorDecisionLedgerEntry();
    entry.selectedBindings = eligible.stream()
        .map(Binding::getName).collect(Collectors.joining(","));
    entry.suppressedBindings = "";
    entry.rationale = "LLM unavailable — degraded to choreography";
    entry.earlyTermination = false;
    entry.eligibleCount = eligible.size();
    entry.degraded = true;
    // ... save as above

    return eligible;  // fire all — ChoreographyStrategy behavior
}
```

Degradation is **per-call, not sticky**. Each planning cycle independently attempts the LLM. If the LLM recovers mid-investigation, subsequent cycles use it. This avoids the complexity of degradation recovery logic while maximizing LLM utilization.

### Context Writing Constraint

`PlanningStrategy.select()` returns `List<Binding>` — it cannot write to the case context directly. The supervisor decision context field (`supervisorDecision`) must be written by a separate mechanism:

**Option A:** An `@ObservesAsync` observer on a CDI event fired by the supervisor before returning from `select()`. The observer writes the context field asynchronously.

**Option B:** A `BlackboardPlanConfigurer` that reads the ledger entry (written synchronously in `select()`) and projects it into the case context on the next planning cycle.

**Choice: Option A.** The supervisor fires a `SupervisorDecisionEvent` CDI event; an observer writes the context field. This keeps the supervisor's contract clean (returns bindings) while ensuring downstream workers see the decision rationale. The CDI event is fired in both normal and degraded paths — the observer writes the appropriate context field (`supervisorDecision` or `supervisorDegraded: true`) based on the event payload.

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
- Empty selectedBindings with earlyTermination=true succeeds
- Empty selectedBindings with earlyTermination=false throws
- Null fields throw

**`AmlInvestigationSupervisorTest`:**
- Gate: single eligible binding → pass-through, no LLM call
- Gate: multiple eligible → LLM called
- Gate: triage binding eligible as sole binding → LLM still called (early termination decision point)
- Validate: hallucinated binding name → fallback
- Validate: valid subset → correct bindings returned
- Fallback: LLM exception → all eligible returned + degraded ledger entry
- Fallback: null chatModelProvider → permanent pass-through
- Context projection: correct fields included for each specialist finding state

**`AmlSupervisorDecisionLedgerEntryTest`:**
- `domainContentBytes()` produces correct pipe-delimited output
- All fields serialized

### @QuarkusTest integration tests (app/)

**`SupervisorModeInvestigationTest`:**
- Full investigation with mock `ChatModelProvider` returning structured decisions
- Verify LLM is consulted at multi-binding decision points
- Verify LLM is NOT consulted for single-binding cycles
- Verify fallback: mock ChatModelProvider.get() throws → investigation completes via choreography
- Verify ledger entries: one `AmlSupervisorDecisionLedgerEntry` per LLM-consulted cycle
- Verify early termination: LLM suppresses remaining specialists → triage fires with partial evidence → deterministic triage evaluates correctly with available findings
- Drain to `status=completed` per protocol PP-20260604-820c35

**`SupervisorDegradedInvestigationTest`:**
- Full investigation with always-failing ChatModelProvider
- Verify all cycles degrade → investigation completes identically to non-supervisor mode
- Verify degraded ledger entries written
- Verify `supervisorDegraded` context marker present (via CDI observer)

## File Inventory

### New files

| File | Module | Description |
|------|--------|-------------|
| `SupervisorDecision.java` | app | Structured LLM output record |
| `AmlInvestigationSupervisor.java` | app | `PlanningStrategy` implementation |
| `AmlSupervisorDecisionLedgerEntry.java` | app | Tamper-evident audit ledger entry |
| `SupervisorDecisionEvent.java` | app | CDI event for context writing |
| `SupervisorDecisionContextObserver.java` | app | Writes supervisor decision to case context |
| `InvalidSupervisorResponseException.java` | app | Thrown on hallucinated binding names |
| `V3012__supervisor_decision_ledger_entry.sql` | app (migration) | Join table for ledger subclass |
| `SupervisorDecisionTest.java` | app (test) | Unit tests for record |
| `AmlInvestigationSupervisorTest.java` | app (test) | Unit tests for strategy |
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
