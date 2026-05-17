---
layout: post
title: "The broken promise in Layer 2"
date: 2026-05-16
type: pivot
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [aml, quarkus, architecture, cdi]
---

The CDI problem surfaced immediately. Layer 3 adds casehub-qhorus —
typed COMMAND/RESPONSE/DONE/DECLINE per specialist agent. The natural
implementation would be a new service implementing
`AmlInvestigationApplicationService`. But `WorkItemAmlInvestigationService`
already implements it without `@DefaultBean`. Two non-default CDI beans,
one interface: startup failure.

The first fix I looked at was an inner interface — `AmlInvestigator` — that
Layer 3's qhorus service would implement. `WorkItemAmlInvestigationService`
would inject it by interface. Clean displacement, nothing else changes.

I kept pushing. If cost and churn don't matter, is this the best architecture
or just the easiest fix? Claude went further and proposed a composer: a stable
`AmlInvestigationCoordinator` composing `AmlInvestigator` (swappable) and a
`ComplianceReviewLifecycle` that extracts the WorkItem concern from Layer 2.
Better separation, cleaner coordinator, explicit naming of what each layer adds.

I pushed again. This is a significant refactor of Layer 2's code. Before
committing, I wanted the full investigation — git history, specs, blog entries.

What Claude found settled it. Commit `22a0f6d` — the Layer 2 refactor that changed
`WorkItemAmlInvestigationService` to delegate to `NaiveAmlInvestigationService` —
carries this message:

> *"Changing specialist implementations in Layer 3 will propagate automatically."*

That promise is broken by this:

```java
@Inject
NaiveAmlInvestigationService naiveInvestigation;  // concrete type
```

CDI `@DefaultBean` displacement works at the interface level. Whatever Layer 3
introduces, it can't propagate into a field typed as `NaiveAmlInvestigationService`.
The Layer 2 refactor understood the right design — investigation and compliance
lifecycle are separate concerns, and swapping the investigator should be transparent
to the WorkItem layer. It just didn't complete the thought. The concrete type was a
workaround that worked for Layer 2 but made the Layer 3 promise impossible to deliver.

An `AmlInvestigator` interface is the correct completion: Layer 2's intent,
formalised. The remaining question is whether to keep WorkItem creation inline
in the coordinator or extract it as `ComplianceReviewLifecycle`.

The Layer 4 argument for extracting it — "WorkItemAmlInvestigationService will
accumulate ledger writes" — turns out to be weaker than expected. Tutorial-strategy
Layer 4 says every agent message and every WorkItem transition writes a ledger entry
automatically via qhorus and casehub-work-ledger infrastructure. AML may not write
explicit coordinator-level entries at all. The accumulation concern may not materialise.

The argument that holds is simpler. `ComplianceReviewLifecycle` encapsulates all
of AML's specific knowledge about the compliance review obligation: the 30-day
FinCEN SLA, the compliance-officers candidate group, the caller reference format.
That's a named domain concept, not boilerplate. Leaving it inline turns the
coordinator's method into infrastructure plumbing. Extracted, it reads as business
logic — investigate, open compliance review, return result.

The composer lands for that reason, not because of Layer 4 accumulation.
