# Handoff — issue-32 closed, Layer 8 shipped (2026-06-05)

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-04 → 2026-06-05)

Layer 8 (CaseMemoryStore integration) designed, implemented, reviewed, and merged.
19 commits squashed to 7. casehubio/aml#32 closed. casehubio/aml main updated on both fork and upstream.
Three garden entries submitted (Merkle frontier cascade GE-20260605-4ed3bd, YAML binding double-dispatch GE-20260605-059dd0, investigation drain technique GE-20260605-c91317).
Two new casehub protocols: PP-20260604-f45c95 (hash-chain disabled in H2 tests), PP-20260604-820c35 (investigation drain pattern).
CLAUDE.md updated with Layer 8 Foundation Layer status and investigation @QuarkusTest conventions.
ARC42STORIES.MD Layer 8 entry added; engine#396 stale ref cleared.
Parent repo issue casehubio/parent#173 filed for PLATFORM.md + casehub-aml.md sync.

Production build fixes: TrustRoutingPolicy gained `bootstrapEscalationRequired` boolean (engine SNAPSHOT); CDI exclusions added for DefaultTestPrincipal and NoOpGroupMembershipProvider.

## Immediate next step

Two stale workspace branches need triage before starting new work:
- `issue-13-remove-test-workarounds` — overdue, no project counterpart → propose deletion
- `issue-26-re-enable-flyway` — was due 2026-06-04, no project counterpart → propose deletion

Then run `/work` to start **#42 (ActionRiskClassifier)** or **#44 (observer failure reconciliation)**.

## What's left

- `issue-13-remove-test-workarounds` workspace branch — overdue, no project counterpart · XS · Low
- `issue-26-re-enable-flyway` workspace branch — overdue, no project counterpart · XS · Low
- Deferred GDPR demo: `AML_SAR_OFFICER_REVIEWED` ledger event — casehubio/aml#44 · S · Low
- casehubio/parent#173 — PLATFORM.md + casehub-aml.md doc sync (peer repo, session-end filed)

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #42 | ActionRiskClassifier oversight gate for consequential AML actions | M | Med | Check casehubio/engine#402 status first |
| #44 | Observer failure reconciliation — detect missing trust attestations at case close | M | Med | Silent evidence gaps |
| #14 | Layer 9 — LLM supervisor mode (investigation triage) | L | High | Needs engine#101 (DSL + LlmPlanningStrategy); CaseContextProvider separated to engine#419 |
| #51 | Merkle hash chain integration test with PostgreSQL | M | Med | Closed — concern belongs in casehub-ledger, not AML |

## References

- Architecture record: `ARC42STORIES.MD` (project root)
- Layer 8 garden entries: GE-20260605-4ed3bd (Merkle cascade), GE-20260605-059dd0 (YAML double-dispatch), GE-20260605-c91317 (drain technique)
- Protocols: casehub/ledger-hash-chain-disabled-in-h2-tests.md (PP-20260604-f45c95), casehub/engine-investigation-test-drain.md (PP-20260604-820c35)
- Parent docs issue: casehubio/parent#173 (PLATFORM.md + casehub-aml.md Layer 8 sync)
