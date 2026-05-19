---
layout: post
title: "Layer 2: the compliance officer's inbox"
date: 2026-05-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
excerpt: "Layer 2 of casehub-aml wires in casehub-work for compliance officer task routing — settling the hexagonal architecture question of where orchestration belongs and using @DefaultBean to make the richer implementation win automatically."
---

The core of Layer 2 isn't the code — it's the architectural question that gets settled first. `api/` and `app/` are the two Maven modules. When `casehub-work` enters as a dependency, where does the orchestration interface live?

Three options: put it in `api/` and accept a foundation dependency in the domain module; put it in `app/` and keep `api/` pure; or skip the interface and let the REST resource orchestrate directly. None is obviously wrong.

I searched the literature. The Quarkus hexagonal architecture guide, Red Hat's clean architecture article, and Milan Jovanovic's use-case layer guide all converge on the same answer: the domain layer (`api/`) is the innermost layer and carries zero dependencies on external frameworks. Orchestration belongs in the application layer (`app/`). The domain owns entities and specialist service interfaces. The application layer owns the use-case class that composes them with `WorkItemService`.

I captured that as a platform protocol — PP-20260512-9b8847, raised against the parent repo. Every casehub application follows the same split.

The CDI wiring turned out to be elegant. Quarkus has a `@DefaultBean` annotation: a bean marked `@DefaultBean` activates only when no other bean of the same type exists. `NaiveAmlInvestigationService` gets `@DefaultBean`. `WorkItemAmlInvestigationService` gets plain `@ApplicationScoped`. When both are present, CDI picks the non-default bean automatically — no config switch, no `@Alternative @Priority`.

```java
@ApplicationScoped
@DefaultBean
public class NaiveAmlInvestigationService implements AmlInvestigationApplicationService { ... }

@ApplicationScoped
public class WorkItemAmlInvestigationService implements AmlInvestigationApplicationService { ... }
```

The richer implementation simply exists and wins.

The WorkItem itself is straightforward. After the SAR narrative is drafted, `WorkItemAmlInvestigationService` creates a WorkItem with `candidateGroups = "compliance-officers"` and `claimDeadline = Instant.now().plus(30, ChronoUnit.DAYS)`. The compliance officer's inbox now contains a formal obligation with a 30-day window.

One non-obvious config issue surfaced. Adding `io.casehub.work.runtime.model` to `quarkus.hibernate-orm.packages` isn't enough. `casehub-work` also ships `FilterRule` in `io.casehub.work.runtime.filter` — a separate package. The error message names `FilterRule` clearly enough, but gives no hint that a second scan package is needed. Claude hit the error in the first test run and traced it to the missing package.

We also caught a design problem in review: the first version of `WorkItemAmlInvestigationService` duplicated the entire investigation orchestration from `NaiveAmlInvestigationService`. Fine for now, brittle for Layer 3 when specialist implementations change. We fixed it — `WorkItemAmlInvestigationService` now injects `NaiveAmlInvestigationService` by concrete type and delegates the investigation step entirely. Layer 2 adds one thing. Everything else is Layer 1's responsibility.

Eight tests pass. The REST endpoint returns the full investigation summary alongside a `complianceReviewTaskId`.
