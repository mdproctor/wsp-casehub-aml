# LLM Supervisor Mode Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #8 — Epic 8: LLM supervisor mode — investigation triage
**Issue group:** #8

**Goal:** Add an LLM-based `PlanningStrategy` that dynamically selects which investigation bindings to fire based on accumulated case context, while the deterministic `InvestigationTriageEvaluator` remains the regulatory quality gate.

**Architecture:** `AmlInvestigationSupervisor` implements the engine's `PlanningStrategy` SPI (`id()` = `"aml-supervisor"`). A `BlackboardPlanConfigurer` registers a compound that scopes the supervisor to decision-relevant bindings (pattern-analysis, osint-screening, investigation-triage, sar-drafting, senior-analyst-required-resolution). The LLM is called via `ChatModelProvider` from `casehub-engine-ai`. Audit writes are deferred to an `@ObservesAsync WorkerDecisionEvent` observer, keeping `select()` side-effect-free.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine-planning (`PlanningStrategy` SPI), casehub-engine-ai (`ChatModelProvider`), casehub-ledger (`JpaLedgerEntry`), LangChain4j `ChatModel`

## Global Constraints

- `PlanningStrategy.select()` must remain side-effect-free — no ledger writes, no CDI events, no context mutations inside the method
- Compounds are registered via `BlackboardPlanConfigurer` Java code, not YAML — the engine schema has no `compounds:` field
- `SupervisorDecision.selectedBindings` must never be empty — returning empty stalls the case (no context change → hot loop)
- `earlyTermination` is audit metadata, not a control signal — the LLM expresses early termination by selecting triage and suppressing other specialists
- Flyway migration range: V3012+ (AML engine-ledger)
- `casehub-engine-ai` artifact: `io.casehub:casehub-engine-ai:0.2-SNAPSHOT`
- Ledger subject isolation: `UUID.nameUUIDFromBytes(("aml-supervisor:" + caseId).getBytes(UTF_8))`
- All `@QuarkusTest` must drain to `status=completed` per protocol PP-20260604-820c35

---

### Task 1: Domain model — SupervisorDecision record + pending store

**Files:**
- Create: `app/src/main/java/io/casehub/aml/supervisor/SupervisorDecision.java`
- Create: `app/src/main/java/io/casehub/aml/supervisor/PendingSupervisorDecision.java`
- Create: `app/src/main/java/io/casehub/aml/supervisor/InvalidSupervisorResponseException.java`
- Create: `app/src/main/java/io/casehub/aml/supervisor/AmlSupervisorPendingStore.java`
- Test: `app/src/test/java/io/casehub/aml/supervisor/SupervisorDecisionTest.java`
- Test: `app/src/test/java/io/casehub/aml/supervisor/AmlSupervisorPendingStoreTest.java`

**Interfaces:**
- Produces: `SupervisorDecision(List<String> selectedBindings, List<String> suppressedBindings, String rationale, boolean earlyTermination)` — used by Tasks 2-5
- Produces: `PendingSupervisorDecision(SupervisorDecision decision, int eligibleCount, boolean degraded, UUID caseId, String tenancyId)` — used by Tasks 4-5
- Produces: `AmlSupervisorPendingStore.put(PendingSupervisorDecision)`, `AmlSupervisorPendingStore.take(UUID caseId) → PendingSupervisorDecision` — used by Tasks 4-5
- Produces: `InvalidSupervisorResponseException(String message)` — used by Task 4

- [ ] **Step 1: Write failing tests for SupervisorDecision**

```java
package io.casehub.aml.supervisor;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class SupervisorDecisionTest {

    @Test
    void valid_construction() {
        var decision = new SupervisorDecision(
            List.of("pattern-analysis"), List.of("osint-screening"),
            "pattern first", false);
        assertThat(decision.selectedBindings()).containsExactly("pattern-analysis");
        assertThat(decision.suppressedBindings()).containsExactly("osint-screening");
        assertThat(decision.rationale()).isEqualTo("pattern first");
        assertThat(decision.earlyTermination()).isFalse();
    }

    @Test
    void empty_selected_throws() {
        assertThatThrownBy(() -> new SupervisorDecision(
            List.of(), List.of("osint-screening"), "no bindings", false))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void null_selected_throws() {
        assertThatThrownBy(() -> new SupervisorDecision(
            null, List.of(), "reason", false))
            .isInstanceOf(NullPointerException.class);
    }

    @Test
    void null_rationale_throws() {
        assertThatThrownBy(() -> new SupervisorDecision(
            List.of("a"), List.of(), null, false))
            .isInstanceOf(NullPointerException.class);
    }

    @Test
    void early_termination_with_selected_bindings_succeeds() {
        var decision = new SupervisorDecision(
            List.of("investigation-triage"), List.of("pattern-analysis"),
            "evidence sufficient", true);
        assertThat(decision.earlyTermination()).isTrue();
        assertThat(decision.selectedBindings()).containsExactly("investigation-triage");
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn test -pl app -am -Dtest=SupervisorDecisionTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Implement SupervisorDecision, PendingSupervisorDecision, InvalidSupervisorResponseException**

`SupervisorDecision.java`:
```java
package io.casehub.aml.supervisor;

import java.util.List;
import java.util.Objects;

