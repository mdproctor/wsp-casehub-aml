# AmlActionRiskClassifier — Layer 9 Oversight Gate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement `AmlActionRiskClassifier` with the five AML consequential action types, wire the gate mechanism via `casehub-engine-work-adapter`, and demonstrate the oversight gate in a new Layer 9 harness that proves PEP entity links require human approval before the investigation proceeds.

**Architecture:** `AmlActionType` enum (api module, pure Java) encodes gate policy, groups, and reason per action type. `AmlActionRiskClassifier` (app/routing, `@RiskClassifier @ApplicationScoped`) classifies worker-declared PlannedActions; discovered automatically by the engine's CDI chain. Layer 9 harness (`AmlOversightCaseHub` + `AmlOversightCoordinator` + `AmlLayer9Resource`) provides a dedicated case definition with an entity-link-proposal worker that declares `PlannedAction(ENTITY_LINK_CREATION)`.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine-api 0.2-SNAPSHOT (ActionRiskClassifier SPI), casehub-engine-work-adapter 0.2-SNAPSHOT (ActionGateWorkItemHandler, WorkItemLifecycleAdapter), JUnit 5, Awaitility, REST Assured.

---

## File Map

| File | Status | Responsibility |
|---|---|---|
| `app/pom.xml` | Modify | Add work-adapter + blackboard compile deps |
| `app/src/main/resources/application.properties` | Modify | Add prod Jandex entries; fix wrong comment |
| `app/src/test/resources/application.properties` | Modify | Add test Jandex entries for work-adapter + blackboard |
| `api/src/main/java/io/casehub/aml/domain/AmlGroups.java` | Create | Group name constants |
| `api/src/main/java/io/casehub/aml/domain/AmlActionType.java` | Create | Action type enum with GatePolicy, groups, reason, scope |
| `api/src/test/java/io/casehub/aml/domain/AmlActionTypeTest.java` | Create | Round-trip, GatePolicy, null/unknown safety |
| `app/src/main/java/io/casehub/aml/routing/AmlActionRiskClassifier.java` | Create | @RiskClassifier implementation |
| `app/src/test/java/io/casehub/aml/routing/AmlActionRiskClassifierTest.java` | Create | All classification paths, fail-closed, boundary values |
| `app/src/main/resources/aml/aml-oversight-investigation.yaml` | Create | Layer 9 case definition (3 capabilities, goal, bindings) |
| `app/src/main/java/io/casehub/aml/engine/AmlOversightCaseHub.java` | Create | YamlCaseHub with entity-resolution, entity-link-proposal, investigation-summary workers |
| `app/src/main/java/io/casehub/aml/engine/AmlOversightCoordinator.java` | Create | Minimal case-start coordinator |
| `app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java` | Create | POST + GET /api/layer9/investigations |
| `app/src/test/java/io/casehub/aml/engine/AmlLayer9ActionGateTest.java` | Create | @QuarkusTest gate fires + autonomous paths |
| `docs/sql/V2011__UQ_TRUST_ATTEST_CASE_CAP_RECONSTRUCTED.sql` | Create | Manual PostgreSQL-only DDL for #57 |

---

## Task 1: Add casehub-engine-work-adapter dependency and Jandex entries

`casehub-engine-work-adapter` is NOT currently in the AML classpath (dependency tree confirmed). Without it, `ActionGateWorkItemHandler` and `WorkItemLifecycleAdapter` are absent — any GateRequired result stalls the case indefinitely. `ActionGateDeploymentHealthCheck` (already in engine runtime) will warn at startup once `AmlActionRiskClassifier` is registered.

**Files:**
- Modify: `app/pom.xml`
- Modify: `app/src/main/resources/application.properties`
- Modify: `app/src/test/resources/application.properties`

- [ ] **Step 1: Add compile deps in app/pom.xml**

  Add after the `casehub-engine-ledger` dependency:

  ```xml
  <!-- Layer 9: oversight gate — ActionGateWorkItemHandler + WorkItemLifecycleAdapter -->
  <dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-work-adapter</artifactId>
    <version>${casehub.version}</version>
  </dependency>
  <!-- Transitive of work-adapter; separate Jandex entry required for CDI scanning -->
  <dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-blackboard</artifactId>
    <version>${casehub.version}</version>
  </dependency>
  ```

- [ ] **Step 2: Add Jandex entries to test application.properties**

  Add after the existing `quarkus.index-dependency.casehub-platform-memory-inmem.*` block:

  ```properties
  # Layer 9: work-adapter provides ActionGateWorkItemHandler + WorkItemLifecycleAdapter (gate mechanism)
  # blackboard provides BlackboardRegistry used by WorkItemLifecycleAdapter
  quarkus.index-dependency.engine-work-adapter.group-id=io.casehub
  quarkus.index-dependency.engine-work-adapter.artifact-id=casehub-engine-work-adapter
  quarkus.index-dependency.engine-blackboard.group-id=io.casehub
  quarkus.index-dependency.engine-blackboard.artifact-id=casehub-engine-blackboard
  ```

- [ ] **Step 3: Fix wrong comment + add Jandex entries in main application.properties**

  In `app/src/main/resources/application.properties`, find the comment:
  ```
  # engine-work-adapter comes in transitively; MemoryPlanItemStore displaces JpaPlanItemStore
  ```
  Replace with:
  ```
  # engine-work-adapter is an explicit compile dep (not transitive); MemoryPlanItemStore displaces JpaPlanItemStore
  ```

  Add at end of `%prod` index-dependency block:
  ```properties
  %prod.quarkus.index-dependency.engine-work-adapter.group-id=io.casehub
  %prod.quarkus.index-dependency.engine-work-adapter.artifact-id=casehub-engine-work-adapter
  %prod.quarkus.index-dependency.engine-blackboard.group-id=io.casehub
  %prod.quarkus.index-dependency.engine-blackboard.artifact-id=casehub-engine-blackboard
  ```

- [ ] **Step 4: Verify build compiles**

  ```bash
  mvn compile -pl app -am -f /Users/mdproctor/claude/casehub/aml/pom.xml
  ```

  Expected: BUILD SUCCESS. If `casehub-engine-blackboard` is not found in local m2, run `mvn install -pl work-adapter -f /Users/mdproctor/claude/casehub/engine/pom.xml` first (it exists at `/Users/mdproctor/.m2/repository/io/casehub/casehub-engine-blackboard/`).

- [ ] **Step 5: Commit**

  ```bash
  git -C /Users/mdproctor/claude/casehub/aml add app/pom.xml \
    app/src/main/resources/application.properties \
    app/src/test/resources/application.properties
  git -C /Users/mdproctor/claude/casehub/aml commit -m "build(#42): add casehub-engine-work-adapter + blackboard deps for oversight gate mechanism

  Refs #42"
  ```

---

## Task 2: AmlGroups and AmlActionType (api module, TDD)

**Files:**
- Create: `api/src/main/java/io/casehub/aml/domain/AmlGroups.java`
- Create: `api/src/main/java/io/casehub/aml/domain/AmlActionType.java`
- Create: `api/src/test/java/io/casehub/aml/domain/AmlActionTypeTest.java`

