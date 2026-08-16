# W3C PROV-DM Export Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #126 — W3C PROV-DM export for investigation lineage
**Issue group:** #126

**Goal:** Export the complete investigation provenance graph for a given case as W3C PROV-JSON, enabling FinCEN examiners to independently verify the investigation decision chain.

**Architecture:** A pure-function mapper (`ProvDmMapper`) converts a flat list of `LedgerEntry` instances into a `ProvDocument` record structured per W3C PROV-JSON. A CDI service (`AmlProvenanceService`) queries entries and Merkle proofs, then delegates to the mapper. A thin JAX-RS endpoint (`AmlProvenanceResource`) exposes `GET /api/investigations/{caseId}/provenance`.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-ledger (LedgerEntryRepository, LedgerVerificationService, InclusionProof), casehub-engine-ledger (CaseLedgerEntry, WorkerDecisionEntry), casehub-qhorus (MessageLedgerEntry)

## Global Constraints

- All entries queried via `LedgerEntryRepository.findBySubjectId(caseId, TenancyConstants.DEFAULT_TENANT_ID)`
- `AmlEntityErasureLedgerEntry` excluded (different subjectId namespace)
- `InclusionProof` fetched via `LedgerVerificationService.inclusionProof(entryId, tenancyId)` — returns null/throws when hash chain disabled
- PROV-JSON serialization only (no PROV-N, PROV-XML)
- Package: `io.casehub.aml.provenance`
- Tests follow CLAUDE.md conventions: drain to `status=completed`, `TenancyConstants.DEFAULT_TENANT_ID`, standard CDI exclusions, `casehub.ledger.hash-chain.enabled=false` in test
- Use `mcp__intellij-index__*` tools for all code navigation and structural editing

---

### Task 1: ProvDocument model and ProvDmMapper (pure-function mapping)

**Files:**
- Create: `app/src/main/java/io/casehub/aml/provenance/ProvDocument.java`
- Create: `app/src/main/java/io/casehub/aml/provenance/ProvDmMapper.java`
- Create: `app/src/test/java/io/casehub/aml/provenance/ProvDmMapperTest.java`

**Interfaces:**
- Consumes: `io.casehub.ledger.api.model.LedgerEntry` (base class), all AML subclasses, `io.casehub.ledger.model.CaseLedgerEntry`, `io.casehub.ledger.model.WorkerDecisionEntry`, `io.casehub.qhorus.runtime.ledger.MessageLedgerEntry`, `io.casehub.ledger.runtime.service.model.InclusionProof`
- Produces: `ProvDocument` record consumed by Task 2's `AmlProvenanceService`

- [ ] **Step 1: Write ProvDocument record**

```java
package io.casehub.aml.provenance;

import java.util.Map;

public record ProvDocument(
    Map<String, String> prefix,
    Map<String, Map<String, Object>> entity,
    Map<String, Map<String, Object>> activity,
    Map<String, Map<String, Object>> agent,
    Map<String, Map<String, Object>> wasGeneratedBy,
    Map<String, Map<String, Object>> wasAssociatedWith,
    Map<String, Map<String, Object>> wasAttributedTo,
    Map<String, Map<String, Object>> wasDerivedFrom
) {}
```

- [ ] **Step 2: Write failing test — empty entry list produces empty document**

```java
package io.casehub.aml.provenance;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class ProvDmMapperTest {

    private final ProvDmMapper mapper = new ProvDmMapper();

    @Test
    void emptyEntryList_producesDocumentWithPrefixesOnly() {
        ProvDocument doc = mapper.map(List.of(), Map.of());
        assertNotNull(doc);
        assertEquals(3, doc.prefix().size());
        assertEquals("http://www.w3.org/ns/prov#", doc.prefix().get("prov"));
        assertEquals("urn:casehub:ledger:", doc.prefix().get("casehub"));
        assertEquals("urn:casehub:aml:", doc.prefix().get("aml"));
        assertTrue(doc.entity().isEmpty());
        assertTrue(doc.activity().isEmpty());
        assertTrue(doc.agent().isEmpty());
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn test -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am -Dtest=ProvDmMapperTest#emptyEntryList_producesDocumentWithPrefixesOnly -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `ProvDmMapper` does not exist

- [ ] **Step 4: Write ProvDmMapper — skeleton with empty-list handling**

```java
package io.casehub.aml.provenance;

