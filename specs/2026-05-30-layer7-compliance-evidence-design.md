# Layer 7 — Compliance Evidence Design
**Date:** 2026-05-30  
**Issue:** casehubio/aml#43  
**Branch:** issue-43-layer7-comparison

---

## Problem

Layers 1–6 deliver five accountability properties the architecture claims against FinCEN/FATF
requirements. None of those properties are currently externally verifiable — there is no endpoint
that surfaces the cryptographic and structural evidence for a given investigation. A FinCEN
examiner asking "prove this investigation happened correctly" has nowhere to go.

The `verify()` boolean available from `LedgerVerificationService` is not compliance evidence — it
is a claim the service makes about itself. An inclusion proof is evidence: the examiner takes the
siblings and reconstructs the Merkle root independently, without trusting the service.

Layer 7 closes this gap by:
1. Fixing a prerequisite chain break (`causedByEntryId` never set on `COMPLIANCE_REVIEW_OPENED`)
2. Capturing trust scores at routing time (currently only available from a drifting cache)
3. Wiring `LedgerErasureService` via a GDPR erasure endpoint
4. Exposing a `GET /api/layer7/investigations/{caseId}/compliance-evidence` endpoint that returns
   requirement-scoped evidence with Merkle inclusion proofs per key ledger event

Future direction (out of scope): sign the whole evidence response with the service's private key,
enabling fully offline verification. `LedgerEntry` already carries `agentSignature`/`agentPublicKey`
fields anticipating this. Filed as a follow-on issue.

---

## Prerequisite fix — `causedByEntryId` chain

`AmlLedgerService.writeComplianceReviewOpened()` currently ignores `causedByEntryId`. The
`COMPLIANCE_REVIEW_OPENED` entry is causally produced by the `CASE_OPENED` event — that link must
be explicit in the ledger before any audit chain evidence can be honest.

**Change:** add `UUID caseOpenedEntryId` parameter; set `entry.causedByEntryId = caseOpenedEntryId`.

```java
// Before
public void writeComplianceReviewOpened(UUID caseId, String taskId)

// After
public void writeComplianceReviewOpened(UUID caseId, String taskId, UUID caseOpenedEntryId)
```

`AmlEngineCoordinator` threads the `caseOpenedEntryId` returned by `AmlLedgerService.writeCaseOpened()`
through to this call. `DefaultAmlInvestigationService.noOp()` / `.stub()` stubs updated accordingly.

---

## New entity — `AmlTrustRoutingAttestation`

`WorkerDecisionEntry` (engine-ledger) records which worker was selected but not the trust score
that drove the decision. The cache score drifts as attestations accumulate. For FATF R.20 compliance
evidence, the score at routing time must be captured immutably.

**Class:** `app/.../compliance/AmlTrustRoutingAttestation extends LedgerEntry`

```java
@Entity
@Table(name = "aml_trust_routing_attestation")
@DiscriminatorValue("AML_TRUST_ROUTING")
public class AmlTrustRoutingAttestation extends LedgerEntry {
    @Column(name = "capability_tag", nullable = false)  public String capabilityTag;
    @Column(name = "selected_worker_id", nullable = false) public String selectedWorkerId;
    @Column(name = "trust_score_at_routing", nullable = false) public double trustScoreAtRouting;
    @Column(name = "threshold_applied", nullable = false) public double thresholdApplied;
    @Column(name = "investigation_case_id", nullable = false) public UUID investigationCaseId;
}
```

**Migration:** `V2004__aml_trust_routing_attestation.sql`

**`subjectId`** is set to `investigationCaseId` so the attestation appears in the subject's ledger
chain alongside `AmlInvestigationLedgerEntry` records. `sequenceNumber` follows the same
`nextSequenceNumber()` logic used by `AmlLedgerService`.

**Future engine issue:** add `trustScoreAtRouting` and `thresholdApplied` directly to
`WorkerDecisionEntry` so `AmlTrustRoutingAttestation` becomes redundant.

---

## New observer — `AmlTrustRoutingObserver`

