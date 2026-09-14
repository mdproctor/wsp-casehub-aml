## D1: Registration pattern — platform @McpDomain, not hand-written MCP tools

**Choice:** Use the platform MCP pattern: `@McpDomain("aml")` interface in `api/` with `@PlatformQuery` annotations. CDI impl in `app/`. Platform auto-generates `casehub_model`, `casehub_action`, `casehub_activate` tools + MCP resources + GraphQL + REST.
**Alternatives:**
- Hand-written `@McpServer("aml")` MCP tools — duplicates platform framework, misses catalog, no REST/GraphQL
- REST-only (no MCP registration) — existing endpoints already serve, but invisible to platform domain catalog
**Rationale:** Platform pattern gives MCP + REST + GraphQL from one annotated interface with hierarchical discovery and on-demand activation. Near-zero cost, full catalog visibility.
**Trade-offs:** Depends on `casehub-platform-mcp` SNAPSHOT — AML is an early adopter.
**Sources:** `casehub-platform/mcp-core/` (DomainModel, ModelRegistry, OperationDescriptor), `casehub-platform/mcp/` (GraphQLModelScanner, DynamicToolRegistrar, CaseHubMcpTools), `platform/boundary-rules.md` §Named MCP server convention
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
