# InvestigationStatus + GDPR Erasure Receipt Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Expand `InvestigationStatus` to distinguish FAULTED/CANCELLED/SUSPENDED from IN_PROGRESS (#78), and enable tamper-evident GDPR erasure receipts (#62).

**Architecture:** Part 1 replaces the lossy `if != COMPLETED` check with an exhaustive switch expression mapping every `CaseStatus` to a domain-meaningful `InvestigationStatus`. Part 2 enables the foundation's `ErasureReceiptLedgerEntry` via config, wraps `LedgerErasureService` in an `AmlErasureService`, and upgrades compliance evidence from static booleans to live config/receipt queries.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA (JOINED inheritance), Flyway, JUnit 5, Mockito, REST Assured

## Global Constraints

- `api/` module has zero framework dependencies — no Jackson, no JPA, no Quarkus annotations.
- All ledger entry writes must guarantee non-null `tenancyId`. Use `TenancyConstants.DEFAULT_TENANT_ID`.
- Test commands: `mvn -pl app -am test -Dtest=ClassName -Dsurefire.failIfNoSpecifiedTests=false`
- `casehub.ledger.hash-chain.enabled=false` in test properties (H2 lacks row-level locking).
- No `default` arm in the `CaseStatus` switch — intentionally causes compilation failure on new enum values.
- Spec: `specs/issue-78-status-erasure/2026-06-29-status-erasure-design.md`

---

### Task 1: InvestigationStatus enum expansion (#78)

**Files:**
- Modify: `api/src/main/java/io/casehub/aml/domain/InvestigationStatus.java`
- Modify: `api/src/test/java/io/casehub/aml/domain/InvestigationStatusTest.java`
- Modify: `app/src/test/java/io/casehub/aml/engine/InvestigationStatusMixinTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `InvestigationStatus.FAILED`, `InvestigationStatus.CANCELLED`, `InvestigationStatus.SUSPENDED` (used by Task 2)

- [ ] **Step 1: Write failing unit tests for new enum values**

Add to `InvestigationStatusTest`:

```java
@Test
void failed_wire_format() {
    assertEquals("failed", InvestigationStatus.FAILED.toWireFormat());
}

@Test
void cancelled_wire_format() {
    assertEquals("cancelled", InvestigationStatus.CANCELLED.toWireFormat());
}

@Test
void suspended_wire_format() {
    assertEquals("suspended", InvestigationStatus.SUSPENDED.toWireFormat());
}

@Test
void from_wire_format_failed() {
    assertEquals(InvestigationStatus.FAILED, InvestigationStatus.fromWireFormat("failed"));
}

@Test
void from_wire_format_cancelled() {
    assertEquals(InvestigationStatus.CANCELLED, InvestigationStatus.fromWireFormat("cancelled"));
}