- [ ] **Step 1: Write failing test**

  Create `api/src/test/java/io/casehub/aml/domain/AmlActionTypeTest.java`:

  ```java
  package io.casehub.aml.domain;

  import org.junit.jupiter.api.Test;
  import java.util.Optional;
  import static org.junit.jupiter.api.Assertions.*;

  class AmlActionTypeTest {

      // --- fromActionType round-trips ---

      @Test
      void fromActionType_sarFiling_returnsSarFiling() {
          Optional<AmlActionType> result = AmlActionType.fromActionType("sar.filing");
          assertEquals(Optional.of(AmlActionType.SAR_FILING), result);
      }

      @Test
      void fromActionType_entityLinkCreation_returnsEntityLinkCreation() {
          assertEquals(Optional.of(AmlActionType.ENTITY_LINK_CREATION),
              AmlActionType.fromActionType("entity.link.creation"));
      }

      @Test
      void fromActionType_allConstantsRoundTrip() {
          for (AmlActionType type : AmlActionType.values()) {
              assertEquals(Optional.of(type),
                  AmlActionType.fromActionType(type.actionType()),
                  "Round-trip failed for: " + type);
          }
      }

      // --- Safe handling of unknown/null ---

      @Test
      void fromActionType_null_returnsEmpty() {
          assertEquals(Optional.empty(), AmlActionType.fromActionType(null));
      }

      @Test
      void fromActionType_unknown_returnsEmpty() {
          assertEquals(Optional.empty(), AmlActionType.fromActionType("unknown.type"));
      }

      @Test
      void fromActionType_uppercase_returnsEmpty() {
          // actionType() is lowercase-dot; uppercase input is not a match
          assertEquals(Optional.empty(), AmlActionType.fromActionType("SAR_FILING"));
      }

      // --- GatePolicy per constant ---

      @Test
      void sarFiling_isAlwaysGated() {
          assertEquals(AmlActionType.GatePolicy.ALWAYS, AmlActionType.SAR_FILING.gatePolicy());
      }

      @Test
      void lawEnforcementReferral_isAlwaysGated() {
          assertEquals(AmlActionType.GatePolicy.ALWAYS,
              AmlActionType.LAW_ENFORCEMENT_REFERRAL.gatePolicy());
      }

      @Test
      void entityLinkCreation_isRiskScoreThreshold() {
          assertEquals(AmlActionType.GatePolicy.RISK_SCORE_THRESHOLD,
              AmlActionType.ENTITY_LINK_CREATION.gatePolicy());
      }

      @Test
      void accountRestriction_isRiskScoreThreshold() {
          assertEquals(AmlActionType.GatePolicy.RISK_SCORE_THRESHOLD,
              AmlActionType.ACCOUNT_RESTRICTION.gatePolicy());
      }

      @Test
      void transactionBlocking_isConfidenceThreshold() {
          assertEquals(AmlActionType.GatePolicy.CONFIDENCE_THRESHOLD,
              AmlActionType.TRANSACTION_BLOCKING.gatePolicy());
      }

      // --- Reversibility ---

      @Test
      void sarFiling_isNotReversible() {
          assertFalse(AmlActionType.SAR_FILING.reversible());
      }

      @Test
      void entityLinkCreation_isReversible() {
          assertTrue(AmlActionType.ENTITY_LINK_CREATION.reversible());
      }

      @Test
      void transactionBlocking_isNotReversible() {
          assertFalse(AmlActionType.TRANSACTION_BLOCKING.reversible());
      }

      // --- Groups ---

      @Test
      void sarFiling_requiresMlro() {
          assertEquals(java.util.List.of(AmlGroups.MLRO),
              AmlActionType.SAR_FILING.candidateGroups());
      }

      @Test
      void lawEnforcementReferral_requiresSeniorCompliance() {
          assertEquals(java.util.List.of(AmlGroups.AML_SENIOR_COMPLIANCE),
              AmlActionType.LAW_ENFORCEMENT_REFERRAL.candidateGroups());
      }

      @Test
      void entityLinkCreation_requiresAmlCompliance() {
          assertEquals(java.util.List.of(AmlGroups.AML_COMPLIANCE),
              AmlActionType.ENTITY_LINK_CREATION.candidateGroups());
      }

      // --- Reason and scope not null ---

      @Test
      void allTypes_haveNonNullReason() {
          for (AmlActionType type : AmlActionType.values()) {
              assertNotNull(type.reason(), "reason() null for: " + type);
              assertFalse(type.reason().isBlank(), "reason() blank for: " + type);
          }
      }

      @Test
      void allTypes_haveOversightScope() {
          for (AmlActionType type : AmlActionType.values()) {
              assertEquals("casehubio/aml/oversight", type.scope(),
                  "scope() wrong for: " + type);
          }
      }

      @Test
      void allTypes_expiresInIsNull() {
          for (AmlActionType type : AmlActionType.values()) {
              assertNull(type.expiresIn(), "expiresIn() non-null for: " + type);
          }
      }

      // --- actionType() format ---

      @Test
      void actionType_usesLowercaseDots() {
          assertEquals("sar.filing", AmlActionType.SAR_FILING.actionType());
          assertEquals("account.restriction", AmlActionType.ACCOUNT_RESTRICTION.actionType());
          assertEquals("transaction.blocking", AmlActionType.TRANSACTION_BLOCKING.actionType());
          assertEquals("entity.link.creation", AmlActionType.ENTITY_LINK_CREATION.actionType());
          assertEquals("law.enforcement.referral", AmlActionType.LAW_ENFORCEMENT_REFERRAL.actionType());
      }
  }
  ```

- [ ] **Step 2: Run test to confirm compile failure**

  ```bash
  mvn test -pl api -am -f /Users/mdproctor/claude/casehub/aml/pom.xml \
    -Dtest=AmlActionTypeTest -Dsurefire.failIfNoSpecifiedTests=false
  ```

  Expected: FAIL — `AmlActionType` not found.

- [ ] **Step 3: Create AmlGroups.java**

  Create `api/src/main/java/io/casehub/aml/domain/AmlGroups.java`:

  ```java
  package io.casehub.aml.domain;

  public final class AmlGroups {
      private AmlGroups() {}

      public static final String MLRO = "aml-mlro";
      public static final String AML_COMPLIANCE = "aml-compliance";
      public static final String AML_SENIOR_COMPLIANCE = "aml-senior-compliance";
  }
  ```

- [ ] **Step 4: Create AmlActionType.java**

  Create `api/src/main/java/io/casehub/aml/domain/AmlActionType.java`:

  ```java
  package io.casehub.aml.domain;

  import java.time.Duration;
  import java.util.Arrays;
  import java.util.List;
  import java.util.Optional;

  public enum AmlActionType {

      SAR_FILING(
          GatePolicy.ALWAYS, false,
          List.of(AmlGroups.MLRO),
          "SAR submission to regulator — MLRO sign-off required (FinCEN/FCA)"),

      ACCOUNT_RESTRICTION(
          GatePolicy.RISK_SCORE_THRESHOLD, true,
          List.of(AmlGroups.AML_COMPLIANCE),
          "Account restriction affects customer — confirm before action"),

      TRANSACTION_BLOCKING(
          GatePolicy.CONFIDENCE_THRESHOLD, false,
          List.of(AmlGroups.AML_COMPLIANCE),
          "Transaction block — low-confidence pattern — human review required"),

      ENTITY_LINK_CREATION(
          GatePolicy.RISK_SCORE_THRESHOLD, true,
          List.of(AmlGroups.AML_COMPLIANCE),
          "Entity network link has downstream investigation implications — confirm evidence basis"),

      LAW_ENFORCEMENT_REFERRAL(
          GatePolicy.ALWAYS, false,
          List.of(AmlGroups.AML_SENIOR_COMPLIANCE),
          "Law enforcement referral — senior compliance director approval required");

      public enum GatePolicy {
          ALWAYS, RISK_SCORE_THRESHOLD, CONFIDENCE_THRESHOLD
      }

      private static final String OVERSIGHT_SCOPE = "casehubio/aml/oversight";

      private final GatePolicy gatePolicy;
      private final boolean reversible;
      private final List<String> candidateGroups;
      private final String reason;

      AmlActionType(GatePolicy gatePolicy, boolean reversible, List<String> candidateGroups, String reason) {
          this.gatePolicy = gatePolicy;
          this.reversible = reversible;
          this.candidateGroups = List.copyOf(candidateGroups);
          this.reason = reason;
      }

      public GatePolicy gatePolicy() { return gatePolicy; }
      public boolean reversible() { return reversible; }
      public List<String> candidateGroups() { return candidateGroups; }
      public String reason() { return reason; }
      public String scope() { return OVERSIGHT_SCOPE; }

      /** expiresIn is null — expiry policy is regulatory and configurable post-GA. */
      public Duration expiresIn() { return null; }

      /** Returns the PlannedAction actionType string: e.g. SAR_FILING → "sar.filing" */
      public String actionType() {
          return name().toLowerCase().replace('_', '.');
      }

      /**
       * Parse a PlannedAction.actionType() string back to the enum constant.
       * Uses stream filter — never throws on unrecognised input.
       */
      public static Optional<AmlActionType> fromActionType(String actionType) {
          if (actionType == null) return Optional.empty();
          return Arrays.stream(values())
              .filter(a -> a.actionType().equals(actionType))
              .findFirst();
      }
  }
  ```

