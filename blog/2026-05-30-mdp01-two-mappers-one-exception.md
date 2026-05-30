---
layout: post
title: "Two mappers, one exception"
date: 2026-05-30
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-aml]
tags: [aml, layer-log, exception-mapper, jackson, trust-routing]
---

Last entry closed with "Layer 7 is next." Two items had to go first.

The first was the full Layer 6 entry in LAYER-LOG.md — a stub had been standing in for weeks. Writing a proper LAYER-LOG entry means going back through the git history commit by commit, reading the actual source files, and verifying every claim. The layer added 7 workers, each with a specific Beta(α,β) seed. I had remembered the pattern as "seniors at Beta(9,1), juniors at Beta(2,8)." That's only true for the SAR drafting pair. The OSINT senior is Beta(9,2). The entity resolution, pattern analysis, and senior analyst agents are all Beta(8,2). A code review caught this — and rightly so. A LAYER-LOG entry that tells someone to seed the wrong initial values is worse than no entry at all.

The entry also needed the four gotchas documented properly: the TrustBootstrapSource SPI that never fires on a fresh deployment, the Flyway V2000 collision with engine-ledger, the CaseLedgerEntryRepository CDI regression, and the StartupEvent priority ordering that makes the seeder fire before the engine registers its case definitions. Those took time to reconstruct — the symptoms were in the git commit messages, the causes had to be traced from the code. But that's the point of the log: so someone replicating this pattern in another domain doesn't spend a day debugging what we already figured out.

The second item was `IllegalArgumentExceptionMapper` — a JAX-RS `@Provider` that maps `IllegalArgumentException` to HTTP 400. The previous entry added a compact constructor to `SarOutcome` that throws `IllegalArgumentException` for out-of-range scores. Without a mapper, clients get a 500.

I wrote one mapper. One `@Provider @ApplicationScoped ExceptionMapper<IllegalArgumentException>`. Three unit tests. Done.

Then a code review. Claude came back with a critical finding: the mapper would never fire for the compact constructor path. When Jackson deserialises the JSON body and the record constructor throws, Jackson doesn't propagate the `IllegalArgumentException` — it wraps it as `ValueInstantiationException extends JsonMappingException`. JAX-RS ExceptionMapper matching is based on the exception type hierarchy of what was thrown, not its cause. The mapper was correct. It just couldn't see the exception.

The fix is two mappers in tandem. One for direct service-layer throws. One for `JsonMappingException` that inspects `getCause()` — if the cause is `IllegalArgumentException`, return 400 with the cause's message; for anything else, return a generic 400 without exposing Jackson internals. We unit tested both paths using `ValueInstantiationException.from(null, message, SimpleType.constructUnsafe(Object.class), cause)` — `null` is valid for the `JsonParser` argument when constructing the exception outside a real parser context.

The wrapping behaviour is in the garden now as GE-20260530-3562b0. It trips people up because the single-mapper approach works perfectly for service-layer throws — you can test it, watch it fire, and believe the implementation is complete. The deserialization path is invisible until you specifically test it.

Layer 7 is next.
