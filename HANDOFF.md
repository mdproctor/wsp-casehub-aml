# Handoff — CI green after a month (2026-06-07)

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-05 → 2026-06-07)

CI had been red since May 12. Three separate bugs were uncovered once the build barrier dropped:

1. **Maven Central 403** (#52): AML had no local `quarkus-bom` import — Maven hit GitHub Packages (with Bearer token) before Central, causing 403. Fixed by pinning `quarkus.platform.version` locally and adding the BOM to AML's own `dependencyManagement`. Added `cache: 'maven'` to CI workflow.

2. **Attestation sequence conflict** (#53): `AmlTrustRoutingAttestation` shared `caseId` as ledger subject with engine + AML domain entries. `nextSequenceNumber()` was scoped to attestation rows only, missed the case-opened entry at seq=1. Fixed with: (a) dedicated namespaced subject `UUID.nameUUIDFromBytes("aml-trust-routing-attestation:" + caseId)`, (b) per-subject lock held outside the transaction so REQUIRES_NEW commits before the next writer reads max.

3. **qhorus dtype-scope bug** (qhorus#253): `MessageLedgerEntryRepository.findLatestBySubjectId()` queried `FROM MessageLedgerEntry` — invisible to AML domain entries sharing the same subject. Filed + resolved upstream as a re-architecture: new `LedgerEntryJpaRepository` using `FROM LedgerEntry`.

4. **WorkerResult API** (#54): engine SNAPSHOT changed `Worker.Builder.function()` return type to `WorkerResult`. All 7 workers in `AmlInvestigationCaseHub` updated to `WorkerResult.of(Map.of(...))`.

casehubio/aml CI is green. CLAUDE.md updated with ledger subject isolation convention and WorkerResult note.

## Immediate next step

Run `/work` to start **#42 (ActionRiskClassifier)** or **#44 (observer failure reconciliation)**.

Check `casehubio/engine#402` state before #42 — ActionRiskClassifier platform SPI is in progress there.

## What's left

- `issue-13-remove-test-workarounds` workspace branch — overdue, no project counterpart · XS · Low
- `issue-26-re-enable-flyway` workspace branch — overdue, no project counterpart · XS · Low
- Deferred GDPR demo: `AML_SAR_OFFICER_REVIEWED` ledger event — casehubio/aml#44 · S · Low
- 11 other stale workspace branches (see epic hygiene) — all correspond to closed issues; safe to delete

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #42 | ActionRiskClassifier oversight gate for consequential AML actions | M | Med | Check casehubio/engine#402 status first |
| #44 | Observer failure reconciliation — detect missing trust attestations at case close | M | Med | Silent evidence gaps |
| #14 | Layer 9 — LLM supervisor mode (investigation triage) | L | High | Needs engine#101 (LlmPlanningStrategy); CaseContextProvider → engine#419 |

## References

- Architecture record: `ARC42STORIES.MD` (project root)
- Garden entries: GE-20260607-3747a1 (Maven Central 403), GE-20260607-1c0a05 (ledger dtype scope), GE-20260607-f0c53e (ARJUNA cascade), GE-20260607-b6d999 (ledger subject isolation), GE-20260607-067ace (lock-outside-tx), GE-20260607-200500 (H2 constraint phantom)
- qhorus fix: casehubio/qhorus#253 — LedgerEntryJpaRepository re-architecture
- Blog: `blog/2026-06-07-mdp01-green-after-a-month.md`
