# Layer 8: CaseMemoryStore Integration — Design Spec

**Date:** 2026-06-03 (revised after review)
**Issue:** casehubio/aml#32
**Branch:** issue-32-case-memory-store
**Depends on:** casehubio/platform#27 (CaseMemoryStore SPI) ✅ closed

---

## Purpose

Every AML investigation currently starts cold. Layer 8 wires `CaseMemoryStore` as a pre-case intelligence layer: facts accumulated from prior investigations enrich every new one.

`CaseMemoryStore` is the queryable semantic layer alongside the ledger. The ledger answers "what happened, tamper-evidently." Memory answers "what do we know about this entity, queryably." They serve different purposes and must not be conflated.

Three capabilities delivered:

1. **Context injection at case open** — prior entity facts loaded into `initialContext` before the engine starts. Query is synchronous and runs before `startCase()` because the context must be available before the first binding evaluates, including the `senior-analyst-required` binding which can fire on the first contextChange.
2. **Fact accumulation at case close** — investigation findings written to memory as each specialist completes, and as SAR verdicts arrive.
3. **Routing enrichment** — the `senior-analyst-required` YAML binding is extended to also evaluate `priorEntityContext.knownHighRisk`, enabling routing before entity resolution completes.

---

## Entity ID Strategy

`CaseMemoryStore` is keyed by `entityId`. AML uses **account IDs** — specifically `originAccountId` and `destinationAccountId` from `SuspiciousTransaction`.

**Why account IDs, not beneficial owner IDs:** The beneficial owner is discovered during entity resolution — after the case starts. At case open (when prior context is needed), only account IDs are known. The `Memory.text` field carries the natural-language narrative that links the account to its owner, preserving cross-entity context without requiring beneficial owner ID as the primary key.

**Network and pattern memories stored under both accounts:** `storeAll()` writes to both `originAccountId` and `destinationAccountId`. Either party appearing in a future investigation surfaces the relationship or pattern.

**SAR outcome memories stored under both accounts** via `storeAll()`.

**caseId in memory entries:** The engine UUID (the case instance UUID returned by `startCase()`), not `transaction.id()`. The engine UUID is the established stable identifier shared by the engine event log, ledger entries (`subjectId`), qhorus message entries, and compliance officer WorkItems. Using `transaction.id()` would create a split identifier space. The engine UUID is made available to behaviours by adding it to `initialContext` as `"caseId"` before `startCase()` is called (see Query Injection section).

### Known architectural limitation: account-scoped memory accumulation

Memory accumulates per-account, not per-entity. If beneficial owner X uses accounts A, B, and C (a textbook layering technique), investigations of accounts A, B, and C build no cross-account intelligence. An investigation starting from account A surfaces no prior context from cases that investigated B or C, even though they share a beneficial owner.

**Two-phase keying** is the deferred fix: after entity resolution, query memory for the resolved beneficial owner ID and inject as `priorBeneficialOwnerContext`; at case close, write memories under beneficial owner IDs in addition to account IDs. This is not in scope for Layer 8.

---

## Memory Domain Taxonomy

Three `MemoryDomain` constants in `AmlMemoryDomains` (app module):

| Constant | Domain name | What it stores |
|---|---|---|
| `ENTITY_RISK` | `aml.entity-risk` | Entity type, risk score, SAR history per account |
| `NETWORK` | `aml.network` | Counterparty relationships between accounts |
| `PATTERN` | `aml.pattern` | Typology matches (layering, structuring, smurfing) |

**Agent performance is excluded.** Handled by the existing trust-weighted routing system (`SarOutcomeFeedbackService`, `AmlTrustRoutingObserver`). Duplicating it in memory would create divergence.

Domain isolation enables: (a) targeted queries — entity-risk at case open does not pull network facts; (b) scoped GDPR erasure — `erase()` can target a single domain.

---

## Core Service: `AmlMemoryService`

A single `@ApplicationScoped` service encapsulating all `CaseMemoryStore` interactions. No caller touches `CaseMemoryStore` directly — all domain semantics (text formatting, domain selection, attribute conventions) live here.

### Read path

```
queryPriorContext(SuspiciousTransaction) → AmlPriorContext
```

Executes three queries (one per domain) across `[originAccountId, destinationAccountId]`. Each query uses `MemoryQuery.forEntities(entityIds, domain, tenantId).withLimit(10).withSince(lookbackCutoff)`. The lookback cutoff is `Instant.now().minus(lookbackDays)` where `lookbackDays` is a configurable `PreferenceKey<IntPreference>` (default: 365 days, following the existing pattern in `TrustRoutingPolicyKeys`).

