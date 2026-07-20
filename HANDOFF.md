# Handoff — CBR retrieve complete (2026-07-20)

## What this project is

CaseHub AML — anti-money laundering investigation application built on CaseHub platform. 9 foundation layers complete. CBR Retain (#94) and CBR Retrieve (#95) now wired — completed investigations are stored in the case base on SAR verdict, and similar past cases are automatically retrieved at case startup via engine CbrConfig.

## This session (2026-07-20)

Closed #95 (CBR Retrieve). Filed engine#761 to wire `CbrRetrievalService` into the case startup lifecycle — the engine had the full retrieval pipeline but zero callers. With that landed, AML's retrieve became pure configuration: `CbrConfig` on `AmlInvestigationCaseHub` with 4 JQ features and weights, `CASE_LIFETIME` timing, `vectorWeight(0.0)`. Excluded `CbrCaseRetainObserver` from CDI (AML has custom retain). Adversarial design review caught phantom features (`jurisdictionRisk`, `networkComplexity` — no worker outputs these) and `vectorWeight` default bug (0.5 would halve all scores). Garden entry GE-20260720-6ea915 for the retain/retrieve coupling gotcha.

**Commit on main:** `5f70267` feat(#95): CBR Retrieve — configure CbrConfig for similarity search at case startup

## Immediate next step

Pick up #96 (CBR Reuse — investigation path adaptation from similar cases). The case base now has both storage and retrieval — bindings can reference `.cbrExperiences` to adapt investigation paths.

## What's left (backlog)

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 96 | CBR Reuse — investigation path adaptation | L | High | Unblocked by #95 |
| 97 | CBR Retain — outcome indexing on case completion | M | Med | Can parallel with #96 |
| 98 | SAR narrative seeding from similar past cases | M | Med | Unblocked by #95 |
| 99 | Cold-start case base seeding | M | Med | Independent |
| 105 | Case Timeline component in Overview tab | M | Med | |
| 106 | Trust Score visualization in Routing & Trust tab | M | Med | |
| 107 | Officer Review — work-item-detail integration | M | Med | |
| 109 | Compliance nav — row selection shows detail | M | Med | |
| 110 | Domain worker workbench | XL | High | Needs brainstorm |

## Cross-repo state

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## References

- Spec: `docs/specs/2026-07-20-cbr-retrieve-design.md` (in project repo, posted on #95)
- Blog: `blog/2026-07-20-mdp01-the-retrieve-that-wasnt.md`
- Engine: casehubio/engine#761 (CbrRetrievalService wiring)
- Garden: GE-20260720-6ea915 (CbrCaseRetainObserver coupling gotcha)
- Epic: #92 (CBR epic)
