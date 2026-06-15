---
layout: post
title: "Five days to the other side of a SNAPSHOT"
date: 2026-06-15
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [snapshot, quarkus, ci, layer-9, casehub-work, casehub-ledger, casehub-engine]
---

The Layer 9 feature — the oversight gate that intercepts consequential agent actions and routes them to human review — shipped on June 10. CI wouldn't pass. Not because the code was wrong. Because three different SNAPSHOTs had moved while the feature was in flight.

The first failure was `fromWire`. casehub-work had added `tenancyId` as a twelfth parameter to `WorkItemLifecycleEvent.fromWire()`, but the published SNAPSHOT on GitHub Packages still had eleven. The test had been written against the local install; CI resolved from the registry. These are not the same thing. casehub-work's CI was also failing at the time, so no new SNAPSHOT was publishing. We ended up waiting several days for the peer CI to stabilise before the fix even mattered.

That cleared, and the next layer appeared. The casehub-ledger SNAPSHOT had added a build-time enforcement rule: every `@Entity` subclass of `LedgerEntry` with persistent fields must override `domainContentBytes()`. The validator runs at Quarkus augmentation time and scans the full combined classpath — not just your module. So `CaseLedgerEntry` and `WorkerDecisionEntry` in casehub-engine also needed the override, even though they live in a different repo. That blocked the engine from publishing its SNAPSHOT. We filed the issue; the engine session fixed it.

Meanwhile, casehub-work had renamed its Quartz scheduler classes. `ExpiryCleanupJob` became `ExpiryTimerJob`, `ClaimDeadlineJob` became `ClaimDeadlineTimerJob`. The test `application.properties` excluded these classes by name to suppress them in `@QuarkusTest`. The old names were gone; CDI deployment failed silently until we noticed the exclusion list was pointing at non-existent classes.

The `TenantScopedPrincipal` problem was subtler. casehub-work had added a new `@RequestScoped CurrentPrincipal` implementation, creating a three-way CDI ambiguity alongside `MockCurrentPrincipal` and `QhorusInboundCurrentPrincipal`. I added the exclusion to the test `application.properties` and the isolated test run turned green. Then `mvn verify` failed with the same error. The difference: `quarkus:build` — the goal that runs during `verify` — validates CDI against the production classpath. It does not load test resources. So the exclusion needs to be in the main `application.properties` too. AML uses Qhorus for tenant context, so excluding `TenantScopedPrincipal` is architecturally correct — it just needs to be explicit in both places.

The last piece was engine#491. After the engine published its fixed SNAPSHOT, investigations would start successfully but never complete. The Quartz scheduler would fire, try to register triggers, log "CaseDefinition not found" and return void — leaving the case permanently in RUNNING state. The engine session tracked it to a registry initialisation race and fixed it. Once that landed, investigations ran to completion.

Then we found the reconciler. `AmlAttestationReconciler` reconstructs missing attestation entries on read. It built `AmlTrustRoutingAttestation` objects without setting `tenancyId` — a field the ledger SNAPSHOT now enforces as NOT NULL. The failure had been hidden because the code path is only exercised when an investigation completes and you immediately query compliance evidence. Before engine#491, investigations never completed in tests. After, they did — and the reconciler path hit the constraint.

One more: ledger#130 had exempted `SYSTEM` and `AGENT` actors from pseudonymisation. The erasure test was asserting `mappingFound=true` for `aml-orchestrator`, which is `ActorType.SYSTEM`. With the exemption in place, no token mapping is ever created for system actors. The assertion was wrong. Fixed it to expect `mappingFound=false`.

Five days of cascading SNAPSHOT regressions, each one revealing the next. The feature itself was fine throughout.