- [ ] **Step 5: Run tests and verify they pass**

  ```bash
  mvn test -pl api -am -f /Users/mdproctor/claude/casehub/aml/pom.xml \
    -Dtest=AmlActionTypeTest -Dsurefire.failIfNoSpecifiedTests=false
  ```

  Expected: BUILD SUCCESS, 21 tests passing.

- [ ] **Step 6: Commit**

  ```bash
  git -C /Users/mdproctor/claude/casehub/aml add \
    api/src/main/java/io/casehub/aml/domain/AmlGroups.java \
    api/src/main/java/io/casehub/aml/domain/AmlActionType.java \
    api/src/test/java/io/casehub/aml/domain/AmlActionTypeTest.java
  git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#42): AmlActionType enum + AmlGroups — AML consequential action vocabulary

  Refs #42"
  ```

---

## Task 3: AmlActionRiskClassifier (app module, TDD)

**Files:**
- Create: `app/src/main/java/io/casehub/aml/routing/AmlActionRiskClassifier.java`
- Create: `app/src/test/java/io/casehub/aml/routing/AmlActionRiskClassifierTest.java`

- [ ] **Step 1: Write failing test**

  Create `app/src/test/java/io/casehub/aml/routing/AmlActionRiskClassifierTest.java`:

  ```java
  package io.casehub.aml.routing;

  import io.casehub.api.spi.PlannedAction;
  import io.casehub.api.spi.RiskDecision;
  import io.casehub.aml.domain.AmlActionType;
  import io.casehub.aml.domain.AmlGroups;
  import org.junit.jupiter.api.BeforeEach;
  import org.junit.jupiter.api.Test;
  import java.util.Map;
  import static org.junit.jupiter.api.Assertions.*;

  class AmlActionRiskClassifierTest {

      AmlActionRiskClassifier classifier;

      @BeforeEach
      void setUp() {
          classifier = new AmlActionRiskClassifier();
      }

      // ── ALWAYS-gated types ───────────────────────────────────────────────────

      @Test
      void sarFiling_alwaysGatesRegardlessOfContext() {
          RiskDecision result = classifier.classify(PlannedAction.of(
              "SAR draft ready for MLRO review",
              AmlActionType.SAR_FILING.actionType(),
              Map.of("transactionId", "TXN-001")));
          assertGateRequired(result, AmlGroups.MLRO, false);
      }

      @Test
      void lawEnforcementReferral_alwaysGates() {
          RiskDecision result = classifier.classify(PlannedAction.of(
              "Refer case to NCA",
              AmlActionType.LAW_ENFORCEMENT_REFERRAL.actionType(),
              Map.of()));
          assertGateRequired(result, AmlGroups.AML_SENIOR_COMPLIANCE, false);
      }

      // ── ENTITY_LINK_CREATION (RISK_SCORE_THRESHOLD) ─────────────────────────

      @Test
      void entityLink_pep_alwaysGates() {
          RiskDecision result = classify(AmlActionType.ENTITY_LINK_CREATION,
              Map.of("entityType", "PEP", "riskScore", 0.2));
          assertGateRequired(result, AmlGroups.AML_COMPLIANCE, true);
      }

      @Test
      void entityLink_highRisk_gates() {
          RiskDecision result = classify(AmlActionType.ENTITY_LINK_CREATION,
              Map.of("entityType", "CORPORATE", "riskScore", 0.9));
          assertGateRequired(result, AmlGroups.AML_COMPLIANCE, true);
      }

      @Test
      void entityLink_atThreshold_gates() {
          RiskDecision result = classify(AmlActionType.ENTITY_LINK_CREATION,
              Map.of("entityType", "CORPORATE", "riskScore", 0.8));
          assertGateRequired(result, AmlGroups.AML_COMPLIANCE, true);
      }

      @Test
      void entityLink_lowRiskCorporate_autonomous() {
          RiskDecision result = classify(AmlActionType.ENTITY_LINK_CREATION,
              Map.of("entityType", "CORPORATE", "riskScore", 0.35));
          assertInstanceOf(RiskDecision.Autonomous.class, result,
              "Low-risk CORPORATE entity link must be Autonomous");
      }

      @Test
      void entityLink_justBelowThreshold_autonomous() {
          RiskDecision result = classify(AmlActionType.ENTITY_LINK_CREATION,
              Map.of("entityType", "CORPORATE", "riskScore", 0.799));
          assertInstanceOf(RiskDecision.Autonomous.class, result);
      }

      @Test
      void entityLink_missingRiskScore_failClosed() {
          RiskDecision result = classify(AmlActionType.ENTITY_LINK_CREATION,
              Map.of("entityType", "CORPORATE"));
          assertGateRequiredWithReason(result, "Risk assessment unavailable");
      }

      @Test
      void entityLink_missingContext_failClosed() {
          RiskDecision result = classify(AmlActionType.ENTITY_LINK_CREATION, Map.of());
          assertGateRequiredWithReason(result, "Risk assessment unavailable");
      }

      @Test
      void entityLink_nullContext_failClosed() {
          RiskDecision result = classifier.classify(
              PlannedAction.of("desc", AmlActionType.ENTITY_LINK_CREATION.actionType(), null));
          assertGateRequiredWithReason(result, "Risk assessment unavailable");
      }

      // ── ACCOUNT_RESTRICTION (RISK_SCORE_THRESHOLD) ──────────────────────────

      @Test
      void accountRestriction_highRiskScore_gates() {
          RiskDecision result = classify(AmlActionType.ACCOUNT_RESTRICTION,
              Map.of("riskScore", 0.85, "entityType", "CORPORATE"));
          assertGateRequired(result, AmlGroups.AML_COMPLIANCE, true);
      }

      @Test
      void accountRestriction_pepEntity_gates() {
          RiskDecision result = classify(AmlActionType.ACCOUNT_RESTRICTION,
              Map.of("riskScore", 0.2, "entityType", "PEP"));
          assertGateRequired(result, AmlGroups.AML_COMPLIANCE, true);
      }

      @Test
      void accountRestriction_lowRiskCorporate_autonomous() {
          RiskDecision result = classify(AmlActionType.ACCOUNT_RESTRICTION,
              Map.of("riskScore", 0.5, "entityType", "CORPORATE"));
          assertInstanceOf(RiskDecision.Autonomous.class, result);
      }

      @Test
      void accountRestriction_missingRiskScore_failClosed() {
          RiskDecision result = classify(AmlActionType.ACCOUNT_RESTRICTION,
              Map.of("entityType", "CORPORATE"));
          assertGateRequiredWithReason(result, "Risk assessment unavailable");
      }

      // ── TRANSACTION_BLOCKING (CONFIDENCE_THRESHOLD) ──────────────────────────

      @Test
      void transactionBlocking_highConfidence_autonomous() {
          RiskDecision result = classify(AmlActionType.TRANSACTION_BLOCKING,
              Map.of("confidenceScore", 0.95));
          assertInstanceOf(RiskDecision.Autonomous.class, result,
              "High confidence must proceed autonomously");
      }

      @Test
      void transactionBlocking_atThreshold_autonomous() {
          // threshold is 0.9 — AT threshold means >= 0.9 which is autonomous
          RiskDecision result = classify(AmlActionType.TRANSACTION_BLOCKING,
              Map.of("confidenceScore", 0.9));
          assertInstanceOf(RiskDecision.Autonomous.class, result);
      }

      @Test
      void transactionBlocking_belowThreshold_gates() {
          RiskDecision result = classify(AmlActionType.TRANSACTION_BLOCKING,
              Map.of("confidenceScore", 0.7));
          assertGateRequired(result, AmlGroups.AML_COMPLIANCE, false);
      }

      @Test
      void transactionBlocking_justBelowThreshold_gates() {
          RiskDecision result = classify(AmlActionType.TRANSACTION_BLOCKING,
              Map.of("confidenceScore", 0.899));
          assertGateRequired(result, AmlGroups.AML_COMPLIANCE, false);
      }

      @Test
      void transactionBlocking_missingConfidenceScore_failClosed() {
          RiskDecision result = classify(AmlActionType.TRANSACTION_BLOCKING, Map.of());
          assertGateRequiredWithReason(result, "Risk assessment unavailable");
      }

      // ── Unknown / null actionType ─────────────────────────────────────────────

      @Test
      void unknownActionType_autonomous() {
          RiskDecision result = classifier.classify(
              PlannedAction.of("something", "foo.bar", Map.of()));
          assertInstanceOf(RiskDecision.Autonomous.class, result,
              "Unknown action type must be Autonomous");
      }

      @Test
      void nullActionType_autonomous() {
          RiskDecision result = classifier.classify(
              PlannedAction.of("something", null, Map.of()));
          assertInstanceOf(RiskDecision.Autonomous.class, result);
      }

      // ── Gate properties ───────────────────────────────────────────────────────

      @Test
      void gateRequired_scopeIsAmlOversight() {
          RiskDecision result = classify(AmlActionType.SAR_FILING, Map.of());
          RiskDecision.GateRequired gate = assertInstanceOf(RiskDecision.GateRequired.class, result);
          assertEquals("casehubio/aml/oversight", gate.scope());
      }

      @Test
      void gateRequired_expiresInIsNull() {
          RiskDecision result = classify(AmlActionType.SAR_FILING, Map.of());
          RiskDecision.GateRequired gate = assertInstanceOf(RiskDecision.GateRequired.class, result);
          assertNull(gate.expiresIn());
      }

      // ── Helpers ────────────────────────────────────────────────────────────────

      private RiskDecision classify(AmlActionType type, Map<String, Object> context) {
          return classifier.classify(PlannedAction.of("test action", type.actionType(), context));
      }

      private void assertGateRequired(RiskDecision result, String expectedGroup, boolean expectedReversible) {
          RiskDecision.GateRequired gate = assertInstanceOf(RiskDecision.GateRequired.class, result);
          assertTrue(gate.candidateGroups().contains(expectedGroup),
              "Expected group " + expectedGroup + " in " + gate.candidateGroups());
          assertEquals(expectedReversible, gate.reversible());
          assertNotNull(gate.reason());
          assertFalse(gate.reason().isBlank());
      }

      private void assertGateRequiredWithReason(RiskDecision result, String reasonFragment) {
          RiskDecision.GateRequired gate = assertInstanceOf(RiskDecision.GateRequired.class, result);
          assertTrue(gate.reason().contains(reasonFragment),
              "Expected reason to contain '" + reasonFragment + "' but was: " + gate.reason());
      }
  }
  ```