**Per-query failure model:** Each of the three domain queries has its own try/catch. A failed query returns an empty list for that domain. Partial results are returned — `AmlPriorContext` reflects whatever was successfully retrieved. Log WARN per failure including domain. The case always proceeds; partial context is better than no context.

Returns an `AmlPriorContext` value record.

### Write path

| Method | Called from | Stored under | Notes |
|---|---|---|---|
| `storeEntityRisk(caseId, entityId, EntityResolutionResult)` | `EntityResolutionBehaviour` | `entityId` from result | |
| `storeNetworkRelationship(caseId, SuspiciousTransaction, EntityResolutionResult)` | `EntityResolutionBehaviour` | both account IDs via `storeAll()` | |
| `storePatternFindings(caseId, SuspiciousTransaction, PatternAnalysisResult)` | `PatternAnalysisBehaviour` | both account IDs via `storeAll()` | Structuring is destination-focal; origin-only would be an intelligence gap |
| `storeSarOutcome(caseId, SuspiciousTransaction, SarOutcome)` | `AmlSarOutcomeMemoryObserver` | both account IDs via `storeAll()` | WITHDRAWN verdict writes confidence = 0.0 reversal entry |

### Attribute conventions

All entries use `MemoryAttributeKeys`:
- `MemoryAttributeKeys.ACTOR_ID` — `"aml-system"` (autonomous specialist)
- `MemoryAttributeKeys.OUTCOME` — verdict name, entity type, or pattern flag
- `MemoryAttributeKeys.CONFIDENCE` — `MemoryAttributeKeys.formatConfidence(riskScore)` where applicable

### Failure handling

All store calls are individually wrapped in try/catch with WARN logging. A memory failure MUST NOT fail the investigation. Investigation is the primary flow; memory is additive.

---

## `AmlPriorContext` — Value Record

```java
record AmlPriorContext(
    List<Memory> entityRisk,
    List<Memory> network,
    List<Memory> pattern
)
```

Computed methods:

- `hasHistory()` — any non-empty list
- `isKnownHighRisk()` — groups entity-risk memories by `entityId`, takes the most recent entry per entity (by `createdAt`), returns true if any has parsed `confidence` attribute ≥ 0.8. Uses most-recent-per-entity so that a WITHDRAWN verdict (confidence 0.0) after an UPHELD verdict (0.9) correctly returns false.

### Fact selection strategy

`toContextMap()` merges results across all three domain lists. Selection: sort all returned memories by `createdAt DESC`, guarantee at least one entry per non-empty domain, fill remaining slots to a maximum of 10 total by recency. With `withLimit(10)` per domain query, up to 30 JPA rows are loaded, then merged and trimmed.

### Serialization via `toContextMap()`

Produces a `Map<String, Object>` for engine injection. Each fact is a structured object (not a plain string) to support machine consumption by future LLM agents:

```json
{
  "hasHistory": true,
  "knownHighRisk": false,
  "entityRiskCount": 2,
  "networkCount": 1,
  "patternCount": 0,
  "facts": [
    {
      "domain": "aml.entity-risk",
      "text": "Account acct-123 appeared in 2 prior AML investigations. Risk classification: STANDARD.",
      "createdAt": "2025-11-15T10:30:00Z",
      "confidence": "0.3500"
    },
    {
      "domain": "aml.network",
      "text": "Account acct-123 is a counterparty of account acct-456 (established in case engine-uuid-xyz).",
      "createdAt": "2025-09-01T08:00:00Z",
      "confidence": null
    }
  ]
}
```

YAML bindings reference `.priorEntityContext.knownHighRisk` and `.priorEntityContext.hasHistory`. LLM agents read `.priorEntityContext.facts[*].text` alongside domain and temporal metadata.

---

## Emission Strategy

**Two patterns used — neither is "Option A everywhere":**

**1. Direct call from agent behaviours** — for facts produced inside a Quartz worker: the behaviour is the single producer of that fact, has the result and the caseId, and the call is explicit and testable. 1:1 relationship between producer and memory write.

**2. CDI event for SAR outcomes** — `AmlLayer6Resource` fires `Event<SarOutcomeRecordedEvent>` once. Multiple observers consume it independently. Adding a future observer costs no changes to the resource. This is the right pattern when one event drives multiple concerns.

