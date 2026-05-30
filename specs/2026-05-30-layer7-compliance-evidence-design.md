# Layer 7 — Compliance Evidence Design
**Date:** 2026-05-30 (amended post-review)
**Issue:** casehubio/aml#43
**Branch:** issue-43-layer7-comparison

---

## Problem

Layers 1–6 deliver five accountability properties the architecture claims against FinCEN/FATF
requirements. None are externally verifiable — there is no endpoint that surfaces the cryptographic
and structural evidence for a given investigation. A FinCEN examiner has nowhere to go.

A `verify()` boolean is not compliance evidence — it is a claim the service makes about itself.
An inclusion proof is evidence: the examiner reconstructs the Merkle root independently from the
siblings, without trusting the service.

Layer 7 closes this gap:
1. Fixes a prerequisite chain break (`causedByEntryId` never set on `COMPLIANCE_REVIEW_OPENED`)
2. Captures trust scores at routing time (currently only in a drifting cache)
3. Wires `LedgerErasureService` via an erasure endpoint
4. Exposes `GET /api/investigations/{caseId}/compliance-evidence` returning requirement-scoped
   evidence with Merkle inclusion proofs per key ledger event

**Future direction (out of scope):** sign the whole evidence response with the service's private
key for offline verification. `LedgerEntry` already carries `agentSignature`/`agentPublicKey`
anticipating this. `ComplianceEvidence` carries a nullable `signature` field as a forward signal.

---

## Prerequisite fix — `causedByEntryId` chain

`AmlLedgerService.writeComplianceReviewOpened()` currently leaves `causedByEntryId` null. The
`COMPLIANCE_REVIEW_OPENED` event is causally produced by `CASE_OPENED` and must say so.

**The threading constraint:** `AmlEngineCoordinator.startInvestigation()` calls
`writeCaseOpened()` and returns immediately. The `sar-drafting` worker (which eventually calls
`writeComplianceReviewOpened` via `ComplianceReviewLifecycle`) runs on a Quartz thread
asynchronously — `caseOpenedEntryId` is not in scope at that call site. Threading the ID through a
parameter is not viable for the Layer 5 path (though it would work for the Layer 3 synchronous
path). The fix must work for both.

**Fix:** derive `causedByEntryId` inside `writeComplianceReviewOpened` itself. No new parameter.

```java
public void writeComplianceReviewOpened(UUID caseId, String taskId) {
    UUID caseOpenedEntryId = repository
        .findBySubjectIdAndEventType(caseId, "CASE_OPENED")
        .map(e -> e.id).orElse(null);
    // ...
    entry.causedByEntryId = caseOpenedEntryId;
}
```

Requires `LedgerEntryRepository.findBySubjectIdAndEventType(UUID, String)` — add this query
method if it does not exist (JPQL over `AmlInvestigationLedgerEntry` filtered by `eventType`).
This approach is self-healing: works regardless of whether the caller is synchronous (Layer 3)
or asynchronous (Layer 5), and survives retries without state threading.

---

## New entity — `AmlTrustRoutingAttestation`

`WorkerDecisionEntry` records which worker was selected but not the trust score at routing time.
The cache drifts as attestations accumulate. For FATF R.20 evidence, the score must be captured
immutably at routing time.

**Package:** `app/.../trust/` (consistent with `AmlTrustScoreSeeder`, `SarOutcomeFeedbackService`)

```java
@Entity
@Table(name = "aml_trust_routing_attestation")
@DiscriminatorValue("AML_TRUST_ROUTING")
public class AmlTrustRoutingAttestation extends LedgerEntry {
    @Column(name = "capability_tag",       nullable = false) public String capabilityTag;
    @Column(name = "selected_worker_id",   nullable = false) public String selectedWorkerId;
    @Column(name = "trust_score_at_routing")                 public Double trustScoreAtRouting; // nullable — no data ≠ zero
    @Column(name = "threshold_applied",    nullable = false) public double thresholdApplied;
    @Column(name = "investigation_case_id",nullable = false) public UUID investigationCaseId;
}
```

