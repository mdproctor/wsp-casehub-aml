---
layout: post
title: "Channel IDs make terrible audit keys"
date: 2026-05-23
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [aml, casehub, ledger, qhorus, quarkus, audit, java]
---

Layer 4 was supposed to be the simple one. Add `casehub-ledger` as an explicit dependency, write a couple of domain entries per investigation, wire them to the HTTP response. The main design question was: how do you link the AML domain entries to the qhorus message entries (COMMAND/DONE/DECLINE per specialist)?

The answer revealed something wrong in qhorus.

Every `LedgerEntry` has a `subjectId` — the domain aggregate being audited. It's how a regulator would query: give me everything for investigation TXN-001. The qhorus `MessageLedgerEntry` (written automatically per agent message) uses `channelId` as `subjectId`. The "entity-resolution" capability lane — not the investigation, not the transaction. The infrastructure label.

A FinCEN audit query for a specific transaction returns nothing useful. The entries are there; they're indexed by the wrong thing.

Fixing it meant redesigning the `MessageService` API before Layer 4 could be implemented correctly. The old API had nine positional parameters and three trailing nulls. We replaced it with a `MessageDispatch` builder carrying `subjectId` and `causedByEntryId` as first-class fields, with validation enforcing protocol invariants at `build()` time:

```java
messageService.dispatch(MessageDispatch.builder()
        .channelId(channel.id)
        .sender(ORCHESTRATOR)
        .type(MessageType.COMMAND)
        .content(transaction.id())
        .correlationId(correlationId)
        .subjectId(caseId)           // domain aggregate — not the channel
        .actorType(ActorType.SYSTEM)
        .build());
```

The design review took three rounds across two days. Each round found something the previous missed. The final validation table covers nine message types across three required fields — `inReplyTo`, `correlationId`, and `target`, each independently required for specific types, with the HANDOFF commitment lifecycle pinned down in round two. The migration touched 104 test call sites that had been sending DONE and DECLINE without `inReplyTo` — protocol violations the old positional API silently accepted.

With qhorus shipped, Layer 4 was straightforward. The coordinator generates a `caseId` UUID at investigation start — distinct from the external transaction ID string, which can't be a UUID without breaking every caller and doesn't model the domain correctly anyway. An investigation is a separate aggregate from the transaction that triggered it; in Layer 5+ a case may span multiple. The UUID flows to `AmlLedgerService` (CASE_OPENED entry, then COMPLIANCE_REVIEW_OPENED) and to `QhorusAmlInvestigator`, which attaches it as `subjectId` on every specialist dispatch.

A review pass caught two things before the commit: `entryType = LedgerEntryType.COMMAND` on the CASE_OPENED entry (wrong — it's a fact, not an intent; should be `EVENT`) and the missing `@JsonInclude(NON_NULL)` on `DispatchResult` (the MCP JSON would have included explicit nulls for ledger fields when ledger writes are suppressed).

Implementation hit two sharp corners.

The first: `@TestTransaction`. I added it to the ledger chain test for isolation. But `@TestTransaction` wraps the test in a rolled-back outer transaction, and `@Transactional(REQUIRED)` joins it. The writes never commit. The subsequent `findEntryById()` query returns nothing. The service returned a non-null entry ID — it appeared to work — but the entry didn't exist in the database. Removing `@TestTransaction` fixed it. Use unique identifiers per test instead.

The second: a stale local snapshot. The qhorus entity was renamed from `agent_message_ledger_entry` to `message_ledger_entry`. qhorus tests use `drop-and-create`, so Hibernate regenerated the schema automatically and everything passed. AML tests use `generation=none` — Flyway only. The installed snapshot jar still contained the old migration SQL. `mvn compile` succeeded against cached class files; `@QuarkusTest` failed at startup with TABLE NOT FOUND. Running `mvn install` against the qhorus source pom fixed it.

Both are in the garden.
