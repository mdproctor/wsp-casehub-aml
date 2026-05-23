# 0001 — case-id as investigation aggregate identifier

Date: 2026-05-23
Status: Accepted

## Context and Problem Statement

Layer 4 introduces a shared `subjectId` on all ledger entries for one investigation
(both AML domain entries and qhorus message entries). The `subjectId` on `LedgerEntry`
is a UUID. The natural candidate for "what is this investigation about" is the
external transaction ID (`SuspiciousTransaction.id()`), but that is a String (e.g.
"TXN-2024-001"), not a UUID.

## Decision Drivers

* `LedgerEntry.subjectId` is `UUID` — cannot directly hold a String transaction ID
* The investigation is a distinct aggregate from the transaction that triggered it
  (in Layer 5+ a case may span multiple transactions)
* The UUID must be consistent across AML domain entries and qhorus message entries
  within one investigation
* Changing `SuspiciousTransaction.id()` to return UUID breaks all existing callers

## Considered Options

* **Option A** — Generate a fresh `UUID caseId` per investigation (used as subjectId)
* **Option B** — Derive a deterministic UUID from the transaction ID string via
  `UUID.nameUUIDFromBytes(txId.getBytes(UTF_8))`
* **Option C** — Change `SuspiciousTransaction.id()` to return UUID

## Decision Outcome

Chosen option: **Option A** — generate a fresh `UUID caseId` in `AmlInvestigationCoordinator`
at investigation start, because it accurately models an investigation case as a distinct
domain aggregate from the transaction, is the simplest implementation, and naturally
extends to Layer 5+ scenarios where a case may be opened for multiple related transactions.

### Positive Consequences

* Clean domain model: an investigation case is a separate aggregate from the triggering transaction
* No string-to-UUID conversion logic in the critical path
* Naturally supports multi-transaction cases (Layer 5+) without model change
* `caseId` is returned in `AmlInvestigationResult` — callers can reference it independently

### Negative Consequences / Tradeoffs

* The transaction ID and case ID are separate — a query by external transaction ID does not
  directly yield the case ID without looking up the ledger entries
* Each investigation generates a new random UUID even for identical transactions
  (no determinism for re-runs)

## Pros and Cons of the Options

### Option A — Fresh UUID per investigation

* ✅ Clean domain model (case ≠ transaction)
* ✅ No conversion logic
* ✅ Extends naturally to multi-transaction cases
* ❌ Lookup by external transaction ID requires a ledger query

### Option B — Deterministic UUID from transaction ID string

* ✅ Deterministic (repeatable for the same transaction ID)
* ✅ No new identifier to track
* ❌ Hidden coupling between string transaction IDs and UUID namespace
* ❌ `nameUUIDFromBytes` uses MD5 — has (vanishingly small) collision risk
* ❌ Re-runs of the same investigation get the same UUID, conflating separate events

### Option C — Change SuspiciousTransaction.id() to UUID

* ✅ Purest model (transaction IS identified by UUID)
* ❌ Breaking change to all existing callers including all tests
* ❌ External transaction IDs are typically strings in banking systems — forces conversion at ingestion

## Links

* Implemented in casehubio/aml#30 (Layer 4: FinCEN audit trail)
* `AmlInvestigationCoordinator.investigate()` — where `caseId = UUID.randomUUID()` is generated
