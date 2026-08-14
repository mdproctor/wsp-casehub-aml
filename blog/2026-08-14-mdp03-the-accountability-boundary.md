---
layout: post
title: "The accountability boundary"
date: 2026-08-14
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [llm-supervisor, planning-strategy, accountability, investigation-triage]
---

The AML investigation had a nine-layer foundation, deterministic triage logic, CBR-adjusted thresholds, and a complete SAR filing pipeline. Everything worked. But every investigation ran the same way: entity resolution first, then pattern analysis and OSINT in parallel, then triage, then SAR drafting if warranted. Static YAML conditions, same sequence, every time.

A real investigator doesn't work like that. They read early findings and adjust — skip pattern analysis when a sanctions hit makes it irrelevant, add a senior analyst review when the entity structure looks unusual, proceed to triage before all specialists finish when the evidence is already overwhelming. The question was how to give the investigation that kind of adaptive judgment without undermining the regulatory guarantees the deterministic system provides.

The design pivoted on one insight: procedural decisions and substantive decisions are different accountability surfaces.

Deciding *which specialist to invoke next* is an efficiency question. Getting it wrong means a slower investigation or incomplete evidence, but it's recoverable — run the skipped specialist. Deciding *whether to file a SAR* is a regulatory question. Getting it wrong means a FinCEN violation. These are different failure domains with different consequences, and collapsing them into one LLM call removes the independent check between them.

So the LLM controls investigation flow — which bindings fire, in what order, whether to suppress or prioritize — while the deterministic `InvestigationTriageEvaluator` remains the quality gate. The LLM can't override a triage outcome. What it can do is improve the *input quality* to triage by making smarter evidence-gathering decisions. If it routes to senior analyst review for a case the static conditions wouldn't have flagged, triage gets better evidence. If it lets triage fire before OSINT completes because entity resolution already found a shell company, triage still applies its hard gates correctly — sanctions hit, confirmed PEP, shell company all trigger SAR_WARRANTED regardless of what other evidence exists.

The engine already had the extension point. `PlanningStrategy` is a pluggable SPI — `select(CasePlanModel, PlanExecutionContext, List<Binding>) → List<Binding>`. The default `ChoreographyStrategy` fires everything that's eligible. `AmlInvestigationSupervisor` replaces it for a compound of supervised bindings, consulting an LLM to filter the eligible set.

The structural review caught three things I'd have shipped without noticing. First: `select()` needs to be side-effect-free. The natural instinct is to write the audit ledger entry inside the method — you know the decision, write it. But `select()` runs inside the engine's planning loop, which owns its own transactional context. Writing to the ledger's datasource from there risks the dual-datasource anti-pattern. The fix: store decisions in an `@ApplicationScoped` `ConcurrentHashMap`, flush to ledger from an `@ObservesAsync WorkerDecisionEvent` observer that runs outside the planning cycle.

Second: `earlyTermination` in the LLM's response can't be a control signal. If the strategy returns an empty list ("fire nothing"), the engine doesn't re-evaluate — no context changes, same bindings stay eligible, hot loop. Early termination is expressed by *selecting triage and suppressing other specialists*, not by returning empty. The `earlyTermination` boolean is audit metadata.

Third: the compound scope. I'd planned a Java invocation gate inside the strategy — "only call the LLM when multiple bindings are eligible." The review pointed out that `CompoundStrategyDispatcher` already handles scoping: register a compound via `BlackboardPlanConfigurer` with `planningStrategy("aml-supervisor")`, and only the compound's bindings route through the LLM. Entity resolution, CBR path advisor, and failure-handling bindings stay under `ChoreographyStrategy`. The scope declaration lives in the case definition infrastructure, not buried in Java gate logic.

What this opens up is the investigation becoming genuinely adaptive. The static YAML conditions encoded the happy path — entity resolution before pattern analysis, all specialists before triage. The supervisor can now reason about the specific case: a high-risk jurisdiction with a complex ownership chain gets a different investigation sequence than a straightforward structuring alert. The deterministic evaluator still makes the call. But the evidence it evaluates is shaped by judgment, not just by static ordering.
