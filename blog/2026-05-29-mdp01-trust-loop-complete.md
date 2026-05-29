---
layout: post
title: "Trust Loop Complete"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [trust-routing, casehub-ledger, casehub-engine, layer-6]
---

Layer 6 is shipped. Experienced agents now route to complex AML cases, and SAR outcomes feed back into the trust scores that drive future routing. The loop is closed.

This layer was blocked on two engine changes: `TrustRoutingPolicy` moving from the implementation module to `casehub-engine-api` (so applications can implement the SPI without pulling in implementation classes), and `WorkerDecisionEntry` to give each worker execution a ledger anchor for attestations. Both landed. The design held.

## What We Built

I brought Claude in to work through the implementation. We built four components:

`AmlTrustRoutingPolicyProvider` supplies per-capability thresholds to the engine's `TrustWeightedAgentStrategy`. OSINT screening sits at 0.70; a junior agent that declines too many cases falls below threshold and is excluded until its score recovers. SAR drafting is 0.75 with a quality floor on investigation accuracy. Senior analyst review is 0.80. The provider resolves thresholds from the Preferences API, falling back to hardcoded defaults — which is always the fallback in tutorial mode, since `MockPreferenceProvider @DefaultBean` returns null for every key.

`AmlTrustScoreSeeder` seeds initial Beta(α,β) scores for each worker at startup via `ActorTrustScoreRepository.upsert()`. The senior SAR drafter gets Beta(9,1) — 90% mean, 10 observations — which immediately puts it in Phase 3 routing. The junior gets Beta(2,8), below threshold. After seeding, the seeder calls `trustScoreCache.hydrate()` to force the cache into a consistent state regardless of whether `TrustScoreCache @Startup` ran before or after.

`SarOutcomeFeedbackService` handles the feedback direction: UPHELD writes `AttestationVerdict.SOUND` against the sar-drafting `WorkerDecisionEntry` for the case; WITHDRAWN and FLAGGED both write `FLAGGED`. The next `TrustScoreJob` cycle picks it up.

`AmlLayer6Resource` at `/api/layer6/investigations` exposes the flow: POST starts the investigation asynchronously (returns 202), GET polls for completion and returns routing decisions with current trust scores, POST `/{caseId}/outcome` records the SAR outcome and returns 204.

## The Bootstrap Detour

I initially planned to implement `TrustBootstrapSource` for seeding. The SPI name and Javadoc both imply it handles "an actor appearing for the first time" — which sounds exactly right.

It does not fire on a fresh deployment. `TrustBootstrapService.bootstrapIfNew()` is called from `TrustScoreJob.runComputation()`, inside `bootstrapIfNew(byActor.keySet())`, where `byActor` groups existing `LedgerEntry` records by `actorId`. On an empty database, that map is empty and the bootstrap source is never called. The SPI is for cross-deployment federation — importing prior history from another deployment. For known initial values, the answer is direct `upsert()`.

This is now in the garden as GE-20260529-d7b6f8.

## A CDI Problem in Engine-Ledger

Adding `casehub-engine-ledger` broke three of the five `AmlLayer5InvestigationTest` cases. Symptom: "CaseDefinition not found for case: {UUID}", thrown inside the engine's `SchedulerService` after the case instance is created.

`CaseLedgerEntryRepository @ApplicationScoped extends JpaLedgerEntryRepository @Alternative`. In CDI, a non-alternative `@ApplicationScoped` bean beats a `selected-alternatives` configuration. Something in that substitution was corrupting the `DefaultCaseDefinitionRegistry`'s `getCaseDefinition(caseInstance.getCaseMetaModel())` lookup — the definition was registered, the case instance had a reference to it, and yet the map returned null. I couldn't pin the exact mechanism; filed engine#396.

The Layer 6 tests were unaffected. The engine fix will close the regression.

The Flyway side also needed work. Engine-ledger ships V2000 and V2001 at `classpath:db/migration/` — the generic path — which collides with qhorus's V2000 (`agent_message_ledger_entry`). We created local re-numbered copies at V2002 and V2003 under `classpath:db/engine-ledger/migration/`. The SQL files have a comment explaining the re-numbering. Filed engine#395 for the underlying scoping violation.

## What the Tutorial Demonstrates

Layer 5 ended with blind selection — any available worker, regardless of history. Layer 6 makes selection visible. The GET response includes routing decisions showing which worker was selected for each capability and its current trust score. Record a SAR outcome and the attestation lands in the ledger; run `TrustScoreJob` and the score shifts.

Layer 7 — the comparison table against IBM AMLSim — is the last piece.
