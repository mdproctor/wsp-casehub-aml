# Art.22 Decision Record Compliance Supplements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #82 — feat: GDPR Art.22 decision record compliance supplements
**Issue group:** #7, #82, #83, #84, #126

**Goal:** Attach GDPR Art.22 compliance supplements to AML investigation triage entries, sanitise PII in decision context, and surface Art.22 compliance status in the ComplianceEvidence report.

**Architecture:** `AmlCaseProfileStoreObserver.doRetain()` attaches a `ComplianceSupplement` to the `AmlCaseProfileLedgerEntry` before `repository.save()` — same transaction, single write, Merkle chain integrity preserved. The supplement is built by `AmlComplianceSupplement.triageDecision()` (factory class in `io.casehub.aml.compliance`). Decision context PII is redacted by `AmlContentSanitiser` (plain class, not CDI — used locally, not as a platform-wide replacement). `AmlComplianceEvidenceService` gains a new `buildArt22DecisionRecord()` method that queries supplements on profile entries and computes CLOSED/PARTIAL/GAP status.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-ledger (ComplianceSupplement, ContentSanitiser), JUnit 5 + Mockito

## Global Constraints

- All AML ledger entries use `TenancyConstants.DEFAULT_TENANT_ID`
- `LedgerEntry.attach()` must be called before `repository.save()` — `supplementJson` is part of `canonicalBytes()` (Merkle chain integrity)
- Use `JpaComplianceSupplement` (runtime), not `ComplianceSupplement` (api) for instantiation
- `casehub.ledger.hash-chain.enabled=false` in test properties
- Tests must drain investigations to terminal status before asserting
- `AmlContentSanitiser` is a plain class (not CDI) — used locally in the observer, not as a global platform replacement

---

### Task 1: Art22DecisionRecordRequirement and Art22DecisionRecord (api/ module)

**Files:**
- Create: `api/src/main/java/io/casehub/aml/compliance/Art22DecisionRecordRequirement.java`
- Create: `api/src/main/java/io/casehub/aml/compliance/Art22DecisionRecord.java`
- Create: `api/src/test/java/io/casehub/aml/compliance/Art22DecisionRecordRequirementTest.java`
- Create: `api/src/test/java/io/casehub/aml/compliance/Art22DecisionRecordTest.java`

**Interfaces:**
- Consumes: `io.casehub.blocks.routing.RequirementStatus` (existing)
- Produces: `Art22DecisionRecordRequirement(String id, String citation, String mechanism, RequirementStatus status, List<Art22DecisionRecord> decisions)`, `Art22DecisionRecord(UUID entryId, String algorithmRef, Double confidenceScore, String rationale, boolean humanOverrideAvailable, String contestationUri, boolean decisionContextPresent)`

- [ ] **Step 1: Write failing tests for Art22DecisionRecord**

```java
package io.casehub.aml.compliance;

import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class Art22DecisionRecordTest {

    @Test
    void construction_allFields() {
        UUID entryId = UUID.randomUUID();
        var record = new Art22DecisionRecord(
                entryId, "AlgorithmRef", 0.85, "rationale",
                true, "/api/contestation", true);
        assertEquals(entryId, record.entryId());
        assertEquals("AlgorithmRef", record.algorithmRef());
        assertEquals(0.85, record.confidenceScore());
        assertEquals("rationale", record.rationale());
        assertTrue(record.humanOverrideAvailable());
        assertEquals("/api/contestation", record.contestationUri());
        assertTrue(record.decisionContextPresent());
    }

    @Test
    void construction_nullableFields() {
        var record = new Art22DecisionRecord(
                UUID.randomUUID(), null, null, null,
                false, null, false);
        assertNull(record.algorithmRef());
        assertNull(record.confidenceScore());
        assertFalse(record.humanOverrideAvailable());
        assertFalse(record.decisionContextPresent());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -am -Dtest=Art22DecisionRecordTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Implement Art22DecisionRecord**

Use `ide_create_file` for `api/src/main/java/io/casehub/aml/compliance/Art22DecisionRecord.java`:

```java
package io.casehub.aml.compliance;

import java.util.UUID;

