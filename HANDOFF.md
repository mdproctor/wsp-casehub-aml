# Handoff — Layer 9: attestation reconciliation + SAR officer reviewed (2026-06-09)

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-07 → 2026-06-09)

Issues #44, #55, and #56 implemented and merged to main.

1. **#56 — engine path never wrote COMPLIANCE_REVIEW_OPENED**: `ComplianceReviewLifecycle.openReview()` now writes the ledger entry internally, so both the Layer 3 sync path and the engine (Quartz) path always write it. `caseId` is obtained from `WorkerExecutionContext.current().caseId()` — not `caseHub.signal()` which turned out to be async (Vert.x event bus, fire-and-forget).

2. **#44 — observer failure reconciliation**: `AmlTrustRoutingObserver` hardened with PP-20260530-49856c double-try/catch — failures write `observerFailed=true` entries instead of disappearing silently. New `AmlAttestationReconciler` fills attestation gaps lazily on compliance evidence read, copying authoritative data from `WorkerDecisionEntry`.

3. **#55 — SAR officer reviewed / GDPR demo**: New `AmlSarOfficerReviewedLedgerEntry` + `AmlWorkItemLifecycleObserver` writes the officer's decision (APPROVED/REJECTED) with the officer's human `actorId`. GDPR Art.17 erasure now has real PII to act on. Full test: start investigation → officer approves WorkItem → erase officer actorId → verify pseudonymized.

Post-merge: casehub-ledger SNAPSHOT added `tenancyId` to all `LedgerEntryRepository` and `LedgerVerificationService` methods. All AML call sites updated to `TenancyConstants.DEFAULT_TENANT_ID`. CLAUDE.md updated.

CI green. 142 tests pass.

## Immediate next step

Run `/work` to start **#42 (ActionRiskClassifier)** or **#57 (production partial unique index for `UQ_TRUST_ATTEST_CASE_CAP_RECONSTRUCTED`)**.

Check `casehubio/engine#402` state before #42 — ActionRiskClassifier platform SPI in progress there.

## What's left

- `issue-13-remove-test-workarounds` workspace branch — closed issue, delete · XS · Low
- `issue-16-23-28-22-aml-test-batch` workspace branch — open, last commit 2 weeks ago, no project counterpart · XS · Low
- Stale workspace branches (closed): issue-26, issue-17, issue-30 — all past deletion date · XS · Low
- aml#57 — apply partial unique index `UQ_TRUST_ATTEST_CASE_CAP_RECONSTRUCTED` on production PostgreSQL (dropped from V2009 — H2 limitation) · XS · Low

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #42 | ActionRiskClassifier oversight gate for consequential AML actions | M | Med | Check casehubio/engine#402 status first |
| #14 | Layer 9 — LLM supervisor mode (investigation triage) | L | High | Needs engine#101 (LlmPlanningStrategy); CaseContextProvider → engine#419 |

## References

- Architecture record: `ARC42STORIES.MD` (project root) — 2 stale entries fixed this session
- Blog: `blog/2026-06-09-mdp01-when-the-examiner-looks.md`
- Garden entries: GE-20260609-ddd4b8 (CaseHub.signal async), GE-20260609-84290d (WorkItemLifecycleEvent.source()), GE-20260609-bc8704 (H2 partial index), GE-20260609-45bd4c (@ActivateRequestContext Quartz)
- API break: `LedgerEntryRepository`/`LedgerVerificationService` now require `tenancyId` — use `TenancyConstants.DEFAULT_TENANT_ID`