public record SupervisorDecision(
    List<String> selectedBindings,
    List<String> suppressedBindings,
    String rationale,
    boolean earlyTermination
) {
    public SupervisorDecision {
        Objects.requireNonNull(selectedBindings, "selectedBindings required");
        Objects.requireNonNull(suppressedBindings, "suppressedBindings required");
        Objects.requireNonNull(rationale, "rationale required");
        if (selectedBindings.isEmpty()) {
            throw new IllegalArgumentException(
                "selectedBindings must not be empty");
        }
        selectedBindings = List.copyOf(selectedBindings);
        suppressedBindings = List.copyOf(suppressedBindings);
    }
}
```

`PendingSupervisorDecision.java`:
```java
package io.casehub.aml.supervisor;

import java.util.UUID;

public record PendingSupervisorDecision(
    SupervisorDecision decision,
    int eligibleCount,
    boolean degraded,
    UUID caseId,
    String tenancyId
) {}
```

`InvalidSupervisorResponseException.java`:
```java
package io.casehub.aml.supervisor;

public class InvalidSupervisorResponseException extends RuntimeException {
    public InvalidSupervisorResponseException(String message) {
        super(message);
    }
}
```

- [ ] **Step 4: Run SupervisorDecisionTest — verify pass**

Run: `mvn test -pl app -am -Dtest=SupervisorDecisionTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Write failing tests for AmlSupervisorPendingStore**

```java
package io.casehub.aml.supervisor;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;

class AmlSupervisorPendingStoreTest {

    private final AmlSupervisorPendingStore store = new AmlSupervisorPendingStore();

    @Test
    void put_and_take_returns_decision() {
        UUID caseId = UUID.randomUUID();
        var decision = new SupervisorDecision(
            List.of("pattern-analysis"), List.of(), "reason", false);
        var pending = new PendingSupervisorDecision(decision, 2, false, caseId, "t1");
        store.put(pending);
        assertThat(store.take(caseId)).isEqualTo(pending);
    }

    @Test
    void take_removes_entry() {
        UUID caseId = UUID.randomUUID();
        var decision = new SupervisorDecision(
            List.of("a"), List.of(), "r", false);
        store.put(new PendingSupervisorDecision(decision, 1, false, caseId, "t1"));
        store.take(caseId);
        assertThat(store.take(caseId)).isNull();
    }

    @Test
    void take_unknown_returns_null() {
        assertThat(store.take(UUID.randomUUID())).isNull();
    }

    @Test
    void put_overwrites_previous() {
        UUID caseId = UUID.randomUUID();
        var d1 = new PendingSupervisorDecision(
            new SupervisorDecision(List.of("a"), List.of(), "first", false),
            1, false, caseId, "t1");
        var d2 = new PendingSupervisorDecision(
            new SupervisorDecision(List.of("b"), List.of(), "second", false),
            2, false, caseId, "t1");
        store.put(d1);
        store.put(d2);
        assertThat(store.take(caseId).decision().rationale()).isEqualTo("second");
    }
}
```

- [ ] **Step 6: Implement AmlSupervisorPendingStore**

```java
package io.casehub.aml.supervisor;

import jakarta.enterprise.context.ApplicationScoped;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class AmlSupervisorPendingStore {

    private final ConcurrentHashMap<UUID, PendingSupervisorDecision> pending =
        new ConcurrentHashMap<>();

    public void put(PendingSupervisorDecision decision) {
        pending.put(decision.caseId(), decision);
    }

    public PendingSupervisorDecision take(UUID caseId) {
        return pending.remove(caseId);
    }
}
```

- [ ] **Step 7: Run AmlSupervisorPendingStoreTest — verify pass**

Run: `mvn test -pl app -am -Dtest=AmlSupervisorPendingStoreTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/aml/supervisor/ app/src/test/java/io/casehub/aml/supervisor/
git commit -m "feat(#8): supervisor domain model — SupervisorDecision record + pending store

Refs casehubio/aml#8"
```

---

### Task 2: Ledger entry + Flyway migration

**Files:**
- Create: `app/src/main/java/io/casehub/aml/ledger/AmlSupervisorDecisionLedgerEntry.java`
- Create: `app/src/main/resources/db/aml-engine-ledger/migration/V3012__supervisor_decision_ledger_entry.sql`
- Test: `app/src/test/java/io/casehub/aml/ledger/AmlSupervisorDecisionLedgerEntryTest.java`

**Interfaces:**
- Consumes: nothing (standalone JPA entity)
- Produces: `AmlSupervisorDecisionLedgerEntry` with fields: `selectedBindings`, `suppressedBindings`, `rationale`, `earlyTermination`, `eligibleCount`, `degraded` — used by Task 5

- [ ] **Step 1: Write failing test for domainContentBytes()**

