# Layer 3: casehub-qhorus — Typed Specialist Agent Communication

**Date:** 2026-05-17
**Issue:** casehubio/aml#19
**Epic:** casehubio/aml#9 (Tutorial layers 1–7)
**Builds on:** Layer 2 (casehubio/aml#15, closed)

---

## Context

Layer 1 established the naive baseline; Layer 2 added the compliance officer WorkItem
with 30-day FinCEN SLA. During Layer 3 design, a flaw in Layer 2's CDI wiring was
discovered: `WorkItemAmlInvestigationService` injected `NaiveAmlInvestigationService`
by concrete type, breaking the Layer 2 commit's stated intent that "changing specialist
implementations in Layer 3 will propagate automatically."

Layer 3 corrects this and adds casehub-qhorus: typed COMMAND/RESPONSE/DONE/DECLINE
per specialist agent. DECLINE from the OSINT agent (insufficient clearance) is a formal
scope boundary — not an error.

---

## Architecture

### Composer pattern

Layer 3 replaces the Layer 2 service with a stable coordinator that composes two
independently swappable concerns:

```
AmlInvestigationCoordinator          implements AmlInvestigationApplicationService
  ├── AmlInvestigator                investigation concern (swapped by CDI per layer)
  │     NaiveAmlInvestigator         Layer 1, @DefaultBean
  │     QhorusAmlInvestigator        Layer 3, displaces Naive
  └── ComplianceReviewLifecycle      WorkItem concern (stable from Layer 2 onwards)

AgentBehaviour                       what a stub does when it receives a COMMAND
AgentDispatchMechanism               how a stub receives COMMANDs (push vs polling)
  PushAgentDispatch                  in-process: registers as AgentChannelBackend
  PollingAgentDispatch               skeleton — for out-of-process agents (future)
```

`WorkItemAmlInvestigationService` is deleted — its two concerns are split between
coordinator (compose) and `ComplianceReviewLifecycle` (WorkItem).

CDI chain: `AmlInvestigationCoordinator` injects `AmlInvestigator` → resolves to
`QhorusAmlInvestigator` (no `@DefaultBean`) over `NaiveAmlInvestigator` (`@DefaultBean`).

---

## Domain Model Changes (`api/`)

### New: `SpecialistOutcome<T>`

```java
public sealed interface SpecialistOutcome<T>
    permits SpecialistOutcome.Completed, SpecialistOutcome.Declined, SpecialistOutcome.Failed {

    record Completed<T>(T result)                                         implements SpecialistOutcome<T> {}
    record Declined<T>(String agentId, String capability, String reason)  implements SpecialistOutcome<T> {}
    record Failed<T>  (String agentId, String capability, String reason)  implements SpecialistOutcome<T> {}
}
```

Three variants map to qhorus message types: DONE/RESPONSE → Completed,
DECLINE → Declined, FAILURE → Failed. Anticipates Epic 5 (DECLINED vs FAILED routing).

### Refactored: `InvestigationSummary`

```java
public record InvestigationSummary(
    SuspiciousTransaction transaction,
    SpecialistOutcome<EntityResolutionResult> entityResolution,
    SpecialistOutcome<PatternAnalysisResult>  patternAnalysis,
    SpecialistOutcome<OsintResult>            osintScreening,
    String sarNarrative) {}
```

### Refactored: `NaiveSarDraftingService`

Accepts `SpecialistOutcome<OsintResult>`, pattern-matches all three variants:

```java
String osintSummary = switch (osint) {
    case SpecialistOutcome.Completed<OsintResult> c -> formatOsint(c.result());
    case SpecialistOutcome.Declined<OsintResult> d  -> "OSINT declined (" + d.capability() + "): " + d.reason();
    case SpecialistOutcome.Failed<OsintResult> f    -> "OSINT failed: " + f.reason() + " — manual review required";
};
```

At Layer 1 (naive), all three outcomes are `Completed`. `Declined` and `Failed` only
appear via the qhorus path introduced in Layer 3.

---

## Coordinator and Inner Interfaces (`app/`)

### New: `AmlInvestigator`

```java
public interface AmlInvestigator {
    InvestigationSummary investigate(SuspiciousTransaction transaction);
}
```

### Renamed: `NaiveAmlInvestigator` (was `NaiveAmlInvestigationService`)

Same logic; implements `AmlInvestigator`; wraps each result in
`SpecialistOutcome.Completed`; retains `@DefaultBean`.

### New: `ComplianceReviewLifecycle`

Encapsulates all AML domain knowledge about the compliance review obligation:
30-day FinCEN SLA, compliance-officers group, HIGH priority, callerRef format.

```java
@ApplicationScoped
public class ComplianceReviewLifecycle {
    @Inject WorkItemService workItemService;

    public String openReview(SuspiciousTransaction transaction, InvestigationSummary summary) {
        WorkItem workItem = workItemService.create(new WorkItemCreateRequest(
            "Compliance review — SAR for transaction " + transaction.id(),
            null, "aml-compliance", null, WorkItemPriority.HIGH, null,
            "compliance-officers", null, null, "aml-system", null,
            Instant.now().plus(30, ChronoUnit.DAYS), null, null, null, null,
            "aml:investigation/" + transaction.id(), null, null));
        return workItem.id.toString();
    }
}
```

### New: `AmlInvestigationCoordinator`

Stable outer coordinator — unchanged for Layers 3 and 4. Layer 5 engine replaces it.

```java
@ApplicationScoped
public class AmlInvestigationCoordinator implements AmlInvestigationApplicationService {
    @Inject AmlInvestigator investigator;
    @Inject ComplianceReviewLifecycle compliance;

    @Override
    public AmlInvestigationResult investigate(SuspiciousTransaction transaction) {
        InvestigationSummary summary = investigator.investigate(transaction);
        String taskId = compliance.openReview(transaction, summary);
        return new AmlInvestigationResult(summary, taskId);
    }
}
```

### Deleted: `WorkItemAmlInvestigationService`

---

## Qhorus Wiring (`app/`)

### `QhorusAmlInvestigator`

`@ApplicationScoped`, no `@DefaultBean`. Displaces `NaiveAmlInvestigator` via CDI.

```java
@Override
public InvestigationSummary investigate(SuspiciousTransaction transaction) {
    String caseRef = "aml:investigation/" + transaction.id();
    SpecialistOutcome<EntityResolutionResult> entity  = dispatch(ENTITY_RESOLUTION, transaction, caseRef);
    SpecialistOutcome<PatternAnalysisResult>  pattern = dispatch(PATTERN_ANALYSIS,  transaction, caseRef);
    SpecialistOutcome<OsintResult>            osint   = dispatch(OSINT_SCREENING,   transaction, caseRef);
    String narrative = sarDraftingService.draft(transaction, entity, pattern, osint);
    return new InvestigationSummary(transaction, entity, pattern, osint, narrative);
}
```

`dispatch()` sends a COMMAND and maps the reply:

```java
private <T> SpecialistOutcome<T> dispatch(String capability, SuspiciousTransaction tx, String ref) {
    String correlationId = messageService.sendCommand(capability, tx.toJson(), ref);
    Message reply = messageService.waitForReply(correlationId, Duration.ofSeconds(5));
    return switch (reply.type()) {
        case DONE, RESPONSE -> new SpecialistOutcome.Completed<>(deserialize(reply, capability));
        case DECLINE         -> new SpecialistOutcome.Declined<>(reply.sender(), capability, reply.content());
        case FAILURE         -> new SpecialistOutcome.Failed<>(reply.sender(), capability, reply.content());
        default              -> new SpecialistOutcome.Failed<>(reply.sender(), capability, "unexpected: " + reply.type());
    };
}
```

---

## Agent Dispatch Strategy (`app/agents/`)

### SPIs

```java
public interface AgentBehaviour {
    String capability();
    SpecialistOutcome<?> handle(Message command);
}

public interface AgentDispatchMechanism {
    void start(AgentBehaviour behaviour);
    void stop();
}
```

### `PushAgentDispatch`

`@Dependent` — one instance per agent. Implements `AgentDispatchMechanism` and
qhorus `AgentChannelBackend`. Follows gateway-backend-registration-ordering protocol:
`open()` before `registerBackend()`.

Fan-out from `channelGateway.fanOut()` calls `receive()` immediately on COMMAND send —
no scheduler, no polling interval, resolves `wait_for_reply` within the HTTP request.

### `PollingAgentDispatch`

Skeleton only — documented extension point for out-of-process agents (claudony workers,
later layers). Not wired in Layer 3.

### Stub agent behaviours

Three `@ApplicationScoped @DefaultBean` beans. Each injects `PushAgentDispatch`
(`@Dependent`) and calls `dispatch.start(this)` in `@PostConstruct`.

| Bean | Capability | Response |
|------|-----------|---------|
| `EntityResolutionBehaviour` | `entity-resolution` | `Completed` — naive resolution result |
| `PatternAnalysisBehaviour` | `pattern-analysis` | `Completed` — naive pattern result |
| `OsintScreeningBehaviour` | `osint-screening` | `Declined` — "insufficient clearance for PEP database access" |

`@DefaultBean` means real AI agent behaviours (claudony workers, Layer 5+) displace
the stubs without touching the dispatch infrastructure.

---

## REST Contract

**Request:** unchanged — `POST /api/investigations` with `SuspiciousTransaction` body.

**Response (Layer 3):**

```json
{
  "summary": {
    "transaction": { "id": "TXN-001", ... },
    "entityResolution": { "type": "Completed", "result": { ... } },
    "patternAnalysis":  { "type": "Completed", "result": { ... } },
    "osintScreening":   { "type": "Declined",  "agentId": "...", "capability": "osint-screening",
                          "reason": "insufficient clearance for PEP database access" },
    "sarNarrative": "... OSINT screening declined: insufficient clearance ..."
  },
  "complianceReviewTaskId": "a1b2c3d4-..."
}
```

Intentional breaking change from Layer 2 — specialist results are now tagged unions.

---

## Testing

### Unit tests (no Quarkus)

- `SpecialistOutcome<T>` — pattern matching exhaustiveness on all three variants
- `NaiveAmlInvestigator` — all outcomes `Completed`; `sarNarrative` produced
- `NaiveSarDraftingService` — all three `SpecialistOutcome` variants produce correct narrative
- `ComplianceReviewLifecycle` — correct `WorkItemCreateRequest` fields (deadline, group, callerRef)
- `OsintScreeningBehaviour` — always `Declined` with expected capability and reason
- `EntityResolutionBehaviour`, `PatternAnalysisBehaviour` — return `Completed` with expected shape

### `@QuarkusTest`

Scheduler is disabled (already configured). `PushAgentDispatch` registers stubs as
channel backends on startup — fan-out fires immediately on COMMAND send, `wait_for_reply`
resolves before HTTP response. Verify `casehub-qhorus-testing` utilities before finalising
test approach — may provide channel harness support that removes the need for live stubs
in tests.

**Three scenarios for `AmlInvestigationResourceTest`:**

1. **Happy path** — HTTP 200; entity and pattern are `Completed`; OSINT is `Declined`
   with correct reason; `complianceReviewTaskId` non-null
2. **DECLINE is not an error** — response is 200, not 4xx/5xx; SAR narrative mentions decline
3. **WorkItem created regardless of DECLINE** — WorkItem in DB with 30-day deadline

---

## What This Does NOT Do

- No Flyway migration — qhorus manages its own schema; infrastructure already configured
- No ledger writes — qhorus writes `MessageLedgerEntry` automatically per message (Layer 4 concern)
- No adaptive routing on DECLINE — handled by engine in Layer 5
- No change to `api/pom.xml` — `casehub-qhorus-api` not needed (qhorus integration is in `app/`)
- No change to `AmlInvestigationResource` — injects `AmlInvestigationApplicationService`, unchanged
- `NaiveSarDraftingService` remains instantiated with `new` inside `QhorusAmlInvestigator`
  (same pattern as Layer 1) — not promoted to a CDI bean

---

## Protocol Compliance

- `qhorus-actor-type-mapping.md` — specialist agents are `ActorType.AGENT`
- `gateway-backend-registration-ordering.md` — `open()` before `registerBackend()` in `PushAgentDispatch`
- `qhorus-event-content-null.md` — EVENT messages excluded from `check_messages` in tests
- `layer-log.md` — LAYER-LOG.md Layer 3 entry started when implementation begins, filled when code ships
- Hexagonal placement — `AmlInvestigator`, `AgentBehaviour`, `AgentDispatchMechanism` in `app/`
