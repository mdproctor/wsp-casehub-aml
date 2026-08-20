# Trust Attestation for TRIAGE_OVERRULED — Design Spec

**Issue:** casehubio/aml#128
**Date:** 2026-08-19
**Status:** Draft

## Summary

When a case reaches `investigation-closed-no-sar`, the triage evaluator originally said SAR_WARRANTED but human authority overruled. Extend `AmlTrustRoutingObserver` to observe `CaseOutcomeEvent` and write a `LedgerAttestation` with `AttestationVerdict.CHALLENGED` against the triage worker's `WorkerDecisionEntry` on the `investigation-accuracy` trust dimension.

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Attestation entity (D1) | `LedgerAttestation` with `CHALLENGED` verdict | Outcome feedback against an existing decision — exactly what LedgerAttestation is for. No migration, follows SarOutcomeFeedbackService pattern |
| Target scope (D2) | Original triage worker only | post-rejection-triage is single-worker — attestation has no routing effect. Follow-up when capability gets multiple workers |
| dimensionScore (D3) | Fixed 0.2 | CHALLENGED verdict is the primary Bayesian Beta signal; score is secondary calibration. Refineable later |

## Architecture

### Two Paths to investigation-closed-no-sar

**Path A — re-triage score drops:**
```
triage → SAR_WARRANTED → SAR drafted → MLRO rejects gate
  → senior analyst finds new evidence → re-triage → FALSE_POSITIVE
  → investigation-closed-no-sar
```

**Path B — head of compliance overrules:**
```
triage → SAR_WARRANTED → SAR drafted → MLRO rejects gate
  → senior analyst → re-triage → SAR_WARRANTED (confirmed)
  → head of compliance → NO_SAR
  → investigation-closed-no-sar
```

Both paths produce a CHALLENGED attestation against the original `investigation-triage-agent`. The evidence text distinguishes the overrule source.

### Observer Mechanism

`AmlTrustRoutingObserver` adds `implements CaseOutcomeObserver`. The engine discovers `CaseOutcomeObserver` beans via CDI and calls `onOutcome()` at case completion — the same mechanism `AmlCaseProfileStoreObserver` uses.

The class now handles two independent observation paths:
- `@ObservesAsync WorkerDecisionEvent` → writes `AmlTrustRoutingAttestation` (routing-time capture, existing)
- `CaseOutcomeObserver.onOutcome(CaseOutcomeEvent)` → writes `LedgerAttestation` (outcome feedback, new)

### Attestation Fields

| Field | Value | Rationale |
|-------|-------|-----------|
| `ledgerEntryId` | `WorkerDecisionEntry.id` for the `investigation-triage` capability | Links the attestation to the specific worker decision being challenged |
| `subjectId` | `event.caseId()` | Same subject as SarOutcomeFeedbackService — the investigation case |
| `attestorId` | `"aml-orchestrator"` | Consistent with AmlTrustRoutingObserver's existing identity |
| `attestorType` | `ActorType.SYSTEM` | Automated observation |
| `attestorRole` | `"TriageOutcomeFeedback"` | Distinguishes from routing-time attestations |
| `verdict` | `AttestationVerdict.CHALLENGED` | Decision was challenged and reversed by human authority. Distinct from FLAGGED (deficient output) |
| `capabilityTag` | `"investigation-triage"` | The capability whose decision was overruled |
| `trustDimension` | `"investigation-accuracy"` | Existing dimension — no new dimension needed |
| `dimensionScore` | `0.2` | Fixed low score — "poor investigation accuracy on this decision" |
| `confidence` | `1.0` | The overrule is a fact, not a probabilistic judgment |
| `evidence` | Plain text with overrule details | See Evidence Format below |
| `occurredAt` | `event.closedAt()` | When the case closed, not when the attestation was written |

### Evidence Format

```
TRIAGE_OVERRULED: originalDecision=SAR_WARRANTED, overruleSource=RE_TRIAGE_DROP|ESCALATION_NO_SAR
```

Determined from `caseFileSnapshot`:
- If `rejectionEscalation` present with `decision == "NO_SAR"` → `ESCALATION_NO_SAR`
- If `postRejectionTriage` present with `decision == "FALSE_POSITIVE"` and gate type is `sar.filing` → `RE_TRIAGE_DROP`

### Implementation Detail

```java
@Override
public void onOutcome(CaseOutcomeEvent event) {
    if (!"aml-investigation".equals(event.caseType())) return;
    if (!"investigation-closed-no-sar".equals(event.outcomeLabel())) return;

    try {
        writeTriageOverruledAttestation(event);
    } catch (Exception e) {
        LOG.warnf(e, "Triage overruled attestation failed caseId=%s", event.caseId());
    }
}
```

The method:
1. Queries `AmlWorkerDecisionRepository.findLatestByCaseIdAndCapability(caseId, "investigation-triage")`
2. If not found → log warning, return (no crash — defensive)
3. Determines overrule source from `caseFileSnapshot`
4. Persists `LedgerAttestation` via the qhorus `EntityManager`

