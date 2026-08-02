---
layout: post
title: "Where the sanitiser sits"
date: 2026-07-26
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [cbr, sar-narrative, gdpr, architecture]
series: issue-98-sar-narrative-seeding
---

The CBR pipeline already stores SAR narratives from past cases. It already retrieves them at case startup. The gap was the last metre: getting those narratives to the sar-drafting worker so it can use them as templates instead of generating from scratch.

Three places you could extract narratives from the retrieved experiences. Inside the sar-drafting worker itself — cheapest change, one input projection tweak. In the cbr-path-advisor, which already iterates every experience for path statistics. Or in a dedicated new worker, which felt over-engineered for what amounts to ten lines of filtering.

I recommended the worker-side approach initially. Minimal change surface, lazy extraction (only runs when triage says SAR is warranted), independently testable via a service class. Claude and I had the design nearly locked in.

Then I stress-tested it from first principles: where does the GDPR sanitisation boundary actually sit? The spec says narratives must be sanitised before reaching the worker. Under the worker-side approach, raw `cbrExperiences` — complete with unsanitised PII from past cases — flows through the input projection into the worker. The `SarNarrativeSeeder` sanitises inside the worker, but the raw data has already crossed the boundary.

Today that doesn't matter. `PassThroughDecisionContextSanitiser` is a no-op. But when workers become LLM-powered, the input projection defines the LLM's context window. Putting raw experiences in the projection puts unsanitised PII in the model's attention. The seeder sanitises after the fact, but the damage is done — the model has already seen it.

The advisor approach puts the sanitisation upstream. The advisor extracts and sanitises narratives as part of its existing experience traversal, writes them to context under `cbrPathAdvice.similarSarNarratives`. The sar-drafting worker receives only clean data. When it becomes an LLM call, its context window never contains raw PII from past cases. The architecture is correct by construction, not by discipline.

The "eager extraction is wasteful" objection dissolves under scrutiny. Filtering ten experiences for SAR_WARRANTED outcome, extracting a string field, calling `sanitise()` — microseconds, against a backdrop of database queries and async worker dispatch. The advisor already does work that's equally "wasted" in the false-positive case. Adding narrative extraction doesn't change the cost profile.

The SNAPSHOT dependency cascade was the other lesson from this branch — but that one's in the garden now, not here. The architecture is what matters: sanitisation boundaries are design decisions with consequences that only surface when the system evolves. Get them right at the boundary, not inside the consumer.
