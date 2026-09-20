## D1: Registration pattern — JAX-RS SPI + @McpDomain (platform#295)

**Choice:** Use the agreed cross-platform approach: SPI interface in `api/` annotated with `@McpDomain("aml/<sub-domain>")` + standard JAX-RS annotations (`@GET`, `@POST`, `@Path`, `@QueryParam`, `@PathParam`). `@Description` on every method. HTTP verb determines query vs mutation. CDI impl in `app/`. Generator produces REST + GraphQL + MCP from the single interface definition.
**Alternatives:**
- `@PlatformQuery`/`@PlatformMutation` — deprecated, do not use
- Hand-written `@McpServer("aml")` MCP tools — duplicates framework, misses catalog, no REST/GraphQL generation
- Hand-written REST resources (current AML pattern) — no MCP/GraphQL, not discoverable in domain catalog
**Rationale:** One SPI interface → three surfaces (REST + GraphQL + MCP). Hierarchical `@McpDomain` labels enable progressive MCP discovery. JAX-RS is already the standard shape language — no new annotation vocabulary needed.
**Trade-offs:** Depends on `casehub-platform-mcp` SNAPSHOT and generator (platform#295) — AML is an early adopter.
**Sources:** `parent/docs/audit/api-generation-audit.md`, `casehubio/platform#295`, `casehub-platform/mcp/` (GraphQLModelScanner, DynamicToolRegistrar)
**Exploration:** quick
**Status:** captured

## D2: API scope — all 4 operations in one pass

**Choice:** Define all 4 operations now: `investigationStatus`, `sarPipeline`, `investigationAudit`, `stalledInvestigators`. All query-only, all data sources exist.
**Alternatives:**
- Start with 2 (simplest first, extend later) — unnecessary when all data sources exist and operations are independent
**Rationale:** All operations are queries wrapping existing services. No unknowns, no new data models. Implementing all 4 is barely more work than implementing 2.
**Trade-offs:** None meaningful.
**Sources:** `AmlInvestigationOutcomeService`, `AmlAuditTrailResource`, `InvestigationStallDetector`, `ComplianceReviewLifecycle`
**Exploration:** quick
**Status:** captured
