# AML MCP Domain Registration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #10 — Epic 10: Operational tooling — MCP tools and observability
**Issue group:** #10

**Goal:** Register AML's operational queries in the platform MCP domain catalog via `@McpDomain` SPI interfaces, making them discoverable through `casehub_model`, executable via `casehub_action`, and activatable as individual tools via `casehub_activate`.

**Architecture:** Three SPI interfaces in `api/` annotated with `@McpDomain("aml/<sub-domain>")` + `@PlatformQuery`. CDI implementations in `app/` wrap existing services. The platform `GraphQLResolverProcessor` auto-generates REST + GraphQL + MCP from each interface. `@PathParam` is from `io.casehub.platform.api.mcp`.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-platform-mcp, `@McpDomain` + `@PlatformQuery` + `@PathParam` annotations

## Global Constraints

- SPI interfaces in `api/` — JPA-free, no framework deps beyond `casehub-platform-api`
- Implementations in `app/` — CDI `@ApplicationScoped`, inject existing services
- Return typed POJOs — no `Response`, no `Map<String,Object>`
- `@PlatformQuery` description is the text the LLM sees — make it specific
- `@McpDomain` uses hierarchical `aml/<sub-domain>` naming
- `@PathParam` is from `io.casehub.platform.api.mcp` (not `jakarta.ws.rs`)
- All edits to `.java` files use IntelliJ MCP (`ide_create_file`, `ide_insert_member`, `ide_edit_member`)

---

## Batch 1: SPI Interfaces + Response Types (api module)

### Task 1: Add `casehub-platform-mcp` dependency + create SPI interfaces and response types

**Files:**
- Modify: `app/pom.xml` — add `casehub-platform-mcp` compile dep
- Create: `api/src/main/java/io/casehub/aml/api/mcp/AmlInvestigationApi.java`
- Create: `api/src/main/java/io/casehub/aml/api/mcp/AmlComplianceApi.java`
- Create: `api/src/main/java/io/casehub/aml/api/mcp/AmlAuditApi.java`
- Create: `api/src/main/java/io/casehub/aml/api/mcp/InvestigationDetail.java`
- Create: `api/src/main/java/io/casehub/aml/api/mcp/SarPipelineStatus.java`
- Create: `api/src/main/java/io/casehub/aml/api/mcp/AuditTrailEntry.java`
- Test: `api/src/test/java/io/casehub/aml/api/mcp/AmlMcpApiTest.java`

**Interfaces:**
- Produces: `AmlInvestigationApi`, `AmlComplianceApi`, `AmlAuditApi` — consumed by Task 2 impls
- Produces: `InvestigationDetail`, `SarPipelineStatus`, `AuditTrailEntry` — return types used by impls

- [ ] **Step 1: Add `casehub-platform-mcp` to `app/pom.xml`**

In the CaseHub Foundation section, add after the `casehub-platform-oidc` entry:

```xml
    <!-- MCP domain registration — auto-generates REST + GraphQL + MCP from SPI interfaces -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-mcp</artifactId>
      <version>${casehub.version}</version>
    </dependency>
```

- [ ] **Step 2: Write test for SPI interface existence and annotations**

Create `api/src/test/java/io/casehub/aml/api/mcp/AmlMcpApiTest.java`:

```java
package io.casehub.aml.api.mcp;

import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PlatformQuery;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class AmlMcpApiTest {

    @Test
    void investigationApiHasMcpDomain() {
        McpDomain domain = AmlInvestigationApi.class.getAnnotation(McpDomain.class);
        assertNotNull(domain);
        assertEquals("aml/investigations", domain.value());
    }

    @Test
    void complianceApiHasMcpDomain() {
        McpDomain domain = AmlComplianceApi.class.getAnnotation(McpDomain.class);
        assertNotNull(domain);
        assertEquals("aml/compliance", domain.value());
    }

    @Test
    void auditApiHasMcpDomain() {
        McpDomain domain = AmlAuditApi.class.getAnnotation(McpDomain.class);
        assertNotNull(domain);
        assertEquals("aml/audit", domain.value());
    }

    @Test
    void investigationApiMethodsHavePlatformQuery() throws NoSuchMethodException {
        assertNotNull(AmlInvestigationApi.class.getMethod("getInvestigation", java.util.UUID.class)
                .getAnnotation(PlatformQuery.class));
        assertNotNull(AmlInvestigationApi.class.getMethod("listStalled")
                .getAnnotation(PlatformQuery.class));
    }

    @Test
    void complianceApiMethodsHavePlatformQuery() throws NoSuchMethodException {
        assertNotNull(AmlComplianceApi.class.getMethod("getSarPipeline")
                .getAnnotation(PlatformQuery.class));
    }

    @Test
    void auditApiMethodsHavePlatformQuery() throws NoSuchMethodException {
        assertNotNull(AmlAuditApi.class.getMethod("getAuditTrail", java.util.UUID.class)
                .getAnnotation(PlatformQuery.class));
    }
}
```

