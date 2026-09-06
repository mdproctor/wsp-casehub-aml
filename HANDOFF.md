# HANDOFF — casehub-aml

## Last Session

Rewrote the AML workbench spec from scratch. The June 2026 spec designed custom components; pages and blocks-ui have since shipped purpose-built equivalents for nearly everything. The new spec composes from existing components — 16 blocks-ui components used directly, 4 enhancements needed, 5 AML-specific components remain. 10 design decisions captured, standard review (2 rounds) applied. Implementation plan written: 8 batches, 12 tasks. Batch 1 (dockWorkbench layout shell) landed — `layout.ts` + `centre.ts` + `investigation-nav.ts` replace the old sidebar/hash routing.

## Immediate Next Step

Batch 2: right dock panels (audit-trail-viewer, compliance-summary, trust-workbench, findings) + investigation-context topic wiring so selecting an investigation updates all contextual panels.

## Cross-Module

- blocks-ui: `worker-task-pane` new component needed before Batch 6 (no issue filed yet)
- blocks-ui: push update support on `list-pane`, `work-item-inbox` needed before Batch 7 (no issue filed yet)

## Garden Entries Consulted

GE-20260810-cfc53d, GE-20260804-befd45, GE-20260804-84ac70, GE-20260805-e3211c, GE-20260804-24d409, GE-20260814-0d4123, GE-20260804-0959d2, GE-20260822-dd986e, GE-20260804-a89d3b, GE-20260823-211f3e

## References

- `specs/issue-111-workbench-spec-rewrite/2026-09-01-aml-workbench-v2-design.md` — v2 spec
- `specs/issue-111-workbench-spec-rewrite/decisions.md` — 10 design decisions
- `plans/2026-09-01-aml-workbench-v2.md` — implementation plan (8 batches)
- `blog/2026-09-01-mdp01-workbench-v2-composition-over-construction.md` — diary entry
