# Handoff — #58 SAR-drafting split (mid-work)

## What this project is

*Unchanged — `git show HEAD~1:HANDOFF.md` §What this project is*

## This session (2026-06-27)

Implemented #58: separated sar-drafting from compliance-review-opening for SAR_FILING oversight gate. 7 commits on `issue-58-separate-sar-drafting` (a4154bb..6abc99e). Spec at `specs/issue-58-separate-sar-drafting/2026-06-26-sar-drafting-split-design.md`, plan at `plans/2026-06-26-sar-drafting-split.md`.

**Core changes:** sar-drafting workers converted from FlowWorkerFunction to WorkerFunction.Sync + PlannedAction(SAR_FILING). New `compliance-review-opening-agent` worker (Flow) calls `openReview()` post-MLRO gate. YAML updated (6 capabilities, 7 bindings). Trust seeder updated (8 seeds). ARC42STORIES.MD updated (counts, resolved gaps, tech debt).

**Test status:** 182/184 pass. 2 test classes fail — root cause under investigation (see below).

## Blocking issue — 2 test classes fail

`AmlLayer7ResourceTest` (3 tests) and `AmlTrustRoutingAttestationTest` (1 test) fail with `(RECIPIENT_FAILURE,8191) CaseInstance not found or wrong tenant` from `InMemoryCaseInstanceRepository.update()`.

**What's known:**
- Both pass from main's code; fail only with #58's larger case definition
- Layer 7 fails even when Layer 6 runs first in the same JVM (not a first-in-JVM issue)
- Explicit `selected-alternatives` for all persistence-memory `@Alternative` beans fixed 4 other test classes but NOT these 2
- The error happens at `startCase()` time, before any gate fires
- All other test classes (Layer 5, 6, 8, 9, SarOutcome, descriptors, CaseHub, seeder) pass

**Uncommitted change:** `app/src/test/resources/application.properties` — added 4 `@Alternative` beans to `selected-alternatives`. Commit this first.

**Next step to debug:** Add diagnostic logging to `InMemoryCaseInstanceRepository.update()` to capture `store.size()`, `instance.getUuid()`, and `tenancyId` at failure time. Or temporarily instrument `CaseHubReactor.buildInstance()` to log the save/event-bus sequence.

## Immediate next step

Resume on branch `issue-58-separate-sar-drafting`. Commit the uncommitted `application.properties` change. Then debug the Layer 7 / TrustRoutingAttestation test failures — the root cause is in the interaction between #58's case definition and the engine's `startCase()` path.

## What's left

- Debug and fix 2 remaining test failures (Layer 7, TrustRoutingAttestation) · S · High
- File follow-up issue for gate rejection handling (scoped out of #58) · XS · Low

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #14 | Layer 10 — LLM supervisor mode | L | High | Blocked on engine#101 |

## References

- Spec: `specs/issue-58-separate-sar-drafting/2026-06-26-sar-drafting-split-design.md`
- Plan: `plans/2026-06-26-sar-drafting-split.md`
- Blog: `blog/2026-06-27-mdp01-where-the-gate-meets-the-worker.md`
- Garden: GE-20260531-1e51d4 (variant — implicit @Alternative activation)
- Progress ledger: `.superpowers/sdd/progress.md`
