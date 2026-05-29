---
layout: post
title: "Before Layer 7: invariants at the boundary and a retroactive plan"
date: 2026-05-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [aml, layer-log, domain-model, trust-routing]
---

Layer 6 shipped last entry. Between the trust loop closing and Layer 7 starting, there was a small backlog to clear — two doc improvements and a quality fix that had been sitting in the queue.

The doc work turned out to require more thought than it looked. The task was to add a Vertical Slice Index to LAYER-LOG.md: a table showing which slices the AML tutorial constitutes and which layers each covers. The challenge is that AML was built layer-by-layer — the guidance to build vertical slices before deepening any single layer arrived after Layers 1–3 were already done.

Mapping that history to slices retrospectively means deciding where one slice ends and the next begins. My first instinct was one slice per layer, but that's wrong. Layers 1, 2, and 3 together form S1, the first complete working path. A flagged transaction that dispatches typed COMMAND messages to specialist agents and ends with a compliance WorkItem is the thinnest useful thing the tutorial demonstrates — and it needs all three layers to exist. Layer 2 without Layer 3 has a WorkItem but no typed agent interactions; Layer 3 without Layer 2 has dispatch but no formal SLA on what happens after. They belong in the same slice.

The ordering rationale between slices is also more subtle than it first appears. S1→S2 looks like a hard dependency — ledger builds on qhorus. It isn't. Ledger can record any entry type without qhorus in play. The dependency is soft: qhorus gives the audit trail something meaningful to record. Getting that distinction right matters for anyone building a different application on the same stack and wondering what order to work in.

The quality fix was on `SarOutcome`, the record type that carries a SAR outcome back to update trust scores. The original had no validation — any double could be stored as `investigationAccuracyScore`, including values below zero or above one. A compact constructor handles the range check in six lines of pure Java, no framework involved.

What made it worth writing about: a code review caught something I hadn't considered. A null `verdict` field — which Jackson produces without complaint if the client submits a JSON body missing the field — would survive the range check and NPE inside `SarOutcomeFeedbackService` when it tries to map verdict to an attestation type. Accept a malformed record at the API boundary; produce an opaque 500 somewhere deeper. Claude caught that null `verdict` and null `reason` needed `Objects.requireNonNull` treatment alongside the range validation. Two lines added.

The double deserialization in the SAR drafting workers was straightforward: both workers already hold a `SuspiciousTransaction` when they call the shared `buildSummary` method, which was extracting the transaction from the input map and converting it a second time. Passing the pre-deserialized object through removes the duplication.

Layer 7 is next: the comparison table against IBM AMLSim and industry whitepapers. That's where the tutorial closes — showing what the CaseHub foundation gives you that a naive implementation can't, in terms a compliance team would actually recognise.
