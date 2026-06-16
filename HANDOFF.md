# Handoff — Layer 9 closed (2026-06-16)

## What this project is

*Unchanged — `git show HEAD~3:HANDOFF.md` §What this project is*

## This session (2026-06-10 → 2026-06-16)

Layer 9 (`AmlActionRiskClassifier`, #42 + #57) shipped, merged, squashed, and closed.

Five cascading SNAPSHOT adaptations required before CI went green:
1. `WorkItemLifecycleEvent.fromWire()` 12-param API (casehub-work SNAPSHOT)
2. `domainContentBytes()` enforcement on all `@Entity LedgerEntry` subclasses (casehub-ledger feat/#128)
3. Timer job renames: `ExpiryCleanupJob`→`ExpiryTimerJob`, `ClaimDeadlineJob`→`ClaimDeadlineTimerJob`
4. `TenantScopedPrincipal` CDI exclusion — must be in BOTH main and test `application.properties` (quarkus:build uses main classpath only)
5. `AmlAttestationReconciler` missing `tenancyId` on reconstructed entries (hidden by engine#491 until investigations completed)

engine#491 (investigations never completing) filed and fixed by the engine session. ledger#130 (SYSTEM actor tokenisation exemption) changed `eraseActor_systemActor` test expectation to `mappingFound=false`.

Also: ADR-0003 (AmlActionType as gate metadata authority), protocol PP-20260615-d274cc (TenantScopedPrincipal exclusion), parent#252 filed (casehub-aml.md missing Layer 9 capabilities).

## Immediate next step

Branch is closed. Start **#14 (Layer 10 — LLM triage supervisor)** once engine#101 (LlmPlanningStrategy SPI) lands.

## What's left

| # | Description | Scale | Complexity | Blocked by | Notes |
|---|-------------|-------|------------|------------|-------|
| #14 | Layer 10 — LLM supervisor mode (investigation triage) | L | High | engine#101 (LlmPlanningStrategy) | CaseContextProvider → engine#419 |

## References

- Blog: `blog/2026-06-15-mdp01-five-days-to-the-other-side-of-a-snapshot.md`
- ADR: `docs/adr/0003-amlactiontype-as-gate-metadata-authority.md`
- Protocol: PP-20260615-d274cc
- Garden: GE-20260615-ffff65 (SNAPSHOT divergence), GE-20260531-70e07c REVISE, GE-20260601-60efe8 REVISE
- Outstanding: parent#252 (casehub-aml.md Layer 9 capabilities), #14 (LLM triage — engine#101)
- Architecture record: `ARC42STORIES.MD` + `LAYER-LOG.md` — Layer 9 entry written