import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.runtime.service.model.InclusionProof;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

public class ProvDmMapper {

    private static final Map<String, String> PREFIXES = Map.of(
        "prov", "http://www.w3.org/ns/prov#",
        "casehub", "urn:casehub:ledger:",
        "aml", "urn:casehub:aml:"
    );

    public ProvDocument map(List<? extends LedgerEntry> entries, Map<UUID, InclusionProof> proofs) {
        Map<String, Map<String, Object>> entities = new LinkedHashMap<>();
        Map<String, Map<String, Object>> activities = new LinkedHashMap<>();
        Map<String, Map<String, Object>> agents = new LinkedHashMap<>();
        Map<String, Map<String, Object>> wasGeneratedBy = new LinkedHashMap<>();
        Map<String, Map<String, Object>> wasAssociatedWith = new LinkedHashMap<>();
        Map<String, Map<String, Object>> wasAttributedTo = new LinkedHashMap<>();
        Map<String, Map<String, Object>> wasDerivedFrom = new LinkedHashMap<>();

        for (LedgerEntry entry : entries) {
            String entryKey = "casehub:entry-" + entry.id;
            String activityKey = "aml:activity-" + entry.id;
            String agentKey = "aml:agent-" + entry.actorId;

            entities.put(entryKey, buildEntity(entry, proofs.get(entry.id)));
            activities.put(activityKey, buildActivity(entry));
            agents.putIfAbsent(agentKey, buildAgent(entry));

            wasGeneratedBy.put("_:wgb-" + entry.id, Map.of(
                "prov:entity", entryKey, "prov:activity", activityKey));
            wasAssociatedWith.put("_:waw-" + entry.id, Map.of(
                "prov:activity", activityKey, "prov:agent", agentKey));
            wasAttributedTo.put("_:wat-" + entry.id, Map.of(
                "prov:entity", entryKey, "prov:agent", agentKey));

            if (entry.causedByEntryId != null) {
                wasDerivedFrom.put("_:wdf-" + entry.id, Map.of(
                    "prov:generatedEntity", entryKey,
                    "prov:usedEntity", "casehub:entry-" + entry.causedByEntryId));
            }
        }

        return new ProvDocument(PREFIXES, entities, activities, agents,
            wasGeneratedBy, wasAssociatedWith, wasAttributedTo, wasDerivedFrom);
    }

    private Map<String, Object> buildEntity(LedgerEntry entry, InclusionProof proof) {
        Map<String, Object> attrs = new LinkedHashMap<>();
        attrs.put("prov:type", entityType(entry));
        attrs.put("prov:generatedAtTime", entry.occurredAt.toString());
        attrs.put("casehub:sequenceNumber", entry.sequenceNumber);
        if (entry.digest != null) {
            attrs.put("casehub:digest", entry.digest);
        }
        if (proof != null) {
            attrs.put("casehub:merkleProofEntryIndex", proof.entryIndex());
            attrs.put("casehub:merkleProofTreeSize", proof.treeSize());
            attrs.put("casehub:merkleProofLeafHash", proof.leafHash());
            attrs.put("casehub:merkleProofTreeRoot", proof.treeRoot());
            attrs.put("casehub:merkleProofSiblings", proof.siblings().stream()
                .map(s -> Map.of("hash", s.hash(), "side", s.side().name()))
                .toList());
        }
        return attrs;
    }

    private Map<String, Object> buildActivity(LedgerEntry entry) {
        Map<String, Object> attrs = new LinkedHashMap<>();
        attrs.put("prov:type", activityType(entry));
        attrs.put("prov:startTime", entry.occurredAt.toString());
        addDomainAttributes(entry, attrs);
        return attrs;
    }

