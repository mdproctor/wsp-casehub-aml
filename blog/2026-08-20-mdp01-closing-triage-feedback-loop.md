---
layout: post
title: "Closing the triage feedback loop"
date: 2026-08-20
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [trust-routing, attestation, case-outcome, feedback-loop]
---

When a case reaches `investigation-closed-no-sar`, the triage evaluator originally said "file a SAR" but human authority overruled. Until now, the trust routing system had no way to learn from that. The triage agent's score stayed the same regardless of whether its SAR_WARRANTED decisions were vindicated or reversed.

We extended `AmlTrustRoutingObserver` — the existing observer that captures trust scores at routing time — to also observe case outcomes. When a case closes with the `investigation-closed-no-sar` goal, the observer writes a `LedgerAttestation` with `AttestationVerdict.CHALLENGED` against the triage worker's `WorkerDecisionEntry`. The Bayesian Beta trust model picks this up on the next scoring run and nudges the triage agent's `investigation-accuracy` score downward.

The interesting design question was what to write. `AmlTrustRoutingAttestation` — the custom entity we use for routing-time captures — records trust score, threshold, and selected worker at dispatch time. But triage overruled is outcome feedback, not a routing snapshot. It belongs in `LedgerAttestation`, the same mechanism `SarOutcomeFeedbackService` uses for SAR verdicts. `CHALLENGED` (not `FLAGGED`) because the triage decision wasn't deficient — it was a judgment call that a higher authority reversed. Different semantic, separable in analytics.

Two paths lead to `investigation-closed-no-sar`. In one, the post-rejection re-triage drops below the SAR threshold — the senior analyst's new evidence changed the risk picture. In the other, re-triage still says SAR_WARRANTED but the head of compliance overrules. The evidence field distinguishes them (`RE_TRIAGE_DROP` vs `ESCALATION_NO_SAR`), though the attestation verdict is the same in both cases. The `dimensionScore` is fixed at 0.2 — the `CHALLENGED` verdict is the primary signal driving the Bayesian update, not the score magnitude.

There's an asymmetry worth noting. This adds negative feedback for wrong triage but no positive feedback when triage is vindicated by a successful SAR filing. Over time, that systematically degrades triage workers' trust scores even when they're mostly right. A follow-up should add an `ENDORSED` attestation when a SAR is filed and upheld — closing the feedback loop in both directions.
