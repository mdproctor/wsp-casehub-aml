# Design: AmlActionRiskClassifier — Layer 9 Oversight Gate

**Issue:** casehubio/aml#42 · **Also closes:** casehubio/aml#57  
**Branch:** issue-42-aml-action-risk-classifier  
**Date:** 2026-06-09

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
- Layer 9 is self-contained: new YAML, new workers, new endpoint, new tests

---

## §1 — AmlActionType (api module)

**File:** `api/src/main/java/io/casehub/aml/domain/AmlActionType.java`

Enum encoding the full AML consequential action vocabulary. Each constant encodes:
`GatePolicy`, `reversible`, `candidateGroups`, reason template. No classification logic
lives in the classifier itself — it delegates entirely to the action type.

```
Constant                 GatePolicy                   reversible  groups
──────────────────────── ──────────────────────────── ─────────── ─────────────────────────────
SAR_FILING               ALWAYS                       false       ["aml-mlro"]
ACCOUNT_RESTRICTION      RISK_SCORE (≥0.8 or PEP)     true        ["aml-compliance"]
TRANSACTION_BLOCKING     CONFIDENCE (<0.9 gates)       false       ["aml-compliance"]
ENTITY_LINK_CREATION     RISK_SCORE (≥0.8 or PEP)     true        ["aml-compliance"]
LAW_ENFORCEMENT_REFERRAL ALWAYS                       false       ["aml-senior-compliance"]
```

**`candidateGroups` note (GE-20260607-326c7e):** fewer entries = more restrictive in the
engine chain. `["aml-mlro"]` (1 group) is the most restrictive gate in the system. The
compliance director group `["aml-senior-compliance"]` is also 1 entry but a narrower pool
by organisational convention — not by chain semantics. Documented here to prevent confusion.

