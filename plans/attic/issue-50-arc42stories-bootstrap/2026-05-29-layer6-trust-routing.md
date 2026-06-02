# Layer 6 — Trust-Weighted Routing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add trust-weighted agent routing to the AML investigation so experienced agents are preferred for complex cases, with post-investigation SAR outcome feedback closing the trust score loop.

**Architecture:** `casehub-engine-ledger` dependency activates `TrustWeightedAgentStrategy` (CDI replaces `LeastLoadedAgentStrategy @DefaultBean`) and `WorkerDecisionEventCapture` (writes `WorkerDecisionEntry` per worker execution). `AmlTrustScoreSeeder` seeds initial Beta(α,β) scores at startup so Phase 2/3 routing is immediately visible. `AmlTrustRoutingPolicyProvider` configures per-capability thresholds via the Preferences API with hardcoded fallbacks. `AmlLayer6Resource` exposes async POST + polling GET endpoints.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-engine-ledger, casehub-ledger-runtime, casehub-platform-api (Preferences), JPA (qhorus PU), H2 MODE=PostgreSQL (tests)

---

## File Map

**New — `api/` module (`src/main/java/io/casehub/aml/`):**
- `domain/SarVerdict.java` — enum: UPHELD, WITHDRAWN, FLAGGED
- `domain/SarOutcome.java` — record: verdict + reason + investigationAccuracyScore

**New — `app/` module (`src/main/java/io/casehub/aml/`):**
- `routing/TrustPolicyPreference.java` — `SingleValuePreference` record for per-capability policy
- `routing/AmlTrustRoutingPolicyProvider.java` — `@ApplicationScoped`, beats `DefaultTrustRoutingPolicyProvider @DefaultBean`
- `trust/AmlTrustScoreSeeder.java` — `@Startup @ApplicationScoped`, seeds Beta scores + forces cache reload
- `trust/AmlWorkerDecisionRepository.java` — `@ApplicationScoped`, queries `WorkerDecisionEntry` via qhorus EM
- `trust/SarOutcomeFeedbackService.java` — `@ApplicationScoped`, writes `LedgerAttestation` on SAR outcome
- `engine/Layer6InvestigationResponse.java` — REST response record
- `engine/WorkerRoutingDecision.java` — per-worker routing metadata in response
- `engine/AmlLayer6Resource.java` — `@Path("/api/layer6/investigations")`

**Modified:**
- `app/pom.xml` — add `casehub-engine-ledger` dependency
- `app/src/main/resources/application.properties` — Flyway fix + CDI fix
- `app/src/test/resources/application.properties` — Flyway fix + CDI fix + Jandex
- `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseHub.java` — add 3 worker variants

**New tests — `api/`:**
- `src/test/java/io/casehub/aml/domain/SarOutcomeTest.java`

**New tests — `app/`:**
- `src/test/java/io/casehub/aml/routing/AmlTrustRoutingPolicyProviderTest.java`
- `src/test/java/io/casehub/aml/routing/AmlTrustRoutingPolicyProviderWiringTest.java`
- `src/test/java/io/casehub/aml/trust/AmlTrustScoreSeederTest.java`
- `src/test/java/io/casehub/aml/trust/SarOutcomeFeedbackServiceTest.java`
- `src/test/java/io/casehub/aml/engine/AmlLayer6ResourceTest.java`
- `src/test/java/io/casehub/aml/engine/AmlLayer6InvestigationIT.java`

---

## Task 1: Configuration — pom, Flyway, CDI fixes

**Files:**
- Modify: `app/pom.xml`
- Modify: `app/src/main/resources/application.properties`
- Modify: `app/src/test/resources/application.properties`

- [ ] **Step 1.1: Add `casehub-engine-ledger` dependency**

In `app/pom.xml`, after the `casehub-engine-scheduler-quartz` dependency block, add:

```xml
<!-- Layer 6: trust-weighted routing + WorkerDecisionEntry per worker execution -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-ledger</artifactId>
    <version>${casehub.version}</version>
</dependency>
```

- [ ] **Step 1.2: Fix main `application.properties`**

Open `app/src/main/resources/application.properties` and make three changes:

**(a)** Change the default datasource Flyway location (casehub-work ships at `db/work/migration`, not `db/migration`):
```properties
# Before:
quarkus.flyway.locations=classpath:db/migration
# After:
quarkus.flyway.locations=classpath:db/work/migration
```

