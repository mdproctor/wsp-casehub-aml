# AML MCP Domain Registration

**Issue:** casehubio/aml#10
**Date:** 2026-09-14
**Scale:** M | **Complexity:** Med

## Context

Epic #10 covers operational tooling for AML — MCP tools, OTel, and PROV-DM export.
PROV-DM is done (#126). OTel is a platform concern (parent#473). What remains is
registering AML's operational queries in the platform MCP domain catalog.

The platform's API generation strategy (platform#295, closed) uses `@McpDomain` +
`@PlatformQuery`/`@PlatformMutation` SPI interfaces as the single source of truth.
The `GraphQLResolverProcessor` generates REST resources + GraphQL resolvers + MCP
tools from the interface. `@PathParam` is from `io.casehub.platform.api.mcp` (not
JAX-RS). `@RestMethod(HttpMethod.DELETE)` overrides the default verb convention.

AML's investigation lifecycle is engine-orchestrated with comprehensive existing
services. This work wraps those services in SPI interfaces so they're discoverable
in the platform domain catalog.

## SPI Interfaces

Three interfaces in `api/`, grouped by sub-domain:

### AmlInvestigationApi

```java
@McpDomain("aml/investigations")
public interface AmlInvestigationApi {

    @PlatformQuery("Investigation status — outcome, specialist findings, gate decisions")
    InvestigationDetail getInvestigation(@PathParam("caseId") UUID caseId);

    @PlatformQuery("Investigations with workers past stall threshold")
    List<StalledInvestigation> listStalled();
}
```

### AmlComplianceApi

```java
@McpDomain("aml/compliance")
public interface AmlComplianceApi {

    @PlatformQuery("SAR pipeline — pending reviews, queue depth, SLA health")
    SarPipelineStatus getSarPipeline();
}
```

### AmlAuditApi

```java
@McpDomain("aml/audit")
public interface AmlAuditApi {

    @PlatformQuery("Full causal audit chain with Merkle verification")
    List<AuditTrailEntry> getAuditTrail(@PathParam("caseId") UUID caseId);
}
```

`@PlatformQuery` determines query semantics — generator produces `@GET` REST +
GraphQL `@Query` + MCP readOnly tool hint. `@PathParam` is from
`io.casehub.platform.api.mcp` (not JAX-RS). Typed POJO returns — no `Response`,
no `Map<String,Object>`.

**Generated output (per interface):**
- REST resource: `GET /api/aml/investigations/{caseId}`, `GET /api/aml/investigations/stalled`, etc.
- GraphQL resolver: `@Query` methods with typed returns
- MCP: domain entry in `casehub_model("aml/investigations")`, dispatchable via `casehub_action`

## Return Types

All in `api/` (JPA-free):

### InvestigationDetail

Composite view of a single investigation:

```java
public record InvestigationDetail(
    UUID caseId,
    String status,
    String outcome,
    List<SpecialistFinding> findings,
    List<GateDecision> gates,
    List<RoutingDecision> routing
) {}
```

Assembled from `AmlInvestigationOutcomeService.resolveInvestigation()` +
`AmlInvestigationQueryResource` findings/gates/routing methods.

### StalledInvestigation

Already exists: `InvestigationStallDetector.StalledInvestigation(UUID caseId,
String waitingForWorkId, Duration stalledFor)`. Re-export from `api/` or use
directly if the inner record is accessible.

### SarPipelineStatus

New record:

```java
public record SarPipelineStatus(
    int pendingReviews,
    int pendingGates,
    long averageReviewAgeMs,
    int slaBreaches
) {}
```

Built from WorkItem queries filtered by `scope = "casehubio/aml/oversight"`.

### AuditTrailEntry

Already exists as `AuditTrailEntryResponse` in `AmlAuditTrailResource`. Either
re-use or create a JPA-free equivalent in `api/`.

## Implementations

CDI beans in `app/` implementing the SPI interfaces:

### AmlInvestigationApiImpl

```java
@ApplicationScoped
public class AmlInvestigationApiImpl implements AmlInvestigationApi {

    @Inject AmlInvestigationOutcomeService outcomeService;
    @Inject AmlInvestigationQueryService queryService;
    @Inject InvestigationStallDetector stallDetector;

    @Override
    public InvestigationDetail getInvestigation(UUID caseId) {
        // Delegate to outcomeService + queryService, assemble InvestigationDetail
    }

    @Override
    public List<StalledInvestigation> listStalled() {
        return stallDetector.detectStalled();
    }
}
```

### AmlComplianceApiImpl

```java
@ApplicationScoped
public class AmlComplianceApiImpl implements AmlComplianceApi {

    @Inject WorkItemService workItemService;
    @Inject EntityManager em;

    @Override
    public SarPipelineStatus getSarPipeline() {
        // Query WorkItems by scope "casehubio/aml/oversight",
        // count pending reviews vs pending gates,
        // compute average age and SLA breach count
    }
}
```

### AmlAuditApiImpl

```java
@ApplicationScoped
public class AmlAuditApiImpl implements AmlAuditApi {

    @Inject AmlAuditTrailService auditTrailService;

    @Override
    public List<AuditTrailEntry> getAuditTrail(UUID caseId) {
        return auditTrailService.buildAuditTrail(caseId);
    }
}
```

## Dependencies

### app/pom.xml

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-mcp</artifactId>
  <version>${casehub.version}</version>
</dependency>
```

`casehub-platform-api` (already a dep of `api/`) provides `@McpDomain`,
`@PlatformQuery`, `@PathParam`.

## Hierarchical Discovery

MCP agents navigate: `casehub_model()` → sees `aml` group → drills into
`aml/investigations`, `aml/compliance`, `aml/audit` → sees operations with
descriptions. `casehub_activate("aml/investigations")` promotes operations to
individual named tools.

## Relationship to Existing REST Endpoints

The generated `/api/aml/...` paths coexist alongside existing endpoints:

| Existing | Generated from SPI | Notes |
|----------|-------------------|-------|
| `GET /api/layer9/investigations/{id}` | `GET /api/aml/investigations/{id}` | SPI returns typed POJO vs Response |
| `GET /api/investigations/{id}/audit-trail` | `GET /api/aml/audit/{id}` | Same data, generated surface |
| (none) | `GET /api/aml/investigations/stalled` | New — wraps InvestigationStallDetector |
| (none) | `GET /api/aml/compliance/sar-pipeline` | New — aggregated compliance view |

No migration of existing endpoints in this branch. Future consolidation is a
separate concern.

## Testing

- **Unit test:** `AmlInvestigationApiImpl` with mocked services — verify delegation
  and response assembly
- **Unit test:** `SarPipelineStatus` assembly — verify WorkItem query logic
- **`@QuarkusTest`:** Verify `ModelRegistry` contains `aml/investigations`,
  `aml/compliance`, `aml/audit` domains after startup
- **`@QuarkusTest`:** Verify generated REST endpoints return data
- **`@QuarkusTest`:** Verify `casehub_action` dispatch for each operation

## What This Does NOT Change

- Existing REST endpoints — unchanged, no migration
- Engine orchestration — unchanged, workers still dispatched via engine
- Workbench UI — unchanged, still uses existing REST endpoints
- OTel — platform concern (parent#473)
- PROV-DM — already done (#126)

## References

- `parent/docs/audit/api-generation-audit.md` — platform API strategy
- `casehubio/platform#295` — unified API generation (closed)
- `casehub-platform/mcp-core/` — DomainModel, ModelRegistry, OperationDescriptor
- `casehub-platform/mcp/` — GraphQLModelScanner, DynamicToolRegistrar, GraphQLResolverProcessor
- `casehub-platform/platform-api/` — `@McpDomain`, `@PlatformQuery`, `@PathParam`, `@RestMethod`
- `app/src/main/java/io/casehub/aml/engine/AmlInvestigationOutcomeService.java`
- `app/src/main/java/io/casehub/aml/compliance/InvestigationStallDetector.java`
- `app/src/main/java/io/casehub/aml/engine/AmlAuditTrailResource.java`
- `app/src/main/java/io/casehub/aml/engine/AmlInvestigationQueryResource.java`
- `casehubio/parent#473` — OTel BOM + docs (filed from this session)
- `casehubio/aml#126` — W3C PROV-DM export (closed)