### Emission points

| Behaviour / observer | Stores | Pattern |
|---|---|---|
| `EntityResolutionBehaviour` | `storeEntityRisk()` + `storeNetworkRelationship()` | Direct call |
| `PatternAnalysisBehaviour` | `storePatternFindings()` | Direct call |
| `OsintScreeningBehaviour` | — | Sanction-list status at a point in time — not persistent entity facts |
| `SarDraftingBehaviour` | — | SAR narrative is intermediate; verdict comes from WorkItem outcome |
| `AmlSarOutcomeMemoryObserver` | `storeSarOutcome()` | CDI observer of `SarOutcomeRecordedEvent` |

### `SarOutcomeRecordedEvent`

A new CDI event type carrying `UUID caseId`. `AmlLayer6Resource` fires it after the SAR outcome POST is received.

- `SarOutcomeFeedbackService` becomes `@Observes SarOutcomeRecordedEvent` (synchronous — participates in existing qhorus transaction)
- `AmlSarOutcomeMemoryObserver` becomes `@Observes @Transactional(REQUIRES_NEW) SarOutcomeRecordedEvent` (own transaction on default datasource via memory-jpa)

### YAML inputSchema extension

Each capability that writes memory needs `caseId` in its payload. The engine UUID is added to `initialContext` as `"caseId"` at case start and is referenced in the inputSchema:

```yaml
capabilities:
  - name: entity-resolution
    inputSchema: "{ transaction: .transaction, caseId: .caseId }"
  - name: pattern-analysis
    inputSchema: "{ transaction: .transaction, entityGraph: .entityResolution.ownershipChain, caseId: .caseId }"
```

Other capabilities that do not write memory do not need the extension.

---

## SAR Outcome Observer

`AmlSarOutcomeMemoryObserver` observes `SarOutcomeRecordedEvent`. To retrieve both account IDs, it queries `AmlCaseOpenedLedgerEntry` (see Ledger Subclass section) by `subjectId = caseId`. `AmlCaseOpenedLedgerEntry` stores `originAccountId` and `destinationAccountId` as non-nullable fields written by `AmlLedgerService.writeCaseOpened()`.

If no `AmlCaseOpenedLedgerEntry` exists for the caseId (e.g. race or bug), the observer logs WARN and skips — never blocks.

**SAR outcome text example:** `"Transaction from acct-123 to acct-456 resulted in SAR filing (UPHELD). Investigation accuracy: 0.9200."`

**Verdict handling:**
- `UPHELD` → confidence = `investigationAccuracyScore`, outcome = `"UPHELD"`
- `WITHDRAWN` or `FLAGGED` → confidence = 0.0, outcome = verdict name. This writes a reversal entry. Subsequent `isKnownHighRisk()` calls for that account surface the reversal as the most recent entity-risk entry, returning false.

---

## Query Injection — Prior Context Enters the Case

`AmlEngineCoordinator.startInvestigation()` is extended with these steps before `startCase()`:

```
1. caseId = caseHub.startCase(initialContext).get(timeout)  ← unchanged first
```

Wait — the caseId is returned BY startCase. The query needs to happen BEFORE startCase to inject prior context. The solution:

```
1. query AmlMemoryService.queryPriorContext(transaction)    → AmlPriorContext
2. serialize via AmlPriorContext.toContextMap()             → Map<String, Object>
3. build initialContext:
       "transaction" → txMap
       "caseId"      → to be filled after startCase (see below)
       "priorEntityContext" → contextMap
4. caseId = caseHub.startCase(initialContext).get(timeout)  → engine UUID
5. update initialContext with "caseId" = caseId.toString()  [if engine supports context update]
```

**Problem:** The engine UUID is not known until after `startCase()`. But `caseId` must be in the COMMAND payload for behaviours. The engine uses the initialContext as the starting blackboard. After case start, the engine UUID can be injected via a context update OR added as `"caseId"` immediately after via an engine API if one exists.

**Simpler resolution:** Use a two-step initialContext build:
- Build `priorEntityContext` synchronously before `startCase()`
- After `startCase()` returns the engine UUID, inject it into context via an engine context-update call (if the engine supports it) OR accept that `caseId` is not available to the FIRST binding (entity-resolution fires on first contextChange, by which point the coordinator can have updated context)

If no context-update API exists in the engine: add `caseId` to a second initialContext update after case start. This is an engine-level question requiring clarification before implementation. **Flag as a pre-implementation clarification item.**

