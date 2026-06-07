---
layout: post
title: "Green after a month"
date: 2026-06-07
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [ci, ledger, quarkus, maven, qhorus]
---

The CI had been red on casehub-aml since May 12. Not flaky red — consistently, deterministically red. Every run, every commit, same failure. The Maven Central BOM download was returning 403 before a single test had a chance to run.

The interesting thing wasn't the 403 — it was that engine, ledger, and work were all green simultaneously. Same GitHub Actions setup, same parent POM, same Quarkus BOM version. The difference turned out to be ordering. Those repos defined their own `dependencyManagement` with a local version property, so Maven resolved the Quarkus BOM from Maven Central *before* ever contacting GitHub Packages. AML relied entirely on the parent POM for the import, which meant Maven hit GitHub Packages first (Bearer token, authenticated session), then hit Maven Central immediately after. Somewhere in that sequence — Maven's HTTP connection pool, CloudFlare's bot detection, something — the Central request came back 403.

The fix was mundane: add `quarkus.platform.version` as a local property in the root pom.xml and import the BOM directly. Maven can now resolve the version without downloading the parent first. Central gets a cold request, no auth headers in flight, 200 OK.

With that unblocked, the tests ran — and promptly told me there were three separate constraint violations on `IDX_LEDGER_ENTRY_SUBJECT_SEQ`.

The first was in `AmlTrustRoutingObserver`. The observer was writing attestation entries with `subjectId = event.caseId()`, which put them in the same ledger timeline as `AmlCaseOpenedLedgerEntry` (also subject=caseId). The observer's `nextSequenceNumber()` only queried `AmlTrustRoutingAttestation` entries — so it couldn't see the case-opened entry at seq=1, returned 1 for the attestation, and hit the constraint. The fix had two parts: give attestations their own namespaced subject (`UUID.nameUUIDFromBytes("aml-trust-routing-attestation:" + caseId)`), and serialize concurrent writes with a per-subject lock held outside the transaction so the REQUIRES_NEW commits before the next writer reads the max.

The second constraint violation was in `AmlInvestigationResourceTest` and `AmlLedgerChainTest`. Both called the Layer 3 coordinator, which dispatches qhorus COMMAND messages to specialist agents. Those messages also used `caseId` as the ledger subject, via `LedgerWriteService.record()`. The sequence assignment there was `repository.findLatestBySubjectId(caseId)` — but `repository` was `MessageLedgerEntryRepository`, which implemented `LedgerEntryRepository` with a subclass-scoped query: `FROM MessageLedgerEntry`. With JOINED inheritance, that only returns `QHORUS_MESSAGE` rows. It couldn't see `AmlCaseOpenedLedgerEntry`, returned empty, assigned seq=1, and collided.

This one was a library bug — the query scope was wrong in casehub-qhorus. We filed casehubio/qhorus#253 with a clear root cause and a test specification. The qhorus team fixed it properly: a new `LedgerEntryJpaRepository` with `FROM LedgerEntry` throughout, `MessageLedgerEntryRepository` no longer implementing `LedgerEntryRepository`, and `LedgerWriteService` getting separate injections for each. A re-architecture rather than a patch.

While all this was landing, the engine published a SNAPSHOT with a breaking API change: `Worker.Builder.function()` now returns `WorkerResult` instead of `Map<String, Object>`. Seven workers in `AmlInvestigationCaseHub` needed `WorkerResult.of(Map.of(...))` wrapping — straightforward once the compilation error told us what changed.

CI is green. The journey from 403 to green surfaced bugs in two libraries, a concurrency design issue in the observer, and an engine API change — none of which were visible while the Maven resolution error was blocking the build.