public record Art22DecisionRecord(
        UUID entryId,
        String algorithmRef,
        Double confidenceScore,
        String rationale,
        boolean humanOverrideAvailable,
        String contestationUri,
        boolean decisionContextPresent) {}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl api -am -Dtest=Art22DecisionRecordTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Write failing tests for Art22DecisionRecordRequirement**

```java
package io.casehub.aml.compliance;

import io.casehub.blocks.routing.RequirementStatus;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class Art22DecisionRecordRequirementTest {

    @Test
    void constants() {
        assertEquals("GDPR-ART22-DECISION-RECORD", Art22DecisionRecordRequirement.REQUIREMENT_ID);
        assertNotNull(Art22DecisionRecordRequirement.CITATION);
        assertNotNull(Art22DecisionRecordRequirement.MECHANISM);
    }

    @Test
    void construction_closed() {
        var decision = new Art22DecisionRecord(
                UUID.randomUUID(), "alg", 0.9, "rationale", true, "/contest", true);
        var req = new Art22DecisionRecordRequirement(
                Art22DecisionRecordRequirement.REQUIREMENT_ID,
                Art22DecisionRecordRequirement.CITATION,
                Art22DecisionRecordRequirement.MECHANISM,
                RequirementStatus.CLOSED, List.of(decision));
        assertEquals(RequirementStatus.CLOSED, req.status());
        assertEquals(1, req.decisions().size());
    }

    @Test
    void construction_gap() {
        var req = new Art22DecisionRecordRequirement(
                Art22DecisionRecordRequirement.REQUIREMENT_ID,
                Art22DecisionRecordRequirement.CITATION,
                Art22DecisionRecordRequirement.MECHANISM,
                RequirementStatus.GAP, List.of());
        assertEquals(RequirementStatus.GAP, req.status());
        assertTrue(req.decisions().isEmpty());
    }
}
```

- [ ] **Step 6: Implement Art22DecisionRecordRequirement**

Use `ide_create_file` for `api/src/main/java/io/casehub/aml/compliance/Art22DecisionRecordRequirement.java`:

```java
package io.casehub.aml.compliance;

import io.casehub.blocks.routing.RequirementStatus;
import java.util.List;

public record Art22DecisionRecordRequirement(
        String id,
        String citation,
        String mechanism,
        RequirementStatus status,
        List<Art22DecisionRecord> decisions) {

    public static final String REQUIREMENT_ID = "GDPR-ART22-DECISION-RECORD";
    public static final String CITATION =
            "GDPR Art.22 — automated decision-making transparency and contestation rights";
    public static final String MECHANISM =
            "ComplianceSupplement attached to AmlCaseProfileLedgerEntry at case completion. " +
            "Records algorithm reference, confidence score, sanitised decision context, " +
            "and contestation URI. Human override available via compliance officer review gate.";
}
```

- [ ] **Step 7: Run all Task 1 tests**

Run: `mvn test -pl api -am -Dtest="Art22DecisionRecord*Test" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/aml/compliance/Art22DecisionRecord.java api/src/main/java/io/casehub/aml/compliance/Art22DecisionRecordRequirement.java api/src/test/java/io/casehub/aml/compliance/Art22DecisionRecordTest.java api/src/test/java/io/casehub/aml/compliance/Art22DecisionRecordRequirementTest.java
git commit -m "feat(#82): Art22DecisionRecordRequirement and Art22DecisionRecord types"
```

---

### Task 2: ComplianceEvidence record extension (api/ module)

**Files:**
- Modify: `api/src/main/java/io/casehub/aml/compliance/ComplianceEvidence.java`

**Interfaces:**
- Consumes: `Art22DecisionRecordRequirement` (Task 1)
- Produces: `ComplianceEvidence` with new `art22DecisionRecord` field (8th parameter, before `signature`)

**Depends on:** Task 1

- [ ] **Step 1: Modify ComplianceEvidence record**

Use `ide_replace_member` on `ComplianceEvidence` to add the `art22DecisionRecord` field:

```java
public record ComplianceEvidence(
    UUID caseId,
    Instant generatedAt,
    AuditChainRequirement auditChain,
    SlaRequirement sla,
    TrustRoutingRequirement trustRouting,
    GdprErasureRequirement gdprErasure,
    Art22DecisionRecordRequirement art22DecisionRecord,
    String signature
) {}
```

