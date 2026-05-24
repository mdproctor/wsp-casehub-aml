---
layout: post
title: "What Flyway Was Hiding"
date: 2026-05-21
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [flyway, quarkus, casehub, testing, migration]
---

Flyway has been disabled in `@QuarkusTest` since Layer 3 shipped. The workaround — `drop-and-create` and `migrate-at-start=false` — was supposed to be temporary. qhorus#174 moved the qhorus migrations out of `db/migration/qhorus/` (a subdirectory Flyway's recursive scan would pick up) into `db/qhorus/migration/`. That should have been enough.

It wasn't.

We turned Flyway back on and hit a V1 conflict immediately. The qhorus datasource manages both qhorus and ledger entities, so its Flyway needed to scan `classpath:db/migration` to pick up ledger's V1000-V1007 migrations. That same scan also found casehub-work's `V1__initial_schema.sql` at the root. Conflict.

This was ledger's version of the same problem. Filed casehubio/ledger#95: move ledger migrations from `db/migration/` to `db/ledger/migration/`. Done quickly. Turned Flyway back on.

```
FlywayException: Found more than one migration with version 1003
```

qhorus has `V1003__agent_message_ledger_entry.sql` — its join table for the ledger subclass. ledger has `V1003__ledger_entry_archive.sql`. Both in different directories, both scanned together by the qhorus datasource Flyway. The platform protocol reserves V1000–V1007 for the ledger base schema; V1004+ is for consumer-owned subclass joins. qhorus's join table was filed at V1003 before ledger had used that slot. Filed casehubio/qhorus#180.

qhorus renamed it to V2000. No conflicts.

The config that makes it work:

```properties
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration
```

Two paths, non-overlapping version ranges, one datasource. The qhorus extension's embedded `application.properties` only declares `db/qhorus/migration` — the consumer overrides it, which is straightforward because extension-embedded `application.properties` (ordinal 250) is overridable by the consumer's own `application.properties`. That's the opposite of `META-INF/microprofile-config.properties` (ordinal 100), which gets baked in at augmentation time and can't be overridden at test time.

One more decision: both datasources now have explicit `locations` set, not just qhorus. Flyway's classpath scan is recursive — leaving the default unconstrained means any future jar adding a `db/migration/` entry is picked up silently. Pinned explicitly. Small insurance, no ongoing cost.

Three upstream fixes. The schema is finally what production will see.
