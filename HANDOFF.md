# Handoff — Layer 9: ActionRiskClassifier oversight gate (2026-06-10)

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-09 → 2026-06-10)

Issues #42 and #57 implemented, PR #60 open at casehubio/aml.

**#42 — AmlActionRiskClassifier (Layer 9)**

`AmlActionType` enum encodes gate policy for five AML action types: `SAR_FILING` and `LAW_ENFORCEMENT_REFERRAL` always gate; `ENTITY_LINK_CREATION` and `ACCOUNT_RESTRICTION` gate on risk score ≥ 0.8 or any PEP entity; `TRANSACTION_BLOCKING` inverts (gates when confidence < 0.9). All gate metadata (`reversible`, `candidateGroups`, `scope`, `expiresIn`) lives on the domain type — fail-closed paths read from the type, not from hardcoded defaults (PP-20260610-66fc79, GE-20260610-583563).

New Layer 9 harness: `AmlOversightCaseHub` (YamlCaseHub with entity-link-proposal worker declaring `PlannedAction(ENTITY_LINK_CREATION)`), `AmlOversightCoordinator`, `AmlLayer9Resource`. New deps: `casehub-engine-work-adapter` + `casehub-engine-blackboard`. Gate test (`AmlLayer9ActionGateTest`) validates PEP entity triggers gate → approval → case completes; low-risk CORPORATE proceeds autonomously.

Completion detection uses `CaseInstanceCache.get(caseId).getState()` — resilient to concurrent `ledger_subject_sequence` INSERT race when multiple async `WorkerDecisionEvent` observers fire simultaneously.

**#57 — Production partial unique index**

`docs/sql/V2011__UQ_TRUST_ATTEST_CASE_CAP_RECONSTRUCTED.sql` — PostgreSQL-only DDL, run manually. H2 doesn't support partial indexes even in `MODE=PostgreSQL`.

**Post-session fixes**

- `AmlTrustRoutingObserver` + `AmlTrustAttestationRepository` — null-check `event.tenancyId()` and fall back to `TenancyConstants.DEFAULT_TENANT_ID` (ledger SNAPSHOT made `tenancy_id` NOT NULL; PP-20260610-ae4535).
- Two new protocols: `docs/protocols/application/action-risk-classifier-fail-closed-metadata.md` (PP-20260610-66fc79), `docs/protocols/aml/aml-ledger-entry-tenancy-id-non-null.md` (PP-20260610-ae4535).

## Immediate next step

**casehub-ledger#133 must be fixed before this project can build.** The current ledger SNAPSHOT added `LedgerProcessor.validateLedgerEntryFieldShadowing()` but `WorkerDecisionEntry` and `CaseLedgerEntry` still declare their own `tenancyId` field — removing the shadowing fields in the ledger repo will unblock all consumers.

After ledger#133 resolves: merge PR #60.

**aml#59 (casehub-work SNAPSHOT regression)** also blocks Layer 5–8 `@QuarkusTest` runs. `TenantScopedPrincipal @RequestScoped` displaces `MockCurrentPrincipal @DefaultBean`, leaving `caseInstance.tenancyId=null` on Vert.x event loop threads. Fix needed in casehub-work.

## What's left

| # | Description | Scale | Complexity | Blocked by | Notes |
|---|-------------|-------|------------|------------|-------|
| #59 | SNAPSHOT: TenantScopedPrincipal displaces MockCurrentPrincipal | XS | Low | casehub-work fix | Layer 5–8 QuarkusTests failing |
| #14 | Layer 10 — LLM supervisor mode (investigation triage) | L | High | engine#101 (LlmPlanningStrategy) | CaseContextProvider → engine#419 |

## What's next

Resolve casehubio/ledger#133 (tenancyId field shadowing) then merge PR #60. After that, pick up #14 (LLM triage) once engine#101 lands.

## References

- PR: casehubio/aml#60 (open — squashed, ready to merge once ledger#133 resolved)
- Blog: `blog/2026-06-10-mdp01-when-the-gate-itself-is-wrong.md`
- Protocols: PP-20260610-66fc79 (fail-closed classifier metadata), PP-20260610-ae4535 (AML ledger tenancyId)
- Garden entries: GE-20260609-77a6f9 (TenantScopedPrincipal SNAPSHOT), GE-20260609-18a0b1 (quarkus.index-dependency double-scan), GE-20260609-bc9bab (ConcurrentHashMap JMM scope), GE-20260610-583563 (fail-closed classifier metadata)
- Outstanding: casehubio/ledger#133 (tenancyId shadowing), casehubio/parent#219 (docs sync), aml#59 (work SNAPSHOT)
- Architecture record: `ARC42STORIES.MD` + `LAYER-LOG.md` — Layer 9 entry written