Observes `WorkerDecisionEvent` (CDI, fired by engine on each worker dispatch). Reads the
current score from `TrustScoreCache` at event time — before any subsequent attestation cycle
can cause drift — and writes `AmlTrustRoutingAttestation` to the ledger.

```java
@ApplicationScoped
public class AmlTrustRoutingObserver {
    @Inject TrustScoreCache trustScoreCache;
    @Inject AmlTrustRoutingPolicyProvider policyProvider;
    @Inject LedgerEntryRepository ledgerRepo;

    @Transactional
    void onWorkerDecision(@Observes WorkerDecisionEvent event) {
        double score = trustScoreCache
            .getCapabilityScore(event.workerId(), event.capabilityTag())
            .orElse(0.0);
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

**Transaction boundary:** `@Transactional` on the observer — the attestation must be committed
before the evidence endpoint reads it. `WorkerDecisionEvent` is a synchronous CDI event.

**Edge case — zero score:** if `TrustScoreCache` has no entry for this worker/capability pair
(can happen on a fresh deployment if the seeder hasn't run), the score is recorded as `0.0` and
the attestation is written. The evidence report reflects the actual value used for routing, not
a sanitised one.

**Concurrency risk — sequence numbers:** Layer 5 fires parallel workers for `pattern-analysis`
and `osint-screening` simultaneously. If the engine dispatches their `WorkerDecisionEvent`s on
concurrent threads, two observer calls share the same `caseId` subject and `nextSequenceNumber()`
(which reads the DB max, not concurrent-safe) may produce duplicate values, causing a constraint
violation on `(subject_id, sequence_number)`. Implementation must verify whether the engine
serializes binding dispatch for a given case. If concurrent dispatch is confirmed, use a
pessimistic DB lock in `nextSequenceNumber()` or delegate sequence assignment to the DB.

---

## API types in `api/`

New package: `io.casehub.aml.compliance`

### Root response

```java
public record ComplianceEvidence(
    UUID caseId,
    Instant generatedAt,
    AuditChainRequirement auditChain,
    SlaRequirement sla,
    TrustRoutingRequirement trustRouting,
    GdprErasureRequirement gdprErasure
) {}
```

### `RequirementStatus`

```java
public enum RequirementStatus { CLOSED, PARTIAL, GAP }
```

`CLOSED` — requirement is demonstrably met with evidence.  
`PARTIAL` — requirement is partially met; evidence incomplete (e.g. chain present but not verified).  
`GAP` — architectural gap; requirement is not addressed.

### `AuditChainRequirement`

Covers 31 CFR §1020.320(a) (auditable evidence chain) and FATF R.16 (tamper-evident record) in
one record — both requirements are evidenced by the same ledger entries and inclusion proofs.

```java
public record AuditChainRequirement(
    String id,                     // "FINCEN-31CFR1020.320-AUDIT-CHAIN"
    String citation,
    String mechanism,
    RequirementStatus status,
    String treeRoot,               // Merkle root for this caseId's subject chain
    boolean chainVerified,         // LedgerVerificationService.verify(caseId)
    List<LedgerEventRecord> events // CASE_OPENED then COMPLIANCE_REVIEW_OPENED
) {}

