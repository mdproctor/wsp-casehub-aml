---
layout: post
title: "The Retrieve That Wasn't"
date: 2026-07-20
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [cbr, engine, design-review]
---

The CBR epic has three steps that matter: store completed investigations (#94), retrieve similar ones when a new case starts (#95), and adapt the investigation path from what worked before (#96). Yesterday shipped Retain — storing case profiles on SAR verdict. Today was supposed to be Retrieve.

It almost wasn't.

The engine already had the entire retrieval pipeline — `CbrRetrievalService`, `CbrConfig`, `RetrievedExperience`, the builder API, the caching, the failure recovery. All of it written, tested in isolation, and ready to go. Zero callers. Zero references. Nobody had ever wired it into the case lifecycle. The infrastructure existed in a vacuum.

I had three approaches on the table. The obvious one: build an AML-specific retrieval service that calls `CbrCaseMemoryStore.retrieveSimilar()` directly, maps `CaseProfile` dimensions to `CbrQuery` features, returns domain-typed results. Clean, self-contained, would have worked.

But it's a workaround. The platform has a declarative mechanism — `CbrConfig` on the `CaseDefinition` — that's supposed to handle exactly this. JQ feature extraction, configurable weights, `CASE_LIFETIME` caching, automatic injection into `CaseContext`. The engine just... never called the service it built.

So I filed engine#761 and fixed the root. One chain in `CaseStartedEventHandler` — after the `RUNNING` transition, before `CONTEXT_CHANGED` publishes. The `injectCbrExperiences()` method checks for `CbrConfig`, calls `retrieve()`, serialises the results as a list of maps under `"cbrExperiences"` in the working layer. Guards for null at every level. Recovery returns empty on failure. The whole change was 43 lines.

With that landed, the AML side became configuration:

```java
definition.setCbrConfig(CbrConfig.builder()
        .feature("flag_reason", ".transaction.flagReason")
        .feature("transaction_amount", ".transaction.amount")
        .feature("prior_incident_count", ".priorEntityContext.entityRiskCount")
        .feature("entity_type", ".entityResolution.entityType")
        .domain("aml.cbr")
        .caseType(AmlCbrSchema.CASE_TYPE)
        .topK(10)
        .minSimilarity(0.5)
        .vectorWeight(0.0)
        .timing(CbrRetrievalTiming.CASE_LIFETIME)
        .build());
```

Four JQ expressions. Four weights. That's the entire retrieve implementation. No `AmlCbrRetrievalService`. No `SimilarCase` domain record. No coordinator changes. The engine handles everything.

The adversarial design review caught things I'd missed. The original spec declared all 7 features from the schema, including `jurisdiction_risk` and `network_complexity`. Claude traced the worker outputs and found neither field actually exists — the osint worker produces `{declined, reason, pepHit, sanctionsHit}`, not `jurisdictionRisk`. Phantom features. The spec also defaulted `vectorWeight` to the builder's implicit 0.5, which would silently blend feature similarity 50/50 with vector similarity — and the in-memory store has no embedding model, so vector similarity returns 0.0, halving all scores. Setting `vectorWeight(0.0)` explicitly documents the intent and prevents that.

One gotcha worth recording: `CbrConfig` couples retrieve and retain. Adding it to the case definition activates both `CbrRetrievalService` (via `CaseStartedEventHandler`) and `CbrCaseRetainObserver` (via `CaseOutcomeEvent`). AML already has domain-specific retain from #94 — `AmlCaseProfileStoreObserver` with `CaseProfile` feature extraction and compliance ledger entries. The platform observer would create a second entry with generic JQ-extracted features. No error, no warning, just quietly degraded retrieval quality. The fix is a CDI exclusion, but the coupling itself is a platform design gap.

At case startup, only three of the four declared features are present — `flag_reason`, `transaction_amount`, and `prior_incident_count`. The scorer normalises by the weight of present features, so the query still produces a full [0, 1] similarity range. `entity_type` is declared for forward compatibility — once `PER_EVALUATION` timing exists, it'll contribute after entity-resolution completes.

The end-to-end test stores an UPHELD structuring case, starts a new investigation with a similar transaction, and verifies `cbrExperiences` appears in the case context with the correct outcome and a positive similarity score. The case base now has both storage and retrieval — the CBR loop is half closed.