`trustScoreAtRouting` is `Double` (nullable). `orElse(null)` when the cache has no entry —
"no trust data available" must not be conflated with "trust score was 0.0". The column is
nullable; the evidence report surfaces null explicitly.

**Migration:** `V2004__aml_trust_routing_attestation.sql`

**Future engine issue:** add `trustScoreAtRouting` and `thresholdApplied` natively to
`WorkerDecisionEntry` so `AmlTrustRoutingAttestation` becomes redundant.

---

## New observer — `AmlTrustRoutingObserver`

**Package:** `app/.../trust/`

Observes `WorkerDecisionEvent` (synchronous CDI, fired by engine on each worker dispatch). Reads
trust score from `TrustScoreCache` before any subsequent attestation cycle causes drift.

```java
@ApplicationScoped
public class AmlTrustRoutingObserver {
    @Inject TrustScoreCache trustScoreCache;
    @Inject AmlTrustRoutingPolicyProvider policyProvider;
    @Inject LedgerEntryRepository ledgerRepo;

    @Transactional(Transactional.TxType.REQUIRES_NEW)
    void onWorkerDecision(@Observes WorkerDecisionEvent event) {
        Double score = trustScoreCache
            .getCapabilityScore(event.workerId(), event.capabilityTag())
            .stream().boxed().findFirst().orElse(null);
        double threshold = policyProvider.forCapability(event.capabilityTag()).threshold();

        AmlTrustRoutingAttestation entry = new AmlTrustRoutingAttestation();
        entry.id = UUID.randomUUID();
        entry.subjectId = event.caseId();
        entry.investigationCaseId = event.caseId();
        entry.capabilityTag = event.capabilityTag();
        entry.selectedWorkerId = event.workerId();
        entry.trustScoreAtRouting = score;
        entry.thresholdApplied = threshold;
        entry.sequenceNumber = nextSequenceNumber(event.caseId());
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = "aml-orchestrator";
        entry.actorType = ActorType.SYSTEM;
        entry.actorRole = "AmlInvestigationOrchestrator";
        entry.occurredAt = Instant.now();
        ledgerRepo.save(entry);
    }
}
```

**`REQUIRES_NEW` and sequence number safety:** Layer 5 fires `pattern-analysis` and
`osint-screening` in parallel on the same context tick — `WorkerDecisionEvent` can fire
concurrently for the same `caseId`. `REQUIRES_NEW` gives each observer call its own transaction.
Combined with a pessimistic lock on `(subject_id)` in `nextSequenceNumber()` — or a DB
`SELECT ... FOR UPDATE` on the latest sequence row — this serializes writes per case without
blocking across cases. Implement `nextSequenceNumber()` with:

```java
em.createQuery("SELECT COALESCE(MAX(e.sequenceNumber), 0) FROM LedgerEntry e " +
               "WHERE e.subjectId = :subjectId")
  .setParameter("subjectId", subjectId)
  .setLockMode(LockModeType.PESSIMISTIC_WRITE)
  .getSingleResult()
```

**Observer failure mode (document for examiners):** if the observer fails (lock timeout, DB
error), the investigation continues but the attestation is lost. The compliance evidence
endpoint will show `trustRouting.status = PARTIAL` for that capability — this means the routing
decision was not captured, not that Layer 6 was inactive. Examiners should treat `PARTIAL` on
trust routing as an infrastructure failure, not an architectural gap. A reconciliation mechanism
(detect missing attestations at case completion) is not in Layer 7 scope but should be a
follow-on issue.

---

## API types in `api/`

New package: `io.casehub.aml.compliance`

### `RequirementStatus`

```java
public enum RequirementStatus { CLOSED, PARTIAL, BREACHED, GAP }
```