Alternative if context update is unavailable: the SAR outcome observer gets the caseId from the event payload (it is always the engine UUID), so the memory writes from the SAR observer are correctly keyed. For behaviour-generated memories, if `caseId` cannot be in the first COMMAND payload, the `caseId` field in those memory entries can be null — acceptable, since the primary query key is `entityId`, not `caseId`.

If `queryPriorContext()` throws, log WARN, inject `{"hasHistory": false, "knownHighRisk": false, "entityRiskCount": 0, "networkCount": 0, "patternCount": 0, "facts": []}`, and proceed.

---

## YAML Binding — Merged Senior Analyst Routing

The `senior-analyst-required` binding is extended to evaluate prior context in an OR with the entity resolution result. The separate `immediate-senior-required` binding is NOT added — merging eliminates a timing race between two bindings dispatching the same capability.

```yaml
## Fires on first contextChange where any high-risk signal is present.
## Prior context signal: fires at case start for entities with established high-risk history.
## Entity resolution signal: fires after entity resolution for first-time encounters.
## The .seniorAnalystReview == null guard prevents re-fire after capability result is written back.
- name: senior-analyst-required
  on: { contextChange: {} }
  when: >-
    ((.priorEntityContext.knownHighRisk == true) or
     (.entityResolution.entityType == "PEP") or
     (.entityResolution.riskScore > 0.8)) and
    .seniorAnalystReview == null
  capability: senior-analyst-review
```

This binding fires once for a known-high-risk entity (from prior context, on first contextChange), and once for a first-time-encountered PEP or high-risk entity (after entity resolution). The `.seniorAnalystReview == null` guard ensures only one dispatch occurs regardless of how many contextChange events fire.

---

## Ledger Subclass Redesign

The existing `AmlInvestigationLedgerEntry` has a dual-use design flaw flagged in its own Javadoc comment. Layer 8 is the right moment to resolve it — before it is deeper.

**New design:** Replace `AmlInvestigationLedgerEntry` with two sibling subclasses of `LedgerEntry`:

**`AmlCaseOpenedLedgerEntry`** (discriminator: `AML_CASE_OPENED`):
- `transactionId` (non-nullable) — external transaction reference
- `originAccountId` (non-nullable) — source account
- `destinationAccountId` (non-nullable) — destination account
- Own join table: `aml_case_opened_ledger_entry`

**`AmlComplianceReviewLedgerEntry`** (discriminator: `AML_COMPLIANCE_REVIEW`):
- `taskId` (non-nullable) — the SAR review WorkItem task ID
- Own join table: `aml_compliance_review_ledger_entry`

**Why two siblings instead of the reviewer's proposed child of existing parent:** The sibling approach eliminates the shared parent entirely, removing the dual-use smell at the root rather than patching it. Since this is a tutorial harness with ephemeral H2 test data, the migration can recreate the schema fresh without a data migration.

**Migration**: Drop `aml_investigation_ledger_entry`. Create `aml_case_opened_ledger_entry` and `aml_compliance_review_ledger_entry`. Update discriminator values on any existing `ledger_entry` rows (or accept data loss in H2 test context).

**`AmlLedgerService` changes:**
- `writeCaseOpened()` creates `AmlCaseOpenedLedgerEntry` with `originAccountId` and `destinationAccountId` from the transaction
- `writeComplianceReviewOpened()` creates `AmlComplianceReviewLedgerEntry` with `taskId`

### Pre-implementation verification required: Merkle hash coverage

**Before merging any schema change**, confirm that the Merkle hash computation in casehub-ledger does NOT cover JOIN table columns — only base `LedgerEntry` fields. If the hash covers subclass columns, existing entries would fail verification after schema changes. Read the hash computation in `casehub-ledger` and document the conclusion in the LAYER-LOG before proceeding. This is a blocker.

---

## GDPR / Regulatory Design

### Dual-write resolves the FinCEN vs GDPR tension

`CaseMemoryStore` is the intelligence layer; the ledger is the compliance record. GDPR Art.17(3)(b) exempts data processed for legal compliance (FinCEN SAR retention) — this applies to the ledger, not the memory store. Erasing memory leaves the compliance record intact.

### Known limitation: network memory GDPR erasure is not cascaded

When `erase(entityId = A, domain = aml.network)` is called, account A's network memories are deleted. Account B's memories may still contain text referencing account A as a counterparty. Under strict GDPR interpretation, account A's identity in account B's memory is personal data of account A and should also be erased.

