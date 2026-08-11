# SAR Worker Flow Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate `sarDraftingWorkerJunior` and `sarDraftingWorkerSenior` in `AmlInvestigationCaseDescriptor` from `WorkerFunction.Sync` (raw lambda) to `WorkerFunction.Flow` (FuncWorkflowBuilder), completing PP-20260531 compliance for all 7 investigation workers.

**Architecture:** Both workers wrap their existing lambda bodies in `workflow("name").tasks(function(s -> { ... }, Map.class)).build()`. The `WorkerResult.of(...)` wrapper is removed — `FlowWorkerExecutor` wraps the `Map` return internally. `WorkerExecutionContext.current().caseId()` is now safe in the flow path after engine#559. Instance fields (`objectMapper`, `complianceReviewLifecycle`) are captured by closure; both methods remain `private Worker` (non-static).

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine 0.2-SNAPSHOT (engine#559 required), FuncWorkflowBuilder / FuncDSL (serverlessworkflow-experimental-fluent-func 7.13.4.Final)

---

## File Map

| File | Repo | Change |
|------|------|--------|
| `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptor.java` | project | Migrate 2 workers; update 3 Javadoc locations |
| `app/src/test/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptorTest.java` | project | Collapse two-set classification to one unified set; update comment |
| `/Users/mdproctor/claude/public/casehub/aml/CLAUDE.md` | workspace | Update misleading `Worker.Builder.function()` return type note |
| `/Users/mdproctor/claude/casehub/aml/ARC42STORIES.MD` | project | Update 3 stale references (lines 1050, 1060, 1462) — after issues filed |
| `/Users/mdproctor/.hortora/garden/jvm/casehub-engine/GE-20260609-ddd4b8.md` | garden | Revise ⚠️ caveat to ✅ resolved |

---

## Task 1: Pre-requisite — Force-refresh engine SNAPSHOT

**Files:** none modified

- [ ] **Step 1.1: Force-refresh the engine SNAPSHOT**

```bash
mvn -U dependency:resolve -pl app -am -q
```

Expected: Maven resolves without error. If the updated SNAPSHOT (containing engine#559) is not yet published, wait for CI before continuing to Task 4. The unit tests in Tasks 2–3 do not depend on the SNAPSHOT version, but `AmlLayer6InvestigationTest` in Task 4 does.

---

## Task 2: Red — update unit test to classify all workers as Flow

**Files:**
- Modify: `app/src/test/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptorTest.java`

- [ ] **Step 2.1: Replace `worker_execution_model_classification_is_exhaustive` with unified Flow assertion**

In `AmlInvestigationCaseDescriptorTest`, replace the entire `worker_execution_model_classification_is_exhaustive` method:

```java
@Test
void worker_execution_model_classification_is_exhaustive() {
    // All workers use WorkerFunction.Flow (FuncWorkflowBuilder) per PP-20260531-worker-func-exec.
    // Any new worker must be explicitly classified here — this prevents silent omissions.
    final Set<String> allWorkers = Set.of(
            "entity-resolution-agent",
            "pattern-analysis-agent",
            "osint-screening-agent",
            "osint-screening-agent-senior",
            "senior-analyst-agent",
            "sar-drafting-agent-junior",
            "sar-drafting-agent-senior");

    for (final Worker w : descriptor.workers()) {
        if (allWorkers.contains(w.getName())) {
            assertInstanceOf(WorkerFunction.Flow.class, w.getFunction(),
                    "Worker " + w.getName() + " must use WorkerFunction.Flow (FuncWorkflowBuilder).");
        } else {
            fail("Worker " + w.getName() + " is unclassified — add it to allWorkers in this test.");
        }
    }
}
```

- [ ] **Step 2.2: Run the test to confirm it fails**

```bash
mvn test -pl app -am -Dtest=AmlInvestigationCaseDescriptorTest#worker_execution_model_classification_is_exhaustive -Dsurefire.failIfNoSpecifiedTests=false
```

Expected output: `FAIL` with:
```
expected: an instance of class io.casehub.api.model.WorkerFunction$Flow
 but was: an instance of class io.casehub.api.model.WorkerFunction$Sync
```
(for `sar-drafting-agent-junior` or `sar-drafting-agent-senior`)

- [ ] **Step 2.3: Commit the failing test**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/test/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptorTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "test(#66): assert all workers use WorkerFunction.Flow — red for SAR workers"
```

---

## Task 3: Green — migrate both SAR workers + update Javadoc

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptor.java`

- [ ] **Step 3.1: Replace the class-level Javadoc paragraph**

Find and replace this exact paragraph in the class Javadoc:

```java
 * <p>Pure-computation workers use {@code FuncWorkflowBuilder.workflow().tasks(function(...)).build()}
 * per protocol PP-20260531-worker-func-exec. SAR drafting workers remain as
 * {@code WorkerFunction.Sync} pending engine support for {@link WorkerExecutionContext} in
 * the flow execution path (see #66).
```

Replace with:

```java
 * <p>All workers use {@code FuncWorkflowBuilder.workflow().tasks(function(...)).build()}
 * per protocol PP-20260531-worker-func-exec.
```

- [ ] **Step 3.2: Replace the full `sarDraftingWorkerJunior` method**

Replace the entire `sarDraftingWorkerJunior` method (Javadoc + body) with:

```java
    /**
     * Junior SAR drafting worker — minimal narrative, suitable for routine cases.
     * Opens the compliance officer WorkItem (Layer 2 — 30-day FinCEN SLA).
     * Runs on a Quartz worker thread; JPA calls via ComplianceReviewLifecycle are safe here.
     */
    private Worker sarDraftingWorkerJunior() {
        return Worker.builder()
                .name("sar-drafting-agent-junior")
                .capabilities(List.of(cap("sar-drafting")))
                .function(
                    workflow("sar-drafting-junior")
                        .tasks(
                            function(s -> {
                                @SuppressWarnings("unchecked")
                                final Map<String, Object> input = (Map<String, Object>) s;
                                @SuppressWarnings("unchecked")
                                final Map<String, Object> txMap = (Map<String, Object>) input.get("transaction");
                                final SuspiciousTransaction tx =
                                        objectMapper.convertValue(txMap, SuspiciousTransaction.class);
                                @SuppressWarnings("unchecked")
                                final Map<String, Object> osintMap =
                                        (Map<String, Object>) input.get("osintScreening");
                                final boolean osintDeclined = osintMap != null
                                        && Boolean.TRUE.equals(osintMap.get("declined"));
                                final String sarNarrative = "SAR filed for transaction " + tx.id()
                                        + ". Amount: " + tx.amount() + " " + tx.currency()
                                        + (osintDeclined ? " OSINT screening declined." : "");
                                final UUID caseId = WorkerExecutionContext.current().caseId();
                                final String complianceTaskId =
                                        complianceReviewLifecycle.openReview(tx, buildSummary(input, tx, sarNarrative), caseId);
                                return Map.of("sarNarrative", sarNarrative, "complianceTaskId", complianceTaskId);
                            }, Map.class))
                        .build())
                .build();
    }
```

- [ ] **Step 3.3: Replace the full `sarDraftingWorkerSenior` method**

Replace the entire `sarDraftingWorkerSenior` method (Javadoc + body) with:

```java
    /**
     * Senior SAR drafting worker — full narrative including entity type and flag reason.
     * Used for complex or PEP cases routed via trust-weighted selection.
     * Opens the compliance officer WorkItem (Layer 2 — 30-day FinCEN SLA).
     */
    private Worker sarDraftingWorkerSenior() {
        return Worker.builder()
                .name("sar-drafting-agent-senior")
                .capabilities(List.of(cap("sar-drafting")))
                .function(
                    workflow("sar-drafting-senior")
                        .tasks(
                            function(s -> {
                                @SuppressWarnings("unchecked")
                                final Map<String, Object> input = (Map<String, Object>) s;
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
                                final UUID caseId = WorkerExecutionContext.current().caseId();
                                final String complianceTaskId =
                                        complianceReviewLifecycle.openReview(tx, buildSummary(input, tx, sarNarrative), caseId);
                                return Map.of("sarNarrative", sarNarrative, "complianceTaskId", complianceTaskId);
                            }, Map.class))
                        .build())
                .build();
    }
```

- [ ] **Step 3.4: Run the unit test to confirm it passes**

```bash
mvn test -pl app -am -Dtest=AmlInvestigationCaseDescriptorTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected output: `BUILD SUCCESS`, all 5 tests pass including `worker_execution_model_classification_is_exhaustive`.

- [ ] **Step 3.5: Commit the migration**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptor.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#66): migrate SAR drafting workers to FuncWorkflowBuilder (engine#559)"
```

---

## Task 4: Integration test validation

**Files:** none modified

- [ ] **Step 4.1: Run AmlLayer6InvestigationTest**

```bash
mvn test -pl app -am -Dtest=AmlLayer6InvestigationTest -Dsurefire.failIfNoSpecifiedTests=false
```

Expected output: `BUILD SUCCESS`.

If this fails with an Awaitility timeout (not an assertion error), the engine SNAPSHOT does not yet include engine#559. Wait for CI to publish a new SNAPSHOT, re-run Task 1, then retry this step. The failure chain when the SNAPSHOT is stale: `WorkerExecutionContext.current()` returns null → NPE in the SAR lambda → engine case fails → `status=completed` never seen → timeout.

Do NOT proceed to Task 5 until this passes.

---

## Task 5: Update CLAUDE.md return type note

**Files:**
- Modify: `/Users/mdproctor/claude/public/casehub/aml/CLAUDE.md`

- [ ] **Step 5.1: Replace the engine worker return type note**

Find this exact line at line 174 of `/Users/mdproctor/claude/public/casehub/aml/CLAUDE.md`:

```
> **Engine worker return type:** `Worker.Builder.function()` requires `Function<Map<String, Object>, WorkerResult>`. Return `WorkerResult.of(Map.of(...))` — not `Map.of(...)` directly. Applies to all `YamlCaseHub` worker lambdas (casehubio/aml#54).
```

Replace with:

```
> **Engine worker return type:** `WorkerFunction.Sync` (raw lambda) workers — e.g. `AmlOversightCaseHub` — must return `WorkerResult.of(Map.of(...))`. `WorkerFunction.Flow` (FuncWorkflowBuilder) workers return `Map<String, Object>` directly; `executeFlow` calls `model.asMap()` and wraps internally. Do not wrap Flow worker returns.
```

- [ ] **Step 5.2: Commit to workspace**

```bash
git -C /Users/mdproctor/claude/public/casehub/aml add CLAUDE.md
git -C /Users/mdproctor/claude/public/casehub/aml commit -m "docs(#66): update worker return type note — distinguish Sync vs Flow"
```

---

## Task 6: Update garden entry GE-20260609-ddd4b8

**Files:**
- Modify: `/Users/mdproctor/.hortora/garden/jvm/casehub-engine/GE-20260609-ddd4b8.md`

- [ ] **Step 6.1: Replace the ⚠️ caveat with a ✅ resolved note**

Find this exact paragraph at the end of the Fix section:

```
**⚠️ Caveat — `WorkerFunction.Flow` (FuncWorkflowBuilder) workers:** `WorkerExecutionContext.current()` returns `null` inside FuncDSL lambdas. `DefaultWorkerExecutor.executeFlow()` does not call `WorkerExecutionContext.set()` before delegating to `FlowWorkerExecutor` — only `executeSync()` and `executeAgentSync()` set it. Workers migrated to `FuncWorkflowBuilder` that call `WorkerExecutionContext.current()` will get NPE at runtime. Either keep such workers as `WorkerFunction.Sync`, or request the engine team to add `WorkerExecutionContext.set(context)` in `executeFlow()`. (casehub-aml#66)
```

Replace with:

```
**✅ Resolved (engine#559, 2026-06-23)** — `DefaultWorkerExecutor.executeFlow()` now calls `WorkerExecutionContext.set(context)` before delegating to `FlowWorkerExecutor`, matching the behaviour of `executeSync()`. `WorkerExecutionContext.current().caseId()` is safe inside FuncDSL lambdas. See casehubio/aml#66 for the AML migration that removed the Sync workaround.
```

- [ ] **Step 6.2: Commit to the garden repo**

The garden repo is at `/Users/mdproctor/.hortora/garden`. Commit to whichever branch is active (typically `main`):

```bash
git -C /Users/mdproctor/.hortora/garden add jvm/casehub-engine/GE-20260609-ddd4b8.md
git -C /Users/mdproctor/.hortora/garden commit -m "fix(GE-20260609-ddd4b8): mark WorkerExecutionContext flow-path gap as resolved (engine#559)"
```

---

## Task 7: File the two new issues (ordering constraint for Task 8)

**Files:** none modified locally

These issues must exist before Task 8 so their numbers can be written directly into ARC42STORIES.MD, avoiding a two-pass commit.

- [ ] **Step 7.1: File the AML oversight worker migration issue**

```bash
gh issue create --repo casehubio/aml \
  --title "feat: migrate entityResolutionWorker + investigationSummaryWorker in AmlOversightCaseHub to FuncWorkflowBuilder" \
  --label "enhancement" \
  --body "$(cat <<'EOF'
PP-20260531 requires all production workers to use FuncWorkflowBuilder (WorkerFunction.Flow).
AmlOversightCaseHub has 3 workers; 2 are unblocked:

| Worker | Method | Return type | Notes |
|--------|--------|-------------|-------|
| oversight-entity-resolution-agent | entityResolutionWorker() | WorkerResult.of(Map) — single arg | private static Worker, static qualifier retained |
| oversight-investigation-summary-agent | investigationSummaryWorker() | WorkerResult.of(Map) — single arg | private static Worker, static qualifier retained |
| oversight-entity-link-proposal-agent | entityLinkProposalWorker() | WorkerResult.of(Map, PlannedAction) | Blocked — see engine issue for PlannedAction support in FlowWorkerExecutor |

Migration pattern is identical to the 7 workers migrated in aml#46 and aml#66.
Static qualifier is retained — neither worker captures instance state.

Refs #66
EOF
)"
```

Note the issue number returned (call it `AML_ISSUE_N`).

- [ ] **Step 7.2: File the engine PlannedAction support issue**

```bash
gh issue create --repo casehubio/engine \
  --title "feat: add PlannedAction support to FlowWorkerExecutor / FuncDSL" \
  --label "enhancement" \
  --body "$(cat <<'EOF'
## Problem

FlowWorkerExecutor wraps the Map output of a FuncDSL workflow as WorkerResult internally
(via model.asMap()). There is no mechanism to attach a PlannedAction to the WorkerResult
in the flow path. Workers that need to declare a PlannedAction (for ActionRiskClassifier
gate evaluation) cannot be migrated from WorkerFunction.Sync to FuncWorkflowBuilder — the
PlannedAction is silently lost.

## Blocked migration

casehubio/aml entityLinkProposalWorker in AmlOversightCaseHub returns:
WorkerResult.of(Map.of("proposedLink", ..., "entityType", ..., "riskScore", ...),
                PlannedAction.of("Entity network link proposed: " + entityType,
                                AmlActionType.ENTITY_LINK_CREATION.actionType(),
                                Map.of("entityType", ..., "riskScore", ..., "ownershipChain", ...)));

The PlannedAction metadata requires ownershipChain, which is in the task input
(entityResolution.ownershipChain) but NOT in the output map — so actionFn cannot
receive the output.

## Proposed API

Add withPlannedAction to FuncDSL's function task builder:

function(lambda, Map.class).withPlannedAction(actionFn)

where actionFn is Function<Map<String, Object>, PlannedAction> and receives
the TASK INPUT (same Map as the lambda's first argument s), not the output.
FlowWorkerExecutor includes the resulting PlannedAction in the wrapped WorkerResult.

For single-task workflows (current pattern), task input == workflow input.
For multi-step workflows, withPlannedAction receives the task-local input
(output of the prior step). If a future case needs the original workflow input
mid-chain, restructure to pass it as a named output rather than adding BiFunction.

Scale: S | Complexity: Med
EOF
)"
```

Note the issue number returned (call it `ENGINE_ISSUE_N`).

---

## Task 8: Update ARC42STORIES.MD (with real issue numbers)

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/aml/ARC42STORIES.MD`

Use `AML_ISSUE_N` and `ENGINE_ISSUE_N` from Task 7 in the edits below.

- [ ] **Step 8.1: Update line 1050 (Layer 5 descriptor bullet)**

Find this exact text at line 1050:

```
- `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptor.java` — plain POJO (no CDI); carries 5 `WorkerFunction.Flow` workers (FuncWorkflowBuilder, per PP-20260531) + 2 SAR drafting workers as `WorkerFunction.Sync` (pending `WorkerExecutionContext` support in `executeFlow` — aml#66); testable without Quarkus; per protocol PP-20260518
```

Replace with:

```
- `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptor.java` — plain POJO (no CDI); carries all 7 workers as `WorkerFunction.Flow` (FuncWorkflowBuilder per PP-20260531; SAR workers migrated in aml#66 after engine#559 fixed `WorkerExecutionContext` in `executeFlow`); testable without Quarkus; per protocol PP-20260518
```

- [ ] **Step 8.2: Update line 1060 (Layer 5 narrative)**

Find this exact text at line 1060:

```
Five pure-computation workers use `WorkerFunction.Flow` (FuncWorkflowBuilder per PP-20260531): entity-resolution, pattern-analysis, osint-screening (×2), senior-analyst-review. Two SAR drafting workers remain `WorkerFunction.Sync` — they call `WorkerExecutionContext.current().caseId()` to pass the case ID to `ComplianceReviewLifecycle.openReview()`, and `DefaultWorkerExecutor.executeFlow` does not set `WorkerExecutionContext` before invoking `FlowWorkerExecutor`. Migration tracked in aml#66.
```

Replace with (substitute real issue numbers for `AML_ISSUE_N` and `ENGINE_ISSUE_N`):

```
All 7 workers use `WorkerFunction.Flow` (FuncWorkflowBuilder per PP-20260531). SAR drafting workers migrated in aml#66 — engine#559 added `WorkerExecutionContext.set(context)` in `DefaultWorkerExecutor.executeFlow()`. `AmlOversightCaseHub` workers remain `WorkerFunction.Sync` pending engine PlannedAction support in flow path (aml#AML_ISSUE_N, engine#ENGINE_ISSUE_N).
```

- [ ] **Step 8.3: Update line 1462 (§12 Technical Debt table)**

Find this exact row at line 1462:

```
| Raw worker lambdas in `AmlInvestigationCaseDescriptor` (SAR workers) | **Partially resolved** — 5/7 workers migrated to `WorkerFunction.Flow` (FuncWorkflowBuilder) in aml#46 ✅; 2 SAR drafting workers remain `WorkerFunction.Sync` pending `WorkerExecutionContext` support in `executeFlow` (engine gap) | L5 | casehubio/aml#66 |
```

Replace with (substitute real issue numbers for `AML_ISSUE_N` and `ENGINE_ISSUE_N`):

```
| Raw worker lambdas in `AmlInvestigationCaseDescriptor` (SAR workers) | ✅ Resolved — all 7 workers use `WorkerFunction.Flow` (aml#46 + aml#66). `AmlOversightCaseHub` workers tracked separately (aml#AML_ISSUE_N, engine#ENGINE_ISSUE_N). | L5 | casehubio/aml#66 ✅ |
```

- [ ] **Step 8.4: Commit ARC42STORIES.MD**

```bash
git -C /Users/mdproctor/claude/casehub/aml add ARC42STORIES.MD
git -C /Users/mdproctor/claude/casehub/aml commit -m "docs(#66): sync ARC42STORIES.MD — all 7 workers now Flow, oversight tracked in aml#AML_ISSUE_N + engine#ENGINE_ISSUE_N"
```

(Replace `AML_ISSUE_N` and `ENGINE_ISSUE_N` with real numbers in the commit message too.)

---

## Self-Review

**Spec coverage check:**

| Spec section | Task |
|---|---|
| Pre-requisite: engine SNAPSHOT | Task 1 |
| §1 Migration (both workers) | Task 3 steps 3.2–3.3 |
| §2 Javadoc (class-level, junior, senior) | Task 3 steps 3.1–3.3 |
| §3 CLAUDE.md note | Task 5 |
| §4 Garden entry GE-20260609-ddd4b8 | Task 6 |
| §5 Unit test (red + green) | Tasks 2 + 3.4 |
| §5 Integration test validation | Task 4 |
| §6 ARC42STORIES.MD (3 locations) | Task 8 |
| §6 Ordering constraint (issues before ARC42) | Tasks 7 before 8 |
| §6 LAYER-LOG.md exclusion | Confirmed: no task needed |
| New issues to file | Task 7 |

All spec sections covered. No gaps.

**Placeholder scan:** No TBD, TODO, or vague steps. All code blocks contain exact production-ready content. Issue numbers in Task 8 are deliberately `AML_ISSUE_N` / `ENGINE_ISSUE_N` — these are runtime values from Task 7, not placeholders the plan can pre-fill.

**Type consistency:** `WorkerFunction.Flow`, `WorkerFunction.Sync`, `Map<String, Object>`, `WorkerResult.of()` — consistent across all tasks. `workflow()` and `function()` are static imports already present in the class.