- [ ] **Step 2: Fix compilation errors from callers**

Use `ide_find_references` on `ComplianceEvidence` to find all construction sites. Update each to pass `null` for `art22DecisionRecord` (placeholder until Task 5 wires it).

The main caller is `AmlComplianceEvidenceService.build()` — update to pass `null`:

```java
return new ComplianceEvidence(
        caseId, Instant.now(),
        buildAuditChain(caseId, caseEntries, reviewEntries, officerReviewEntries),
        buildSla(reviewEntries),
        buildTrustRouting(caseId),
        buildGdprErasure(),
        null,
        null);
```

Also update any test code that constructs `ComplianceEvidence` directly.

- [ ] **Step 3: Run existing tests to verify no regressions**

Run: `mvn test -pl api,app -am -Dtest="AmlComplianceEvidence*,AmlLayer7Resource*" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add api/src/main/java/io/casehub/aml/compliance/ComplianceEvidence.java app/src/main/java/io/casehub/aml/compliance/AmlComplianceEvidenceService.java
git commit -m "feat(#82): add art22DecisionRecord field to ComplianceEvidence"
```

---

### Task 3: AmlComplianceSupplement factory + AmlContentSanitiser (app/ module)

**Files:**
- Create: `app/src/main/java/io/casehub/aml/compliance/AmlComplianceSupplement.java`
- Create: `app/src/main/java/io/casehub/aml/compliance/AmlContentSanitiser.java` (plain class, not CDI)
- Create: `app/src/test/java/io/casehub/aml/compliance/AmlComplianceSupplementTest.java`
- Create: `app/src/test/java/io/casehub/aml/compliance/AmlContentSanitiserTest.java`

**Interfaces:**
- Consumes: `io.casehub.ledger.api.model.supplement.ComplianceSupplement`, `io.casehub.ledger.runtime.model.supplement.JpaComplianceSupplement`
- Produces: `AmlComplianceSupplement.triageDecision(String outcome, Double confidence, String investigationPath, String sanitisedDecisionContext)` → `ComplianceSupplement`, `AmlContentSanitiser.sanitise(String)` → `String` (plain class, no CDI)

- [ ] **Step 1: Write failing tests for AmlContentSanitiser**

```java
package io.casehub.aml.compliance;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class AmlContentSanitiserTest {

    private final AmlContentSanitiser sanitiser = new AmlContentSanitiser();

    @Test
    void sanitise_redactsIban() {
        String input = "account: GB29NWBK60161331926819";
        String result = sanitiser.sanitise(input);
        assertEquals("account: [REDACTED:account]", result);
    }

    @Test
    void sanitise_preservesNumericStrings() {
        String input = "{\"priorIncidentCount\":12345678901234}";
        String result = sanitiser.sanitise(input);
        assertEquals(input, result);
    }

    @Test
    void sanitise_preservesShortNumbers() {
        String input = "{\"count\": 3}";
        assertEquals(input, sanitiser.sanitise(input));
    }

    @Test
    void sanitise_nullReturnsNull() {
        assertNull(sanitiser.sanitise(null));
    }

    @Test
    void sanitise_noPiiUnchanged() {
        String input = "{\"flagReason\":\"HIGH_RISK_JURISDICTION\"}";
        assertEquals(input, sanitiser.sanitise(input));
    }

    @Test
    void sanitise_multipleIbans() {
        String input = "from GB29NWBK60161331926819 to DE89370400440532013000";
        String result = sanitiser.sanitise(input);
        assertEquals("from [REDACTED:account] to [REDACTED:account]", result);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl app -am -Dtest=AmlContentSanitiserTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Implement AmlContentSanitiser**

First, check `PassThroughContentSanitiser` CDI registration:

```bash
# Use ide_find_references on PassThroughContentSanitiser to see how it's registered
```

Use `ide_create_file` for `app/src/main/java/io/casehub/aml/compliance/AmlContentSanitiser.java`:

```java
package io.casehub.aml.compliance;

import java.util.regex.Pattern;

public final class AmlContentSanitiser {

    private static final Pattern IBAN_PATTERN =
            Pattern.compile("\\b[A-Z]{2}\\d{2}[A-Z0-9]{4,30}\\b");

