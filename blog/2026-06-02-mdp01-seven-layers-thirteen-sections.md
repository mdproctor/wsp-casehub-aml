---
layout: post
title: "Seven Layers, Thirteen Sections"
date: 2026-06-02
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [aml, arc42stories, documentation, architecture]
---

casehub-aml's LAYER-LOG has had seven complete entries since May 30th. Each one has
Pattern to replicate, Key wiring, Gotchas, Accountability gaps. The content was there.
Today Claude and I gave it a format.

ARC42STORIES.MD is a multi-mode document — different section types follow different
structural constraints. Gotchas are diagnostic: Symptom → Cause → Fix, nothing hedged.
Pattern to replicate sections are tutorial: numbered imperative steps, zero assumed
context, domain-agnostic. §1–3 are reference/lookup: tables and bullets, no prose.
§4 and §9.3 are explanation/comparative: Before:/After: contrast leading every entry.

Get the wrong mode and the content is correct but the document reads as AI-generated —
not because the words are wrong but because a reference section written as an essay asks
the reader to work too hard, and a tutorial written as discursive prose buries the steps.

I wanted those gates in place before writing started. That meant loading all five mode
files and confirming the section-to-mode mapping from the arc42stories spec. We also
read everything first: LAYER-LOG.md, the spec, the CaseHub profile, devtown's reference
implementation, ARCHITECTURE.md, the workspace DESIGN.md, the ADR, and the six blog
entries specifically flagged for gotchas.

Three gotchas from blogs weren't in LAYER-LOG.md explicitly. The sealed interface
switch-completeness problem: the compiler enforces exhaustive arms but not correct arms.
`NaiveSarDraftingService` had all nine switch arms; six had never been exercised. The
`@Transactional` coordinator failure with dual datasources: the error says "Unable to
acquire JDBC Connection [Exception in association of connection to existing transaction]",
which reads like pool exhaustion — it's JTA trying to enlist H2 in an XA transaction
it can't do. And the Jackson wrapping: a compact constructor throwing
`IllegalArgumentException` gets wrapped as `ValueInstantiationException`; a single
`ExceptionMapper<IllegalArgumentException>` never sees it.

Claude ran find against every class name in the Key files sections. All §12 issues
confirmed open. Every blog path and Flyway migration file checked. 1,518 lines,
thirteen sections, self-assessment included.

The self-assessment is honest about gaps: Layer 6's Pattern to replicate doesn't list
all 7 worker Beta seed values. The SLA GAP on the engine async path appears in §12
but the structural explanation is only in §9.4 L7 Gotchas. An LLM replicating trust
routing from ARC42STORIES.MD would choose appropriate Beta parameters based on domain
judgment — the document explains the mechanism; calibration is left to the implementer.

LAYER-LOG.md gets a migration note. CLAUDE.md now points to ARC42STORIES.MD as the
primary architecture record. Workflow: write LAYER-LOG first when a layer extends,
then sync to §9.4.