@Test
void from_wire_format_suspended() {
    assertEquals(InvestigationStatus.SUSPENDED, InvestigationStatus.fromWireFormat("suspended"));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl api -am test -Dtest=InvestigationStatusTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `FAILED`, `CANCELLED`, `SUSPENDED` don't exist

- [ ] **Step 3: Add enum values**

Replace `InvestigationStatus.java`:

```java
package io.casehub.aml.domain;

public enum InvestigationStatus {
    IN_PROGRESS,
    COMPLETED,
    FAILED,
    CANCELLED,
    SUSPENDED;

    public String toWireFormat() {
        return name().toLowerCase().replace('_', '-');
    }

    public static InvestigationStatus fromWireFormat(String value) {
        return valueOf(value.toUpperCase().replace('-', '_'));
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn -pl api -am test -Dtest=InvestigationStatusTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 9 tests PASS

- [ ] **Step 5: Add Jackson mixin tests for new values**

Add to `InvestigationStatusMixinTest`:

```java
@Test
void serializes_failed() throws Exception {
    assertEquals("\"failed\"", mapper.writeValueAsString(InvestigationStatus.FAILED));
}

@Test
void serializes_cancelled() throws Exception {
    assertEquals("\"cancelled\"", mapper.writeValueAsString(InvestigationStatus.CANCELLED));
}

@Test
void serializes_suspended() throws Exception {
    assertEquals("\"suspended\"", mapper.writeValueAsString(InvestigationStatus.SUSPENDED));
}

@Test
void deserializes_failed() throws Exception {
    assertEquals(InvestigationStatus.FAILED,
            mapper.readValue("\"failed\"", InvestigationStatus.class));
}

@Test
void deserializes_cancelled() throws Exception {
    assertEquals(InvestigationStatus.CANCELLED,
            mapper.readValue("\"cancelled\"", InvestigationStatus.class));
}

@Test
void deserializes_suspended() throws Exception {
    assertEquals(InvestigationStatus.SUSPENDED,
            mapper.readValue("\"suspended\"", InvestigationStatus.class));
}
```

- [ ] **Step 6: Run mixin tests**

Run: `mvn -pl app -am test -Dtest=InvestigationStatusMixinTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 10 tests PASS (4 existing + 6 new)

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/aml/domain/InvestigationStatus.java api/src/test/java/io/casehub/aml/domain/InvestigationStatusTest.java app/src/test/java/io/casehub/aml/engine/InvestigationStatusMixinTest.java
git commit -m "feat(#78): expand InvestigationStatus with FAILED, CANCELLED, SUSPENDED

Refs #78"
```

---

### Task 2: Exhaustive CaseStatus mapping in AmlInvestigationOutcomeService (#78)

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationOutcomeService.java`
- Modify: `app/src/test/java/io/casehub/aml/engine/AmlInvestigationOutcomeServiceTest.java`

**Interfaces:**
- Consumes: `InvestigationStatus.FAILED`, `InvestigationStatus.CANCELLED`, `InvestigationStatus.SUSPENDED` from Task 1
- Produces: `resolveInvestigation()` now returns FAILED/CANCELLED/SUSPENDED for matching CaseStatus values

- [ ] **Step 1: Write failing tests for new status mappings**

Add helper methods and tests to `AmlInvestigationOutcomeServiceTest`:

```java
private static CaseInstance instanceWithState(UUID caseId, CaseStatus state) {
    final CaseInstance instance = new CaseInstance();
    instance.setUuid(caseId);
    instance.setState(state);
    return instance;
}

@Test
void resolveInvestigation_faulted_returns_failed() {
    final UUID caseId = UUID.randomUUID();
    final CaseInstance instance = instanceWithState(caseId, CaseStatus.FAULTED);
    final AmlInvestigationOutcomeService service = serviceWith(List.of(), instance, null);
    final Optional<InvestigationResolution> result = service.resolveInvestigation(caseId);
    assertTrue(result.isPresent());
    assertEquals(InvestigationStatus.FAILED, result.get().status());
    assertNull(result.get().outcome());
}

@Test
void resolveInvestigation_cancelled_returns_cancelled() {
    final UUID caseId = UUID.randomUUID();
    final CaseInstance instance = instanceWithState(caseId, CaseStatus.CANCELLED);
    final AmlInvestigationOutcomeService service = serviceWith(List.of(), instance, null);
    final Optional<InvestigationResolution> result = service.resolveInvestigation(caseId);
    assertTrue(result.isPresent());
    assertEquals(InvestigationStatus.CANCELLED, result.get().status());
    assertNull(result.get().outcome());
}

@Test
void resolveInvestigation_suspended_returns_suspended() {
    final UUID caseId = UUID.randomUUID();
    final CaseInstance instance = instanceWithState(caseId, CaseStatus.SUSPENDED);
    final AmlInvestigationOutcomeService service = serviceWith(List.of(), instance, null);
    final Optional<InvestigationResolution> result = service.resolveInvestigation(caseId);
    assertTrue(result.isPresent());
    assertEquals(InvestigationStatus.SUSPENDED, result.get().status());
    assertNull(result.get().outcome());
}

@Test
void resolveInvestigation_starting_returns_in_progress() {
    final UUID caseId = UUID.randomUUID();
    final CaseInstance instance = instanceWithState(caseId, CaseStatus.STARTING);
    final AmlInvestigationOutcomeService service = serviceWith(List.of(), instance, null);
    final Optional<InvestigationResolution> result = service.resolveInvestigation(caseId);
    assertTrue(result.isPresent());
    assertEquals(InvestigationStatus.IN_PROGRESS, result.get().status());
}

@Test
void resolveInvestigation_waiting_returns_in_progress() {
    final UUID caseId = UUID.randomUUID();
    final CaseInstance instance = instanceWithState(caseId, CaseStatus.WAITING);
    final AmlInvestigationOutcomeService service = serviceWith(List.of(), instance, null);
    final Optional<InvestigationResolution> result = service.resolveInvestigation(caseId);
    assertTrue(result.isPresent());
    assertEquals(InvestigationStatus.IN_PROGRESS, result.get().status());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl app -am test -Dtest=AmlInvestigationOutcomeServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: `resolveInvestigation_faulted_returns_failed` FAILS — gets IN_PROGRESS instead of FAILED

- [ ] **Step 3: Replace if-check with exhaustive switch**

Replace the `resolveInvestigation` method in `AmlInvestigationOutcomeService.java`:

```java
public Optional<InvestigationResolution> resolveInvestigation(final UUID caseId) {
    CaseInstance instance = caseInstanceCache.get(caseId);
    if (instance == null) {
        instance = caseInstanceRepository
                .findByUuid(caseId, TenancyConstants.DEFAULT_TENANT_ID)
                .await().indefinitely();
    }
    if (instance == null) {
        return Optional.empty();
    }

    InvestigationStatus status = switch (instance.getState()) {
        case STARTING, RUNNING, WAITING -> InvestigationStatus.IN_PROGRESS;
        case COMPLETED -> InvestigationStatus.COMPLETED;
        case FAULTED -> InvestigationStatus.FAILED;
        case CANCELLED -> InvestigationStatus.CANCELLED;
        case SUSPENDED -> InvestigationStatus.SUSPENDED;
    };

    if (status != InvestigationStatus.COMPLETED) {
        return Optional.of(new InvestigationResolution(status, null));
    }
    final InvestigationOutcome outcome = resolveOutcome(caseId);
    return Optional.of(new InvestigationResolution(InvestigationStatus.COMPLETED, outcome));
}
```

- [ ] **Step 4: Run all tests**

Run: `mvn -pl app -am test -Dtest=AmlInvestigationOutcomeServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 16 tests PASS (11 existing + 5 new)

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/aml/engine/AmlInvestigationOutcomeService.java app/src/test/java/io/casehub/aml/engine/AmlInvestigationOutcomeServiceTest.java
git commit -m "feat(#78): exhaustive CaseStatus→InvestigationStatus mapping

Replaces lossy if-check with exhaustive switch expression. FAULTED maps to
FAILED, CANCELLED and SUSPENDED map directly. No default arm — new
CaseStatus values cause a compilation failure, forcing explicit mapping.

Closes #78"
```

---

### Task 3: GDPR erasure service and result type (#62)

**Files:**
- Create: `api/src/main/java/io/casehub/aml/compliance/AmlErasureResult.java`
- Create: `app/src/main/java/io/casehub/aml/compliance/AmlErasureService.java`
- Create: `app/src/test/java/io/casehub/aml/compliance/AmlErasureServiceTest.java`
- Modify: `app/src/main/resources/application.properties` (enable erasure receipt)
- Modify: `app/src/test/resources/application.properties` (enable erasure receipt + activate JPA repo)

**Interfaces:**
- Consumes: `LedgerErasureService.erase(String, ErasureReason)` → `ErasureResult` (foundation)
- Produces: `AmlErasureService.erase(String, ErasureReason)` → `AmlErasureResult` (used by Task 4)

- [ ] **Step 1: Create `AmlErasureResult` record in api/**

```java
package io.casehub.aml.compliance;

import java.util.UUID;

public record AmlErasureResult(
        String erasedActorId,
        boolean mappingFound,
        long affectedEntryCount,
        UUID receiptEntryId) {}
```

- [ ] **Step 2: Write `AmlErasureServiceTest`**

```java
package io.casehub.aml.compliance;

import io.casehub.ledger.api.model.ErasureReason;
import io.casehub.ledger.runtime.privacy.LedgerErasureService;
import io.casehub.ledger.runtime.privacy.LedgerErasureService.ErasureResult;
import org.junit.jupiter.api.Test;

import java.util.Optional;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class AmlErasureServiceTest {

    @Test
    void erase_maps_ledger_result_with_receipt() {
        final UUID receiptId = UUID.randomUUID();
        final ErasureResult ledgerResult = new ErasureResult(
                "officer-jane", true, 5L, Optional.of(receiptId));
        final AmlErasureService service = new AmlErasureService(
                (actorId, reason) -> ledgerResult);
        final AmlErasureResult result = service.erase("officer-jane", ErasureReason.GDPR_ART_17_REQUEST);
        assertEquals("officer-jane", result.erasedActorId());
        assertTrue(result.mappingFound());
        assertEquals(5L, result.affectedEntryCount());
        assertEquals(receiptId, result.receiptEntryId());
    }

    @Test
    void erase_maps_ledger_result_without_receipt() {
        final ErasureResult ledgerResult = new ErasureResult(
                "officer-jane", false, 0L, Optional.empty());
        final AmlErasureService service = new AmlErasureService(
                (actorId, reason) -> ledgerResult);
        final AmlErasureResult result = service.erase("officer-jane", ErasureReason.GDPR_ART_17_REQUEST);
        assertEquals("officer-jane", result.erasedActorId());
        assertFalse(result.mappingFound());
        assertEquals(0L, result.affectedEntryCount());
        assertNull(result.receiptEntryId());
    }
}
```

Note: `LedgerErasureService` is a concrete class, not an interface. The test uses a lambda approach — this requires extracting the erasure call behind a functional interface, or using Mockito. Let me use Mockito since that's the project convention:

```java
package io.casehub.aml.compliance;

import io.casehub.ledger.api.model.ErasureReason;
import io.casehub.ledger.runtime.privacy.LedgerErasureService;
import io.casehub.ledger.runtime.privacy.LedgerErasureService.ErasureResult;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import java.util.Optional;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.when;

class AmlErasureServiceTest {

    @Test
    void erase_maps_ledger_result_with_receipt() {
        final UUID receiptId = UUID.randomUUID();
        final LedgerErasureService ledger = Mockito.mock(LedgerErasureService.class);
        when(ledger.erase(eq("officer-jane"), eq(ErasureReason.GDPR_ART_17_REQUEST)))
                .thenReturn(new ErasureResult("officer-jane", true, 5L, Optional.of(receiptId)));
        final AmlErasureService service = new AmlErasureService(ledger);

        final AmlErasureResult result = service.erase("officer-jane", ErasureReason.GDPR_ART_17_REQUEST);

        assertEquals("officer-jane", result.erasedActorId());
        assertTrue(result.mappingFound());
        assertEquals(5L, result.affectedEntryCount());
        assertEquals(receiptId, result.receiptEntryId());
    }

    @Test
    void erase_maps_empty_receipt_to_null() {
        final LedgerErasureService ledger = Mockito.mock(LedgerErasureService.class);
        when(ledger.erase(anyString(), any()))
                .thenReturn(new ErasureResult("officer-jane", false, 0L, Optional.empty()));
        final AmlErasureService service = new AmlErasureService(ledger);

        final AmlErasureResult result = service.erase("officer-jane", ErasureReason.GDPR_ART_17_REQUEST);

        assertFalse(result.mappingFound());
        assertEquals(0L, result.affectedEntryCount());
        assertNull(result.receiptEntryId());
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn -pl app -am test -Dtest=AmlErasureServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `AmlErasureService` doesn't exist

- [ ] **Step 4: Create `AmlErasureService`**

```java
package io.casehub.aml.compliance;

import io.casehub.ledger.api.model.ErasureReason;
import io.casehub.ledger.runtime.privacy.LedgerErasureService;
import io.casehub.ledger.runtime.privacy.LedgerErasureService.ErasureResult;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class AmlErasureService {

    private final LedgerErasureService ledgerErasureService;

    @Inject
    public AmlErasureService(final LedgerErasureService ledgerErasureService) {
        this.ledgerErasureService = ledgerErasureService;
    }

    public AmlErasureResult erase(final String actorId, final ErasureReason reason) {
        final ErasureResult ledgerResult = ledgerErasureService.erase(actorId, reason);
        return new AmlErasureResult(
                ledgerResult.rawActorId(),
                ledgerResult.mappingFound(),
                ledgerResult.affectedEntryCount(),
                ledgerResult.receiptEntryId().orElse(null));
    }
}
```

- [ ] **Step 5: Run tests**

Run: `mvn -pl app -am test -Dtest=AmlErasureServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 6: Enable erasure receipt config**

Add to `app/src/main/resources/application.properties` (after the `casehub.ledger.datasource=qhorus` line):

```properties
casehub.ledger.erasure-receipt.enabled=true
```

Add to `app/src/test/resources/application.properties` (after the `casehub.ledger.identity.tokenisation.enabled=true` line):

```properties
casehub.ledger.erasure-receipt.enabled=true
```

Add `JpaErasureReceiptRepository` to the test `selected-alternatives` line (append to existing comma-separated list):

```
,io.casehub.ledger.runtime.repository.jpa.JpaErasureReceiptRepository
```

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/aml/compliance/AmlErasureResult.java app/src/main/java/io/casehub/aml/compliance/AmlErasureService.java app/src/test/java/io/casehub/aml/compliance/AmlErasureServiceTest.java app/src/main/resources/application.properties app/src/test/resources/application.properties
git commit -m "feat(#62): add AmlErasureService and enable erasure receipts

Wraps LedgerErasureService with domain-specific AmlErasureResult. Enables
casehub.ledger.erasure-receipt.enabled=true in both main and test properties.
Activates JpaErasureReceiptRepository in test selected-alternatives.

Refs #62"
```

---

### Task 4: REST resource update + compliance evidence enhancement (#62)

**Files:**
- Modify: `api/src/main/java/io/casehub/aml/compliance/GdprErasureRequirement.java`
- Modify: `app/src/main/java/io/casehub/aml/compliance/AmlLayer7Resource.java` (the `AmlGdprErasureResource` inner class)
- Modify: `app/src/main/java/io/casehub/aml/compliance/AmlComplianceEvidenceService.java`
- Modify: `app/src/test/java/io/casehub/aml/compliance/AmlComplianceEvidenceServiceTest.java`

**Interfaces:**
- Consumes: `AmlErasureService.erase()` from Task 3, `ErasureReceiptRepository.countByTenant()` from ledger#160, `LedgerConfig.identity().tokenisation().enabled()`, `LedgerConfig.erasureReceipt().enabled()`
- Produces: updated REST response, dynamic compliance evidence

- [ ] **Step 1: Update `GdprErasureRequirement`**

Replace `api/src/main/java/io/casehub/aml/compliance/GdprErasureRequirement.java`:

```java
package io.casehub.aml.compliance;

public record GdprErasureRequirement(
        String id,
        String citation,
        String mechanism,
        RequirementStatus status,
        boolean tokenisationEnabled,
        boolean erasureReceiptEnabled,
        long erasureReceiptCount,
        String erasureEndpoint) {

    public static final String REQUIREMENT_ID = "GDPR-ART17-ERASURE";
    public static final String CITATION =
            "GDPR Art. 17 / FATF privacy obligation — PII erasure preserving audit structure";
    public static final String MECHANISM =
            "LedgerErasureService pseudonymizes actorId in ledger_entry rows via ActorIdentity token. " +
            "Audit entries remain intact; actor identity is replaced with an opaque token. " +
            "Tamper-evident ErasureReceiptLedgerEntry records each erasure in the Merkle chain.";
    public static final String ERASURE_ENDPOINT = "POST /api/actors/{actorId}/erasure";
}
```

- [ ] **Step 2: Update `AmlGdprErasureResource`**

In `AmlLayer7Resource.java`, replace the `AmlGdprErasureResource` class (starts at line 46):

```java
@ApplicationScoped
@Path("/api/actors/{actorId}/erasure")
@Produces(MediaType.APPLICATION_JSON)
class AmlGdprErasureResource {

    @Inject
    AmlErasureService erasureService;

    @POST
    public AmlErasureResult eraseActor(@PathParam("actorId") String actorId) {
        return erasureService.erase(actorId, ErasureReason.GDPR_ART_17_REQUEST);
    }
}
```

Update imports: add `AmlErasureResult`, `AmlErasureService`; remove direct `LedgerErasureService` import.

- [ ] **Step 3: Write failing compliance evidence tests**

Add to `AmlComplianceEvidenceServiceTest`:

```java
@Mock LedgerConfig ledgerConfig;
@Mock LedgerConfig.IdentityConfig identityConfig;
@Mock LedgerConfig.IdentityConfig.TokenisationConfig tokenisationConfig;
@Mock LedgerConfig.ErasureReceiptConfig erasureReceiptConfig;
@Mock ErasureReceiptRepository erasureReceiptRepo;
```

Update the `setUp()` method:

```java
@BeforeEach
void setUp() {
    MockitoAnnotations.openMocks(this);
    when(ledgerConfig.identity()).thenReturn(identityConfig);
    when(identityConfig.tokenisation()).thenReturn(tokenisationConfig);
    when(ledgerConfig.erasureReceipt()).thenReturn(erasureReceiptConfig);
    service = new AmlComplianceEvidenceService(
            ledgerRepo, verificationService, attestationRepo,
            workerDecisionRepo, em, mockReconciler,
            ledgerConfig, erasureReceiptRepo);
}
```

Add test methods:

```java
@Test
void gdprErasure_both_enabled_returns_closed() {
    when(tokenisationConfig.enabled()).thenReturn(true);
    when(erasureReceiptConfig.enabled()).thenReturn(true);
    when(erasureReceiptRepo.countByTenant(any())).thenReturn(3L);
    setupMinimalLedgerEntries();

    ComplianceEvidence evidence = service.assembleEvidence(caseId);

    assertEquals(RequirementStatus.CLOSED, evidence.gdprErasure().status());
    assertTrue(evidence.gdprErasure().tokenisationEnabled());
    assertTrue(evidence.gdprErasure().erasureReceiptEnabled());
    assertEquals(3L, evidence.gdprErasure().erasureReceiptCount());
}

@Test
void gdprErasure_tokenisation_only_returns_partial() {
    when(tokenisationConfig.enabled()).thenReturn(true);
    when(erasureReceiptConfig.enabled()).thenReturn(false);
    when(erasureReceiptRepo.countByTenant(any())).thenReturn(0L);
    setupMinimalLedgerEntries();

    ComplianceEvidence evidence = service.assembleEvidence(caseId);

    assertEquals(RequirementStatus.PARTIAL, evidence.gdprErasure().status());
    assertTrue(evidence.gdprErasure().tokenisationEnabled());
    assertFalse(evidence.gdprErasure().erasureReceiptEnabled());
}

@Test
void gdprErasure_neither_enabled_returns_gap() {
    when(tokenisationConfig.enabled()).thenReturn(false);
    when(erasureReceiptConfig.enabled()).thenReturn(false);
    when(erasureReceiptRepo.countByTenant(any())).thenReturn(0L);
    setupMinimalLedgerEntries();

    ComplianceEvidence evidence = service.assembleEvidence(caseId);

    assertEquals(RequirementStatus.GAP, evidence.gdprErasure().status());
}

@Test
void gdprErasure_db_failure_degrades_gracefully() {
    when(tokenisationConfig.enabled()).thenReturn(true);
    when(erasureReceiptConfig.enabled()).thenReturn(true);
    when(erasureReceiptRepo.countByTenant(any())).thenThrow(new RuntimeException("DB down"));
    setupMinimalLedgerEntries();

    ComplianceEvidence evidence = service.assembleEvidence(caseId);

    assertEquals(RequirementStatus.CLOSED, evidence.gdprErasure().status());
    assertEquals(0L, evidence.gdprErasure().erasureReceiptCount());
}

private void setupMinimalLedgerEntries() {
    var opened = caseOpenedEntry(caseId, caseOpenedId);
    when(ledgerRepo.findBySubjectId(eq(caseId), any())).thenReturn(List.of(opened));
    when(workerDecisionRepo.findAllByCaseId(caseId)).thenReturn(List.of());
    when(attestationRepo.findByInvestigationCaseId(caseId)).thenReturn(List.of());
    when(mockReconciler.reconcileIfNeeded(eq(caseId), any(), any())).thenReturn(List.of());
}
```

Import `ErasureReceiptRepository`:

```java
import io.casehub.ledger.runtime.repository.ErasureReceiptRepository;
import io.casehub.ledger.runtime.config.LedgerConfig;
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `mvn -pl app -am test -Dtest=AmlComplianceEvidenceServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `AmlComplianceEvidenceService` constructor doesn't accept `LedgerConfig` and `ErasureReceiptRepository`

- [ ] **Step 5: Update `AmlComplianceEvidenceService`**

Add new fields and update constructor:

```java
private final LedgerConfig ledgerConfig;
private final ErasureReceiptRepository erasureReceiptRepo;

@Inject
public AmlComplianceEvidenceService(
        LedgerEntryRepository ledgerRepo,
        LedgerVerificationService verificationService,
        AmlTrustAttestationRepository attestationRepo,
        AmlWorkerDecisionRepository workerDecisionRepo,
        EntityManager em,
        AmlAttestationReconciler reconciler,
        LedgerConfig ledgerConfig,
        ErasureReceiptRepository erasureReceiptRepo) {
    this.ledgerRepo = ledgerRepo;
    this.verificationService = verificationService;
    this.attestationRepo = attestationRepo;
    this.workerDecisionRepo = workerDecisionRepo;
    this.em = em;
    this.reconciler = reconciler;
    this.ledgerConfig = ledgerConfig;
    this.erasureReceiptRepo = erasureReceiptRepo;
}
```

Add imports:

```java
import io.casehub.ledger.runtime.config.LedgerConfig;
import io.casehub.ledger.runtime.repository.ErasureReceiptRepository;
```

Replace the `buildGdprErasure()` method:

```java
private GdprErasureRequirement buildGdprErasure() {
    boolean tokenisationEnabled = ledgerConfig.identity().tokenisation().enabled();
    boolean receiptEnabled = ledgerConfig.erasureReceipt().enabled();

    long receiptCount = 0L;
    try {
        receiptCount = erasureReceiptRepo.countByTenant(
                io.casehub.platform.api.identity.TenancyConstants.DEFAULT_TENANT_ID);
    } catch (Exception ignored) {
    }

    RequirementStatus status;
    if (tokenisationEnabled && receiptEnabled) {
        status = RequirementStatus.CLOSED;
    } else if (tokenisationEnabled || receiptEnabled) {
        status = RequirementStatus.PARTIAL;
    } else {
        status = RequirementStatus.GAP;
    }

    return new GdprErasureRequirement(
            GdprErasureRequirement.REQUIREMENT_ID,
            GdprErasureRequirement.CITATION,
            GdprErasureRequirement.MECHANISM,
            status, tokenisationEnabled, receiptEnabled, receiptCount,
            GdprErasureRequirement.ERASURE_ENDPOINT);
}
```

- [ ] **Step 6: Fix any existing tests broken by the constructor change**

The `setUp()` in `AmlComplianceEvidenceServiceTest` must pass the two new parameters. All existing test methods should also stub `tokenisationConfig` and `erasureReceiptConfig` via the `setUp()` defaults (already done in Step 3). Set sensible defaults in `setUp()`:

```java
when(tokenisationConfig.enabled()).thenReturn(true);
when(erasureReceiptConfig.enabled()).thenReturn(true);
when(erasureReceiptRepo.countByTenant(any())).thenReturn(0L);
```

This ensures existing tests continue to pass without modification — GDPR evidence defaults to CLOSED with 0 receipts.

- [ ] **Step 7: Run all compliance evidence tests**

Run: `mvn -pl app -am test -Dtest=AmlComplianceEvidenceServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all tests PASS (existing + 4 new)

- [ ] **Step 8: Run full test suite**

Run: `mvn -pl app -am test -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all tests PASS

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/aml/compliance/GdprErasureRequirement.java app/src/main/java/io/casehub/aml/compliance/AmlLayer7Resource.java app/src/main/java/io/casehub/aml/compliance/AmlComplianceEvidenceService.java app/src/test/java/io/casehub/aml/compliance/AmlComplianceEvidenceServiceTest.java
git commit -m "feat(#62): dynamic GDPR compliance evidence + erasure receipt endpoint

GdprErasureRequirement now queries LedgerConfig for tokenisation/receipt
enabled flags and ErasureReceiptRepository for tenant-wide receipt count.
AmlGdprErasureResource delegates to AmlErasureService. Error handling
matches sibling compliance evidence methods.

Closes #62"
```

---

### Task 5: File deferred issues

**Files:** none (GitHub only)

- [ ] **Step 1: File entity data erasure issue**

```bash
gh issue create --repo casehubio/aml --title "feat: entity-level memory erasure (GDPR Art.17 by account ID)" --body "## Context

AML memory entries are keyed by entityId (account IDs), not actorId. The current erasure endpoint erases ledger identity (actorId pseudonymisation) but does not erase memory about investigated entities.

## What to implement

A separate endpoint or extension to erase CaseMemoryStore data for a given account/entity ID. This is a distinct GDPR data subject erasure path — the subject of investigation (account holder) exercises their right to be forgotten.

## Origin

Design review R1-08 on #62 — actorId/entityId key mismatch means CaseMemoryStore.eraseEntity(actorId) is a silent no-op."
```

- [ ] **Step 2: File investigation failure context issue**

```bash
gh issue create --repo casehubio/aml --title "feat: failure context on terminal InvestigationStatus (FAILED/CANCELLED)" --body "## Context

InvestigationResolution returns null outcome for FAILED and CANCELLED investigations. Compliance officers need failure context — fault reason, cancellation justification.

## What to implement

Enrich InvestigationResolution with failure context from the engine's CaseInstance APIs for non-COMPLETED terminal states.

## Origin

Design review R1-03 on #78."
```

- [ ] **Step 3: File automated retention expiry issue**

```bash
gh issue create --repo casehubio/aml --title "feat: automated retention expiry (ErasureReason.RETENTION_EXPIRED)" --body "## Context

ErasureReason.RETENTION_EXPIRED exists but no scheduled job triggers retention-based erasure.

## What to implement

A @Scheduled job that identifies actors whose data retention window has expired and triggers LedgerErasureService.erase() with RETENTION_EXPIRED reason.

## Origin

Spec not-in-scope for #62."
```

- [ ] **Step 4: File GDPR Art.22 decision records issue**

```bash
gh issue create --repo casehubio/aml --title "feat: GDPR Art.22 decision record compliance supplements" --body "## Context

Separate Layer 7 concern — decision record compliance supplements need implementation for full Art.22 compliance.

## Origin

Spec not-in-scope for #62."
```

- [ ] **Step 5: Commit** (nothing to commit — GitHub issues only)

---