public record LedgerEventRecord(
    UUID entryId,
    String eventType,
    String actorId,
    String actorRole,
    Instant occurredAt,
    UUID causedByEntryId,          // null for CASE_OPENED; non-null for COMPLIANCE_REVIEW_OPENED
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

public record AmlProofStep(
    String hash,
    String position                // "LEFT" or "RIGHT"
) {}
```

`status` logic: `CLOSED` if `chainVerified = true` AND all entries have `causedByEntryId` set as
expected. `PARTIAL` if chain present but verification fails or `causedByEntryId` is null on
`COMPLIANCE_REVIEW_OPENED`. `GAP` if no ledger entries exist for this caseId.

### `SlaRequirement`

```java
public record SlaRequirement(
    String id,                     // "FINCEN-SAR-30DAY-SLA"
    String citation,
    String mechanism,
    RequirementStatus status,
    UUID workItemId,
    Instant claimDeadline,
    Instant completedAt,           // null if WorkItem not yet completed
    boolean slaMet,                // completedAt != null && completedAt.isBefore(claimDeadline)
    List<String> candidateGroups,
    String escalationPolicy        // "senior-compliance-officers after claimDeadline breach"
) {}
```

`status` logic: `CLOSED` if `workItemId` non-null and `claimDeadline` non-null. `PARTIAL` if
WorkItem found but already past deadline without completion. `GAP` if no `COMPLIANCE_REVIEW_OPENED`
ledger entry exists for this case.

### `TrustRoutingRequirement`

```java
public record TrustRoutingRequirement(
    String id,                            // "FATF-R20-TRUST-ROUTING"
    String citation,
    String mechanism,
    RequirementStatus status,
    List<RoutingDecisionRecord> decisions
) {}

public record RoutingDecisionRecord(
    String capabilityTag,
    String selectedWorker,
    double trustScoreAtRouting,
    double thresholdApplied,
    UUID attestationEntryId               // UUID of AmlTrustRoutingAttestation ledger entry
) {}
```

`status` logic: `CLOSED` if at least one `RoutingDecisionRecord` present. `GAP` if no attestations
found (engine never fired `WorkerDecisionEvent` for this case — indicates Layer 6 not active).

### `GdprErasureRequirement`

```java
public record GdprErasureRequirement(
    String id,                     // "GDPR-ART17-ERASURE"
    String citation,
    String mechanism,
    RequirementStatus status,
    boolean erasureCapabilityWired,
    boolean pseudonymizationActive,
    String erasureEndpoint         // "POST /api/layer7/actors/{actorId}/erasure"
) {}
```

`erasureCapabilityWired` is always `true` — the capability is wired at build time.
`pseudonymizationActive` is always `true` — `ActorIdentityProvider` is registered.
`status` is always `CLOSED` for this requirement. These fields are explicit rather than implied
to give an examiner a named hook to verify: "yes, the service claims this capability is live."

---

## New service — `AmlComplianceEvidenceService`

**Assembly logic per requirement:**

**Audit chain:** Query `AmlInvestigationLedgerEntry` by `subjectId = caseId` ordered by
`sequenceNumber`. Call `LedgerVerificationService.verify(caseId)` for `chainVerified`.
Call `LedgerVerificationService.inclusionProof(entryId)` per entry. Project ledger's
`InclusionProof` + `ProofStep` to `AmlInclusionProof` + `AmlProofStep`. Confirm the exact field
names of `ProofStep` from the decompiled ledger class during implementation.

**SLA:** Find entry with `eventType = 'COMPLIANCE_REVIEW_OPENED'`. Extract `transactionId`
(stores the WorkItem task ID — dual-use field documented in Layer 4). Inject `WorkItemStore`
directly (not `WorkItemService`); call `workItemStore.get(UUID.fromString(transactionId))`.
Read `claimDeadline`, `completedAt`, `candidateGroups` from the returned `WorkItem`.

**Trust routing:** Query `AmlTrustAttestationRepository.findByInvestigationCaseId(caseId)`.
Map each `AmlTrustRoutingAttestation` to a `RoutingDecisionRecord`.

**GDPR:** Construct `GdprErasureRequirement` as constants — capability is statically wired.

---

## New resource — `AmlLayer7Resource`

```
GET  /api/layer7/investigations/{caseId}/compliance-evidence → ComplianceEvidence (200)
POST /api/layer7/actors/{actorId}/erasure                   → ErasureResult      (200)
```

`GET` delegates entirely to `AmlComplianceEvidenceService`. Returns 404 if no ledger entries
exist for `caseId`.

`POST /erasure` delegates to `LedgerErasureService.erase(actorId)`. Returns the
`ErasureResult` record directly (`rawActorId`, `mappingFound`, `affectedEntryCount`). No body
required — `actorId` is the path parameter.

---

## Configuration changes

Both `application.properties` (main and test):

```properties
# AmlTrustRoutingAttestation entity package
quarkus.hibernate-orm.qhorus.packages=...,io.casehub.aml.compliance

# V2004 migration
quarkus.flyway.qhorus.locations=...,classpath:db/aml-trust-routing/migration
```

`quarkus.arc.exclude-types` in test `application.properties`: no new exclusions needed —
`AmlTrustRoutingObserver` must remain active in tests so attestations are written during the
round-trip test.

---

## Testing

### Unit — `AmlComplianceEvidenceServiceTest`

No Quarkus. Stubs for `LedgerEntryRepository`, `LedgerVerificationService`, `WorkItemStore`,
`AmlTrustAttestationRepository`.

- Happy path: all four requirements `CLOSED`, two ledger events, `causedByEntryId` non-null
  on second event, `inclusionProof` non-null on each
- `chainVerified = false` → `auditChain.status = PARTIAL`
- `completedAt` after `claimDeadline` → `sla.slaMet = false`
- No attestations → `trustRouting.status = GAP`
- No `COMPLIANCE_REVIEW_OPENED` entry → `sla.status = GAP`

### `@QuarkusTest` — `AmlLayer7ResourceTest`

Full round-trip against H2:
1. `POST /api/layer6/investigations` with a PEP transaction
2. Awaitility: poll `WORKER_SCHEDULED` events until `sar-drafting` fires
3. `GET /api/layer7/investigations/{caseId}/compliance-evidence`
4. Assert:
   - `auditChain.chainVerified = true`
   - `auditChain.events` has exactly 2 entries; `events[1].causedByEntryId = events[0].entryId`
   - Each entry's `inclusionProof.treeRoot` equals `auditChain.treeRoot`
   - `sla.workItemId` non-null; `sla.claimDeadline` approximately 30 days from now
   - `trustRouting.decisions` non-empty; each has `trustScoreAtRouting > 0.0`
   - `gdprErasure.erasureCapabilityWired = true`

### `@QuarkusTest` — `AmlLayer7ErasureTest`

1. Run an investigation (writes entries under `actorId = "aml-orchestrator"`)
2. `POST /api/layer7/actors/aml-orchestrator/erasure`
3. Assert response: `mappingFound = true`, `affectedEntryCount > 0`
4. Query `ledger_entry` table directly: entries still exist (audit structure preserved)

### `@QuarkusTest` — `AmlTrustRoutingAttestationTest`

1. Run an investigation via Layer 6 endpoint
2. Await `sar-drafting` worker scheduled
3. Query `aml_trust_routing_attestation` table via `EntityManager`
4. Assert one row per fired capability tag; `trust_score_at_routing > 0.0`; `threshold_applied`
   matches `AmlTrustRoutingPolicyProvider` policy for that capability

---

## Files created / modified

**New in `api/`:**
- `api/.../compliance/ComplianceEvidence.java`
- `api/.../compliance/RequirementStatus.java`
- `api/.../compliance/AuditChainRequirement.java`
- `api/.../compliance/LedgerEventRecord.java`
- `api/.../compliance/AmlInclusionProof.java`
- `api/.../compliance/AmlProofStep.java`
- `api/.../compliance/SlaRequirement.java`
- `api/.../compliance/TrustRoutingRequirement.java`
- `api/.../compliance/RoutingDecisionRecord.java`
- `api/.../compliance/GdprErasureRequirement.java`

**New in `app/`:**
- `app/.../compliance/AmlTrustRoutingAttestation.java`
- `app/.../compliance/AmlTrustRoutingObserver.java`
- `app/.../compliance/AmlTrustAttestationRepository.java`
- `app/.../compliance/AmlComplianceEvidenceService.java`
- `app/.../compliance/AmlLayer7Resource.java`
- `app/src/main/resources/db/aml-trust-routing/migration/V2004__aml_trust_routing_attestation.sql`

**Modified:**
- `app/.../ledger/AmlLedgerService.java` — `writeComplianceReviewOpened` gains `caseOpenedEntryId` param
- `app/.../engine/AmlEngineCoordinator.java` — threads `caseOpenedEntryId` through to `AmlLedgerService`
- `app/src/main/resources/application.properties` — package scan + Flyway location
- `app/src/test/resources/application.properties` — package scan + Flyway location

**Future issue:** casehubio/engine — add `trustScoreAtRouting` + `thresholdApplied` to
`WorkerDecisionEntry` so `AmlTrustRoutingAttestation` becomes redundant.