- [ ] **Step 2: Run test to confirm failure**

  ```bash
  mvn test -pl app -am -f /Users/mdproctor/claude/casehub/aml/pom.xml \
    -Dtest=AmlActionRiskClassifierTest -Dsurefire.failIfNoSpecifiedTests=false
  ```

  Expected: FAIL — `AmlActionRiskClassifier` not found.

- [ ] **Step 3: Create AmlActionRiskClassifier.java**

  Create `app/src/main/java/io/casehub/aml/routing/AmlActionRiskClassifier.java`:

  ```java
  package io.casehub.aml.routing;

  import io.casehub.api.spi.ActionRiskClassifier;
  import io.casehub.api.spi.PlannedAction;
  import io.casehub.api.spi.RiskClassifier;
  import io.casehub.api.spi.RiskDecision;
  import io.casehub.aml.domain.AmlActionType;
  import jakarta.enterprise.context.ApplicationScoped;
  import java.util.Map;
  import java.util.Optional;

  /**
   * AML-specific ActionRiskClassifier — discovered by casehub-engine via @RiskClassifier CDI
   * qualifier. Encodes FinCEN/FCA regulatory gate requirements for consequential AML actions.
   *
   * <p>Fail-closed for known types with missing context: if a recognized consequential action
   * type arrives without the context needed to assess risk, the action is gated rather than
   * allowed to proceed unreviewed.
   *
   * <p>Unknown actionType → Autonomous (this classifier does not gate actions it doesn't own).
   */
  @ApplicationScoped
  @RiskClassifier
  public class AmlActionRiskClassifier implements ActionRiskClassifier {

      static final double RISK_SCORE_GATE_THRESHOLD = 0.8;
      static final double CONFIDENCE_GATE_THRESHOLD = 0.9;

      @Override
      public RiskDecision classify(PlannedAction action) {
          Optional<AmlActionType> typeOpt = AmlActionType.fromActionType(action.actionType());
          if (typeOpt.isEmpty()) {
              return new RiskDecision.Autonomous();
          }
          AmlActionType type = typeOpt.get();
          return switch (type.gatePolicy()) {
              case ALWAYS -> gate(type);
              case RISK_SCORE_THRESHOLD -> classifyByRiskScore(type, action.context());
              case CONFIDENCE_THRESHOLD -> classifyByConfidence(type, action.context());
          };
      }

      private RiskDecision classifyByRiskScore(AmlActionType type, Map<String, Object> context) {
          if (context != null && "PEP".equals(context.get("entityType"))) {
              return gate(type);
          }
          Object raw = context != null ? context.get("riskScore") : null;
          if (raw == null) return missingContext(type);
          try {
              double score = Double.parseDouble(raw.toString());
              return score >= RISK_SCORE_GATE_THRESHOLD ? gate(type) : new RiskDecision.Autonomous();
          } catch (NumberFormatException e) {
              return missingContext(type);
          }
      }

      private RiskDecision classifyByConfidence(AmlActionType type, Map<String, Object> context) {
          Object raw = context != null ? context.get("confidenceScore") : null;
          if (raw == null) return missingContext(type);
          try {
              double score = Double.parseDouble(raw.toString());
              return score < CONFIDENCE_GATE_THRESHOLD ? gate(type) : new RiskDecision.Autonomous();
          } catch (NumberFormatException e) {
              return missingContext(type);
          }
      }

      private RiskDecision.GateRequired gate(AmlActionType type) {
          return new RiskDecision.GateRequired(
              type.reason(), type.reversible(), type.candidateGroups(),
              type.expiresIn(), type.scope());
      }

      private RiskDecision.GateRequired missingContext(AmlActionType type) {
          return new RiskDecision.GateRequired(
              "Risk assessment unavailable — human review required",
              true, type.candidateGroups(), null, type.scope());
      }
  }
  ```

