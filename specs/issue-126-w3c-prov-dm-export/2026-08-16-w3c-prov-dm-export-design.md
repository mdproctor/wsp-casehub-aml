# W3C PROV-DM Export for Investigation Lineage

**Issue:** casehubio/aml#126
**Epic:** casehubio/aml#7 (GDPR and regulatory audit)
**Date:** 2026-08-16

## Problem

The `causedByEntryId` chain and Merkle inclusion proofs already exist in the ledger. What's missing is a standardised export format that makes investigation provenance interoperable and independently auditable outside CaseHub. A FinCEN examiner should be able to reconstruct the full investigation decision chain — who did what, when, and why — without CaseHub access.

## Scope

Export the complete investigation provenance graph for a given case in W3C PROV-DM format (PROV-JSON serialisation).

**Included:** All ledger entries sharing `subjectId = caseId`:
- AML domain entries (8 types): `AmlCaseOpenedLedgerEntry`, `AmlComplianceReviewLedgerEntry`, `AmlSarOfficerReviewedLedgerEntry`, `AmlSupervisorDecisionLedgerEntry`, `AmlCaseProfileLedgerEntry`, `AmlCbrAdvisoryLedgerEntry`, `AmlTrustRoutingAttestation`, plus any future AML entry types
- Engine entries: `CaseLedgerEntry`, `WorkerDecisionEntry`
- Qhorus entries: `MessageLedgerEntry`

**Excluded:** `AmlEntityErasureLedgerEntry` — uses a different `subjectId` namespace (`UUID.nameUUIDFromBytes("aml-entity-erasure:" + entityId)`). It's a post-investigation GDPR action, not part of the investigation's causal chain.

## Data Flow

1. `GET /api/investigations/{caseId}/provenance` → `AmlProvenanceResource`
2. Resource delegates to `AmlProvenanceService`
3. Service queries `LedgerEntryRepository.findBySubjectId(caseId, tenancyId)` → flat list of entries
4. Service queries `LedgerVerificationService.inclusionProof(entryId, tenancyId)` per entry → Merkle proofs
5. Service passes entries + proofs to `ProvDmMapper` — pure function, no CDI
6. Mapper returns a `ProvDocument` record (the PROV-JSON model)
7. Resource serializes `ProvDocument` as JSON response (200 OK, or 404 if no entries found)

## PROV-DM Mapping Model

Each `LedgerEntry` maps to three PROV-DM constructs: an Entity (the record), an Activity (the event), and an Agent (the actor). This is the idiomatic PROV-DM pattern for audit logs — separating "what was recorded" from "what happened" from "who did it."

### prov:Entity (the tamper-evident record)

- **ID:** `casehub:entry-{entry.id}`
- `prov:type`: mapped from discriminator value (see Type Mapping Table)
- `prov:generatedAtTime`: `entry.occurredAt`
- `casehub:sequenceNumber`: `entry.sequenceNumber`
- `casehub:digest`: `entry.digest`
- `casehub:merkleInclusionProof`: proof hash list from `LedgerVerificationService`

### prov:Activity (the event it records)

- **ID:** `aml:activity-{entry.id}`
- `prov:type`: mapped from discriminator (see Type Mapping Table)
- `prov:startTime`: `entry.occurredAt`
- Domain-specific attributes vary by entry type (see Domain Attribute Table)

### prov:Agent (the actor)

- **ID:** `aml:agent-{entry.actorId}` (deduplicated — same `actorId` produces one Agent node)
- `prov:type`: `prov:SoftwareAgent` for SYSTEM/AGENT, `prov:Person` for HUMAN
- `casehub:actorType`: SYSTEM / HUMAN / AGENT
- `casehub:actorRole`: `entry.actorRole`

### Relations

| Relation | Meaning | Source |
|----------|---------|--------|
| `prov:wasGeneratedBy` | Entity → Activity | Each Entity linked to its paired Activity |
| `prov:wasAssociatedWith` | Activity → Agent | Each Activity linked to its Agent |
| `prov:wasAttributedTo` | Entity → Agent | Each Entity linked to its Agent |
| `prov:wasDerivedFrom` | Entity → Entity | Following `causedByEntryId` (only where non-null) |

### Namespace Prefixes

| Prefix | URI | Purpose |
|--------|-----|---------|
| `prov` | `http://www.w3.org/ns/prov#` | W3C PROV-DM standard |
| `casehub` | `urn:casehub:ledger:` | Tamper-evidence attributes (digest, proof, sequence) |
| `aml` | `urn:casehub:aml:` | Domain-specific types and attributes |

### Type Mapping Table

| Discriminator | Entity prov:type | Activity prov:type |
|---------------|-----------------|-------------------|
| `AML_CASE_OPENED` | `aml:CaseOpenedRecord` | `aml:CaseOpening` |
| `AML_COMPLIANCE_REVIEW` | `aml:ComplianceReviewRecord` | `aml:ComplianceReviewOpening` |
| `AML_SAR_OFFICER_REVIEWED` | `aml:SarOfficerReviewRecord` | `aml:SarOfficerReview` |
| `AML_SUPERVISOR_DECISION` | `aml:SupervisorDecisionRecord` | `aml:SupervisorDecision` |
| `AML_CASE_PROFILE` | `aml:CaseProfileRecord` | `aml:CaseProfileCapture` |
| `AML_CBR_ADVISORY` | `aml:CbrAdvisoryRecord` | `aml:CbrAdvisoryGeneration` |
| `AML_TRUST_ATTESTATION` | `aml:TrustAttestationRecord` | `aml:TrustAttestation` |
| `CASE_LEDGER` | `aml:CaseLedgerRecord` | `aml:CaseLifecycleEvent` |
| `WORKER_DECISION` | `aml:WorkerDecisionRecord` | `aml:AgentRoutingDecision` |
| `QHORUS_MESSAGE` | `aml:MessageRecord` | `aml:SpecialistCommunication` |
| *(unknown)* | `aml:LedgerRecord` | `aml:LedgerEvent` |

