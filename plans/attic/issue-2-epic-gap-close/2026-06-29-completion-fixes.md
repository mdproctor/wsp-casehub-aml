# Completion Detection Consolidation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Consolidate duplicated completion detection into `AmlInvestigationOutcomeService`, capture officer rejection reasons end-to-end, and fill test gaps — closing #74, #75, #77.

**Architecture:** Create `InvestigationResolution` domain abstraction bridging engine case lifecycle and ledger audit outcome. The service moves from `compliance/` to `engine/` package, gains cache/repo deps, and exposes `resolveInvestigation()` returning `Optional<InvestigationResolution>`. Both Layer 6 and Layer 9 resources delegate to this method. Rejection reason flows from `WorkItemLifecycleEvent.detail()` through the observer and ledger entry to `InvestigationOutcome.reason`.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA (JOINED inheritance), Flyway, JUnit 5, REST Assured, Awaitility

## Global Constraints

- `api/` module has zero framework dependencies — no Jackson, no JPA, no Quarkus annotations. JSON binding uses mixins in `AmlJacksonConfig` (app/ module).
- All ledger entry writes must guarantee non-null `tenancyId` (protocol PP-20260610-ae4535). Use `TenancyConstants.DEFAULT_TENANT_ID`.
- `domainContentBytes()` changes must be backward-compatible — existing Merkle leaf hashes must be reproducible from loaded entity state.
- Flyway migrations go in `db/aml-ledger/migration/` (qhorus datasource). Next safe version: V2012.
- Test commands: `mvn -pl <module> -am test -Dtest=ClassName -Dsurefire.failIfNoSpecifiedTests=false`
- Gate approval before attestation wait in integration tests (GE-20260628-dbc656).
- `casehub.ledger.hash-chain.enabled=false` in test properties (H2 lacks row-level locking).

---

### Task 1: Domain types and api/ tests (#74)

**Files:**
- Create: `api/src/main/java/io/casehub/aml/domain/InvestigationStatus.java`
- Create: `api/src/main/java/io/casehub/aml/domain/InvestigationResolution.java`
- Modify: `api/src/main/java/io/casehub/aml/domain/InvestigationOutcome.java`
- Modify: `api/src/test/java/io/casehub/aml/domain/InvestigationOutcomeTest.java`
- Create: `api/src/test/java/io/casehub/aml/domain/InvestigationStatusTest.java`

**Interfaces:**
- Consumes: nothing (leaf types)
- Produces:
  - `InvestigationStatus.IN_PROGRESS`, `InvestigationStatus.COMPLETED`, `InvestigationStatus.toWireFormat()`, `InvestigationStatus.fromWireFormat(String)`
  - `InvestigationResolution(InvestigationStatus status, InvestigationOutcome outcome)`
  - `InvestigationOutcome(String type, String reason)`, `InvestigationOutcome.fromReviewDecision(String reviewDecision, String rejectionReason)`

- [ ] **Step 1: Write `InvestigationStatusTest`**

```java
package io.casehub.aml.domain;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class InvestigationStatusTest {

    @Test
    void in_progress_wire_format() {
        assertEquals("in-progress", InvestigationStatus.IN_PROGRESS.toWireFormat());
    }

    @Test
    void completed_wire_format() {
        assertEquals("completed", InvestigationStatus.COMPLETED.toWireFormat());
    }

    @Test
    void from_wire_format_in_progress() {
        assertEquals(InvestigationStatus.IN_PROGRESS, InvestigationStatus.fromWireFormat("in-progress"));
    }

    @Test
    void from_wire_format_completed() {
        assertEquals(InvestigationStatus.COMPLETED, InvestigationStatus.fromWireFormat("completed"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -pl api -am test -Dtest=InvestigationStatusTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `InvestigationStatus` does not exist

- [ ] **Step 3: Implement `InvestigationStatus`**

```java
package io.casehub.aml.domain;

public enum InvestigationStatus {
    IN_PROGRESS,
    COMPLETED;

    public String toWireFormat() {
        return name().toLowerCase().replace('_', '-');
    }

