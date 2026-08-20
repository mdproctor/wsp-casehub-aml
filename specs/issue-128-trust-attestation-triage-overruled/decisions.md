# Decisions — Trust Attestation for TRIAGE_OVERRULED (#128)

## D1: Attestation entity type

**Choice:** `LedgerAttestation` against the triage worker's `WorkerDecisionEntry`, using `AttestationVerdict.CHALLENGED`
**Alternatives:**
- New `AmlTriageOverruledEntry extends JpaLedgerEntry` with typed fields — overkill for two string values, requires Flyway migration
- Extend `AmlTrustRoutingAttestation` with verdict fields — mixes routing-time capture with outcome feedback
**Rationale:** TRIAGE_OVERRULED is outcome feedback against an existing decision entry — exactly what `LedgerAttestation` is for. `CHALLENGED` maps well: the triage decision was challenged and reversed by human authority. Follows the `SarOutcomeFeedbackService` pattern already proven in this codebase. `overruledDecision` and `overruledBy` are carried in the `evidence` field.
**Trade-offs:** No typed fields for `overruledDecision`/`overruledBy` — they live in the `evidence` string. Acceptable for a single attestation type with two metadata values.
**Sources:** SarOutcomeFeedbackService.java, AmlTrustRoutingAttestation.java, LedgerAttestation.class, gate-rejection-routing-design.md §Trust Routing Integration
**Exploration:** quick
**Status:** captured

## D2: Target worker scope

**Choice:** Original triage worker (`investigation-triage-agent`) only — post-rejection-triage attestation out of scope
**Alternatives:**
- Both triage workers in path B (escalation NO_SAR) — post-rejection-triage also said SAR_WARRANTED and was overruled. Would provide more complete calibration signal.
**Rationale:** `post-rejection-triage` is a single-worker capability — trust scoring has no routing effect when there's no alternative. The attestation would be recorded with zero operational impact. Can be added when the capability gets multiple workers.
**Trade-offs:** Incomplete feedback — post-rejection-triage doesn't learn from its wrong decisions. Acceptable for showcase; noted for follow-up.
**Sources:** aml-investigation.yaml (post-rejection-triage capability, single worker), gate-rejection-routing-design.md §Trust Routing Integration
**Exploration:** deep-analysis
**Depends on:** D1 (attestation type determines what we write against the target)
**Status:** captured

## D3: dimensionScore value

**Choice:** Fixed 0.2 for all TRIAGE_OVERRULED attestations
**Alternatives:**
- Path-dependent value (0.3 for re-triage drop, 0.15 for escalation NO_SAR) — more nuanced but adds complexity for a signal that's secondary to the verdict
- Derived from original risk score distance above threshold — elegant but requires reading and computing from the case file snapshot
**Rationale:** The `CHALLENGED` verdict is the primary signal driving the Bayesian Beta update. `dimensionScore` is secondary calibration. A fixed 0.2 ("poor investigation accuracy on this decision") is reasonable for the showcase. Can be refined if the trust scoring model is extended to weight by dimensionScore.
**Trade-offs:** Path A (re-triage drop, weaker signal) and path B (escalation NO_SAR, stronger signal) get identical scores. The distinction is captured in the evidence text but not in the numeric signal.
**Sources:** SarOutcomeFeedbackService.java (dimensionScore from caller), TrustScoreJob (Bayesian Beta model)
**Exploration:** deep-analysis
**Depends on:** D1 (LedgerAttestation has a dimensionScore field)
**Status:** captured
