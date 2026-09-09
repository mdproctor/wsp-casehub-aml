# AML Workbench — Authentication and Role-Based Access

**Issue:** casehubio/aml#86
**Date:** 2026-09-09
**Scale:** M | **Complexity:** Med

## Context

The AML workbench is operational — compliance officers, MLROs, and ops staff
use it daily. Gate actions are already API-protected by WorkItem `candidateGroup`
matching. What's missing is:

1. REST endpoint authorization for non-gated consequential actions (GDPR erasure,
   suspend/resume)
2. UI-level role filtering so users only see actions they can perform
3. Formal documentation of AML role names in the platform role registry

The platform provides all infrastructure: `casehub-platform-oidc` for
`@RolesAllowed` enforcement, `casehub-pages` `withAccess()` for UI filtering,
and WorkItem `candidateGroups` for gate protection. casehub-life wired the
same pattern at Layer 8 (life#40) — AML follows the reference implementation.

## Role Model

Four groups form a two-tier access model:

| Group | Constant | Purpose |
|-------|----------|---------|
| `compliance-officers` | `AmlGroups.COMPLIANCE_OFFICERS` | SAR compliance review WorkItems |
| `aml-compliance` | `AmlGroups.AML_COMPLIANCE` | Gate approval (ACCOUNT_RESTRICTION, TRANSACTION_BLOCKING, ENTITY_LINK_CREATION), suspend/resume |
| `aml-mlro` | `AmlGroups.MLRO` | SAR_FILING gate (exclusive), suspend/resume |
| `aml-senior-compliance` | `AmlGroups.AML_SENIOR_COMPLIANCE` | LAW_ENFORCEMENT_REFERRAL gate, GDPR erasure |

**Tier 1 — Gate-protected actions:** Already enforced by WorkItem `candidateGroup`
matching. No changes needed. The groups are set in `AmlActionType.candidateGroups()`
and `ComplianceReviewLifecycle.openReview()`.

**Tier 2 — Non-gated consequential actions:** Need `@RolesAllowed` on REST
endpoints and `withAccess` in the UI.

## Backend Changes

### 1. Maven Dependency

Add `casehub-platform-oidc` as a compile dependency to `app/pom.xml`:

```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-platform-oidc</artifactId>
</dependency>
```

This activates `SecurityIdentityAugmentor`, which bridges
`GroupMembershipProvider.groupsOf()` to `SecurityIdentity.getRoles()`.
`@RolesAllowed` annotations become enforceable.

**Dev/demo profile:** Add `quarkus.oidc.enabled=false` to the dev profile
properties (or `%dev.quarkus.oidc.enabled=false` in `application.properties`).
This prevents the OIDC extension from failing at startup when no OIDC
server is configured. With OIDC disabled, `@RolesAllowed` annotations are
inert — all requests pass through. This matches casehub-life's approach
(life#40).

### 2. AmlGroups — Add COMPLIANCE_OFFICERS

Add the missing constant to `io.casehub.aml.domain.AmlGroups`:

```java
public static final String COMPLIANCE_OFFICERS = "compliance-officers";
```

Replace the string literal `"compliance-officers"` in
`ComplianceReviewLifecycle.java:69` with `AmlGroups.COMPLIANCE_OFFICERS`.

Replace the string literal `"senior-compliance-officers"` in
`ComplianceEscalationService.java:35` — this appears to use a non-standard
group name. Verify whether this should be `AmlGroups.AML_SENIOR_COMPLIANCE`
(`"aml-senior-compliance"`) instead.

### 3. @RolesAllowed Annotations

**Class-level** (single-purpose resources):

| Resource | Path | Role |
|----------|------|------|
| `AmlGdprErasureResource` | `POST /api/actors/{actorId}/erasure` | `aml-senior-compliance` |
| `AmlEntityErasureResource` | `POST /api/entities/{entityId}/erasure` | `aml-senior-compliance` |
| `AmlCrossTenantErasureResource` | `POST /api/entities/{entityId}/erasure/cross-tenant` | `aml-senior-compliance` |

**Method-level** (mixed read/write resources):

| Resource | Method | Path | Role |
|----------|--------|------|------|
| `AmlLayer9Resource` | `suspendInvestigation` | `POST /{caseId}/suspend` | `aml-compliance`, `aml-mlro` |
| `AmlLayer9Resource` | `resumeInvestigation` | `POST /{caseId}/resume` | `aml-compliance`, `aml-mlro` |

GET endpoints on `AmlLayer9Resource` remain unannotated — open to all
authenticated users.

All other resources (`AmlInvestigationResource`, `AmlInvestigationQueryResource`,
`AmlLayer5Resource`, `AmlLayer6Resource`, `AmlLayer7Resource` compliance evidence
GET, `AmlAuditTrailResource`, `AmlMetricsResource`, `AmlCbrResource`,
`AmlProvenanceResource`, `AmlWorkerTaskResource`) are read-only or
agent-facing and remain unannotated.

`AmlSimulationResource` is dev/demo only — no RBAC annotation.

## Frontend Changes

### 4. Dock Split — Compliance Evidence vs GDPR Erasure

Split `aml-compliance-dock` into two components:

**`aml-compliance-dock`** (modified) — Remove the GDPR erasure section.
Retains regulatory compliance evidence only:

```html
<div class="section-label">Regulatory Compliance</div>
<blocks-compliance-summary
  endpoint="/api/investigations/${this._caseId}/compliance-evidence">
</blocks-compliance-summary>
```

**`aml-gdpr-dock`** (new) — GDPR erasure action extracted from the
compliance dock:

```html
<div class="section-label">GDPR Erasure</div>
<blocks-gdpr-erasure-action
  endpoint="/api/actors"
  subject-label="Actor">
</blocks-gdpr-erasure-action>
```

This component does NOT need case context — GDPR erasure targets actors,
not investigations.

### 5. Layout — withAccess Wiring

Update `layout.ts` to reflect the dock split and add role gating:

```typescript
import { dockWorkbench, hostPanel, withAccess } from '@casehubio/pages-ui/dist/dsl/builders.js';

export const workbench = dockWorkbench({
  storageKey: 'aml-workbench',
  centre: hostPanel('aml-centre'),
  left: [
    { key: 'investigations', label: 'Investigations', icon: 'search',
      defaultOpen: true, content: hostPanel('aml-investigation-nav') },
    { key: 'worker-tasks', label: 'My Tasks', icon: 'assignment',
      content: hostPanel('aml-worker-nav') },
    { key: 'work-queue', label: 'Work Queue', icon: 'inbox',
      content: hostPanel('aml-work-queue-nav') },
  ],
  right: [
    { key: 'findings', label: 'Findings', icon: 'biotech',
      content: hostPanel('aml-findings-dock') },
    { key: 'compliance', label: 'Compliance', icon: 'verified',
      content: hostPanel('aml-compliance-dock') },
    { key: 'gdpr', label: 'GDPR', icon: 'delete_sweep',
      content: withAccess({ roles: ['aml-senior-compliance'] },
        hostPanel('aml-gdpr-dock')) },
    { key: 'audit', label: 'Audit', icon: 'history',
      defaultOpen: true, content: hostPanel('aml-audit-dock') },
    { key: 'routing', label: 'Routing', icon: 'route',
      content: hostPanel('aml-routing-dock') },
  ],
  bottom: [
    { key: 'operations', label: 'Operations', icon: 'monitoring',
      content: hostPanel('aml-operations-dock') },
    { key: 'scenarios', label: 'Scenarios', icon: 'play_circle',
      content: hostPanel('aml-scenario-dock') },
  ],
});
```

When OIDC is not configured (dev/demo), `withAccess` is a no-op — all
panels are visible. When OIDC is active, the pages runtime checks the
user's roles from the auth context and hides panels where the user lacks
the required role.

## Platform Documentation

### 6. Role Registry Update

Add AML roles to the "Known roles" table in
`../parent/docs/platform/auth.md`:

| Role name | Harness | What it gates |
|-----------|---------|---------------|
| `compliance-officers` | `casehub-aml` | SAR compliance review WorkItems |
| `aml-compliance` | `casehub-aml` | Gate approval (ACCOUNT_RESTRICTION, TRANSACTION_BLOCKING, ENTITY_LINK_CREATION), suspend/resume |
| `aml-mlro` | `casehub-aml` | SAR_FILING gate (exclusive), suspend/resume |
| `aml-senior-compliance` | `casehub-aml` | LAW_ENFORCEMENT_REFERRAL gate, GDPR erasure |

## Testing

### Backend Tests

**Unit test — AmlGroups:**
- Verify `COMPLIANCE_OFFICERS` constant exists and equals `"compliance-officers"`

**@QuarkusTest — Erasure authorization:**
- Caller with `aml-senior-compliance` group → 200 on erasure endpoints
- Caller without `aml-senior-compliance` group → 403 on erasure endpoints
- Caller with `aml-compliance` only → 403 on erasure endpoints

**@QuarkusTest — Suspend/resume authorization:**
- Caller with `aml-compliance` group → 200 on suspend/resume
- Caller with `aml-mlro` group → 200 on suspend/resume
- Caller without either group → 403 on suspend/resume

**Test principal setup:** Use `MockCurrentPrincipal` with explicit group
lists. The OIDC module is not needed for `@QuarkusTest` — `@RolesAllowed`
enforcement is wired by `SecurityIdentityAugmentor` from
`casehub-platform-oidc`. If the OIDC module requires OIDC config even in
test, exclude `OidcCurrentPrincipal` in test `application.properties` and
rely on `MockCurrentPrincipal @DefaultBean`.

### Frontend Tests

**Dock split test:**
- `aml-compliance-dock` renders compliance evidence, does NOT render GDPR section
- `aml-gdpr-dock` renders GDPR erasure action

**Layout test:**
- Verify `withAccess` is applied to the GDPR dock panel in the layout
- Verify the `access.roles` property is `['aml-senior-compliance']`

## What This Does NOT Change

- **Gate protection** — unchanged; WorkItem `candidateGroups` already enforced
- **Work queue filtering** — unchanged; filters by candidateGroup already
- **Simulation resource** — no RBAC; dev/demo only
- **Read-only endpoints** — no role restriction
- **Agent-facing endpoints** — no role restriction (agents use service-to-service trust)

## References

- `api/src/main/java/io/casehub/aml/domain/AmlGroups.java` — existing group constants
- `api/src/main/java/io/casehub/aml/domain/AmlActionType.java` — gate candidateGroups
- `app/src/main/java/io/casehub/aml/compliance/AmlLayer7Resource.java` — erasure endpoints
- `app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java` — suspend/resume endpoints
- `app/src/main/java/io/casehub/aml/ComplianceReviewLifecycle.java:69` — string literal to replace
- `app/src/main/java/io/casehub/aml/compliance/ComplianceEscalationService.java:35` — verify group name
- `app/src/main/webui/src/layout.ts` — workbench layout definition
- `app/src/main/webui/src/panels/compliance-dock.ts` — dock to split
- `../parent/docs/platform/auth.md` — platform role registry
- casehub-life consumer guide §Layer 8 — reference RBAC implementation (life#40)
- parent#251 — tracks RBAC adoption across harnesses
