# Layer 8: CaseMemoryStore Integration — Design Spec

**Date:** 2026-06-03  
**Issue:** casehubio/aml#32  
**Branch:** issue-32-case-memory-store  
**Depends on:** casehubio/platform#27 (CaseMemoryStore SPI) ✅ closed

---

## Purpose

Every AML investigation currently starts cold. Layer 8 wires `CaseMemoryStore` as a pre-case intelligence layer: facts accumulated from prior investigations enrich every new one.

`CaseMemoryStore` is the queryable semantic layer alongside the ledger. The ledger answers "what happened, tamper-evidently." Memory answers "what do we know about this entity, queryably." They serve different purposes and must not be conflated.

Three capabilities delivered:

1. **Context injection at case open** — prior entity facts loaded into `initialContext` before the engine starts
2. **Fact accumulation at case close** — investigation findings written to memory as each specialist completes
3. **Routing enrichment** — the YAML case definition references `priorEntityContext.knownHighRisk` to route known-high-risk entities to senior analysts immediately, without waiting for entity resolution

---

## Entity ID Strategy

`CaseMemoryStore` is keyed by `entityId`. AML uses **account IDs** — specifically `originAccountId` and `destinationAccountId` from `SuspiciousTransaction`.

**Why account IDs, not beneficial owner IDs:** The beneficial owner is discovered during entity resolution — after the case starts. At case open (when prior context is needed), only account IDs are known. The `Memory.text` field carries the natural-language narrative that links the account to its owner, preserving cross-entity context without requiring beneficial owner ID as the primary key.

**Network memories stored under both accounts:** A transaction A→B stores the relationship fact under both `originAccountId` and `destinationAccountId` via `storeAll()`. Either party appearing in a future investigation surfaces the relationship.

**SAR outcome memories stored under both accounts:** The SAR covers the full transaction. Both accounts accumulate SAR history.

