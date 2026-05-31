# Handoff — Layer 7 shipped, branch closed
2026-05-31

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-05-31)

Layer 7 compliance evidence complete and shipped to `casehubio/aml` main. The
endpoint `GET /api/investigations/{caseId}/compliance-evidence` returns
requirement-scoped evidence with Merkle inclusion proofs for five FinCEN/FATF
obligations. Branch `issue-43-layer7-comparison` closed, issue #43 closed.

Key decisions: claims vs evidence (boolean `verify()` is self-attestation; inclusion
proofs are independently verifiable). SLA PARTIAL for open WorkItem (CLOSED means
demonstrably met, not just "not breached"). `@ObservesAsync` required for engine
`WorkerDecisionEvent` (engine fires async). `@Path("/")` is too greedy in RESTEasy
Reactive — use fully-specified class-level paths. `LedgerEntryRepository.save()`
triggers the Merkle enricher; non-Merkle entities must persist via `EntityManager`
directly.

Three upstream issues filed: casehubio/engine#403 (`trustScoreAtRouting` on
`WorkerDecisionEntry`), casehubio/work#241 (public WorkItem read API),
casehubio/aml#44 (observer failure reconciliation).

## Immediate next step

Pickup `#42` (ActionRiskClassifier oversight gate) or `#32` (CaseMemoryStore).
`#42` depends on casehubio/engine#402 (SPI exists) — check engine#402 status first.
Run `/work` to start.

## What's left

- Deferred GDPR demo: `AML_SAR_OFFICER_REVIEWED` ledger event with human `actorId` so erasure test demonstrates the right data subject — casehubio/aml#44 · S · Low
- Pre-existing `quarkus:build` CDI failure (26 unsatisfied engine SPI deps) — not a Layer 7 regression; needs fix in app's production deps or scope adjustment · S · Med

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #42 | ActionRiskClassifier oversight gate for consequential AML actions | M | Med | Depends on casehubio/engine#402 (SPI exists) — check status before starting |
| #32 | CaseMemoryStore — surface entity history before/during investigation | M | Med | Blocked on casehubio/platform#27 |
| #44 | Observer failure reconciliation — detect missing trust attestations at case close | M | Med | Silent evidence gaps; see spec details in issue |
| #14 | Layer 8 — LLM supervisor mode (investigation triage) | L | High | Blocked on casehubio/engine#101 (labeled `future`) |

## References

- Blog: `blog/2026-05-31-mdp01-proof-not-claims.md`
- LAYER-LOG Layer 7: `LAYER-LOG.md §Layer 7`
- Garden entries: GE-20260531-dd44a2 (@Path root greedy), GE-20260531-864d8e (@ObservesAsync), GE-20260531-d2ed26 (LedgerRepo Merkle side-effect), GE-20260531-1587fe (MerkleFrontier alternative), GE-20260531-46f8ab (tokenisation flag)
