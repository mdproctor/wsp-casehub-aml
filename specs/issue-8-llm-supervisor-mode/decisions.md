# Decisions — Epic #8: LLM Supervisor Mode

## D1: LLM scope — procedural binding selection only

**Choice:** LLM PlanningStrategy controls investigation flow (binding selection, ordering, early termination). Deterministic `InvestigationTriageEvaluator` remains the quality gate for SAR/FALSE_POSITIVE/INCONCLUSIVE decisions.
**Alternatives:**
- Smarter triage judgment — LLM replaces or augments `InvestigationTriageEvaluator`. Rejected: collapses the accountability boundary between "how to investigate" and "what the evidence means"; triage decision chain must remain fully auditable for FinCEN examiners.
- Both (full supervisor) — LLM controls flow AND triage. Rejected: same accountability concern plus no independent check on evidence-gathering bias; two failure domains collapsed into one.
**Rationale:** The deterministic system's real weakness is procedural (static JQ conditions, full pipeline always runs), not substantive (hard gates + scoring + CBR handle triage well). The LLM adds genuine capability where the deterministic system is weakest — adaptive investigation paths, early termination, domain-aware sequencing — while preserving defense-in-depth for the regulatory decision.
**Trade-offs:** The LLM cannot override a deterministic triage outcome — if it skips evidence that would have changed the score, the triage evaluator works with what it has. This is a feature (defense-in-depth), but means the LLM's procedural quality directly affects triage input quality.
**Exploration:** deep-analysis
**Status:** captured

## D2: LLM integration — ChatModel via engine-ai ChatModelProvider

**Choice:** Use `ChatModelProvider` from `casehub-engine-ai` which returns `dev.langchain4j.model.chat.ChatModel` directly. Built-in providers: Anthropic (`AnthropicChatModelProvider`), Ollama, OpenAI, Gemini. Dependency: `casehub-engine-ai` (already a natural engine dependency for LLM-enabled applications).
**Alternatives:**
- `AgentProviderChatModel` wrapping `ClaudeAgentProvider` (Claude CLI subprocess) — rejected after decision review R1-03: impedance mismatch; wraps a subprocess-based multi-turn agent provider to present a ChatModel interface for a single-turn structured decision. Wrong abstraction at the wrong layer.
- Raw `AgentProvider` directly — subprocess per call, multi-turn agent sessions. Rejected: too heavy for single structured selection decisions.
- `TextClassifier` from `casehub-neocortex` — lightweight but insufficient expressiveness for contextual binding selection over variable-length evidence.
**Rationale:** `engine-ai` is the engine's native LLM integration layer. It returns `ChatModel` natively, supports prompt caching via the Anthropic SDK, and is provider-swappable without platform-layer subprocess dependencies. The `PlanningStrategy` lives in the engine planning module — using the engine's own LLM layer is the natural fit.
**Trade-offs:** Ties to LangChain4j's Anthropic SDK rather than the Claude CLI SDK. The Anthropic SDK supports the same features (prompt caching, structured output) via direct API calls rather than subprocess management.
**Exploration:** quick
**Status:** revised (R1-03)

## D3: Operating mode — filter, not override

**Choice:** Filter mode. The YAML JQ conditions remain the eligibility gate. The LLM receives only already-eligible bindings and selects a subset to fire. The LLM can suppress but cannot add bindings whose conditions haven't matched.
**Alternatives:**
- Override mode — LLM receives all bindings and makes all selection decisions, bypassing JQ conditions. Rejected: JQ conditions encode domain invariants (e.g., entity resolution before pattern analysis) that must be enforced regardless of LLM judgment; would require engine SPI changes to pass all bindings.
**Rationale:** The `PlanningStrategy` SPI naturally receives `List<Binding> eligible` — filter mode is the intended contract. JQ conditions enforce safety invariants; the LLM adds value by making within-eligible decisions ("three things are eligible, fire only this one first").
**Trade-offs:** The LLM cannot fire a specialist whose preconditions haven't been met. If the LLM judges additional evidence is needed beyond what's eligible, it can only express this by suppressing other bindings (forcing the engine to re-evaluate after the next context change).
**Exploration:** quick
**Status:** captured

## D4: Invocation frequency — selective, not every cycle

