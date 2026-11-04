# exercise-06: RAG System Card — Privacy Slice

**Estimated effort:** 3 hours

## Objective

Author the complete privacy slice of a system card for a RAG + agent deployment, following the chapter-06 nine-section template. The deliverable is a document a customer's privacy officer would receive under NDA, a regulator's technical consultant would be shown on request, and your own incident-response team would consult during a reported-leak investigation. The card must be checkable — every claim it makes should be traceable to a concrete artefact (a schema, a test, a config, a runbook).

## Scenario

Pick one of the following (or invent a plausible equivalent and justify it in a short preamble):

- **A B2B SaaS RAG support assistant.** Tenants are customer organisations; users are their employees; retrieval is scoped to the tenant's workspace; the agent can call internal tools on the user's behalf.
- **A consumer-facing personal-knowledge agent.** Each user has their own workspace; the agent can call Gmail / Calendar / Notion / Drive on the user's behalf.
- **A regulated-vertical RAG for healthcare.** Clinicians query a RAG grounded in clinical guidelines and (separately) patient-specific records; tool access is strictly audited.

Use the artefacts from the preceding exercises (01–05) where they fit. Where you do not have an exercise artefact, invent a plausible one for the scenario and flag it as a design assumption.

## Requirements

1. **Section 1 — Scope and version.** Name the system, deployment environment, card version, maintainer, DPIA reference.
2. **Section 2 — Data flow diagram.** One figure plus caption. Label every arrow with the data class flowing over it, every box with its trust owner, and every boundary with its encryption / residency properties.
3. **Section 3 — Retrieval access-control.** Isolation shape, caller-identity contract, ABAC extensions (if any). Reference the chapter-01 and exercise-01 artefacts.
4. **Section 4 — Context-window-leakage test posture.** Negative test cases, property-based layer, staging canary, production shadow, beyond-the-retriever path tests. Reference exercise-02.
5. **Section 5 — Logging and retention.** Log-schema classes, redact-before-log integration, vault, retention schedule, access matrix, provider-side logging. Reference exercise-03 and exercise-04.
6. **Section 6 — Agent-tool scoping.** The per-tool table plus the prose paragraph on token exchange, NHI, data-boundary reviewer, delegation chain audit. Reference exercise-05.
7. **Section 7 — Corpus ingest and provenance.** Sources, provenance records, PII pre-scan policy, change-rate anomaly detection, tombstoning. Even if you did not do an ingest-specific exercise, the chapter-05 artefact list is enough to populate this section as a design claim.
8. **Section 8 — Erasure and takedown.** User erasure runbook, legal takedown runbook, internal removal runbook. Internal SLOs; current attainment (invent a plausible rolling 30-day number and flag it as mock).
9. **Section 9 — Residual privacy risks and boundaries.** Model-output leakage (boundary to 105 and 106), side-channel residency, prompt injection via retrieved content, sub-processor trust, platform-security boundary.
10. **Section 10 — Changelog.** This is v1.0; one row.
11. **Appendices.** The structured artefacts the prose refers to: chunk schema, caller-identity object, tool-scope list, consent record shape, provenance record, retention table, access matrix, source list, erasure runbook, sub-processor list.
12. **Framework mapping.** A one-page appendix mapping card sections to the NIST AI RMF (Map / Measure / Manage / Govern) and to EU AI Act Articles 10, 12, 13.

## Deliverables

Place artefacts under `exercises/exercise-06-rag-system-card-privacy-slice-work/`:

- `SYSTEM_CARD.md` — the main nine-section card.
- `appendices/A_schemas.md`
- `appendices/B_test_coverage.md`
- `appendices/C_retention_table.md`
- `appendices/D_access_matrix.md`
- `appendices/E_source_list.md`
- `appendices/F_erasure_runbook.md`
- `appendices/G_data_boundary_rules.md`
- `appendices/H_subprocessors.md`
- `appendices/I_framework_mapping.md`
- `diagrams/data_flow.svg` (or `.png` — PlantUML / Mermaid / draw.io acceptable).
- `README.md` — a one-paragraph summary, the list of design assumptions you flagged, and the pointer to each exercise whose artefact you drew from.

## Starter guidance

- The main card runs three to six pages. The appendices carry the structured detail.
- Where an exercise artefact exists, quote it by pointer rather than restating it in full. "See appendix A.1 for the chunk-record schema" is better than inlining the schema in the body.
- Quantify every claim. "Retention is 90 days" not "retention is appropriate". If quantification is impossible, say *why* and name the boundary.
- Boundary statements earn trust. Section 9 is the hardest and the most important; do not skip it.
- The system card is a living document. Add the review cadence (per-deploy, quarterly, on regulatory change) to the metadata at the top.
- The framework mapping does not have to be exhaustive. Pick the three or four most load-bearing NIST subcategories and the three AI Act articles; cite them precisely.
- If you have not done the earlier exercises, mark the specific claims as design assumptions and move on. The card is a drafting exercise regardless.

## Acceptance criteria

- All nine sections are present with the subsections chapter 06 named.
- The data-flow diagram has arrows labelled with data classes, boxes with trust owners, and boundaries with encryption / residency.
- The retrieval, logging, agent-tool, and ingest sections each reference at least one checkable appendix.
- Every retention / SLO / coverage number is concrete; weasel words ("industry-standard", "appropriate", "reasonable") are not used as substitutes for numbers.
- Section 9 names at least five residual risks, each with the boundary-owner and the complementary control.
- The appendices are populated — not every one must be exhaustive, but each must contain enough for a reviewer to verify the claim in the body.
- The framework-mapping appendix cites specific NIST RMF subcategories and specific AI Act article numbers.
- The document reads in a consistent voice across sections (the chapter-06 style): quantitative, boundary-aware, verifiable.

## Stretch goals

- Produce a public-facing variant of the card: the same nine sections, with internal-only information redacted and operational detail reduced. Mark the difference in the changelog.
- Build a tiny verification script (`scripts/verify_card.py`) that reads the card's claims about retention and access, reads the corresponding YAML config from exercise 03, and reports any mismatches. The shape of the automation beats the fidelity for the exercise.
- Pair with another learner (or review your own after 24 hours) as if you were the customer's privacy officer; log the three hardest questions the review surfaced and the changes they drive in the card.
- Add a DPIA-companion paragraph that summarises the card in the shape mod-111 will consume.
