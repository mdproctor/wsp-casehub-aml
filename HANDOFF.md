# Handoff — showcase UX polish complete (2026-07-18)

## What this project is

CaseHub AML — anti-money laundering investigation application built on CaseHub platform. 9 foundation layers complete. Frontend migrated from pages-DSL to Lit + blocks-ui web components.

## This session (2026-07-17 → 2026-07-18)

Closed epic #111 S-scale issues (#103, #104, #108). Investigation list reduced from 10 truncated columns to 6 high-signal columns with status badges, risk score percentages, and amount+currency merge. Column picker (⋮ button) verified working — was never broken, just needed `columnConfig` entries with `visible: false`. Interventions tab now has metrics dashboard (escalation/DECLINE/rejection/override counts + recent feed) above the action cards. Also fixed TypedRow `createdAt` accessor, mock data shape mismatches across all panels, EventSource mock for SSE, work-item-inbox identity property.

**Commits on main:**
- `0fa33ae` feat(#103,#108): showcase UX — column optimization, intervention metrics
- `b75b685` fix(#102): showcase TypedRow extraction + sidebar width
- `2298170` fix(#102): showcase element-guard + mock route fixes

## Immediate next step

Pick up M-scale showcase issues from epic #111 — #105 (Case Timeline), #106 (Trust Score viz), #107 (Officer Review integration), or #109 (Compliance row detail). Or start the larger #110 (domain worker workbench) with a brainstorm.

## What's left (backlog)

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 105 | Case Timeline component in Overview tab | M | Med | Audit trail data exists, needs visual timeline |
| 106 | Trust Score visualization in Routing & Trust tab | M | Med | Data exists from Layer 6 response |
| 107 | Officer Review — work-item-detail integration | M | Med | Depends on blocks-ui work-item-detail |
| 108 | ~~Operations Interventions tab~~ | — | — | Closed this session |
| 109 | Compliance nav — row selection shows detail | M | Med | work-item-inbox emits events, needs detail pane |
| 110 | Domain worker workbench — specialist task queue | XL | High | Needs brainstorm — the real "workbench" |
| 101 | Dual-mode datasets — inline mocks + live endpoints | M | Med | |
| 92 | CBR epic — case-based reasoning | XL | High | Start with #93 |
| 89 | WebSocket/SSE real-time updates | M | Med | |
| 86 | Authentication and role-based access | L | High | |
| 81 | Automated retention expiry | M | Med | |

## Cross-repo state

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## References

- Blog: `blog/2026-07-17-mdp01-six-columns-that-matter.md`
- Spec: `specs/2026-06-30-aml-workbench-ui-design.md`
- Epic: #111 (showcase UX gaps and domain worker workflow)