The unknown/fallback row ensures new entry types added in foundation modules are still exported — they appear with generic types rather than being silently dropped.

### Domain Attribute Table

| Entry type | Activity attributes |
|------------|-------------------|
| `AmlCaseOpenedLedgerEntry` | `aml:transactionId`, `aml:originAccountId`, `aml:destinationAccountId` |
| `AmlComplianceReviewLedgerEntry` | `aml:taskId` |
| `AmlSarOfficerReviewedLedgerEntry` | `aml:reviewDecision`, `aml:rejectionReason` |
| `AmlSupervisorDecisionLedgerEntry` | `aml:selectedBindings`, `aml:suppressedBindings`, `aml:rationale`, `aml:earlyTermination`, `aml:eligibleCount`, `aml:degraded` |
| `AmlCaseProfileLedgerEntry` | *(fields TBD — read from class at implementation time)* |
| `AmlCbrAdvisoryLedgerEntry` | *(fields TBD — read from class at implementation time)* |
| `AmlTrustRoutingAttestation` | *(fields TBD — read from class at implementation time)* |
| `CaseLedgerEntry` | *(fields from engine — read at implementation time)* |
| `WorkerDecisionEntry` | `aml:routingRationale` + *(remaining fields read at implementation time)* |
| `MessageLedgerEntry` | *(fields from qhorus — read at implementation time)* |
| *(unknown)* | No domain attributes — base fields only |

Fields marked "TBD" are resolved during implementation by reading the concrete class. The mapping is mechanical: each non-transient persistent field becomes a namespaced attribute.

## GDPR-Erased Actors

No special handling required. The ledger's existing identity tokenisation replaces `actorId` with a pseudonym before the export reads it. The pseudonymised ID becomes the `prov:Agent` identifier. `casehub:actorType = HUMAN` combined with a pseudonymised ID signals that erasure has occurred. SYSTEM and AGENT actors are never pseudonymised (not natural persons).

## Merkle Inclusion Proofs

Each `prov:Entity` carries `casehub:digest` (the entry's content hash) and `casehub:merkleInclusionProof` (the hash path from the entry to the Merkle tree root). This allows a FinCEN examiner to independently verify that no entry was tampered with — without CaseHub access.

The proof is fetched via `LedgerVerificationService.inclusionProof(entryId, tenancyId)`. If the hash chain is disabled (test environments), the proof attributes are omitted.

## Serialization Format

PROV-JSON only, per W3C PROV-JSON specification. Additional formats (PROV-N, PROV-XML) can be added later via content negotiation without changing the domain model.

## Implementation Architecture

**Package:** `io.casehub.aml.provenance`

| Class | Role |
|-------|------|
| `ProvDocument` | Record — the PROV-JSON model. Fields: `prefix`, `entity`, `activity`, `agent`, `wasGeneratedBy`, `wasAssociatedWith`, `wasAttributedTo`, `wasDerivedFrom`. Each is a `Map<String, Map<String, Object>>` matching PROV-JSON structure. |
| `ProvDmMapper` | Pure function — `ProvDocument map(List<LedgerEntry> entries, Map<UUID, InclusionProof> proofs)`. No CDI, no state. Uses `instanceof` pattern matching for entry-type-specific attribute extraction. |
| `AmlProvenanceService` | `@ApplicationScoped` — injects `LedgerEntryRepository`, `LedgerVerificationService`. Queries entries by `subjectId`, fetches proofs, delegates to mapper. |
| `AmlProvenanceResource` | `@Path("/api/investigations/{caseId}/provenance")` — thin JAX-RS resource. Delegates to service, returns 200 or 404. |

**Why `instanceof` in the mapper:** A visitor would require modifying foundation `LedgerEntry` subclasses we don't own. An SPI adds framework machinery for a simple type-switch over ~15 known discriminators. `instanceof` with pattern matching is explicit, local, and a new entry type triggers a fallback to generic mapping rather than a compilation error.

## Testing Strategy

### Unit tests (no Quarkus)

**`ProvDmMapperTest`** — the highest-value test class. The mapper is a pure function:
- One test per entry type verifying attribute extraction
- Empty entry list → empty ProvDocument
- Single entry (no `causedByEntryId`) → no `wasDerivedFrom` edges
- Chain of 3 entries → correct `wasDerivedFrom` edges
- Agent deduplication — same `actorId` produces one Agent node
- Null `causedByEntryId` → omitted from `wasDerivedFrom` (not mapped to null)
- Unknown entry type → generic `aml:LedgerRecord` / `aml:LedgerEvent` fallback

### @QuarkusTest (integration)

**`AmlProvenanceResourceTest`** — HTTP round-trip:
- Start investigation via existing Layer 9 path, drain to `status=completed`
- `GET /api/investigations/{caseId}/provenance` → 200 with PROV-JSON
- Response contains expected entry types (at minimum: `AML_CASE_OPENED`, `AML_COMPLIANCE_REVIEW`, `WORKER_DECISION`)
- `wasDerivedFrom` chain is present
- Agents are populated with correct `actorType`
- Unknown `caseId` → 404

Follows existing test conventions: drain to completion before asserting, `TenancyConstants.DEFAULT_TENANT_ID`, standard CDI exclusions per CLAUDE.md.

### Not tested here

- Merkle proof correctness (casehub-ledger owns this)
- GDPR tokenisation (casehub-ledger owns this)
- Entry creation and chain integrity (existing Layer 4-9 tests)

This feature is read-only — it queries the ledger and maps. The tests verify the query and mapping, not the write path.