    public String sanitise(String decisionContextJson) {
        if (decisionContextJson == null) return null;
        return IBAN_PATTERN.matcher(decisionContextJson)
                .replaceAll("[REDACTED:account]");
    }
}
```

Plain class — no CDI. Instantiated locally in the observer. Does not displace the platform's `PassThroughContentSanitiser` for other domains.

- [ ] **Step 4: Run sanitiser tests**

Run: `mvn test -pl app -am -Dtest=AmlContentSanitiserTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Write failing tests for AmlComplianceSupplement**

```java
package io.casehub.aml.compliance;

import io.casehub.ledger.api.model.supplement.ComplianceSupplement;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class AmlComplianceSupplementTest {

    @Test
    void triageDecision_allFieldsPopulated() {
        ComplianceSupplement s = AmlComplianceSupplement.triageDecision(
                "SAR_WARRANTED", 0.85, "entity-resolution→pattern-analysis→sar-drafting",
                "{\"flagReason\":\"HIGH_RISK_JURISDICTION\"}");
        assertEquals("GDPR Art.22(1) — automated decision-making transparency", s.planRef);
        assertEquals("AmlInvestigationTriageService (CBR-weighted rule-based triage)", s.algorithmRef);
        assertEquals(0.85, s.confidenceScore);
        assertNotNull(s.rationale);
        assertTrue(s.rationale.contains("SAR_WARRANTED"));
        assertTrue(s.rationale.contains("entity-resolution"));
        assertEquals("{\"flagReason\":\"HIGH_RISK_JURISDICTION\"}", s.decisionContext);
        assertTrue(s.humanOverrideAvailable);
        assertEquals("/api/investigations/{caseId}/contestation", s.contestationUri);
    }

    @Test
    void triageDecision_nullConfidence() {
        ComplianceSupplement s = AmlComplianceSupplement.triageDecision(
                "INVESTIGATION_CLEARED", null, "(direct-verdict)", null);
        assertNull(s.confidenceScore);
        assertNull(s.decisionContext);
        assertNotNull(s.algorithmRef);
        assertTrue(s.humanOverrideAvailable);
    }
}
```

- [ ] **Step 6: Implement AmlComplianceSupplement**

Use `ide_create_file` for `app/src/main/java/io/casehub/aml/compliance/AmlComplianceSupplement.java`:

```java
package io.casehub.aml.compliance;

import io.casehub.ledger.api.model.supplement.ComplianceSupplement;
import io.casehub.ledger.runtime.model.supplement.JpaComplianceSupplement;

public final class AmlComplianceSupplement {

    private AmlComplianceSupplement() {}

    public static ComplianceSupplement triageDecision(
            String outcome, Double confidence, String investigationPath,
            String sanitisedDecisionContext) {
        ComplianceSupplement s = new JpaComplianceSupplement();
        s.planRef = "GDPR Art.22(1) — automated decision-making transparency";
        s.algorithmRef = "AmlInvestigationTriageService (CBR-weighted rule-based triage)";
        s.confidenceScore = confidence;
        s.rationale = "Triage outcome: " + outcome + ". Path: " + investigationPath;
        s.decisionContext = sanitisedDecisionContext;
        s.humanOverrideAvailable = true;
        s.contestationUri = "/api/investigations/{caseId}/contestation";
        return s;
    }
}
```

- [ ] **Step 7: Run all Task 3 tests**

Run: `mvn test -pl app -am -Dtest="AmlComplianceSupplementTest,AmlContentSanitiserTest" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/aml/compliance/AmlComplianceSupplement.java app/src/main/java/io/casehub/aml/compliance/AmlContentSanitiser.java app/src/test/java/io/casehub/aml/compliance/AmlComplianceSupplementTest.java app/src/test/java/io/casehub/aml/compliance/AmlContentSanitiserTest.java
git commit -m "feat(#82): AmlComplianceSupplement factory and AmlContentSanitiser"
```

---

