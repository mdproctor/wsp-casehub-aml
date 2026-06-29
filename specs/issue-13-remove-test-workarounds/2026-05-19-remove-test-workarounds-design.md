# Design Spec — Remove Test Workarounds (Issue #13)
2026-05-19

## Problem

`app/src/test/resources/application.properties` contains two sets of workarounds for upstream bugs that were preventing `@QuarkusTest` from starting cleanly.

**Flyway V2 conflict workaround** — `flyway.migrate-at-start=false` + `drop-and-create` were added because of a reported conflict between casehub-work's and casehub-qhorus's V2 Flyway migrations. Analysis shows this conflict cannot occur: each V2 script lives in a separate scoped directory (`db/migration/` vs `db/migration/qhorus/`) scanned by its own named datasource with its own `flyway_schema_history` table. The "conflict" was a misdiagnosis. casehubio/qhorus#142 is closed; casehubio/work#162 is effectively moot and will be closed with this analysis.

**Reactive activation workaround** — `quarkus.datasource.reactive=false` + `quarkus.datasource.qhorus.reactive=false` suppress Hibernate Reactive, which qhorus currently activates unconditionally. casehubio/qhorus#141 (gate the reactive dependency behind a build-time flag) is still open. These properties remain until qhorus#141 is resolved. The three ledger services in `arc.exclude-types` are excluded for the same reason — they depend on `ReactiveLedgerEntryRepository`, which is vetoed in JDBC-only mode. They also stay.

## Changes

### `app/src/test/resources/application.properties`

**Remove (Flyway workaround):**
- `quarkus.flyway.migrate-at-start=false`
- `quarkus.flyway.qhorus.migrate-at-start=false`
- `quarkus.hibernate-orm.database.generation=drop-and-create`
- `quarkus.hibernate-orm.qhorus.database.generation=drop-and-create`
- `quarkus.flyway.locations=db/migration/default` — non-existent path used as second lock; remove entirely, use Quarkus default (`classpath:db/migration`)

**Restore to standard (per quarkus-test-database.md protocol):**
- `quarkus.flyway.migrate-at-start=true`
- `quarkus.flyway.qhorus.migrate-at-start=true`
- `quarkus.hibernate-orm.database.generation=none`
- `quarkus.hibernate-orm.qhorus.database.generation=none`

**Unchanged (reactive workaround — pending qhorus#141):**
- `quarkus.datasource.reactive=false`
- `quarkus.datasource.qhorus.reactive=false`
- `quarkus.arc.exclude-types` block (three ledger services)

## GitHub Housekeeping

- Close `casehubio/work#162` with isolation analysis: separate datasources + scoped directories = no conflict possible
- Update `casehubio/aml#13`: Flyway workaround removed; reactive workaround remains pending qhorus#141

## Success Criteria

`JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode verify -pl app -am` passes with 19 tests green.