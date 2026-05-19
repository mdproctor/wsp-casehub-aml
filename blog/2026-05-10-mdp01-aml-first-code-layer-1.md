---
layout: post
title: "casehub-aml: the anti-pattern comes first"
date: 2026-05-10
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [aml, quarkus, tutorial, casehub]
excerpt: "The first real code in casehub-aml is intentionally wrong — a naive sequential baseline that makes every missing capability visible before CaseHub layers are introduced to close the gaps."
---

The first real code in casehub-aml is, by design, wrong.

Layer 1 of the tutorial is a naive baseline — four specialist services called sequentially, no error handling, no audit trail, no formal accountability. The point is to show what's missing before introducing anything that fixes it. Every CaseHub layer added on top has to earn its place by closing a visible gap.

The mechanism is four comments, one before each service call:

```java
// LAYER 1 GAP: no attribution — who resolved this entity graph?
// No record of which agent made this decision or when.
EntityResolutionResult entity = entityResolutionService.resolve(transaction);

// LAYER 1 GAP: no failure resilience — if this call times out or throws,
// the entire investigation is lost with no trace of partial work.
PatternAnalysisResult pattern = patternAnalysisService.analyze(transaction);

// LAYER 1 GAP: no deadline tracking — OSINT runs sequentially after pattern
// analysis. No FinCEN 30-day SLA. No parallel execution. No formal obligation.
OsintResult osint = osintScreeningService.screen(transaction);

// LAYER 1 GAP: no audit trail — this narrative cannot be proven to FinCEN.
// No tamper-evident record of the reasoning chain exists.
String sarNarrative = sarDraftingService.draft(transaction, entity, pattern, osint);
```

Each comment names a specific FinCEN compliance failure, not a generic TODO. The idea is that a reader who has built AML infrastructure in Java has probably lived at least one of these — the investigation lost when a service timed out, the SAR filing where nobody could say which agent made the call. The comments are anchors, not annotations.

The stubs return empty results. I considered making them return plausible data — a named entity graph, a structuring finding — so the endpoint looked convincing when curled. The risk is that realistic values drift the moment the domain model changes, and a stub that looks real is harder to recognise as fake. Empty is unambiguous.

Two surprises surfaced integrating the foundation extensions. casehub-qhorus pulls in `quarkus-hibernate-reactive-panache` unconditionally, regardless of whether reactive is enabled. In a JDBC H2 test environment, Quarkus tries to create a reactive Vert.x pool for every datasource and fails at startup — nothing in the error message points to qhorus. The fix requires three properties in combination:

```properties
casehub.qhorus.reactive.enabled=false
quarkus.datasource.reactive=false
quarkus.datasource.qhorus.reactive=false
```

The qhorus flag alone isn't enough. Extension activation happens at augmentation time, before the flag takes effect. This is tracked in casehubio/qhorus#141.

The second: casehub-work and casehub-qhorus both ship a Flyway migration named V2. When both are on the classpath, Flyway finds two scripts at the same version and refuses to start. In tests, the workaround is to disable Flyway and use `drop-and-create` — Layer 1 needs no schema anyway. The workarounds sit in casehubio/aml#13 until casehubio/qhorus#142 and casehubio/work#162 resolve the naming conflict at source.

The multi-datasource configuration that makes all three coexist — default persistence unit for work and AML entities, named qhorus persistence unit for qhorus and ledger — came from reading claudony's `application.properties` directly. It wasn't documented anywhere else.

A code reviewer Claude dispatched after the implementation flagged a test that wasn't asserting what its name claimed. `investigateCalledTwice_producesIndependentSummaries` was asserting both results were non-null, not that they were distinct objects. One line fixed it.

The scaffold is up. Layer 1 is intentionally broken. That's the plan.
