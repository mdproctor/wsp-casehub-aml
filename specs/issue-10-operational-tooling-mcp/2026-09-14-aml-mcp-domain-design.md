# AML MCP Domain Registration

**Issue:** casehubio/aml#10
**Date:** 2026-09-14
**Scale:** M | **Complexity:** Med

## Context

Epic #10 covers operational tooling for AML — MCP tools, OTel, and PROV-DM export.
PROV-DM is done (#126). OTel is a platform concern (parent#473). What remains is
registering AML's operational queries in the platform MCP domain catalog.

The platform's agreed cross-platform API strategy (platform#295) uses JAX-RS SPI
interfaces as the single source of truth. One annotated interface generates REST +
GraphQL + MCP. `@McpDomain` provides hierarchical, on-demand MCP discovery.
`@PlatformQuery`/`@PlatformMutation` are deprecated — HTTP verb determines query
vs mutation.

AML's investigation lifecycle is engine-orchestrated with comprehensive existing
services. This work wraps those services in SPI interfaces so they're discoverable
in the platform domain catalog.

## SPI Interfaces

Three interfaces in `api/`, grouped by sub-domain:

### AmlInvestigationApi

```java
@McpDomain("aml/investigations")
@Path("/api/aml/investigations")
public interface AmlInvestigationApi {

    @GET
    @Path("/{caseId}")
    @Description("Investigation status — outcome, specialist findings, gate decisions")
    InvestigationDetail getInvestigation(@PathParam("caseId") UUID caseId);

    @GET
    @Path("/stalled")
    @Description("Investigations with workers past stall threshold")
    List<StalledInvestigation> listStalled();
}
```

### AmlComplianceApi

```java
@McpDomain("aml/compliance")
@Path("/api/aml/compliance")
public interface AmlComplianceApi {

    @GET
    @Path("/sar-pipeline")
    @Description("SAR pipeline — pending reviews, queue depth, SLA health")
    SarPipelineStatus getSarPipeline();
}
```

### AmlAuditApi

```java
@McpDomain("aml/audit")
@Path("/api/aml/audit")
public interface AmlAuditApi {

    @GET
    @Path("/{caseId}")
    @Description("Full causal audit chain with Merkle verification")
    List<AuditTrailEntry> getAuditTrail(@PathParam("caseId") UUID caseId);
}
```

All operations are `@GET` — query-only. Generates `readOnly` MCP tool hints and
GraphQL `@Query` operations.

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

`casehub-platform-api` (already a dep of `api/`) provides `@McpDomain` and
`@Description`.

## Hierarchical Discovery

MCP agents navigate: `casehub_model()` → sees `aml` group → drills into
`aml/investigations`, `aml/compliance`, `aml/audit` → sees operations with
descriptions. `casehub_activate("aml/investigations")` promotes operations to
individual named tools.

## Relationship to Existing REST Endpoints

The new `/api/aml/...` paths coexist alongside existing endpoints:

| Existing | New SPI | Notes |
|----------|---------|-------|
| `GET /api/layer9/investigations/{id}` | `GET /api/aml/investigations/{id}` | SPI returns typed POJO vs Response |
| `GET /api/investigations/{id}/audit-trail` | `GET /api/aml/audit/{id}` | Same data, generated surface |
| (none) | `GET /api/aml/investigations/stalled` | New — wraps InvestigationStallDetector |
| (none) | `GET /api/aml/compliance/sar-pipeline` | New — aggregated compliance view |

No migration of existing endpoints in this branch. Future consolidation is a
separate concern tracked under platform#295.

## Testing

- **Unit test:** `AmlInvestigationApiImpl` with mocked services — verify delegation
  and response assembly
- **Unit test:** `SarPipelineStatus` assembly — verify WorkItem query logic
- **`@QuarkusTest`:** Verify `ModelRegistry` contains `aml/investigations`,
  `aml/compliance`, `aml/audit` domains after startup
- **`@QuarkusTest`:** Verify `casehub_action` dispatch for each operation returns
  data (via MCP tool manager or REST)

## What This Does NOT Change

- Existing REST endpoints — unchanged, no migration
- Engine orchestration — unchanged, workers still dispatched via engine
- Workbench UI — unchanged, still uses existing REST endpoints
- OTel — platform concern (parent#473)
- PROV-DM — already done (#126)

## References

- `parent/docs/audit/api-generation-audit.md` — platform API strategy
- `casehubio/platform#295` — MCP/REST/GraphQL generator issue
- `casehub-platform/mcp-core/` — DomainModel, ModelRegistry, OperationDescriptor
- `casehub-platform/mcp/` — GraphQLModelScanner, DynamicToolRegistrar
- `app/src/main/java/io/casehub/aml/engine/AmlInvestigationOutcomeService.java`
- `app/src/main/java/io/casehub/aml/compliance/InvestigationStallDetector.java`
- `app/src/main/java/io/casehub/aml/engine/AmlAuditTrailResource.java`
- `app/src/main/java/io/casehub/aml/engine/AmlInvestigationQueryResource.java`
- `casehubio/parent#473` — OTel BOM + docs (filed from this session)
- `casehubio/aml#126` — W3C PROV-DM export (closed)
