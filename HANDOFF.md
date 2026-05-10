# Handoff — casehub-aml Layer 1 Complete
2026-05-10

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Current state

Layer 1 (naive Java baseline) is complete and pushed to `origin` (`mdproctor/aml`).

**Project structure:**
```
casehub-aml/
├── pom.xml              casehub-aml-parent (packaging=pom)
├── api/                 casehub-aml-api — pure Java domain records + service interfaces
└── app/                 casehub-aml-app — Quarkus app, tutorial stubs, REST endpoint, tests
```

**What's in the code:**
- `api/`: `SuspiciousTransaction`, result records (EntityResolutionResult, PatternAnalysisResult, OsintResult, InvestigationSummary), four service interfaces
- `app/tutorial/`: 4 package-private naive stubs + `NaiveAmlInvestigationService` (public, 4 LAYER 1 GAP comments)
- `app/`: `AmlInvestigationResource` — `POST /api/investigations`
- Tests: 3 unit tests + 2 `@QuarkusTest` — all green

## Open issues to watch

| Issue | Repo | What |
|-------|------|------|
| #13 | casehubio/aml | Remove test workarounds (qhorus reactive + Flyway V2) when upstream fixed |
| #141 | casehubio/qhorus | qhorus pulls in hibernate-reactive unconditionally |
| #142 | casehubio/qhorus | Flyway V2 conflict with casehub-work |
| #162 | casehubio/work | Flyway V2 conflict with casehub-qhorus |
| #15 | casehubio/parent | Update casehub-aml.md — Layer 1 complete |
| #16 | casehubio/parent | Formalise multi-PU datasource config as a protocol |

## Known workarounds in test application.properties

`app/src/test/resources/application.properties` has two sets of workarounds tracked by aml#13. Until upstream fixes land, these must stay.

## What to build next

**Layer 2** — add `casehub-work`: `WorkItemRequest` with 30-day `claimDeadline` for the compliance officer SAR review. Closes the "no deadline tracking" gap comment. Epic: casehubio/aml#9.

Before starting: check for an active child issue under Epic #9 for Layer 2. If none, run issue-workflow Phase 1.

## References

- Plan: `plans/2026-05-08-layer1-naive-java.md`
- Blog: `blog/2026-05-10-mdp01-aml-first-code-layer-1.md`
- Foundation status: *Unchanged — `git show HEAD~1:HANDOFF.md` §Foundation status*