```java
package io.casehub.aml.ledger;

import org.junit.jupiter.api.Test;
import java.nio.charset.StandardCharsets;
import static org.assertj.core.api.Assertions.*;

class AmlSupervisorDecisionLedgerEntryTest {

    @Test
    void domainContentBytes_all_fields() {
        var entry = new AmlSupervisorDecisionLedgerEntry();
        entry.selectedBindings = "pattern-analysis,osint-screening";
        entry.suppressedBindings = "sar-drafting";
        entry.rationale = "parallel first";
        entry.earlyTermination = false;
        entry.eligibleCount = 3;
        entry.degraded = false;

        String content = new String(entry.domainContentBytes(), StandardCharsets.UTF_8);
        assertThat(content).isEqualTo(
            "pattern-analysis,osint-screening|sar-drafting|parallel first|false|3|false");
    }

    @Test
    void domainContentBytes_null_suppressed() {
        var entry = new AmlSupervisorDecisionLedgerEntry();
        entry.selectedBindings = "investigation-triage";
        entry.suppressedBindings = null;
        entry.rationale = "early termination";
        entry.earlyTermination = true;
        entry.eligibleCount = 2;
        entry.degraded = false;

        String content = new String(entry.domainContentBytes(), StandardCharsets.UTF_8);
        assertThat(content).isEqualTo(
            "investigation-triage||early termination|true|2|false");
    }

    @Test
    void domainContentBytes_degraded() {
        var entry = new AmlSupervisorDecisionLedgerEntry();
        entry.selectedBindings = "pattern-analysis,osint-screening";
        entry.suppressedBindings = "";
        entry.rationale = "LLM unavailable — degraded to choreography";
        entry.earlyTermination = false;
        entry.eligibleCount = 2;
        entry.degraded = true;

        String content = new String(entry.domainContentBytes(), StandardCharsets.UTF_8);
        assertThat(content).contains("|true");
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn test -pl app -am -Dtest=AmlSupervisorDecisionLedgerEntryTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Implement AmlSupervisorDecisionLedgerEntry**

```java
package io.casehub.aml.ledger;

import io.casehub.ledger.runtime.model.jpa.JpaLedgerEntry;
import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;
import java.nio.charset.StandardCharsets;

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
        ).getBytes(StandardCharsets.UTF_8);
    }
}
```

- [ ] **Step 4: Create Flyway migration V3012**

`app/src/main/resources/db/aml-engine-ledger/migration/V3012__supervisor_decision_ledger_entry.sql`:
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

- [ ] **Step 5: Run test — verify pass**

Run: `mvn test -pl app -am -Dtest=AmlSupervisorDecisionLedgerEntryTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/aml/ledger/AmlSupervisorDecisionLedgerEntry.java app/src/main/resources/db/aml-engine-ledger/migration/V3012__supervisor_decision_ledger_entry.sql app/src/test/java/io/casehub/aml/ledger/
git commit -m "feat(#8): supervisor decision ledger entry + V3012 migration

Refs casehubio/aml#8"
```

---

### Task 3: LLM adapter — prompt building, LLM call, JSON parsing

**Files:**
- Modify: `app/pom.xml` — add `casehub-engine-ai` dependency
- Create: `app/src/main/java/io/casehub/aml/supervisor/AmlSupervisorLlmAdapter.java`
- Test: `app/src/test/java/io/casehub/aml/supervisor/AmlSupervisorLlmAdapterTest.java`

**Interfaces:**
- Consumes: `SupervisorDecision` (from Task 1), `ChatModelProvider` (from `casehub-engine-ai`), `CasePlanModel` + `PlanExecutionContext` + `Binding` (from engine-planning)
- Produces: `AmlSupervisorLlmAdapter.isAvailable() → boolean`, `AmlSupervisorLlmAdapter.consult(CasePlanModel, PlanExecutionContext, List<Binding>) → SupervisorDecision` — used by Task 4

- [ ] **Step 1: Add casehub-engine-ai dependency to app/pom.xml**

Add to `app/pom.xml` `<dependencies>`:
```xml
      <artifactId>casehub-engine-ai</artifactId>
```
(groupId and version inherited from parent BOM)

Run: `mvn compile -pl app -am` to verify dependency resolves.

- [ ] **Step 2: Write failing tests for AmlSupervisorLlmAdapter**

```java
package io.casehub.aml.supervisor;