- [ ] **Step 4: Run tests and verify they pass**

  ```bash
  mvn test -pl app -am -f /Users/mdproctor/claude/casehub/aml/pom.xml \
    -Dtest=AmlActionRiskClassifierTest -Dsurefire.failIfNoSpecifiedTests=false
  ```

  Expected: BUILD SUCCESS, ~28 tests passing.

- [ ] **Step 5: Commit**

  ```bash
  git -C /Users/mdproctor/claude/casehub/aml add \
    app/src/main/java/io/casehub/aml/routing/AmlActionRiskClassifier.java \
    app/src/test/java/io/casehub/aml/routing/AmlActionRiskClassifierTest.java
  git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#42): AmlActionRiskClassifier — AML oversight gate classifier

  Refs #42"
  ```

---

## Task 4: Layer 9 YAML + Harness Infrastructure (TDD)

Write the integration test first. It will fail because the endpoint doesn't exist. Then create all the infrastructure to make it pass.

**Files:**
- Create: `app/src/test/java/io/casehub/aml/engine/AmlLayer9ActionGateTest.java`
- Create: `app/src/main/resources/aml/aml-oversight-investigation.yaml`
- Create: `app/src/main/java/io/casehub/aml/engine/AmlOversightCaseHub.java`
- Create: `app/src/main/java/io/casehub/aml/engine/AmlOversightCoordinator.java`
- Create: `app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java`

- [ ] **Step 1: Write AmlLayer9ActionGateTest.java**

  Create `app/src/test/java/io/casehub/aml/engine/AmlLayer9ActionGateTest.java`:

  ```java
  package io.casehub.aml.engine;

  import io.casehub.aml.domain.AmlActionType;
  import io.casehub.aml.domain.AmlGroups;
  import io.casehub.aml.domain.SuspiciousTransaction;
  import io.casehub.work.runtime.model.WorkItem;
  import io.casehub.work.runtime.service.WorkItemService;
  import io.quarkus.test.junit.QuarkusTest;
  import io.restassured.http.ContentType;
  import jakarta.inject.Inject;
  import jakarta.persistence.EntityManager;
  import jakarta.persistence.PersistenceContext;
  import org.awaitility.Awaitility;
  import org.junit.jupiter.api.Test;
  import java.math.BigDecimal;
  import java.time.Instant;
  import java.util.List;
  import java.util.UUID;
  import java.util.concurrent.TimeUnit;
  import static io.restassured.RestAssured.given;
  import static org.junit.jupiter.api.Assertions.*;

  /**
   * Layer 9: verifies the AML oversight gate fires for PEP entity link creation and
   * that low-risk CORPORATE cases proceed autonomously.
   *
   * <p>Gate WorkItem lives on the default PU (WorkItem is in io.casehub.work.runtime.model,
   * mapped to the default datasource). Use @PersistenceContext with no unitName.
   *
   * <p>completeFromSystem() fires both fire() and fireAsync() — the @ObservesAsync
   * WorkItemLifecycleAdapter receives the async event and routes to ActionGateCompletionApplier,
   * which resumes the engine case.
   */
  @QuarkusTest
  class AmlLayer9ActionGateTest {

      @PersistenceContext  // no unitName — WorkItem on default datasource, not qhorus
      EntityManager em;

      @Inject
      WorkItemService workItemService;

      @Test
      void gate_fires_for_pep_entity_and_resumes_on_approval() {
          SuspiciousTransaction tx = new SuspiciousTransaction(
              "TXN-GATE-" + UUID.randomUUID(), "ACC-GATE-A", "ACC-GATE-B",
              new BigDecimal("500000"), "GBP",
              Instant.parse("2024-12-01T00:00:00Z"),
              "Entity link — PEP suspected beneficial owner");

          String caseIdStr = given().contentType(ContentType.JSON).body(tx)
              .when().post("/api/layer9/investigations")
              .then().statusCode(202)
              .extract().path("caseId");

          UUID caseId = UUID.fromString(caseIdStr);

          // Wait for gate WorkItem to appear (positive signal: gate fired, engine paused)
          Awaitility.await()
              .atMost(15, TimeUnit.SECONDS)
              .pollInterval(300, TimeUnit.MILLISECONDS)
              .until(() -> !findGateWorkItems(caseId).isEmpty());

          List<WorkItem> gateItems = findGateWorkItems(caseId);
          assertEquals(1, gateItems.size(), "Exactly one gate WorkItem must be created");
          WorkItem gate = gateItems.get(0);

          // Verify gate WorkItem properties
          assertEquals(AmlGroups.AML_COMPLIANCE, gate.candidateGroups,
              "candidateGroups must be aml-compliance (ENTITY_LINK_CREATION)");
          assertTrue(gate.callerRef.startsWith("case:" + caseId + "/gate:"),
              "callerRef must follow GateCallerRef format");

          // Engine is paused — investigation must not be completed yet
          String statusBeforeApproval = given()
              .when().get("/api/layer9/investigations/" + caseIdStr)
              .then().statusCode(200)
              .extract().path("status");
          assertNotEquals("completed", statusBeforeApproval,
              "Investigation must be paused at gate, not completed");

          // Approve the gate as MLRO-equivalent test actor
          workItemService.completeFromSystem(gate.id, "test-aml-compliance", "approved");

          // Await completion — ActionGateCompletionApplier resumes the engine asynchronously
          Awaitility.await()
              .atMost(15, TimeUnit.SECONDS)
              .pollInterval(300, TimeUnit.MILLISECONDS)
              .until(() -> "completed".equals(
                  given().when().get("/api/layer9/investigations/" + caseIdStr)
                      .then().extract().path("status")));
      }

      @Test
      void gate_not_fired_for_low_risk_corporate() {
          SuspiciousTransaction tx = new SuspiciousTransaction(
              "TXN-AUTO-" + UUID.randomUUID(), "ACC-AUTO-C", "ACC-AUTO-D",
              new BigDecimal("50000"), "GBP",
              Instant.parse("2024-12-01T00:00:00Z"),
              "Routine structured layering — CORPORATE");

          String caseIdStr = given().contentType(ContentType.JSON).body(tx)
              .when().post("/api/layer9/investigations")
              .then().statusCode(202)
              .extract().path("caseId");

          UUID caseId = UUID.fromString(caseIdStr);

          // Classifier returns Autonomous for low-risk CORPORATE — no gate should fire.
          // Await completion directly.
          Awaitility.await()
              .atMost(15, TimeUnit.SECONDS)
              .pollInterval(300, TimeUnit.MILLISECONDS)
              .until(() -> "completed".equals(
                  given().when().get("/api/layer9/investigations/" + caseIdStr)
                      .then().extract().path("status")));

          // Confirm no gate WorkItem was created for this case
          assertTrue(findGateWorkItems(caseId).isEmpty(),
              "No gate WorkItem must be created for low-risk CORPORATE entity");
      }

      private List<WorkItem> findGateWorkItems(UUID caseId) {
          return em.createQuery(
              "SELECT w FROM WorkItem w WHERE w.callerRef LIKE :pattern",
              WorkItem.class)
              .setParameter("pattern", "case:" + caseId + "/gate:%")
              .getResultList();
      }
  }
  ```