**Choice:** Selective invocation. The LLM is consulted only when there are genuine choices to make — multiple eligible bindings, or when specific domain conditions suggest a decision point (e.g., triage eligibility, early termination candidate). Single-eligible-binding cycles pass through without LLM overhead.
**Alternatives:**
- Every cycle — LLM consulted on every non-empty `select()` call. Rejected: 5-8 LLM calls per investigation, most with obvious single-binding answers. Unnecessary cost and latency.
- First-call-only — LLM produces a full investigation plan once at case start. Rejected: least adaptive; can't react to unexpected findings from specialists; plan invalidation logic adds complexity.
**Rationale:** Most planning cycles have one obvious eligible binding. The LLM adds value at genuine decision points — parallel specialist selection, early termination, sequencing choices. A simple gate (eligible count > 1, or triage-related binding present) keeps calls to ~2-3 per investigation.
**Trade-offs:** The invocation gate must be carefully designed — too narrow and the LLM misses decision points; too wide and it's called on every cycle anyway. The gate conditions become part of the domain logic.
**Depends on:** D3 (filter mode determines what "eligible" means)
**Exploration:** quick
**Status:** captured

## D5: Fallback — degrade gracefully with marker

**Choice:** On LLM failure (unavailable, timeout, unparseable response), fall back to firing all eligible bindings (ChoreographyStrategy behavior) and write a context marker (`supervisorDegraded: true`) so downstream consumers know the investigation ran without LLM guidance.
**Alternatives:**
- Silent fallback to ChoreographyStrategy — fire all eligible, no marker. Rejected: triage evaluator and audit trail should know this investigation didn't benefit from adaptive evidence gathering.
- Fail the planning cycle — return empty list. Rejected: risks stalling the investigation; regulatory failure if LLM stays down.
**Rationale:** The investigation must never stall because of an LLM failure — that's a regulatory risk. But the degradation signal is cheap and informative for compliance review and audit purposes.
**Trade-offs:** The context marker adds a field to the case context that triage and other consumers need to be aware of (even if they don't act on it). Must be documented as a known context field.
**Exploration:** quick
**Status:** captured

## D6: Audit — both context field and ledger entry

**Choice:** Write supervisor decisions to case context (for downstream worker visibility) AND to a tamper-evident `AmlSupervisorDecisionLedgerEntry` extending `JpaLedgerEntry` (for compliance audit). Both on every LLM-consulted planning cycle.
**Alternatives:**
- Context field only — rejected after decision review R1-11: procedural decisions that influence which evidence reaches the triage evaluator have substantive regulatory consequences. A mutable context field is invisible to the compliance evidence chain (`GET /api/layer7/evidence/{caseId}`).
- Ledger entry only — rejected: downstream workers need to see supervisor rationale in real-time context, not by querying the ledger.
**Rationale:** The supervisor's procedural decisions directly affect triage input quality (D1 trade-off). If the LLM suppresses OSINT screening, the regulatory outcome may change. FinCEN examiners tracing "why no SAR?" must reach a tamper-evident record of the procedural decision that shaped the evidence. The platform's architectural identity is tamper-evident accountability — supervisor decisions belong in the Merkle chain from the start.
**Trade-offs:** One ledger entry per LLM-consulted planning cycle (~2-3 per investigation). Adds a Flyway migration for the subclass join table. Acceptable cost for audit completeness.
**Depends on:** D1 (procedural scope, but with substantive consequences acknowledged)
**Exploration:** quick
**Status:** revised (R1-11)

## D7: Placement — in app/ module

**Choice:** `AmlInvestigationSupervisor` lives in `app/` alongside other case-definition-specific CDI beans (`AmlInvestigationCaseHub`, `AmlInvestigationCaseDescriptor`, etc.). Activated via `@ApplicationScoped` with `id()` returning `"aml-supervisor"`. YAML case definition sets `planningStrategy: aml-supervisor`.
**Alternatives:**
- New `supervisor/` Maven module — isolates LLM dependencies from core `app/`. Rejected: premature separation for a single CDI bean; dependency footprint is two JARs, not a framework.
**Rationale:** The supervisor is one CDI bean with a focused concern. It belongs alongside the other AML case-definition beans. The LLM dependencies (`agent-claude`, `agent-langchain4j`) are lightweight additions to `app/pom.xml`.
**Trade-offs:** LLM dependencies are on the `app/` classpath even in deployments that don't use supervisor mode. Mitigated by the `planningStrategy: aml-supervisor` YAML config — the bean exists but is only resolved when the case definition references it. With D2 revised to use `casehub-engine-ai` (already a natural engine dependency), the footprint concern is further reduced.
**Exploration:** quick
**Status:** captured