### Task 4: Attach Art.22 supplement in AmlCaseProfileStoreObserver (app/ module)

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/cbr/AmlCaseProfileStoreObserver.java`
- Modify: `app/src/test/java/io/casehub/aml/cbr/AmlCaseProfileStoreObserverTest.java`

**Interfaces:**
- Consumes: `AmlComplianceSupplement.triageDecision()` (Task 3), `AmlContentSanitiser.sanitise()` (Task 3), `ObjectMapper` (already injected)
- Produces: `AmlCaseProfileLedgerEntry` saved with `ComplianceSupplement` attached — single write, Merkle chain integrity preserved

**Depends on:** Task 3

- [ ] **Step 1: Write failing tests for supplement attachment**

Add to `AmlCaseProfileStoreObserverTest.java`:

```java
@Test
void onOutcome_attachesArt22SupplementToProfileEntry() {
    // Set up a CaseOutcomeEvent with valid triage snapshot
    // After observer fires, verify the saved AmlCaseProfileLedgerEntry
    // has a ComplianceSupplement attached via entry.compliance()
    // Use ArgumentCaptor on ledgerRepository.save() to capture the entry
    ArgumentCaptor<LedgerEntry> captor = ArgumentCaptor.forClass(LedgerEntry.class);
    // ... trigger observer with snapshot containing investigationTriage ...
    verify(ledgerRepository).save(captor.capture(), any());
    var saved = (AmlCaseProfileLedgerEntry) captor.getValue();
    assertTrue(saved.compliance().isPresent());
    assertEquals("GDPR Art.22(1) — automated decision-making transparency",
            saved.compliance().get().planRef);
    assertTrue(saved.compliance().get().humanOverrideAvailable);
}

@Test
void onOutcome_sanitisesDecisionContext() {
    // Verify decisionContext does not contain account IDs
    // even if the transaction map includes them
    // ... trigger observer with snapshot including originAccountId ...
    ArgumentCaptor<LedgerEntry> captor = ArgumentCaptor.forClass(LedgerEntry.class);
    verify(ledgerRepository).save(captor.capture(), any());
    var saved = (AmlCaseProfileLedgerEntry) captor.getValue();
    String context = saved.compliance().get().decisionContext;
    assertFalse(context.contains("originAccountId"));
    assertFalse(context.contains("destinationAccountId"));
}
```

Note: Adapt these to the existing test structure in `AmlCaseProfileStoreObserverTest`. The exact setup depends on how the test currently creates `CaseOutcomeEvent` instances.

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl app -am -Dtest=AmlCaseProfileStoreObserverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — no supplement attached

- [ ] **Step 3: Modify AmlCaseProfileStoreObserver.doRetain()**

Use `ide_edit_member` to add a field and modify `doRetain()`.

Add field:
```java
private static final AmlContentSanitiser ART22_SANITISER = new AmlContentSanitiser();
```

Inside the `QuarkusTransaction.requiringNew().run(() -> {...})` block, after all entry fields are set and before `ledgerRepository.save(entry, tenantId)`, add:

```java
// Art.22 compliance supplement — attached before save to preserve Merkle chain integrity
try {
    var decisionContextMap = new java.util.LinkedHashMap<String, Object>();
    decisionContextMap.put("decision", fOutcome);
    decisionContextMap.put("flagReason", tx.flagReason().name());
    decisionContextMap.put("amount", tx.amount().toPlainString());
    decisionContextMap.put("currency", tx.currency());
    if (fEntityType != null) decisionContextMap.put("entityType", fEntityType);
    if (fJurisdiction != null) decisionContextMap.put("jurisdictionRisk", fJurisdiction);
    if (fNetwork != null) decisionContextMap.put("networkComplexity", fNetwork);
    String contextJson = objectMapper.writeValueAsString(decisionContextMap);
    String sanitised = ART22_SANITISER.sanitise(contextJson);
    entry.attach(AmlComplianceSupplement.triageDecision(
            fOutcome, entry.confidence, fSolution, sanitised));
} catch (Exception e) {
    LOG.warnf(e, "Art.22 supplement creation failed for caseId=%s — entry saved without supplement", caseId);
}
ledgerRepository.save(entry, tenantId);
```

The supplement is built from the same parsed snapshot data already available in `doRetain()` — no duplicate parsing. The try/catch ensures a supplement failure doesn't prevent the profile entry from being saved.

- [ ] **Step 4: Run tests**

Run: `mvn test -pl app -am -Dtest=AmlCaseProfileStoreObserverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/aml/cbr/AmlCaseProfileStoreObserver.java app/src/test/java/io/casehub/aml/cbr/AmlCaseProfileStoreObserverTest.java
git commit -m "feat(#82): attach Art.22 compliance supplement in AmlCaseProfileStoreObserver"
```

---

### Task 5: Wire Art.22 into AmlComplianceEvidenceService (app/ module)

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/compliance/AmlComplianceEvidenceService.java`
- Modify: `app/src/test/java/io/casehub/aml/compliance/AmlComplianceEvidenceServiceTest.java`

