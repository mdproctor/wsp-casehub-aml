# AML Auth & RBAC Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #86 — feat: AML workbench UI — authentication and role-based access
**Issue group:** #86

**Goal:** Wire platform RBAC infrastructure into the AML workbench — backend `@RolesAllowed` on consequential endpoints, UI role filtering via `withAccess()`, and platform role documentation.

**Architecture:** Follow casehub-life Layer 8 pattern. Add `casehub-platform-oidc` dependency to activate `@RolesAllowed`. Add `quarkus-test-security` for `@TestSecurity` in integration tests. Split the compliance dock into evidence (read-only) and GDPR (role-gated) panels. Wire `withAccess()` at the pages layout level.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-platform-oidc, quarkus-test-security, Lit (TypeScript), casehub-pages `withAccess()` DSL

## Global Constraints

- `casehub-platform-oidc` is compile scope — activates `SecurityIdentityAugmentor`
- `quarkus-test-security` is test scope — provides `@TestSecurity`
- OIDC disabled in dev profile: `%dev.quarkus.oidc.enabled=false`
- `@RolesAllowed` annotations are inert without OIDC config — dev/demo unaffected
- Role constants use `AmlGroups` — no string literals for group names
- All edits to `.java` files use IntelliJ MCP (`ide_edit_member`, `ide_insert_member`, `ide_replace_member`)
- All edits to `.ts` files use IntelliJ MCP or Edit tool (TypeScript)

---

## Batch 1: Backend RBAC Foundation

### Task 1: Add platform-oidc dependency + AmlGroups constant + fix group name bug

**Files:**
- Modify: `app/pom.xml` — add `casehub-platform-oidc` (compile) and `quarkus-test-security` (test)
- Modify: `app/src/main/resources/application.properties` — add `%dev.quarkus.oidc.enabled=false`
- Modify: `api/src/main/java/io/casehub/aml/domain/AmlGroups.java` — add `COMPLIANCE_OFFICERS`
- Modify: `api/src/test/java/io/casehub/aml/domain/AmlGroupsTest.java` — test new constant
- Modify: `app/src/main/java/io/casehub/aml/ComplianceReviewLifecycle.java:69` — replace string literal
- Modify: `app/src/main/java/io/casehub/aml/compliance/ComplianceEscalationService.java:35` — fix `"senior-compliance-officers"` → `AmlGroups.AML_SENIOR_COMPLIANCE`
- Test: `api/src/test/java/io/casehub/aml/domain/AmlGroupsTest.java`

**Interfaces:**
- Produces: `AmlGroups.COMPLIANCE_OFFICERS = "compliance-officers"` — used by Task 2 and 3

- [ ] **Step 1: Add Maven dependencies to `app/pom.xml`**

In the CaseHub Foundation section, add after the existing dependencies:

```xml
    <!-- Layer 10: RBAC enforcement (@RolesAllowed bridge) -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-platform-oidc</artifactId>
    </dependency>
```

In the test dependencies section, add:

```xml
    <!-- @TestSecurity — controls SecurityIdentity in @QuarkusTest without a real OIDC server -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-test-security</artifactId>
      <scope>test</scope>
    </dependency>
```

- [ ] **Step 2: Add dev profile OIDC disable to `application.properties`**

Add to `app/src/main/resources/application.properties`:

```properties
# RBAC — OIDC disabled in dev; @RolesAllowed annotations are inert
%dev.quarkus.oidc.enabled=false
```

- [ ] **Step 3: Write failing test for `COMPLIANCE_OFFICERS` constant**

Add to `AmlGroupsTest.java`:

```java
@Test
void complianceOfficersConstantExists() {
    assertEquals("compliance-officers", AmlGroups.COMPLIANCE_OFFICERS);
}
```

- [ ] **Step 4: Run test — verify it fails**

Run: `mvn test -pl api -am -Dtest=AmlGroupsTest#complianceOfficersConstantExists -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation error — `COMPLIANCE_OFFICERS` not defined

- [ ] **Step 5: Add `COMPLIANCE_OFFICERS` to `AmlGroups`**

Use `ide_insert_member` on `AmlGroups.java`. Add after the existing constants:

```java
public static final String COMPLIANCE_OFFICERS = "compliance-officers";
```

- [ ] **Step 6: Run test — verify it passes**

Run: `mvn test -pl api -am -Dtest=AmlGroupsTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all `AmlGroupsTest` tests pass

