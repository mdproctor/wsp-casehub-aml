---
layout: post
title: "Schema Consolidation and the Art.17 Contingency"
date: 2026-08-17
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [flyway, migration, gdpr, merkle, compliance]
---

casehub-aml has been accumulating Flyway migrations since Layer 4 — twelve files just for `aml-engine-ledger`, four more for `aml-ledger`, a handful each for trust routing and the query view. Twenty incremental DDL scripts across four modules, and no production database to honour any of them. A consolidation was overdue.

The work itself was straightforward: collapse each module's migration history into a single initial schema. What made it worth doing now rather than later was the audit it forced. Reviewing the consolidated DDL against the actual JPA entities surfaced two categories of drift that incremental migrations hide.

First, orphaned columns. `case_ledger_entry` and `worker_decision_entry` both carried a `tenancy_id` column with an index — added in V3002 and V3003 when tenancy support first landed. But with JOINED inheritance, `tenancy_id` is mapped on the parent `ledger_entry` table by `JpaLedgerEntry`. The child table columns were never read by Hibernate. They existed because the migration that added them predated the schema reorganisation that moved tenancy to the parent, and nothing ever cleaned them up. Incremental migrations accumulate; they don't consolidate.

Second, missing primary keys. `aml_cbr_advisory_ledger_entry` and `aml_supervisor_decision_ledger_entry` had foreign keys to `ledger_entry(id)` but no explicit `PRIMARY KEY` constraint — unlike the other six child tables, which all had them. The original incremental migrations simply omitted them. Hibernate tolerates this in H2; PostgreSQL would too, since the FK target is itself a PK. But it's technically incomplete DDL, and consolidation was the right moment to fix it.

Twenty-one files became four. The schema is now a single-page read per module.

Separately, I wrote ADR-0004 to document a decision that had been made implicitly but never recorded: entity identifiers in `domainContentBytes()` — transaction IDs, account IDs — participate in the Merkle leaf hash and are exempt from GDPR erasure under Art.17(3)(b). The AML record retention obligation under the Bank Secrecy Act and EU 4th AML Directive is the legal basis. The existing actor-token erasure mechanism handles the personal data dimension without touching the financial evidence chain.

The contingency spec sits alongside the ADR: if a jurisdiction determines the exemption doesn't hold, what would content redaction actually require? Deterministic redaction tokens, a foundation-level `ContentRedactionService` SPI, and Merkle chain re-computation — O(log N) per affected entry, but subtle enough that an off-by-one in tree traversal corrupts the entire chain. The chameleon hash alternative was considered and rejected: a trapdoor key that lets anyone silently modify entries undermines the tamper-evidence property that FinCEN requires. The audit trail of what was redacted is more valuable than the illusion of an unchanged chain.

The contingency may never activate. That's the point — documenting the design before the pressure arrives means the decision can be made on technical merits, not under regulatory deadline.
