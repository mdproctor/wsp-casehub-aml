---
layout: post
title: "What the officer decided"
date: 2026-06-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [investigation-outcome, compliance, adversarial-review, ledger]
---

Three issues closed on one branch — #69 (worker imports, already done), #68 (CI rename), and #71 (gate rejection outcome). The first two were housekeeping. The third is where the design work happened.

The question behind #71 was straightforward: when the compliance officer rejects a SAR filing, how does the investigation API surface that fact? The existing response returned `status: "completed"` with no indication of what the officer actually decided.

I wanted lifecycle status and domain outcome separated. `status` answers "is it done?" — that's an engine concern. `outcome` answers "what happened?" — that's domain. A gate rejection is a domain event that triggers a lifecycle transition, not a lifecycle state. Making `status: "rejected"` would conflate the two, and it breaks entirely for reversible actions where a gate rejection doesn't terminate the case.

The outcome derives from `AmlSarOfficerReviewedLedgerEntry.reviewDecision` — three values map to three types: `sar-filed`, `gate-rejected`, `decision-not-recorded`. No new ledger entry types, no new columns. The data was already there; we just needed to read it.

The adversarial review caught five things I'd missed. The most consequential: `InvestigationOutcome.fromReviewDecision` had a `default -> null` arm for unexpected values. In practice, a corrupt `reviewDecision` would silently return null — indistinguishable from the eventual-consistency window where the async observer hasn't fired yet. A consumer following the retry guidance would poll for five seconds, conclude "permanent recording failure," and move on. The actual problem — data corruption requiring investigation — would be invisible. Claude's reviewer caught this and pushed for `IllegalStateException`. That's the right call. Fail-fast on data corruption; don't make it look like a timing issue.

The second catch was subtler. The PP-20260530-49856c double-try/catch failure path discards the officer's actual decision. The observer knows whether the officer approved or rejected — the `reviewDecision` variable is in scope when the primary write fails. But `writeSarOfficerReviewedFailure` was writing `"UNKNOWN"` unconditionally. Information available at failure time, thrown away. The fix was a one-line signature change: accept `reviewDecision` as a parameter and write the real value. `decision-not-recorded` is retained for backwards compatibility with existing `"UNKNOWN"` entries, but new failure records now preserve the actual decision.

The entry selection rule matters too. The failure path can produce a second `AmlSarOfficerReviewedLedgerEntry` alongside the primary one — one HUMAN-attributed (the officer's decision), one SYSTEM-attributed (the failure marker). The outcome service prefers HUMAN over SYSTEM, with `sequenceNumber` as a tiebreaker for deterministic ordering.

One architectural surprise: the plan assumed SAR filing ran through the Layer 9 endpoint. It doesn't. Layer 9 handles oversight gate investigations — entity link creation decisions. The SAR filing flow goes through Layer 6. We wired outcome into both APIs, but the tests had to target Layer 6 to exercise the compliance review WorkItem lifecycle. Layer 9 oversight investigations correctly return null outcome — no SAR was filed, so there's nothing to report.

The outcome service landed in the `compliance` package, not `engine`. It reads ledger entries and derives a compliance-adjacent domain outcome. Zero engine-type dependencies. The reviewer pointed out the package mismatch — `AmlComplianceEvidenceService` follows the same pattern and lives in `compliance`.

Three follow-up issues filed: #74 (consolidate the duplicated completion detection across Layer 6 and Layer 9 into the outcome service), #75 (capture the officer's rejection reason — `WorkItemLifecycleEvent` has `detail()` and `rationale()` accessors that the observer currently ignores), and #77 (test gaps for the committed review corrections). The completion detection consolidation also fixes a pre-existing bug: nonexistent caseIds return `{status: "in-progress"}` instead of 404.