The `CaseMemoryStore` SPI provides no cascade mechanism. Cascade erasure would require scanning all memory entries for cross-references — not feasible with the current SPI.

**This must be assessed by AML's legal team before production deployment.** The technical system cannot resolve this; it can only document it. Technical options if legal review requires cascade erasure:
- Store counterparty account ID as a structured attribute (not just in text) to enable targeted erasure
- Implement a cross-entity memory scan in `AmlMemoryService.eraseAccountMemory()` that erases both direct and cross-reference entries

---

## Dependencies and Persistence

### `casehub-aml-app/pom.xml` additions

```xml
<!-- Layer 8: JPA-backed CaseMemoryStore (prod) — displaces NoOpCaseMemoryStore @DefaultBean -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-memory-jpa</artifactId>
  <version>${casehub.version}</version>
</dependency>

<!-- Layer 8: in-memory CaseMemoryStore (test isolation) — @Alternative @Priority(1) -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-memory-inmem</artifactId>
  <version>${casehub.version}</version>
  <scope>test</scope>
</dependency>
```

### `application.properties` changes

```properties
# Layer 8: add memory table migration (V1000__memory_entry.sql from platform)
# No version collision: work uses V1-V999 on this datasource; memory uses V1000
quarkus.flyway.locations=classpath:db/work/migration,classpath:db/memory/migration

# Layer 8: add MemoryEntry JPA entity to default persistence unit
quarkus.hibernate-orm.packages=io.casehub.work.runtime.model,io.casehub.work.runtime.filter,io.casehub.aml.domain,io.casehub.platform.memory.jpa
```

### `src/test/resources/application.properties` (new or extended)

```properties
# Layer 8: use in-memory store in @QuarkusTest — volatile, no datasource required
quarkus.arc.selected-alternatives=io.casehub.platform.memory.inmem.InMemoryMemoryStore
```

### New AML-owned migrations

- **VNNnn** (on `aml-ledger` classpath, `db/aml-ledger/migration`) — confirm exact version by scanning the directory during implementation, never assert blindly. Creates `aml_case_opened_ledger_entry` and `aml_compliance_review_ledger_entry`, drops `aml_investigation_ledger_entry`.
- No new version needed for `memory_entry` — that schema comes from the platform's `V1000__memory_entry.sql`.

---

## New and Modified Types

### New types (app module)

| Class | Purpose |
|---|---|
| `AmlMemoryDomains` | `MemoryDomain` constants: `ENTITY_RISK`, `NETWORK`, `PATTERN` |
| `AmlMemoryService` | Central service — all store/query logic, text formatting, failure guard |
| `AmlPriorContext` | Value record: three domain results + `hasHistory()` + `isKnownHighRisk()` + `toContextMap()` |
| `AmlSarOutcomeMemoryObserver` | CDI observer of `SarOutcomeRecordedEvent`; writes SAR outcome memory |
| `SarOutcomeRecordedEvent` | CDI event type carrying `UUID caseId`; fired by `AmlLayer6Resource` |
| `AmlCaseOpenedLedgerEntry` | Replaces `AmlInvestigationLedgerEntry` for CASE_OPENED events |
| `AmlComplianceReviewLedgerEntry` | Replaces `AmlInvestigationLedgerEntry` for COMPLIANCE_REVIEW_OPENED events |

### Modified types

| Class/file | Change |
|---|---|
| `AmlEngineCoordinator` | Query prior context, inject `priorEntityContext` and `caseId` into initialContext before/after `startCase()` |
| `EntityResolutionBehaviour` | Inject `AmlMemoryService`, call `storeEntityRisk()` + `storeNetworkRelationship()`; read caseId from payload |
| `PatternAnalysisBehaviour` | Inject `AmlMemoryService`, call `storePatternFindings()` under both accounts; read caseId from payload |
| `AmlLedgerService` | `writeCaseOpened()` creates `AmlCaseOpenedLedgerEntry`; `writeComplianceReviewOpened()` creates `AmlComplianceReviewLedgerEntry` |
| `AmlLayer6Resource` | Fire `SarOutcomeRecordedEvent`; remove direct call to `SarOutcomeFeedbackService` |
| `SarOutcomeFeedbackService` | Becomes `@Observes SarOutcomeRecordedEvent` |
| `aml-investigation.yaml` | Extend `entity-resolution` + `pattern-analysis` inputSchemas with `caseId: .caseId`; merge `senior-analyst-required` binding |
| `application.properties` | Flyway locations + Hibernate packages |
| `casehub-aml-app/pom.xml` | Add `memory-jpa` (compile) + `memory-inmem` (test) |
| VNNnn migration | Drop `aml_investigation_ledger_entry`; create two subclass tables |

