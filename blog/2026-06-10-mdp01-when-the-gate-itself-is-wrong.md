---
layout: post
title: "When the gate itself is wrong"
date: 2026-06-10
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [oversight-gate, action-risk-classifier, fail-closed, domain-model, snapshot]
---

Layer 9 adds the oversight gate — the mechanism that intercepts a worker's `PlannedAction` and decides whether to proceed autonomously or hold for human review. Five action types in AML, three policies: always gate, gate if risk score exceeds a threshold, gate if confidence falls below one.

The design question is where policy lives. The obvious answer is the classifier itself — a switch on action type, each branch evaluating the conditions and building the gate. That works for the normal path. It breaks on the fail-closed path.

Every classifier needs a fail-closed path. When context is missing — no `riskScore`, no `entityType`, null input — the classifier can't evaluate the normal conditions. The safe response is to gate anyway. And that's where the hardcoding creeps in: `reversible=true`, `candidateGroups=aml-compliance`, because those seem like reasonable defaults for an unknown case.

Claude flagged it in the code review. `SAR_FILING` is `reversible=false` — a filed Suspicious Activity Report cannot be recalled. A fail-closed gate claiming `reversible=true` puts incorrect information in the audit trail. The compliance officer sees a gate that implies the action is undoable when it isn't. The bug passes tests because the tests check that a gate fires, not that the gate's properties match the action type's policy.

The fix: `AmlActionType` owns all gate metadata — `reversible()`, `candidateGroups()`, `scope()`, `expiresIn()`, `gatePolicy()`. The fail-closed path reads from the type exactly like the normal path does. `type.reversible()`, not `true`. There is no separate safe-default logic in the classifier. The domain enum already knows what these actions are.

The PEP carve-out is the most interesting detail in the risk score path. Score ≥ 0.8 triggers a gate. But a PEP entity with score 0.2 also triggers a gate — the entity type overrides the numeric threshold. FATF guidance treats PEP exposure as categorically different from risk-score analysis. That distinction is now explicit in the domain model; the classifier just evaluates it.

---

The SNAPSHOT updates brought their own problems.

`casehub-ledger` made `tenancy_id` NOT NULL. Our trust attestation writes were passing `event.tenancyId()` without a null guard — silent until the database rejected the INSERT.

Harder to diagnose: `casehub-work` added `TenantScopedPrincipal @RequestScoped`. Because it's not `@DefaultBean`, it displaced `MockCurrentPrincipal @DefaultBean` in `@QuarkusTest`. On the HTTP test thread, a request context exists and the bean works. On Vert.x event loop threads — where the case engine runs workers — no request context exists. The bean returns null tenancyId silently, propagating into every entity write. That one is still open as aml#59.

The gate test itself ran into a concurrent INSERT race on `ledger_subject_sequence`. Multiple async `WorkerDecisionEvent` observers fire simultaneously for the same case, all trying to MERGE into the sequence table — the first succeeds, the rest fail. The `WorkerDecisionEntry` saves roll back, and completion detection breaks with them.

The fix was to read from `CaseInstanceCache` instead. The engine's in-memory state reflects completion the moment the case finishes, without any ledger dependency. The gate test now checks `CaseStatus.COMPLETED` from the cache and is resilient to the race.