- [ ] **Step 3: Run test — verify it fails (classes don't exist)**

Run: `mvn test -pl api -am -Dtest=AmlMcpApiTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation error

- [ ] **Step 4: Create response types**

Use `ide_create_file` for each:

`api/src/main/java/io/casehub/aml/api/mcp/InvestigationDetail.java`:
```java
package io.casehub.aml.api.mcp;

import io.casehub.aml.api.model.InvestigationFindingsResponse;
import io.casehub.aml.api.model.InvestigationGatesResponse;
import io.casehub.aml.api.model.InvestigationRoutingResponse;

import java.util.UUID;

public record InvestigationDetail(
        UUID caseId,
        String status,
        String outcome,
        InvestigationFindingsResponse findings,
        InvestigationGatesResponse gates,
        InvestigationRoutingResponse routing
) {}
```

`api/src/main/java/io/casehub/aml/api/mcp/SarPipelineStatus.java`:
```java
package io.casehub.aml.api.mcp;

public record SarPipelineStatus(
        int pendingReviews,
        int pendingGates,
        long averageReviewAgeMs,
        int slaBreaches
) {}
```

`api/src/main/java/io/casehub/aml/api/mcp/AuditTrailEntry.java`:
```java
package io.casehub.aml.api.mcp;

import java.time.Instant;
import java.util.UUID;

public record AuditTrailEntry(
        UUID id,
        String dtype,
        String actorId,
        String actorType,
        Instant timestamp,
        long sequenceNumber,
        UUID causedByEntryId,
        String digest
) {}
```

- [ ] **Step 5: Create SPI interfaces**

`api/src/main/java/io/casehub/aml/api/mcp/AmlInvestigationApi.java`:
```java
package io.casehub.aml.api.mcp;

import io.casehub.aml.compliance.InvestigationStallDetector.StalledInvestigation;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformQuery;

import java.util.List;
import java.util.UUID;

@McpDomain("aml/investigations")
public interface AmlInvestigationApi {

    @PlatformQuery("Investigation status — outcome, specialist findings, gate decisions")
    InvestigationDetail getInvestigation(@PathParam("caseId") UUID caseId);

    @PlatformQuery("Investigations with workers past stall threshold")
    List<StalledInvestigation> listStalled();
}
```

Note: `StalledInvestigation` is a public inner record of `InvestigationStallDetector`. If the import doesn't resolve (inner record not accessible from `api/`), extract it to a standalone record in `api/src/main/java/io/casehub/aml/api/mcp/StalledInvestigation.java` instead:

```java
package io.casehub.aml.api.mcp;

import java.time.Duration;
import java.util.UUID;

public record StalledInvestigation(UUID caseId, String waitingForWorkId, Duration stalledFor) {}
```

`api/src/main/java/io/casehub/aml/api/mcp/AmlComplianceApi.java`:
```java
package io.casehub.aml.api.mcp;

import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PlatformQuery;

@McpDomain("aml/compliance")
public interface AmlComplianceApi {

    @PlatformQuery("SAR pipeline — pending reviews, queue depth, SLA health")
    SarPipelineStatus getSarPipeline();
}
```

`api/src/main/java/io/casehub/aml/api/mcp/AmlAuditApi.java`:
```java
package io.casehub.aml.api.mcp;

import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformQuery;

import java.util.List;
import java.util.UUID;

@McpDomain("aml/audit")
public interface AmlAuditApi {

    @PlatformQuery("Full causal audit chain with Merkle verification")
    List<AuditTrailEntry> getAuditTrail(@PathParam("caseId") UUID caseId);
}
```

- [ ] **Step 6: Run test — verify it passes**

Run: `mvn test -pl api -am -Dtest=AmlMcpApiTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 6 tests PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/aml/api/mcp/ \
       api/src/test/java/io/casehub/aml/api/mcp/ \
       app/pom.xml
git commit -m "feat(#10): AML MCP SPI interfaces + response types

Three @McpDomain interfaces (aml/investigations, aml/compliance, aml/audit)
with @PlatformQuery operations. Typed POJO response records in api/ module.
Platform generator produces REST + GraphQL + MCP from these interfaces.

Refs #10"
```

---

## Batch 2: CDI Implementations (app module)

### Task 2: Implement AmlInvestigationApiImpl

**Files:**
- Create: `app/src/main/java/io/casehub/aml/mcp/AmlInvestigationApiImpl.java`
- Test: `app/src/test/java/io/casehub/aml/mcp/AmlInvestigationApiImplTest.java`

**Interfaces:**
- Consumes: `AmlInvestigationApi` from Task 1
- Consumes: `InvestigationDetail`, `StalledInvestigation` from Task 1
- Consumes: `AmlInvestigationOutcomeService.resolveInvestigation(UUID)` — returns `Optional<InvestigationResolution>`
- Consumes: `InvestigationStallDetector.detectStalled()` — returns `List<StalledInvestigation>`
- Consumes: `AmlInvestigationFindingsService.getFindings(UUID)` — returns `InvestigationFindingsResponse`
- Consumes: `AmlInvestigationGatesService.getGates(UUID)` — returns `InvestigationGatesResponse`
- Consumes: `AmlInvestigationRoutingService.getRoutingDecisions(UUID)` — returns `InvestigationRoutingResponse`

- [ ] **Step 1: Write failing test**

Create `app/src/test/java/io/casehub/aml/mcp/AmlInvestigationApiImplTest.java`:

```java
package io.casehub.aml.mcp;

import io.casehub.aml.api.mcp.InvestigationDetail;
import io.casehub.aml.compliance.InvestigationStallDetector;
import io.casehub.aml.domain.InvestigationResolution;
import io.casehub.aml.domain.InvestigationStatus;
import io.casehub.aml.engine.AmlInvestigationFindingsService;
import io.casehub.aml.engine.AmlInvestigationGatesService;
import io.casehub.aml.engine.AmlInvestigationOutcomeService;
import io.casehub.aml.engine.AmlInvestigationRoutingService;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.List;
import java.util.Optional;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class AmlInvestigationApiImplTest {

    @Mock AmlInvestigationOutcomeService outcomeService;
    @Mock AmlInvestigationFindingsService findingsService;
    @Mock AmlInvestigationGatesService gatesService;
    @Mock AmlInvestigationRoutingService routingService;
    @Mock InvestigationStallDetector stallDetector;
    @InjectMocks AmlInvestigationApiImpl impl;

    @Test
    void getInvestigation_returnsDetail() {
        UUID caseId = UUID.randomUUID();
        when(outcomeService.resolveInvestigation(caseId))
                .thenReturn(Optional.of(new InvestigationResolution(
                        InvestigationStatus.COMPLETED, "SAR_FILED", null)));

        InvestigationDetail detail = impl.getInvestigation(caseId);

        assertNotNull(detail);
        assertEquals(caseId, detail.caseId());
        assertEquals("COMPLETED", detail.status());
    }

    @Test
    void getInvestigation_returnsNullWhenNotFound() {
        UUID caseId = UUID.randomUUID();
        when(outcomeService.resolveInvestigation(caseId)).thenReturn(Optional.empty());

        InvestigationDetail detail = impl.getInvestigation(caseId);

        assertNull(detail);
    }

    @Test
    void listStalled_delegatesToDetector() {
        when(stallDetector.detectStalled()).thenReturn(List.of());

        var result = impl.listStalled();

        assertNotNull(result);
        verify(stallDetector).detectStalled();
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn test -pl app -am -Dtest=AmlInvestigationApiImplTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation error — `AmlInvestigationApiImpl` not defined

- [ ] **Step 3: Implement `AmlInvestigationApiImpl`**

Create `app/src/main/java/io/casehub/aml/mcp/AmlInvestigationApiImpl.java`:

```java
package io.casehub.aml.mcp;

import io.casehub.aml.api.mcp.AmlInvestigationApi;
import io.casehub.aml.api.mcp.InvestigationDetail;
import io.casehub.aml.compliance.InvestigationStallDetector;
import io.casehub.aml.compliance.InvestigationStallDetector.StalledInvestigation;
import io.casehub.aml.domain.InvestigationResolution;
import io.casehub.aml.engine.AmlInvestigationFindingsService;
import io.casehub.aml.engine.AmlInvestigationGatesService;
import io.casehub.aml.engine.AmlInvestigationOutcomeService;
import io.casehub.aml.engine.AmlInvestigationRoutingService;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.List;
import java.util.UUID;

@ApplicationScoped
public class AmlInvestigationApiImpl implements AmlInvestigationApi {

    @Inject AmlInvestigationOutcomeService outcomeService;
    @Inject AmlInvestigationFindingsService findingsService;
    @Inject AmlInvestigationGatesService gatesService;
    @Inject AmlInvestigationRoutingService routingService;
    @Inject InvestigationStallDetector stallDetector;

    @Override
    public InvestigationDetail getInvestigation(UUID caseId) {
        return outcomeService.resolveInvestigation(caseId)
                .map(r -> new InvestigationDetail(
                        caseId,
                        r.status().name(),
                        r.outcome(),
                        findingsService.getFindings(caseId),
                        gatesService.getGates(caseId),
                        routingService.getRoutingDecisions(caseId)))
                .orElse(null);
    }

    @Override
    public List<StalledInvestigation> listStalled() {
        return stallDetector.detectStalled();
    }
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn test -pl app -am -Dtest=AmlInvestigationApiImplTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/aml/mcp/AmlInvestigationApiImpl.java \
       app/src/test/java/io/casehub/aml/mcp/AmlInvestigationApiImplTest.java
git commit -m "feat(#10): AmlInvestigationApiImpl — wraps outcome + stall services

Refs #10"
```

### Task 3: Implement AmlComplianceApiImpl + AmlAuditApiImpl

**Files:**
- Create: `app/src/main/java/io/casehub/aml/mcp/AmlComplianceApiImpl.java`
- Create: `app/src/main/java/io/casehub/aml/mcp/AmlAuditApiImpl.java`
- Test: `app/src/test/java/io/casehub/aml/mcp/AmlComplianceApiImplTest.java`
- Test: `app/src/test/java/io/casehub/aml/mcp/AmlAuditApiImplTest.java`

**Interfaces:**
- Consumes: `AmlComplianceApi`, `AmlAuditApi` from Task 1
- Consumes: `SarPipelineStatus`, `AuditTrailEntry` from Task 1
- Consumes: `WorkItemService` — query WorkItems by scope
- Consumes: `LedgerEntryRepository.findBySubjectId(UUID, String)` — returns ledger entries
- Consumes: `TenancyConstants.DEFAULT_TENANT_ID`

- [ ] **Step 1: Write failing tests**

Create `app/src/test/java/io/casehub/aml/mcp/AmlComplianceApiImplTest.java`:

```java
package io.casehub.aml.mcp;

import io.casehub.aml.api.mcp.SarPipelineStatus;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class AmlComplianceApiImplTest {

    @Test
    void sarPipelineStatus_isRecord() {
        SarPipelineStatus status = new SarPipelineStatus(3, 1, 86400000L, 0);
        assertEquals(3, status.pendingReviews());
        assertEquals(1, status.pendingGates());
        assertEquals(86400000L, status.averageReviewAgeMs());
        assertEquals(0, status.slaBreaches());
    }
}
```

Create `app/src/test/java/io/casehub/aml/mcp/AmlAuditApiImplTest.java`:

```java
package io.casehub.aml.mcp;

import io.casehub.aml.api.mcp.AuditTrailEntry;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class AmlAuditApiImplTest {

    @Test
    void auditTrailEntry_isRecord() {
        UUID id = UUID.randomUUID();
        UUID causedBy = UUID.randomUUID();
        AuditTrailEntry entry = new AuditTrailEntry(
                id, "AmlInvestigationLedgerEntry", "agent-1", "SYSTEM",
                Instant.now(), 1L, causedBy, "sha256:abc");
        assertEquals(id, entry.id());
        assertEquals("AmlInvestigationLedgerEntry", entry.dtype());
        assertEquals(causedBy, entry.causedByEntryId());
    }
}
```

- [ ] **Step 2: Implement `AmlComplianceApiImpl`**

Create `app/src/main/java/io/casehub/aml/mcp/AmlComplianceApiImpl.java`:

```java
package io.casehub.aml.mcp;

import io.casehub.aml.api.mcp.AmlComplianceApi;
import io.casehub.aml.api.mcp.SarPipelineStatus;
import io.casehub.work.api.WorkItemStatus;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import jakarta.transaction.Transactional;

import java.time.Duration;
import java.time.Instant;
import java.util.List;

@ApplicationScoped
public class AmlComplianceApiImpl implements AmlComplianceApi {

    @PersistenceContext
    EntityManager em;

    @Override
    @Transactional
    public SarPipelineStatus getSarPipeline() {
        @SuppressWarnings("unchecked")
        List<Object[]> rows = em.createQuery(
                "SELECT w.status, w.createdAt, w.expiresAt FROM WorkItemEntity w " +
                "WHERE w.scope = :scope AND w.status IN :statuses")
                .setParameter("scope", "casehubio/aml/oversight")
                .setParameter("statuses", List.of(WorkItemStatus.PENDING, WorkItemStatus.CLAIMED))
                .getResultList();

        int pendingReviews = 0;
        int pendingGates = 0;
        long totalAgeMs = 0;
        int slaBreaches = 0;
        Instant now = Instant.now();

        for (Object[] row : rows) {
            WorkItemStatus status = (WorkItemStatus) row[0];
            Instant createdAt = (Instant) row[1];
            Instant expiresAt = (Instant) row[2];

            if (status == WorkItemStatus.PENDING) {
                pendingReviews++;
            } else {
                pendingGates++;
            }
            totalAgeMs += Duration.between(createdAt, now).toMillis();
            if (expiresAt != null && now.isAfter(expiresAt)) {
                slaBreaches++;
            }
        }

        long avgAge = (pendingReviews + pendingGates) > 0
                ? totalAgeMs / (pendingReviews + pendingGates) : 0;
        return new SarPipelineStatus(pendingReviews, pendingGates, avgAge, slaBreaches);
    }
}
```

- [ ] **Step 3: Implement `AmlAuditApiImpl`**

Create `app/src/main/java/io/casehub/aml/mcp/AmlAuditApiImpl.java`:

```java
package io.casehub.aml.mcp;

import io.casehub.aml.api.mcp.AmlAuditApi;
import io.casehub.aml.api.mcp.AuditTrailEntry;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.platform.api.identity.TenancyConstants;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.List;
import java.util.UUID;

@ApplicationScoped
public class AmlAuditApiImpl implements AmlAuditApi {

    @Inject
    LedgerEntryRepository ledgerEntryRepository;

    @Override
    public List<AuditTrailEntry> getAuditTrail(UUID caseId) {
        return ledgerEntryRepository.findBySubjectId(caseId, TenancyConstants.DEFAULT_TENANT_ID)
                .stream()
                .map(e -> new AuditTrailEntry(
                        e.id,
                        e.getClass().getSimpleName(),
                        e.actorId,
                        e.actorType != null ? e.actorType.name() : null,
                        e.timestamp,
                        e.sequenceNumber,
                        e.causedByEntryId,
                        e.digest))
                .toList();
    }
}
```

- [ ] **Step 4: Run tests**

Run: `mvn test -pl app -am -Dtest=AmlComplianceApiImplTest,AmlAuditApiImplTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/aml/mcp/AmlComplianceApiImpl.java \
       app/src/main/java/io/casehub/aml/mcp/AmlAuditApiImpl.java \
       app/src/test/java/io/casehub/aml/mcp/AmlComplianceApiImplTest.java \
       app/src/test/java/io/casehub/aml/mcp/AmlAuditApiImplTest.java
git commit -m "feat(#10): AmlComplianceApiImpl + AmlAuditApiImpl

SAR pipeline status from WorkItem queries. Audit trail from ledger entries.

Refs #10"
```

---

## Batch 3: Integration Verification

### Task 4: @QuarkusTest — verify domain registration and generated endpoints

**Files:**
- Create: `app/src/test/java/io/casehub/aml/mcp/AmlMcpDomainRegistrationTest.java`

**Interfaces:**
- Consumes: All SPI interfaces + impls from Tasks 1-3
- Consumes: `ModelRegistry` from `casehub-platform-mcp` — domain catalog

- [ ] **Step 1: Write integration test**

Create `app/src/test/java/io/casehub/aml/mcp/AmlMcpDomainRegistrationTest.java`:

```java
package io.casehub.aml.mcp;

import io.casehub.platform.mcp.ModelRegistry;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class AmlMcpDomainRegistrationTest {

    @Inject
    ModelRegistry registry;

    @Test
    void amlInvestigationsDomainRegistered() {
        var domain = registry.getDomain("aml/investigations");
        assertTrue(domain.isPresent(), "aml/investigations domain should be registered");
        assertFalse(domain.get().operations().isEmpty(), "should have operations");
    }

    @Test
    void amlComplianceDomainRegistered() {
        var domain = registry.getDomain("aml/compliance");
        assertTrue(domain.isPresent(), "aml/compliance domain should be registered");
    }

    @Test
    void amlAuditDomainRegistered() {
        var domain = registry.getDomain("aml/audit");
        assertTrue(domain.isPresent(), "aml/audit domain should be registered");
    }

    @Test
    void investigationOperationsPresent() {
        var domain = registry.getDomain("aml/investigations").orElseThrow();
        assertTrue(domain.operations().stream().anyMatch(op -> op.name().equals("getInvestigation")));
        assertTrue(domain.operations().stream().anyMatch(op -> op.name().equals("listStalled")));
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `mvn test -pl app -am -Dtest=AmlMcpDomainRegistrationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS — domains discovered and registered by `GraphQLModelScanner`

If the test fails because the scanner doesn't pick up `@PlatformQuery` interfaces, check:
1. Is `casehub-platform-mcp` on the classpath?
2. Does the `GraphQLModelScanner` scan for `@McpDomain` on interfaces? (Confirmed in source review — it does, lines 88-112)
3. Does the AML `api/` module have a jandex index? If not, add `jandex-maven-plugin` to `api/pom.xml`

- [ ] **Step 3: Commit**

```bash
git add app/src/test/java/io/casehub/aml/mcp/AmlMcpDomainRegistrationTest.java
git commit -m "feat(#10): @QuarkusTest verifies MCP domain registration

Confirms aml/investigations, aml/compliance, aml/audit registered in
ModelRegistry with expected operations.

Closes #10"
```

## References

- [2026-09-14-aml-mcp-domain-design.md] — design spec this plan implements
- [casehubio/platform#295] — unified API generation (closed)
- [casehub-platform/mcp/GraphQLModelScanner.java:88-112] — interface scan for @PlatformQuery
- [app/src/main/java/io/casehub/aml/engine/AmlInvestigationOutcomeService.java:53] — `resolveInvestigation(UUID)`
- [app/src/main/java/io/casehub/aml/compliance/InvestigationStallDetector.java:27] — `detectStalled()`
- [app/src/main/java/io/casehub/aml/engine/AmlAuditTrailResource.java:52] — `getAuditTrail(UUID)`
- [app/src/main/java/io/casehub/aml/engine/AmlInvestigationQueryResource.java] — findings/gates/routing services
- [casehubio/parent#473] — OTel BOM + docs
- [GitHub #10] — focal issue