- `CLOSED` — requirement demonstrably met with evidence
- `PARTIAL` — mechanism present but evidence incomplete (chain exists but not verified;
  some but not all capabilities attested)
- `BREACHED` — mechanism present but obligation not met (SLA deadline passed without completion)
- `GAP` — architectural gap; requirement unaddressed

### Root response

```java
public record ComplianceEvidence(
    UUID caseId,
    Instant generatedAt,
    AuditChainRequirement auditChain,
    SlaRequirement sla,
    TrustRoutingRequirement trustRouting,
    GdprErasureRequirement gdprErasure,
    String signature          // null — reserved for future offline signing
) {}
```

### `AuditChainRequirement`

Covers 31 CFR §1020.320(a) (auditable evidence chain) and FATF R.16 (tamper-evident record).
Both requirements share the same ledger entries and inclusion proofs.

**Scope constraint (document explicitly in `mechanism` string):** these inclusion proofs cover
`AmlInvestigationLedgerEntry` records only — case lifecycle events (`CASE_OPENED`,
`COMPLIANCE_REVIEW_OPENED`). Specialist dispatch evidence (COMMAND/DONE/DECLINE per agent) lives
in the qhorus `MessageLedgerEntry` chain, which has a separate Merkle tree. Trust routing
attestations are in `AmlTrustRoutingAttestation`. An examiner reading this chain sees case
lifecycle events; the full specialist audit requires querying the qhorus ledger chain by
`subjectId = caseId`. This must be stated in `mechanism` — not left for an examiner to discover.

```java
public record AuditChainRequirement(
    String id,            // "FINCEN-31CFR1020.320-AUDIT-CHAIN"
    String citation,
    String mechanism,     // must state scope: "covers AML domain lifecycle events; specialist
                          // dispatch audit lives in qhorus ledger chain (subjectId = caseId)"
    RequirementStatus status,
    String treeRoot,
    boolean chainVerified,
    List<LedgerEventRecord> events
) {}

public record LedgerEventRecord(
    UUID entryId,
    String eventType,
    String actorId,
    String actorRole,
    Instant occurredAt,
    UUID causedByEntryId,     // null for CASE_OPENED; non-null for COMPLIANCE_REVIEW_OPENED
    String digest,
    AmlInclusionProof inclusionProof
) {}

public record AmlInclusionProof(
    int entryIndex,
    int treeSize,
    String leafHash,
    List<AmlProofStep> siblings,
    String treeRoot
) {}

public record AmlProofStep(String hash, String position) {} // position: "LEFT" | "RIGHT"
```

`status` logic:
- `CLOSED`: `chainVerified = true` AND `events[1].causedByEntryId` non-null
- `PARTIAL`: chain exists but `chainVerified = false`, OR `causedByEntryId` null on
  `COMPLIANCE_REVIEW_OPENED`
- `GAP`: no ledger entries for this `caseId`

**`LedgerVerificationService` in tests — required fix before implementation:** both
`JpaLedgerEntryRepository` and `JpaLedgerMerkleFrontierRepository` are `@Alternative`. Currently
only `JpaLedgerEntryRepository` appears in `quarkus.arc.selected-alternatives` in test
`application.properties`. `LedgerVerificationService` injects `LedgerMerkleFrontierRepository` —
CDI will fail to resolve it without also selecting `JpaLedgerMerkleFrontierRepository`. Add:

```properties
quarkus.arc.selected-alternatives=\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerMerkleFrontierRepository
```

Verify `LedgerVerificationService` injection succeeds in `@QuarkusTest` startup before writing
`chainVerified = true` as an assertion.

### `SlaRequirement`

```java
public record SlaRequirement(
    String id,            // "FINCEN-SAR-30DAY-SLA"
    String citation,
    String mechanism,
    RequirementStatus status,
    UUID workItemId,
    Instant claimDeadline,
    Instant completedAt,  // null if not yet completed
    boolean slaMet,
    List<String> candidateGroups,
    String escalationPolicy
) {}
```

