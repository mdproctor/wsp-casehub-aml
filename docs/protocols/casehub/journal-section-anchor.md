---
id: PP-20260519-0692ff
title: "JOURNAL.md entries must carry a §SectionName anchor to be merged at epic close"
type: rule
scope: application
applies_to: "All CaseHub workspace repos using design/JOURNAL.md and the epic close workflow"
severity: important
refs:
  - design/JOURNAL.md
violation_hint: "An entry header with no §SectionName (e.g. '## 2026-05-17 — Title') will be silently skipped by the journal merge step — its narrative never reaches DESIGN.md"
created: 2026-05-19
---

Every entry in `design/JOURNAL.md` must include a `§SectionName` anchor in its header
line, where `SectionName` matches a `##` heading in `DESIGN.md`. The epic close
workflow scans for `§` to determine which DESIGN.md section each journal entry targets;
entries without it are silently skipped and their narrative is permanently lost. The
format in use in this workspace is `## YYYY-MM-DD — Title §SectionName` (H2 with
em-dash separator). The epic skill's hygiene check grep pattern (`^### .*·.*§`)
uses H3 and middot — verify against the actual format before running hygiene checks.