---

## Pre-Implementation Clarification Items

1. **Merkle hash coverage** — confirm hash does not cover JOIN table columns before any schema change. Blocker.
2. **Engine context update API** — confirm whether casehub-engine supports post-start context injection. Determines whether `caseId` can be in the initial COMMAND payload to behaviours or whether it must be null in behaviour-generated memory entries.
3. **YAML inputSchema and Java payload access** — confirm whether behaviours read their input strictly from YAML-defined keys or can access raw COMMAND payload fields directly.

---

## Test Strategy

| Layer | Scenario | Approach |
|---|---|---|
| Unit | `AmlMemoryService` text formatting is human-readable and attribute-complete | JUnit 5, mock `CaseMemoryStore` |
| Unit | `isKnownHighRisk()` threshold — true at 0.8, false at 0.79 | JUnit 5, no CDI |
| Unit | `isKnownHighRisk()` uses most-recent-per-entity — WITHDRAWN reversal (0.0) after UPHELD (0.9) returns false | JUnit 5, construct `Memory` records with `createdAt` ordering |
| Unit | `isKnownHighRisk()` respects lookback window — entry older than configured window does NOT trigger | JUnit 5, construct `Memory` with old `createdAt` |
| Unit | `AmlMemoryDomains` constants have correct domain name strings | JUnit 5 |
| `@QuarkusTest` | `storeEntityRisk()` → `queryPriorContext()` roundtrip returns the stored fact | `InMemoryMemoryStore` via selected-alternatives |
| `@QuarkusTest` | `storeAll()` for network/pattern — both account IDs independently queryable | Assert both account IDs return the fact |
| `@QuarkusTest` | GDPR erasure round-trip: `erase()` on accountId → `queryPriorContext()` returns empty for that account | Domain-scoped erase |
| `@QuarkusTest` | Cross-domain erasure isolation: erase `ENTITY_RISK` only → `NETWORK` domain unaffected | Assert non-erased domain still populated |
| `@QuarkusTest` | Partial query failure: mock 2 of 3 domain queries to throw; assert partial `AmlPriorContext` populated from successful queries | Mock `CaseMemoryStore` with selective failure |
| `@QuarkusTest` | `startInvestigation()` injects `priorEntityContext` map into engine initialContext | Assert initialContext Map structure before `startCase()` |
| `@QuarkusTest` | `immediate-senior-required` via merged binding: high-confidence entity-risk memory → `senior-analyst-required` binding fires before entity-resolution result written to context | Use slow `EntityResolutionBehaviour` stub (200ms sleep); assert capability dispatched before `entityResolution` appears in context |
| `@QuarkusTest` | Duplicate dispatch prevention: entity known-high-risk in memory AND confirmed PEP during resolution → senior-analyst-review dispatched exactly once | Assert one dispatch, not two |
| `@QuarkusTest` | SAR outcome CDI event → both `SarOutcomeFeedbackService` and `AmlSarOutcomeMemoryObserver` execute | Fire event, assert both trust attestation and memory entry written |
| `@QuarkusTest` | SAR outcome memory: both account IDs receive the SAR outcome entry | Assert both accounts in store after event |
| `@QuarkusTest` | WITHDRAWN SAR outcome writes reversal entry with confidence 0.0 | Assert reversal entry; subsequent `isKnownHighRisk()` returns false |
| `@QuarkusTest` | Fact selection: 15 memories across domains → exactly 10 in `toContextMap()` facts, at least one per non-empty domain, ordered by recency | Assert selection strategy |

---

## Out of Scope

- **Account-to-entity two-phase keying** — deferred; see Known architectural limitation section
- **Cascade GDPR erasure for cross-references** — requires legal review first; see GDPR section
- **`memory-sqlite` adapter** — SQLite is for single-process deployments; AML targets PostgreSQL
- **Platform `memory-cdi/` module (Option B)** — deferred to platform; this issue implements direct calls from behaviours and CDI events at the resource level
- **REST endpoint for memory query** — expose via `GET /api/layer8/memory/{accountId}` in a follow-up
- **Semantic adapter (Mem0, Graphiti)** — JPA adapter with FTS sufficient for text search; vector search deferred
