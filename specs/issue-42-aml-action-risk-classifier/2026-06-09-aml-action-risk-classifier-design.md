# Design: AmlActionRiskClassifier — Layer 9 Oversight Gate

**Issue:** casehubio/aml#42 · **Also closes:** casehubio/aml#57  
**Branch:** issue-42-aml-action-risk-classifier  
**Date:** 2026-06-09 (revised post-review)

---

## Context

`casehub-engine#402` shipped `ActionRiskClassifier` — a platform-level SPI that lets any
worker declare a `PlannedAction` before the engine advances the case. The classifier returns
either `Autonomous` (proceed) or `GateRequired` (pause, create human approval WorkItem). The
engine chains all `@RiskClassifier`-qualified beans automatically; the most restrictive result
wins (GE-20260607-3b6711).

casehub-aml needs an AML-specific classifier that encodes FinCEN/FCA regulatory requirements:
certain investigation actions must always route to an MLRO or senior compliance director before
proceeding. This is not the same as the casehub-work compliance officer WorkItem (Layer 2) — it
is a pre-flight gate on the *engine step itself*, producing a machine-readable, ledgered approval
record before any consequential state change.

---

## Key Design Decision: Layer 9 as a Dedicated Oversight Harness

The existing Layer 1–8 investigation workers (`AmlInvestigationCaseHub`) do not declare
`PlannedAction` instances and are unchanged. The `AmlActionRiskClassifier` is registered in
CDI via `@RiskClassifier` and fires for any worker in any case hub that declares a matching
action type. A new Layer 9 case hub demonstrates the gate in a meaningful, isolated context.

**Consequences:**
- Zero changes to existing tests — no bypass classifiers needed
- Gate is live in production from day one for any future worker that declares a PlannedAction
- Layer 9 is self-contained: new YAML, new workers, new coordinator, new endpoint, new tests

---

## §1 — AmlActionType (api module)

**File:** `api/src/main/java/io/casehub/aml/domain/AmlActionType.java`

Enum encoding the full AML consequential action vocabulary. Each constant encodes:
`GatePolicy`, `reversible`, `candidateGroups`, reason string. No classification logic
lives in the classifier itself — it reads data from the action type.

### Inner enum GatePolicy

```java
public enum GatePolicy {
    ALWAYS,              // unconditional gate regardless of context
    RISK_SCORE_THRESHOLD, // gate when context["riskScore"] >= threshold OR entityType == "PEP"
    CONFIDENCE_THRESHOLD  // gate when context["confidenceScore"] < threshold (low confidence)
}
```

### Action type table

```
Constant                 GatePolicy               reversible  candidateGroups
──────────────────────── ──────────────────────── ─────────── ───────────────────────────
SAR_FILING               ALWAYS                   false       ["aml-mlro"]
ACCOUNT_RESTRICTION      RISK_SCORE_THRESHOLD      true        ["aml-compliance"]
TRANSACTION_BLOCKING     CONFIDENCE_THRESHOLD      false       ["aml-compliance"]
ENTITY_LINK_CREATION     RISK_SCORE_THRESHOLD      true        ["aml-compliance"]
LAW_ENFORCEMENT_REFERRAL ALWAYS                   false       ["aml-senior-compliance"]
```