import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.AiMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.response.ChatResponse;
import io.casehub.api.model.ai.ChatModelProvider;
import io.casehub.api.model.ai.ModelType;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class AmlSupervisorLlmAdapterTest {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Test
    void isAvailable_true_when_provider_present() {
        var adapter = createAdapter(mockProvider());
        assertThat(adapter.isAvailable()).isTrue();
    }

    @Test
    void isAvailable_false_when_no_provider() {
        var adapter = new AmlSupervisorLlmAdapter(null, objectMapper);
        assertThat(adapter.isAvailable()).isFalse();
    }

    @Test
    void consult_parses_json_response() {
        String json = """
            {"selectedBindings": ["pattern-analysis"],
             "suppressedBindings": ["osint-screening"],
             "rationale": "pattern first",
             "earlyTermination": false}
            """;
        var adapter = createAdapter(mockProviderReturning(json));
        var decision = adapter.consult(null, null, List.of());
        assertThat(decision.selectedBindings()).containsExactly("pattern-analysis");
        assertThat(decision.suppressedBindings()).containsExactly("osint-screening");
        assertThat(decision.rationale()).isEqualTo("pattern first");
    }

    @Test
    void consult_extracts_json_from_markdown_fence() {
        String response = """
            Here is my decision:
            ```json
            {"selectedBindings": ["investigation-triage"],
             "suppressedBindings": [],
             "rationale": "sufficient evidence",
             "earlyTermination": true}
            ```
            """;
        var adapter = createAdapter(mockProviderReturning(response));
        var decision = adapter.consult(null, null, List.of());
        assertThat(decision.earlyTermination()).isTrue();
    }

    @Test
    void consult_throws_on_malformed_json() {
        var adapter = createAdapter(mockProviderReturning("not json"));
        assertThatThrownBy(() -> adapter.consult(null, null, List.of()))
            .isInstanceOf(InvalidSupervisorResponseException.class);
    }

    private ChatModelProvider mockProvider() {
        return mockProviderReturning(
            "{\"selectedBindings\":[\"a\"],\"suppressedBindings\":[],\"rationale\":\"r\",\"earlyTermination\":false}");
    }

    private ChatModelProvider mockProviderReturning(String text) {
        ChatModel chatModel = mock(ChatModel.class);
        var response = ChatResponse.builder()
            .aiMessage(AiMessage.from(text))
            .build();
        when(chatModel.chat(any(dev.langchain4j.model.chat.request.ChatRequest.class)))
            .thenReturn(response);
        ChatModelProvider provider = mock(ChatModelProvider.class);
        when(provider.type()).thenReturn(ModelType.ANTHROPIC);
        when(provider.get()).thenReturn(chatModel);
        return provider;
    }

    private AmlSupervisorLlmAdapter createAdapter(ChatModelProvider provider) {
        return new AmlSupervisorLlmAdapter(provider, objectMapper);
    }
}
```

- [ ] **Step 3: Run tests — verify they fail**

Run: `mvn test -pl app -am -Dtest=AmlSupervisorLlmAdapterTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 4: Implement AmlSupervisorLlmAdapter**

```java
package io.casehub.aml.supervisor;

import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import io.casehub.api.context.CaseContext;
import io.casehub.api.engine.PlanExecutionContext;
import io.casehub.api.model.Binding;
import io.casehub.api.model.ai.ChatModelProvider;
import io.casehub.api.model.ai.ModelType;
import io.casehub.engine.planning.plan.CasePlanModel;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

@ApplicationScoped
public class AmlSupervisorLlmAdapter {

    private static final Pattern JSON_FENCE =
        Pattern.compile("```(?:json)?\\s*\\n(\\{.*?})\\s*\\n```", Pattern.DOTALL);

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

    AmlSupervisorLlmAdapter(ChatModelProvider provider, ObjectMapper objectMapper) {
        this.chatModelProvider = provider;
        this.objectMapper = objectMapper;
    }

    public boolean isAvailable() {
        return chatModelProvider != null;
    }

    public SupervisorDecision consult(
            CasePlanModel plan, PlanExecutionContext ctx, List<Binding> eligible) {
        String prompt = buildPrompt(plan, ctx, eligible);
        return callLlm(prompt);
    }

    String buildPrompt(CasePlanModel plan, PlanExecutionContext ctx, List<Binding> eligible) {
        var sb = new StringBuilder();
        sb.append("""
            You are an AML investigation supervisor. Decide which investigation \
            steps should fire next based on accumulated evidence.

            ## Case Context
            """);
        if (ctx != null && ctx.caseContext() != null) {
            sb.append(projectCaseContext(ctx.caseContext()));
        } else {
            sb.append("No context available yet.\n");
        }
        sb.append("\n## Eligible Bindings (select a subset to fire NOW)\n");
        if (eligible != null) {
            for (Binding b : eligible) {
                sb.append("- ").append(b.getName()).append("\n");
            }
        }
        sb.append("""

            ## Instructions
            - Select which bindings to fire NOW. Suppress bindings not yet useful.
            - You MUST select at least one binding.
            - If evidence is sufficient for triage, select investigation-triage \
            and suppress remaining specialists (early termination).
            - Do not reason about the triage outcome — that is handled by the \
            deterministic evaluator.
            - Respond with JSON only:
            {"selectedBindings": ["name1"], "suppressedBindings": ["name2"], \
            "rationale": "brief explanation", "earlyTermination": false}
            """);
        return sb.toString();
    }

    private String projectCaseContext(CaseContext ctx) {
        if (ctx == null || ctx.asJsonNode() == null) return "Empty\n";
        var node = ctx.asJsonNode();
        var sb = new StringBuilder();
        appendIfPresent(sb, node, "entityResolution");
        appendIfPresent(sb, node, "patternAnalysis");
        appendIfPresent(sb, node, "osintScreening");
        appendIfPresent(sb, node, "cbrPathAdvice");
        appendIfPresent(sb, node, "investigationTriage");
        appendIfPresent(sb, node, "priorEntityContext");
        appendIfPresent(sb, node, "seniorAnalystReview");
        if (sb.isEmpty()) sb.append("No specialist findings yet.\n");
        return sb.toString();
    }

    private void appendIfPresent(StringBuilder sb, com.fasterxml.jackson.databind.JsonNode node,
                                  String field) {
        if (node.has(field) && !node.get(field).isNull()) {
            sb.append("- ").append(field).append(": ").append(node.get(field)).append("\n");
        }
    }

    SupervisorDecision callLlm(String prompt) {
        try {
            ChatModel chatModel = chatModelProvider.get();
            var request = ChatRequest.builder()
                .messages(List.of(UserMessage.from(prompt)))
                .build();
            var response = chatModel.chat(request);
            String text = response.aiMessage().text();
            String json = extractJsonBlock(text);
            return objectMapper.readValue(json, SupervisorDecision.class);
        } catch (InvalidSupervisorResponseException e) {
            throw e;
        } catch (Exception e) {
            throw new InvalidSupervisorResponseException(
                "Failed to parse LLM response: " + e.getMessage());
        }
    }

    static String extractJsonBlock(String text) {
        if (text == null || text.isBlank()) {
            throw new InvalidSupervisorResponseException("Empty LLM response");
        }
        Matcher m = JSON_FENCE.matcher(text);
        if (m.find()) return m.group(1);
        String trimmed = text.trim();
        if (trimmed.startsWith("{")) return trimmed;
        throw new InvalidSupervisorResponseException(
            "No JSON block found in LLM response");
    }
}
```