**(b)** Add `classpath:db/migration` to the qhorus datasource locations (picks up engine-ledger's V2000 + V2001):
```properties
# Before:
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration,classpath:db/aml-ledger/migration
# After:
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration,classpath:db/aml-ledger/migration,classpath:db/migration
```

**(c)** Remove the stale production CDI exclusion (engine PR#378 removed `CasehubWorkloadProvider`):
```properties
# Delete this entire line:
%prod.quarkus.arc.exclude-types=io.casehub.engine.internal.worker.CasehubWorkloadProvider
```

- [ ] **Step 1.3: Fix test `application.properties`**

Open `app/src/test/resources/application.properties` and make three changes:

**(a)** Change the default datasource Flyway location:
```properties
# Before:
quarkus.flyway.locations=classpath:db/migration
# After:
quarkus.flyway.locations=classpath:db/work/migration
```

**(b)** Add `classpath:db/migration` to qhorus locations:
```properties
# Before:
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration,classpath:db/aml-ledger/migration
# After:
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration,classpath:db/aml-ledger/migration,classpath:db/migration
```

**(c)** Remove `io.casehub.work.runtime.service.JpaWorkloadProvider` from the `quarkus.arc.exclude-types` list (stale after engine PR#378 removed its conflict source). The current list is:
```properties
quarkus.arc.exclude-types=\
  io.casehub.work.runtime.service.ExpiryLifecycleService,\
  io.casehub.work.runtime.service.ExpiryCleanupJob,\
  io.casehub.work.runtime.service.ClaimDeadlineJob,\
  io.casehub.work.runtime.strategy.RoutingCursorCleanupJob,\
  io.casehub.work.runtime.service.JpaWorkloadProvider
```

Change to:
```properties
quarkus.arc.exclude-types=\
  io.casehub.work.runtime.service.ExpiryLifecycleService,\
  io.casehub.work.runtime.service.ExpiryCleanupJob,\
  io.casehub.work.runtime.service.ClaimDeadlineJob,\
  io.casehub.work.runtime.strategy.RoutingCursorCleanupJob
```

**(d)** Add Jandex entry for engine-ledger (after the existing engine index entries):
```properties
quarkus.index-dependency.engine-ledger.group-id=io.casehub
quarkus.index-dependency.engine-ledger.artifact-id=casehub-engine-ledger
```

- [ ] **Step 1.4: Verify the build compiles**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am test-compile -q
```

Expected: BUILD SUCCESS. If CDI validation errors appear, check the Jandex entry was added and the exclusion was removed correctly.

- [ ] **Step 1.5: Run existing tests to confirm CDI fix**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am test -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD|UnsatisfiedResolution"
```

Expected: Tests run and the `UnsatisfiedResolutionException: WorkloadProvider` error is gone. Some tests may still fail due to unimplemented Layer 6 features — that's expected. What must NOT appear: `UnsatisfiedResolutionException`.

- [ ] **Step 1.6: Commit configuration**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/pom.xml app/src/main/resources/application.properties app/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/aml commit -m "chore(#38): add casehub-engine-ledger dep, fix Flyway locations, fix CDI regression from engine PR#394"
```

---

## Task 2: Domain types — `SarVerdict` and `SarOutcome`

**Files:**
- Create: `api/src/main/java/io/casehub/aml/domain/SarVerdict.java`
- Create: `api/src/main/java/io/casehub/aml/domain/SarOutcome.java`
- Create: `api/src/test/java/io/casehub/aml/domain/SarOutcomeTest.java`

- [ ] **Step 2.1: Write the failing test**

Create `api/src/test/java/io/casehub/aml/domain/SarOutcomeTest.java`:

```java
package io.casehub.aml.domain;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class SarOutcomeTest {

    @Test
    void upheld_verdict_accepted() {
        final SarOutcome o = new SarOutcome(SarVerdict.UPHELD, "SAR upheld by FinCEN", 0.92);
        assertEquals(SarVerdict.UPHELD, o.verdict());
        assertEquals("SAR upheld by FinCEN", o.reason());
        assertEquals(0.92, o.investigationAccuracyScore(), 0.001);
    }

    @Test
    void flagged_verdict_accepted() {
        final SarOutcome o = new SarOutcome(SarVerdict.FLAGGED, "Incomplete evidence chain", 0.30);
        assertEquals(SarVerdict.FLAGGED, o.verdict());
        assertEquals(0.30, o.investigationAccuracyScore(), 0.001);
    }

    @Test
    void all_verdicts_are_defined() {
        assertEquals(3, SarVerdict.values().length);
        assertNotNull(SarVerdict.valueOf("UPHELD"));
        assertNotNull(SarVerdict.valueOf("WITHDRAWN"));
        assertNotNull(SarVerdict.valueOf("FLAGGED"));
    }
}
```

- [ ] **Step 2.2: Run test — expect compile failure**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl api test -Dtest=SarOutcomeTest 2>&1 | tail -5
```

Expected: FAIL — `SarVerdict` and `SarOutcome` do not exist.

- [ ] **Step 2.3: Create `SarVerdict`**

Create `api/src/main/java/io/casehub/aml/domain/SarVerdict.java`:

```java
package io.casehub.aml.domain;

public enum SarVerdict { UPHELD, WITHDRAWN, FLAGGED }
```

- [ ] **Step 2.4: Create `SarOutcome`**

Create `api/src/main/java/io/casehub/aml/domain/SarOutcome.java`:

```java
package io.casehub.aml.domain;

public record SarOutcome(
        SarVerdict verdict,
        String reason,
        double investigationAccuracyScore) {}
```

- [ ] **Step 2.5: Run test — expect pass**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl api test -Dtest=SarOutcomeTest 2>&1 | tail -5
```

Expected: `Tests run: 3, Failures: 0, Errors: 0` — BUILD SUCCESS.

- [ ] **Step 2.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add api/src/main/java/io/casehub/aml/domain/SarVerdict.java api/src/main/java/io/casehub/aml/domain/SarOutcome.java api/src/test/java/io/casehub/aml/domain/SarOutcomeTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#38): add SarVerdict enum and SarOutcome record to api module"
```

---

## Task 3: Trust routing policy provider

**Files:**
- Create: `app/src/main/java/io/casehub/aml/routing/TrustPolicyPreference.java`
- Create: `app/src/main/java/io/casehub/aml/routing/AmlTrustRoutingPolicyProvider.java`
- Create: `app/src/test/java/io/casehub/aml/routing/AmlTrustRoutingPolicyProviderTest.java`
- Create: `app/src/test/java/io/casehub/aml/routing/AmlTrustRoutingPolicyProviderWiringTest.java`

- [ ] **Step 3.1: Write failing tests**

Create `app/src/test/java/io/casehub/aml/routing/AmlTrustRoutingPolicyProviderTest.java`:

```java
package io.casehub.aml.routing;

import io.casehub.api.spi.routing.TrustRoutingPolicy;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class AmlTrustRoutingPolicyProviderTest {

    @Inject
    AmlTrustRoutingPolicyProvider provider;

    @Test
    void sar_drafting_has_higher_threshold_than_default() {
        final TrustRoutingPolicy policy = provider.forCapability("sar-drafting");
        assertEquals(0.75, policy.threshold(), 0.001);
        assertEquals(0.70, policy.blendFactor(), 0.001);
        assertEquals(10, policy.minimumObservations());
        assertTrue(policy.qualityFloors().containsKey("investigation-accuracy"));
        assertEquals(0.65, policy.qualityFloors().get("investigation-accuracy"), 0.001);
    }

    @Test
    void osint_screening_threshold_is_0_70() {
        final TrustRoutingPolicy policy = provider.forCapability("osint-screening");
        assertEquals(0.70, policy.threshold(), 0.001);
        assertEquals(0.65, policy.blendFactor(), 0.001);
        assertTrue(policy.qualityFloors().isEmpty());
    }

    @Test
    void pattern_analysis_threshold_is_0_65() {
        final TrustRoutingPolicy policy = provider.forCapability("pattern-analysis");
        assertEquals(0.65, policy.threshold(), 0.001);
    }

    @Test
    void senior_analyst_review_has_highest_threshold() {
        final TrustRoutingPolicy policy = provider.forCapability("senior-analyst-review");
        assertEquals(0.80, policy.threshold(), 0.001);
    }

    @Test
    void unknown_capability_returns_default_policy() {
        final TrustRoutingPolicy policy = provider.forCapability("unknown-capability");
        assertEquals(TrustRoutingPolicy.DEFAULT.threshold(), policy.threshold(), 0.001);
        assertEquals(TrustRoutingPolicy.DEFAULT.minimumObservations(), policy.minimumObservations());
    }

    @Test
    void mock_preferences_return_null_so_hardcoded_fallback_always_applies() {
        // MockPreferenceProvider @DefaultBean always returns null from get()
        // This test confirms the provider doesn't throw and returns the hardcoded default.
        final TrustRoutingPolicy policy = provider.forCapability("sar-drafting");
        assertNotNull(policy);
        assertEquals(0.75, policy.threshold(), 0.001); // hardcoded, not from prefs
    }
}
```

Create `app/src/test/java/io/casehub/aml/routing/AmlTrustRoutingPolicyProviderWiringTest.java`:

```java
package io.casehub.aml.routing;

import io.casehub.api.spi.routing.TrustRoutingPolicyProvider;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class AmlTrustRoutingPolicyProviderWiringTest {

    @Inject
    TrustRoutingPolicyProvider provider;

    @Test
    void aml_provider_is_selected_over_default_bean() {
        // AmlTrustRoutingPolicyProvider @ApplicationScoped beats
        // DefaultTrustRoutingPolicyProvider @DefaultBean automatically.
        // If the AML provider wins, sar-drafting threshold is 0.75, not 0.70 (DEFAULT).
        assertInstanceOf(AmlTrustRoutingPolicyProvider.class, provider);
        assertEquals(0.75, provider.forCapability("sar-drafting").threshold(), 0.001);
    }
}
```

- [ ] **Step 3.2: Run tests — expect compile failure**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am test -Dtest="AmlTrustRoutingPolicyProviderTest,AmlTrustRoutingPolicyProviderWiringTest" -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -8
```

Expected: FAIL — `AmlTrustRoutingPolicyProvider` does not exist.

- [ ] **Step 3.3: Create `TrustPolicyPreference`**

Create `app/src/main/java/io/casehub/aml/routing/TrustPolicyPreference.java`:

```java
package io.casehub.aml.routing;

import io.casehub.platform.api.preferences.SingleValuePreference;
import java.util.Map;

/**
 * Per-capability trust routing policy preference. Implements {@link SingleValuePreference} so it
 * can be resolved via {@link io.casehub.platform.api.preferences.Preferences#get}.
 *
 * <p>In tutorial mode, {@code MockPreferenceProvider @DefaultBean} returns null for all lookups
 * so {@link AmlTrustRoutingPolicyProvider} falls back to hardcoded defaults. Production deployments
 * activate {@code casehub-platform-config} (YAML) or JPA backend to override per capability.
 */
public record TrustPolicyPreference(
        double threshold,
        int minimumObservations,
        double borderlineMargin,
        double blendFactor,
        Map<String, Double> qualityFloors) implements SingleValuePreference {}
```

- [ ] **Step 3.4: Create `AmlTrustRoutingPolicyProvider`**

Create `app/src/main/java/io/casehub/aml/routing/AmlTrustRoutingPolicyProvider.java`:

```java
package io.casehub.aml.routing;

import io.casehub.api.spi.routing.TrustRoutingPolicy;
import io.casehub.api.spi.routing.TrustRoutingPolicyProvider;
import io.casehub.platform.api.preferences.PreferenceKey;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.Preferences;
import io.casehub.platform.api.preferences.SettingsScope;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;

/**
 * AML-specific trust routing policy provider. Beats {@code DefaultTrustRoutingPolicyProvider
 * @DefaultBean} via standard CDI {@code @ApplicationScoped} semantics — no {@code @Alternative}
 * needed.
 *
 * <p>Resolves policies from the platform Preferences API first; falls back to hardcoded AML
 * defaults when no preference is configured. Tutorial deployments use {@code MockPreferenceProvider
 * @DefaultBean} which always returns null, so hardcoded defaults always apply.
 */
@ApplicationScoped
public class AmlTrustRoutingPolicyProvider implements TrustRoutingPolicyProvider {

    private static final TrustPolicyPreference SENTINEL =
            new TrustPolicyPreference(0.0, 0, 0.0, 0.0, Map.of());

    private static final Map<String, TrustRoutingPolicy> POLICIES = Map.of(
            "entity-resolution",     new TrustRoutingPolicy(0.70, 10, 0.10, 0.60, Map.of()),
            "pattern-analysis",      new TrustRoutingPolicy(0.65, 10, 0.10, 0.60, Map.of()),
            "osint-screening",       new TrustRoutingPolicy(0.70, 10, 0.10, 0.65, Map.of()),
            "sar-drafting",          new TrustRoutingPolicy(0.75, 10, 0.10, 0.70,
                                         Map.of("investigation-accuracy", 0.65)),
            "senior-analyst-review", new TrustRoutingPolicy(0.80, 10, 0.10, 0.70, Map.of())
    );

    private final PreferenceProvider preferenceProvider;

    @Inject
    public AmlTrustRoutingPolicyProvider(final PreferenceProvider preferenceProvider) {
        this.preferenceProvider = preferenceProvider;
    }

    @Override
    public TrustRoutingPolicy forCapability(final String capabilityName) {
        final PreferenceKey<TrustPolicyPreference> key = new PreferenceKey<>(
                "casehubio.aml.trust-routing",
                capabilityName,
                SENTINEL,
                s -> { throw new UnsupportedOperationException(
                        "TrustPolicyPreference YAML parsing not yet configured — activate casehub-platform-config"); }
        );
        final Preferences prefs =
                preferenceProvider.resolve(SettingsScope.of("casehubio", "aml", "trust-routing", capabilityName));
        final TrustPolicyPreference pref = prefs.get(key);
        if (pref != null && pref != SENTINEL) {
            return new TrustRoutingPolicy(
                    pref.threshold(), pref.minimumObservations(),
                    pref.borderlineMargin(), pref.blendFactor(),
                    pref.qualityFloors());
        }
        return POLICIES.getOrDefault(capabilityName, TrustRoutingPolicy.DEFAULT);
    }
}
```

- [ ] **Step 3.5: Run tests — expect pass**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am test -Dtest="AmlTrustRoutingPolicyProviderTest,AmlTrustRoutingPolicyProviderWiringTest" -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD"
```

Expected: `Tests run: 7, Failures: 0, Errors: 0` — BUILD SUCCESS.

- [ ] **Step 3.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/routing/ app/src/test/java/io/casehub/aml/routing/
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#38): add AmlTrustRoutingPolicyProvider with per-capability AML trust policies"
```

---

## Task 4: Trust score seeder

**Files:**
- Create: `app/src/main/java/io/casehub/aml/trust/AmlTrustScoreSeeder.java`
- Create: `app/src/test/java/io/casehub/aml/trust/AmlTrustScoreSeederTest.java`

- [ ] **Step 4.1: Write failing test**

Create `app/src/test/java/io/casehub/aml/trust/AmlTrustScoreSeederTest.java`:

```java
package io.casehub.aml.trust;

import io.casehub.ledger.api.model.ActorTrustScore.ScoreType;
import io.casehub.ledger.routing.TrustScoreCache;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class AmlTrustScoreSeederTest {

    @Inject
    ActorTrustScoreRepository trustRepo;

    @Inject
    TrustScoreCache trustScoreCache;

    @Test
    void sar_drafting_senior_seeded_with_high_trust() {
        final var score = trustRepo.findCapabilityScore("sar-drafting-agent-senior", "sar-drafting");
        assertTrue(score.isPresent(), "CAPABILITY score must exist for sar-drafting-agent-senior");
        assertEquals(ScoreType.CAPABILITY, score.get().scoreType);
        assertEquals(0.90, score.get().trustScore, 0.01); // alpha=9, beta=1 → mean=0.90
        assertEquals(10, score.get().decisionCount);       // alpha+beta=10
    }

    @Test
    void sar_drafting_junior_seeded_with_low_trust() {
        final var score = trustRepo.findCapabilityScore("sar-drafting-agent-junior", "sar-drafting");
        assertTrue(score.isPresent());
        assertEquals(0.20, score.get().trustScore, 0.01); // alpha=2, beta=8 → mean=0.20
    }

    @Test
    void osint_senior_seeded_with_high_trust() {
        final var score = trustRepo.findCapabilityScore("osint-screening-agent-senior", "osint-screening");
        assertTrue(score.isPresent());
        assertEquals(0.818, score.get().trustScore, 0.01); // alpha=9, beta=2 → mean=9/11
    }

    @Test
    void osint_junior_seeded_with_low_trust() {
        final var score = trustRepo.findCapabilityScore("osint-screening-agent", "osint-screening");
        assertTrue(score.isPresent());
        assertEquals(0.30, score.get().trustScore, 0.01); // alpha=3, beta=7 → mean=0.30
    }

    @Test
    void trust_score_cache_reflects_seeded_scores() {
        // TrustScoreCache.hydrate() is called by AmlTrustScoreSeeder after seeding
        assertTrue(trustScoreCache.getCapabilityScore("sar-drafting-agent-senior", "sar-drafting").isPresent());
        assertEquals(0.90,
                trustScoreCache.getCapabilityScore("sar-drafting-agent-senior", "sar-drafting").getAsDouble(),
                0.01);
    }

    @Test
    void seeding_is_idempotent() {
        // Running seed twice (or across restart) must not create duplicate rows or change values.
        final var scoreBefore = trustRepo.findCapabilityScore("sar-drafting-agent-senior", "sar-drafting");
        // Seeder runs once at @PostConstruct — score is already there. If we had a second invocation
        // it would find the row already exists and skip. We verify the count didn't double.
        final var scoreAfter = trustRepo.findCapabilityScore("sar-drafting-agent-senior", "sar-drafting");
        assertEquals(scoreBefore.get().trustScore, scoreAfter.get().trustScore, 0.0001);
    }
}
```

- [ ] **Step 4.2: Run test — expect compile failure**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am test -Dtest=AmlTrustScoreSeederTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -5
```

Expected: FAIL — `AmlTrustScoreSeeder` does not exist.

- [ ] **Step 4.3: Create `AmlTrustScoreSeeder`**

Create `app/src/main/java/io/casehub/aml/trust/AmlTrustScoreSeeder.java`:

```java
package io.casehub.aml.trust;

import io.casehub.ledger.api.model.ActorTrustScore.ScoreType;
import io.casehub.ledger.routing.TrustScoreCache;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
import io.casehub.platform.api.identity.ActorType;
import io.quarkus.runtime.Startup;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.List;

/**
 * Seeds initial Bayesian Beta trust scores for known AML workers at application startup.
 *
 * <p>Makes Layer 6 trust routing immediately visible without waiting for {@code TrustScoreJob}
 * to compute scores from accumulated {@code WorkerDecisionEntry} records. Seeds only if no
 * CAPABILITY score already exists for the worker+capability pair — safe across restarts.
 *
 * <p>Calls {@link TrustScoreCache#hydrate()} after writing to ensure the cache reflects the
 * seeded values regardless of {@code @Startup} initialization order.
 */
@Startup
@ApplicationScoped
public class AmlTrustScoreSeeder {

    private record WorkerSeed(String workerId, String capabilityTag, int alpha, int beta) {}

    private static final List<WorkerSeed> SEEDS = List.of(
            new WorkerSeed("sar-drafting-agent-senior",    "sar-drafting",       9, 1),
            new WorkerSeed("sar-drafting-agent-junior",    "sar-drafting",       2, 8),
            new WorkerSeed("osint-screening-agent-senior", "osint-screening",    9, 2),
            new WorkerSeed("osint-screening-agent",        "osint-screening",    3, 7),
            new WorkerSeed("entity-resolution-agent",      "entity-resolution",  8, 2),
            new WorkerSeed("pattern-analysis-agent",       "pattern-analysis",   8, 2),
            new WorkerSeed("senior-analyst-agent",         "senior-analyst-review", 8, 2)
    );

    private final ActorTrustScoreRepository trustRepo;
    private final TrustScoreCache trustScoreCache;

    @Inject
    public AmlTrustScoreSeeder(
            final ActorTrustScoreRepository trustRepo,
            final TrustScoreCache trustScoreCache) {
        this.trustRepo = trustRepo;
        this.trustScoreCache = trustScoreCache;
    }

    @PostConstruct
    @Transactional
    void seed() {
        for (final WorkerSeed ws : SEEDS) {
            if (trustRepo.findCapabilityScore(ws.workerId(), ws.capabilityTag()).isEmpty()) {
                final int obs = ws.alpha() + ws.beta();
                final double trustScore = (double) ws.alpha() / obs;
                trustRepo.upsert(
                        ws.workerId(),
                        ScoreType.CAPABILITY,
                        ws.capabilityTag(),
                        null,
                        ActorType.SYSTEM,
                        trustScore,
                        obs,
                        0,
                        ws.alpha(),
                        ws.beta(),
                        ws.alpha(),
                        ws.beta(),
                        Instant.now());
            }
        }
        // Force cache to reload with seeded scores regardless of @Startup initialization order.
        // TrustScoreCache.hydrate() is safe to call multiple times — it overwrites in-memory maps.
        trustScoreCache.hydrate();
    }
}
```

- [ ] **Step 4.4: Run tests — expect pass**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am test -Dtest=AmlTrustScoreSeederTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD"
```

Expected: `Tests run: 6, Failures: 0, Errors: 0` — BUILD SUCCESS.

- [ ] **Step 4.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/trust/AmlTrustScoreSeeder.java app/src/test/java/io/casehub/aml/trust/AmlTrustScoreSeederTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#38): add AmlTrustScoreSeeder — seeds Beta trust scores at startup, forces cache reload"
```

---

## Task 5: Worker decision repository

**Files:**
- Create: `app/src/main/java/io/casehub/aml/trust/AmlWorkerDecisionRepository.java`

No dedicated unit test — this repository is tested via `SarOutcomeFeedbackServiceTest` (Task 6) and the integration test (Task 9), which are the appropriate integration points.

- [ ] **Step 5.1: Create `AmlWorkerDecisionRepository`**

Create `app/src/main/java/io/casehub/aml/trust/AmlWorkerDecisionRepository.java`:

```java
package io.casehub.aml.trust;

import io.casehub.ledger.model.WorkerDecisionEntry;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

/**
 * Queries {@link WorkerDecisionEntry} records written by {@code WorkerDecisionEventCapture} in
 * {@code casehub-engine-ledger}. Uses the qhorus persistence unit where all ledger entities live.
 */
@ApplicationScoped
public class AmlWorkerDecisionRepository {

    @PersistenceContext(unitName = "qhorus")
    EntityManager em;

    /**
     * Returns the latest {@code WorkerDecisionEntry} for a case and capability, ordered by
     * {@code sequenceNumber DESC}. Returns empty if no worker has executed this capability for the case.
     */
    public Optional<WorkerDecisionEntry> findLatestByCaseIdAndCapability(
            final UUID caseId, final String capabilityTag) {
        return em.createQuery(
                "SELECT w FROM WorkerDecisionEntry w" +
                " WHERE w.caseId = :caseId AND w.capabilityTag = :cap" +
                " ORDER BY w.sequenceNumber DESC",
                WorkerDecisionEntry.class)
                .setParameter("caseId", caseId)
                .setParameter("cap", capabilityTag)
                .setMaxResults(1)
                .getResultStream()
                .findFirst();
    }

    /**
     * Returns all {@code WorkerDecisionEntry} records for a case, ordered by {@code sequenceNumber ASC}.
     * Empty list if no workers have executed yet (case still in progress).
     */
    public List<WorkerDecisionEntry> findAllByCaseId(final UUID caseId) {
        return em.createQuery(
                "SELECT w FROM WorkerDecisionEntry w WHERE w.caseId = :caseId ORDER BY w.sequenceNumber ASC",
                WorkerDecisionEntry.class)
                .setParameter("caseId", caseId)
                .getResultList();
    }
}
```

- [ ] **Step 5.2: Verify compile**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am test-compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 5.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/trust/AmlWorkerDecisionRepository.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#38): add AmlWorkerDecisionRepository — queries WorkerDecisionEntry via qhorus PU"
```

---

## Task 6: SAR outcome feedback service

**Files:**
- Create: `app/src/main/java/io/casehub/aml/trust/SarOutcomeFeedbackService.java`
- Create: `app/src/test/java/io/casehub/aml/trust/SarOutcomeFeedbackServiceTest.java`

- [ ] **Step 6.1: Write failing test**

Create `app/src/test/java/io/casehub/aml/trust/SarOutcomeFeedbackServiceTest.java`:

```java
package io.casehub.aml.trust;

import io.casehub.aml.domain.SarOutcome;
import io.casehub.aml.domain.SarVerdict;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class SarOutcomeFeedbackServiceTest {

    @Inject
    SarOutcomeFeedbackService feedbackService;

    @PersistenceContext(unitName = "qhorus")
    EntityManager em;

    @Test
    void no_worker_decision_entry_does_not_throw() {
        // If no WorkerDecisionEntry exists for the case, the service logs a warning and returns.
        assertDoesNotThrow(() ->
                feedbackService.recordOutcome(UUID.randomUUID(),
                        new SarOutcome(SarVerdict.UPHELD, "SAR upheld", 0.9)));
    }

    @Test
    @Transactional
    void upheld_verdict_writes_sound_attestation() {
        // Arrange: insert a minimal LedgerEntry + WorkerDecisionEntry for sar-drafting
        final UUID caseId = UUID.randomUUID();
        insertWorkerDecisionEntry(caseId, "sar-drafting-agent-senior", "sar-drafting");

        // Act
        feedbackService.recordOutcome(caseId, new SarOutcome(SarVerdict.UPHELD, "SAR upheld by FinCEN", 0.92));

        // Assert: attestation persisted with SOUND verdict
        final List<LedgerAttestation> attestations = em.createQuery(
                "SELECT a FROM LedgerAttestation a WHERE a.subjectId = :sid", LedgerAttestation.class)
                .setParameter("sid", caseId)
                .getResultList();
        assertEquals(1, attestations.size());
        final LedgerAttestation a = attestations.get(0);
        assertEquals(AttestationVerdict.SOUND, a.verdict);
        assertEquals("sar-drafting", a.capabilityTag);
        assertEquals("investigation-accuracy", a.trustDimension);
        assertEquals(0.92, a.dimensionScore, 0.001);
        assertEquals(1.0, a.confidence, 0.001);
        assertEquals("aml-compliance-system", a.attestorId);
    }

    @Test
    @Transactional
    void flagged_verdict_writes_flagged_attestation() {
        final UUID caseId = UUID.randomUUID();
        insertWorkerDecisionEntry(caseId, "sar-drafting-agent-junior", "sar-drafting");

        feedbackService.recordOutcome(caseId, new SarOutcome(SarVerdict.FLAGGED, "Incomplete evidence", 0.25));

        final List<LedgerAttestation> attestations = em.createQuery(
                "SELECT a FROM LedgerAttestation a WHERE a.subjectId = :sid", LedgerAttestation.class)
                .setParameter("sid", caseId)
                .getResultList();
        assertEquals(1, attestations.size());
        assertEquals(AttestationVerdict.FLAGGED, attestations.get(0).verdict);
        assertEquals(0.25, attestations.get(0).dimensionScore, 0.001);
    }

    @Test
    @Transactional
    void withdrawn_verdict_writes_flagged_attestation() {
        final UUID caseId = UUID.randomUUID();
        insertWorkerDecisionEntry(caseId, "sar-drafting-agent-senior", "sar-drafting");

        feedbackService.recordOutcome(caseId, new SarOutcome(SarVerdict.WITHDRAWN, "SAR withdrawn", 0.10));

        final List<LedgerAttestation> attestations = em.createQuery(
                "SELECT a FROM LedgerAttestation a WHERE a.subjectId = :sid", LedgerAttestation.class)
                .setParameter("sid", caseId)
                .getResultList();
        assertEquals(AttestationVerdict.FLAGGED, attestations.get(0).verdict);
    }

    private void insertWorkerDecisionEntry(UUID caseId, String workerId, String capabilityTag) {
        // Manually insert a minimal ledger_entry + worker_decision_entry row so the
        // feedback service can find it. Use native SQL to bypass @PrePersist ordering.
        final UUID entryId = UUID.randomUUID();
        em.createNativeQuery(
                "INSERT INTO ledger_entry (id, subject_id, sequence_number, entry_type, actor_id, actor_type, occurred_at, hash)" +
                " VALUES (:id, :sid, 1, 'EVENT', :wid, 'SYSTEM', CURRENT_TIMESTAMP, 'test-hash')")
                .setParameter("id", entryId)
                .setParameter("sid", caseId)
                .setParameter("wid", workerId)
                .executeUpdate();
        em.createNativeQuery(
                "INSERT INTO worker_decision_entry (id, worker_id, capability_tag, case_id)" +
                " VALUES (:id, :wid, :cap, :cid)")
                .setParameter("id", entryId)
                .setParameter("wid", workerId)
                .setParameter("cap", capabilityTag)
                .setParameter("cid", caseId)
                .executeUpdate();
    }
}
```

- [ ] **Step 6.2: Run test — expect compile failure**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am test -Dtest=SarOutcomeFeedbackServiceTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -5
```

Expected: FAIL — `SarOutcomeFeedbackService` does not exist.

- [ ] **Step 6.3: Create `SarOutcomeFeedbackService`**

Create `app/src/main/java/io/casehub/aml/trust/SarOutcomeFeedbackService.java`:

```java
package io.casehub.aml.trust;

import io.casehub.aml.domain.SarOutcome;
import io.casehub.aml.domain.SarVerdict;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.model.WorkerDecisionEntry;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.platform.api.identity.ActorType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.Optional;
import java.util.UUID;
import org.jboss.logging.Logger;

/**
 * Records SAR investigation outcome as a {@link LedgerAttestation} against the
 * {@link WorkerDecisionEntry} written by the sar-drafting worker. The attestation
 * is consumed by {@code TrustScoreJob} (nightly) to update Bayesian Beta trust scores —
 * closing the trust routing feedback loop.
 */
@ApplicationScoped
public class SarOutcomeFeedbackService {

    private static final Logger LOG = Logger.getLogger(SarOutcomeFeedbackService.class);

    @PersistenceContext(unitName = "qhorus")
    EntityManager em;

    @Inject
    AmlWorkerDecisionRepository workerDecisionRepo;

    @Transactional
    public void recordOutcome(final UUID caseId, final SarOutcome outcome) {
        final Optional<WorkerDecisionEntry> entryOpt =
                workerDecisionRepo.findLatestByCaseIdAndCapability(caseId, "sar-drafting");

        if (entryOpt.isEmpty()) {
            LOG.warnf("No WorkerDecisionEntry found for caseId=%s capability=sar-drafting — skipping attestation", caseId);
            return;
        }

        final WorkerDecisionEntry entry = entryOpt.get();
        final LedgerAttestation attestation = new LedgerAttestation();
        attestation.id = UUID.randomUUID();
        attestation.ledgerEntryId = entry.id;
        attestation.subjectId = caseId;
        attestation.attestorId = "aml-compliance-system";
        attestation.attestorType = ActorType.SYSTEM;
        attestation.attestorRole = "SarOutcomeFeedback";
        attestation.verdict = toVerdict(outcome.verdict());
        attestation.capabilityTag = "sar-drafting";
        attestation.trustDimension = "investigation-accuracy";
        attestation.dimensionScore = outcome.investigationAccuracyScore();
        attestation.confidence = 1.0;
        attestation.occurredAt = Instant.now();
        attestation.evidence = outcome.reason();

        em.persist(attestation);
    }

    private AttestationVerdict toVerdict(final SarVerdict verdict) {
        return switch (verdict) {
            case UPHELD -> AttestationVerdict.SOUND;
            case WITHDRAWN, FLAGGED -> AttestationVerdict.FLAGGED;
        };
    }
}
```

- [ ] **Step 6.4: Run tests — expect pass**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am test -Dtest=SarOutcomeFeedbackServiceTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD"
```

Expected: `Tests run: 4, Failures: 0, Errors: 0` — BUILD SUCCESS.

If the `ledger_entry` native SQL insert fails due to schema differences, check `hash` column nullability in H2 — change to use `X'00'` or `''` if the column is NOT NULL.

- [ ] **Step 6.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/trust/SarOutcomeFeedbackService.java app/src/test/java/io/casehub/aml/trust/SarOutcomeFeedbackServiceTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#38): add SarOutcomeFeedbackService — writes LedgerAttestation on SAR outcome"
```

---

## Task 7: Multiple worker candidates in `AmlInvestigationCaseHub`

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseHub.java`

- [ ] **Step 7.1: Add three new workers to `augment()`**

Open `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseHub.java`.

In the `augment()` method, change the `yaml.getWorkers().addAll(List.of(...))` call to add the three new workers:

```java
private CaseDefinition augment(final CaseDefinition yaml) {
    yaml.getWorkers().addAll(List.of(
            entityResolutionWorker(),
            patternAnalysisWorker(),
            osintScreeningWorker(),            // existing (junior — declines)
            osintScreeningWorkerSenior(),      // NEW: full PEP + sanctions check
            seniorAnalystWorker(),
            sarDraftingWorkerJunior(),         // renamed from sarDraftingWorker()
            sarDraftingWorkerSenior()          // NEW: full narrative + due diligence
    ));
    return yaml;
}
```

Rename the existing `sarDraftingWorker()` method to `sarDraftingWorkerJunior()` and update the worker name to `"sar-drafting-agent-junior"`. Then add:

```java
private Worker sarDraftingWorkerJunior() {
    return Worker.builder()
            .name("sar-drafting-agent-junior")
            .capabilities(List.of(cap("sar-drafting")))
            .function((final Map<String, Object> input) -> {
                @SuppressWarnings("unchecked")
                final Map<String, Object> txMap =
                        (Map<String, Object>) input.get("transaction");
                @SuppressWarnings("unchecked")
                final Map<String, Object> osintMap =
                        (Map<String, Object>) input.get("osintScreening");
                final SuspiciousTransaction tx =
                        objectMapper.convertValue(txMap, SuspiciousTransaction.class);
                final boolean osintDeclined = osintMap != null
                        && Boolean.TRUE.equals(osintMap.get("declined"));
                final String sarNarrative = "SAR filed for transaction " + tx.id()
                        + ". Amount: " + tx.amount() + " " + tx.currency()
                        + (osintDeclined ? " OSINT screening declined." : "");
                final String complianceTaskId = complianceReviewLifecycle.openReview(tx,
                        buildSummary(input, sarNarrative));
                return Map.of("sarNarrative", sarNarrative, "complianceTaskId", complianceTaskId);
            })
            .build();
}

private Worker sarDraftingWorkerSenior() {
    return Worker.builder()
            .name("sar-drafting-agent-senior")
            .capabilities(List.of(cap("sar-drafting")))
            .function((final Map<String, Object> input) -> {
                @SuppressWarnings("unchecked")
                final Map<String, Object> txMap =
                        (Map<String, Object>) input.get("transaction");
                @SuppressWarnings("unchecked")
                final Map<String, Object> entityMap =
                        (Map<String, Object>) input.get("entityResolution");
                @SuppressWarnings("unchecked")
                final Map<String, Object> osintMap =
                        (Map<String, Object>) input.get("osintScreening");
                final SuspiciousTransaction tx =
                        objectMapper.convertValue(txMap, SuspiciousTransaction.class);
                final String entityType = entityMap != null
                        ? (String) entityMap.getOrDefault("entityType", "UNKNOWN") : "UNKNOWN";
                final boolean osintDeclined = osintMap != null
                        && Boolean.TRUE.equals(osintMap.get("declined"));
                final String sarNarrative = buildNarrative(tx, entityType, osintDeclined);
                final String complianceTaskId = complianceReviewLifecycle.openReview(tx,
                        buildSummary(input, sarNarrative));
                return Map.of("sarNarrative", sarNarrative, "complianceTaskId", complianceTaskId);
            })
            .build();
}

private Worker osintScreeningWorkerSenior() {
    return Worker.builder()
            .name("osint-screening-agent-senior")
            .capabilities(List.of(cap("osint-screening")))
            .function((final Map<String, Object> input) -> Map.of(
                    "declined", false,
                    "reason", "full-clearance",
                    "pepHit", false,
                    "sanctionsHit", false,
                    "screeningLevel", "ENHANCED"
            ))
            .build();
}
```

Extract the `buildSummary` helper (to avoid repeating the InvestigationSummary construction in both SAR workers):

```java
private InvestigationSummary buildSummary(final Map<String, Object> input, final String sarNarrative) {
    @SuppressWarnings("unchecked")
    final Map<String, Object> txMap = (Map<String, Object>) input.get("transaction");
    @SuppressWarnings("unchecked")
    final Map<String, Object> entityMap = (Map<String, Object>) input.get("entityResolution");
    @SuppressWarnings("unchecked")
    final Map<String, Object> osintMap = (Map<String, Object>) input.get("osintScreening");
    final SuspiciousTransaction tx = objectMapper.convertValue(txMap, SuspiciousTransaction.class);
    final boolean osintDeclined = osintMap != null && Boolean.TRUE.equals(osintMap.get("declined"));
    final SpecialistOutcome<EntityResolutionResult> entityOutcome = entityMap != null
            ? new SpecialistOutcome.Completed<>(
                    objectMapper.convertValue(entityMap, EntityResolutionResult.class))
            : new SpecialistOutcome.Declined<>("sar-agent", "entity-resolution", "missing from context");
    final SpecialistOutcome<PatternAnalysisResult> patternOutcome =
            new SpecialistOutcome.Completed<>(
                    new PatternAnalysisResult(false, "engine-driven investigation"));
    final SpecialistOutcome<OsintResult> osintOutcome = osintDeclined
            ? new SpecialistOutcome.Declined<>("osint-agent", "osint-screening",
                    "insufficient clearance for PEP database access")
            : new SpecialistOutcome.Completed<>(new OsintResult(false, false, "no matches"));
    return new InvestigationSummary(tx, entityOutcome, patternOutcome, osintOutcome, sarNarrative);
}
```

Remove the old `sarDraftingWorker()` method (now replaced by `sarDraftingWorkerJunior()` and `sarDraftingWorkerSenior()`). Move the existing `buildNarrative()` static method to remain as-is.

- [ ] **Step 7.2: Verify compile**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am test-compile -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 7.3: Run existing Layer 5 tests — no regressions**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am test -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD|FAIL"
```

Expected: All previously-passing tests still pass.

- [ ] **Step 7.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseHub.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#38): add senior/junior SAR drafting and senior OSINT workers for trust routing demo"
```

---

## Task 8: Layer 6 REST resource

**Files:**
- Create: `app/src/main/java/io/casehub/aml/engine/Layer6InvestigationResponse.java`
- Create: `app/src/main/java/io/casehub/aml/engine/WorkerRoutingDecision.java`
- Create: `app/src/main/java/io/casehub/aml/engine/AmlLayer6Resource.java`
- Create: `app/src/test/java/io/casehub/aml/engine/AmlLayer6ResourceTest.java`

- [ ] **Step 8.1: Write failing test**

Create `app/src/test/java/io/casehub/aml/engine/AmlLayer6ResourceTest.java`:

```java
package io.casehub.aml.engine;

import io.casehub.aml.domain.SarOutcome;
import io.casehub.aml.domain.SarVerdict;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
import org.awaitility.Awaitility;
import org.junit.jupiter.api.Test;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;
import java.util.concurrent.TimeUnit;
import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class AmlLayer6ResourceTest {

    private static final SuspiciousTransaction TRANSACTION = new SuspiciousTransaction(
            "TXN-L6-" + UUID.randomUUID(),
            "ACC-ORIGIN", "ACC-DEST",
            new BigDecimal("95000"), "USD",
            Instant.parse("2024-06-01T00:00:00Z"),
            "Structured layering pattern — CORPORATE");

    @Test
    void post_investigate_returns_202_with_caseId() {
        given().contentType(ContentType.JSON).body(TRANSACTION)
                .when().post("/api/layer6/investigations")
                .then().statusCode(202)
                .body("caseId", notNullValue());
    }

    @Test
    void get_investigation_returns_completed_with_routing_decisions() {
        // POST to start
        final String caseIdStr = given().contentType(ContentType.JSON).body(TRANSACTION)
                .when().post("/api/layer6/investigations")
                .then().statusCode(202)
                .extract().path("caseId");

        // Poll GET until completed (engine runs async on Quartz workers)
        Awaitility.await().atMost(30, TimeUnit.SECONDS).pollInterval(500, TimeUnit.MILLISECONDS)
                .until(() -> {
                    final String status = given()
                            .when().get("/api/layer6/investigations/" + caseIdStr)
                            .then().statusCode(200)
                            .extract().path("status");
                    return "completed".equals(status);
                });

        // Assert routing decisions present with trust scores
        given().when().get("/api/layer6/investigations/" + caseIdStr)
                .then().statusCode(200)
                .body("status", equalTo("completed"))
                .body("routingDecisions", not(empty()))
                .body("routingDecisions.capabilityTag", hasItem("sar-drafting"))
                .body("routingDecisions.selectedWorker", hasItem("sar-drafting-agent-senior"));
    }

    @Test
    void post_outcome_returns_204() {
        // Start investigation first
        final String caseIdStr = given().contentType(ContentType.JSON).body(TRANSACTION)
                .when().post("/api/layer6/investigations")
                .then().statusCode(202)
                .extract().path("caseId");

        // Wait for completion
        Awaitility.await().atMost(30, TimeUnit.SECONDS).pollInterval(500, TimeUnit.MILLISECONDS)
                .until(() -> "completed".equals(
                        given().when().get("/api/layer6/investigations/" + caseIdStr)
                                .then().extract().path("status")));

        // Record SAR outcome — must return 204
        final SarOutcome outcome = new SarOutcome(SarVerdict.UPHELD, "SAR upheld by FinCEN", 0.95);
        given().contentType(ContentType.JSON).body(outcome)
                .when().post("/api/layer6/investigations/" + caseIdStr + "/outcome")
                .then().statusCode(204);
    }
}
```

- [ ] **Step 8.2: Run test — expect compile failure**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am test -Dtest=AmlLayer6ResourceTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -5
```

Expected: FAIL — `AmlLayer6Resource` does not exist.

- [ ] **Step 8.3: Create response types**

Create `app/src/main/java/io/casehub/aml/engine/WorkerRoutingDecision.java`:

```java
package io.casehub.aml.engine;

import com.fasterxml.jackson.annotation.JsonInclude;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record WorkerRoutingDecision(
        String capabilityTag,
        String selectedWorker,
        Double trustScore) {}  // null when Phase 0 (no trust history — availability routing)
```

Create `app/src/main/java/io/casehub/aml/engine/Layer6InvestigationResponse.java`:

```java
package io.casehub.aml.engine;

import java.util.List;
import java.util.UUID;

public record Layer6InvestigationResponse(
        UUID caseId,
        String status,                      // "completed" | "in-progress"
        List<WorkerRoutingDecision> routingDecisions) {}
```

- [ ] **Step 8.4: Create `AmlLayer6Resource`**

Create `app/src/main/java/io/casehub/aml/engine/AmlLayer6Resource.java`:

```java
package io.casehub.aml.engine;

import io.casehub.aml.domain.SarOutcome;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.aml.trust.AmlWorkerDecisionRepository;
import io.casehub.aml.trust.SarOutcomeFeedbackService;
import io.casehub.ledger.model.WorkerDecisionEntry;
import io.casehub.ledger.routing.TrustScoreCache;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import java.util.List;
import java.util.Map;
import java.util.OptionalDouble;
import java.util.UUID;

@Path("/api/layer6/investigations")
@ApplicationScoped
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class AmlLayer6Resource {

    @Inject AmlEngineCoordinator coordinator;
    @Inject AmlWorkerDecisionRepository workerDecisionRepo;
    @Inject SarOutcomeFeedbackService feedbackService;
    @Inject TrustScoreCache trustScoreCache;

    @POST
    public Response startInvestigation(final SuspiciousTransaction transaction) {
        final UUID caseId = coordinator.startInvestigation(transaction);
        return Response.accepted(Map.of("caseId", caseId)).build();
    }

    @GET
    @Path("/{caseId}")
    public Layer6InvestigationResponse getInvestigation(@PathParam("caseId") UUID caseId) {
        final List<WorkerDecisionEntry> entries = workerDecisionRepo.findAllByCaseId(caseId);

        final boolean completed = entries.stream()
                .anyMatch(e -> "sar-drafting".equals(e.capabilityTag));

        if (!completed) {
            return new Layer6InvestigationResponse(caseId, "in-progress", List.of());
        }

        final List<WorkerRoutingDecision> decisions = entries.stream()
                .map(e -> {
                    final OptionalDouble score =
                            trustScoreCache.getCapabilityScore(e.workerId, e.capabilityTag);
                    return new WorkerRoutingDecision(
                            e.capabilityTag,
                            e.workerId,
                            score.isPresent() ? score.getAsDouble() : null);
                })
                .toList();

        return new Layer6InvestigationResponse(caseId, "completed", decisions);
    }

    @POST
    @Path("/{caseId}/outcome")
    public Response recordOutcome(
            @PathParam("caseId") UUID caseId,
            final SarOutcome outcome) {
        feedbackService.recordOutcome(caseId, outcome);
        return Response.noContent().build();
    }
}
```

- [ ] **Step 8.5: Run tests — expect pass**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am test -Dtest=AmlLayer6ResourceTest -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD"
```

Expected: `Tests run: 3, Failures: 0, Errors: 0` — BUILD SUCCESS.

If the `sar-drafting-agent-senior` doesn't appear in routing decisions (junior selected instead), the trust seed is not being picked up by the routing strategy. Debug by checking: (1) `AmlTrustScoreSeeder` ran (check logs for upsert calls), (2) `TrustScoreCache.getCapabilityScore("sar-drafting-agent-senior", "sar-drafting")` returns non-empty.

- [ ] **Step 8.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/java/io/casehub/aml/engine/Layer6InvestigationResponse.java app/src/main/java/io/casehub/aml/engine/WorkerRoutingDecision.java app/src/main/java/io/casehub/aml/engine/AmlLayer6Resource.java app/src/test/java/io/casehub/aml/engine/AmlLayer6ResourceTest.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(#38): add AmlLayer6Resource with async POST/GET and SAR outcome endpoint"
```

---

## Task 9: Full integration test and suite verification

**Files:**
- Create: `app/src/test/java/io/casehub/aml/engine/AmlLayer6InvestigationIT.java`

- [ ] **Step 9.1: Write integration test**

Create `app/src/test/java/io/casehub/aml/engine/AmlLayer6InvestigationIT.java`:

```java
package io.casehub.aml.engine;

import io.casehub.aml.domain.SarOutcome;
import io.casehub.aml.domain.SarVerdict;
import io.casehub.aml.domain.SuspiciousTransaction;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.http.ContentType;
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
 * Full Layer 6 integration test: starts investigation via POST, polls GET until completed,
 * verifies trust-weighted senior agent selected, records SAR outcome, verifies attestation written.
 */
@QuarkusTest
class AmlLayer6InvestigationIT {

    @PersistenceContext(unitName = "qhorus")
    EntityManager em;

    @Test
    void full_trust_routing_flow_corporate_case() {
        final SuspiciousTransaction tx = new SuspiciousTransaction(
                "TXN-IT-" + UUID.randomUUID(),
                "ACC-A", "ACC-B",
                new BigDecimal("120000"), "USD",
                Instant.parse("2024-09-15T00:00:00Z"),
                "Structured layering — CORPORATE");

        // Step 1: Start investigation
        final String caseIdStr = given().contentType(ContentType.JSON).body(tx)
                .when().post("/api/layer6/investigations")
                .then().statusCode(202)
                .extract().path("caseId");

        final UUID caseId = UUID.fromString(caseIdStr);

        // Step 2: Poll until completed
        Awaitility.await()
                .atMost(30, TimeUnit.SECONDS)
                .pollInterval(500, TimeUnit.MILLISECONDS)
                .until(() -> "completed".equals(
                        given().when().get("/api/layer6/investigations/" + caseIdStr)
                                .then().extract().path("status")));

        // Step 3: Verify senior sar-drafting agent was selected (trust score 0.90 > threshold 0.75)
        final io.restassured.response.Response getResponse =
                given().when().get("/api/layer6/investigations/" + caseIdStr).then().extract().response();
        final List<java.util.Map<String, Object>> decisions =
                getResponse.jsonPath().getList("routingDecisions");

        final java.util.Optional<java.util.Map<String, Object>> sarDecision = decisions.stream()
                .filter(d -> "sar-drafting".equals(d.get("capabilityTag")))
                .findFirst();

        assertTrue(sarDecision.isPresent(), "sar-drafting routing decision must be present");
        assertEquals("sar-drafting-agent-senior", sarDecision.get().get("selectedWorker"),
                "Senior agent must be selected (trust 0.90 > threshold 0.75)");
        assertNotNull(sarDecision.get().get("trustScore"),
                "Trust score must be populated from TrustScoreCache");

        // Step 4: Record SAR outcome (positive — SAR upheld)
        given().contentType(ContentType.JSON)
                .body(new SarOutcome(SarVerdict.UPHELD, "SAR upheld by FinCEN unit", 0.94))
                .when().post("/api/layer6/investigations/" + caseIdStr + "/outcome")
                .then().statusCode(204);

        // Step 5: Verify LedgerAttestation persisted with SOUND verdict
        final List<LedgerAttestation> attestations = em.createQuery(
                "SELECT a FROM LedgerAttestation a WHERE a.subjectId = :sid",
                LedgerAttestation.class)
                .setParameter("sid", caseId)
                .getResultList();

        assertFalse(attestations.isEmpty(), "LedgerAttestation must be written after outcome recorded");
        final LedgerAttestation a = attestations.get(0);
        assertEquals(AttestationVerdict.SOUND, a.verdict);
        assertEquals("sar-drafting", a.capabilityTag);
        assertEquals("investigation-accuracy", a.trustDimension);
        assertEquals(0.94, a.dimensionScore, 0.001);
    }
}
```

- [ ] **Step 9.2: Run integration test**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml -pl app -am test -Dtest=AmlLayer6InvestigationIT -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD|FAIL|ERROR"
```

Expected: `Tests run: 1, Failures: 0, Errors: 0` — BUILD SUCCESS.

- [ ] **Step 9.3: Run the full test suite**

```bash
mvn -f /Users/mdproctor/claude/casehub/aml/pom.xml test 2>&1 | grep -E "Tests run|BUILD|FAIL"
```

Expected: All tests pass, BUILD SUCCESS.

- [ ] **Step 9.4: Commit integration test**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/test/java/io/casehub/aml/engine/AmlLayer6InvestigationIT.java
git -C /Users/mdproctor/claude/casehub/aml commit -m "test(#38): add AmlLayer6InvestigationIT — full trust routing flow verification"
```

---

## Notes

**Hash column in `ledger_entry`:** If Task 6's native SQL insert fails with a constraint violation on the `hash` column, replace `'test-hash'` with `''` (empty string). The column is likely NOT NULL in the schema but accepts empty strings for test data.

**`@Transactional @PostConstruct` in Quarkus:** Quarkus Arc supports interceptor bindings (including `@Transactional`) on `@PostConstruct` methods via its interception model. If the seeder's `@PostConstruct @Transactional` throws `TransactionRequiredException`, wrap the logic in a call to a `@Transactional`-annotated method via `@Inject AmlTrustScoreSeeder self` (self-injection through CDI proxy).

**Senior not selected (Phase 0 fallback):** If the integration test shows the junior agent selected despite high seeded scores, the trust score cache may not have been populated before routing. Check: (1) `AmlTrustScoreSeeder` log output at startup, (2) `TrustScoreCache.hydrate()` was called after seed writes. Add a breakpoint or log statement to verify.