- [ ] **Step 7: Replace string literal in `ComplianceReviewLifecycle`**

Use `ide_edit_member` on `ComplianceReviewLifecycle.openReview`. Replace line 69:

```java
// Before:
.candidateGroups("compliance-officers")
// After:
.candidateGroups(AmlGroups.COMPLIANCE_OFFICERS)
```

Add import: `import io.casehub.aml.domain.AmlGroups;`

- [ ] **Step 8: Fix bug in `ComplianceEscalationService`**

The group name `"senior-compliance-officers"` at line 35 is not a defined AML group. Replace with the correct constant. Use `ide_edit_member` on `ComplianceEscalationService.escalateToSeniorCompliance`:

```java
// Before:
.candidateGroups("senior-compliance-officers")
// After:
.candidateGroups(AmlGroups.AML_SENIOR_COMPLIANCE)
```

Also fix the log message at line 28:
```java
// Before:
LOG.infof("Escalating expired compliance review for caseId=%s to senior-compliance-officers", caseId);
// After:
LOG.infof("Escalating expired compliance review for caseId=%s to %s", caseId, AmlGroups.AML_SENIOR_COMPLIANCE);
```

Add import: `import io.casehub.aml.domain.AmlGroups;`

- [ ] **Step 9: Verify compilation**

Run: `mvn compile -pl app -am`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```bash
git add api/src/main/java/io/casehub/aml/domain/AmlGroups.java \
       api/src/test/java/io/casehub/aml/domain/AmlGroupsTest.java \
       app/pom.xml \
       app/src/main/resources/application.properties \
       app/src/main/java/io/casehub/aml/ComplianceReviewLifecycle.java \
       app/src/main/java/io/casehub/aml/compliance/ComplianceEscalationService.java
git commit -m "feat(#86): add platform-oidc dep, COMPLIANCE_OFFICERS constant, fix escalation group name

Adds casehub-platform-oidc compile dep and quarkus-test-security test dep.
Adds AmlGroups.COMPLIANCE_OFFICERS constant and replaces string literals.
Fixes ComplianceEscalationService using non-existent 'senior-compliance-officers'
group — corrected to AmlGroups.AML_SENIOR_COMPLIANCE.

Refs #86"
```

### Task 2: @RolesAllowed on erasure endpoints

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/compliance/AmlLayer7Resource.java` — add `@RolesAllowed` to erasure resource classes
- Create: `app/src/test/java/io/casehub/aml/compliance/AmlErasureAuthorizationTest.java` — RBAC integration test
- Test: `app/src/test/java/io/casehub/aml/compliance/AmlErasureAuthorizationTest.java`

**Interfaces:**
- Consumes: `AmlGroups.AML_SENIOR_COMPLIANCE` from Task 1

- [ ] **Step 1: Write failing test — 403 without correct role**

Create `AmlErasureAuthorizationTest.java`:

```java
package io.casehub.aml.compliance;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;

@QuarkusTest
class AmlErasureAuthorizationTest {

    @Test
    @TestSecurity(user = "analyst-1", roles = "aml-compliance")
    void actorErasure_forbiddenWithoutSeniorCompliance() {
        RestAssured.given()
                .contentType("application/json")
                .post("/api/actors/actor-123/erasure")
                .then()
                .statusCode(403);
    }

    @Test
    @TestSecurity(user = "senior-officer-1", roles = "aml-senior-compliance")
    void actorErasure_allowedWithSeniorCompliance() {
        // 404 is expected — actor doesn't exist, but we passed the auth check
        RestAssured.given()
                .contentType("application/json")
                .post("/api/actors/actor-123/erasure")
                .then()
                .statusCode(404);
    }

    @Test
    @TestSecurity(user = "analyst-1", roles = "aml-compliance")
    void entityErasure_forbiddenWithoutSeniorCompliance() {
        RestAssured.given()
                .contentType("application/json")
                .post("/api/entities/entity-456/erasure")
                .then()
                .statusCode(403);
    }

    @Test
    @TestSecurity(user = "senior-officer-1", roles = "aml-senior-compliance")
    void entityErasure_allowedWithSeniorCompliance() {
        RestAssured.given()
                .contentType("application/json")
                .post("/api/entities/entity-456/erasure")
                .then()
                .statusCode(404);
    }

