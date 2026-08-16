## D1: PROV-DM mapping model

**Choice:** Entry + Activity pairs (rich) — each LedgerEntry maps to both a prov:Entity (the tamper-evident record) and a prov:Activity (the event it records)
**Alternatives:**
- Entries-as-Entities (flat) — simpler but omits Activities, not idiomatic PROV-DM, standard tools render poorly
- Grouped activities (phase-level) — better readability but requires inference logic to group entries by investigation phase, fragile against new entry types
**Rationale:** Idiomatic PROV-DM modeling. Mechanical mapping (no inference). Separates concerns: tamper-evidence on Entity, domain facts on Activity. Standard PROV tools render correctly. ~52 nodes for typical investigation — well within tool limits.
**Trade-offs:** ~2x node count vs flat approach. More mapping code (boilerplate, not complex).
**Exploration:** quick
**Status:** captured

## D2: Serialization format

**Choice:** PROV-JSON only
**Alternatives:**
- PROV-JSON + PROV-N — human-readable text format for debugging, but convertible from PROV-JSON via standard tools
- Content negotiation (PROV-JSON, PROV-XML, PROV-N) — maximum interoperability but three serializers for no current consumers
**Rationale:** Issue body specifies PROV-JSON. Every PROV tool imports it. Additional formats addable later via content negotiation without changing the domain model.
**Trade-offs:** No human-readable format for quick visual inspection (mitigated by PROV tool converters).
**Exploration:** quick
**Status:** captured

## D3: Merkle inclusion proofs in the export

**Choice:** Include as Entity attributes — digest and inclusion proof as `casehub:` namespaced attributes on each prov:Entity
**Alternatives:**
- Separate verification endpoint — keeps PROV-DM clean but requires CaseHub access to verify, defeating "independently auditable"
- Optional via query parameter — flexibility but premature for pre-release
**Rationale:** "Independently auditable outside CaseHub" requires proofs to travel with the data. PROV-DM supports domain-specific attributes via namespace prefixes. Digest is a single string; inclusion proof is a hash list — both fit as attributes.
**Trade-offs:** Slightly larger JSON payload. Non-standard attributes on PROV entities (mitigated by namespace prefixes, which is the PROV-DM-endorsed extension mechanism).
**Exploration:** quick
**Status:** captured

## D4: GDPR-erased actor representation

**Choice:** No special handling — use tokenised pseudonym from ledger as-is
**Alternatives:** None — the ledger's identity tokenisation already handles erasure. An explicit `gdprErased` flag was considered but rejected: the combination of `actorType=HUMAN` + pseudonymised ID already signals erasure, and an explicit flag would require tracking erasure state outside the ledger.
**Rationale:** The PROV-DM export reads from the ledger after tokenisation. Pseudonymised actorId becomes the prov:Agent identifier. actorType (SYSTEM/HUMAN/AGENT) is included as a casehub: attribute so consumers know the agent's role category even after erasure.
**Trade-offs:** None — this is a non-decision. The ledger handles it.
**Exploration:** quick
**Status:** captured

## D5: API surface

**Choice:** `GET /api/investigations/{caseId}/provenance` returning PROV-JSON (`application/json`). 404 if case not found.
**Alternatives:**
- Nested under compliance-evidence (`/compliance-evidence/provenance`) — unnecessarily deep, provenance is a peer resource not a child
- Separate `/api/provenance/{caseId}` path — breaks the existing `/api/investigations/{caseId}/*` convention
**Rationale:** Follows established Layer 7 pattern. Thin JAX-RS resource (`AmlProvenanceResource`) delegates to `AmlProvenanceService` (query + mapping). Lives alongside compliance evidence code — same audit/compliance concern.
**Trade-offs:** None significant.
**Exploration:** quick
**Status:** captured
