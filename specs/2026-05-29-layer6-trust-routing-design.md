# Layer 6 — Trust-Weighted Routing Design

**Issue:** casehubio/aml#38  
**Date:** 2026-05-29  
**Branch:** issue-38-layer6-trust-routing  
**Blockers resolved:** engine#382 ✅, engine#390 ✅

---

## What This Layer Adds

Layer 6 closes the final major compliance gap: agents are selected randomly regardless of their investigation track record. After Layer 6, experienced agents are routed complex cases based on SAR outcome attestations. Post-investigation feedback writes `LedgerAttestation` records that `TrustScoreJob` uses to update Bayesian Beta trust scores nightly — completing the feedback loop.

**Teaching contrast:**
- Before Layer 6: `LeastLoadedAgentStrategy` picks any idle worker
- After Layer 6: `TrustWeightedAgentStrategy` picks trusted workers; junior agents excluded until they build a track record

---

## Engine Foundation (already delivered)

**engine#382 — SPI placement:**  
`TrustRoutingPolicy` + `TrustRoutingPolicyProvider` moved to `casehub-engine-api`. AML implements the SPI without depending on the implementation module.

**engine#390 — Worker decision ledger trail:**  
`WorkerDecisionEvent` (CDI record, `casehub-engine-common`) fired after each successful worker execution. `WorkerDecisionEventCapture` (in `casehub-engine-ledger`) observes it and writes a `WorkerDecisionEntry extends LedgerEntry` with `actorId = workerId`, `capabilityTag`, `caseId`. These entries are what `LedgerAttestation` records reference — enabling per-agent trust scoring.

Both activate automatically when `casehub-engine-ledger` is on the classpath.

---

## Section 1: Dependencies, Flyway, Jandex, CDI Fixes

### New dependency (`app/pom.xml`)

```xml
<!-- Layer 6: trust-weighted routing + WorkerDecisionEntry per worker execution -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-ledger</artifactId>
    <version>${casehub.version}</version>
</dependency>
```

Activates automatically:
- `TrustWeightedAgentStrategy @Alternative @Priority(1) @ApplicationScoped` — Arc auto-selects any `@Alternative` with `@Priority ≥ 0` without needing an explicit `quarkus.arc.selected-alternatives` entry. Beats `LeastLoadedAgentStrategy @DefaultBean`.
- `WorkerDecisionEventCapture` (writes `WorkerDecisionEntry` per worker)

### Flyway fixes (both `application.properties` and `test/resources/application.properties`)

