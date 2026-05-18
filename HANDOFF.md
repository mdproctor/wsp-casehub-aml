# Handoff — casehub-aml Layer 3 Complete
2026-05-18

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Current state

**Epic branch:** `epic-layer3-qhorus` (both project repo and workspace)

Layers 1, 2, and 3 are complete. Layer 3 shipped this session.

**Layer 3 additions:**
- `SpecialistOutcome<T>` — sealed interface (Completed/Declined/Failed) in `api/`; `InvestigationSummary` and `SarDraftingService` updated to use it for all three specialists
- `AmlInvestigator` — inner interface separating investigation from compliance lifecycle
- `ComplianceReviewLifecycle` — WorkItem concern extracted from deleted `WorkItemAmlInvestigationService`
- `AmlInvestigationCoordinator` — stable outer coordinator; replaces Layer 2's service
- `QhorusAmlInvestigator` — sends COMMAND/DONE/DECLINE to qhorus; dispatches in-process to stub agent behaviours
- `AgentBehaviour`/`AgentDispatchMechanism` SPIs + three stub behaviours; `OsintScreeningBehaviour` always DECLINEs
- `AmlJacksonConfig` — `ObjectMapperCustomizer` mixin adds `"type"` discriminator to `SpecialistOutcome<T>` JSON
- 19 tests passing (`@QuarkusTest` starts cleanly after CDI fixes)

**Key CDI fixes applied this session:**
- `casehub.qhorus.reactive.enabled=false` removed — upstream bug resolved
- `LedgerVerificationService` chain excluded from test CDI via `quarkus.arc.exclude-types`
- `@Typed({AgentDispatchMechanism.class, PushAgentDispatch.class})` prevents ambiguity with `QhorusChannelBackend`

## Open issues to watch

| Issue | What |
|-------|------|
| #22 | `channelGateway.fanOut()` not triggering `PushAgentDispatch.post()` — root cause unknown |
| #23 | Verify qhorus COMMAND/DONE/DECLINE messages persisted in `@QuarkusTest` |
| #24 | Minor Layer 3 code quality items |
| #13 | Flyway V2 conflict workaround still in test config (reactive part resolved; Flyway part remains) |
| #18 | Meta: agentic harness framing — CLAUDE.md update still pending |

## What to build next

**Close the epic** — `epic-layer3-qhorus` is ready to close. LAYER-LOG.md Layer 3 entry is written, blog entry committed, docs synced.

**Then: Layer 4** — add `casehub-ledger` as the explicit FinCEN audit trail. The ledger is already on the classpath (transitively via qhorus). Layer 4's teaching point is demonstrating what's already being written to the Merkle chain. Before starting: investigate #22 (fan-out) — Layer 4 will want qhorus message ledger entries to be part of the story.

Before starting Layer 4: check for an active child issue under Epic #9. If none, run issue-workflow Phase 1.

## References

- Design spec: `specs/2026-05-17-layer3-composer-qhorus-design.md`
- Blog (architectural investigation): `blog/2026-05-16-mdp01-broken-promise-layer-2.md`
- Blog (implementation): `blog/2026-05-18-mdp01-layer3-five-surprises.md`
- LAYER-LOG.md: Layer 3 entry written (full wiring, all 5 CDI gotchas documented)
- Epic branch: `epic-layer3-qhorus` on both repos