**`scope`:** `"casehubio/aml/oversight"` on all gates, matching casehub-life convention
(engine#437 tracks exact semantics; applying the same pattern is safe now).

**`expiresIn`:** `null` on all types. Expiry policy is regulatory and configurable later;
null means the WorkItem lives until manual action, which is correct for compliance gates.

**`actionType()` mapping:** `ENTITY_LINK_CREATION` → `"entity.link.creation"` (kebab),
`fromActionType(String)` reverses via `toUpperCase().replace('.', '_')` + `valueOf()`.

**`AmlGroups` companion:** `api/src/main/java/io/casehub/aml/domain/AmlGroups.java`  
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

**Classification flow:**
1. `AmlActionType.fromActionType(action.actionType())` → `Optional<AmlActionType>`
2. If empty: return `Autonomous` (classifier does not gate unrecognised types)
3. Dispatch to per-type method; each reads from `action.context()`:
   - `riskScore` → `Double` (ACCOUNT_RESTRICTION, ENTITY_LINK_CREATION)
   - `entityType` → `String` (check `"PEP"`)
   - `confidenceScore` → `Double` (TRANSACTION_BLOCKING)
4. If context key is missing or unparseable → `Autonomous` (fail-open on missing data;
   the engine's fail-safe catches actual classifier *exceptions*, not missing context)

**Thresholds:** hardcoded constants — `RISK_SCORE_GATE_THRESHOLD = 0.8`,
`CONFIDENCE_GATE_THRESHOLD = 0.9`. Compliance thresholds are regulatory, not user
preferences; no `PreferenceProvider` dependency.

**GateRequired construction:** classifier reads `AmlActionType.reason()`, `.candidateGroups()`,
`.reversible()`, `.scope()` and constructs `GateRequired` directly. Action type owns the data;
classifier owns the condition check and record construction. `AmlActionType` remains pure Java
with no `casehub-engine-api` dependency — matching the `HouseholdActionType` pattern.

**No PreferenceProvider dependency** — avoids pulling platform-api into routing concerns.

---

## §3 — Layer 9 Oversight Harness

### YAML case definition

**File:** `app/src/main/resources/aml/aml-oversight-investigation.yaml`

Three-step investigation:
```
entity-resolution      → {entityType, riskScore, ownershipChain}
entity-link-proposal   → PlannedAction(ENTITY_LINK_CREATION, {entityType, riskScore, ownershipChain})
                         [gate fires if PEP or riskScore ≥ 0.8]
investigation-summary  → writes final findings, no WorkItem side effects
```

**Why ENTITY_LINK_CREATION as the demonstration action:**
- Produces both Autonomous (low-risk CORPORATE) and GateRequired (PEP/high-risk) paths
  within the same harness, enabling a clean two-test suite
- First consequential decision in a real AML investigation (entity graph validation
  precedes everything else)
- Requires no restructuring of existing sar-drafting or compliance review workers

### AmlOversightCaseHub

**File:** `app/src/main/java/io/casehub/aml/engine/AmlOversightCaseHub.java`

`YamlCaseHub("aml/aml-oversight-investigation.yaml")` — augments with three workers:
- `entityResolutionWorker` (same logic as Layer 5 — returns entityType, riskScore, ownershipChain)
- `entityLinkProposalWorker` — returns `WorkerResult.of(output, PlannedAction.of(reason, ENTITY_LINK_CREATION.actionType(), context))`
  where context = `{entityType, riskScore, ownershipChain}`
- `investigationSummaryWorker` — returns summary map, no side effects

### AmlLayer9Resource

**File:** `app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java`

```
POST /api/layer9/investigations     → starts investigation, returns {caseId}
GET  /api/layer9/investigations/{id} → returns {status}
```

Follows the same pattern as `AmlLayer5Resource` / `AmlLayer6Resource`.

---

## §4 — Testing

### Unit tests (api module, pure JUnit 5)

`AmlActionTypeTest` — `fromActionType` round-trip for all five constants; `actionType()`
format; `gatePolicy()`, `reversible()`, `candidateGroups()` for each constant.

### Unit tests (app module, Mockito)

`AmlActionRiskClassifierTest` — all five action types:
- ALWAYS types: verify GateRequired with correct reason, groups, reversible
- Threshold types: below threshold → Autonomous; at threshold → GateRequired; above → GateRequired
- Unknown actionType → Autonomous
- Missing context key → Autonomous (fail-open)
- Null actionType → Autonomous

Mockito note (GE-20260607-ab9f37): if a `@BeforeEach` stubs context key reads shared
across test methods, use `lenient()` — not all tests exercise all context keys.

### @QuarkusTest (app module)

**`AmlLayer9ActionGateTest`** — two test methods:

**`gate_fires_for_pep_entity`:**
1. POST investigation with `flagReason` containing "PEP" → entity-resolution returns
   `{entityType: "PEP", riskScore: 0.87}`
2. Await: gate WorkItem appears in H2 — poll:
   `SELECT w FROM WorkItem w WHERE w.callerRef LIKE 'case:<id>/gate:%'`
   This is the positive signal that the gate fired (engine paused)
3. Assert: WorkItem found, `candidateGroups = "aml-compliance"` (from AmlGroups.AML_COMPLIANCE)
4. Assert: `GET /api/layer9/investigations/{id}` status is NOT `completed`
   (engine is paused — investigation did not proceed autonomously)
5. `workItemService.completeFromSystem(workItem.id, "test-aml-compliance", "approved")`
   This fires `WorkItemLifecycleEvent(COMPLETED)` → `WorkItemLifecycleAdapter` routes to
   `ActionGateCompletionApplier` → engine resumes
6. Await: `status = completed`

**`gate_not_fired_for_low_risk_corporate`:**
1. POST investigation with CORPORATE flagReason → entity-resolution returns `{riskScore: 0.35, entityType: "CORPORATE"}`
2. Await: `status = completed` (classifier returns Autonomous, no gate)
3. Assert: no gate WorkItem exists for this caseId (EntityManager query returns empty)

**Test isolation:** `casehub.ledger.hash-chain.enabled=false` already in test
`application.properties` (protocol PP-20260604-f45c95). Both test methods must drain to
`status=completed` before returning (protocol PP-20260604-820c35).

---

## §5 — Issue #57: Partial Unique Index

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

## §6 — Deferred: sar-drafting design flaw

The current `sarDraftingWorkerJunior` / `sarDraftingWorkerSenior` call
`complianceReviewLifecycle.openReview()` unconditionally as a side effect inside the worker.
The correct design separates pure SAR drafting from the consequential filing step — with
`PlannedAction(SAR_FILING)` as the gate between them, and a `compliance-review-opening`
worker that only calls `openReview()` after MLRO approval.

This restructuring is out of scope for #42. File as a new GitHub issue before closing
this branch. Reference #42 as context.

---

## Coherence check

- SPI placement: `AmlActionRiskClassifier` in `app/` (has CDI, Quarkus deps) ✓
- `AmlActionType` in `api/` (pure Java, no deps, accessible to consumers) ✓
- Platform boundary: application-tier classifier, no domain knowledge in foundation ✓
- `@RiskClassifier @ApplicationScoped` on classifier: matches engine CDI discovery ✓
- candidateGroups fewer-is-more-restrictive: documented explicitly in AmlActionType ✓
- Layer 9 isolation: existing tests unaffected ✓
- Both Autonomous and GateRequired paths covered in @QuarkusTest ✓
- #57 closed as SQL-only, no code risk ✓