casehub-work ships migrations at `classpath:db/work/migration` (verified from published jar — confirmed work#229 complete). The aml default Flyway location was incorrectly left at `classpath:db/migration`. Fix and add engine-ledger path to qhorus:

```properties
# casehub-work migrations are at db/work/migration (work#229 complete)
quarkus.flyway.locations=classpath:db/work/migration

# Add engine-ledger V2000 + V2001 to qhorus datasource (where ledger_entry lives)
# classpath:db/migration is engine-ledger's current path pending engine#395 (scoping fix)
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration,classpath:db/aml-ledger/migration,classpath:db/migration
```

### CDI cleanup (two separate items)

**Item 1 — blocking augmentation error (main `application.properties`):**  
Engine PR#378 removed `CasehubWorkloadProvider`. The `%prod.quarkus.arc.exclude-types` entry still references it. Quarkus fails augmentation at build time when `exclude-types` names a class that does not exist on the classpath. Remove:
```properties
# DELETE this line — CasehubWorkloadProvider was removed in engine PR#378
%prod.quarkus.arc.exclude-types=io.casehub.engine.internal.worker.CasehubWorkloadProvider
```

**Item 2 — stale exclusion cleanup (test `application.properties`):**  
`JpaWorkloadProvider` was excluded to prevent CDI ambiguity with `CasehubWorkloadProvider`. Since `CasehubWorkloadProvider` no longer exists, there is nothing to conflict with. The engine does not inject `WorkloadProvider` as a CDI point — `runningJobs` in `AgentCandidate` comes from Quartz directly. The exclusion is now dead weight; remove `io.casehub.work.runtime.service.JpaWorkloadProvider` from `quarkus.arc.exclude-types` for clarity.

### Jandex (test `application.properties`)

```properties
quarkus.index-dependency.engine-ledger.group-id=io.casehub
quarkus.index-dependency.engine-ledger.artifact-id=casehub-engine-ledger
```

`WorkerDecisionEntry` is in `io.casehub.ledger.model` — already in `quarkus.hibernate-orm.qhorus.packages`. No package addition needed.

---

## Section 2: `AmlTrustRoutingPolicyProvider`

**Location:** `io.casehub.aml.routing.AmlTrustRoutingPolicyProvider`  
**Annotation:** `@ApplicationScoped` — beats `DefaultTrustRoutingPolicyProvider @DefaultBean` automatically  
**Implements:** `TrustRoutingPolicyProvider` (`casehub-engine-api`)

Resolves per-capability trust policies from the platform `Preferences` API via `SettingsScope(Path.of("casehubio", "aml", "trust-routing", capabilityName), Instant.now())`, falling back to hardcoded AML defaults when no preference is configured.

Tutorial deployment uses `MockPreferenceProvider @DefaultBean` → empty → fallbacks always apply. Production overrides with `casehub-platform-config` (YAML) or JPA backend — no code change required.

### Preference type (`app/` module — not `api/`)

`TrustPolicyPreference` lives in `app/` since it depends on `casehub-platform-api` (a pure-Java dep, acceptable in `app/` but not in `api/` which has zero dependencies):

```java
// io.casehub.aml.routing.TrustPolicyPreference
public record TrustPolicyPreference(
    double threshold,
    int minimumObservations,
    double borderlineMargin,
    double blendFactor,
    Map<String, Double> qualityFloors
) implements Preference {}
```

Preference keys: `PreferenceKey.of("casehubio.aml.trust-routing.<capability>", TrustPolicyPreference.class)`.

### AML capability policies (hardcoded fallbacks)

| Capability | Threshold | Min obs | Blend | Quality floors |
|---|---|---|---|---|
| `entity-resolution` | 0.70 | 10 | 0.60 | — |
| `pattern-analysis` | 0.65 | 10 | 0.60 | — |
| `osint-screening` | 0.70 | 10 | 0.65 | — |
| `sar-drafting` | 0.75 | 10 | 0.70 | investigation-accuracy ≥ 0.65 |
| `senior-analyst-review` | 0.80 | 10 | 0.70 | — |
| anything else | `TrustRoutingPolicy.DEFAULT` | | | |

Quality floors for OSINT dimensions (`pep-clearance`, `scope-awareness`) omitted — the feedback service only writes attestations for `sar-drafting` this session. Permanently dormant quality floors confuse tutorial readers. Can be added when OSINT outcome feedback is implemented.

---

## Section 3: Programmatic Worker Additions (`AmlInvestigationCaseHub`)

Adds senior/junior variants for `sar-drafting` and `osint-screening`. Workers are registered programmatically in `AmlInvestigationCaseHub.augment()` — `aml-investigation.yaml` has no `workers:` section.

Both workers in each pair declare the same capability tag — the engine passes all matching candidates to `AgentRoutingStrategy.select()`.

| Worker | Capability | Behaviour |
|---|---|---|
| `sar-drafting-agent-senior` | `sar-drafting` | Full narrative + enhanced due diligence note + PEP flags |
| `sar-drafting-agent-junior` | `sar-drafting` | Minimal narrative — same structure, less analytical depth |
| `osint-screening-agent-senior` | `osint-screening` | Full PEP + sanctions screening, returns findings |
| `osint-screening-agent` (existing) | `osint-screening` | Declines — recast as the junior variant |

Single-candidate capabilities unchanged: `entity-resolution-agent`, `pattern-analysis-agent`, `senior-analyst-agent`.

---

## Section 4: `AmlTrustScoreSeeder`

**Location:** `io.casehub.aml.trust.AmlTrustScoreSeeder`  
**Annotations:** `@Startup @ApplicationScoped`

Seeds trust scores directly via `ActorTrustScoreRepository.upsert()` at application start — before `TrustScoreCache` hydrates. Idempotent: only seeds if no `ActorTrustScore` rows exist for the actor (production-restart safe).

`TrustBootstrapSource` SPI is NOT used here — it's designed for cross-deployment federation ("actor appears for the first time after having history elsewhere"). For pre-seeding known values, direct upsert is correct.

| Worker | α | β | Mean | obs | Routing outcome |
|---|---|---|---|---|---|
| `sar-drafting-agent-senior` | 9 | 1 | 0.90 | 10 | Selected — passes threshold 0.75 |
| `sar-drafting-agent-junior` | 2 | 8 | 0.20 | 10 | Excluded — below threshold |
| `osint-screening-agent-senior` | 9 | 2 | 0.82 | 11 | Selected — passes threshold 0.70 |
| `osint-screening-agent` | 3 | 7 | 0.30 | 10 | Excluded — below threshold |
| `entity-resolution-agent` | 8 | 2 | 0.80 | 10 | Selected (single candidate) |
| `pattern-analysis-agent` | 8 | 2 | 0.80 | 10 | Selected (single candidate) |
| `senior-analyst-agent` | 8 | 2 | 0.80 | 10 | Selected (single candidate) |

Seeds CAPABILITY score type per worker, keyed by capability tag matching their declared capability.

---

## Section 5: Post-Investigation Feedback

### `AmlWorkerDecisionRepository`

**Location:** `io.casehub.aml.trust.AmlWorkerDecisionRepository`  
Injects the qhorus EntityManager via the Quarkus idiom (not Jakarta `@PersistenceContext`):
```java
@Inject @io.quarkus.hibernate.orm.PersistenceUnit("qhorus") EntityManager em;
```
Query: find `WorkerDecisionEntry` by `caseId` and `capabilityTag`, ordered by `sequenceNumber DESC`. Returns the latest entry when multiple rows exist (parallel execution, retries). Returns empty if none found.

### `SarOutcomeFeedbackService`

**Location:** `io.casehub.aml.trust.SarOutcomeFeedbackService`

Given `caseId` + `SarOutcome`:
1. Query latest `WorkerDecisionEntry` for `capabilityTag = "sar-drafting"` and this `caseId`
2. If not found → log warning, return (no crash — investigation may have used a different path)
3. Write `io.casehub.ledger.runtime.model.LedgerAttestation` (the `@Entity`, not the `@MappedSuperclass`) via qhorus EntityManager:
   - `id` = `UUID.randomUUID()`
   - `ledgerEntryId` = `WorkerDecisionEntry.id`
   - `subjectId` = caseId
   - `attestorId` = `"aml-compliance-system"`
   - `attestorType` = `ActorType.SYSTEM`
   - `verdict` = `SOUND` (UPHELD) or `FLAGGED` (WITHDRAWN/FLAGGED)
   - `capabilityTag` = `"sar-drafting"`
   - `trustDimension` = `"investigation-accuracy"`
   - `dimensionScore` = `SarOutcome.investigationAccuracyScore`
   - `confidence` = `1.0`
   - `occurredAt` = `Instant.now()` (also set by `@PrePersist` — belt-and-suspenders)

### Input types (`api/` module — pure Java, no deps)

```java
public record SarOutcome(
    SarVerdict verdict,
    String reason,
    double investigationAccuracyScore   // 0.0–1.0
) {}

public enum SarVerdict { UPHELD, WITHDRAWN, FLAGGED }
```

`UPHELD` → `AttestationVerdict.SOUND`; `WITHDRAWN` / `FLAGGED` → `AttestationVerdict.FLAGGED`.

---

## Section 6: Layer 6 REST Resource

**Location:** `io.casehub.aml.engine.AmlLayer6Resource`  
**Path prefix:** `/api/layer6/investigations` (consistent with Layer 5 `/api/layer5/investigations`)

### `POST /api/layer6/investigations`

Delegates to `AmlEngineCoordinator.startInvestigation()`. Returns 202 with `{caseId}` immediately — investigation runs asynchronously on Quartz workers.

### `GET /api/layer6/investigations/{caseId}`

Polls for completion. Queries `AmlWorkerDecisionRepository` for all `WorkerDecisionEntry` rows for this `caseId`. If case is still in progress (no SAR drafting entry yet), returns `{caseId, status: "in-progress"}`. Once entries exist, assembles `Layer6InvestigationResponse`.

```java
public record Layer6InvestigationResponse(
    UUID caseId,
    String status,                          // "completed" | "in-progress"
    String sarNarrative,
    String complianceTaskId,
    List<WorkerRoutingDecision> routingDecisions
) {}

public record WorkerRoutingDecision(
    String capabilityTag,
    String selectedWorker,
    OptionalDouble trustScore    // from TrustScoreCache; empty if Phase 0 (no history)
) {}
```

`trustScore` populated by querying `TrustScoreCache.getCapabilityScore(workerId, capabilityName)` — shows the actual score the routing algorithm used. More accurate and instructive than name-suffix inference.

### `POST /api/layer6/investigations/{caseId}/outcome`

Accepts `SarOutcome`, delegates to `SarOutcomeFeedbackService`. Returns `204 No Content`.

---

## Section 7: Tests

### Unit (`api/` — pure JUnit 5)
- `SarOutcomeTest` — verdict-to-attestation mapping, record construction

### `@QuarkusTest` (`app/`)
- `AmlTrustRoutingPolicyProviderTest` — per-capability policies correct; unknown → DEFAULT; `Preferences` returns empty → hardcoded fallbacks
- `AmlTrustScoreSeederTest` — seeds correct CAPABILITY scores at startup; idempotent on restart (no second seed if rows exist)
- `SarOutcomeFeedbackServiceTest` — UPHELD writes SOUND attestation; FLAGGED writes FLAGGED; missing entry → graceful no-op
- `AmlLayer6ResourceTest` — POST returns 202; GET with awaitility returns routingDecisions with senior workers selected; POST outcome returns 204, attestation persisted

### SPI wiring (CDI `@ApplicationScoped` override pattern)
- `AmlTrustRoutingPolicyProviderWiringTest` — `@ApplicationScoped` wins over `DefaultTrustRoutingPolicyProvider @DefaultBean`

### Integration
- `AmlLayer6InvestigationIT` — full HTTP round-trip; GET with awaitility confirms senior workers in routing decisions; POST outcome triggers attestation visible in ledger

---

## Open Issues Filed This Session

| Issue | Repo | What |
|---|---|---|
| engine#395 | casehubio/engine | Flyway scoping violation — V2000/V2001 must move to `db/engine-ledger/migration/` |
