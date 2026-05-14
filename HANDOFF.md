# Handoff — casehub-aml Layer 2 Complete
2026-05-13

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Current state

Layers 1 and 2 are complete and pushed to `origin` (`casehubio/aml`).

**Layer 2 additions (on top of Layer 1):**
- `api/domain/AmlInvestigationResult.java` — pure record: `InvestigationSummary summary` + `String complianceReviewTaskId`
- `app/AmlInvestigationApplicationService.java` — use-case port interface
- `app/tutorial/NaiveAmlInvestigationService.java` — `@ApplicationScoped @DefaultBean`, returns result with null taskId
- `app/tutorial/WorkItemAmlInvestigationService.java` — `@ApplicationScoped`, delegates investigation to naive service, creates compliance WorkItem with `claimDeadline=now+30d`, `candidateGroups=compliance-officers`
- `AmlInvestigationResource` — injects interface, returns `AmlInvestigationResult`
- Hibernate scan packages: `io.casehub.work.runtime.filter` added alongside `runtime.model`
- 8 tests passing (4 unit + 4 `@QuarkusTest`)

**Architecture decision made this session:** hexagonal architecture — `api/` is pure domain (no external deps), `app/` owns use-case orchestration. Protocol PP-20260512-9b8847 raised to parent#18.

## Open issues to watch

| Issue | Repo | What |
|-------|------|------|
| #13 | casehubio/aml | Remove test workarounds (qhorus reactive + Flyway V2) when upstream fixed |
| #14 | casehubio/aml | Foundation blockers tracker — Layers 6 and 8 await engine P1.3 and LlmPlanningStrategy SPI |
| #16 | casehubio/aml | `GET /workitems/{id}` assertion missing from resource test (deferred spec deviation) |
| #17 | casehubio/aml | `@Transactional` on `investigate()` — evaluate at Layer 3 |
| #15 | casehubio/parent | Update casehub-aml.md (updated with Layer 2 state) |
| #18 | casehubio/parent | Add hexagonal application-service placement protocol (PP-20260512-9b8847) |
| #19 | casehubio/parent | Add casehub-work Hibernate scan package protocol |
| #168 | casehubio/work | Builder for `WorkItemCreateRequest` (19-field positional record) |

## Known workarounds

*Unchanged — `git show HEAD~1:HANDOFF.md` §Known workarounds*

## What to build next

**Layer 3** — add `casehub-qhorus`: typed `COMMAND/RESPONSE/DONE/DECLINE` per specialist agent. Closes the "no formal obligation per investigation task" gap. Epic: casehubio/aml#9.

Before starting: check for an active child issue under Epic #9 for Layer 3. If none, run issue-workflow Phase 1.

## References

- Spec: `specs/2026-05-13-layer2-casehub-work-design.md`
- Plan: `plans/2026-05-13-layer2-casehub-work.md`
- Blog: `blog/2026-05-13-mdp01-layer2-compliance-workitem.md`
- Garden: `GE-20260513-74dc72` (FilterRule scan package), `GE-20260513-4f26a7` (@DefaultBean layer displacement)
- Foundation status: *Unchanged — `git show HEAD~1:HANDOFF.md` §Foundation status*