**Interfaces:**
- Consumes: `Art22DecisionRecordRequirement` (Task 1), `Art22DecisionRecord` (Task 1), `AmlCaseProfileLedgerEntry`, `ComplianceSupplement`, `LedgerEntry.compliance()`
- Produces: `ComplianceEvidence.art22DecisionRecord()` populated with CLOSED/PARTIAL/GAP status

**Depends on:** Tasks 1, 2

- [ ] **Step 1: Write failing tests for Art.22 evidence assembly**

Add to `AmlComplianceEvidenceServiceTest.java`:

```java
@Test
void assembleEvidence_withArt22Supplement_statusClosed() {
    setupMinimalLedgerEntries();
    var profileEntry = profileEntryWithSupplement(caseId);
    when(ledgerRepo.findBySubjectId(eq(caseId), any()))
            .thenReturn(List.of(
                    caseOpenedEntry(caseId, caseOpenedId),
                    reviewOpenedEntry(caseId, reviewOpenedId, taskId, caseOpenedId),
                    profileEntry));
    when(verificationService.inclusionProof(any(), any())).thenReturn(stubProof());
    when(attestationRepo.findByInvestigationCaseId(caseId)).thenReturn(List.of());
    when(workerDecisionRepo.findAllByCaseId(caseId)).thenReturn(List.of());
    when(mockReconciler.reconcileIfNeeded(any(), any(), any())).thenReturn(List.of());

    ComplianceEvidence evidence = service.assembleEvidence(caseId);

    assertNotNull(evidence.art22DecisionRecord());
    assertEquals(RequirementStatus.CLOSED, evidence.art22DecisionRecord().status());
    assertEquals(1, evidence.art22DecisionRecord().decisions().size());
    assertTrue(evidence.art22DecisionRecord().decisions().get(0).humanOverrideAvailable());
}

@Test
void assembleEvidence_noProfileEntry_art22Gap() {
    setupMinimalLedgerEntries();
    when(verificationService.inclusionProof(any(), any())).thenReturn(stubProof());
    when(attestationRepo.findByInvestigationCaseId(caseId)).thenReturn(List.of());
    when(workerDecisionRepo.findAllByCaseId(caseId)).thenReturn(List.of());
    when(mockReconciler.reconcileIfNeeded(any(), any(), any())).thenReturn(List.of());

    ComplianceEvidence evidence = service.assembleEvidence(caseId);

    assertNotNull(evidence.art22DecisionRecord());
    assertEquals(RequirementStatus.GAP, evidence.art22DecisionRecord().status());
}

@Test
void assembleEvidence_profileWithoutSupplement_art22Gap() {
    setupMinimalLedgerEntries();
    var profileEntry = new AmlCaseProfileLedgerEntry();
    profileEntry.subjectId = caseId;
    profileEntry.id = UUID.randomUUID();
    profileEntry.outcome = "SAR_WARRANTED";
    when(ledgerRepo.findBySubjectId(eq(caseId), any()))
            .thenReturn(List.of(
                    caseOpenedEntry(caseId, caseOpenedId),
                    reviewOpenedEntry(caseId, reviewOpenedId, taskId, caseOpenedId),
                    profileEntry));
    when(verificationService.inclusionProof(any(), any())).thenReturn(stubProof());
    when(attestationRepo.findByInvestigationCaseId(caseId)).thenReturn(List.of());
    when(workerDecisionRepo.findAllByCaseId(caseId)).thenReturn(List.of());
    when(mockReconciler.reconcileIfNeeded(any(), any(), any())).thenReturn(List.of());

    ComplianceEvidence evidence = service.assembleEvidence(caseId);

    assertEquals(RequirementStatus.GAP, evidence.art22DecisionRecord().status());
}

private AmlCaseProfileLedgerEntry profileEntryWithSupplement(UUID caseId) {
    var entry = new AmlCaseProfileLedgerEntry();
    entry.id = UUID.randomUUID();
    entry.subjectId = caseId;
    entry.outcome = "SAR_WARRANTED";
    entry.flagReason = "HIGH_RISK_JURISDICTION";
    entry.transactionAmount = java.math.BigDecimal.valueOf(50000);
    entry.priorIncidentCount = 0;
    entry.investigationPath = "entity-resolution→sar-drafting";
    entry.attach(AmlComplianceSupplement.triageDecision(
            "SAR_WARRANTED", null, "entity-resolution→sar-drafting",
            "{\"flagReason\":\"HIGH_RISK_JURISDICTION\"}"));
    return entry;
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl app -am -Dtest=AmlComplianceEvidenceServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `art22DecisionRecord()` returns null

- [ ] **Step 3: Implement buildArt22DecisionRecord and wire into build()**

Use `ide_insert_member` on `AmlComplianceEvidenceService` to add `buildArt22DecisionRecord`:

```java
private Art22DecisionRecordRequirement buildArt22DecisionRecord(List<LedgerEntry> all) {
    List<AmlCaseProfileLedgerEntry> profileEntries = all.stream()
            .filter(AmlCaseProfileLedgerEntry.class::isInstance)
            .map(AmlCaseProfileLedgerEntry.class::cast)
            .toList();

    if (profileEntries.isEmpty()) {
        return new Art22DecisionRecordRequirement(
                Art22DecisionRecordRequirement.REQUIREMENT_ID,
                Art22DecisionRecordRequirement.CITATION,
                Art22DecisionRecordRequirement.MECHANISM,
                RequirementStatus.GAP, List.of());
    }

    List<Art22DecisionRecord> records = new ArrayList<>();
    boolean allComplete = true;

    for (AmlCaseProfileLedgerEntry entry : profileEntries) {
        Optional<ComplianceSupplement> supplement = entry.compliance();
        if (supplement.isPresent()) {
            ComplianceSupplement s = supplement.get();
            records.add(new Art22DecisionRecord(
                    entry.id, s.algorithmRef, s.confidenceScore,
                    s.rationale, Boolean.TRUE.equals(s.humanOverrideAvailable),
                    s.contestationUri, s.decisionContext != null));
            if (s.algorithmRef == null || s.humanOverrideAvailable == null) {
                allComplete = false;
            }
        } else {
            allComplete = false;
        }
    }

    RequirementStatus status;
    if (!records.isEmpty() && allComplete) {
        status = RequirementStatus.CLOSED;
    } else if (!records.isEmpty()) {
        status = RequirementStatus.PARTIAL;
    } else {
        status = RequirementStatus.GAP;
    }

    return new Art22DecisionRecordRequirement(
            Art22DecisionRecordRequirement.REQUIREMENT_ID,
            Art22DecisionRecordRequirement.CITATION,
            Art22DecisionRecordRequirement.MECHANISM,
            status, records);
}
```

Update `build()` to thread `all` through and pass to `buildArt22DecisionRecord`:

Use `ide_replace_member` on `build` method. The method signature gains `List<LedgerEntry> all` and the body passes it:

```java
private ComplianceEvidence build(UUID caseId,
        List<AmlCaseOpenedLedgerEntry> caseEntries,
        List<AmlComplianceReviewLedgerEntry> reviewEntries,
        List<AmlSarOfficerReviewedLedgerEntry> officerReviewEntries,
        List<LedgerEntry> all) {
    return new ComplianceEvidence(
            caseId, Instant.now(),
            buildAuditChain(caseId, caseEntries, reviewEntries, officerReviewEntries),
            buildSla(reviewEntries),
            buildTrustRouting(caseId),
            buildGdprErasure(),
            buildArt22DecisionRecord(all),
            null);
}
```

Update `findEvidence()` and `assembleEvidence()` to pass `all` to `build()`.

Add required imports: `AmlCaseProfileLedgerEntry`, `Art22DecisionRecordRequirement`, `Art22DecisionRecord`, `ComplianceSupplement` (supplement import).

- [ ] **Step 4: Run tests**

Run: `mvn test -pl app -am -Dtest=AmlComplianceEvidenceServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS (all existing + new tests)

