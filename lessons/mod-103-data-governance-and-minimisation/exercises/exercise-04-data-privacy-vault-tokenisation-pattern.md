# Exercise 04: Data Privacy Vault + Tokenisation Pattern

**Estimated effort:** 2 hours · **Output:** `vault-design.md` (architecture + policy) + working or pseudocode `vault-client/` integration + a demonstration end-to-end flow · **Primary chapter:** 05

## Objective

Design and prototype a **data privacy vault + tokenisation integration** for one AI system, covering training, retrieval, and inference flows. Produce the vault policy (who can detokenise what, for which purpose), the token-shape decision per field, the call-site changes at each integration point, and an end-to-end demo that shows the raw PII never crosses the model-layer boundary.

## Prerequisites

- Chapter 05 (data privacy vaults and tokenisation) read.
- Chapter 03 propagation contract (ex. 02) at hand — the vault's purpose-binding uses the same purpose enum.
- The DFD from mod-102 exercise 01 so the trust boundaries the vault sits behind are already drawn.
- Optional: access to a managed vault (Skyflow / VGS / Evervault) or a self-hostable primitive (HashiCorp Vault transit / FPE). You can prototype with a local stub that simulates the vault API.

## Problem statement

Produce three deliverables.

### Deliverable A — `vault-design.md`

A design document covering:

- **Architecture** — where the vault sits in the DFD; which services call it; which services are explicitly *prevented* from calling it (e.g. training service account); the KMS underneath; the audit sink; the network boundary.
- **Field-level tokenisation decisions** — a table listing each sensitive field (from exercise 01's datasheet), the chosen token shape (opaque / deterministic-opaque / FPE / masked), the rationale, and the residual-risk note.
- **Policy** — the vault policy in the shape of chapter 05's example YAML, with at least four roles (ingest, training service account, retrieval service account, support agent, DSR processor, analytics) and purpose + condition clauses per role.
- **Integration points** — one subsection each for training, retrieval, inference, logging. Each names the current call-site, the change required, and the expected invariant *after* the change.
- **Failure-mode register** — the five chapter-05 failure modes (detokenise egress, token-leak via joins, FPE leakage, plaintext fallthrough, provider-side retention) with your project-specific mitigation and residual-risk note for each.
- **Datasheet entry** — the `tokenisation:` subsection that chapter 05 specifies, matching your design.

### Deliverable B — `vault-client/` prototype

A working (or detailed-pseudocode) implementation of the vault client used by the four integration points. Minimum:

- `tokenise(field_name, raw_value, scheme) -> token` — stores raw in vault, returns token.
- `detokenise(tokens, purpose, requested_format) -> values_or_rejection` — enforces policy; records audit.
- `tokenise_text(text, pii_spans, scheme) -> text_with_tokens` — replaces PII spans inline with tokens.
- A small policy engine that evaluates the Deliverable-A YAML against a call.

Implementation may be real (against a managed vault / HashiCorp Vault transit) or stub (an in-memory map with policy enforcement). The policy enforcement must be real either way.

### Deliverable C — end-to-end demonstration

A small driver script that:

1. Ingests one or two seeded records with sensitive fields.
2. Writes the tokenised form to a demo "warehouse" (SQLite / pandas / in-memory).
3. Writes tokenised chunks + metadata to a demo "vector DB" (local Qdrant / Weaviate / Chroma / pandas).
4. Assembles a prompt from a retrieved chunk + a user input; passes to a stubbed "model provider" that logs what it received.
5. Receives a tokenised completion; the egress layer detokenises for the authorised consumer (the end-user) and logs the tokenised form to the "log sink".
6. Attempts a detokenise call from the training service account — the policy rejects it.
7. Demonstrates a DSR erasure: deleting the subject's vault entries invalidates all downstream tokens.

Capture a short transcript (stdout / README-like walkthrough) that shows each step succeeded or failed as expected.

## Starter guidance

- **Keep the vault single-purpose.** If you are reaching into the vault for something that is not a sensitive-field tokenise or detokenise, that logic belongs elsewhere.
- **Use deterministic tokens where you need joins; opaque otherwise.** Document the trade-off.
- **FPE is tempting and often wrong.** Length-leakage on ten-digit phone numbers is often fine; length-leakage on zip codes is often a re-id vector. Reach for FPE only when the downstream validator requires format preservation.
- **Policy conditions are the hard part.** Write at least one policy that depends on request context (e.g. the support agent's `ticket_id` must match the token's subject's `ticket_id`).
- **Instrument the audit log from day one.** The audit is the artifact the DPIA will quote. Record caller, purpose, fields, format, outcome, record IDs.
- **Make the training-service-account path explicitly deny.** A rejection is the point; do not silently return a masked value when the path should not exist at all.

## Acceptance criteria

- [ ] The design document covers all four integration points (training, retrieval, inference, logging) with concrete before / after call-site descriptions.
- [ ] At least four distinct token shapes appear in the field-level table, with rationale per field.
- [ ] Policy has at least four roles; at least one role has a condition clause that goes beyond field + purpose.
- [ ] Prototype supports `tokenise`, `detokenise`, `tokenise_text`, and policy enforcement.
- [ ] End-to-end demo runs the seven steps; the rejected training-service detokenise call is visible in the transcript.
- [ ] DSR erasure step demonstrates that deleting the vault entry invalidates downstream tokens.
- [ ] Audit log captures every detokenise attempt with enough context to reconstruct the call.
- [ ] Failure-mode register lists the five chapter-05 failure modes with project-specific mitigations.
- [ ] Datasheet `tokenisation:` entry is populated and is consistent with the design.

## Stretch goals

- Add **rate-limiting + anomaly detection** on the detokenise endpoint; demonstrate it catching a bulk-detokenise call (e.g. >100 records in 60s).
- Add **cross-sink key separation**: two sinks that each tokenise the same field but with different keys, so cross-sink linkage requires both keys. Document the cost (e.g. in datasheet residual-risk).
- Integrate with a **confidential-compute inference** path (mod-108 depth): the detokenise step happens inside the attested enclave so the model-provider side never sees the raw value even transiently. Record the trust-boundary change.
- Produce a **side-by-side cost sketch** — self-hosted vault vs. managed (Skyflow / VGS). Record the build-vs-buy decision in the datasheet.
- Extend the prototype to support an **SDR / unlearning-ready** vault — deleting a vault entry emits an event that the retrieval re-index job and the fine-tune unlearning plan subscribe to.

## What this lands

- The vault design is a concrete control that appears in the DPIA and the datasheet.
- The prototype is a working reference the peer `training-pipeline-engineer` / `rag-engineer` / `llm-application-developer` can adapt into their production system.
- The audit log + policy are the Art. 32 "measures" the regulator will ask about on the vault specifically.

Version the design. Keep the policy in the same repo as the service that enforces it.