- [ ] **Step 5: Run tests — verify pass**

Run: `mvn test -pl app -am -Dtest=AmlSupervisorLlmAdapterTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add app/pom.xml app/src/main/java/io/casehub/aml/supervisor/AmlSupervisorLlmAdapter.java app/src/test/java/io/casehub/aml/supervisor/AmlSupervisorLlmAdapterTest.java
git commit -m "feat(#8): LLM adapter — prompt building, ChatModel call, JSON parsing

Refs casehubio/aml#8"
```

---

### Task 4: PlanningStrategy + plan configurer

**Files:**
- Create: `app/src/main/java/io/casehub/aml/supervisor/AmlInvestigationSupervisor.java`
- Create: `app/src/main/java/io/casehub/aml/supervisor/AmlSupervisorPlanConfigurer.java`
- Test: `app/src/test/java/io/casehub/aml/supervisor/AmlInvestigationSupervisorTest.java`

**Interfaces:**
- Consumes: `AmlSupervisorLlmAdapter.isAvailable()`, `AmlSupervisorLlmAdapter.consult()` (from Task 3), `AmlSupervisorPendingStore.put()` (from Task 1), `PlanningStrategy` SPI (from engine-planning), `BlackboardPlanConfigurer` SPI (from engine-planning)
- Produces: `AmlInvestigationSupervisor` CDI bean with `id() = "aml-supervisor"` — wired by engine via `CompoundStrategyDispatcher`
- Produces: `AmlSupervisorPlanConfigurer` — registers `supervised-investigation` compound with scoped bindings

- [ ] **Step 1: Write failing tests for AmlInvestigationSupervisor**

```java
package io.casehub.aml.supervisor;

import io.casehub.api.engine.PlanExecutionContext;
import io.casehub.api.model.Binding;
import io.casehub.engine.planning.plan.CasePlanModel;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class AmlInvestigationSupervisorTest {

    @Test
    void id_returns_aml_supervisor() {
        var supervisor = new AmlInvestigationSupervisor(
            mock(AmlSupervisorLlmAdapter.class), new AmlSupervisorPendingStore());
        assertThat(supervisor.id()).isEqualTo("aml-supervisor");
    }

    @Test
    void select_delegates_to_adapter_and_returns_validated_subset() {
        var adapter = mock(AmlSupervisorLlmAdapter.class);
        when(adapter.isAvailable()).thenReturn(true);
        var decision = new SupervisorDecision(
            List.of("pattern-analysis"), List.of("osint-screening"), "reason", false);
        when(adapter.consult(any(), any(), any())).thenReturn(decision);

        var store = new AmlSupervisorPendingStore();
        var supervisor = new AmlInvestigationSupervisor(adapter, store);

        Binding b1 = mockBinding("pattern-analysis");
        Binding b2 = mockBinding("osint-screening");
        var ctx = mockContext();

        List<Binding> result = supervisor.select(null, ctx, List.of(b1, b2));
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getName()).isEqualTo("pattern-analysis");
    }

    @Test
    void select_passthrough_when_adapter_unavailable() {
        var adapter = mock(AmlSupervisorLlmAdapter.class);
        when(adapter.isAvailable()).thenReturn(false);
        var supervisor = new AmlInvestigationSupervisor(adapter, new AmlSupervisorPendingStore());

        Binding b1 = mockBinding("pattern-analysis");
        List<Binding> result = supervisor.select(null, mockContext(), List.of(b1));
        assertThat(result).containsExactly(b1);
        verify(adapter, never()).consult(any(), any(), any());
    }

    @Test
    void select_fallback_on_adapter_exception() {
        var adapter = mock(AmlSupervisorLlmAdapter.class);
        when(adapter.isAvailable()).thenReturn(true);
        when(adapter.consult(any(), any(), any()))
            .thenThrow(new InvalidSupervisorResponseException("fail"));

        var store = new AmlSupervisorPendingStore();
        var supervisor = new AmlInvestigationSupervisor(adapter, store);

        Binding b1 = mockBinding("a");
        Binding b2 = mockBinding("b");
        var ctx = mockContext();
        UUID caseId = ctx.caseId();

        List<Binding> result = supervisor.select(null, ctx, List.of(b1, b2));
        assertThat(result).containsExactly(b1, b2);

        var pending = store.take(caseId);
        assertThat(pending).isNotNull();
        assertThat(pending.degraded()).isTrue();
    }

    @Test
    void select_fallback_on_hallucinated_binding() {
        var adapter = mock(AmlSupervisorLlmAdapter.class);
        when(adapter.isAvailable()).thenReturn(true);
        var decision = new SupervisorDecision(
            List.of("nonexistent"), List.of(), "reason", false);
        when(adapter.consult(any(), any(), any())).thenReturn(decision);

        var store = new AmlSupervisorPendingStore();
        var supervisor = new AmlInvestigationSupervisor(adapter, store);

        Binding b1 = mockBinding("pattern-analysis");
        var ctx = mockContext();

        List<Binding> result = supervisor.select(null, ctx, List.of(b1));
        assertThat(result).containsExactly(b1);
        assertThat(store.take(ctx.caseId()).degraded()).isTrue();
    }

    @Test
    void select_stores_pending_decision_on_success() {
        var adapter = mock(AmlSupervisorLlmAdapter.class);
        when(adapter.isAvailable()).thenReturn(true);
        var decision = new SupervisorDecision(
            List.of("pattern-analysis"), List.of(), "reason", false);
        when(adapter.consult(any(), any(), any())).thenReturn(decision);

        var store = new AmlSupervisorPendingStore();
        var supervisor = new AmlInvestigationSupervisor(adapter, store);
        var ctx = mockContext();

        supervisor.select(null, ctx, List.of(mockBinding("pattern-analysis")));

        var pending = store.take(ctx.caseId());
        assertThat(pending).isNotNull();
        assertThat(pending.degraded()).isFalse();
        assertThat(pending.decision().selectedBindings()).containsExactly("pattern-analysis");
    }

    private Binding mockBinding(String name) {
        Binding b = mock(Binding.class);
        when(b.getName()).thenReturn(name);
        return b;
    }

    private PlanExecutionContext mockContext() {
        return new PlanExecutionContext(
            UUID.randomUUID(), null, null,
            io.casehub.api.model.CaseStatus.RUNNING, "t1",
            List.of(), null, null);
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn test -pl app -am -Dtest=AmlInvestigationSupervisorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Implement AmlInvestigationSupervisor**

```java
package io.casehub.aml.supervisor;

