---
layout: post
title: "Layer 3 shipped: five infrastructure surprises"
date: 2026-05-18
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [aml, quarkus, cdi, qhorus, layer3]
---

The architectural investigation was yesterday. Today was implementation —
and the gap between a clean design and a working build was, predictably, messier.

Layer 3 adds casehub-qhorus: typed COMMAND/RESPONSE/DONE/DECLINE per specialist
agent. The design was settled: `SpecialistOutcome<T>` sealed interface, composer
pattern, `QhorusAmlInvestigator` dispatching to stub agent behaviours. The code
took five separate infrastructure surprises to get green.

**Surprise one:** `casehub.qhorus.reactive.enabled=false` no longer maps to any
config root. The SmallRye config validator fails at test startup with no useful
hint. The property we'd been carrying since Layer 1 — the workaround for
casehubio/qhorus#141 — has simply been removed from the qhorus config schema.
The upstream bug was fixed without announcement. Delete the property, tests
start cleanly.

**Surprise two:** `LedgerVerificationService` (casehub-ledger) now injects
`ReactiveLedgerEntryRepository` which is vetoed in JDBC-only test mode. This
bean didn't exist when AML's `@QuarkusTest` was first written. Excluding it
via `quarkus.arc.exclude-types` surfaces a second failure: something else
injects `LedgerVerificationService`. We traced the chain: `LedgerComplianceReportService`
and `LedgerRetentionJob` — two more beans to exclude. Three beans, one line:

```properties
quarkus.arc.exclude-types=io.casehub.ledger.runtime.service.LedgerVerificationService,\
  io.casehub.ledger.runtime.service.LedgerComplianceReportService,\
  io.casehub.ledger.runtime.service.LedgerRetentionJob
```

**Surprise three:** `PushAgentDispatch` implements `AgentChannelBackend`, which
qhorus already provides via `QhorusChannelBackend`. CDI sees two candidates and
fails with `AmbiguousResolutionException`. Fix: `@Typed({AgentDispatchMechanism.class, PushAgentDispatch.class})` — restricts CDI visibility so
`QhorusChannelBackend` keeps the injection point to itself. This one's now in
the garden (GE-20260517-9e571a).

**Surprise four:** `channelGateway.fanOut()` was supposed to trigger `PushAgentDispatch.post()` on the registered backend. It didn't. We tried everything
the qhorus deep-dive describes: `initChannel()` before `registerBackend()`,
`@Startup` on the registry, `open()` before registration per protocol. The
backends were registered; the fan-out completed without calling `post()` on any
of them. Issue #22 tracks the root cause investigation.

The fix was pragmatic: `QhorusAmlInvestigator` calls `AgentBehaviour.handle()`
directly after sending the COMMAND message. The COMMAND and DONE/DECLINE messages
are still persisted to qhorus — the formal commitment records exist in the DB.
The ChannelBackend fan-out path is documented as the production extension point
for Layer 5 (claudony workers, real AI agents). For the tutorial, direct dispatch
is correct.

**Surprise five:** The first few test runs returned HTTP 409 Conflict after a
5-second delay. 409 is HTTP Conflict — unexpected for a timeout. Claude traced it:
`casehub-work` ships `IllegalStateExceptionMapper`, a JAX-RS exception mapper that
silently routes all `IllegalStateException` to 409. The poll-timeout was throwing
`IllegalStateException`; the mapper was catching it. Change it to `RuntimeException`
— 500, as expected. This is now protocol PP-20260517-4b61ae.

The Jackson mixin for `SpecialistOutcome<T>` was the one non-infrastructure piece
that needed thought. You can't put `@JsonTypeInfo` on a sealed interface in `api/`
without adding Jackson as a dependency. The fix: `ObjectMapperCustomizer` in `app/`
registers a mixin that carries the annotation without touching the domain module.
Nineteen lines of code; the REST response now includes `"type": "Declined"` for the
OSINT outcome.

All 19 tests pass. The REST response shows OSINT as `{"type":"Declined","agentId":"aml-osint-stub","capability":"osint-screening","reason":"insufficient clearance for PEP database access"}`. Layer 4 (casehub-ledger as explicit audit trail) is next.
