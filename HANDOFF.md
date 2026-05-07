# Handoff — casehub-aml Bootstrap
2026-05-07

## What this project is

`casehub-aml` is the Anti-Money Laundering investigation application — the primary community tutorial for CaseHub targeting Java/Quarkus developers. It demonstrates all CaseHub capabilities in the domain these developers know best: banking and financial crime compliance.

The strategic rationale is in the use-case analysis: AML scored 44/50 (22 market + 22 community) — the only use case strong on both dimensions simultaneously. Java dominates banking infrastructure; enterprise developers have built these systems and know what fails in practice. The compliance gap current agentic AML systems cannot close (FinCEN auditable evidence chains, GDPR erasure, formal obligation tracking per investigation) is what CaseHub provides structurally.

Full analysis: `https://raw.githubusercontent.com/casehubio/parent/main/docs/use-case-analysis.md` §8.2
Tutorial strategy: `https://raw.githubusercontent.com/casehubio/parent/main/docs/tutorial-strategy.md` §6

## Current state

Greenfield repo — no code yet. CLAUDE.md written. GitHub repo at `casehubio/aml`. Application built entirely on the CaseHub foundation; no foundation changes needed.

## Foundation status relevant to AML

| Foundation capability | Status |
|----------------------|--------|
| CasePlanModel, bindings, blackboard | ✅ Production |
| qhorus COMMAND/RESPONSE/DONE/FAILURE/DECLINE | ✅ Production |
| Commitment lifecycle (7 states) | ✅ Production |
| Commitment outcomes → trust scoring | ✅ DONE (qhorus#123) |
| WorkItem lifecycle + SLA (30-day FinCEN deadline) | ✅ Production |
| EscalationPolicy (head of compliance on SLA miss) | ✅ Production |
| LedgerErasureService (GDPR Art.17 — PII on transactions) | ✅ Production |
| CaseLedgerEntry (Merkle audit per case event) | ✅ DONE (2026-04-26) |
| GDPR Art.22 ComplianceSupplement | ✅ Production |
| W3C PROV-DM export (FinCEN evidence chain) | ✅ Production |
| engine COMMAND dispatch after worker scheduling | ✅ DONE (engine#186) |
| Parallel binding evaluation (simultaneous specialist checks) | ✅ Production |
| TrustWeightedSelectionStrategy wired in engine | ⚠️ Pending (P1.3) |
| LlmPlanningStrategy (investigation triage supervisor) | ⚠️ Pending (engine#102) |

## What to build first

### Layer 1 — Tutorial baseline (no CaseHub)

Write the naive Java approach first — the anti-pattern the tutorial teaches against:

```java
EntityResolutionResult entity = entityService.resolve(transaction);
PatternResult pattern = patternService.analyzeStructuring(transaction);
OsintResult osint = osintService.checkSanctions(transaction);
// Who was accountable? What if one failed? Can we prove this to FinCEN?
```

This is Layer 1 from tutorial-strategy.md §6.1. Writing it first makes every CaseHub addition feel like an earned solution to a visible problem.

### Layer 2 — casehub-work: compliance officer WorkItem with SLA

```java
WorkItemRequest sarReview = WorkItemRequest.builder()
    .title("SAR Review: TXN-2024-001")
    .candidateGroups("compliance-officers")
    .claimDeadline(Instant.now().plus(30, ChronoUnit.DAYS)) // FinCEN SLA
    .build();
```

### Layer 3 — casehub-qhorus: typed specialist communication

Key teaching moment: OSINT agent DECLINE because it lacks PEP clearance. A formal record, not an error. "DECLINED means the agent knew its limits and said so formally — different from FAILED."

### Later layers (after foundation P0 fully wired)

- Layer 4: casehub-ledger (Merkle chain, GDPR Art.17)
- Layer 5: casehub-engine (adaptive path — PEP routing, parallel checks)
- Layer 6: trust routing (experienced agents on complex cases, auto from SAR outcomes)

## Investigation flow

```
Transaction flagged (EVENT)
    ↓
[Entity Resolution]  COMMAND → resolve beneficial ownership
    ↓ RESPONSE: entity graph
[Pattern Analysis]   COMMAND → assess layering/structuring     ─┐ simultaneous
[OSINT Screening]    COMMAND → sanctions, PEP, adverse media   ─┘ (parallel bindings)
    ↓
[OSINT Agent]  DECLINE → "Insufficient clearance for PEP database"
    → re-route to senior analyst immediately (agent healthy, wrong capability)
    ↓
[SAR Drafting]  COMMAND → synthesise findings
    ↓ DONE: narrative complete
[Compliance Officer] WorkItem: 30-day claimDeadline (FinCEN SLA)
    → DONE: SAR filed  OR  DECLINE: case cleared
[Ledger] attestation written; trust scores updated
```

## References

- Use case analysis (§8.2): `https://raw.githubusercontent.com/casehubio/parent/main/docs/use-case-analysis.md`
- Tutorial strategy (AML §6): `https://raw.githubusercontent.com/casehubio/parent/main/docs/tutorial-strategy.md`
- IBM AMLSim baseline: https://github.com/IBM/AMLSim/
- GitHub epics: casehubio/aml issues labelled `epic`
