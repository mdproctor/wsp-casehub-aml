# Design Journal — issue-26-re-enable-flyway

## 2026-05-21 — Flyway locations must be pinned explicitly on all datasources §Architecture

The core decision this issue: always set `quarkus.flyway.locations` explicitly rather than relying on the Quarkus default (`classpath:db/migration`).

**Why:** Flyway's classpath scan is recursive. Any future dependency jar that adds a `db/migration/` entry is silently picked up, potentially causing V-number conflicts or applying unexpected migrations. The qhorus datasource hit exactly this: the extension default only specified `db/qhorus/migration`, missing the ledger migrations that the qhorus PU also needs (`casehub.ledger.datasource=qhorus`).

**What this means for aml:**
- Default datasource: `classpath:db/migration` (pinned — casehub-work migrations only)
- Qhorus datasource: `classpath:db/qhorus/migration,classpath:db/ledger/migration` (overrides extension default; ledger entities live on this PU)

**Propagated to:** `quarkus-test-database.md` protocol (universal rule) and both `application.properties` files (test and main).
