---
layout: post
title: "The Missing Abstraction Between Engine and Ledger"
date: 2026-06-29
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [domain-model, investigation-outcome, adversarial-review]
---

## The Missing Abstraction Between Engine and Ledger

Both Layer 6 and Layer 9 resources had the same six lines of code: check the cache, fall back to the repo, compare against `CaseStatus.COMPLETED`. Both returned 200 with `"in-progress"` for cases that didn't exist — a bug hiding behind duplication.

The duplication wasn't the problem. The problem was a missing domain concept. The engine knows whether a case is complete. The ledger knows what the officer decided. Nothing bridged the two into "what is the current state of this investigation?" — so every caller reinvented the bridge inline.

`InvestigationResolution` is that bridge. A record with an `InvestigationStatus` enum and an `InvestigationOutcome`. The service resolves it. The resources consume it. Nonexistent cases produce `Optional.empty()` — which becomes a 404 instead of a lie.

## What the Review Caught

The adversarial review ran four rounds and surfaced things I'd missed. Two stood out.

The `domainContentBytes()` change would have broken Merkle hash verification for existing entries. Adding `rejectionReason` to the pipe-delimited output changes the bytes — and the hash was computed at save time. Existing entries with null `rejectionReason` need to produce identical bytes to the pre-change implementation. The fix: only use the pipe-delimited format when the new field is non-null.

The second catch: `@JsonValue` on `InvestigationStatus` in the api/ module. The api/ module has zero framework dependencies — no Jackson, no JPA, no Quarkus annotations. The enum needed `toWireFormat()` and `fromWireFormat()` methods with the annotations applied via a Jackson mixin in `AmlJacksonConfig` instead. Same registration mechanism as `SpecialistOutcome`, different annotation types.

## Layer 9 Doesn't Have SAR Outcomes

This one tripped both the plan and the adversarial reviewer. Both assumed Layer 9 investigations would produce `sar-filed` or `gate-rejected` outcomes like Layer 6. They don't.

Layer 9's YAML case definition has only `entity-resolution → entity-link-proposal → investigation-summary`. No `sar-drafting`, no `compliance-review`. The `AmlWorkItemLifecycleObserver` fires for `aml:investigation:<UUID>` callerRefs — gate WorkItems use `case:<UUID>/gate:<name>` and are invisible to it. So `outcome` is always null for Layer 9. Correctly null — there's no officer review to produce an outcome.

The resources look structurally identical — same service injection, same `resolveInvestigation()` call, same return type. The difference is entirely in the YAML case definition and the callerRef pattern, neither of which is visible from the resource code. The assumption survived four rounds of design review before implementation exposed it.