The method runs in the engine's case completion context. Wrapping in try-catch ensures a failed attestation does not break case completion — same defensive pattern as `AmlCaseProfileStoreObserver`.

### Transaction Boundary

`CaseOutcomeObserver.onOutcome()` is called by the engine during case completion. The triage worker's `WorkerDecisionEntry` was committed long before the case reached `investigation-closed-no-sar` (triage runs early; rejection routing, re-triage, and escalation follow). No timing concern.

The attestation write uses `QuarkusTransaction.requiringNew()` to isolate it from the engine's completion transaction — a failure must not roll back case completion.

## Asymmetry Note

This adds negative feedback (CHALLENGED when triage is overruled) but no positive feedback (when triage is vindicated by a successful SAR). Over time, this systematically degrades triage workers' trust scores. A follow-up issue should add a SOUND attestation for correct triage outcomes (e.g., SAR filed and UPHELD → ENDORSED attestation on the investigation-triage worker).

## Testing Strategy

### Unit Tests (api/) — not applicable

No new domain types. The observer is CDI-wired; unit testing adds no value here.

### @QuarkusTest Integration Tests (app/)

All tests follow established conventions: drain to completion, Awaitility polling, gate approval before attestation wait (GE-20260628-dbc656), ledger subject isolation, CBR store isolation (GE-20260716-986cd1).

**`TriageOverruledAttestationTest`:**

| # | Scenario | Expected |
|---|----------|----------|
| 1 | SAR_FILING rejected → re-triage score drops → `investigation-closed-no-sar` | CHALLENGED attestation written against investigation-triage worker, evidence contains `RE_TRIAGE_DROP` |
| 2 | SAR_FILING rejected → escalation NO_SAR → `investigation-closed-no-sar` | CHALLENGED attestation written against investigation-triage worker, evidence contains `ESCALATION_NO_SAR` |
| 3 | Normal SAR path → `investigation-complete` | No CHALLENGED attestation for investigation-triage |
| 4 | Triage FALSE_POSITIVE → `investigation-cleared` | No CHALLENGED attestation |

Each test:
1. Starts investigation with appropriate flag reason (`HIGH_RISK_JURISDICTION` for SAR path per GE-20260726-00e4df)
2. Drains to the relevant gate or terminal state
3. For rejection scenarios: rejects the SAR_FILING gate, awaits rejection routing
4. For escalation scenarios: resolves the head-of-compliance WorkItem with NO_SAR
5. Drains to terminal state
6. Queries `LedgerAttestation` by the triage worker's `WorkerDecisionEntry.id`
7. Asserts verdict, trustDimension, capabilityTag, dimensionScore, evidence content

**Note:** Scenarios 1 and 2 overlap with `SarFilingRejectionRoutingTest` from #72. The attestation assertions can be added to those existing tests OR kept in a separate test class for single-concern focus. Separate class recommended — the attestation concern is independent of the routing logic.

## File Inventory

### Modified Files

| File | Change |
|------|--------|
| `AmlTrustRoutingObserver.java` | Add `implements CaseOutcomeObserver`, add `@Inject AmlWorkerDecisionRepository` and `@PersistenceContext(unitName = "qhorus") EntityManager`, add `onOutcome()` and `writeTriageOverruledAttestation()` methods |

### New Files

| File | Module | Description |
|------|--------|-------------|
| `TriageOverruledAttestationTest.java` | app (test) | @QuarkusTest — 4 scenarios verifying CHALLENGED attestation |

### Dependencies

No new dependencies. `CaseOutcomeObserver` and `CaseOutcomeEvent` are in `casehub-engine-api` (already on classpath). `LedgerAttestation` is in `casehub-ledger` (already on classpath).

## References

- `gate-rejection-routing-design.md` §Trust Routing Integration — the spec that defines this work
- `gate-rejection-routing-design.md` §Rejection Flow — SAR_FILING — the two paths to `investigation-closed-no-sar`
- `AmlTrustRoutingObserver.java:51` — existing `onWorkerDecision` observer method
- `SarOutcomeFeedbackService.java:70-88` — `writeAttestation()` pattern for LedgerAttestation
- `AmlCaseProfileStoreObserver.java:72-78` — `CaseOutcomeObserver` implementation pattern with defensive try-catch
- `AmlWorkerDecisionRepository.java:28-39` — `findLatestByCaseIdAndCapability()` query
- `AttestationVerdict.class` — SOUND, FLAGGED, ENDORSED, CHALLENGED enum values
- `CaseOutcomeEvent.class` — `outcomeLabel`, `caseFileSnapshot`, `closedAt` fields
- `layer6-trust-routing-design.md` — Layer 6 trust routing foundation, Bayesian Beta model
- GE-20260628-dbc656 — gate approval before attestation wait ordering
- GE-20260726-00e4df — HIGH_RISK_JURISDICTION flag reason for SAR path tests
- GE-20260716-986cd1 — CBR store isolation in tests
