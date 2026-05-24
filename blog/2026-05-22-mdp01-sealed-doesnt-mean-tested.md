---
layout: post
title: "Sealed Doesn't Mean Tested"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [testing, java, sealed-interfaces, quarkus, casehub]
---

The sealed interface `SpecialistOutcome<T>` has three permits — `Completed`, `Declined`, `Failed`. Every switch expression that pattern-matches on it must handle all three, or it won't compile. Java enforces this.

Tests don't.

`NaiveSarDraftingService.draft()` takes three parameters: one per specialist. For each, it switch-matches the outcome to a narrative fragment. Nine cases total, nine switch arms, all required by the compiler. The implementation was complete.

The tests weren't. We had osint covered in full — completed, declined, failed, each with a test verifying the narrative contained the right signal. Entity and pattern had only completed fixtures. Six arms had never been exercised.

The implementation was correct. The switch expressions handled all nine combinations correctly. But there was no evidence of that — just the compiler's assurance the code was present, and the tests' silence about whether it did anything sensible when an entity resolution declined or a pattern analysis failed mid-investigation.

The fix was four tests and four field declarations. The assertion style was already established by the osint tests: keyword presence with `||`, tolerant of wording but verifying the outcome was acknowledged:

```java
@Test
void draft_withDeclinedEntity_includesDeclineInNarrative() {
    String narrative = service.draft(tx, declinedEntity, completedPattern, completedOsint);
    assertTrue(narrative.contains("declined") || narrative.contains("clearance"),
            "Narrative should reference the entity decline: " + narrative);
}
```

Four more like this. The coverage matrix is now complete across all three specialists.

A review pass caught a dead local in `draft_withCompletedOsint_includesTransactionId` — the field extraction had introduced a local variable `osint = completedOsint` that existed solely to be passed to the next line. Claude flagged it; one line removed.

The final test run also surfaced a pre-existing failure: `AmlInvestigationResourceTest` throws `NoClassDefFoundError: io/casehub/ledger/api/model/ActorType` at runtime. The class reaches the compile classpath via casehub-ledger-api through qhorus, but doesn't land on the test classpath. None of the tests we added are affected — they're pure unit tests without Quarkus — but the REST integration tests have been quietly broken.
