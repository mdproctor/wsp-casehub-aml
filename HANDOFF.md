# Handoff — CBR retain+reuse complete (2026-07-21)

## What this project is

CaseHub AML — anti-money laundering investigation application built on CaseHub platform. 9 foundation layers complete. Full CBR cycle now wired: Retain (#94), Retrieve (#95), Retain outcome indexing (#97), and Reuse path adaptation (#96).

## This session (2026-07-21)

Closed #97 (CBR Retain — outcome indexing) and #96 (CBR Reuse — path adaptation). Refactored retention from `FeatureVectorCbrCase` to `PlanCbrCase` with structured `PlanTrace` entries. Changed trigger from `SarOutcomeRecordedEvent` to `CaseOutcomeObserver` SPI. Added investigation triage flow (stub) with non-SAR exit path (`FALSE_POSITIVE`, `INCONCLUSIVE`). Built `CbrPathAdvisorWorker` producing capability-oriented statistics. Senior-analyst binding now fires on CBR frequency >60%. Filed #112 for real triage logic. Design spec adversarially reviewed (5 rounds, 17 issues resolved).

**Commits on main:** `cdc35f8`, `9347175`, `54bcd51`

## Immediate next step

Pick up #112 (investigation triage logic — replace SAR_WARRANTED stub) or #98 (SAR narrative seeding from similar past cases).

## What's left (backlog)

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 112 | Investigation triage logic — replace stub | M | Med | Unblocked by #97 |
| 98 | SAR narrative seeding from similar past cases | M | Med | Unblocked by #95 |
| 99 | Cold-start case base seeding | M | Med | Independent |
| 105 | Case Timeline component in Overview tab | M | Med | |
| 106 | Trust Score visualization in Routing & Trust tab | M | Med | |
| 107 | Officer Review — work-item-detail integration | M | Med | |
| 109 | Compliance nav — row selection shows detail | M | Med | |
| 110 | Domain worker workbench | XL | High | Needs brainstorm |

## References

- Spec: `docs/specs/2026-07-20-cbr-retain-reuse-design.md`
- Review: `~/adr/casehub-aml/cbr-retain-reuse-20260720-172920/`
- Garden: GE-20260721-4564db (CaseOutcomeObserver transaction gotcha)
- Epic: #92 (CBR epic)
