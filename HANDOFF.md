# Handoff — Layer 9: ActionRiskClassifier oversight gate (2026-06-10)

*Updated: ledger#133, parent#219 closed — removed from blockers.*

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-09 → 2026-06-10)

*Unchanged — `git show HEAD:HANDOFF.md` §This session*

## Immediate next step

**Merge PR #60** — casehubio/aml#60 is squashed and ready. ledger#133 (tenancyId shadowing) is now CLOSED — the build blocker is gone. Merge it.

After merge: pick up aml#59 (work SNAPSHOT regression) so Layer 5–8 `@QuarkusTest` runs clean again.

## What's left

| # | Description | Scale | Complexity | Blocked by | Notes |
|---|-------------|-------|------------|------------|-------|
| #59 | SNAPSHOT: TenantScopedPrincipal displaces MockCurrentPrincipal | XS | Low | casehub-work fix | Layer 5–8 QuarkusTests failing |
| #14 | Layer 10 — LLM supervisor mode (investigation triage) | L | High | engine#101 (LlmPlanningStrategy) | CaseContextProvider → engine#419 |

## What's next

Merge PR #60, then fix aml#59 (work SNAPSHOT), then pick up #14 (LLM triage) once engine#101 lands.

## References

- PR: casehubio/aml#60 (open — squashed, ready to merge)
- Blog: `blog/2026-06-10-mdp01-when-the-gate-itself-is-wrong.md`
- Protocols: PP-20260610-66fc79 (fail-closed classifier metadata), PP-20260610-ae4535 (AML ledger tenancyId)
- Garden entries: GE-20260609-77a6f9 (TenantScopedPrincipal SNAPSHOT), GE-20260609-18a0b1 (quarkus.index-dependency double-scan), GE-20260609-bc9bab (ConcurrentHashMap JMM scope), GE-20260610-583563 (fail-closed classifier metadata)
- Outstanding: aml#59 (work SNAPSHOT), aml#14 (LLM triage — blocked on engine#101)
- Architecture record: `ARC42STORIES.MD` + `LAYER-LOG.md` — Layer 9 entry written