`status` logic:
- `CLOSED`: `workItemId` non-null AND `claimDeadline` non-null AND `slaMet = true`
- `BREACHED`: `workItemId` non-null AND `claimDeadline` non-null AND `slaMet = false`
  (deadline passed without completion — a FinCEN obligation failure, not a partial state)
- `GAP`: no `COMPLIANCE_REVIEW_OPENED` entry found for this case

**WorkItem lookup:** `casehub-work-api` has no public query interface for reading a `WorkItem`
by ID. Do not inject `WorkItemStore` (internal implementation class). Use JPA directly:

```java
@Inject EntityManager em;  // default PU — WorkItem is on the default datasource

WorkItem item = em.find(WorkItem.class, UUID.fromString(taskId));
```

`WorkItem` is already in `quarkus.hibernate-orm.packages` and on the default datasource. This
requires no additional dependency. File casehubio/work issue for a public read API.

**UUID parse guard:** `transactionId` on `AmlInvestigationLedgerEntry` is typed as `String`.
The `COMPLIANCE_REVIEW_OPENED` event stores a WorkItem task UUID there, but the field is
structurally unconstrained. Wrap `UUID.fromString(transactionId)` in a try-catch
`IllegalArgumentException` — return `sla.status = GAP` if the value does not parse as a UUID
rather than propagating a runtime exception to the examiner.

### `TrustRoutingRequirement`

```java
public record TrustRoutingRequirement(
    String id,            // "FATF-R20-TRUST-ROUTING"
    String citation,
    String mechanism,
    RequirementStatus status,
    List<RoutingDecisionRecord> decisions
) {}

public record RoutingDecisionRecord(
    String capabilityTag,
    String selectedWorker,
    Double trustScoreAtRouting,  // nullable — null means no trust data was available at routing
    double thresholdApplied,
    UUID attestationEntryId
) {}
```

`status` logic — compare actual attestations against expected capabilities:
- `CLOSED`: attestations present for all capabilities defined in `AmlTrustRoutingPolicyProvider.POLICIES.keySet()`
- `PARTIAL`: at least one attestation present but not all expected capabilities covered
  (observer failure for some workers — see failure mode above)
- `GAP`: no attestations found

### `GdprErasureRequirement`

No `status` field — this is a capability descriptor, not per-case evidence. `status = CLOSED`
unconditionally would be a self-attestation, not evidence.

```java
public record GdprErasureRequirement(
    String id,            // "GDPR-ART17-ERASURE"
    String citation,
    String mechanism,
    boolean erasureCapabilityWired,
    boolean pseudonymizationActive,
    String erasureEndpoint
) {}
```

Both booleans are statically `true` — the capability is wired at build time. An examiner uses
these as named capability claims; verification is by invoking the erasure endpoint.

---

## New service — `AmlComplianceEvidenceService`

**Package:** `app/.../compliance/`

Assembly per requirement:

**Audit chain:** query `AmlInvestigationLedgerEntry` by `subjectId = caseId` ordered by
`sequenceNumber`. Call `LedgerVerificationService.verify(caseId)` and
`LedgerVerificationService.treeRoot(caseId)`. Call `LedgerVerificationService.inclusionProof(entryId)`
per entry. Project ledger `InclusionProof`/`ProofStep` to `AmlInclusionProof`/`AmlProofStep` —
confirm exact field names of `ProofStep` from decompiled ledger class during implementation.

**SLA:** find `AmlInvestigationLedgerEntry` with `eventType = 'COMPLIANCE_REVIEW_OPENED'`,
extract `transactionId` (stores WorkItem task ID — dual-use field per Layer 4). Fetch `WorkItem`
via `em.find(WorkItem.class, UUID.fromString(transactionId))`. Read `claimDeadline`,
`completedAt`, `candidateGroups`.