**caseId in memory entries:** `transaction.id()` (the flagged transaction's own ID) is used as the `caseId` field in all memory entries. It is available in every COMMAND payload to behaviours via the inputSchema and is stable across the investigation. The engine UUID ↔ transaction ID mapping is preserved in the ledger for retrospective queries.

---

## Memory Domain Taxonomy

Three `MemoryDomain` constants in `AmlMemoryDomains` (app module):

| Constant | Domain name | What it stores |
|---|---|---|
| `ENTITY_RISK` | `aml.entity-risk` | Entity type, risk score, SAR history per account |
| `NETWORK` | `aml.network` | Counterparty relationships between accounts |
| `PATTERN` | `aml.pattern` | Typology matches (layering, structuring, smurfing) |

**Agent performance is excluded.** That concern belongs to the existing trust-weighted routing system (`SarOutcomeFeedbackService`, `AmlTrustRoutingObserver`). Duplicating it in memory would create divergence.

Domain isolation enables: (a) targeted queries — entity-risk at case open does not pull network facts; (b) scoped GDPR erasure — `erase()` can target a single domain.

---

## Core Service: `AmlMemoryService`

A single `@ApplicationScoped` service encapsulating all `CaseMemoryStore` interactions. No caller touches `CaseMemoryStore` directly — all domain semantics (text formatting, domain selection, attribute conventions) live here.

### Read path

```
queryPriorContext(SuspiciousTransaction) → AmlPriorContext
```

Executes three queries (one per domain) across `[originAccountId, destinationAccountId]`. Returns an `AmlPriorContext` value record.

### Write path

| Method | Called from | Stored under |
|---|---|---|
| `storeEntityRisk(transactionId, entityId, EntityResolutionResult)` | `EntityResolutionBehaviour` | `entityId` from result |
| `storeNetworkRelationship(transactionId, SuspiciousTransaction, EntityResolutionResult)` | `EntityResolutionBehaviour` | both account IDs via `storeAll()` |
| `storePatternFindings(transactionId, entityId, PatternAnalysisResult)` | `PatternAnalysisBehaviour` | `originAccountId` |
| `storeSarOutcome(transactionId, SuspiciousTransaction, SarOutcome)` | `AmlSarOutcomeMemoryObserver` | both account IDs via `storeAll()` |

### Attribute conventions

All entries use `MemoryAttributeKeys`:
- `MemoryAttributeKeys.ACTOR_ID` — `"aml-system"` (autonomous specialist)
- `MemoryAttributeKeys.OUTCOME` — verdict name, entity type, or pattern flag
- `MemoryAttributeKeys.CONFIDENCE` — `MemoryAttributeKeys.formatConfidence(riskScore)` where applicable

### Failure handling

All store calls are wrapped in try/catch with WARN logging. A memory failure MUST NOT fail the investigation. Investigation is the primary flow; memory is additive. Log warn, never rethrow.

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
- `isKnownHighRisk()` — any entity-risk memory has `confidence` attribute ≥ 0.8

**Serialization via `toContextMap()`:** Produces a flat, YAML-binding-friendly `Map<String, Object>`:

```json
{
  "hasHistory": true,
  "knownHighRisk": false,
  "entityRiskCount": 2,
  "networkCount": 1,
  "patternCount": 0,
  "facts": [
    "Account acct-123 appeared in 2 prior AML investigations. Risk classification: STANDARD.",
    "Account acct-123 is a counterparty of account acct-456 (established in case tx-789)."
  ]
}
```

`facts` is a list of `Memory.text()` summaries, bounded to 10 entries. YAML bindings reference `.priorEntityContext.knownHighRisk`; future LLM agents read `.priorEntityContext.facts` for context.

---

## Emission Strategy

**Pattern: direct call from each agent behaviour** (Option A of platform#48).

The engine drives each behaviour asynchronously via Quartz. The behaviour is the right owner: it has the result, the payload, and the call is explicit and testable. A CDI event indirection adds complexity without benefit when the behaviour is the single producer of that fact.

### Emission points

| Behaviour | Stores | What is skipped |
|---|---|---|
| `EntityResolutionBehaviour` | `storeEntityRisk()` + `storeNetworkRelationship()` | |
| `PatternAnalysisBehaviour` | `storePatternFindings()` | |
| `OsintScreeningBehaviour` | — | OSINT results are sanction-list status at a point in time — not persistent entity facts |
| `SarDraftingBehaviour` | — | SAR narrative is an intermediate result; verdict comes from the WorkItem outcome |

### YAML inputSchema extension

The `entity-resolution` capability's `inputSchema` is extended to make `transactionId` available in the COMMAND payload:

```yaml
inputSchema: "{ transaction: .transaction, transactionId: .transaction.id }"
```

Behaviours read `transactionId` from their input to use as `caseId` in memory entries.

---

## SAR Outcome Observer

A new `AmlSarOutcomeMemoryObserver` class stores SAR outcome memory when a verdict is recorded. It is called from `AmlLayer6Resource` alongside the existing `SarOutcomeFeedbackService.recordOutcome()` call.

To retrieve both account IDs, the observer reads the `CASE_OPENED` `AmlInvestigationLedgerEntry` for the case, queried by `subjectId = caseId`. The ledger entry is extended with `originAccountId` and `destinationAccountId` columns (written by `AmlLedgerService.writeCaseOpened()`, which already receives the full `SuspiciousTransaction`). This requires a V2007 migration on the `aml-ledger` classpath. The observer avoids changing the request body or adding a new endpoint parameter.

SAR outcome memory text example: `"Transaction from acct-123 to acct-456 resulted in SAR filing (UPHELD). Investigation accuracy: 0.9200."`

Attributes: `outcome = verdict.name()`, `confidence = investigationAccuracyScore`.

This is the highest-value memory fact. A future case involving either account immediately surfaces whether prior SAR filings were upheld or declined.

---

## Query Injection — Prior Context Enters the Case

`AmlEngineCoordinator.startInvestigation()` is extended with three steps before `startCase()`:

```
1. query AmlMemoryService.queryPriorContext(transaction)  → AmlPriorContext
2. serialize via AmlPriorContext.toContextMap()            → Map<String, Object>
3. initialContext.put("priorEntityContext", contextMap)
4. caseHub.startCase(initialContext)                       [unchanged]
```

The query is synchronous and fast (indexed by `tenant_id, entity_id, domain, created_at DESC`). If `queryPriorContext()` throws, log WARN, inject `{"hasHistory": false, "knownHighRisk": false, "entityRiskCount": 0, "networkCount": 0, "patternCount": 0, "facts": []}`, and proceed.

---

## YAML Binding Enrichment

A new binding fires at case start for known-high-risk entities, before entity resolution completes:

```yaml
## Fires at case start for entities with established high-risk history in memory.
## Runs in parallel with entity-resolution — does not wait for it.
- name: immediate-senior-required
  on: { contextChange: {} }
  when: >-
    .transaction != null and
    .priorEntityContext != null and
    .priorEntityContext.knownHighRisk == true and
    .seniorAnalystReview == null
  capability: senior-analyst-review
```

The existing `senior-analyst-required` binding remains unchanged for first-time encounters (entities not yet in memory who turn out to be PEP or high-risk during resolution).

This delivers a meaningful improvement over the issue's stated intent: a money launderer with established history gets routed to a senior analyst at case start, not after entity resolution confirms it. Entity resolution runs in parallel and may surface additional beneficiaries.

---

## Regulatory Design: GDPR vs FinCEN

**Tension:** GDPR Art.17 right to erasure applies to `CaseMemoryStore`. FinCEN 31 CFR 1020.320 requires 5-year SAR record retention.

**Resolution: dual-write, not retention holds in memory.**

`CaseMemoryStore` is the intelligence layer. The compliance record is the ledger (`AmlInvestigationLedgerEntry`, Merkle chain, `CaseLedgerEntry`). GDPR Art.17(3)(b) exempts data processed for legal compliance obligations — this exemption applies to the **ledger**, not to the memory store.

If `eraseEntity()` is called for an account, its memory is wiped and future investigations start fresh for that account. This is acceptable: the SAR filing is preserved in the ledger (FinCEN-compliant). Memory is not the regulatory record.

No special retention attributes are needed in `CaseMemoryStore`. This resolves the tension cleanly.

Document this in `AmlMemoryService` Javadoc. Submit Option A (direct call) as AML's answer to platform#48.

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

**New AML-owned migration: V2007** — on the `aml-ledger` classpath (`db/aml-ledger/migration`), run on the `qhorus` datasource:

```sql
ALTER TABLE aml_investigation_ledger_entry
  ADD COLUMN origin_account_id VARCHAR(255),
  ADD COLUMN destination_account_id VARCHAR(255);
```

Nullable columns: existing CASE_OPENED rows predate Layer 8. The SAR outcome observer handles nulls gracefully (skips memory write, logs WARN).

The `memory_entry` table schema comes from `V1000__memory_entry.sql` in `casehub-platform-memory-jpa` — no AML-owned migration needed for it.

---

## New and Modified Types

### New types (app module)

| Class | Purpose |
|---|---|
| `AmlMemoryDomains` | `MemoryDomain` constants: `ENTITY_RISK`, `NETWORK`, `PATTERN` |
| `AmlMemoryService` | Central service — all store/query logic, text formatting, failure guard |
| `AmlPriorContext` | Value record: three domain results + `hasHistory()` + `isKnownHighRisk()` + `toContextMap()` |
| `AmlSarOutcomeMemoryObserver` | Writes SAR outcome memory when verdict is recorded |

### Modified types

| Class/file | Change |
|---|---|
| `AmlEngineCoordinator` | Query prior context, inject into initialContext before `startCase()` |
| `EntityResolutionBehaviour` | Inject `AmlMemoryService`, call `storeEntityRisk()` + `storeNetworkRelationship()` |
| `PatternAnalysisBehaviour` | Inject `AmlMemoryService`, call `storePatternFindings()` |
| `aml-investigation.yaml` | Add `immediate-senior-required` binding; extend entity-resolution inputSchema with `transactionId` |
| `AmlInvestigationLedgerEntry` | Add `originAccountId` + `destinationAccountId` JPA columns |
| `AmlLedgerService.writeCaseOpened()` | Write the two new account ID fields from the transaction |
| `AmlLayer6Resource` | Call `AmlSarOutcomeMemoryObserver.recordOutcome()` alongside existing service call |
| `application.properties` | Flyway locations + Hibernate packages |
| `casehub-aml-app/pom.xml` | Add `memory-jpa` (compile) + `memory-inmem` (test) |
| `V2007__aml_investigation_account_ids.sql` | New migration on aml-ledger classpath |

---

## Test Strategy

| Layer | Scenario | Approach |
|---|---|---|
| Unit | `AmlMemoryService` text formatting is human-readable and attribute-complete | JUnit 5, mock `CaseMemoryStore` |
| Unit | `AmlPriorContext.isKnownHighRisk()` threshold logic | JUnit 5, no CDI |
| Unit | `AmlMemoryDomains` constants have correct domain name strings | JUnit 5 |
| `@QuarkusTest` | `storeEntityRisk()` → `queryPriorContext()` roundtrip returns the stored fact | `InMemoryMemoryStore` via selected-alternatives |
| `@QuarkusTest` | `startInvestigation()` injects `priorEntityContext` map into engine initialContext | Assert initialContext Map shape before `startCase()` |
| `@QuarkusTest` | Entity resolution behaviour writes entity-risk memory after running | Run via engine, assert store state |
| `@QuarkusTest` | Known-high-risk entity: `immediate-senior-required` binding fires at case start | Pre-populate store with high-confidence entity-risk memory; assert senior-analyst-review capability dispatched before entity-resolution completes |
| `@QuarkusTest` | SAR outcome observer writes memory for both account IDs | Call `AmlLayer6Resource` POST; assert both account IDs in store |

The routing test (last `@QuarkusTest`) closes the loop: prior context written in one case drives routing in the next. It is the most important test for validating the architectural claim.

---

## Out of Scope

- **Agent performance domain** — handled by trust-weighted routing (Layer 6); would duplicate it
- **`memory-sqlite` adapter** — SQLite is for single-process deployments; AML targets PostgreSQL
- **Platform `memory-cdi/` module (Option B)** — deferred to platform; this issue implements Option A
- **REST endpoint for memory query** — expose via `GET /api/layer8/memory/{accountId}` in a follow-up if compliance officers need direct access
- **Semantic adapter (Mem0, Graphiti)** — the JPA adapter with FTS is sufficient for text search; vector search deferred
