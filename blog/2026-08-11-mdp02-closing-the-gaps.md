---
layout: post
title: "Closing the gaps nobody noticed"
date: 2026-08-11
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [trust-routing, compliance, failure-handling, epic-audit]
---

The AML harness had nine completed foundation layers and looked healthy from the commit log. A systematic audit told a different story: five epics that were done but still open, three more that had been partially implemented and then forgotten, and six that were genuinely blocked on upstream work. The gap between "implemented" and "closed" had been growing invisibly.

The audit itself was mechanical — fan out verification agents across the codebase, cross-reference against git history, check whether what the issues described actually exists. Five epics closed immediately: project scaffold, CasePlanModel, trust-weighted routing, tutorial layers 1–7, and the CBR cycle. All had been complete for weeks. The issues just hadn't been updated.

The interesting work was in the three partially-done epics.

**Trust dimensions were 33% complete.** The domain model defined three trust dimensions — `investigation-accuracy`, `pep-clearance`, `scope-awareness` — but only the first had feedback paths. `pep-clearance` tracks whether OSINT agents correctly handle PEP entities: when a SAR outcome arrives for a case involving a PEP, the feedback service now writes a `pep-clearance` attestation against the OSINT worker. `scope-awareness` tracks whether agents decline correctly when outside their clearance: a positive attestation fires when the case context shows a declined outcome. Both dimensions have quality floors in the routing config — OSINT screening needs a 0.60 pep-clearance floor, senior-analyst-review needs 0.65.

The implementation required reading entity type from the engine's `CaseInstanceCache` rather than adding fields to `SarOutcome`. This kept the public API unchanged while giving the feedback service access to the full investigation context.

**Compliance escalation had the 30-day SLA but no teeth.** The WorkItem was created with `claimDeadline = Instant.now().plus(30, ChronoUnit.DAYS)`, and the FinCEN clock was ticking, but nothing happened when it expired. A `ComplianceEscalationService` now creates an URGENT-priority WorkItem targeted at `senior-compliance-officers` when the original expires. The compliance review also gained `permittedOutcomes` — file, clear, or escalate — giving the officer a structured decision rather than a free-text resolution.

**Failure handling existed only for DECLINE.** The OSINT agent's decline path worked correctly — it wrote `declined: true` to the case context and the investigation continued. But no bindings existed for actual worker failures. Three new YAML bindings cover the gap: `pattern-analysis-retry` fires on first failure, `pattern-analysis-escalation` fires when the retry also fails (routing to senior analyst), and `osint-screening-failed-escalation` catches OSINT worker errors. An `investigation-stalled` failure goal completes the case when stall detection flags an investigation stuck in WAITING state beyond four hours.

None of this was architecturally novel — the foundation already supported all of it. The gaps existed because the tutorial layers focused on the happy path and deferred the error paths. Closing them makes the harness credible as a compliance demonstration rather than just a technical showcase.