**Trust routing:** query `AmlTrustAttestationRepository.findByInvestigationCaseId(caseId)`.
Compare returned `capabilityTag` set against `AmlTrustRoutingPolicyProvider.POLICIES.keySet()`
to determine status.

**GDPR:** construct as constants.

Returns 404 if no `AmlInvestigationLedgerEntry` exists for `caseId`.

---

## New resource — `AmlLayer7Resource`

**Package:** `app/.../compliance/`

Domain vocabulary URLs — not layer-prefixed:

```
GET  /api/investigations/{caseId}/compliance-evidence → ComplianceEvidence (200 | 404)
POST /api/actors/{actorId}/erasure                   → ErasureResult      (200)
```

`GET` delegates to `AmlComplianceEvidenceService`.  
`POST /erasure` delegates to `LedgerErasureService.erase(actorId)`. Path parameter only, no body.

---

## Configuration changes

Both `application.properties` (main and test):

```properties
# AmlTrustRoutingAttestation entity — trust/ package
quarkus.hibernate-orm.qhorus.packages=...,io.casehub.aml.trust

# V2004 migration
quarkus.flyway.qhorus.locations=...,classpath:db/aml-trust-routing/migration
```

Test `application.properties` — add `JpaLedgerMerkleFrontierRepository` to selected
alternatives (required for `LedgerVerificationService`):

```properties
quarkus.arc.selected-alternatives=\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,\
  io.casehub.ledger.runtime.repository.jpa.JpaLedgerMerkleFrontierRepository
```

`AmlTrustRoutingObserver` must remain active in tests — do not add it to `exclude-types`.

---

## Testing

### Unit — `AmlComplianceEvidenceServiceTest`

No Quarkus. Stubs for `LedgerEntryRepository`, `LedgerVerificationService`, `EntityManager`
(for WorkItem lookup), `AmlTrustAttestationRepository`.

- Happy path: all four requirements return expected status; two ledger events; `causedByEntryId`
  non-null on second event; `inclusionProof` non-null on each
- `chainVerified = false` → `auditChain.status = PARTIAL`
- `completedAt` after `claimDeadline` → `sla.status = BREACHED`, `sla.slaMet = false`
- No `COMPLIANCE_REVIEW_OPENED` entry → `sla.status = GAP`
- Partial attestation set → `trustRouting.status = PARTIAL`
- No attestations → `trustRouting.status = GAP`
- `trustScoreAtRouting = null` when cache empty → surfaces as null in `RoutingDecisionRecord`

### `@QuarkusTest` — `AmlLayer7ResourceTest`

Full round-trip against H2. Prerequisites: `JpaLedgerMerkleFrontierRepository` in
`selected-alternatives` (verify CDI startup succeeds before writing assertions).

1. `POST /api/layer6/investigations` with a PEP transaction
2. Awaitility: poll `WORKER_SCHEDULED` events until `sar-drafting` fires
3. `GET /api/investigations/{caseId}/compliance-evidence`
4. Assert:
   - `auditChain.chainVerified = true`
   - `auditChain.events` has 2 entries; `events[1].causedByEntryId = events[0].entryId`
   - Each `inclusionProof.treeRoot` equals `auditChain.treeRoot`
   - `sla.workItemId` non-null; `sla.claimDeadline` approx 30 days from now; `sla.status = CLOSED`
   - `trustRouting.decisions` non-empty; each has `trustScoreAtRouting` non-null and `> 0.0`
   - `trustRouting.status = CLOSED` (all expected capabilities attested)
   - `gdprErasure.erasureCapabilityWired = true`

### `@QuarkusTest` — `AmlLayer7ErasureTest`

Verifies the GDPR erasure mechanism end-to-end. Uses `aml-orchestrator` as the erasure subject
because it is the only actorId that writes `LedgerEntry` rows in the current tutorial.

**Why not `analyst-alice`:** `WorkItemService.completeFromSystem()` writes to
`work_item_audit_entry` (work-internal audit store), not to `ledger_entry`. No `ActorIdentity`
row is created for `analyst-alice`. `LedgerErasureService.erase("analyst-alice")` would return
`mappingFound = false`, making the assertion useless.