    private Map<String, Object> buildAgent(LedgerEntry entry) {
        Map<String, Object> attrs = new LinkedHashMap<>();
        attrs.put("prov:type", switch (entry.actorType) {
            case HUMAN -> "prov:Person";
            case SYSTEM, AGENT -> "prov:SoftwareAgent";
        });
        attrs.put("casehub:actorType", entry.actorType.name());
        if (entry.actorRole != null) {
            attrs.put("casehub:actorRole", entry.actorRole);
        }
        return attrs;
    }

    private String entityType(LedgerEntry entry) {
        return switch (entry) {
            case io.casehub.aml.ledger.AmlCaseOpenedLedgerEntry e -> "aml:CaseOpenedRecord";
            case io.casehub.aml.ledger.AmlComplianceReviewLedgerEntry e -> "aml:ComplianceReviewRecord";
            case io.casehub.aml.ledger.AmlSarOfficerReviewedLedgerEntry e -> "aml:SarOfficerReviewRecord";
            case io.casehub.aml.ledger.AmlSupervisorDecisionLedgerEntry e -> "aml:SupervisorDecisionRecord";
            case io.casehub.aml.ledger.AmlCaseProfileLedgerEntry e -> "aml:CaseProfileRecord";
            case io.casehub.aml.ledger.AmlCbrAdvisoryLedgerEntry e -> "aml:CbrAdvisoryRecord";
            case io.casehub.aml.trust.AmlTrustRoutingAttestation e -> "aml:TrustAttestationRecord";
            case io.casehub.ledger.model.CaseLedgerEntry e -> "aml:CaseLedgerRecord";
            case io.casehub.ledger.model.WorkerDecisionEntry e -> "aml:WorkerDecisionRecord";
            case io.casehub.qhorus.runtime.ledger.MessageLedgerEntry e -> "aml:MessageRecord";
            default -> "aml:LedgerRecord";
        };
    }

    private String activityType(LedgerEntry entry) {
        return switch (entry) {
            case io.casehub.aml.ledger.AmlCaseOpenedLedgerEntry e -> "aml:CaseOpening";
            case io.casehub.aml.ledger.AmlComplianceReviewLedgerEntry e -> "aml:ComplianceReviewOpening";
            case io.casehub.aml.ledger.AmlSarOfficerReviewedLedgerEntry e -> "aml:SarOfficerReview";
            case io.casehub.aml.ledger.AmlSupervisorDecisionLedgerEntry e -> "aml:SupervisorDecision";
            case io.casehub.aml.ledger.AmlCaseProfileLedgerEntry e -> "aml:CaseProfileCapture";
            case io.casehub.aml.ledger.AmlCbrAdvisoryLedgerEntry e -> "aml:CbrAdvisoryGeneration";
            case io.casehub.aml.trust.AmlTrustRoutingAttestation e -> "aml:TrustAttestation";
            case io.casehub.ledger.model.CaseLedgerEntry e -> "aml:CaseLifecycleEvent";
            case io.casehub.ledger.model.WorkerDecisionEntry e -> "aml:AgentRoutingDecision";
            case io.casehub.qhorus.runtime.ledger.MessageLedgerEntry e -> "aml:SpecialistCommunication";
            default -> "aml:LedgerEvent";
        };
    }