Each constant also carries:
- `reason()` — human-readable reason string for the gate WorkItem title
- `scope()` — `"casehubio/aml/oversight"` for all types (engine#437 tracks mapping semantics)
- `expiresIn()` — `null` for all types; expiry policy is regulatory and configurable later

**`candidateGroups` note (GE-20260607-326c7e):** fewer entries = more restrictive in the
engine chain. `["aml-mlro"]` (1 group) is the most restrictive gate in the system. The
compliance director group `["aml-senior-compliance"]` is also 1 entry but a narrower pool
by organisational convention — not by chain semantics. Documented here to prevent confusion.

In this deployment (single classifier, one PlannedAction per worker result), the chain's
tie-breaking logic (`narrower()`) never applies in practice. Documented for future
multi-classifier deployments.

### fromActionType implementation

```java
public static Optional<AmlActionType> fromActionType(String actionType) {
    if (actionType == null) return Optional.empty();
    return Arrays.stream(values())
        .filter(a -> a.actionType().equals(actionType))
        .findFirst();
}
```

**Do not use `valueOf()`** — it throws `IllegalArgumentException` on unknown strings;
`findFirst()` handles unknown types safely without exception semantics.

### actionType() format

`SAR_FILING` → `"sar.filing"`, `ENTITY_LINK_CREATION` → `"entity.link.creation"` etc.
(lowercase, underscores replaced with dots).

### AmlGroups companion

**File:** `api/src/main/java/io/casehub/aml/domain/AmlGroups.java`

Constants: `MLRO = "aml-mlro"`, `AML_COMPLIANCE = "aml-compliance"`,
`AML_SENIOR_COMPLIANCE = "aml-senior-compliance"`.

---

## §2 — AmlActionRiskClassifier (app/routing)

**File:** `app/src/main/java/io/casehub/aml/routing/AmlActionRiskClassifier.java`

```
@ApplicationScoped
@RiskClassifier
public class AmlActionRiskClassifier implements ActionRiskClassifier
```

**Discovery:** automatic — `ChainedReactiveActionRiskClassifier` in the engine runtime
injects `@RiskClassifier Instance<ActionRiskClassifier>` and chains all registered beans.

### Classification flow

1. Null-check `action.actionType()` — if null: `Autonomous`
2. `AmlActionType.fromActionType(action.actionType())` → `Optional<AmlActionType>`
3. If empty (unknown type): `Autonomous` — classifier does not gate unrecognised types
4. Dispatch to per-GatePolicy method reading from `action.context()`:
   - `ALWAYS` → `GateRequired` unconditionally
   - `RISK_SCORE_THRESHOLD` → read `riskScore` (Double) and `entityType` (String)
   - `CONFIDENCE_THRESHOLD` → read `confidenceScore` (Double)
5. **If context key is missing or unparseable for a known type: `GateRequired` with
   reason `"Risk assessment unavailable — human review required"`, using the action
   type's `candidateGroups`, `reversible`, and `scope`.** This is fail-closed:
   unknown risk on a recognized consequential action type is grounds for gating.
   (The engine's fail-safe covers classifier *exceptions*; missing-context is not an
   exception — it is an explicit classifier decision.)

### Thresholds

```java
private static final double RISK_SCORE_GATE_THRESHOLD = 0.8;
private static final double CONFIDENCE_GATE_THRESHOLD = 0.9;
```

Hardcoded constants — compliance thresholds are regulatory, not user preferences.
No `PreferenceProvider` dependency.

### GateRequired construction

Classifier reads `type.reason()`, `type.candidateGroups()`, `type.reversible()`,
`type.scope()` from `AmlActionType` and constructs `GateRequired` directly.
`AmlActionType` owns the data; classifier owns the condition check and record construction.
`AmlActionType` stays pure Java with no `casehub-engine-api` dependency.

---

## §3 — New pom.xml dependency: casehub-engine-work-adapter

**This is a blocker, not a note.** `casehub-engine-work-adapter` is NOT currently in the
AML dependency graph — confirmed via Maven dependency tree. It is not declared in `pom.xml`
and is not reachable transitively (the engine runtime depends on `work-adapter`, not the
reverse). The comment in `application.properties` ("engine-work-adapter comes in
transitively") is wrong and must be corrected.

Without `work-adapter`:
- `ActionGateWorkItemHandler` (creates WorkItem when gate fires) is absent
- `WorkItemLifecycleAdapter` (resumes case after WorkItem approval) is absent
- When any `@RiskClassifier` bean returns `GateRequired`, the `ActionGateScheduleEvent`
  fires with no consumer → event dropped → case stalls indefinitely
- `ActionGateDeploymentHealthCheck` (already in engine runtime) will log this at startup

**Required pom.xml additions (app module):**

```xml
<!-- Layer 9: oversight gate mechanism — ActionGateWorkItemHandler + WorkItemLifecycleAdapter -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-work-adapter</artifactId>
  <version>${casehub.version}</version>
</dependency>
<!-- Transitive of work-adapter; also needs Jandex for CDI scanning -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-engine-blackboard</artifactId>
  <version>${casehub.version}</version>
</dependency>
```

**Required Jandex entries — test application.properties:**

```properties
quarkus.index-dependency.engine-work-adapter.group-id=io.casehub
quarkus.index-dependency.engine-work-adapter.artifact-id=casehub-engine-work-adapter
quarkus.index-dependency.engine-blackboard.group-id=io.casehub
quarkus.index-dependency.engine-blackboard.artifact-id=casehub-engine-blackboard
```

**Required Jandex entries — production application.properties (under `%prod`):**
Same two entries, prefixed with `%prod.`.

**Correct the wrong comment** in main `application.properties`:
Remove "engine-work-adapter comes in transitively" — it does not.

---

## §4 — Layer 9 Oversight Harness

### YAML case definition

**File:** `app/src/main/resources/aml/aml-oversight-investigation.yaml`

```yaml
dsl: "0.1"
version: "1.0.0"
name: aml-oversight-investigation
namespace: casehub-aml
title: AML Oversight Gate — entity network link confirmation

spec:

  capabilities:
    - name: entity-resolution
      description: "Identify entities and risk profile from suspicious transaction"
      inputSchema: "{ transaction: .transaction }"
      outputSchema: "{ entityResolution: . }"

    - name: entity-link-proposal
      description: "Propose adding entity network link to investigation graph"
      inputSchema: "{ entityResolution: .entityResolution }"
      outputSchema: "{ entityLinkProposal: . }"

    - name: investigation-summary
      description: "Summarise investigation findings after entity link is confirmed"
      inputSchema: "{ entityResolution: .entityResolution, entityLinkProposal: .entityLinkProposal }"
      outputSchema: "{ investigationSummary: . }"

  goals:
    - name: investigation-complete
      kind: success
      condition: ".investigationSummary != null"

  completion:
    success:
      allOf:
        - investigation-complete

  bindings:
    - name: entity-resolution
      on: { contextChange: {} }
      when: ".transaction != null and .entityResolution == null"
      capability: entity-resolution

    - name: entity-link-proposal
      on: { contextChange: {} }
      when: ".entityResolution != null and .entityLinkProposal == null"
      capability: entity-link-proposal

    - name: investigation-summary
      on: { contextChange: {} }
      when: ".entityLinkProposal != null and .investigationSummary != null"
      capability: investigation-summary
```

Wait — **correction to the investigation-summary binding:** the `when:` should check that
`entityLinkProposal != null` (gate was approved and output was applied) and
`investigationSummary == null` (not yet run). The condition as written above has a bug.
Correct:

```yaml
    - name: investigation-summary
      on: { contextChange: {} }
      when: ".entityLinkProposal != null and .investigationSummary == null"
      capability: investigation-summary
```

**Gate semantics in this YAML:** the `entity-link-proposal` worker returns
`WorkerResult(output, PlannedAction(ENTITY_LINK_CREATION, context))`. The engine calls the
classifier. If `GateRequired` → output is NOT committed to case context; case pauses.
When gate WorkItem is approved → engine re-fires; output IS committed; `entityLinkProposal`
key appears in context → `investigation-summary` binding fires → goal satisfied → completed.

**Why ENTITY_LINK_CREATION as the demonstrated action:**
- Produces both Autonomous (low-risk CORPORATE, riskScore 0.35) and GateRequired (PEP,
  riskScore 0.87) paths from the same worker with different input data — enabling a clean
  two-test suite with no test profile switching
- First consequential decision in a real AML investigation (entity graph validation
  precedes everything else)
- Requires no restructuring of existing sar-drafting or compliance review workers

### Workers in AmlOversightCaseHub

**File:** `app/src/main/java/io/casehub/aml/engine/AmlOversightCaseHub.java`

`YamlCaseHub("aml/aml-oversight-investigation.yaml")` — augments with three workers:

**`entityResolutionWorker`** — same logic as the existing Layer 5 worker. Returns
`{entityType, riskScore, ownershipChain}` based on `flagReason` containing "PEP".

**`entityLinkProposalWorker`** — reads `entityResolution` from input, builds context map
`{entityType, riskScore, ownershipChain}`, returns:
```java
WorkerResult.of(
    Map.of("proposedLink", entityType + " → " + ownershipChain,
           "entityType", entityType,
           "riskScore", riskScore),
    PlannedAction.of(
        "Entity network link proposed: " + entityType,
        AmlActionType.ENTITY_LINK_CREATION.actionType(),
        Map.of("entityType", entityType,
               "riskScore", riskScore,
               "ownershipChain", ownershipChain)));
```

The classifier reads `entityType` and `riskScore` from the PlannedAction context:
- PEP or riskScore ≥ 0.8 → `GateRequired`
- Otherwise → `Autonomous`

**`investigationSummaryWorker`** — returns a summary map with no side effects.

### AmlOversightCoordinator

**File:** `app/src/main/java/io/casehub/aml/engine/AmlOversightCoordinator.java`

Minimal coordinator — injects `AmlOversightCaseHub`, calls `startCase(initialContext)`,
returns caseId. No memory query (no prior context for oversight investigations), no ledger
write (the gate WorkItem is the audit artefact for Layer 9). Same blocking `get(5, SECONDS)`
pattern as `AmlEngineCoordinator`.

### AmlLayer9Resource

**File:** `app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java`

```
POST /api/layer9/investigations          → {caseId}
GET  /api/layer9/investigations/{caseId} → {status: "in-progress"|"completed"}
```

**Completion determination:** inject `AmlWorkerDecisionRepository` (qhorus PU); check
`findLatestByCaseIdAndCapability(caseId, "investigation-summary")`. Present → "completed";
absent → "in-progress". Same pattern as `AmlLayer6Resource` (which checks for
"sar-drafting"). Mirrors the pattern — not a new abstraction.

---

## §5 — Testing

### Unit tests (api module, pure JUnit 5)

`AmlActionTypeTest`:
- `fromActionType` round-trip for all five constants — `fromActionType(type.actionType())` returns the same constant
- `fromActionType(null)` → `Optional.empty()`
- `fromActionType("unknown.type")` → `Optional.empty()` (no exception)
- `gatePolicy()`, `reversible()`, `candidateGroups()`, `reason()` for each constant

`AmlGroupsTest` — constants are non-null and non-blank.

### Unit tests (app module, Mockito)

`AmlActionRiskClassifierTest` — instantiated directly with `new AmlActionRiskClassifier()`:

| Scenario | Expected |
|---|---|
| SAR_FILING with any context | GateRequired, groups=["aml-mlro"], reversible=false |
| LAW_ENFORCEMENT_REFERRAL | GateRequired, groups=["aml-senior-compliance"] |
| ENTITY_LINK_CREATION, riskScore=0.35, entityType="CORPORATE" | Autonomous |
| ENTITY_LINK_CREATION, riskScore=0.87, entityType="PEP" | GateRequired |
| ENTITY_LINK_CREATION, riskScore=0.87, entityType="CORPORATE" | GateRequired (score above threshold) |
| ENTITY_LINK_CREATION, riskScore=0.35, entityType="PEP" | GateRequired (PEP overrides score) |
| ENTITY_LINK_CREATION at threshold boundary (riskScore=0.8) | GateRequired |
| TRANSACTION_BLOCKING, confidenceScore=0.95 | Autonomous |
| TRANSACTION_BLOCKING, confidenceScore=0.7 | GateRequired |
| ENTITY_LINK_CREATION, missing riskScore key | GateRequired (fail-closed) |
| ACCOUNT_RESTRICTION, missing riskScore key | GateRequired (fail-closed) |
| TRANSACTION_BLOCKING, missing confidenceScore key | GateRequired (fail-closed) |
| Unknown actionType "foo.bar" | Autonomous |
| null actionType | Autonomous |

Mockito note (GE-20260607-ab9f37): no shared stubs needed here — the classifier has no
injected dependencies, so Mockito is not needed. Pure `new` instantiation is correct.

### @QuarkusTest (app module)

**`AmlLayer9ActionGateTest`** — two test methods:

**`gate_fires_for_pep_entity`:**
1. `POST /api/layer9/investigations` with transaction whose `flagReason` contains "PEP"
   → entity-resolution returns `{entityType: "PEP", riskScore: 0.87}`
2. **Await gate WorkItem** — poll default-PU `EntityManager` (unitName = none):
   ```java
   @PersistenceContext  // no unitName — WorkItem is on the default datasource
   EntityManager em;

   Awaitility.await().atMost(15, SECONDS).until(() ->
       !em.createQuery(
           "SELECT w FROM WorkItem w WHERE w.callerRef LIKE :pattern",
           io.casehub.work.runtime.model.WorkItem.class)
         .setParameter("pattern", "case:" + caseId + "/gate:%")
         .getResultList().isEmpty());
   ```
   This is the positive signal that the gate fired. The awaitility poll runs on the test
   thread without an explicit `@Transactional` boundary — this matches the existing pattern
   in `AmlLayer6InvestigationTest` which queries `LedgerAttestation` outside a transaction.
   Quarkus `@QuarkusTest` maintains a CDI request context, so `EntityManager` is accessible.
3. Load the gate WorkItem, assert `candidateGroups = "aml-compliance"`
4. Assert `GET /api/layer9/investigations/{id}` returns `status = "in-progress"` (gate paused the engine)
5. Approve: `workItemService.completeFromSystem(workItem.id, "test-aml-compliance", "approved")`
   → fires `WorkItemLifecycleEvent(COMPLETED)` → `WorkItemLifecycleAdapter` routes to
   `ActionGateCompletionApplier` → engine resumes → `entityLinkProposal` committed →
   `investigation-summary` fires → goal satisfied
6. Await `status = "completed"` via `GET /api/layer9/investigations/{id}`

**`gate_not_fired_for_low_risk_corporate`:**
1. `POST` with CORPORATE flagReason → entity-resolution returns `{riskScore: 0.35, entityType: "CORPORATE"}`
2. Await `status = "completed"` (classifier returned Autonomous, gate never fired)
3. Assert no gate WorkItem exists for this caseId: same JPQL query on default-PU EM → empty result

**`completeFromSystem` signature (confirmed from decompiled `casehub-work`):**
`WorkItemService.completeFromSystem(UUID id, String actorId, String resolution)`

Both test methods must drain to `status = completed` before returning (protocol
PP-20260604-820c35). `casehub.ledger.hash-chain.enabled=false` already in test properties
(protocol PP-20260604-f45c95) — no new test properties needed beyond the Jandex entries.

---

## §6 — Issue #57: Partial Unique Index

**File:** `docs/sql/V2011__UQ_TRUST_ATTEST_CASE_CAP_RECONSTRUCTED.sql`

```sql
-- PostgreSQL only — H2 does not support partial unique indexes even in MODE=PostgreSQL.
-- Apply manually on the production PostgreSQL database after V2009 has run.
-- Do NOT place in db/migration/ paths — Flyway will fail on H2.
CREATE UNIQUE INDEX IF NOT EXISTS UQ_TRUST_ATTEST_CASE_CAP_RECONSTRUCTED
    ON aml_trust_routing_attestation (investigation_case_id, capability_tag)
    WHERE reconstructed = TRUE;
```

Commit closes casehubio/aml#57. No Flyway migration. No code changes.

---

## §7 — Deferred: sar-drafting design flaw

The current `sarDraftingWorkerJunior` / `sarDraftingWorkerSenior` call
`complianceReviewLifecycle.openReview()` unconditionally as a side effect inside the worker.
The correct design separates pure SAR drafting from the consequential filing step — with
`PlannedAction(SAR_FILING)` as the gate between them, and a `compliance-review-opening`
worker that only calls `openReview()` after MLRO approval.

This restructuring is out of scope for #42. **File as a new GitHub issue before closing
this branch.** Reference #42 as context.

---

## Coherence check

- SPI placement: `AmlActionRiskClassifier` in `app/` (has CDI, Quarkus deps) ✓
- `AmlActionType` + `AmlGroups` in `api/` (pure Java, no deps, no engine-api dep) ✓
- `GatePolicy` as inner enum of `AmlActionType` in `api/` ✓
- `fromActionType` uses stream, not valueOf() — safe on unknown strings ✓
- Platform boundary: application-tier classifier, no domain knowledge in foundation ✓
- `@RiskClassifier @ApplicationScoped` on classifier: matches engine CDI discovery ✓
- candidateGroups fewer-is-more-restrictive: documented in AmlActionType ✓
- Fail-closed on missing context for known types ✓
- `casehub-engine-work-adapter` + `casehub-engine-blackboard` added as compile deps ✓
- Jandex entries for both in test and production application.properties ✓
- Wrong "comes in transitively" comment corrected ✓
- YAML has full goal/binding/completion blocks with explicit when: conditions ✓
- YAML output keys (`entityLinkProposal`, `investigationSummary`) explicit ✓
- GET completion check uses `CaseInstanceCache` (engine state) rather than `AmlWorkerDecisionRepository` — resilient to concurrent H2 `ledger_subject_sequence` INSERT race that causes `WorkerDecisionEntry` saves to fail under casehub-work SNAPSHOT regression (aml#59) ✓
- `AmlOversightCoordinator` listed as required class ✓
- Gate WorkItem poll uses default-PU EntityManager (WorkItem on default datasource) ✓
- Layer 9 isolation: existing tests unaffected ✓
- Both Autonomous and GateRequired paths covered in @QuarkusTest ✓
- #57 closed as SQL-only, no code risk ✓