import io.casehub.api.engine.PlanExecutionContext;
import io.casehub.api.model.Binding;
import io.casehub.engine.planning.control.PlanningStrategy;
import io.casehub.engine.planning.plan.CasePlanModel;
import io.quarkus.arc.Unremovable;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;
import org.jboss.logging.Logger;

@ApplicationScoped
@Unremovable
public class AmlInvestigationSupervisor implements PlanningStrategy {

    private static final Logger LOG = Logger.getLogger(AmlInvestigationSupervisor.class);

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
    public String id() {
        return "aml-supervisor";
    }

    @Override
    public String getName() {
        return "AML Investigation LLM Supervisor";
    }

    @Override
    public List<Binding> select(
            CasePlanModel plan, PlanExecutionContext context, List<Binding> eligible) {
        if (!llmAdapter.isAvailable() || eligible.isEmpty()) {
            return eligible;
        }
        try {
            SupervisorDecision decision = llmAdapter.consult(plan, context, eligible);
            List<Binding> selected = validateAndResolve(decision, eligible);
            pendingStore.put(new PendingSupervisorDecision(
                decision, eligible.size(), false,
                context.caseId(), context.tenancyId()));
            return selected;
        } catch (Exception e) {
            LOG.warnf(e, "Supervisor degraded for case %s: %s",
                context.caseId(), e.getMessage());
            return degradedFallback(eligible, context);
        }
    }

