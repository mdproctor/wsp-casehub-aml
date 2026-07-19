# Handoff — case profile store complete (2026-07-19)

## What this project is

CaseHub AML — anti-money laundering investigation application built on CaseHub platform. 9 foundation layers complete. CBR case profile store (#94) now wired — completed investigations are stored in CbrCaseMemoryStore on SAR verdict with tamper-evident ledger audit.

## This session (2026-07-19)

Closed #94 (case profile store). `AmlCaseProfileStoreObserver` fires on `SarOutcomeRecordedEvent`, extracts `CaseProfile` dimensions + investigation path from engine context, stores via `CbrCaseMemoryStore`, writes `AmlCaseProfileLedgerEntry` (V3005). Schema gained `semanticText("sar_narrative")` for embedding-based retrieval. Adversarial design review caught 15 issues ($21.27) — most critical: `@ObservesAsync` would silently never fire (event uses `.fire()`), `CaseInstance.getCompletedPlanItems()` doesn't exist, `FeatureField.text()` creates non-semantic field. Code review caught `valueOf()` crash on malformed context values.

Also fixed pre-existing SNAPSHOT dependency cache corruption (deleted all casehub JARs; rebuilt from source + GitHub Packages).

**Commit on main:** `0e8d85b` feat(#94): case profile store — CBR retain on SAR outcome

## Immediate next step

Pick up #95 (CBR Retrieve — similarity search against case base). The case base now has entries to search against.

## What's left (backlog)

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 95 | CBR Retrieve — similarity search against case base | M | Med | Unblocked by #94 |
| 96 | CBR Reuse — investigation path adaptation | L | High | Depends on #95 |
| 97 | CBR Retain — outcome indexing on case completion | M | Med | Can parallel with #96 |
| 98 | SAR narrative seeding from similar past cases | M | Med | Depends on #95 |
| 99 | Cold-start case base seeding | M | Med | Independent |
| 105 | Case Timeline component in Overview tab | M | Med | |
| 106 | Trust Score visualization in Routing & Trust tab | M | Med | |
| 107 | Officer Review — work-item-detail integration | M | Med | |
| 109 | Compliance nav — row selection shows detail | M | Med | |
| 110 | Domain worker workbench | XL | High | Needs brainstorm |

## Cross-repo state

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## References

- Spec: `docs/specs/2026-07-19-case-profile-store-design.md` (in project repo, also posted on #94)
- Blog: `blog/2026-07-19-mdp01-teaching-the-case-base.md`
- Epic: #92 (CBR epic)
