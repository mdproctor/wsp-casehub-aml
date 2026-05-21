---
layout: post
title: "Naming the Harness"
date: 2026-05-19
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [casehub, agentic-harness, framing]
---

The parent docs described casehub-aml as "an agentic harness for Anti-Money Laundering investigation." I read that sentence back and it didn't sit right. The application isn't the harness — it's what runs on the harness.

The CaseHub foundation is the harness: casehub-engine, casehub-ledger, casehub-work, casehub-qhorus, casehub-connectors. That's the non-model runtime infrastructure that coordinates agents, enforces accountability, adapts paths, and produces the audit trail. I brought Claude in to check this against the industry before we changed anything.

The definition is unambiguous. LangChain's formulation captures it cleanly: *Agent = Model + Harness*. The model reasons; the harness keeps it on rails — orchestration, memory, human-in-the-loop gates, tool dispatch, audit. Every piece of the CaseHub foundation maps to a harness function:

| Module | Harness function |
|--------|-----------------|
| casehub-qhorus | Agent communication — formal obligation per interaction |
| casehub-work | Human-in-the-loop — SLA-bounded gates, escalation |
| casehub-engine | Orchestration — adaptive paths, task routing |
| casehub-ledger | Persistence and audit — tamper-evident evidence chain |
| casehub-connectors | External tool dispatch |

casehub-aml is an AML investigation application built on that harness. Calling it "an agentic harness" is like calling a Kubernetes application "a Kubernetes."

We fixed it across 11 files in the parent repo — tutorial-strategy.md, AGENTIC-HARNESS-GUIDE.md, four repo deep-dives, five protocol files. The collective noun "CaseHub agentic harnesses" (used throughout to mean the domain apps) became "CaseHub domain applications."

The original framing was trying to say something true: every application built on CaseHub expresses the harness pattern. But expressing a pattern isn't the same as implementing the infrastructure that embodies it. The distinction changes how you explain what the foundation does and what the application layer adds on top.

The journal work also surfaced a protocol worth writing down. JOURNAL.md entries need a `§SectionName` anchor in the header for the epic close machinery to find them. Miss it and the entry is silently skipped at merge — the narrative never reaches DESIGN.md. The machinery scans for `§` specifically, not the heading level or separator character.