    @Test
    @TestSecurity(user = "analyst-1", roles = "aml-compliance")
    void crossTenantErasure_forbiddenWithoutSeniorCompliance() {
        RestAssured.given()
                .contentType("application/json")
                .body("{\"tenantIds\": [\"t1\"]}")
                .post("/api/entities/entity-789/erasure/cross-tenant")
                .then()
                .statusCode(403);
    }
}
```

- [ ] **Step 2: Run tests — verify they fail (no 403 yet)**

Run: `mvn test -pl app -am -Dtest=AmlErasureAuthorizationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: `actorErasure_forbiddenWithoutSeniorCompliance` FAILS — gets 404 instead of 403

- [ ] **Step 3: Add `@RolesAllowed` to erasure resources**

Use `ide_edit_member` on each resource class declaration in `AmlLayer7Resource.java`.

On `AmlGdprErasureResource` (line 47), add class-level annotation:
```java
@RolesAllowed("aml-senior-compliance")
```

On `AmlEntityErasureResource` (line 67), add class-level annotation:
```java
@RolesAllowed("aml-senior-compliance")
```

On `AmlCrossTenantErasureResource` (line 87), add class-level annotation:
```java
@RolesAllowed("aml-senior-compliance")
```

Add import: `import jakarta.annotation.security.RolesAllowed;`

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn test -pl app -am -Dtest=AmlErasureAuthorizationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 5 tests PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/aml/compliance/AmlLayer7Resource.java \
       app/src/test/java/io/casehub/aml/compliance/AmlErasureAuthorizationTest.java
git commit -m "feat(#86): @RolesAllowed on erasure endpoints — aml-senior-compliance

Class-level @RolesAllowed('aml-senior-compliance') on all three erasure
resources. Integration test verifies 403 without correct role.

Refs #86"
```

### Task 3: @RolesAllowed on suspend/resume endpoints

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java` — method-level `@RolesAllowed`
- Create: `app/src/test/java/io/casehub/aml/engine/AmlLayer9AuthorizationTest.java` — RBAC integration test
- Test: `app/src/test/java/io/casehub/aml/engine/AmlLayer9AuthorizationTest.java`

**Interfaces:**
- Consumes: `AmlGroups.AML_COMPLIANCE`, `AmlGroups.MLRO` from `AmlGroups`

- [ ] **Step 1: Write failing test — 403 without correct role**

Create `AmlLayer9AuthorizationTest.java`:

```java
package io.casehub.aml.engine;

import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.security.TestSecurity;
import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;

import java.util.UUID;

@QuarkusTest
class AmlLayer9AuthorizationTest {

    private static final String CASE_ID = UUID.randomUUID().toString();

    @Test
    @TestSecurity(user = "viewer-1", roles = "compliance-officers")
    void suspend_forbiddenWithoutComplianceRole() {
        RestAssured.given()
                .post("/api/layer9/investigations/" + CASE_ID + "/suspend")
                .then()
                .statusCode(403);
    }

    @Test
    @TestSecurity(user = "officer-1", roles = "aml-compliance")
    void suspend_allowedWithComplianceRole() {
        // 404 expected — case doesn't exist, but auth passed
        RestAssured.given()
                .post("/api/layer9/investigations/" + CASE_ID + "/suspend")
                .then()
                .statusCode(404);
    }

    @Test
    @TestSecurity(user = "mlro-1", roles = "aml-mlro")
    void suspend_allowedWithMlroRole() {
        RestAssured.given()
                .post("/api/layer9/investigations/" + CASE_ID + "/suspend")
                .then()
                .statusCode(404);
    }

    @Test
    @TestSecurity(user = "viewer-1", roles = "compliance-officers")
    void resume_forbiddenWithoutComplianceRole() {
        RestAssured.given()
                .post("/api/layer9/investigations/" + CASE_ID + "/resume")
                .then()
                .statusCode(403);
    }

    @Test
    @TestSecurity(user = "officer-1", roles = "aml-compliance")
    void resume_allowedWithComplianceRole() {
        RestAssured.given()
                .post("/api/layer9/investigations/" + CASE_ID + "/resume")
                .then()
                .statusCode(404);
    }

    @Test
    @TestSecurity(user = "viewer-1", roles = "compliance-officers")
    void getInvestigation_allowedWithAnyRole() {
        // GET should work with any role — read-only
        RestAssured.given()
                .get("/api/layer9/investigations/" + CASE_ID)
                .then()
                .statusCode(404); // not 403
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn test -pl app -am -Dtest=AmlLayer9AuthorizationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: `suspend_forbiddenWithoutComplianceRole` FAILS — gets 404 instead of 403

- [ ] **Step 3: Add method-level `@RolesAllowed` to `AmlLayer9Resource`**

Use `ide_edit_member` on `suspendInvestigation` method. Add annotation before `@POST`:

```java
@RolesAllowed({"aml-compliance", "aml-mlro"})
```

Use `ide_edit_member` on `resumeInvestigation` method. Add annotation before `@POST`:

```java
@RolesAllowed({"aml-compliance", "aml-mlro"})
```

Add import: `import jakarta.annotation.security.RolesAllowed;`

- [ ] **Step 4: Run tests — verify they pass**

Run: `mvn test -pl app -am -Dtest=AmlLayer9AuthorizationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 6 tests PASS