**Why `aml-orchestrator` for a mechanism test:** `aml-orchestrator` writes `CASE_OPENED` and
`COMPLIANCE_REVIEW_OPENED` entries to `ledger_entry` — `ActorIdentityProvider.tokenise()` is
called on persist, creating an `ActorIdentity` row IF tokenisation is enabled. Erasure of this
actor replaces its token in `ActorIdentity`, pseudonymizing it in all audit entries.

**Tokenisation must be enabled in test config.** Check the default for
`casehub.ledger.identity.tokenisation.enabled` in `LedgerConfig`. If the default is `false`
(which is likely — `PassThroughActorIdentityProvider` is the no-op path), add to test
`application.properties`:
```properties
casehub.ledger.identity.tokenisation.enabled=true
```
Without this, `erase()` is a no-op regardless of actorId.

Test steps:
1. Run an investigation via Layer 6; await `sar-drafting` worker
2. `POST /api/actors/aml-orchestrator/erasure`
3. Assert: `mappingFound = true`, `affectedEntryCount >= 2`
4. Query `ledger_entry` table directly: `CASE_OPENED` and `COMPLIANCE_REVIEW_OPENED` rows still
   exist; `actor_id` column now holds the pseudonymous token (not `"aml-orchestrator"`)

**What this demonstrates:** the erasure mechanism works — audit structure is preserved while
actor identity is pseudonymized. It does not demonstrate the intended GDPR data subject scenario
(a human analyst or investigated entity). That requires a future layer that writes human actorIds
to `ledger_entry` rows. File casehubio/aml issue: add `AML_SAR_OFFICER_REVIEWED` ledger event
with `actorId = officer.assigneeId` when compliance WorkItem is completed.

### `@QuarkusTest` — `AmlTrustRoutingAttestationTest`

1. Run investigation via Layer 6
2. Await `sar-drafting` worker
3. Query `aml_trust_routing_attestation` table via `EntityManager`
4. Assert one row per fired capability tag; `trust_score_at_routing` non-null and `> 0.0`;
   `threshold_applied` matches `AmlTrustRoutingPolicyProvider` policy for that capability
5. Verify `investigationCaseId` matches the started case UUID

---

## Files created / modified

**New in `api/.../compliance/`:**
- `ComplianceEvidence.java`
- `RequirementStatus.java`
- `AuditChainRequirement.java`
- `LedgerEventRecord.java`
- `AmlInclusionProof.java`
- `AmlProofStep.java`
- `SlaRequirement.java`
- `TrustRoutingRequirement.java`
- `RoutingDecisionRecord.java`
- `GdprErasureRequirement.java`

**New in `app/.../trust/`:**
- `AmlTrustRoutingAttestation.java`
- `AmlTrustRoutingObserver.java`
- `AmlTrustAttestationRepository.java`
- `db/aml-trust-routing/migration/V2004__aml_trust_routing_attestation.sql`

**New in `app/.../compliance/`:**
- `AmlComplianceEvidenceService.java`
- `AmlLayer7Resource.java`

**Modified:**
- `app/.../ledger/AmlLedgerService.java` — `writeComplianceReviewOpened` derives
  `causedByEntryId` internally via `findBySubjectIdAndEventType` query; no new parameter
- `app/src/main/resources/application.properties` — package scan + Flyway location
- `app/src/test/resources/application.properties` — add `JpaLedgerMerkleFrontierRepository`
  to `selected-alternatives`; package scan + Flyway location

**Issues to file before implementation begins:**
- casehubio/engine: add `trustScoreAtRouting` + `thresholdApplied` to `WorkerDecisionEntry`
- casehubio/work: public read API for `WorkItem` by ID (currently no public query interface)
- casehubio/aml: observer failure leaves silent evidence gaps — reconciliation mechanism needed