    private void addDomainAttributes(LedgerEntry entry, Map<String, Object> attrs) {
        switch (entry) {
            case io.casehub.aml.ledger.AmlCaseOpenedLedgerEntry e -> {
                attrs.put("aml:transactionId", e.transactionId);
                attrs.put("aml:originAccountId", e.originAccountId);
                attrs.put("aml:destinationAccountId", e.destinationAccountId);
            }
            case io.casehub.aml.ledger.AmlComplianceReviewLedgerEntry e ->
                attrs.put("aml:taskId", e.taskId);
            case io.casehub.aml.ledger.AmlSarOfficerReviewedLedgerEntry e -> {
                attrs.put("aml:reviewDecision", e.reviewDecision);
                if (e.rejectionReason != null) attrs.put("aml:rejectionReason", e.rejectionReason);
            }
            case io.casehub.aml.ledger.AmlSupervisorDecisionLedgerEntry e -> {
                attrs.put("aml:selectedBindings", e.selectedBindings);
                if (e.suppressedBindings != null) attrs.put("aml:suppressedBindings", e.suppressedBindings);
                attrs.put("aml:rationale", e.rationale);
                attrs.put("aml:earlyTermination", e.earlyTermination);
                attrs.put("aml:eligibleCount", e.eligibleCount);
                attrs.put("aml:degraded", e.degraded);
            }
            case io.casehub.aml.ledger.AmlCaseProfileLedgerEntry e -> {
                attrs.put("aml:flagReason", e.flagReason);
                attrs.put("aml:transactionAmount", e.transactionAmount);
                attrs.put("aml:priorIncidentCount", e.priorIncidentCount);
                if (e.entityType != null) attrs.put("aml:entityType", e.entityType);
                if (e.jurisdictionRisk != null) attrs.put("aml:jurisdictionRisk", e.jurisdictionRisk);
                if (e.networkComplexity != null) attrs.put("aml:networkComplexity", e.networkComplexity);
                attrs.put("aml:outcome", e.outcome);
                if (e.confidence != null) attrs.put("aml:confidence", e.confidence);
                attrs.put("aml:investigationPath", e.investigationPath);
                if (e.narrativeSeeded != null) attrs.put("aml:narrativeSeeded", e.narrativeSeeded);
                if (e.seedCount != null) attrs.put("aml:seedCount", e.seedCount);
                if (e.adaptationMethod != null) attrs.put("aml:adaptationMethod", e.adaptationMethod);
            }
            case io.casehub.aml.ledger.AmlCbrAdvisoryLedgerEntry e -> {
                attrs.put("aml:caseCount", e.caseCount);
                attrs.put("aml:avgSimilarity", e.avgSimilarity);
                attrs.put("aml:confidence", e.confidence);
                if (e.predominantOutcome != null) attrs.put("aml:predominantOutcome", e.predominantOutcome);
                if (e.predominantOutcomeFrequency != null) attrs.put("aml:predominantOutcomeFrequency", e.predominantOutcomeFrequency);
                if (e.recommendedCapabilities != null) attrs.put("aml:recommendedCapabilities", e.recommendedCapabilities);
                attrs.put("aml:active", e.active);
            }
            case io.casehub.aml.trust.AmlTrustRoutingAttestation e -> {
                attrs.put("aml:capabilityTag", e.capabilityTag);
                attrs.put("aml:selectedWorkerId", e.selectedWorkerId);
                if (e.trustScoreAtRouting != null) attrs.put("aml:trustScoreAtRouting", e.trustScoreAtRouting);
                attrs.put("aml:thresholdApplied", e.thresholdApplied);
                attrs.put("aml:investigationCaseId", e.investigationCaseId.toString());
                attrs.put("aml:reconstructed", e.reconstructed);
                attrs.put("aml:observerFailed", e.observerFailed);
            }
            case io.casehub.ledger.model.WorkerDecisionEntry e -> {
                if (e.capabilityTag != null) attrs.put("aml:capabilityTag", e.capabilityTag);
                if (e.selectedWorkerId != null) attrs.put("aml:selectedWorkerId", e.selectedWorkerId);
                if (e.routingRationale != null) attrs.put("aml:routingRationale", e.routingRationale);
            }
            default -> {}
        }
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn test -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am -Dtest=ProvDmMapperTest#emptyEntryList_producesDocumentWithPrefixesOnly -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 6: Write failing test — single AmlCaseOpenedLedgerEntry maps correctly**

```java
@Test
void singleCaseOpenedEntry_mapsEntityActivityAgent() {
    AmlCaseOpenedLedgerEntry entry = new AmlCaseOpenedLedgerEntry();
    entry.id = UUID.randomUUID();
    entry.subjectId = UUID.randomUUID();
    entry.sequenceNumber = 1;
    entry.actorId = "aml-orchestrator";
    entry.actorType = ActorType.SYSTEM;
    entry.actorRole = "AmlInvestigationOrchestrator";
    entry.occurredAt = Instant.parse("2026-08-15T10:00:00Z");
    entry.transactionId = "TXN-2024-001";
    entry.originAccountId = "ACC-A";
    entry.destinationAccountId = "ACC-B";
    entry.digest = "sha256:abc123";

    ProvDocument doc = mapper.map(List.of(entry), Map.of());

    // Entity
    String entryKey = "casehub:entry-" + entry.id;
    assertTrue(doc.entity().containsKey(entryKey));
    assertEquals("aml:CaseOpenedRecord", doc.entity().get(entryKey).get("prov:type"));
    assertEquals("2026-08-15T10:00:00Z", doc.entity().get(entryKey).get("prov:generatedAtTime"));
    assertEquals("sha256:abc123", doc.entity().get(entryKey).get("casehub:digest"));

    // Activity
    String activityKey = "aml:activity-" + entry.id;
    assertTrue(doc.activity().containsKey(activityKey));
    assertEquals("aml:CaseOpening", doc.activity().get(activityKey).get("prov:type"));
    assertEquals("TXN-2024-001", doc.activity().get(activityKey).get("aml:transactionId"));
    assertEquals("ACC-A", doc.activity().get(activityKey).get("aml:originAccountId"));

    // Agent
    String agentKey = "aml:agent-aml-orchestrator";
    assertTrue(doc.agent().containsKey(agentKey));
    assertEquals("prov:SoftwareAgent", doc.agent().get(agentKey).get("prov:type"));
    assertEquals("SYSTEM", doc.agent().get(agentKey).get("casehub:actorType"));

    // Relations
    assertEquals(1, doc.wasGeneratedBy().size());
    assertEquals(1, doc.wasAssociatedWith().size());
    assertEquals(1, doc.wasAttributedTo().size());
    assertTrue(doc.wasDerivedFrom().isEmpty(), "no causedByEntryId → no wasDerivedFrom");
}
```

Add imports at top of test class:
```java
import io.casehub.aml.ledger.AmlCaseOpenedLedgerEntry;
import io.casehub.platform.api.identity.ActorType;
import java.time.Instant;
import java.util.UUID;
```

- [ ] **Step 7: Run test to verify it passes** (mapper already written)

Run: `mvn test -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am -Dtest=ProvDmMapperTest#singleCaseOpenedEntry_mapsEntityActivityAgent -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 8: Write failing test — causedByEntryId chain produces wasDerivedFrom**

```java
@Test
void causalChain_producesWasDerivedFromEdges() {
    AmlCaseOpenedLedgerEntry opened = new AmlCaseOpenedLedgerEntry();
    opened.id = UUID.randomUUID();
    opened.subjectId = UUID.randomUUID();
    opened.sequenceNumber = 1;
    opened.actorId = "aml-orchestrator";
    opened.actorType = ActorType.SYSTEM;
    opened.actorRole = "AmlInvestigationOrchestrator";
    opened.occurredAt = Instant.now();
    opened.transactionId = "TXN-001";
    opened.originAccountId = "A";
    opened.destinationAccountId = "B";

    AmlComplianceReviewLedgerEntry review = new AmlComplianceReviewLedgerEntry();
    review.id = UUID.randomUUID();
    review.subjectId = opened.subjectId;
    review.sequenceNumber = 2;
    review.actorId = "aml-orchestrator";
    review.actorType = ActorType.SYSTEM;
    review.actorRole = "AmlInvestigationOrchestrator";
    review.occurredAt = Instant.now();
    review.taskId = UUID.randomUUID().toString();
    review.causedByEntryId = opened.id;

    ProvDocument doc = mapper.map(List.of(opened, review), Map.of());

    assertEquals(1, doc.wasDerivedFrom().size());
    Map<String, Object> edge = doc.wasDerivedFrom().values().iterator().next();
    assertEquals("casehub:entry-" + review.id, edge.get("prov:generatedEntity"));
    assertEquals("casehub:entry-" + opened.id, edge.get("prov:usedEntity"));
}
```

Add import: `import io.casehub.aml.ledger.AmlComplianceReviewLedgerEntry;`

- [ ] **Step 9: Run test to verify it passes**

Run: `mvn test -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am -Dtest=ProvDmMapperTest#causalChain_producesWasDerivedFromEdges -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 10: Write failing test — agent deduplication**

```java
@Test
void sameActorId_producesOneAgentNode() {
    AmlCaseOpenedLedgerEntry e1 = new AmlCaseOpenedLedgerEntry();
    e1.id = UUID.randomUUID();
    e1.subjectId = UUID.randomUUID();
    e1.sequenceNumber = 1;
    e1.actorId = "aml-orchestrator";
    e1.actorType = ActorType.SYSTEM;
    e1.actorRole = "AmlInvestigationOrchestrator";
    e1.occurredAt = Instant.now();
    e1.transactionId = "T1";
    e1.originAccountId = "A";
    e1.destinationAccountId = "B";

    AmlComplianceReviewLedgerEntry e2 = new AmlComplianceReviewLedgerEntry();
    e2.id = UUID.randomUUID();
    e2.subjectId = e1.subjectId;
    e2.sequenceNumber = 2;
    e2.actorId = "aml-orchestrator"; // same actor
    e2.actorType = ActorType.SYSTEM;
    e2.actorRole = "AmlInvestigationOrchestrator";
    e2.occurredAt = Instant.now();
    e2.taskId = "task-1";

    ProvDocument doc = mapper.map(List.of(e1, e2), Map.of());

    assertEquals(1, doc.agent().size(), "same actorId should produce one agent");
    assertEquals(2, doc.entity().size());
    assertEquals(2, doc.activity().size());
}
```

- [ ] **Step 11: Run test to verify it passes**

Run: `mvn test -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am -Dtest=ProvDmMapperTest#sameActorId_producesOneAgentNode -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 12: Write failing test — HUMAN actor maps to prov:Person**

```java
@Test
void humanActor_mapsToPerson() {
    AmlSarOfficerReviewedLedgerEntry entry = new AmlSarOfficerReviewedLedgerEntry();
    entry.id = UUID.randomUUID();
    entry.subjectId = UUID.randomUUID();
    entry.sequenceNumber = 1;
    entry.actorId = "officer-jane";
    entry.actorType = ActorType.HUMAN;
    entry.actorRole = "ComplianceOfficer";
    entry.occurredAt = Instant.now();
    entry.reviewDecision = "APPROVED";

    ProvDocument doc = mapper.map(List.of(entry), Map.of());

    Map<String, Object> agent = doc.agent().get("aml:agent-officer-jane");
    assertEquals("prov:Person", agent.get("prov:type"));
    assertEquals("HUMAN", agent.get("casehub:actorType"));
    assertEquals("ComplianceOfficer", agent.get("casehub:actorRole"));

    // Activity domain attributes
    Map<String, Object> activity = doc.activity().values().iterator().next();
    assertEquals("APPROVED", activity.get("aml:reviewDecision"));
    assertNull(activity.get("aml:rejectionReason"), "null rejectionReason should be omitted");
}
```

Add import: `import io.casehub.aml.ledger.AmlSarOfficerReviewedLedgerEntry;`

- [ ] **Step 13: Run test to verify it passes**

Run: `mvn test -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am -Dtest=ProvDmMapperTest#humanActor_mapsToPerson -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 14: Write failing test — unknown entry type falls back to generic**

```java
@Test
void unknownEntryType_fallsBackToGeneric() {
    // PlainLedgerEntry is a foundation type not in the switch — triggers default
    io.casehub.ledger.runtime.model.PlainLedgerEntry entry =
        new io.casehub.ledger.runtime.model.PlainLedgerEntry();
    entry.id = UUID.randomUUID();
    entry.subjectId = UUID.randomUUID();
    entry.sequenceNumber = 1;
    entry.actorId = "unknown-system";
    entry.actorType = ActorType.SYSTEM;
    entry.occurredAt = Instant.now();

    ProvDocument doc = mapper.map(List.of(entry), Map.of());

    Map<String, Object> entity = doc.entity().values().iterator().next();
    assertEquals("aml:LedgerRecord", entity.get("prov:type"));
    Map<String, Object> activity = doc.activity().values().iterator().next();
    assertEquals("aml:LedgerEvent", activity.get("prov:type"));
}
```

- [ ] **Step 15: Run test to verify it passes**

Run: `mvn test -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am -Dtest=ProvDmMapperTest#unknownEntryType_fallsBackToGeneric -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 16: Run all ProvDmMapperTest tests together**

Run: `mvn test -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am -Dtest=ProvDmMapperTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: 5 tests PASS

- [ ] **Step 17: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/provenance/ app/src/test/java/io/casehub/aml/provenance/
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#126): ProvDocument model and ProvDmMapper with unit tests

Refs casehubio/aml#126"
```

---

### Task 2: AmlProvenanceService, AmlProvenanceResource, and integration test

**Files:**
- Create: `app/src/main/java/io/casehub/aml/provenance/AmlProvenanceService.java`
- Create: `app/src/main/java/io/casehub/aml/provenance/AmlProvenanceResource.java`
- Create: `app/src/test/java/io/casehub/aml/provenance/AmlProvenanceResourceTest.java`

**Interfaces:**
- Consumes: `ProvDocument` and `ProvDmMapper` from Task 1, `LedgerEntryRepository.findBySubjectId()`, `LedgerVerificationService.inclusionProof()`
- Produces: REST endpoint `GET /api/investigations/{caseId}/provenance` returning PROV-JSON

- [ ] **Step 1: Write AmlProvenanceService**

```java
package io.casehub.aml.provenance;

import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.LedgerVerificationService;
import io.casehub.ledger.runtime.service.model.InclusionProof;
import io.casehub.platform.api.identity.TenancyConstants;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;

@ApplicationScoped
public class AmlProvenanceService {

    @Inject
    LedgerEntryRepository ledgerRepo;

    @Inject
    LedgerVerificationService verificationService;

    private final ProvDmMapper mapper = new ProvDmMapper();

    public Optional<ProvDocument> buildProvenance(UUID caseId) {
        List<LedgerEntry> entries = ledgerRepo.findBySubjectId(
            caseId, TenancyConstants.DEFAULT_TENANT_ID);
        if (entries.isEmpty()) {
            return Optional.empty();
        }
        Map<UUID, InclusionProof> proofs = new LinkedHashMap<>();
        for (LedgerEntry entry : entries) {
            try {
                InclusionProof proof = verificationService.inclusionProof(
                    entry.id, TenancyConstants.DEFAULT_TENANT_ID);
                proofs.put(entry.id, proof);
            } catch (Exception ignored) {
                // hash chain disabled or proof unavailable — omit
            }
        }
        return Optional.of(mapper.map(entries, proofs));
    }
}
```

- [ ] **Step 2: Write AmlProvenanceResource**

```java
package io.casehub.aml.provenance;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.util.UUID;

@ApplicationScoped
@Path("/api/investigations/{caseId}/provenance")
@Produces(MediaType.APPLICATION_JSON)
public class AmlProvenanceResource {

    @Inject
    AmlProvenanceService provenanceService;

    @GET
    public Response getProvenance(@PathParam("caseId") UUID caseId) {
        return provenanceService.buildProvenance(caseId)
            .map(doc -> Response.ok(doc).build())
            .orElse(Response.status(Response.Status.NOT_FOUND).build());
    }
}
```

- [ ] **Step 3: Write failing integration test — provenance for completed investigation**

```java
package io.casehub.aml.provenance;

import io.casehub.aml.domain.FlagReason;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.platform.api.identity.TenancyConstants;
import io.casehub.platform.api.path.Path;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import jakarta.inject.Inject;
import org.awaitility.Awaitility;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class AmlProvenanceResourceTest {

    @Inject CbrCaseMemoryStore cbrStore;

    @BeforeEach
    void clearCbrStore() {
        cbrStore.eraseByScope(Path.root(), TenancyConstants.DEFAULT_TENANT_ID);
    }

    private static final SuspiciousTransaction TX = new SuspiciousTransaction(
        "TXN-PROV-" + UUID.randomUUID(),
        "ACC-PROV-A", "ACC-PROV-B",
        new BigDecimal("50000"), "USD",
        Instant.parse("2024-12-01T00:00:00Z"),
        FlagReason.LAYERING);

    @Test
    void provenance_forCompletedInvestigation_returnsProvJson() {
        // Start investigation via Layer 9
        String caseIdStr = given().contentType(ContentType.JSON).body(TX)
            .when().post("/api/layer9/investigations")
            .then().statusCode(202)
            .extract().path("caseId");

        // Drain to completed (LAYERING is Autonomous — no gate approval needed)
        Awaitility.await().atMost(60, TimeUnit.SECONDS).pollInterval(200, TimeUnit.MILLISECONDS)
            .until(() -> "completed".equals(
                given().when().get("/api/layer9/investigations/" + caseIdStr)
                    .then().extract().path("status")));

        // GET provenance
        given().when().get("/api/investigations/" + caseIdStr + "/provenance")
            .then().statusCode(200)
            .body("prefix.prov", equalTo("http://www.w3.org/ns/prov#"))
            .body("prefix.casehub", equalTo("urn:casehub:ledger:"))
            .body("prefix.aml", equalTo("urn:casehub:aml:"))
            .body("entity.size()", greaterThan(0))
            .body("activity.size()", greaterThan(0))
            .body("agent.size()", greaterThan(0))
            .body("wasGeneratedBy.size()", greaterThan(0))
            .body("wasAttributedTo.size()", greaterThan(0));
    }

    @Test
    void provenance_forUnknownCaseId_returns404() {
        given().when().get("/api/investigations/" + UUID.randomUUID() + "/provenance")
            .then().statusCode(404);
    }
}
```

- [ ] **Step 4: Run integration test to verify it fails** (service/resource not yet on classpath, or compiles but endpoint wiring missing)

Run: `mvn test -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am -Dtest=AmlProvenanceResourceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL initially if classes don't compile together, then iterate

- [ ] **Step 5: Run integration test to verify it passes**

Run: `mvn test -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am -Dtest=AmlProvenanceResourceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: 2 tests PASS

- [ ] **Step 6: Run full test suite to check for regressions**

Run: `mvn test -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS (322+ existing tests + 7 new tests)

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/provenance/ app/src/test/java/io/casehub/aml/provenance/
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#126): AmlProvenanceService and REST endpoint with integration test

Refs casehubio/aml#126"
```

---

## References

- [2026-08-16-w3c-prov-dm-export-design.md] — design spec this plan implements
- [app/src/main/java/io/casehub/aml/ledger/AmlLedgerService.java] — causedByEntryId chain construction
- [app/src/main/java/io/casehub/aml/compliance/AmlComplianceEvidenceService.java] — existing pattern for querying entries and inclusion proofs
- [app/src/main/java/io/casehub/aml/compliance/AmlLayer7Resource.java] — existing JAX-RS endpoint pattern
- [app/src/test/java/io/casehub/aml/engine/AmlLayer9ResourceTest.java] — existing integration test pattern
- [GitHub #126] — focal issue
- [GitHub #7] — parent epic (GDPR and regulatory audit)
