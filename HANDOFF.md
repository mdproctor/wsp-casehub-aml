*Updated: #43 closed — removed from Wave 3 backlog.*

# Handoff — CI green, wave plan ready (2026-08-01)

## What this project is

CaseHub AML — anti-money laundering investigation application. 9 foundation layers complete. Full CBR cycle wired. 322/322 tests green locally (51s).

## This session

Fixed 5 layers of CI failures accumulated since July 19: removed unnecessary sibling yarn checkout, resolved Maven dependency convergence, fixed stale FlagReason enum, propagated engine blackboard→planning rename, updated all integration tests for #112 triage flow. Root-caused flaky coldStart test to CBR store contamination — fix: `eraseByScope(Path.root(), TENANT)`.

**Commits on main:** `157a296`→`6640a4d` (7 commits on #117, #119)

## Immediate next step

Spin up Wave 1 slots: `work-slot create` for #113, #115, #81, #87 — all independent S-scale items.

## Cross-module

**Blocked by:**
- `engine` — `GateRequired` constructor gained `QuorumConfig` param (engine#810), not yet published to GitHub Packages. Gates AML CI green.
- `work` — `casehub-work-engine-adapter` blackboard→planning fix (work#322) not yet published. Gates AML CI green.

## Wave plan

### Wave 1 — quick wins (S-scale, independent, ~1 session each)

| # | Description | Epic | Scale | Complexity |
|---|-------------|------|-------|------------|
| 113 | Typed input retrofit for sarDrafting/complianceReview | CBR | S | Low |
| 115 | Generalize DecisionContextSanitiser → ContentSanitiser | CBR | S | Low |
| 81 | Automated retention expiry job | GDPR | S | Low |
| 87 | Trust score historical trend persistence | Trust | S | Med |

### Wave 2 — parallel pairs (M-scale)

| # | Description | Epic | Scale | Complexity |
|---|-------------|------|-------|------------|
| 99 | Cold-start CBR seeding | CBR | M | Med |
| 105 | Case Timeline component | UI | M | Med |

### Wave 3 — close epics

| # | Description | Epic | Scale | Complexity |
|---|-------------|------|-------|------------|
| 114 | LLM sar-drafting narrative adaptation | CBR | M | Med |
| 106 | Trust Score visualization | UI | M | Med |
| 116 | Quality dashboard — UPHELD-rate segmentation | CBR | M | Med |
| 107 | Officer Review — work-item-detail integration | UI | M | Med |
| 109 | Compliance nav — row selection | UI | M | Med |

## References

- Blog: `blog/2026-08-01-mdp01-five-layers-of-red.md`
- Garden: GE-20260716-986cd1 (CBR store contamination — revised), GE-20260801-de318e (Quartz threads), GE-20260801-aaa398 (FlagReason flow)
