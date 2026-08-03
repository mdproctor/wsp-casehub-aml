---
layout: post
title: "Three agents, four issues, one merge"
date: 2026-08-03
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [parallel-agents, cross-repo, maven, snapshot, worktree]
---

The last session left AML green with a wave plan: four independent S-scale issues, all different subsystems, no shared files. The kind of backlog that begs to be parallelised.

I dispatched three Claude agents simultaneously, each in its own git worktree — isolated copies of the repo where they could write code without stepping on each other. The fourth issue (#115, a cross-repo rename) had to wait because its SNAPSHOT publish would break the other agents' builds mid-flight.

The agents came back within fifteen minutes. All three had implemented their features with tests. Two of the three had also, independently, fixed the same pre-existing compilation break — a `MemoryInput` constructor that gained a seventh parameter in a neocortex-memory-api SNAPSHOT nobody had updated for. Each agent discovered it, fixed it, and moved on. Clean engineering by each agent in isolation; a merge conflict waiting to happen in aggregate.

## The merge is where the work is

Cherry-picking the first agent's commit brought the MemoryInput fix along with the trust score snapshot feature. The second agent's commit had its own copy of the same fix — overlapping changes on the same lines. The third agent hadn't fixed it at all because it couldn't run the test suite. Three different responses to the same pre-existing break, all perfectly reasonable in isolation.

Then the tooling bit. I used `ide_replace_text_in_file` to apply one agent's PlanTrace constructor fix. The tool HTML-encoded the angle brackets: `Map.<String, Object>of()` became `Map.&lt;String, Object&gt;of()`. The tool reported success. Maven reported `<identifier> expected`. The connection between "tool said 1 replacement made" and "Java can't parse HTML entities" took longer to trace than the actual fix.

On top of that, a new engine SNAPSHOT had quietly added `LeastLoadedAgentStrategy` as a `@Default` CDI bean — competing with the `ComposableAgentRoutingStrategy` that AML uses for trust-weighted routing. No changelog, no migration note. The ambiguity surfaced during the Quarkus augmentation phase as an `AmbiguousResolutionException` naming the interface, not the new bean. You have to read the "available beans" list in the stack trace to spot the interloper.

## When a rename cascades through three repos

The fourth issue looked simple: rename `DecisionContextSanitiser` to `ContentSanitiser` in casehub-ledger. The interface has a single `String sanitise(String)` method. The new name reflects that it handles both structured JSON and prose content, not just decision context blobs.

IntelliJ's refactor-rename handled the ledger side in one operation — interface, implementation, producer, repositories, tests, all updated. Six files, thirty seconds.

Then I discovered the qhorus module also injects the interface. Its `QhorusLedgerEntryRepository` had its own import and field. That's a second repo to branch, update, build, and install. Fine — two more lines changed, two minutes of work.

The real problem was Maven. The ledger's `settings.xml` configured a custom `<localRepository>` at `worktrees/30/.m2`. My `mvn install` faithfully placed the renamed jar there. AML's Maven resolved from `~/.m2/repository` — the standard path. The new class existed in one local repo; the old class persisted in the other. `mvn compile` found the old jar, couldn't find `ContentSanitiser`, and reported "cannot find symbol" for a class I'd just installed.

Worse: even after installing to the correct path, old timestamped SNAPSHOT jars (`casehub-ledger-0.2-20260730.095553-251.jar`) sat alongside the fresh install. Maven sometimes resolved the timestamped remote copy instead of the local one. I ended up deleting the old jars manually and forcing `-Dmaven.repo.local` on every cross-repo install.

A rename that looked like thirty minutes of work turned into an hour of Maven archaeology.

## What the parallel approach actually teaches

The dispatch itself was the easy part. Three agents, three worktrees, fifteen minutes of wall-clock time for what would have been forty-five minutes sequential. The speedup is real for genuinely independent work.

But independence is a property of the *merge*, not the *implementation*. Three agents working on different features will discover the same pre-existing break, fix it three different ways, and hand you a reconciliation problem. The tooling that works in the main project (IntelliJ MCP, semantic code navigation) doesn't work in worktrees — agents fall back to text tools, which means less precise edits and more manual verification.

And cross-repo work can't be parallelised with in-repo work at all. A SNAPSHOT publish changes the shared dependency; every concurrent build is now compiling against a moving target.

The practical takeaway: dispatch agents for independent in-repo work, do cross-repo changes sequentially after everything else lands, and budget the merge at half the implementation time. The agents write code; the merge is where you earn the result.