- [ ] **Step 5: Run full test suite to check for regressions**

Run: `mvn test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all tests PASS — `@RolesAllowed` is inert on unannotated methods, and `@TestSecurity` scoping means existing tests are unaffected

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java \
       app/src/test/java/io/casehub/aml/engine/AmlLayer9AuthorizationTest.java
git commit -m "feat(#86): @RolesAllowed on suspend/resume — aml-compliance, aml-mlro

Method-level annotations on suspendInvestigation and resumeInvestigation.
GET remains open. Integration test verifies 403 for unauthorized callers
and pass-through for compliance/MLRO roles.

Refs #86"
```

---

## Batch 2: Frontend Dock Split + Layout Wiring

### Task 4: Split compliance dock and wire `withAccess`

**Files:**
- Modify: `app/src/main/webui/src/panels/compliance-dock.ts` — remove GDPR section
- Create: `app/src/main/webui/src/panels/gdpr-dock.ts` — extracted GDPR panel
- Modify: `app/src/main/webui/src/layout.ts` — add GDPR dock with `withAccess`, import `withAccess`
- Modify: `app/src/main/webui/src/index.ts` — import and register `aml-gdpr-dock`
- Test: `app/src/main/webui/src/panels/dock-panels.test.ts` (if dock tests exist) or new test

**Interfaces:**
- Consumes: `withAccess` from `@casehubio/pages-ui/dist/dsl/builders.js`
- Consumes: `hostPanel` from `@casehubio/pages-ui/dist/dsl/builders.js`

- [ ] **Step 1: Create `gdpr-dock.ts`**

Create `app/src/main/webui/src/panels/gdpr-dock.ts`:

