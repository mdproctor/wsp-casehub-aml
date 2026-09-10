## D1: UI gating granularity for compliance dock

**Choice:** Split the dock — separate `aml-compliance-dock` into a read-only compliance evidence panel (all users) and a GDPR erasure panel (`aml-senior-compliance` only). Apply `withAccess` at the pages layout level.
**Alternatives:**
- Component-level role check — keeps dock unified but requires role-checking inside Lit components, coupling UI to auth mechanism
- Wrap whole dock — simpler but hides read-only compliance evidence from non-compliance users
**Rationale:** Clean separation of read vs write concerns. `withAccess` at the layout level is the canonical pages pattern — no custom role logic inside components.
**Trade-offs:** Adds one more dock panel to the right sidebar.
**Sources:** `app/src/main/webui/src/panels/compliance-dock.ts`, `app/src/main/webui/src/layout.ts`, pages-ui `withAccess()` builder
**Exploration:** quick
**Status:** captured

## D2: Dev/demo profile OIDC handling

**Choice:** Same as casehub-life — OIDC off in dev/demo. `casehub-platform-oidc` not on dev classpath. `@RolesAllowed` annotations are inert. `withAccess` components always visible.
**Alternatives:**
- Mock roles in dev — active OIDC with mock identity provider; allows testing role-based behavior locally but adds config complexity
**Rationale:** Follows established casehub-life pattern (life#40). Development velocity matters more than testing RBAC in dev — RBAC correctness is verified by integration tests.
**Trade-offs:** Cannot manually test role-based filtering in dev mode without OIDC config.
**Sources:** casehub-life consumer guide §Layer 8, platform auth.md §RBAC Enforcement
**Exploration:** quick
**Status:** captured

## D3: @RolesAllowed placement — class-level vs method-level

**Choice:** Method-level on mutations only. Erasure resources (single-purpose POST) get class-level `@RolesAllowed`. `AmlLayer9Resource` gets method-level on POST suspend/resume — GET remains open to all authenticated users.
**Alternatives:**
- Class-level with `@PermitAll` on GET — more explicit but noisier, adds annotations to read-only methods that don't need them
**Rationale:** Matches read-vs-write security boundary. Class-level is clean when the entire resource is one role; method-level avoids over-restricting read endpoints on mixed resources.
**Trade-offs:** Inconsistent annotation level across resources — class on erasure, method on Layer9.
**Sources:** `app/src/main/java/io/casehub/aml/compliance/AmlLayer7Resource.java`, `app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java`
**Exploration:** quick
**Status:** captured