- [ ] **Step 5: Run full test suite to check for regressions**

Run: `mvn test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/aml/compliance/AmlComplianceEvidenceService.java app/src/test/java/io/casehub/aml/compliance/AmlComplianceEvidenceServiceTest.java
git commit -m "feat(#82): wire Art.22 evidence section into AmlComplianceEvidenceService"
```

---

### Task 6: Integration test + TypeScript type update

**Files:**
- Modify: `app/src/test/java/io/casehub/aml/compliance/AmlLayer7ResourceTest.java`
- Modify: `app/src/main/webui/src/types.ts` (add `Art22DecisionRecordRequirement` interface)

**Interfaces:**
- Consumes: Full investigation flow, Layer 7 REST endpoint
- Produces: Verified end-to-end Art.22 compliance evidence in JSON response

**Depends on:** Tasks 1–4 (renamed from 1–5)

- [ ] **Step 1: Add integration test for Art.22 in Layer 7 response**

Add to `AmlLayer7ResourceTest.java`:

```java
@Test
void getComplianceEvidence_includesArt22Section() {
    // Use existing test infrastructure to start an investigation,
    // drain to completion, then GET compliance evidence
    // Verify JSON path: art22DecisionRecord.status exists
    // Verify art22DecisionRecord.decisions is an array
    given()
        .when().get("/api/investigations/" + completedCaseId + "/compliance-evidence")
        .then()
        .statusCode(200)
        .body("art22DecisionRecord.id", equalTo("GDPR-ART22-DECISION-RECORD"))
        .body("art22DecisionRecord.status", notNullValue())
        .body("art22DecisionRecord.decisions", notNullValue());
}
```

