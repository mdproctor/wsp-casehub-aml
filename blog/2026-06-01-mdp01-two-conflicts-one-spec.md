---
layout: post
title: "Two CDI conflicts, one JAX-RS spec surprise"
date: 2026-06-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [cdi, quarkus, jax-rs, build]
---

The session started with a broken build. Not a compilation failure — CDI augmentation. Quarkus couldn't start the test application because it found two `@DefaultBean` implementations for `CurrentPrincipal`: one from `casehub-engine-persistence-memory`, one from `casehub-platform`. Both Jandex-indexed, both on the test classpath, both declaring themselves the default. CDI reports ambiguity and refuses to start.

The fix was clear once identified: add `DefaultTestPrincipal` to `quarkus.arc.exclude-types` in the test `application.properties`. The hardcoded stub loses; the configurable mock from `casehub-platform` wins.

We rebuilt. Same error, different type — `GroupMembershipProvider` this time: `NoOpGroupMembershipProvider` from `casehub-work` vs `MockGroupMembershipProvider` from `casehub-platform`. Added that to the exclusion list too. The deeper issue is that the full `mvn verify` runs against cached augmented bytecode — if that cache predates a conflicting JAR landing on the classpath, the CDI conflict stays invisible. It only surfaces on a clean build or when running a targeted single-test class triggers fresh augmentation. We only found it because running `AmlInvestigationCaseHubTest` alone forced a re-augment.

With CDI resolved, 83 tests ran. Nine failed. The engine integration tests all produced the same error: "CaseDefinition not found." I spent a while with Claude reading decompiled engine JARs — `CaseHubReactor`, `CaseStartedEventHandler`, `SchedulerService` — tracing the event bus chain until Claude found the specific failure point: `SchedulerService.registerScheduledTriggers()` calls `getCaseDefinition(caseInstance.getCaseMetaModel())` and gets null. This is a pre-existing issue in the published engine SNAPSHOT, already tracked as casehubio/engine#409. Not fixable from here.

That left one failure: `post_outcome_with_invalid_score_returns_400_with_domain_message`. The test posts a `SarOutcome` with `investigationAccuracyScore=1.5` (violating the compact constructor's `[0.0, 1.0]` constraint) and expects a 400 with a JSON error body. It was getting 400 with no body and no content-type — even though `JsonMappingExceptionMapper` was registered and its unit tests passed.

The issue is JAX-RS spec §4.2.4. When a message body reader throws `IOException` — and `JsonMappingException` extends `IOException` — the runtime converts it to 400 directly, without consulting any exception mapper. I tried `@ServerExceptionMapper` next, which is Quarkus's own mechanism. Same result. Nothing fires because the exception never reaches the mapper chain.

The fix moves validation to where exception mappers can actually reach it. `AmlLayer6Resource.recordOutcome()` now accepts a plain `SarOutcomeRequest` DTO that doesn't validate in its constructor. Deserialization succeeds. The resource method constructs `SarOutcome` from the DTO, so `IllegalArgumentException` is thrown from inside the method body — where `IllegalArgumentExceptionMapper` fires correctly. One new record class, four changed lines.

Final build: 83 tests, 3 failures, all engine#409.

The session also opened the arc42stories migration question — whether to move from `LAYER-LOG.md` to `ARC42STORIES.MD` now that all 7 layers are complete. The spec and CaseHub profile are written; the migration is just a matter of when. I'm leaning toward doing it before starting Layer 8, but deferred it this session.
