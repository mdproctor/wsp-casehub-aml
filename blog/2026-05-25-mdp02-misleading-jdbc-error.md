---
layout: post
title: "The Misleading JDBC Error"
date: 2026-05-25
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [quarkus, jta, transactions, multi-datasource]
---

Before writing any code, I checked each issue against the current state.
Four of the six were already done.

`ComplianceReviewLifecycle` was already using `WorkItemCreateRequest.builder()` —
the builder had shipped in casehub-work. `QhorusAmlInvestigator` had already been
updated to import `ActorType` from `io.casehub.platform.api.identity`, not the
old ledger location. Both had been real issues once. Neither was real now.
Closed without touching a file.

The next issue asked for `@Transactional` on
`AmlInvestigationCoordinator.investigate()`. The reasoning was sound at Layer 2 —
one datasource, one transaction, atomic write. But this is Layer 3 and 4 now.
We added the annotation and ran the tests.

```
Unable to acquire JDBC Connection
[Exception in association of connection to existing transaction]
```

That error looks like connection pool exhaustion. It isn't.
`AmlInvestigationCoordinator.investigate()` delegates to services on two separate
Hibernate persistence units — `default` for `WorkItemService`, `qhorus` for
`LedgerWriteService` and `MessageService`. A single `@Transactional` at the
coordinator level asks JTA to enlist both in one transaction. H2 in the
dual-datasource test configuration doesn't provide XA-capable connections,
so the association fails.

Nothing in the error message mentions XA or multiple datasources. Without that
context it reads as a pool problem.

The fix is to revert. Each service already owns its transaction boundary. If
true cross-datasource atomicity ever becomes necessary, that's a saga or outbox
pattern, not an annotation on the coordinator. We closed the issue as won't-do
and captured the constraint in the garden.

The remaining fixes were less interesting. We added the GET `/workitems/{id}`
assertion that had been deferred since Layer 2 — `candidateGroups` and
`claimDeadline` at the HTTP layer, the 30-day FinCEN SLA verified end-to-end.
The code review flagged the `candidateGroups` assertion as wrong — Claude had
traced through bytecode and concluded the mapper converts the field to a
`List<String>`. But `WorkItemWithAuditResponse.candidateGroups` is a plain
`String`, and the `@QuarkusTest` had already proven the assertion correct against
the live endpoint. We kept it as written.

The CI workflow was missing `repository_dispatch: types: [upstream-published]`.
One line.

The LAYER-LOG still described `WorkItemCreateRequest` as a 19-field positional
record with "no builder yet." That was two layers ago.
