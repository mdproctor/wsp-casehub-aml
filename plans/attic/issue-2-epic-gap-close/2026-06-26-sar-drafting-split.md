# SAR-Drafting / Compliance-Review-Opening Split — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Separate `openReview()` side effect from SAR-drafting workers, gate SAR filing behind MLRO approval via `PlannedAction(SAR_FILING)`, and add a post-gate `compliance-review-opening` worker.

**Architecture:** Convert sar-drafting workers from FlowWorkerFunction to WorkerFunction.Sync (required for PlannedAction — engine#564). Add a new Flow worker `compliance-review-opening-agent` that calls `openReview()` after MLRO gate approval. Engine's `AmlActionRiskClassifier` returns `GateRequired(ALWAYS)` for `SAR_FILING` — zero classifier changes needed.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, casehub-engine 0.2-SNAPSHOT, casehub-worker-api 0.2-SNAPSHOT

## Global Constraints

- Build with `mvn -pl app -am` (not `./mvnw`). Add `-Dsurefire.failIfNoSpecifiedTests=false` when combining `-am` with `-Dtest=ClassName`.
- `PlannedAction` not supported in `FlowWorkerFunction` (engine#564). Any worker declaring a `PlannedAction` must use `WorkerFunction.Sync`.
- `WorkerExecutionContext.current().caseId()` is ThreadLocal-based. Available in both Flow (set by `FlowWorkerFunctionHandler` before `wfInstance.start()`) and Sync (set by `SyncAgentWorkerFunctionHandler`) handlers.
- `Worker.builder().function(Function<Map<String, Object>, WorkerResult>)` wraps the lambda in `WorkerFunction.Sync` automatically.
- `Worker.builder().function(WorkerFunction)` accepts `FlowWorkerFunction` directly.
- All commits must reference `Refs #58`.

---

### Task 1: Descriptor unit test + core refactor

**Files:**
- Modify: `app/src/test/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptorTest.java`
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptor.java`

**Interfaces:**
- Consumes: `WorkerResult.of(Map, PlannedAction)`, `PlannedAction.of(String, String, Map)`, `AmlActionType.SAR_FILING.actionType()`, `WorkerFunction.Sync` (all from casehub-worker-api / casehub-aml-api)
- Produces: 8 workers from `descriptor.workers()` — 6 Flow, 2 Sync. New worker name: `compliance-review-opening-agent` with capability `compliance-review-opening`.

- [ ] **Step 1: Update descriptor test — worker count and name set**

In `AmlInvestigationCaseDescriptorTest.java`, replace the `workers_returnsSevenDistinctWorkers` method:

```java
@Test
void workers_returnsEightDistinctWorkers() {
    final List<Worker> workers = descriptor.workers();
    assertEquals(8, workers.size(), "Descriptor must declare exactly 8 workers");
    final Set<String> names = workers.stream().map(Worker::name).collect(Collectors.toSet());
    assertEquals(8, names.size(), "All worker names must be distinct");
    assertEquals(Set.of(
            "entity-resolution-agent",
            "pattern-analysis-agent",
            "osint-screening-agent",
            "osint-screening-agent-senior",
            "senior-analyst-agent",
            "sar-drafting-agent-junior",
            "sar-drafting-agent-senior",
            "compliance-review-opening-agent"), names);
}
```

- [ ] **Step 2: Update descriptor test — capability map**

In `capability_names_match_expected_tags`, add the new mapping:

```java
assertEquals("compliance-review-opening", capByWorker.get("compliance-review-opening-agent"));
```

- [ ] **Step 3: Redesign execution model classification test**

Replace the entire `worker_execution_model_classification_is_exhaustive` method. Add `import io.casehub.worker.api.WorkerFunction;` to the imports.

```java
@Test
void worker_execution_model_classification_is_exhaustive() {
    final Set<String> FLOW_WORKERS = Set.of(
            "entity-resolution-agent",
            "pattern-analysis-agent",
            "osint-screening-agent",
            "osint-screening-agent-senior",
            "senior-analyst-agent",
            "compliance-review-opening-agent");

    final Set<String> SYNC_WORKERS = Set.of(
            "sar-drafting-agent-junior",
            "sar-drafting-agent-senior");

    for (final Worker w : descriptor.workers()) {
        if (FLOW_WORKERS.contains(w.name())) {
            assertInstanceOf(FlowWorkerFunction.class, w.function(),
                    "Worker " + w.name() + " must use FlowWorkerFunction.");
        } else if (SYNC_WORKERS.contains(w.name())) {
            assertInstanceOf(WorkerFunction.Sync.class, w.function(),
                    "Worker " + w.name() + " must use WorkerFunction.Sync (PlannedAction support).");
        } else {
            fail("Worker " + w.name() + " is unclassified — add it to FLOW_WORKERS or SYNC_WORKERS.");
        }
    }
}
```

- [ ] **Step 4: Run descriptor test to verify failures**

Run: `mvn -pl app -am test -Dtest=AmlInvestigationCaseDescriptorTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: FAIL — worker count is still 7, `compliance-review-opening-agent` missing, sar-drafting workers are still FlowWorkerFunction.

- [ ] **Step 5: Add imports to descriptor**

In `AmlInvestigationCaseDescriptor.java`, add:

```java
import io.casehub.worker.api.PlannedAction;
import io.casehub.worker.api.WorkerResult;
import io.casehub.aml.domain.AmlActionType;
```

Keep existing imports: `FlowWorkerFunction`, `WorkerExecutionContext`, `Capability`, `Worker`.

- [ ] **Step 6: Convert sarDraftingWorkerJunior to Sync**

Replace the entire `sarDraftingWorkerJunior()` method:

```java
private Worker sarDraftingWorkerJunior() {
    return Worker.builder()
            .name("sar-drafting-agent-junior")
            .capabilities(List.of(cap("sar-drafting")))
            .function((final Map<String, Object> input) -> {
                @SuppressWarnings("unchecked")
                final Map<String, Object> txMap = (Map<String, Object>) input.get("transaction");
                final SuspiciousTransaction tx =
                        objectMapper.convertValue(txMap, SuspiciousTransaction.class);
                @SuppressWarnings("unchecked")
                final Map<String, Object> entityMap =
                        (Map<String, Object>) input.get("entityResolution");
                final String entityType = entityMap != null
                        ? (String) entityMap.getOrDefault("entityType", "UNKNOWN") : "UNKNOWN";
                @SuppressWarnings("unchecked")
                final Map<String, Object> osintMap =
                        (Map<String, Object>) input.get("osintScreening");
                final boolean osintDeclined = osintMap != null
                        && Boolean.TRUE.equals(osintMap.get("declined"));
                final String sarNarrative = "SAR filed for transaction " + tx.id()
                        + ". Amount: " + tx.amount() + " " + tx.currency()
                        + (osintDeclined ? " OSINT screening declined." : "");
                return WorkerResult.of(
                        Map.of("sarNarrative", sarNarrative),
                        PlannedAction.of(
                                "SAR filing for transaction " + tx.id(),
                                AmlActionType.SAR_FILING.actionType(),
                                Map.of("transactionId", tx.id(),
                                       "amount", tx.amount().toString(),
                                       "currency", tx.currency(),
                                       "entityType", entityType)));
            })
            .build();
}
```

- [ ] **Step 7: Convert sarDraftingWorkerSenior to Sync**

Replace the entire `sarDraftingWorkerSenior()` method:

```java
private Worker sarDraftingWorkerSenior() {
    return Worker.builder()
            .name("sar-drafting-agent-senior")
            .capabilities(List.of(cap("sar-drafting")))
            .function((final Map<String, Object> input) -> {
                @SuppressWarnings("unchecked")
                final Map<String, Object> txMap = (Map<String, Object>) input.get("transaction");
                @SuppressWarnings("unchecked")
                final Map<String, Object> entityMap =
                        (Map<String, Object>) input.get("entityResolution");
                @SuppressWarnings("unchecked")
                final Map<String, Object> osintMap =
                        (Map<String, Object>) input.get("osintScreening");
                final SuspiciousTransaction tx =
                        objectMapper.convertValue(txMap, SuspiciousTransaction.class);
                final String entityType = entityMap != null
                        ? (String) entityMap.getOrDefault("entityType", "UNKNOWN")
                        : "UNKNOWN";
                final boolean osintDeclined = osintMap != null
                        && Boolean.TRUE.equals(osintMap.get("declined"));
                final String sarNarrative = buildNarrative(tx, entityType, osintDeclined);
                return WorkerResult.of(
                        Map.of("sarNarrative", sarNarrative),
                        PlannedAction.of(
                                "SAR filing for transaction " + tx.id(),
                                AmlActionType.SAR_FILING.actionType(),
                                Map.of("transactionId", tx.id(),
                                       "amount", tx.amount().toString(),
                                       "currency", tx.currency(),
                                       "entityType", entityType)));
            })
            .build();
}
```

- [ ] **Step 8: Add complianceReviewOpeningWorker**

Add this new method to `AmlInvestigationCaseDescriptor`:

```java
private Worker complianceReviewOpeningWorker() {
    return Worker.builder()
            .name("compliance-review-opening-agent")
            .capabilities(List.of(cap("compliance-review-opening")))
            .function(new FlowWorkerFunction(
                workflow("compliance-review-opening")
                    .tasks(
                        function(s -> {
                            @SuppressWarnings("unchecked")
                            final Map<String, Object> input = (Map<String, Object>) s;
                            @SuppressWarnings("unchecked")
                            final Map<String, Object> txMap =
                                    (Map<String, Object>) input.get("transaction");
                            final SuspiciousTransaction tx =
                                    objectMapper.convertValue(txMap, SuspiciousTransaction.class);
                            final String sarNarrative = (String) input.get("sarNarrative");
                            final UUID caseId = WorkerExecutionContext.current().caseId();
                            final String complianceTaskId =
                                    complianceReviewLifecycle.openReview(
                                            tx, buildSummary(input, tx, sarNarrative), caseId);
                            return Map.of("complianceTaskId", complianceTaskId);
                        }, Map.class))
                    .build()))
            .build();
}
```

- [ ] **Step 9: Update workers() list**

Replace the `workers()` method body:

```java
List<Worker> workers() {
    return List.of(
            entityResolutionWorker(),
            patternAnalysisWorker(),
            osintScreeningWorker(),
            osintScreeningWorkerSenior(),
            seniorAnalystWorker(),
            sarDraftingWorkerJunior(),
            sarDraftingWorkerSenior(),
            complianceReviewOpeningWorker()
    );
}
```

- [ ] **Step 10: Run descriptor test to verify pass**

Run: `mvn -pl app -am test -Dtest=AmlInvestigationCaseDescriptorTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: PASS — all 5 tests green.

- [ ] **Step 11: Commit**

```bash
git add app/src/test/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptorTest.java app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptor.java
git commit -m "refactor: convert sar-drafting workers to Sync + PlannedAction(SAR_FILING), add compliance-review-opening-agent

Refs #58"
```

---

### Task 2: YAML + CaseHub integration test

**Files:**
- Modify: `app/src/main/resources/aml/aml-investigation.yaml`
- Modify: `app/src/test/java/io/casehub/aml/engine/AmlInvestigationCaseHubTest.java`

**Interfaces:**
- Consumes: `compliance-review-opening-agent` from Task 1 (must match capability name `compliance-review-opening`)
- Produces: YAML definition with 6 capabilities, 7 bindings, 1 goal — loaded by `AmlInvestigationCaseHub`

- [ ] **Step 1: Update CaseHub test — capability count**

In `AmlInvestigationCaseHubTest.java`, update `hasFiveCapabilities`:

```java
@Test
void hasSixCapabilities() {
    final var names = caseHub.getDefinition().getCapabilities()
            .stream().map(c -> c.name()).toList();
    assertEquals(6, names.size());
    assertTrue(names.containsAll(List.of(
            "entity-resolution", "pattern-analysis", "osint-screening",
            "senior-analyst-review", "sar-drafting", "compliance-review-opening")));
}
```

- [ ] **Step 2: Update CaseHub test — binding count**

Update `hasSixBindings`:

```java
@Test
void hasSevenBindings() {
    final var names = caseHub.getDefinition().getBindings()
            .stream().map(b -> b.getName()).toList();
    assertEquals(7, names.size());
    assertTrue(names.containsAll(List.of(
            "entity-resolution", "pattern-analysis", "osint-screening",
            "senior-analyst-required-prior-context", "senior-analyst-required-resolution",
            "sar-drafting", "compliance-review-opening")));
}
```

- [ ] **Step 3: Update CaseHub test — worker count**

Update `hasSevenWorkers`:

```java
@Test
void hasEightWorkers() {
    final var workers = caseHub.getDefinition().getWorkers();
    assertEquals(8, workers.size(), "Exactly 8 workers expected — size catches double-augmentation");
    final var names = Set.copyOf(workers.stream().map(w -> w.name()).toList());
    assertEquals(Set.of(
            "entity-resolution-agent", "pattern-analysis-agent",
            "osint-screening-agent", "osint-screening-agent-senior",
            "senior-analyst-agent",
            "sar-drafting-agent-junior", "sar-drafting-agent-senior",
            "compliance-review-opening-agent"), names);
}
```

- [ ] **Step 4: Run CaseHub test to verify failures**

Run: `mvn -pl app -am test -Dtest=AmlInvestigationCaseHubTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: FAIL — YAML still has 5 capabilities and 6 bindings.

- [ ] **Step 5: Update sar-drafting capability in YAML**

In `aml-investigation.yaml`, update the `sar-drafting` capability (around line 31):

```yaml
    - name: sar-drafting
      description: "Synthesise specialist findings into SAR narrative"
      inputSchema: "{ transaction: .transaction, entityResolution: .entityResolution, patternAnalysis: .patternAnalysis, osintScreening: .osintScreening }"
      outputSchema: "{ sarNarrative: .sarNarrative }"
```

- [ ] **Step 6: Add compliance-review-opening capability in YAML**

Add after the sar-drafting capability:

```yaml
    - name: compliance-review-opening
      description: "Open compliance officer review WorkItem after MLRO gate approval"
      inputSchema: "{ transaction: .transaction, entityResolution: .entityResolution, patternAnalysis: .patternAnalysis, osintScreening: .osintScreening, sarNarrative: .sarNarrative }"
      outputSchema: "{ complianceTaskId: .complianceTaskId }"
```

- [ ] **Step 7: Update sar-drafting binding comment in YAML**

Replace the binding comment (around lines 94–95):

```yaml
    ## Fires once all specialist findings are present (entity + pattern + osint).
    ## SAR-drafting worker synthesises the narrative and declares PlannedAction(SAR_FILING);
    ## complianceTaskId is set by compliance-review-opening after MLRO gate approval.
```

- [ ] **Step 8: Add compliance-review-opening binding in YAML**

Add after the sar-drafting binding (at end of bindings section):

```yaml
    ## Fires after MLRO gate approves SAR filing. The compliance-review-opening worker
    ## creates the compliance officer WorkItem (30-day FinCEN SLA) and writes complianceTaskId.
    - name: compliance-review-opening
      on: { contextChange: {} }
      when: ".sarNarrative != null and .complianceTaskId == null"
      capability: compliance-review-opening
```

- [ ] **Step 9: Run CaseHub test to verify pass**

Run: `mvn -pl app -am test -Dtest=AmlInvestigationCaseHubTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: PASS — all 5 tests green.

- [ ] **Step 10: Commit**

```bash
git add app/src/main/resources/aml/aml-investigation.yaml app/src/test/java/io/casehub/aml/engine/AmlInvestigationCaseHubTest.java
git commit -m "refactor: add compliance-review-opening capability + binding to YAML, update counts

Refs #58"
```

---

### Task 3: Trust score seeding

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/trust/AmlTrustScoreSeeder.java`
- Modify: `app/src/test/java/io/casehub/aml/trust/AmlTrustScoreSeederTest.java`

**Interfaces:**
- Consumes: Worker name `compliance-review-opening-agent` and capability tag `compliance-review-opening` from Tasks 1–2
- Produces: Seeded trust score (alpha=9, beta=1, score ≈ 0.90) for `compliance-review-opening-agent`

- [ ] **Step 1: Write failing test for compliance-review-opening seed**

In `AmlTrustScoreSeederTest.java`, add:

```java
@Test
void compliance_review_opening_seeded_with_high_trust() {
    final var score = trustRepo.findCapabilityScore(
            "compliance-review-opening-agent", "compliance-review-opening");
    assertTrue(score.isPresent(),
            "CAPABILITY score must exist for compliance-review-opening-agent");
    assertEquals(0.90, score.get().trustScore, 0.01);
}
```

- [ ] **Step 2: Run test to verify failure**

Run: `mvn -pl app -am test -Dtest=AmlTrustScoreSeederTest#compliance_review_opening_seeded_with_high_trust -Dsurefire.failIfNoSpecifiedTests=false`

Expected: FAIL — no score seeded yet.

- [ ] **Step 3: Add seed entry**

In `AmlTrustScoreSeeder.java`, add to the `SEEDS` list (after the senior-analyst-agent entry):

```java
new WorkerSeed("compliance-review-opening-agent", "compliance-review-opening", 9, 1)
```

- [ ] **Step 4: Run test to verify pass**

Run: `mvn -pl app -am test -Dtest=AmlTrustScoreSeederTest -Dsurefire.failIfNoSpecifiedTests=false`

Expected: PASS — all 7 tests green (6 existing + 1 new).

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/aml/trust/AmlTrustScoreSeeder.java app/src/test/java/io/casehub/aml/trust/AmlTrustScoreSeederTest.java
git commit -m "feat: seed trust score for compliance-review-opening-agent (alpha=9, beta=1)

Refs #58"
```

---

### Task 4: Layer 6 integration tests — gate approval

**Files:**
- Modify: `app/src/test/java/io/casehub/aml/engine/AmlLayer6InvestigationTest.java`
- Modify: `app/src/test/java/io/casehub/aml/engine/AmlLayer6ResourceTest.java`

**Interfaces:**
- Consumes: Gate WorkItem created by engine's `ActionGateWorkItemHandler` with `callerRef` pattern `case:{caseId}/gate:*` and `candidateGroups = "aml-mlro"`. `WorkItemService.completeFromSystem(id, user, outcome)` approves the gate.
- Produces: All integration tests pass with the MLRO gate approval step inserted.

- [ ] **Step 1: Add gate infrastructure to AmlLayer6InvestigationTest**

Add these imports and fields to `AmlLayer6InvestigationTest.java`:

```java
import io.casehub.aml.domain.AmlGroups;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.service.WorkItemService;
import io.quarkus.narayana.jta.QuarkusTransaction;
import jakarta.inject.Inject;
import jakarta.persistence.PersistenceContext;
// (keep existing @PersistenceContext(unitName = "qhorus") EntityManager em;)
```

Add fields:

```java
@PersistenceContext
EntityManager defaultEm;

@Inject
WorkItemService workItemService;
```

Add helper method:

```java
private List<WorkItem> findGateWorkItems(final UUID caseId) {
    return QuarkusTransaction.requiringNew().call(() ->
        defaultEm.createQuery(
            "SELECT w FROM WorkItem w WHERE w.callerRef LIKE :pattern",
            WorkItem.class)
            .setParameter("pattern", "case:" + caseId + "/gate:%")
            .getResultList());
}

private void awaitAndApproveGate(final UUID caseId) {
    Awaitility.await()
            .atMost(15, TimeUnit.SECONDS)
            .pollInterval(300, TimeUnit.MILLISECONDS)
            .until(() -> !findGateWorkItems(caseId).isEmpty());
    final List<WorkItem> gateItems = findGateWorkItems(caseId);
    assertEquals(1, gateItems.size(), "Exactly one SAR_FILING gate WorkItem");
    assertEquals(AmlGroups.MLRO, gateItems.get(0).candidateGroups,
            "candidateGroups must be aml-mlro (SAR_FILING type)");
    workItemService.completeFromSystem(gateItems.get(0).id, "test-mlro", "approved");
}
```

- [ ] **Step 2: Update the full trust routing flow test**

In `full_trust_routing_flow_corporate_case`, insert the gate approval after starting the investigation, BEFORE the existing completion poll:

```java
// Step 1: Start investigation (same as before)
// ...
final UUID caseId = UUID.fromString(caseIdStr);

// Step 2: Await and approve MLRO gate (NEW)
awaitAndApproveGate(caseId);

// Step 3: Poll until completed (same as before)
Awaitility.await()
        .atMost(30, TimeUnit.SECONDS)
        // ...
```

Renumber the subsequent step comments (Step 3→4, Step 4→5, Step 5→6).

- [ ] **Step 3: Add gate infrastructure to AmlLayer6ResourceTest**

Add these imports and fields to `AmlLayer6ResourceTest.java`:

```java
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.service.WorkItemService;
import io.quarkus.narayana.jta.QuarkusTransaction;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import java.util.List;
import java.util.UUID;
```

Add fields:

```java
@PersistenceContext
EntityManager defaultEm;

@Inject
WorkItemService workItemService;
```

Add helper methods (same pattern):

```java
private List<WorkItem> findGateWorkItems(final UUID caseId) {
    return QuarkusTransaction.requiringNew().call(() ->
        defaultEm.createQuery(
            "SELECT w FROM WorkItem w WHERE w.callerRef LIKE :pattern",
            WorkItem.class)
            .setParameter("pattern", "case:" + caseId + "/gate:%")
            .getResultList());
}

private void awaitAndApproveGate(final UUID caseId) {
    Awaitility.await()
            .atMost(15, TimeUnit.SECONDS)
            .pollInterval(300, TimeUnit.MILLISECONDS)
            .until(() -> !findGateWorkItems(caseId).isEmpty());
    final WorkItem gate = findGateWorkItems(caseId).get(0);
    workItemService.completeFromSystem(gate.id, "test-mlro", "approved");
}
```

- [ ] **Step 4: Update post_investigate_returns_202_with_caseId**

After the POST, before the drain poll:

```java
final UUID caseId = UUID.fromString(caseIdStr);
awaitAndApproveGate(caseId);
```

- [ ] **Step 5: Update get_investigation_returns_completed_with_routing_decisions**

After the POST, before the completion poll:

```java
final UUID caseId = UUID.fromString(caseIdStr);
awaitAndApproveGate(caseId);
```

- [ ] **Step 6: Update get_investigation_returns_completed_after_cache_eviction**

After the POST, before the completion poll:

```java
final UUID caseId = UUID.fromString(caseIdStr);
awaitAndApproveGate(caseId);
```

- [ ] **Step 7: Update post_outcome_returns_204**

After the POST, before the completion poll:

```java
final UUID caseId = UUID.fromString(caseIdStr);
awaitAndApproveGate(caseId);
```

- [ ] **Step 8: Run full test suite**

Run: `mvn -pl app -am test -Dsurefire.failIfNoSpecifiedTests=false`

Expected: PASS — all 183+ tests green (some new tests added in Tasks 1–3).

- [ ] **Step 9: Commit**

```bash
git add app/src/test/java/io/casehub/aml/engine/AmlLayer6InvestigationTest.java app/src/test/java/io/casehub/aml/engine/AmlLayer6ResourceTest.java
git commit -m "test: add MLRO gate approval step to Layer 6 integration tests

Refs #58"
```

---

### Task 5: ARC42STORIES.MD documentation

**Files:**
- Modify: `ARC42STORIES.MD`

**Interfaces:**
- Consumes: All changes from Tasks 1–4
- Produces: Updated architecture record reflecting the new worker count, mixed execution model, resolved gaps

- [ ] **Step 1: Update line 203 — capability and binding counts**

Replace:
```
"5 capabilities · 5 contextChange bindings · 1 goal"
```
With:
```
"6 capabilities · 7 contextChange bindings · 1 goal"
```

- [ ] **Step 2: Update line 204 — worker delegation count**

Replace:
```
"delegates 7 worker lambdas to AmlInvestigationCaseDescriptor"
```
With:
```
"delegates 8 worker lambdas to AmlInvestigationCaseDescriptor"
```

- [ ] **Step 3: Update line 1462 — technical debt entry**

Replace:
```
| Raw worker lambdas in `AmlInvestigationCaseDescriptor` (SAR workers) | ✅ Resolved — all 7 workers use `FlowWorkerFunction` (aml#46 + aml#66). `AmlOversightCaseHub`: 2/3 migrated (aml#67); `entityLinkProposalWorker` blocked on engine#564. Worker primitives migrated to `io.casehub.worker.api` (aml#70). | L5 | casehubio/aml#67 ✅ |
```
With:
```
| Raw worker lambdas in `AmlInvestigationCaseDescriptor` (SAR workers) | 6 of 8 workers use `FlowWorkerFunction`. SAR-drafting workers (junior + senior) use `WorkerFunction.Sync` for `PlannedAction(SAR_FILING)` support (engine#564 blocks Flow support). `AmlOversightCaseHub`: 2/3 migrated (aml#67); `entityLinkProposalWorker` also Sync per engine#564. Worker primitives migrated to `io.casehub.worker.api` (aml#70). | L5 | casehubio/aml#58, #67 ✅ |
```

- [ ] **Step 4: Update line 1464 — resolve stale SLA gap**

Replace:
```
| SLA `GAP` on engine async path | `sar-drafting` worker does not write `COMPLIANCE_REVIEW_OPENED` ledger entry; SLA evidence builder has no WorkItem ID | L7 | Deferred |
```
With:
```
| SLA `GAP` on engine async path | ✅ Resolved — `ComplianceReviewLifecycle.openReview()` consolidated WorkItem creation + ledger write (aml#56); `compliance-review-opening` worker calls `openReview()` post-MLRO gate (aml#58) | L7 | casehubio/aml#56 ✅, #58 ✅ |
```

- [ ] **Step 5: Remove line 1504 — resolved gap**

Delete the line:
```
- sar-drafting workers still call `openReview()` unconditionally — correct design separates drafting (PlannedAction) from compliance-review-opening (post-MLRO). aml#58.
```

- [ ] **Step 6: Update line 1522 — glossary worker count**

Replace:
```
| `AmlInvestigationCaseDescriptor` | Plain POJO (no CDI) carrying all 7 AML investigation worker lambdas
```
With:
```
| `AmlInvestigationCaseDescriptor` | Plain POJO (no CDI) carrying all 8 AML investigation worker lambdas
```

- [ ] **Step 7: Commit**

```bash
git add ARC42STORIES.MD
git commit -m "docs: update ARC42STORIES.MD — #58 worker split, resolved gaps, counts

Refs #58"
```
