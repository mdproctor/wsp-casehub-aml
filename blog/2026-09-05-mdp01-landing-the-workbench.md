---
layout: post
title: "Landing the Workbench — Eight Batches and Four Upstream Components"
date: 2026-09-05
entry_type: note
subtype: diary
projects: [casehubio/aml]
tags: [casehub-pages, blocks-ui, dockWorkbench, composition, push-updates, scenario-automation]
series: issue-111-workbench-spec-rewrite
---

# Landing the Workbench — Eight Batches and Four Upstream Components

The v2 spec said "compose from existing components." The implementation proved whether that claim held up under contact with real code.

It did — but the interesting part wasn't the composition itself. It was the upstream work that composition demanded. Four times during implementation, I hit a gap where the right component didn't exist yet. Each time, the choice was: build it in AML (fast, wrong place) or create an upstream issue in blocks-ui or pages (slower, right place). We chose upstream every time.

`blocks-case-flow-viewer` was the first. The spec called for rendering the investigation flow DAG with runtime state overlay — status badges, trust score pills, parallel group layout. `casehub-diagram` could do it, but it's a full editor with palette, YAML editing, and property panels. Overkill for a read-only visualization. `blocks-dag-viewer` does exactly this for HTN plans using `graph-stencil-htn`. The gap: nobody had built the equivalent for case flows using `graph-stencil-case`. So we filed blocks-ui#150 and it shipped.

The AML wrapper? Ninety lines. Half of that is `flowToRuntimeState()` — a pure function that maps the engine's flow response to `CaseRuntimeState`. The diagram rendering, ELK layout, trust score badges, runtime decorations — all handled by the shared component.

`blocks-worker-task-pane` was the meatiest upstream creation. The domain worker workbench — where analysts do entity resolution, pattern analysis, OSINT screening, SAR drafting — has a generic shell (task queue, investigation context, response form) and a domain-specific middle (the specialist workspace). The shell is identical across any CaseHub application with specialist workers. The workspace slot pattern lets each app register its own specialist elements by capability tag. AML registers five; the next application registers different ones against the same scaffold.

The push story was the most instructive. The original spec designed SSE endpoints — five `@RestStreamElement` resources broadcasting via `Multi<SseEvent>`. During implementation, I discovered the platform had quietly unified on WebSocket transport. `SSEManager` was gone. `EventConnection` in `pages-data` speaks WebSocket exclusively — topic-based subscriptions with cursor recovery that SSE can't provide. The backend pattern is a `@WebSocket` endpoint delegating to `EventBroadcaster` from `casehub-pages-push`. The same boilerplate exists in the helpdesk example, FSI trading, and now AML — a clear candidate for extraction into a shared bean.

The net result: 2,820 lines added, 1,298 deleted. The deleted lines include a 950-line hand-rolled operations view replaced by `blocks-kpi-metric-row` with two endpoint calls, and the entire `showcase/` mock-fetch system replaced by six YAML scenario scripts that drive real backend execution.

What I didn't anticipate was how natural the "stop and create an upstream issue" pattern would become. The first time felt like an interruption. By the fourth, it felt like the correct granularity of work — the workbench implementation was teaching us what the shared libraries were still missing. Every upstream issue made the next CaseHub application cheaper to build.