Note: The exact test setup depends on the existing `AmlLayer7ResourceTest` infrastructure. If it already has a completed investigation, add assertions to the existing happy-path test rather than a separate test method.

- [ ] **Step 2: Run integration test**

Run: `mvn test -pl app -am -Dtest=AmlLayer7ResourceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 3: Update TypeScript types**

Add to `app/src/main/webui/src/types.ts`:

```typescript
export interface Art22DecisionRecord {
  entryId: string;
  algorithmRef: string | null;
  confidenceScore: number | null;
  rationale: string | null;
  humanOverrideAvailable: boolean;
  contestationUri: string | null;
  decisionContextPresent: boolean;
}

export interface Art22DecisionRecordRequirement {
  id: string;
  citation: string;
  mechanism: string;
  status: RequirementStatus;
  decisions: Art22DecisionRecord[];
}
```

Update `ComplianceEvidence` interface to include `art22DecisionRecord`:

```typescript
export interface ComplianceEvidence {
  // ... existing fields ...
  art22DecisionRecord: Art22DecisionRecordRequirement;
  signature: string | null;
}
```

- [ ] **Step 4: Run full test suite**

Run: `mvn verify -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/test/java/io/casehub/aml/compliance/AmlLayer7ResourceTest.java app/src/main/webui/src/types.ts
git commit -m "feat(#82): integration test and TypeScript types for Art.22 compliance evidence"
```

---

## Self-Review Checklist

- **Spec coverage:** All 5 parts covered (factory ✓, sanitiser ✓, observer modification ✓, requirement types ✓, evidence extension ✓)
- **Placeholder scan:** No TBDs. All code blocks contain complete implementations. Task 4 tests are sketched (observer test structure varies) — implementer adapts to existing test infrastructure.
- **Type consistency:** `Art22DecisionRecordRequirement` and `Art22DecisionRecord` used consistently across Tasks 1, 2, 5, 6. `AmlComplianceSupplement.triageDecision()` signature matches between Task 3 (definition) and Task 4 (usage).
- **Merkle chain integrity:** Supplement attached before `repository.save()` in single transaction (Task 4). No re-save, no ordering race.
- **Tooling safety scan:** No bash file operations on source files. All code changes use `ide_create_file`, `ide_insert_member`, `ide_replace_member`, `ide_edit_member`.
