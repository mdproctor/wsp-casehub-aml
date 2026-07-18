# Handoff — CBR case similarity model complete (2026-07-18)

## What this project is

CaseHub AML — anti-money laundering investigation application built on CaseHub platform. 9 foundation layers complete. Frontend migrated to Lit + blocks-ui. CBR domain model now wired to neocortex infrastructure.

## This session (2026-07-18)

Closed #93 (case similarity model) — first CBR epic (#92) issue. Typed `FlagReason` enum replacing `String` on `SuspiciousTransaction` (migrated ~30 call sites). `CaseProfile` record with `initial()` (3 dims: flag reason, amount, prior incidents) and `complete()` (+ entity type, jurisdiction risk, network complexity) factories. `toFeatures()` bridges to neocortex `FeatureValue` for `CbrSimilarityScorer`. `AmlCbrSchema` with weighted `CategoricalTable` similarity specs (8 flag pattern pairs, 4 entity type pairs) and `GaussianDecay` for amount/incident numerics. Schema auto-registered at startup. Design spec went through adversarial review (13 findings, all verified, $13.23).

Also fixed pre-existing Flyway V2002 collision with qhorus SNAPSHOT — renumbered AML engine-ledger to V3000+.

**Commit on main:** `d9d5c0c` feat(#93): case similarity model — dimensions, profile, schema

## Immediate next step

Pick up #94 (case profile store — case-level indexing in CaseMemoryStore). The `CaseProfile.toFeatures()` and `AmlCbrSchema` are ready to wire into `CbrCaseMemoryStore.store()`.

## What's left (backlog)

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| 94 | Case profile store — case-level indexing in CaseMemoryStore | M | Med | Unblocked by #93 |
| 95 | CBR Retrieve — similarity search against case base | M | Med | Depends on #94 |
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

- Spec: `docs/specs/2026-07-18-case-similarity-model-design.md` (in project repo, also posted on #93)
- Blog: `blog/2026-07-17-mdp01-six-columns-that-matter.md`
- Epic: #92 (CBR epic), #111 (showcase UX)
