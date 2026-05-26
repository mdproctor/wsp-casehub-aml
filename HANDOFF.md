# Handoff — casehub-aml Layer 5 complete
2026-05-25

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## Current state

**Both repos on main.** Layer 5 shipped this session. Layer 6 is next.

**Completed this session:**
- casehubio/aml#31 closed: Layer 5 — adaptive investigation paths with casehub-engine
- `AmlInvestigationCaseHub extends YamlCaseHub` + `aml-investigation.yaml` — 5 bindings, YAML structure + programmatic workers
- PEP routing (senior-analyst-required binding fires on entityType=PEP), parallel OSINT+pattern, OSINT decline handling
- `EntityResolutionResult` extended: entityType + riskScore fields
- 42 tests passing; engine case reaches COMPLETED status
- ADR-0002: YAML bindings + programmatic workers (decision recorded)
- Blog: `2026-05-25-mdp01-parallel-by-default.md`
- Garden: 2 entries (WORKER_SCHEDULED metadata, engine-testing index-dependency)
- Parent issues filed: #65 (engine artifacts missing from BOM), #69 (casehub-aml.md Layer 5 status), #70 (PLATFORM.md cross-repo deps)
- CLAUDE.md: Layer 4 ✅ and Layer 5 ✅ marked complete

## Actioned from casehub-life session (2026-05-26)

- **aml#34 + aml#35 CLOSED** — commit `2a88356` on main: renamed all 5 `Naive*` classes to `Default*`, dissolved `tutorial/` package (all classes now in `io.casehub.aml` root), moved test classes, removed gap comments from production code, updated LAYER-LOG.md (Layer 1 heading, navigation lines on all layers, accountability gaps table, class name references). Build passes, 0 remaining `Naive` references.

## Issues filed from casehub-life session (2026-05-26)

- **aml#34** — add layer navigation index to LAYER-LOG.md: explicit `**Navigation:** git log --grep="#N"` line per layer entry so LLM sessions can jump directly to any layer boundary
- **aml#35** — rename all `Naive*` classes (`NaiveAmlInvestigationService` → `AmlInvestigationService` etc.), dissolve `tutorial/` package (these are default implementations, not tutorial-only code), update LAYER-LOG.md terminology ("domain baseline" not "naive Java"), remove any `// LAYER N GAP:` comments from production code

## Open issues

| Issue | What | Status |
|-------|------|--------|
| qhorus#190 | OutboundMessage needs inReplyTo + subjectId for PushAgentDispatch fan-out replies | open |
| aml#22 | fanOut() not triggering PushAgentDispatch.post() | skip — qhorus refactoring ongoing |
| parent#48 | casehub-qhorus.md MessageDispatch API docs | needs parent session |
| ~~parent#65~~ | ~~Add engine artifacts to BOM~~ | ✅ closed by parent session 2026-05-26 |
| ~~parent#69~~ | ~~casehub-aml.md: Layer 5 → complete~~ | ✅ closed by parent session 2026-05-26 |
| ~~parent#70~~ | ~~PLATFORM.md: casehub-aml → casehub-engine cross-repo deps~~ | ✅ closed by parent session 2026-05-26 |
| ~~aml#20~~ | ~~Flyway conflict~~ | ✅ closed — conflict doesn't exist; real fix tracked in work#229 (db/work/migration rename) |
| ~~aml#33~~ | ~~CI repository_dispatch trigger~~ | ✅ closed — trigger was already present |
| ~~aml#18~~ | ~~Adopt agentic harness framing~~ | ✅ closed — framing and LAYER-LOG already in place |

## What to build next

**Layer 6** — trust routing: experienced analysts routed to complex cases based on SAR outcome attestations. Requires engine#336 (trust scores in SelectionContext) and engine#337 (WorkOrchestrator resolves WorkerSelectionStrategy via CDI priority).

**Immediate next step:** check epic #9 for a Layer 6 child issue (`gh issue view 9 --repo casehubio/aml`). None exists — create one via issue-workflow Phase 1.

## References

- Blog: `blog/2026-05-25-mdp01-parallel-by-default.md`
- ADR: `adr/0002-case-definition-yaml-with-programmatic-workers.md`
- Spec: `docs/specs/2026-05-24-layer5-engine-design.md`
- LAYER-LOG: Layer 5 entry added with 4 gotchas and replicate pattern