- [ ] **Step 2: Run test to confirm failure (404 on POST)**

  ```bash
  mvn test -pl app -am -f /Users/mdproctor/claude/casehub/aml/pom.xml \
    -Dtest=AmlLayer9ActionGateTest -Dsurefire.failIfNoSpecifiedTests=false
  ```

  Expected: FAIL — 404 on POST /api/layer9/investigations (endpoint doesn't exist yet).

- [ ] **Step 3: Create aml-oversight-investigation.yaml**

  Create `app/src/main/resources/aml/aml-oversight-investigation.yaml`:

  ```yaml
  dsl: "0.1"
  version: "1.0.0"
  name: aml-oversight-investigation
  namespace: casehub-aml
  title: AML Oversight Gate — entity network link confirmation

  spec:

    ## ─── Capabilities ─────────────────────────────────────────────────────────
    capabilities:
      - name: entity-resolution
        description: "Identify entities and risk profile from suspicious transaction"
        inputSchema: "{ transaction: .transaction }"
        outputSchema: "{ entityResolution: . }"

      - name: entity-link-proposal
        description: "Propose adding entity network link to investigation graph"
        inputSchema: "{ entityResolution: .entityResolution }"
        outputSchema: "{ entityLinkProposal: . }"

      - name: investigation-summary
        description: "Summarise investigation findings after entity link is confirmed"
        inputSchema: "{ entityResolution: .entityResolution, entityLinkProposal: .entityLinkProposal }"
        outputSchema: "{ investigationSummary: . }"

    ## ─── Goals ────────────────────────────────────────────────────────────────
    goals:
      - name: investigation-complete
        kind: success
        condition: ".investigationSummary != null"

    ## ─── Completion ───────────────────────────────────────────────────────────
    completion:
      success:
        allOf:
          - investigation-complete

    ## ─── Bindings ─────────────────────────────────────────────────────────────
    bindings:

      ## Fires first — entity resolution has no prerequisite
      - name: entity-resolution
        on: { contextChange: {} }
        when: ".transaction != null and .entityResolution == null"
        capability: entity-resolution

      ## Fires after entity resolution; declares PlannedAction(ENTITY_LINK_CREATION).
      ## Engine gates this step for PEP or high-risk entities — output is not committed
      ## until human approves. For Autonomous cases, output is committed immediately.
      - name: entity-link-proposal
        on: { contextChange: {} }
        when: ".entityResolution != null and .entityLinkProposal == null"
        capability: entity-link-proposal

      ## Fires after entityLinkProposal is present in context (gate approved or Autonomous).
      - name: investigation-summary
        on: { contextChange: {} }
        when: ".entityLinkProposal != null and .investigationSummary == null"
        capability: investigation-summary
  ```

- [ ] **Step 4: Create AmlOversightCaseHub.java**

  Create `app/src/main/java/io/casehub/aml/engine/AmlOversightCaseHub.java`:

  ```java
  package io.casehub.aml.engine;

  import io.casehub.api.engine.YamlCaseHub;
  import io.casehub.api.model.Capability;
  import io.casehub.api.model.CaseDefinition;
  import io.casehub.api.model.Worker;
  import io.casehub.api.model.WorkerResult;
  import io.casehub.api.spi.PlannedAction;
  import io.casehub.aml.domain.AmlActionType;
  import jakarta.enterprise.context.ApplicationScoped;
  import java.util.List;
  import java.util.Map;

  /**
   * Layer 9 case hub — oversight gate demonstration.
   *
   * <p>Loads aml-oversight-investigation.yaml and augments it with three in-process workers.
   * The entity-link-proposal worker declares a PlannedAction(ENTITY_LINK_CREATION) so the
   * engine calls AmlActionRiskClassifier before committing the worker's output. PEP entities
   * and high-risk scores (≥0.8) trigger a GateRequired; low-risk CORPORATE cases proceed
   * autonomously.
   */
  @ApplicationScoped
  public class AmlOversightCaseHub extends YamlCaseHub {

      public AmlOversightCaseHub() {
          super("aml/aml-oversight-investigation.yaml");
      }

      @Override
      public CaseDefinition getDefinition() {
          CaseDefinition def = super.getDefinition();
          def.getWorkers().addAll(List.of(
              entityResolutionWorker(),
              entityLinkProposalWorker(),
              investigationSummaryWorker()
          ));
          return def;
      }

      private static Capability cap(String name) {
          return Capability.builder().name(name).inputSchema(".").outputSchema(".").build();
      }

      private Worker entityResolutionWorker() {
          return Worker.builder()
              .name("oversight-entity-resolution-agent")
              .capabilities(List.of(cap("entity-resolution")))
              .function((Map<String, Object> input) -> {
                  @SuppressWarnings("unchecked")
                  Map<String, Object> tx = (Map<String, Object>) input.get("transaction");
                  String flagReason = tx != null ? (String) tx.getOrDefault("flagReason", "") : "";
                  boolean isPep = flagReason != null && flagReason.contains("PEP");
                  String txId = tx != null ? String.valueOf(tx.getOrDefault("id", "unknown")) : "unknown";
                  return WorkerResult.of(Map.of(
                      "entityId", "entity-" + txId,
                      "ownershipChain", isPep ? "Direct → PEP Principal" : "Direct → Corporate Entity",
                      "entityType", isPep ? "PEP" : "CORPORATE",
                      "riskScore", isPep ? 0.87 : 0.35
                  ));
              })
              .build();
      }

      private Worker entityLinkProposalWorker() {
          return Worker.builder()
              .name("oversight-entity-link-proposal-agent")
              .capabilities(List.of(cap("entity-link-proposal")))
              .function((Map<String, Object> input) -> {
                  @SuppressWarnings("unchecked")
                  Map<String, Object> entityResolution =
                      (Map<String, Object>) input.get("entityResolution");
                  String entityType = entityResolution != null
                      ? (String) entityResolution.getOrDefault("entityType", "UNKNOWN") : "UNKNOWN";
                  double riskScore = entityResolution != null
                      ? ((Number) entityResolution.getOrDefault("riskScore", 0.0)).doubleValue() : 0.0;
                  String ownershipChain = entityResolution != null
                      ? (String) entityResolution.getOrDefault("ownershipChain", "") : "";

                  Map<String, Object> output = Map.of(
                      "proposedLink", entityType + " → investigation graph",
                      "entityType", entityType,
                      "riskScore", riskScore,
                      "confirmed", false
                  );
                  Map<String, Object> context = Map.of(
                      "entityType", entityType,
                      "riskScore", riskScore,
                      "ownershipChain", ownershipChain
                  );
                  return WorkerResult.of(output,
                      PlannedAction.of(
                          "Entity network link proposed: " + entityType,
                          AmlActionType.ENTITY_LINK_CREATION.actionType(),
                          context));
              })
              .build();
      }

      private Worker investigationSummaryWorker() {
          return Worker.builder()
              .name("oversight-investigation-summary-agent")
              .capabilities(List.of(cap("investigation-summary")))
              .function((Map<String, Object> input) -> {
                  @SuppressWarnings("unchecked")
                  Map<String, Object> link = (Map<String, Object>) input.get("entityLinkProposal");
                  String entityType = link != null
                      ? (String) link.getOrDefault("entityType", "UNKNOWN") : "UNKNOWN";
                  return WorkerResult.of(Map.of(
                      "summary", "Entity link confirmed for " + entityType + " entity",
                      "status", "LINK_CONFIRMED"
                  ));
              })
              .build();
      }
  }
  ```

- [ ] **Step 5: Create AmlOversightCoordinator.java**

  Create `app/src/main/java/io/casehub/aml/engine/AmlOversightCoordinator.java`:

  ```java
  package io.casehub.aml.engine;

  import io.casehub.aml.domain.SuspiciousTransaction;
  import com.fasterxml.jackson.core.type.TypeReference;
  import com.fasterxml.jackson.databind.ObjectMapper;
  import jakarta.enterprise.context.ApplicationScoped;
  import jakarta.inject.Inject;
  import org.jboss.logging.Logger;
  import java.util.Map;
  import java.util.UUID;
  import java.util.concurrent.TimeUnit;

  /**
   * Layer 9 coordinator — starts an oversight investigation case and returns its UUID.
   *
   * <p>Minimal: no memory query (no prior context for oversight investigations), no ledger write
   * (the gate WorkItem created by ActionGateWorkItemHandler is the audit artefact for this layer).
   */
  @ApplicationScoped
  public class AmlOversightCoordinator {

      private static final Logger LOG = Logger.getLogger(AmlOversightCoordinator.class);
      private static final TypeReference<Map<String, Object>> MAP_TYPE = new TypeReference<>() {};
      private static final int CASE_START_TIMEOUT_SECONDS = 5;

      @Inject AmlOversightCaseHub caseHub;
      @Inject ObjectMapper objectMapper;

      public UUID startInvestigation(SuspiciousTransaction transaction) {
          Map<String, Object> initialContext = Map.of(
              "transaction", objectMapper.convertValue(transaction, MAP_TYPE));
          try {
              UUID caseId = caseHub.startCase(initialContext)
                  .toCompletableFuture()
                  .get(CASE_START_TIMEOUT_SECONDS, TimeUnit.SECONDS);
              LOG.infof("Oversight investigation started: caseId=%s txId=%s", caseId, transaction.id());
              return caseId;
          } catch (Exception e) {
              LOG.errorf(e, "Failed to start oversight investigation for transaction %s", transaction.id());
              throw new RuntimeException("Failed to start oversight investigation", e);
          }
      }
  }
  ```

- [ ] **Step 6: Create AmlLayer9Resource.java**

  Create `app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java`:

  ```java
  package io.casehub.aml.engine;

  import io.casehub.aml.domain.SuspiciousTransaction;
  import io.casehub.aml.trust.AmlWorkerDecisionRepository;
  import jakarta.enterprise.context.ApplicationScoped;
  import jakarta.inject.Inject;
  import jakarta.ws.rs.*;
  import jakarta.ws.rs.core.MediaType;
  import jakarta.ws.rs.core.Response;
  import java.util.Map;
  import java.util.UUID;

  /**
   * Layer 9 REST resource — oversight gate investigation.
   *
   * <p>Completion is determined by the presence of a WorkerDecisionEntry with
   * capabilityTag="investigation-summary" (same pattern as Layer 6 checking for "sar-drafting").
   */
  @Path("/api/layer9/investigations")
  @ApplicationScoped
  @Produces(MediaType.APPLICATION_JSON)
  @Consumes(MediaType.APPLICATION_JSON)
  public class AmlLayer9Resource {

      @Inject AmlOversightCoordinator coordinator;
      @Inject AmlWorkerDecisionRepository workerDecisionRepo;

      @POST
      public Response startInvestigation(SuspiciousTransaction transaction) {
          UUID caseId = coordinator.startInvestigation(transaction);
          return Response.accepted(Map.of("caseId", caseId)).build();
      }

      @GET
      @Path("/{caseId}")
      public Response getInvestigation(@PathParam("caseId") UUID caseId) {
          boolean completed = workerDecisionRepo
              .findLatestByCaseIdAndCapability(caseId, "investigation-summary")
              .isPresent();
          String status = completed ? "completed" : "in-progress";
          return Response.ok(Map.of("caseId", caseId, "status", status)).build();
      }
  }
  ```

- [ ] **Step 7: Run tests**

  ```bash
  mvn test -pl app -am -f /Users/mdproctor/claude/casehub/aml/pom.xml \
    -Dtest=AmlLayer9ActionGateTest -Dsurefire.failIfNoSpecifiedTests=false
  ```

  Expected: Both tests pass. If `gate_fires_for_pep_entity` hangs at "await completed" after gate approval, check for a startup warning about `casehub-engine-work-adapter` not found (means Task 1 Jandex entries may not have taken effect — verify the index-dependency keys are correct). If `gate_not_fired_for_low_risk_corporate` fails with a timeout, verify entity-resolution correctly sets riskScore=0.35 for non-PEP `flagReason`.

- [ ] **Step 8: Run full test suite to confirm no regressions**

  ```bash
  mvn test -pl app -am -f /Users/mdproctor/claude/casehub/aml/pom.xml
  ```

  Expected: All 142+ tests pass (the two new tests push the count to 144+).

- [ ] **Step 9: Commit**

  ```bash
  git -C /Users/mdproctor/claude/casehub/aml add \
    app/src/main/resources/aml/aml-oversight-investigation.yaml \
    app/src/main/java/io/casehub/aml/engine/AmlOversightCaseHub.java \
    app/src/main/java/io/casehub/aml/engine/AmlOversightCoordinator.java \
    app/src/main/java/io/casehub/aml/engine/AmlLayer9Resource.java \
    app/src/test/java/io/casehub/aml/engine/AmlLayer9ActionGateTest.java
  git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#42): Layer 9 oversight harness — YAML, workers, coordinator, resource, integration test

  Refs #42"
  ```

---

## Task 5: Issue #57 — Partial unique index DDL

**Files:**
- Create: `docs/sql/V2011__UQ_TRUST_ATTEST_CASE_CAP_RECONSTRUCTED.sql`

- [ ] **Step 1: Create the SQL file**

  ```bash
  mkdir -p /Users/mdproctor/claude/casehub/aml/docs/sql
  ```

  Create `docs/sql/V2011__UQ_TRUST_ATTEST_CASE_CAP_RECONSTRUCTED.sql`:

  ```sql
  -- PostgreSQL only — H2 does not support partial unique indexes even in MODE=PostgreSQL.
  -- Apply manually on the production PostgreSQL database after V2009 has run.
  -- Do NOT place in db/migration/ paths — Flyway will fail on H2.
  --
  -- The AmlAttestationReconciler catches PersistenceException(ConstraintViolationException)
  -- to handle idempotent duplicate writes — this catch is dead code in H2 tests but
  -- active on production PostgreSQL once this index is applied.
  CREATE UNIQUE INDEX IF NOT EXISTS UQ_TRUST_ATTEST_CASE_CAP_RECONSTRUCTED
      ON aml_trust_routing_attestation (investigation_case_id, capability_tag)
      WHERE reconstructed = TRUE;
  ```

- [ ] **Step 2: Commit and close issue**

  ```bash
  git -C /Users/mdproctor/claude/casehub/aml add docs/sql/V2011__UQ_TRUST_ATTEST_CASE_CAP_RECONSTRUCTED.sql
  git -C /Users/mdproctor/claude/casehub/aml commit -m "ops(#57): add production-only partial unique index for reconstructed attestations

  H2 does not support partial unique indexes (GE-20260609-bc8704).
  Apply manually on production PostgreSQL after V2009 has run.

  Closes #57"
  ```

---

## Task 6: File sar-drafting design flaw as GitHub issue

- [ ] **Step 1: File the issue**

  ```bash
  gh issue create --repo casehubio/aml \
    --title "refactor: separate sar-drafting from compliance-review-opening for SAR_FILING oversight gate" \
    --body "$(cat <<'EOF'
  ## Context

  Refs aml#42 (AmlActionRiskClassifier — Layer 9 Oversight Gate).

  The current \`sarDraftingWorkerJunior\` and \`sarDraftingWorkerSenior\` in \`AmlInvestigationCaseHub\`
  call \`complianceReviewLifecycle.openReview()\` unconditionally as a side effect. This creates
  a compliance officer WorkItem before any MLRO gate can run.

  ## Problem

  The correct design for \`SAR_FILING\` oversight is:

  1. \`sar-drafting\` — pure analysis: synthesise SAR narrative, declare \`PlannedAction(SAR_FILING)\`
  2. [MLRO gate fires via AmlActionRiskClassifier — GateRequired("SAR submission to regulator...")]
  3. After MLRO approval: \`compliance-review-opening\` worker calls \`openReview()\`, creating the compliance officer WorkItem

  Currently, step 3 happens inside step 1, before any gate. If a gate is added to sar-drafting workers, \`openReview()\` would create a WorkItem even if the MLRO subsequently rejects the SAR.

  ## What to do

  - Split both SAR drafting workers into pure-narrative workers (no \`openReview()\` call)
  - Add \`PlannedAction(SAR_FILING)\` to both SAR drafting workers
  - Add new \`compliance-review-opening\` capability + YAML binding + worker that calls \`openReview()\`
  - Update YAML to fire \`compliance-review-opening\` after \`sar-drafting\` (which only fires after gate approval)
  - Update Layer 5–8 integration tests: they must now approve the MLRO gate before draining to \`completed\`

  ## AmlActionType policy

  \`AML_ACTION_TYPE.SAR_FILING\`: ALWAYS, reversible=false, groups=["aml-mlro"]
  Defined in \`AmlActionType.java\` (shipped in aml#42).
  EOF
  )"
  ```

  Note the returned issue number. If the new issue is e.g. `#58`, record it here for LAYER-LOG.

---

## Task 7: Update LAYER-LOG.md and ARC42STORIES.MD

- [ ] **Step 1: Add Layer 9 entry to LAYER-LOG.md**

  Open `LAYER-LOG.md` in the project repo and add a new section at the top (after the
  Layer 8 entry). Replace `#58` with the actual issue number from Task 6:

  ```markdown
  ## Layer 9: ActionRiskClassifier — Oversight Gate (aml#42)

  **Integrated:** casehub-engine-work-adapter (ActionGateWorkItemHandler, WorkItemLifecycleAdapter, ActionGateCompletionApplier)

  **New files:**
  - `api/…/AmlGroups.java` — group name constants
  - `api/…/AmlActionType.java` — enum with GatePolicy, groups, reason, scope per consequential action type
  - `app/routing/AmlActionRiskClassifier.java` — @RiskClassifier classifier; fail-closed on missing context for known types
  - `app/resources/aml/aml-oversight-investigation.yaml` — 3-step case definition (entity-resolution → entity-link-proposal → investigation-summary)
  - `app/engine/AmlOversightCaseHub.java` — YamlCaseHub with entity-link-proposal declaring PlannedAction(ENTITY_LINK_CREATION)
  - `app/engine/AmlOversightCoordinator.java` — minimal coordinator; no ledger write
  - `app/engine/AmlLayer9Resource.java` — POST + GET /api/layer9/investigations
  - `docs/sql/V2011__UQ_TRUST_ATTEST_CASE_CAP_RECONSTRUCTED.sql` — manual PostgreSQL DDL (closes aml#57)

  **Key design decisions:**
  - Layer 9 is a dedicated oversight harness — existing Layer 1–8 workers unchanged; no bypass classifiers needed
  - AmlActionType encodes gate policy, reversibility, candidateGroups, reason per action type (pure Java, api module)
  - Fail-closed: known action type + missing context → GateRequired("Risk assessment unavailable")
  - ENTITY_LINK_CREATION demonstrated in harness: PEP or riskScore ≥ 0.8 → gate; low-risk CORPORATE → autonomous
  - casehub-engine-work-adapter added as explicit compile dep (was not transitive — comment corrected)
  - Deferred: sar-drafting/compliance-review-opening split tracked in aml#58

  **Tests added:** AmlActionTypeTest (21), AmlActionRiskClassifierTest (~28), AmlLayer9ActionGateTest (2)
  ```

- [ ] **Step 2: Add Layer 9 row to ARC42STORIES.MD §4 layer taxonomy table**

  In `ARC42STORIES.MD`, find the layer taxonomy table in §4. The table currently has rows
  for Layers 1–8. Add after the Layer 8 row:

  ```
  | casehub-engine-work-adapter | `casehub-engine-work-adapter` | L9 | ✅ complete |
  ```

  The section header for Layer 9 in §9.4 (Layer Entries) should also be added:

  ```markdown
  ### Layer 9 — ActionRiskClassifier Oversight Gate (aml#42)

  **Foundation module:** casehub-engine-work-adapter
  **Status:** ✅ complete

  Implements `ActionRiskClassifier` SPI with five AML consequential action types.
  New Layer 9 harness (YAML + workers + resource) demonstrates the oversight gate:
  PEP entity link creation requires human approval before the investigation summary
  can write. Low-risk CORPORATE cases proceed autonomously. See LAYER-LOG.md §Layer 9.
  ```

- [ ] **Step 3: Commit documentation**

  ```bash
  git -C /Users/mdproctor/claude/casehub/aml add LAYER-LOG.md ARC42STORIES.MD
  git -C /Users/mdproctor/claude/casehub/aml commit -m "docs(#42): Layer 9 entry in LAYER-LOG.md and ARC42STORIES.MD

  Refs #42"
  ```

---

## Self-Review Checklist

Run mentally before declaring done:

- [ ] `AmlActionType.fromActionType(null)` → `Optional.empty()` ✓ (null guard in method)
- [ ] `AmlActionType.expiresIn()` returns `null` ✓ (tests cover this)
- [ ] `AmlActionRiskClassifier` — ENTITY_LINK_CREATION with PEP and riskScore=0.2 → GateRequired ✓ (PEP check before riskScore)
- [ ] `AmlActionRiskClassifier` — null context → missingContext() → GateRequired ✓
- [ ] YAML investigation-summary binding: `when: ".entityLinkProposal != null and .investigationSummary == null"` ✓ (fires only after gate approved/autonomous)
- [ ] Layer 9 GET completion: checks for WorkerDecisionEntry with capabilityTag="investigation-summary" ✓
- [ ] Gate WorkItem query uses default-PU EntityManager (no unitName) — WorkItem is in `io.casehub.work.runtime.model` on default datasource ✓
- [ ] `completeFromSystem` fires both `fire()` and `fireAsync()` — `@ObservesAsync WorkItemLifecycleAdapter` receives the async event → gate resumes ✓ (confirmed from decompiled source L246-247)
- [ ] `casehub-engine-work-adapter` in pom.xml compile scope ✓
- [ ] Jandex entries for work-adapter + blackboard in both test and prod application.properties ✓
- [ ] Spec coverage: all five action types classifiable ✓ / Layer 9 harness ✓ / #57 SQL ✓ / sar-drafting issue filed ✓ / LAYER-LOG + ARC42STORIES updated ✓
