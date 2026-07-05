# Handoff — #91 frontend done, #92 CBR epic filed (2026-07-05)

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-30 → 2026-07-05)

Built the full AML workbench frontend (Tasks 12–17 of #91): Quinoa setup with casehub-pages DSL, four sidebar views (work queue, investigations, accountability, operations), investigation flow iframe component with directed graph rendering. All datasets use inline mock data — real API binding is a one-line swap per dataset when foundation endpoints ship. Fixed three runtime issues: placeholder `_` URLs causing fetch failures, nested JSON metrics responses incompatible with pages-data tabular extraction, and Quinoa dev-mode serving from `target/quinoa/build/` not `dist/`. Branch `issue-91-aml-workbench-ui` pushed to origin (28 commits, not yet merged to main). Filed CBR epic #92 with 7 child issues (#93–#99).

## Immediate next step

Run `/work` → resume `issue-91-aml-workbench-ui` → run `/work end` to merge to main (squash the 28 commits). Or pick new work from backlog.

## What's left

- #91 branch needs `/work end` to merge — 28 commits on origin, not on main · XS · Low

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #92 | CBR epic — case-based reasoning for AML investigation | XL | High | Start with #93 (similarity model) |
| #81 | Automated retention expiry (ErasureReason.RETENTION_EXPIRED) | M | Med | |
| #82 | GDPR Art.22 decision record compliance supplements | M | Med | |
| #83 | Entity data erasure in tamper-evident ledger content | L | High | Deferred architectural concern |
| #84 | Cross-tenant entity memory erasure | M | Med | Only when multi-tenancy activates |

## References

- Blog: `blog/2026-07-05-mdp01-the-screen-that-lies.md`
- Spec: `specs/2026-06-30-aml-workbench-ui-design.md`
- Plans: `plans/2026-06-30-aml-workbench-frontend.md`
- Garden: GE-20260705-385e87 (Quinoa dev-mode serving from target/)
