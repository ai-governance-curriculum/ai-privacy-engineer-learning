# Exercise 02: Consent / Legal-Basis Propagation Contract

**Estimated effort:** 3 hours · **Output:** `propagation-contract.md` + `schema/` directory with SQL and vector-DB payload schemas + `job-manifest.yaml` + two CI test stubs · **Primary chapter:** 03

## Objective

Design and specify a **consent / legal-basis propagation contract** that carries record-level privacy context from capture to training to retrieval to inference to logging, surviving ETL, embedding, and inference logging. Produce a schema, a job-manifest matcher, and two CI tests that would catch the two most common propagation bugs.

## Prerequisites

- Chapter 03 (consent and legal-basis propagation) read.
- Chapter 02 (datasheets) read — the datasheet summarises; this exercise is what makes the summary *testable*.
- Exercise 01 of this module completed, or a representative training corpus you can propagate against.
- Working knowledge of SQL (BigQuery policy tags / Snowflake tags / Unity Catalog preferred) and a vector-DB metadata model (Qdrant / Weaviate / Pinecone / Milvus / pgvector).

## Problem statement

Produce four deliverables.

### Deliverable A — `propagation-contract.md`

A Markdown document that specifies, for your chosen AI system (the mod-102 default scenario is fine):

- The **record-level propagation fields** (chapter 03 §"The record-level fields that must travel") with per-field data type, allowed values, defaulting rules, and the required-vs-optional posture per data class.
- Where the propagation fields live in each of five carriers: **tabular**, **document**, **chunk**, **embedding**, **prompt / log row**.
- A statement for each of **capture**, **ETL**, **job-manifest matcher**, **egress** enforcement points describing what is enforced and who is accountable.
- A section on **revocation and objection** — the propagation signal for `consent_withdrawn_at` / `objection_upheld` and the derived-artifact reach (training snapshot, retrieval index, vector embedding, cache, log).
- A section on the **four weak points** (scraped data, UGC in prompts, agent-tool arguments, provider-side caches) with your project-specific position on each.

### Deliverable B — `schema/sql/tickets_raw.sql`

A CREATE TABLE statement for the raw corpus with every propagation field (`lawful_basis`, `purpose`, `consent_token`, `collection_timestamp`, `source`, `jurisdiction`, `data_classes`, `retention_deadline`, `controller`, `processor`, `consent_withdrawn_at`, `objection_status`, `lia_id` where applicable). Include a representative set of application columns alongside.

And `schema/sql/training_view.sql` — a view / materialised view that filters the raw table to the fine-tune-safe subset, with the chapter 03 worked-example filter clauses (basis / objection / retention / class / jurisdiction / revocation). The view is the matcher.

### Deliverable C — `schema/vector-db/chunk-payload.json`

A JSON Schema for the per-chunk metadata that will live in the vector DB alongside the embedding. Include every propagation field plus retrieval-specific fields (`doc_id`, `tenant_id`, `encoder`, `chunk_version`). This is the schema the rag-engineer peer implements against.

### Deliverable D — `job-manifest.yaml`

A job manifest for one specific downstream job — a fine-tune, a retrieval re-index, an analytics aggregate, or an eval-run — declaring `purpose`, `basis_allowed`, `data_classes_allowed`, `data_classes_denied`, `jurisdiction_allowed`, `retention_horizon`, and any job-specific condition clauses. This is the matcher's input.

### Deliverable E — CI test stubs

Two stub test files (pseudocode or your language of choice) that would fail CI if a developer breaks propagation:

- `test_propagation_etl.py` — asserts that a seeded record with `objection_status = 'objection_upheld'` is excluded from the training view, from the retrieval index after re-ingest, and from the egress to the model provider.
- `test_propagation_logger.py` — asserts that a prompt-completion log row for a record whose `consent_withdrawn_at` has been set is either deleted or redacted within the documented SLA.

Full test implementation is not required; the stub must name fixtures, assertions, and the oracle each test checks.

## Starter guidance

- **Make the propagation fields NOT NULL.** A record without basis / purpose at ingest is the kind of silent defect this contract is designed to prevent. Reject at the capture gateway.
- **Use enums, not strings.** `lawful_basis` as a VARCHAR accepts typos; as an enum it accepts six values. Downstream CASE statements become enumerable.
- **Carry the propagation fields through derived views explicitly.** A view that joins `tickets_raw` to `user_consents` to compute the effective basis should re-materialise the fields, not assume the reader will re-derive.
- **Deterministic consent-tokens let you do set-level analysis without re-identification.** Use them, note the trade-off in the datasheet (chapter 05 cross-sink linkage).
- **Write the egress-side explicitly.** Model-provider calls are the trust-boundary crossing most pipelines forget. Either include the propagation in the request metadata (Pattern A) or document the Pattern B tokens-only substitution for exercise 04.
- **Keep the contract short enough to be read.** A 30-page contract is a 0-page contract. Target 5 pages of Markdown that a peer engineer could implement from.

## Acceptance criteria

- [ ] Every field in chapter 03 §"core propagation fields" is in the schema with type and allowed values.
- [ ] Legitimate-interests-specific fields (`lia_id`, `objection_status`) appear where the basis permits.
- [ ] Consent-specific fields (`consent_version`, `consent_scope`, `consent_withdrawn_at`) appear where the basis permits.
- [ ] The tabular carrier, the document / chunk carrier, and the vector-DB payload carrier each have a schema.
- [ ] The job manifest + the training view together form a matcher that explicitly rejects records for at least three distinct reasons (basis, class, retention, revocation, jurisdiction — pick three).
- [ ] The revocation / objection propagation path is drawn end-to-end: raw → view → index → log → cache.
- [ ] The four weak points (scraped data, UGC in prompts, agent-tool args, provider-side caches) each have a one-paragraph position.
- [ ] The two CI test stubs name fixtures, assertions, and oracles — a developer reading them would know what the test means to catch.
- [ ] A reader can implement the contract on a new dataset without a clarifying call.

## Stretch goals

- Implement one of the two CI tests against a real local Postgres + a local vector DB (Qdrant / Weaviate). Report whether it catches a deliberate regression.
- Author the Pattern-A vs. Pattern-B vs. Pattern-C decision table for the trust-boundary crossings in your system (chapter 03 §"Propagation across trust boundaries").
- Prototype OpenLineage emission from your ETL job so column-level propagation is machine-queryable.
- Produce a propagation-status dashboard mockup: records-at-rest with no basis; records with pending objection; exclusions-per-job over time.

## What this lands

- The propagation contract is the engineering substrate for every later control: minimisation (ex. 03) matches against it, vault + tokenisation (ex. 04) runs where it is enforced, synth audit (ex. 05) consumes it, ML-BOM (ex. 06) references the version.
- The job manifest is reusable — every downstream job that reads personal data gets one.
- The CI tests are durable — they outlive the current implementation and catch regressions on later changes.

Keep the contract in version control next to the dataset manifest. Date the version. Review on dataset regeneration.