```typescript
import { LitElement, html, css } from 'lit';
import { customElement } from 'lit/decorators.js';
import '@casehubio/blocks-ui-gdpr-erasure-action';

@customElement('aml-gdpr-dock')
export class AmlGdprDock extends LitElement {
  static override styles = css`
    :host { display: block; height: 100%; overflow-y: auto; }
    .section-label {
      font-size: var(--pages-font-size-xs, 11px);
      font-weight: 600; text-transform: uppercase; letter-spacing: 0.5px;
      color: var(--pages-neutral-8, #404040);
      padding: var(--pages-space-3, 12px) var(--pages-space-4, 16px) var(--pages-space-2, 8px);
    }
  `;

  override render() {
    return html`
      <div class="section-label">GDPR Erasure</div>
      <blocks-gdpr-erasure-action
        endpoint="/api/actors"
        subject-label="Actor">
      </blocks-gdpr-erasure-action>
    `;
  }
}
```

- [ ] **Step 2: Remove GDPR section from `compliance-dock.ts`**

Edit `app/src/main/webui/src/panels/compliance-dock.ts`:

Remove the import:
```typescript
// Remove: import '@casehubio/blocks-ui-gdpr-erasure-action';
```

Remove from `render()` method — the GDPR section (last two elements):
```html
<!-- Remove: -->
<div class="section-label">GDPR Erasure</div>
<blocks-gdpr-erasure-action
  endpoint="/api/actors"
  subject-label="Actor">
</blocks-gdpr-erasure-action>
```

- [ ] **Step 3: Update `layout.ts` — add GDPR dock with `withAccess`**

Edit `app/src/main/webui/src/layout.ts`. Update import:

```typescript
import { dockWorkbench, hostPanel, withAccess } from '@casehubio/pages-ui/dist/dsl/builders.js';
```

Add the GDPR dock to the `right` array, between compliance and audit:

```typescript
    { key: 'gdpr', label: 'GDPR', icon: 'delete_sweep',
      content: withAccess({ roles: ['aml-senior-compliance'] },
        hostPanel('aml-gdpr-dock')) },
```

- [ ] **Step 4: Update `index.ts` — import and register**

Add import at the top with other panel imports:
```typescript
import './panels/gdpr-dock.js';
```

Add registration after the existing `registerPanel` calls:
```typescript
registerPanel('aml-gdpr-dock', 'aml-gdpr-dock');
```

- [ ] **Step 5: Verify TypeScript compilation**

Run: `cd app/src/main/webui && npx tsc --noEmit`
Expected: no errors

- [ ] **Step 6: Run frontend tests**

Run: `cd app/src/main/webui && npx vitest run`
Expected: all tests pass

- [ ] **Step 7: Commit**

```bash
git add app/src/main/webui/src/panels/compliance-dock.ts \
       app/src/main/webui/src/panels/gdpr-dock.ts \
       app/src/main/webui/src/layout.ts \
       app/src/main/webui/src/index.ts
git commit -m "feat(#86): split compliance dock, wire withAccess on GDPR panel

Extract GDPR erasure action from aml-compliance-dock into new aml-gdpr-dock.
Wrap with withAccess({ roles: ['aml-senior-compliance'] }) in layout.
Compliance evidence panel remains visible to all authenticated users.

Refs #86"
```

---

## Batch 3: Platform Documentation

### Task 5: Document AML roles in platform auth.md

**Files:**
- Modify: `../parent/docs/platform/auth.md` — add AML roles to Known roles table

**Interfaces:**
- None (documentation only)

- [ ] **Step 1: Add AML roles to the Known roles table**

Edit `/Users/mdproctor/claude/casehub/parent/docs/platform/auth.md`. Find the "Known roles" table and add:

```markdown
| `compliance-officers` | `casehub-aml` | SAR compliance review WorkItems |
| `aml-compliance` | `casehub-aml` | Gate approval (ACCOUNT_RESTRICTION, TRANSACTION_BLOCKING, ENTITY_LINK_CREATION), suspend/resume |
| `aml-mlro` | `casehub-aml` | SAR_FILING gate (exclusive), suspend/resume |
| `aml-senior-compliance` | `casehub-aml` | LAW_ENFORCEMENT_REFERRAL gate, GDPR erasure |
```

- [ ] **Step 2: Commit to parent repo**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/platform/auth.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs: add AML roles to platform Known roles table

Refs casehubio/aml#86"
```

- [ ] **Step 3: File a GitHub issue on casehubio/parent for the docs update**

```bash
gh issue create --repo casehubio/parent \
  --title "docs: AML roles added to platform auth.md Known roles table" \
  --body "AML roles documented as part of casehubio/aml#86. Commit landed on local main — needs push."
```

## References

- [2026-09-09-aml-workbench-auth-rbac-design.md] — design spec this plan implements
- [api/src/main/java/io/casehub/aml/domain/AmlGroups.java] — existing group constants
- [app/src/main/java/io/casehub/aml/compliance/AmlLayer7Resource.java] — erasure endpoints
- [app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java] — suspend/resume endpoints
- [app/src/main/java/io/casehub/aml/ComplianceReviewLifecycle.java:69] — string literal to replace
- [app/src/main/java/io/casehub/aml/compliance/ComplianceEscalationService.java:35] — bug fix: non-existent group name
- [app/src/main/webui/src/layout.ts] — workbench layout
- [app/src/main/webui/src/panels/compliance-dock.ts] — dock to split
- [app/src/main/webui/src/index.ts] — panel registration
- [../parent/docs/platform/auth.md] — platform role registry
- [casehub-life consumer guide §Layer 8] — reference RBAC pattern
- [casehub-life app/pom.xml:202-205] — quarkus-test-security reference
- [GitHub #86] — focal issue
