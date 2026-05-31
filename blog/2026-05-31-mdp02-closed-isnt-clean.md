---
layout: post
title: "Closed isn't clean"
date: 2026-05-31
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [aml, hygiene, branch-management]
---

The closure stamps were all there. Every branch had its `chore: branch closed` commit, every deletion date in the future, every project branch stamped. By that metric, everything was in order.

Promotion is a different check.

I ran a full hygiene audit today — workspace and project repos, all branches. Claude worked through the git history while I tracked the gaps. Fourteen blog entries, all published to `_notes/`. Good. But `2026-05-30-mdp01-two-mappers-one-exception.md` had been published directly from its branch and never made it to workspace main's `blog/` directory or INDEX.md. Three plans — Layer 6 trust routing, Layer 7 compliance evidence — sat on their closed branches and had never been copied to `plans/` on main. The Layer 7 compliance evidence spec hadn't been promoted to project `docs/specs/`. The `journal-section-anchor` protocol, written in May on `epic-layer3-qhorus`, had never been filed in the garden.

The `issue-43-layer7-comparison` project branch had a closure stamp and no remote. Local-only, no backup.

We fixed all of it: blog promoted, plans promoted, spec promoted, protocol filed in casehub's HARNESS-INDEX, issue-43 pushed to remote.

One thing worth noting: `git log main...$branch` gives misleading "ahead" counts for squash-merged branches. All eight closed project branches showed 7–36 commits ahead of main, which looked alarming. The code was there — we confirmed it by checking specific classes via `git ls-tree` (`AmlLayer7Resource`, `AmlTrustRoutingPolicyProvider`, etc.). The original branch SHAs don't match the rewritten squash commits on main, so they show as unmerged even when the code is identical. Not a real gap, just a diagnostic that doesn't work in a squash-merge workflow.

One genuine update: `casehubio/platform#27` (CaseMemoryStore SPI) closed. That unblocks aml#32. `casehubio/engine#402` (ActionRiskClassifier SPI) is still open; aml#42 waits.
