---
layout: post
title: "Teaching the case base to remember"
date: 2026-07-19
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [cbr, retain, ledger, case-profile]
series: issue-94-case-profile-store
---

The [previous entry](2026-07-17-mdp01-six-columns-that-matter.md) built the similarity model — six dimensions that describe an AML investigation for CBR matching. This entry closes the loop: when an investigation reaches a SAR verdict, the case profile goes into the case base for future retrieval.

## The design question that wasn't

Issue #94 asked whether the case profile store should extend `CaseMemoryStore` or use a dedicated SPI. I expected this to be a genuine design fork. It wasn't. The platform already has `CbrCaseMemoryStore` — a purpose-built SPI with `store()`, `retrieveSimilar()`, `recordOutcome()`, and `registerSchema()`. The existing `AmlMemoryService` uses `CaseMemoryStore` for entity-level facts (risk classifications per account, network relationships, pattern findings). The case profile is a case-level concern — the whole investigation as a retrievable unit, not per-entity fragments. Different granularity, different store.

The real design question turned out to be *when* to store. Two candidates: case completion (all specialist workers done) or SAR outcome recording (compliance officer has reviewed and rendered a verdict). I went with SAR outcome. The verdict — UPHELD, WITHDRAWN, FLAGGED — is the quality signal. A case that completed but hasn't been reviewed is a case without a learning signal. The CBR Retrieve step (#95) will rank similar cases partly by outcome confidence; storing before the verdict exists would inject noise.

## What went in

`AmlCaseProfileStoreObserver` fires on `SarOutcomeRecordedEvent`. It pulls the `CaseProfile` dimensions from the engine's case context (transaction amount, flag reason, entity type, jurisdiction risk, network complexity, prior incident count), builds the investigation path from `PlanItemStore` (which workers ran and in what order), grabs the SAR narrative if the drafting worker produced one, and stores the whole thing as a `FeatureVectorCbrCase` via `CbrCaseMemoryStore`.

Alongside the CBR store write, a `AmlCaseProfileLedgerEntry` goes into the tamper-evident ledger. The ledger entry captures all six profile dimensions plus the outcome, confidence score, and investigation path. Its `causedByEntryId` points to the `AmlSarOfficerReviewedLedgerEntry` — closing the evidence chain from officer review to case base indexing.

Both writes are in independent try/catch blocks. A CBR store failure doesn't prevent the ledger write and vice versa. This follows the existing AML convention from `AmlSarOutcomeMemoryObserver`.

## What the review caught

The adversarial design review (15 issues across 5 rounds) found several things I'd have shipped as bugs:

- The spec said `@ObservesAsync`, but `SarOutcomeRecordedEvent` is dispatched with `.fire()` — CDI spec says `@ObservesAsync` requires `.fireAsync()`. The observer would have silently never fired.
- The investigation path extraction used `CaseInstance.getCompletedPlanItems()` — a method that doesn't exist. The correct path goes through `PlanItemStore.findByCaseId()`, which returns `PlanItemRecord` with `bindingName()`, `executorName()`, `status()`, and `createdAt()`.
- `FeatureField.text("sar_narrative")` creates a non-semantic field (`semantic=false`). For embedding-based similarity matching, it needs `FeatureField.semanticText("sar_narrative")` (`semantic=true`). The difference: `text()` does keyword matching only; `semanticText()` generates embeddings for vector similarity.
- The spec used `caseId`-derived UUIDs for both the CBR store entity ID and the ledger `subjectId`. Wrong — the ledger convention is raw `caseId` as `subjectId` (so `findBySubjectId` returns the complete investigation audit trail). The CBR store needs a namespaced UUID to avoid collisions with engine entries in the memory store. Two different ID spaces for two different concerns.

The `@ObservesAsync` one is the kind of bug that passes every test (the observer simply doesn't run, so nothing fails) and only surfaces in production when the case base never accumulates entries.