    private List<Binding> validateAndResolve(
            SupervisorDecision decision, List<Binding> eligible) {
        Set<String> eligibleNames = eligible.stream()
            .map(Binding::getName)
            .collect(Collectors.toSet());
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

    private List<Binding> degradedFallback(
            List<Binding> eligible, PlanExecutionContext context) {
        List<String> allNames = eligible.stream()
            .map(Binding::getName).toList();
        var degradedDecision = new SupervisorDecision(
            allNames, List.of(),
            "LLM unavailable — degraded to choreography", false);
        pendingStore.put(new PendingSupervisorDecision(
            degradedDecision, eligible.size(), true,
            context.caseId(), context.tenancyId()));
        return eligible;
    }
}
```

- [ ] **Step 4: Implement AmlSupervisorPlanConfigurer**

```java
package io.casehub.aml.supervisor;

import io.casehub.api.engine.PlanExecutionContext;
import io.casehub.api.model.CaseDefinition;
import io.casehub.engine.planning.control.BlackboardPlanConfigurer;
import io.casehub.engine.planning.plan.CasePlanModel;
import io.casehub.engine.planning.plan.PlanItemDefinition;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class AmlSupervisorPlanConfigurer implements BlackboardPlanConfigurer {

    private static final String AML_INVESTIGATION = "aml-investigation";

    @Override
    public boolean supports(CaseDefinition definition) {
        return definition != null
            && AML_INVESTIGATION.equals(definition.getName());
    }

    @Override
    public void configure(CasePlanModel plan, PlanExecutionContext context) {
        var compound = PlanItemDefinition.Compound.builder("supervised-investigation")
            .planningStrategy("aml-supervisor")
            .binding("pattern-analysis")
            .binding("osint-screening")
            .binding("investigation-triage")
            .binding("sar-drafting")
            .binding("senior-analyst-required-resolution")
            .build();
        plan.registerDefinition(compound);
    }
}
```

- [ ] **Step 5: Run tests — verify pass**

Run: `mvn test -pl app -am -Dtest=AmlInvestigationSupervisorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/aml/supervisor/AmlInvestigationSupervisor.java app/src/main/java/io/casehub/aml/supervisor/AmlSupervisorPlanConfigurer.java app/src/test/java/io/casehub/aml/supervisor/AmlInvestigationSupervisorTest.java
git commit -m "feat(#8): AmlInvestigationSupervisor PlanningStrategy + compound configurer

Refs casehubio/aml#8"
```

---

### Task 5: Audit observer — deferred ledger writes

**Files:**
- Create: `app/src/main/java/io/casehub/aml/supervisor/AmlSupervisorAuditObserver.java`
- Test: `app/src/test/java/io/casehub/aml/supervisor/AmlSupervisorAuditObserverTest.java`

**Interfaces:**
- Consumes: `AmlSupervisorPendingStore.take(UUID)` (from Task 1), `AmlSupervisorDecisionLedgerEntry` (from Task 2), `WorkerDecisionEvent` (from engine-common), `LedgerEntryRepository.save()` (from casehub-ledger)
- Produces: writes `AmlSupervisorDecisionLedgerEntry` to ledger on each `WorkerDecisionEvent`

- [ ] **Step 1: Write failing test**

```java
package io.casehub.aml.supervisor;

import io.casehub.aml.ledger.AmlSupervisorDecisionLedgerEntry;
import io.casehub.engine.common.spi.event.WorkerDecisionEvent;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.UUID;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class AmlSupervisorAuditObserverTest {

    @Test
    void writes_ledger_entry_from_pending_store() {
        var store = new AmlSupervisorPendingStore();
        var repo = mock(LedgerEntryRepository.class);
        var observer = new AmlSupervisorAuditObserver(store, repo);

        UUID caseId = UUID.randomUUID();
        var decision = new SupervisorDecision(
            List.of("pattern-analysis"), List.of("osint-screening"), "reason", false);
        store.put(new PendingSupervisorDecision(decision, 2, false, caseId, "t1"));

        observer.onWorkerDecision(new WorkerDecisionEvent(
            caseId, "t1", "pattern-analysis-agent", "pattern-analysis", null));

        verify(repo).save(any(AmlSupervisorDecisionLedgerEntry.class), eq("t1"));
    }

    @Test
    void no_op_when_no_pending_decision() {
        var store = new AmlSupervisorPendingStore();
        var repo = mock(LedgerEntryRepository.class);
        var observer = new AmlSupervisorAuditObserver(store, repo);

        observer.onWorkerDecision(new WorkerDecisionEvent(
            UUID.randomUUID(), "t1", "worker", "cap", null));

        verify(repo, never()).save(any(), any());
    }

    @Test
    void degraded_decision_writes_degraded_entry() {
        var store = new AmlSupervisorPendingStore();
        var repo = mock(LedgerEntryRepository.class);
        var observer = new AmlSupervisorAuditObserver(store, repo);

        UUID caseId = UUID.randomUUID();
        var decision = new SupervisorDecision(
            List.of("a", "b"), List.of(), "degraded", false);
        store.put(new PendingSupervisorDecision(decision, 2, true, caseId, "t1"));

        observer.onWorkerDecision(new WorkerDecisionEvent(
            caseId, "t1", "worker", "a", null));

        var captor = org.mockito.ArgumentCaptor.forClass(AmlSupervisorDecisionLedgerEntry.class);
        verify(repo).save(captor.capture(), eq("t1"));
        assertThat(captor.getValue().degraded).isTrue();
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn test -pl app -am -Dtest=AmlSupervisorAuditObserverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Implement AmlSupervisorAuditObserver**

```java
package io.casehub.aml.supervisor;

import io.casehub.aml.ledger.AmlSupervisorDecisionLedgerEntry;
import io.casehub.engine.common.spi.event.WorkerDecisionEvent;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.platform.api.identity.TenancyConstants;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.UUID;
import org.jboss.logging.Logger;

@ApplicationScoped
public class AmlSupervisorAuditObserver {

    private static final Logger LOG = Logger.getLogger(AmlSupervisorAuditObserver.class);

    private final AmlSupervisorPendingStore pendingStore;
    private final LedgerEntryRepository ledgerEntryRepository;

    @Inject
    public AmlSupervisorAuditObserver(
            AmlSupervisorPendingStore pendingStore,
            LedgerEntryRepository ledgerEntryRepository) {
        this.pendingStore = pendingStore;
        this.ledgerEntryRepository = ledgerEntryRepository;
    }

    public void onWorkerDecision(@ObservesAsync WorkerDecisionEvent event) {
        PendingSupervisorDecision pending = pendingStore.take(event.caseId());
        if (pending == null) return;

        try {
            var entry = new AmlSupervisorDecisionLedgerEntry();
            entry.id = UUID.randomUUID();
            entry.selectedBindings = String.join(",", pending.decision().selectedBindings());
            entry.suppressedBindings = String.join(",", pending.decision().suppressedBindings());
            entry.rationale = pending.decision().rationale();
            entry.earlyTermination = pending.decision().earlyTermination();
            entry.eligibleCount = pending.eligibleCount();
            entry.degraded = pending.degraded();
            entry.subjectId = UUID.nameUUIDFromBytes(
                ("aml-supervisor:" + event.caseId()).getBytes(StandardCharsets.UTF_8));
            entry.tenancyId = pending.tenancyId() != null
                ? pending.tenancyId() : TenancyConstants.DEFAULT_TENANT_ID;
            entry.actorId = "aml-supervisor";
            entry.actorType = ActorType.SYSTEM;
            entry.entryType = LedgerEntryType.EVENT;
            entry.occurredAt = Instant.now();
            entry.reconstructed = false;
            ledgerEntryRepository.save(entry, entry.tenancyId);
        } catch (Exception e) {
            LOG.warnf(e, "Failed to write supervisor audit entry for case %s",
                event.caseId());
        }
    }
}
```

- [ ] **Step 4: Run tests — verify pass**

Run: `mvn test -pl app -am -Dtest=AmlSupervisorAuditObserverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/aml/supervisor/AmlSupervisorAuditObserver.java app/src/test/java/io/casehub/aml/supervisor/AmlSupervisorAuditObserverTest.java
git commit -m "feat(#8): supervisor audit observer — deferred ledger writes on WorkerDecisionEvent

Refs casehubio/aml#8"
```

---

### Task 6: Integration tests + wiring

**Files:**
- Modify: `app/src/main/resources/aml/aml-investigation.yaml` — no YAML change (compound via configurer)
- Modify: `app/src/test/resources/application.properties` — CDI exclusions for engine-ai
- Test: `app/src/test/java/io/casehub/aml/supervisor/SupervisorModeInvestigationTest.java`
- Test: `app/src/test/java/io/casehub/aml/supervisor/SupervisorDegradedInvestigationTest.java`

**Interfaces:**
- Consumes: All components from Tasks 1-5, AML investigation infrastructure (`AmlLayer6Resource`/`AmlLayer9Resource` for starting investigations), mock `ChatModelProvider` for test isolation

**Note:** The integration tests require the full AML Quarkus test infrastructure. The mock `ChatModelProvider` is registered as a `@QuarkusTestProfile` alternative or inner `@ApplicationScoped` bean that displaces the real provider. Follow existing `InvestigationTriageFlowTest` patterns for investigation lifecycle management.

- [ ] **Step 1: Add CDI exclusions to test application.properties**

Check if `casehub-engine-ai` introduces any CDI beans that conflict. The `AnthropicChatModelProvider` uses ServiceLoader (no CDI annotations), so it should not conflict. If `casehub-engine-ai` has CDI beans that conflict, add to test `application.properties`:
```properties
# Engine-ai CDI exclusions (if needed)
```

Verify with: `mvn test -pl app -am -Dtest=InvestigationTriageFlowTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: existing tests still pass with new dependency

- [ ] **Step 2: Write SupervisorModeInvestigationTest**

This test uses a mock `ChatModelProvider` that returns structured decisions. The mock is an inner `@ApplicationScoped @Alternative @Priority(1)` class. The test starts an investigation, verifies the LLM supervisor is consulted for compound-scoped bindings, and drains to completion.

```java
package io.casehub.aml.supervisor;

import io.casehub.aml.ledger.AmlSupervisorDecisionLedgerEntry;
import io.casehub.api.model.ai.ChatModelProvider;
import io.casehub.api.model.ai.ModelType;
import io.quarkus.test.junit.QuarkusTest;
// ... full integration test following existing InvestigationTriageFlowTest patterns
// Key assertions:
// 1. Investigation completes (drain to status=completed)
// 2. AmlSupervisorDecisionLedgerEntry entries exist in ledger
// 3. Entries show selected/suppressed bindings from mock decisions
// 4. earlyTermination flag correctly reflects mock decisions
```

The exact test code depends on the existing test infrastructure (REST endpoints, mock workers, Awaitility patterns). The implementer should:
1. Read `InvestigationTriageFlowTest` for the investigation lifecycle pattern
2. Add a mock `ChatModelProvider` inner class
3. Start an investigation with `FlagReason.HIGH_RISK_JURISDICTION` (triggers SAR path)
4. Verify supervisor ledger entries after completion

- [ ] **Step 3: Write SupervisorDegradedInvestigationTest**

Same structure but the mock `ChatModelProvider.get()` always throws. Verify:
1. Investigation completes identically to non-supervisor mode
2. Degraded ledger entries present
3. All bindings fire (choreography fallback)

- [ ] **Step 4: Run integration tests**

Run: `mvn test -pl app -am -Dtest="SupervisorModeInvestigationTest,SupervisorDegradedInvestigationTest" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Run full test suite to verify no regressions**

Run: `mvn verify -pl app -am`
Expected: All 322+ tests pass

- [ ] **Step 6: Commit**

```bash
git add app/src/test/java/io/casehub/aml/supervisor/ app/src/test/resources/
git commit -m "test(#8): supervisor mode integration tests — LLM-guided and degraded paths

Refs casehubio/aml#8"
```