    public static InvestigationStatus fromWireFormat(String value) {
        return valueOf(value.toUpperCase().replace('-', '_'));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -pl api -am test -Dtest=InvestigationStatusTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 4 tests PASS

- [ ] **Step 5: Update `InvestigationOutcomeTest` for new factory signature**

Replace the entire test class:

```java
package io.casehub.aml.domain;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class InvestigationOutcomeTest {

    @Test
    void approved_maps_to_sar_filed() {
        final InvestigationOutcome outcome = InvestigationOutcome.fromReviewDecision("APPROVED", null);
        assertNotNull(outcome);
        assertEquals("sar-filed", outcome.type());
        assertNull(outcome.reason());
    }

    @Test
    void rejected_maps_to_gate_rejected() {
        final InvestigationOutcome outcome = InvestigationOutcome.fromReviewDecision("REJECTED", "Insufficient evidence");
        assertNotNull(outcome);
        assertEquals("gate-rejected", outcome.type());
        assertEquals("Insufficient evidence", outcome.reason());
    }

    @Test
    void rejected_without_reason() {
        final InvestigationOutcome outcome = InvestigationOutcome.fromReviewDecision("REJECTED", null);
        assertNotNull(outcome);
        assertEquals("gate-rejected", outcome.type());
        assertNull(outcome.reason());
    }

    @Test
    void unknown_maps_to_decision_not_recorded() {
        final InvestigationOutcome outcome = InvestigationOutcome.fromReviewDecision("UNKNOWN", null);
        assertNotNull(outcome);
        assertEquals("decision-not-recorded", outcome.type());
        assertNull(outcome.reason());
    }

    @Test
    void null_reviewDecision_throws() {
        assertThrows(NullPointerException.class,
                () -> InvestigationOutcome.fromReviewDecision(null, null));
    }

    @Test
    void unrecognised_value_throws() {
        assertThrows(IllegalStateException.class,
                () -> InvestigationOutcome.fromReviewDecision("SOMETHING_ELSE", null));
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `mvn -pl api -am test -Dtest=InvestigationOutcomeTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `fromReviewDecision` takes 1 arg, not 2; `reason()` does not exist

- [ ] **Step 7: Update `InvestigationOutcome`**

Replace the entire file:

```java
package io.casehub.aml.domain;

import java.util.Objects;

public record InvestigationOutcome(String type, String reason) {

    public static InvestigationOutcome fromReviewDecision(
            final String reviewDecision, final String rejectionReason) {
        Objects.requireNonNull(reviewDecision,
                "reviewDecision must not be null — column is NOT NULL");
        return switch (reviewDecision) {
            case "APPROVED" -> new InvestigationOutcome("sar-filed", null);
            case "REJECTED" -> new InvestigationOutcome("gate-rejected", rejectionReason);
            case "UNKNOWN" -> new InvestigationOutcome("decision-not-recorded", null);
            default -> throw new IllegalStateException(
                    "Unexpected reviewDecision: " + reviewDecision);
        };
    }
}
```

- [ ] **Step 8: Create `InvestigationResolution`**

```java
package io.casehub.aml.domain;

public record InvestigationResolution(
        InvestigationStatus status,
        InvestigationOutcome outcome) {}
```

- [ ] **Step 9: Run all api/ tests**

Run: `mvn -pl api -am test -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all tests PASS (InvestigationOutcomeTest + InvestigationStatusTest)

- [ ] **Step 10: Commit**

```
feat(#74): add InvestigationStatus, InvestigationResolution, update InvestigationOutcome

Refs #74
```

---

### Task 2: Service consolidation + Jackson mixin (#74)

**Files:**
- Move: `app/src/main/java/io/casehub/aml/compliance/AmlInvestigationOutcomeService.java` → `app/src/main/java/io/casehub/aml/engine/AmlInvestigationOutcomeService.java`
- Modify: `app/src/main/java/io/casehub/aml/AmlJacksonConfig.java`
- Move: `app/src/test/java/io/casehub/aml/engine/AmlInvestigationOutcomeServiceTest.java` (stays in engine/ — matches new service location)
- Create: `app/src/test/java/io/casehub/aml/engine/InvestigationStatusMixinTest.java`

**Interfaces:**
- Consumes: `InvestigationStatus`, `InvestigationResolution`, `InvestigationOutcome` (from Task 1); `CaseInstanceCache.get(UUID)`, `CaseInstanceRepository.findByUuid(UUID, String)`, `LedgerEntryRepository.findBySubjectId(UUID, String)`
- Produces: `AmlInvestigationOutcomeService.resolveInvestigation(UUID caseId)` → `Optional<InvestigationResolution>`; `AmlInvestigationOutcomeService.resolveOutcome(UUID caseId)` → `InvestigationOutcome` (package-private)

- [ ] **Step 1: Write `resolveInvestigation()` unit tests**

Add to `AmlInvestigationOutcomeServiceTest` (which stays in `io.casehub.aml.engine`). First, update the test helper to support the new constructor and add stub implementations for the new deps. Add these tests at the end of the class:

```java
@Test
void resolveInvestigation_cache_hit_completed_returns_completed_with_outcome() {
    final UUID caseId = UUID.randomUUID();
    final AmlSarOfficerReviewedLedgerEntry entry = officerEntry("APPROVED", ActorType.HUMAN);
    final CaseInstance instance = completedInstance(caseId);
    final AmlInvestigationOutcomeService service = serviceWith(List.of(entry), instance, null);
    final Optional<InvestigationResolution> result = service.resolveInvestigation(caseId);
    assertTrue(result.isPresent());
    assertEquals(InvestigationStatus.COMPLETED, result.get().status());
    assertNotNull(result.get().outcome());
    assertEquals("sar-filed", result.get().outcome().type());
}

@Test
void resolveInvestigation_cache_hit_not_completed_returns_in_progress() {
    final UUID caseId = UUID.randomUUID();
    final CaseInstance instance = inProgressInstance(caseId);
    final AmlInvestigationOutcomeService service = serviceWith(List.of(), instance, null);
    final Optional<InvestigationResolution> result = service.resolveInvestigation(caseId);
    assertTrue(result.isPresent());
    assertEquals(InvestigationStatus.IN_PROGRESS, result.get().status());
    assertNull(result.get().outcome());
}

@Test
void resolveInvestigation_cache_miss_repo_hit_returns_completed() {
    final UUID caseId = UUID.randomUUID();
    final AmlSarOfficerReviewedLedgerEntry entry = officerEntry("REJECTED", ActorType.HUMAN);
    final CaseInstance instance = completedInstance(caseId);
    final AmlInvestigationOutcomeService service = serviceWith(List.of(entry), null, instance);
    final Optional<InvestigationResolution> result = service.resolveInvestigation(caseId);
    assertTrue(result.isPresent());
    assertEquals(InvestigationStatus.COMPLETED, result.get().status());
    assertEquals("gate-rejected", result.get().outcome().type());
}

@Test
void resolveInvestigation_cache_miss_repo_miss_returns_empty() {
    final UUID caseId = UUID.randomUUID();
    final AmlInvestigationOutcomeService service = serviceWith(List.of(), null, null);
    final Optional<InvestigationResolution> result = service.resolveInvestigation(caseId);
    assertTrue(result.isEmpty());
}
```

Add helper methods for creating `CaseInstance` stubs and the updated `serviceWith`:

```java
private static CaseInstance completedInstance(final UUID caseId) {
    final CaseInstance instance = new CaseInstance();
    instance.setUuid(caseId);
    instance.setState(CaseStatus.COMPLETED);
    return instance;
}

private static CaseInstance inProgressInstance(final UUID caseId) {
    final CaseInstance instance = new CaseInstance();
    instance.setUuid(caseId);
    instance.setState(CaseStatus.ACTIVE);
    return instance;
}
```

Update the `serviceWith` to accept cache/repo stubs. Add a new overload:

```java
private static AmlInvestigationOutcomeService serviceWith(
        final List<LedgerEntry> entries,
        final CaseInstance cacheResult,
        final CaseInstance repoResult) {
    final LedgerEntryRepository repo = /* existing anonymous impl — same as current */;
    final CaseInstanceCache cache = caseId -> cacheResult;
    final CaseInstanceRepository caseRepo = new CaseInstanceRepository() {
        @Override
        public Uni<CaseInstance> findByUuid(UUID uuid, String tenancyId) {
            return Uni.createFrom().item(repoResult);
        }
        @Override public Uni<CaseInstance> save(CaseInstance i, String t) { return Uni.createFrom().nullItem(); }
        @Override public Uni<CaseInstance> update(CaseInstance i, String t) { return Uni.createFrom().nullItem(); }
        @Override public Uni<Void> updateStateAndAppendEvent(CaseInstance i, EventLog e, String t) { return Uni.createFrom().voidItem(); }
    };
    return new AmlInvestigationOutcomeService(repo, cache, caseRepo);
}
```

Keep the existing `serviceWith(List<LedgerEntry>)` overload for backward compat with existing `resolveOutcome()` tests — just delegate to the new overload with null cache/repo:

```java
private static AmlInvestigationOutcomeService serviceWith(final List<LedgerEntry> entries) {
    return serviceWith(entries, null, null);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl app -am test -Dtest=AmlInvestigationOutcomeServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `resolveInvestigation()` does not exist, constructor takes 1 arg not 3

- [ ] **Step 3: Move service to `engine/` package and implement**

Use IntelliJ MCP to move the file (updates all imports project-wide):

```
ide_move_file: file=app/src/main/java/io/casehub/aml/compliance/AmlInvestigationOutcomeService.java
               destination=app/src/main/java/io/casehub/aml/engine/
```

Then update the class. The full updated file:

```java
package io.casehub.aml.engine;

import io.casehub.aml.domain.InvestigationOutcome;
import io.casehub.aml.domain.InvestigationResolution;
import io.casehub.aml.domain.InvestigationStatus;
import io.casehub.aml.ledger.AmlSarOfficerReviewedLedgerEntry;
import io.casehub.api.model.CaseStatus;
import io.casehub.engine.common.internal.model.CaseInstance;
import io.casehub.engine.common.spi.CaseInstanceRepository;
import io.casehub.engine.common.spi.cache.CaseInstanceCache;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.platform.api.identity.TenancyConstants;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.Comparator;
import java.util.Optional;
import java.util.UUID;

@ApplicationScoped
public class AmlInvestigationOutcomeService {

    private static final Comparator<AmlSarOfficerReviewedLedgerEntry> HUMAN_FIRST_LATEST_SEQ =
            Comparator.<AmlSarOfficerReviewedLedgerEntry, Integer>comparing(
                    e -> e.actorType == ActorType.HUMAN ? 0 : 1)
            .thenComparing(e -> e.sequenceNumber, Comparator.reverseOrder());

    private final LedgerEntryRepository ledgerEntryRepository;
    private final CaseInstanceCache caseInstanceCache;
    private final CaseInstanceRepository caseInstanceRepository;

    @Inject
    public AmlInvestigationOutcomeService(
            final LedgerEntryRepository ledgerEntryRepository,
            final CaseInstanceCache caseInstanceCache,
            final CaseInstanceRepository caseInstanceRepository) {
        this.ledgerEntryRepository = ledgerEntryRepository;
        this.caseInstanceCache = caseInstanceCache;
        this.caseInstanceRepository = caseInstanceRepository;
    }

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
        if (instance.getState() != CaseStatus.COMPLETED) {
            return Optional.of(new InvestigationResolution(InvestigationStatus.IN_PROGRESS, null));
        }
        final InvestigationOutcome outcome = resolveOutcome(caseId);
        return Optional.of(new InvestigationResolution(InvestigationStatus.COMPLETED, outcome));
    }

    InvestigationOutcome resolveOutcome(final UUID caseId) {
        return ledgerEntryRepository
                .findBySubjectId(caseId, TenancyConstants.DEFAULT_TENANT_ID).stream()
                .filter(AmlSarOfficerReviewedLedgerEntry.class::isInstance)
                .map(AmlSarOfficerReviewedLedgerEntry.class::cast)
                .min(HUMAN_FIRST_LATEST_SEQ)
                .map(e -> InvestigationOutcome.fromReviewDecision(e.reviewDecision, e.rejectionReason))
                .orElse(null);
    }
}
```

Note: `e.rejectionReason` does not exist yet on the entity — it will be added in Task 4. For now, pass `null` as the second argument to avoid compilation failure:

```java
.map(e -> InvestigationOutcome.fromReviewDecision(e.reviewDecision, null))
```

The `e.rejectionReason` change happens in Task 4 Step 6 when the field is added.

- [ ] **Step 4: Write `InvestigationStatusMixinTest`**

```java
package io.casehub.aml.engine;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.aml.AmlJacksonConfig;
import io.casehub.aml.domain.InvestigationStatus;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertEquals;

class InvestigationStatusMixinTest {

    private ObjectMapper mapper;

    @BeforeEach
    void setUp() {
        mapper = new ObjectMapper();
        new AmlJacksonConfig().customize(mapper);
    }

    @Test
    void serializes_in_progress() throws Exception {
        assertEquals("\"in-progress\"", mapper.writeValueAsString(InvestigationStatus.IN_PROGRESS));
    }

    @Test
    void serializes_completed() throws Exception {
        assertEquals("\"completed\"", mapper.writeValueAsString(InvestigationStatus.COMPLETED));
    }

    @Test
    void deserializes_in_progress() throws Exception {
        assertEquals(InvestigationStatus.IN_PROGRESS,
                mapper.readValue("\"in-progress\"", InvestigationStatus.class));
    }

    @Test
    void deserializes_completed() throws Exception {
        assertEquals(InvestigationStatus.COMPLETED,
                mapper.readValue("\"completed\"", InvestigationStatus.class));
    }
}
```

- [ ] **Step 5: Add `InvestigationStatusMixin` to `AmlJacksonConfig`**

Add inside `AmlJacksonConfig`:

```java
import com.fasterxml.jackson.annotation.JsonCreator;
import com.fasterxml.jackson.annotation.JsonValue;
import io.casehub.aml.domain.InvestigationStatus;

// Add this inner interface:
interface InvestigationStatusMixin {
    @JsonValue String toWireFormat();
    @JsonCreator static InvestigationStatus fromWireFormat(String value) { return null; }
}

// Add to customize():
mapper.addMixIn(InvestigationStatus.class, InvestigationStatusMixin.class);
```

- [ ] **Step 6: Add sequenceNumber tiebreaker and failure-only tests**

Add to `AmlInvestigationOutcomeServiceTest`:

```java
@Test
void prefers_higher_sequenceNumber_among_human_entries() {
    final AmlSarOfficerReviewedLedgerEntry older = officerEntry("REJECTED", ActorType.HUMAN);
    older.sequenceNumber = 3;
    final AmlSarOfficerReviewedLedgerEntry newer = officerEntry("APPROVED", ActorType.HUMAN);
    newer.sequenceNumber = 7;
    final AmlInvestigationOutcomeService service = serviceWith(List.of(older, newer));
    final InvestigationOutcome outcome = service.resolveOutcome(UUID.randomUUID());
    assertNotNull(outcome);
    assertEquals("sar-filed", outcome.type());
}

@Test
void system_only_rejected_returns_gate_rejected() {
    final AmlSarOfficerReviewedLedgerEntry entry = officerEntry("REJECTED", ActorType.SYSTEM);
    final AmlInvestigationOutcomeService service = serviceWith(List.of(entry));
    final InvestigationOutcome outcome = service.resolveOutcome(UUID.randomUUID());
    assertNotNull(outcome);
    assertEquals("gate-rejected", outcome.type());
}
```

- [ ] **Step 7: Run all tests**

Run: `mvn -pl app -am test -Dtest=AmlInvestigationOutcomeServiceTest,InvestigationStatusMixinTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```
feat(#74): consolidate completion detection into AmlInvestigationOutcomeService

Move service from compliance/ to engine/. Add resolveInvestigation()
returning Optional<InvestigationResolution>. Add Jackson mixin for
InvestigationStatus.

Refs #74, Refs #77
```

---

### Task 3: Resource simplification + 404 fix (#74)

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/engine/Layer6InvestigationResponse.java`
- Create: `app/src/main/java/io/casehub/aml/engine/Layer9InvestigationResponse.java`
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlLayer6Resource.java`
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java`
- Modify: `app/src/test/java/io/casehub/aml/engine/AmlLayer6ResourceTest.java`
- Modify: `app/src/test/java/io/casehub/aml/engine/AmlLayer9ResourceTest.java`

**Interfaces:**
- Consumes: `AmlInvestigationOutcomeService.resolveInvestigation(UUID)` (from Task 2); `InvestigationStatus`, `InvestigationResolution`, `InvestigationOutcome` (from Task 1)
- Produces: REST endpoints `GET /api/layer6/investigations/{caseId}` and `GET /api/layer9/investigations/{caseId}` — both return 404 for nonexistent caseIds

- [ ] **Step 1: Write 404 regression tests**

Add to `AmlLayer6ResourceTest`:

```java
@Test
void get_nonexistent_investigation_returns_404() {
    given().when().get("/api/layer6/investigations/" + UUID.randomUUID())
            .then().statusCode(404);
}
```

Add to `AmlLayer9ResourceTest`:

```java
@Test
void get_nonexistent_investigation_returns_404() {
    given().when().get("/api/layer9/investigations/" + UUID.randomUUID())
            .then().statusCode(404);
}
```

- [ ] **Step 2: Run 404 tests to verify they fail (current behavior returns 200)**

Run: `mvn -pl app -am test -Dtest=AmlLayer6ResourceTest#get_nonexistent_investigation_returns_404+AmlLayer9ResourceTest#get_nonexistent_investigation_returns_404 -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — both currently return 200 with `"in-progress"`

- [ ] **Step 3: Update `Layer6InvestigationResponse`**

```java
package io.casehub.aml.engine;

import io.casehub.aml.domain.InvestigationOutcome;
import io.casehub.aml.domain.InvestigationStatus;
import java.util.List;
import java.util.UUID;

public record Layer6InvestigationResponse(
        UUID caseId,
        InvestigationStatus status,
        List<WorkerRoutingDecision> routingDecisions,
        InvestigationOutcome outcome) {}
```

- [ ] **Step 4: Create `Layer9InvestigationResponse`**

```java
package io.casehub.aml.engine;

import io.casehub.aml.domain.InvestigationOutcome;
import io.casehub.aml.domain.InvestigationStatus;
import java.util.UUID;

public record Layer9InvestigationResponse(
        UUID caseId,
        InvestigationStatus status,
        InvestigationOutcome outcome) {}
```

- [ ] **Step 5: Simplify `AmlLayer6Resource`**

Remove `CaseInstanceCache` and `CaseInstanceRepository` injections. Change `getInvestigation()` return type to `Response`. Replace body:

```java
@GET
@Path("/{caseId}")
public Response getInvestigation(@PathParam("caseId") UUID caseId) {
    final Optional<InvestigationResolution> resolution =
            outcomeService.resolveInvestigation(caseId);
    if (resolution.isEmpty()) {
        return Response.status(Response.Status.NOT_FOUND).build();
    }
    final InvestigationResolution r = resolution.get();
    if (r.status() != InvestigationStatus.COMPLETED) {
        return Response.ok(new Layer6InvestigationResponse(
                caseId, r.status(), List.of(), null)).build();
    }
    final List<WorkerDecisionEntry> entries = workerDecisionRepo.findAllByCaseId(caseId);
    final List<WorkerRoutingDecision> decisions = entries.stream()
            .map(e -> {
                final OptionalDouble score =
                        trustScoreSource.capabilityScore(e.workerId, e.capabilityTag);
                return new WorkerRoutingDecision(
                        e.capabilityTag, e.workerId,
                        score.isPresent() ? score.getAsDouble() : null);
            })
            .toList();
    return Response.ok(new Layer6InvestigationResponse(
            caseId, r.status(), decisions, r.outcome())).build();
}
```

Add missing imports: `Optional`, `InvestigationResolution`, `InvestigationStatus`. Remove unused imports for `CaseInstance`, `CaseInstanceRepository`, `CaseInstanceCache`, `CaseStatus`, `TenancyConstants`.

- [ ] **Step 6: Simplify `AmlLayer9Resource`**

Remove `CaseInstanceCache` and `CaseInstanceRepository` injections. Replace `getInvestigation()` body:

```java
@GET
@Path("/{caseId}")
public Response getInvestigation(@PathParam("caseId") final UUID caseId) {
    final Optional<InvestigationResolution> resolution =
            outcomeService.resolveInvestigation(caseId);
    if (resolution.isEmpty()) {
        return Response.status(Response.Status.NOT_FOUND).build();
    }
    final InvestigationResolution r = resolution.get();
    return Response.ok(new Layer9InvestigationResponse(caseId, r.status(), r.outcome())).build();
}
```

Remove unused imports: `HashMap`, `Map`, `CaseInstance`, `CaseInstanceRepository`, `CaseInstanceCache`, `CaseStatus`, `TenancyConstants`, `InvestigationOutcome`.

- [ ] **Step 7: Run all tests**

Run: `mvn -pl app -am test -Dtest=AmlLayer6ResourceTest,AmlLayer9ResourceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all tests PASS — including the new 404 tests

- [ ] **Step 8: Commit**

```
fix(#74): return 404 for nonexistent investigations, simplify resources

Both Layer 6 and Layer 9 resources now delegate completion detection
to AmlInvestigationOutcomeService. Nonexistent caseIds return 404
instead of misleading 200 with "in-progress". Layer 9 uses typed
Layer9InvestigationResponse instead of HashMap.

Refs #74
```

---

### Task 4: Rejection reason capture (#75)

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/ledger/AmlSarOfficerReviewedLedgerEntry.java`
- Create: `app/src/main/resources/db/aml-ledger/migration/V2012__aml_sar_officer_reviewed_rejection_reason.sql`
- Modify: `app/src/main/java/io/casehub/aml/compliance/AmlWorkItemLifecycleObserver.java`
- Modify: `app/src/main/java/io/casehub/aml/ledger/AmlLedgerService.java`
- Modify: `app/src/test/java/io/casehub/aml/compliance/AmlWorkItemLifecycleObserverTest.java`
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationOutcomeService.java` (wire `e.rejectionReason` through)

**Interfaces:**
- Consumes: `WorkItemLifecycleEvent.detail()` (casehub-work), `InvestigationOutcome.fromReviewDecision(String, String)` (from Task 1)
- Produces: `AmlSarOfficerReviewedLedgerEntry.rejectionReason` field; `AmlLedgerService.writeSarOfficerReviewed(UUID, String, String, String)`

- [ ] **Step 1: Write observer test for rejection reason capture**

Add overloaded event helper and rejection reason test to `AmlWorkItemLifecycleObserverTest`:

```java
private WorkItemLifecycleEvent event(WorkItemStatus status, String callerRef,
        String actor, String detail) {
    WorkItem wi = new WorkItem();
    wi.id = UUID.randomUUID();
    wi.status = status;
    wi.callerRef = callerRef;
    return WorkItemLifecycleEvent.of(status.name(), wi, actor, detail);
}
```

Update existing 3-arg helper to delegate:

```java
private WorkItemLifecycleEvent event(WorkItemStatus status, String callerRef, String actor) {
    return event(status, callerRef, actor, null);
}
```

Add test:

```java
@Test
void rejected_captures_rejection_reason_from_event_detail() {
    observer.onWorkItemLifecycle(event(WorkItemStatus.REJECTED,
            "aml:investigation:" + caseId, "compliance-officer-001",
            "Insufficient evidence for SAR filing"));

    verify(ledgerService).writeSarOfficerReviewed(eq(caseId),
            eq("compliance-officer-001"), eq("REJECTED"),
            eq("Insufficient evidence for SAR filing"));
}
```

- [ ] **Step 2: Update existing observer test verify calls for 4-arg signature**

All existing `verify(ledgerService).writeSarOfficerReviewed(...)` calls change from 3-arg to 4-arg. Add `eq(null)` as the fourth argument to every existing verify call that currently uses the 3-arg form. Specifically:

- `completed_validCallerRef_writesApproved`: verify with `eq("APPROVED"), eq(null)`
- `rejected_validCallerRef_writesRejected`: verify with `eq("REJECTED"), eq(null)`
- `nullActor_fallsBackToUnknown`: verify with `eq("APPROVED"), eq(null)`
- `ledgerWriteFails_writesFailureEntry`: verify on `writeSarOfficerReviewed` with 4 args, and `writeSarOfficerReviewedFailure` with 4 args
- `inProgress_noWrite` and `differentDomain_noWrite`: update `never()` verify to use `any(), any(), any(), any()`

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn -pl app -am test -Dtest=AmlWorkItemLifecycleObserverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `writeSarOfficerReviewed` takes 3 args not 4

- [ ] **Step 4: Add `rejectionReason` field to `AmlSarOfficerReviewedLedgerEntry`**

```java
@Column(name = "rejection_reason", length = 1000)
public String rejectionReason;
```

Update `domainContentBytes()` with backward-compatible logic:

```java
@Override
protected byte[] domainContentBytes() {
    if (rejectionReason == null) {
        return (reviewDecision != null ? reviewDecision : "")
                .getBytes(java.nio.charset.StandardCharsets.UTF_8);
    }
    return String.join("|",
            reviewDecision != null ? reviewDecision : "",
            rejectionReason).getBytes(java.nio.charset.StandardCharsets.UTF_8);
}
```

- [ ] **Step 5: Create Flyway V2012 migration**

File: `app/src/main/resources/db/aml-ledger/migration/V2012__aml_sar_officer_reviewed_rejection_reason.sql`

```sql
-- Add rejection reason column to SAR officer reviewed ledger entry.
-- Nullable — existing entries predate rejection reason capture.
ALTER TABLE aml_sar_officer_reviewed_ledger_entry
    ADD COLUMN rejection_reason VARCHAR(1000);
```

- [ ] **Step 6: Update `AmlLedgerService` — 4-arg write methods**

Change `writeSarOfficerReviewed` signature:

```java
@Transactional(TxType.REQUIRED)
public void writeSarOfficerReviewed(final UUID caseId, final String officerId,
        final String reviewDecision, final String rejectionReason) {
```

Add `entry.rejectionReason = rejectionReason;` before `repository.save(entry, ...)`.

Change `writeSarOfficerReviewedFailure` signature the same way:

```java
@Transactional(TxType.REQUIRES_NEW)
public void writeSarOfficerReviewedFailure(final UUID caseId, final String officerId,
        final String reviewDecision, final String rejectionReason) {
```

Add `entry.rejectionReason = rejectionReason;` before `repository.save(entry, ...)`.

Update `noOp()` and `stub(UUID)` overrides — both methods change from 3-arg to 4-arg:

```java
@Override public void writeSarOfficerReviewed(UUID caseId, String officerId, String decision, String rejectionReason) {}
@Override public void writeSarOfficerReviewedFailure(UUID caseId, String officerId, String reviewDecision, String rejectionReason) {}
```

- [ ] **Step 7: Update `AmlWorkItemLifecycleObserver` to capture rejection reason**

Change lines 72-78:

```java
final String officerId = event.actor() != null ? event.actor() : "unknown-officer";
final String reviewDecision = event.status() == WorkItemStatus.COMPLETED
        ? "APPROVED" : "REJECTED";
final String rejectionReason = event.status() == WorkItemStatus.REJECTED
        ? event.detail() : null;

boolean written = false;
try {
    ledgerService.writeSarOfficerReviewed(caseId, officerId, reviewDecision, rejectionReason);
    written = true;
} catch (Exception e) {
    LOG.warnf(e, "Failed to write SAR_OFFICER_REVIEWED for caseId=%s officer=%s",
            caseId, officerId);
    if (!written) {
        try {
            ledgerService.writeSarOfficerReviewedFailure(caseId, officerId, reviewDecision, rejectionReason);
        } catch (Exception inner) {
            LOG.errorf(inner,
                    "AUDIT GAP: SAR_OFFICER_REVIEWED failure entry also failed caseId=%s",
                    caseId);
        }
    }
}
```

- [ ] **Step 8: Wire `rejectionReason` through `resolveOutcome()` in `AmlInvestigationOutcomeService`**

In `resolveOutcome()`, change the `null` placeholder from Task 2 to the actual field:

```java
.map(e -> InvestigationOutcome.fromReviewDecision(e.reviewDecision, e.rejectionReason))
```

- [ ] **Step 9: Run all tests**

Run: `mvn -pl app -am test -Dtest=AmlWorkItemLifecycleObserverTest,AmlInvestigationOutcomeServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all tests PASS

- [ ] **Step 10: Commit**

```
feat(#75): capture rejection reason from officer review lifecycle event

Add rejectionReason column (V2012), capture event.detail() in observer,
thread through AmlLedgerService and AmlInvestigationOutcomeService to
InvestigationOutcome.reason.

Closes #75, Closes #73
```

---

### Task 5: Layer 9 integration tests (#77)

**Files:**
- Modify: `app/src/test/java/io/casehub/aml/engine/AmlLayer9ResourceTest.java`

**Interfaces:**
- Consumes: `GET /api/layer9/investigations/{caseId}` (from Task 3); `WorkItemService.completeFromSystem(UUID, String, String)`, `WorkItemService.rejectFromSystem(UUID, String, String)` (casehub-work)
- Produces: integration test coverage for Layer 9 outcome assertions

- [ ] **Step 1: Add test infrastructure to `AmlLayer9ResourceTest`**

Add imports and injections:

```java
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.service.WorkItemService;
import io.quarkus.narayana.jta.QuarkusTransaction;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import java.util.List;

// Add field injections:
@PersistenceContext
EntityManager defaultEm;

@Inject
WorkItemService workItemService;
```

Add helper methods:

```java
private static SuspiciousTransaction pepTransaction(final String id) {
    return new SuspiciousTransaction(id, "ACC-PEP-A", "ACC-PEP-B",
            new java.math.BigDecimal("200000"), "USD",
            java.time.Instant.parse("2024-12-01T00:00:00Z"),
            "PEP -- high risk");
}

private WorkItem findComplianceReviewWorkItem(final UUID caseId) {
    return QuarkusTransaction.requiringNew().call(() ->
        defaultEm.createQuery(
            "SELECT w FROM WorkItem w WHERE w.callerRef = :ref",
            WorkItem.class)
            .setParameter("ref", "aml:investigation:" + caseId)
            .getSingleResult());
}

private List<WorkItem> findGateWorkItems(final UUID caseId) {
    return QuarkusTransaction.requiringNew().call(() ->
        defaultEm.createQuery(
            "SELECT w FROM WorkItem w WHERE w.callerRef LIKE :pattern",
            WorkItem.class)
            .setParameter("pattern", "case:" + caseId + "/gate:%")
            .getResultList());
}

private void awaitAndApproveGate(final UUID caseId) {
    Awaitility.await()
            .atMost(15, TimeUnit.SECONDS)
            .pollInterval(300, TimeUnit.MILLISECONDS)
            .until(() -> !findGateWorkItems(caseId).isEmpty());
    final WorkItem gate = findGateWorkItems(caseId).get(0);
    workItemService.completeFromSystem(gate.id, "test-mlro", "approved");
}
```

- [ ] **Step 2: Write outcome integration tests**

```java
@Test
void officer_approval_surfaces_sar_filed_outcome() {
    final String caseIdStr = given().contentType(ContentType.JSON)
            .body(pepTransaction("TXN-L9-OUTCOME-" + UUID.randomUUID()))
            .when().post("/api/layer9/investigations")
            .then().statusCode(202)
            .extract().path("caseId");

    final UUID caseId = UUID.fromString(caseIdStr);
    awaitAndApproveGate(caseId);

    Awaitility.await().atMost(30, TimeUnit.SECONDS).pollInterval(500, TimeUnit.MILLISECONDS)
            .until(() -> "completed".equals(
                    given().when().get("/api/layer9/investigations/" + caseIdStr)
                            .then().extract().path("status")));

    final WorkItem review = findComplianceReviewWorkItem(caseId);
    workItemService.completeFromSystem(review.id, "test-compliance-officer", "SAR approved");

    Awaitility.await().atMost(10, TimeUnit.SECONDS).pollInterval(300, TimeUnit.MILLISECONDS)
            .until(() -> "sar-filed".equals(
                    given().when().get("/api/layer9/investigations/" + caseIdStr)
                            .then().extract().path("outcome.type")));
}

@Test
void officer_rejection_surfaces_gate_rejected_outcome_with_reason() {
    final String caseIdStr = given().contentType(ContentType.JSON)
            .body(pepTransaction("TXN-L9-REJECT-" + UUID.randomUUID()))
            .when().post("/api/layer9/investigations")
            .then().statusCode(202)
            .extract().path("caseId");

    final UUID caseId = UUID.fromString(caseIdStr);
    awaitAndApproveGate(caseId);

    Awaitility.await().atMost(30, TimeUnit.SECONDS).pollInterval(500, TimeUnit.MILLISECONDS)
            .until(() -> "completed".equals(
                    given().when().get("/api/layer9/investigations/" + caseIdStr)
                            .then().extract().path("status")));

    final WorkItem review = findComplianceReviewWorkItem(caseId);
    workItemService.rejectFromSystem(review.id, "test-compliance-officer", "Insufficient evidence");

    Awaitility.await().atMost(10, TimeUnit.SECONDS).pollInterval(300, TimeUnit.MILLISECONDS)
            .until(() -> "gate-rejected".equals(
                    given().when().get("/api/layer9/investigations/" + caseIdStr)
                            .then().extract().path("outcome.type")));
}
```

- [ ] **Step 3: Run Layer 9 integration tests**

Run: `mvn -pl app -am test -Dtest=AmlLayer9ResourceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all tests PASS (including existing tests + new 404 test from Task 3 + 2 new outcome tests)

- [ ] **Step 4: Run full test suite**

Run: `mvn -pl app -am test -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```
test(#77): add Layer 9 outcome integration tests and complete test gaps

Layer 9 approval/rejection surfaces outcome.type and outcome.reason.
sequenceNumber tiebreaker and failure-only scenario covered.

Closes #77
```

---

## Verification

After all 5 tasks are complete, run the full build:

```bash
mvn -pl api,app -am verify -Dsurefire.failIfNoSpecifiedTests=false
```

All tests must pass. The final state closes #74, #75 (and duplicate #73), and #77.
