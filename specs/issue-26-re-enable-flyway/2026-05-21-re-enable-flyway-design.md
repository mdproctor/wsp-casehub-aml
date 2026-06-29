# Design Spec — Re-enable Flyway in @QuarkusTest (Issue #26)
2026-05-21

## Context

Three upstream fixes unblocked Flyway in tests:
- **qhorus#174** — moved qhorus migrations from `db/migration/qhorus/` to `db/qhorus/migration/`
- **ledger#95** — moved ledger migrations from `db/migration/` to `db/ledger/migration/`
- **qhorus#180** — renamed qhorus's join-table migration from V1003 to V2000 (V1000–V1007 reserved for ledger)

With these path changes, each datasource's Flyway can scan a non-overlapping classpath location.

## Changes

### `app/src/test/resources/application.properties`

**Default datasource** — standard config per quarkus-test-database.md protocol:
```properties
quarkus.hibernate-orm.database.generation=none
quarkus.flyway.migrate-at-start=true
```
`classpath:db/migration` (Flyway default) finds only casehub-work migrations (V1–V27). No explicit `locations` needed.

**Qhorus datasource** — explicit locations to combine qhorus and ledger:
```properties
quarkus.hibernate-orm.qhorus.database.generation=none
quarkus.flyway.qhorus.migrate-at-start=true
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration
```
`db/qhorus/migration/` — qhorus domain schema (V1–V10, V2000)
`db/ledger/migration/` — ledger base schema (V1000–V1007), needed because `casehub.ledger.datasource=qhorus`

Remove all `drop-and-create` and `migrate-at-start=false` lines and their blocking comments.

### `CLAUDE.md` — test schema note

Replace the "Flyway disabled" note in the test-driven development section with the correct pattern:
- Flyway is enabled on both datasources
- The qhorus datasource requires explicit `locations` because it manages both qhorus and ledger entities
- Document the `classpath:db/qhorus/migration,classpath:db/ledger/migration` pattern for future sessions

## Success Criteria

`JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode verify -pl app -am` passes 19/19 with:
- No `drop-and-create` in test config
- No `migrate-at-start=false` in test config
- Flyway running real migrations on both H2 databases