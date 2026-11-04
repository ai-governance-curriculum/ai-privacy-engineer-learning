# The DSR Engineering Surface for an AI System — One Runbook Per Right

A data-subject-rights request is a legal obligation that arrives at your engineering system as a database query, a corpus edit, a log-purge job, and sometimes a model retrain. The GDPR gives the data subject six operative rights (Articles 15–21); the UK GDPR is identical in substance; the CCPA / CPRA, LGPD, and the growing US state-law bench give materially-equivalent rights under different names. The privacy engineer's job is not to argue those rights — that is the `ai-governance-analyst` (level 15) and the DPO's job — but to make sure that when a request lands, the engineering system can *answer it correctly*, *produce evidence*, and *do so within the regulatory clock*.

This chapter is the map. For each of the six DSR rights, it names:

- What the right asks of a traditional database-backed system (the familiar case).
- What the same right asks when the system is an AI system — with training data, derivative embeddings, RAG indices, model logs, and inference-time caches on top of the primary database.
- The engineering runbook — the specific steps your team runs when the request arrives.
- The hand-off boundary with `ai-governance-analyst` upstream (request intake, legal interpretation) and with the rest of mod-110's chapters downstream (02–04 for the unlearning mechanics, 05 for the decision tree, 06 for RAG / agent specifics, 07 for the audit trail).

The six rights, in the order they tend to arrive in practice:

1. **Right of access** (Art. 15) — "what data do you hold about me, where, and what have you inferred?"
2. **Right of rectification** (Art. 16) — "correct an inaccurate fact about me and propagate the correction."
3. **Right of erasure** (Art. 17) — "delete my personal data, including from derivative products."
4. **Right to restrict processing** (Art. 18) — "stop processing while a dispute is resolved."
5. **Right to data portability** (Art. 20) — "return my data in a structured, machine-readable form."
6. **Right to object** (Art. 21) — "stop processing for marketing / profiling / specified purposes."

Primary references — read them before building the runbooks:

- Regulation (EU) 2016/679, *General Data Protection Regulation*, Articles 12–22 (the DSR chapter). [eur-lex.europa.eu/eli/reg/2016/679/oj](https://eur-lex.europa.eu/eli/reg/2016/679/oj).
- European Data Protection Board — *Guidelines 01/2022 on data subject rights – Right of access*, v2.0 (2023). [edpb.europa.eu/system/files/2023-04/edpb_guidelines_202201_data_subject_rights_access_v2_en.pdf](https://www.edpb.europa.eu/system/files/2023-04/edpb_guidelines_202201_data_subject_rights_access_v2_en.pdf).
- UK ICO — *Rights of individuals* guidance. [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/).
- UK ICO — *Guidance on AI and data protection*, sections on individual rights and AI. [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/).
- California Privacy Protection Agency — *CCPA regulations*. [cppa.ca.gov/regulations/](https://cppa.ca.gov/regulations/).
- NIST — *SP 800-53 Rev. 5*, IP (Individual Participation) and PT (PII Processing and Transparency) control families. [doi.org/10.6028/NIST.SP.800-53r5](https://doi.org/10.6028/NIST.SP.800-53r5).
- NIST — *Privacy Framework, v1.0* (2020). [nist.gov/privacy-framework](https://www.nist.gov/privacy-framework).

## Why this is harder for an AI system than for a database

A classical system has one place a subject's data lives: a row in a table, with foreign keys to related rows. Erasure is `DELETE FROM users WHERE id = $1 CASCADE`. Access is `SELECT ... WHERE user_id = $1`. The engineering work is to enumerate the tables that reference the subject.

An AI system has at least six places the same subject may be present, and the mapping between them is not referential:

- **The primary operational database** — the familiar case, a row keyed to the subject.
- **The training corpus** — a dataset snapshot, possibly versioned and immutable by design. The subject may appear as an exact row, as a derived feature, or as free text that happens to mention them.
- **The model weights** — the training algorithm's output. The subject's influence on the weights is diffuse, not addressable. Removing it requires an unlearning method (chapters 02–04) or a full retrain.
- **The retrieval corpus (RAG index / vector store)** — a per-chunk index with embeddings. The subject may be in the chunk text, or inferable from the embedding neighbourhood.
- **The prompt / completion logs** — operational telemetry that may contain the subject's prompts or model outputs about them. Retention policy and purge discipline are a first-order DSR concern.
- **Derivative products** — fine-tuned models, distilled models, synthetic datasets, labelled eval sets, cached feature stores, analytics dashboards. Each is a downstream artefact that captured something from an upstream corpus.

The engineering runbook for any one right is the enumeration of *which of these six* the right reaches into, and the specific action for each. The regulatory clock (one month for access, erasure, and most others under GDPR Art. 12(3); extensible by two months for complex cases) does not stop because your training corpus is a Parquet file in cold storage.

## The system of record — before you build any runbook

A DSR runbook begins with a map. You cannot answer "what do you hold about me" if you do not know what you hold at all. Before running any of the per-right runbooks below, build and maintain:

- **A data inventory** that lists every *location* personal data may live. Six categories above, with concrete instances — `postgresql://prod-main/users`, `s3://training-snapshots/2025-Q4/`, `qdrant://rag-prod/corpus`, `s3://logs/prompt-completions/`, and so on. Each entry has an owner, a retention policy, and the subject-identifier shape (user ID, email, device ID, tenant-scoped user ID). mod-103 is the primary home of this inventory discipline; mod-110 is its most demanding consumer.
- **A linkage table** that maps a subject identifier in one store to the subject identifier in another. The `user_id` in the primary DB is not the same string as the `author_id` in a training corpus row; the DSR team needs the join path.
- **A record of processing activity (RoPA)** — the GDPR Art. 30 artefact the DPO maintains. For each processing activity, name the AI system (and its training / retrieval / log corpora) that touches it.

Without these, every DSR runbook devolves into a scavenger hunt, and the one-month clock becomes a fire drill. The discipline is `ai-governance-analyst`-adjacent; your contribution as the privacy engineer is to make sure every corpus you ship is in the inventory *before* the first DSR arrives.

## Runbook — Right of access (GDPR Art. 15)

The subject asks: *what personal data do you process about me, for what purposes, how long will you keep it, and who do you share it with*? The response is a copy of the data, plus the processing metadata.

The engineering actions, per store:

- **Operational database.** `SELECT ... WHERE subject_id = $1` across the tables the inventory lists. Export as JSON / CSV. The familiar case.
- **Training corpus.** Search the (versioned) training snapshot for the subject. The expected shape of the hit depends on how the subject got in: an explicit row keyed to them (easy — report the row and the training-version the row appeared in), a free-text mention (harder — a grep over the corpus; see the `ai-governance-analyst` playbook on "proximate match" for the standard for a hit), a derived feature (hardest — the raw record is recoverable only via a reverse join from the feature store). Report what is in the corpus and under what training snapshot.
- **Model weights.** You do not export weights; you report "the model was trained on training-snapshot v2025-Q4, which contained the following records keyed to you." The right of access over inferences (Art. 15(1)(h) in the automated-decision context) is the harder question: if your model *inferred* something about the subject — a credit score, a propensity, a cluster assignment — the inference is personal data and is in scope. If the inference is stored (in a feature store, in a decision log), export it. If the inference is produced on the fly for each decision, the Art. 15 response is a *description of the logic involved* and the *consequences* of the processing (Art. 15(1)(h)) rather than a dump of every past inference.
- **Retrieval corpus.** `SELECT chunk, source_id FROM chunks WHERE text ILIKE '%<subject>%' OR source_subject_id = $1` — the index is still a database. Export the chunks that contain or are attributable to the subject.
- **Prompt / completion logs.** Logs often key on a `user_id` for the *caller*, not for the *subject-of-the-prompt*. A caller may have asked about a third party; the third-party subject's DSR needs a full-text search across the log, not a join. Export the log entries.
- **Derivative products.** For each derivative that was produced from a corpus the subject appeared in, either (a) report the derivative's existence and the upstream corpus the subject was in, or (b) if the derivative is itself queryable per subject (a fine-tune's eval log), export the subject's slice.

The deliverable is a bundle — JSON-structured where the shape allows, a per-store manifest naming the store, the search predicate, the version, the retention policy, and the record count. `ai-governance-analyst` reviews it for completeness and translates it into the language the subject sees. The audit trail (chapter 07) records the bundle's manifest, hash, and send timestamp.

### The "what you inferred about me" slice

A right-of-access response for an AI system that is incomplete without the inferences slice is a response that will be challenged. The regulator and the subject care about *what the model thinks of me*, not only *what the model was told about me*. Two engineering obligations follow:

- **Inference logs are personal data.** If the model infers a score, class, or embedding about an identifiable subject and the inference is retained for any period, the retention is processing that falls under Art. 15. The log needs a `subject_id` index.
- **On-the-fly inferences require a logic description.** If the inference is produced for every decision and is not retained, Art. 15(1)(h) requires "meaningful information about the logic involved, as well as the significance and the envisaged consequences of such processing for the data subject." This is a documentation obligation, not an engineering one — but the engineer owns the input: feature list, model family, version, inputs the model saw. Produce it from the model card and the feature-store manifest.

The boundary with `ai-governance-analyst` is sharp here: the *text* of the logic description is theirs; the *content* is yours.

## Runbook — Right of rectification (GDPR Art. 16)

The subject asserts a fact about them is inaccurate and asks for correction. The engineering actions:

- **Operational database.** Update the row. Familiar.
- **Training corpus.** The training snapshot is usually immutable by design — it is the ground truth for a training run and should not change behind the model's back. The engineering response is one of:
  - *Correct the next snapshot.* Record the correction in the primary system, let the next training-corpus freeze inherit it, retrain on the next scheduled cycle. The current model is unchanged.
  - *Rectify in place with an annotation.* Keep the original row in the snapshot (for reproducibility of the current model) but add a correction overlay the eval harness reads. This is uncommon; most teams prefer snapshot immutability and a correction-in-next-snapshot rhythm.
  - *Treat rectification as erasure + re-insertion.* Remove the inaccurate row and insert the corrected one. The audit trail must record both actions.
- **Model weights.** A rectification-against-the-weights is either (a) acceptable to defer to the next full retrain, (b) a reason to apply unlearning on the incorrect fact followed by re-training on the correct one, or (c) ignored because the incorrect fact's influence on the weights is small and the correction will propagate at the next refresh. Chapter 05's decision tree walks the choice.
- **Retrieval corpus.** Replace the chunk whose text was inaccurate. The embedding is re-generated; the metadata's `source_updated_at` is bumped. This is the easiest of the four and the one most commonly overlooked.
- **Prompt / completion logs.** You do not rectify historical log content — it is a record of what happened. You may rectify an attached subject-fact field (if your logs join to a user profile), but the historical prompt / completion is kept as-is for operational-integrity reasons. The DPO signs off on this interpretation.
- **Derivative products.** Same logic as the model: either defer to the next refresh or apply unlearning + re-training. If the derivative is a static artefact (a published fine-tune) that the public has already pulled, the rectification is practically limited to future versions.

Rectification is the right that stresses the *correction propagation* wiring. If your pipeline cannot answer "a correction entered in the primary DB on day D will be reflected in the training snapshot version N+1, the retrieval corpus by day D+1, and the next model refresh in two weeks," you are not ready to receive the right.

## Runbook — Right of erasure (GDPR Art. 17)

The subject asks for removal. This is the right chapters 02–06 spend most of their time on; here we name the top-level runbook.

- **Operational database.** `DELETE FROM users WHERE id = $1 CASCADE`. Soft-delete markers are acceptable only if the row is *provably* excluded from every downstream process; most teams prefer a hard delete with a tombstone row that records the deletion fact.
- **Training corpus.** Add the subject to the corpus's erasure list. The next snapshot is produced minus the subject's rows. The *current* snapshot is either (a) left alone because the corresponding model will be retired at the next refresh, (b) *re-published* minus the subject if downstream consumers of the snapshot need it, or (c) replaced with a corrected version and old versions garbage-collected. Which of the three depends on how the snapshot is consumed; the runbook names a choice per snapshot.
- **Model weights.** This is the chapter-02-to-05 territory. Options:
  - *Full retrain* on the corrected corpus. Correct, expensive.
  - *SISA-style exact unlearning* (chapter 02). Correct, cheaper if the training pipeline was SISA-shaped from the start.
  - *Approximate unlearning* (chapter 03 for classical models, chapter 04 for LLMs). Cheaper, correctness bounded by a verification test.
  - *No action* — defensible when the chapter-05 decision tree says the subject's data was not materially in the model's weights (e.g. small, deduplicated, dropped by a filter before training).
- **Retrieval corpus.** `DELETE FROM chunks WHERE source_subject_id = $1 OR text ILIKE '%<subject>%'`. Rebuild the HNSW / IVF index if the delete rate is high; incremental delete is fine if low. The embedding store's `delete-by-filter` API is the primitive. The erasure must propagate to any cache / replica (chapter 06).
- **Prompt / completion logs.** The subject's prompts and the completions mentioning them are deleted or purged according to the log's retention policy; the DPO signs off on whether a log retention shorter than the DSR clock can be the mechanism (often yes) vs. an explicit purge job (often required for anything older than that retention). Chapter 06 covers the log-purge pattern.
- **Derivative products.** For each derivative produced from a corpus that contained the subject: apply the same decision tree as to the model.

Erasure is the right with the longest tail — a complete erasure runbook enumerates *every corpus and every derivative* and names the action for each. Chapter 05 makes the per-corpus decision tree explicit; chapter 07 packages the evidence.

## Runbook — Right to restrict processing (GDPR Art. 18)

The subject asks to *stop processing* their data while a correction or objection is disputed. Engineering actions:

- **Operational database.** A `processing_restricted_until` timestamp on the subject's row. Application code gates reads / writes on it. The row is retained; it is just not acted on.
- **Training corpus.** Exclude the subject from the next snapshot while the restriction is in place. Keep the row in the "restricted" partition that the snapshot build filters out.
- **Model weights.** Usually no action — the current model was trained before the restriction and is not re-training on live data. If the model is re-trained during the restriction window, the restricted rows are excluded from that training run.
- **Retrieval corpus.** Soft-delete the subject's chunks (set `active = false`); the retriever's filter excludes them. The soft-delete is reversible when the restriction lifts, which is why restriction is different from erasure.
- **Prompt / completion logs.** Pause any analytics or training-data-extraction that would operate on the subject's historical logs during the restriction.
- **Derivative products.** Each derivative's refresh job respects the restriction: the next re-build excludes the subject.

Restriction is often the *interim* state between a request and its resolution. The runbook is similar to erasure but *reversible* and *scoped to a time window*. The audit trail records the restriction window and the resolution (restored, escalated to erasure, or escalated to rectification).

## Runbook — Right to data portability (GDPR Art. 20)

The subject asks for a copy of their data "in a structured, commonly-used and machine-readable format" (Art. 20(1)) so they can take it to another controller. Portability is narrower than access — it covers data the subject *provided* and data *generated from the subject's use of the service*; it does not cover inferred data.

- **Operational database.** Export the subject's rows in JSON / CSV / Parquet.
- **Training corpus.** Not usually in scope — the subject did not "provide" the training corpus in the Art. 20 sense; they provided the primary-DB rows that fed into it. The portability bundle is the primary-DB export.
- **Model weights.** Not in scope.
- **Retrieval corpus.** If the subject provided documents that were chunked into the retrieval corpus, the *source documents* are in scope and are already in the primary DB. The chunks are a derivative.
- **Prompt / completion logs.** The subject's own prompts are in scope. The model's completions about them are inferred data and are *not* portable in the Art. 20 sense (though they remain in the Art. 15 bundle).
- **Derivative products.** Not in scope.

Portability is the right that most cleanly separates "provided / observed" data (in scope) from "inferred" data (out of scope). Your export format matters; the ICO and EDPB expect JSON / CSV / XML, not a proprietary binary.

## Runbook — Right to object (GDPR Art. 21)

The subject objects to processing for one or more purposes — marketing (Art. 21(2)), direct profiling, research (Art. 21(6)), or any processing justified under legitimate-interests (Art. 21(1)). Objection is a *purpose-specific* request, not a wholesale erasure.

- **Operational database.** Flag the subject as objecting to the specified purpose. Each processing pipeline checks the flag.
- **Training corpus.** If the AI system's purpose is one the subject objects to (a marketing propensity model, a profiling model), exclude the subject from the next training snapshot for that model. If the AI system serves multiple purposes, the exclusion is scoped to the models trained for the objected purpose.
- **Model weights.** Usually no live action for the current model unless the model is itself the processing purpose — in which case the decision tree in chapter 05 applies.
- **Retrieval corpus.** Scope-dependent: if the retrieval is used for the objected purpose, exclude the subject's chunks. If retrieval is used for multiple purposes (common), attach a per-purpose exclusion flag to the chunk metadata.
- **Prompt / completion logs.** Pause the subject's historical logs from being fed into training / analytics for the objected purpose.
- **Derivative products.** Scope the exclusion to derivatives that serve the objected purpose.

The engineering challenge in objection is *purpose-awareness*. Your pipeline must know *why* each processing operation is happening and must have a per-purpose exclusion filter. If your pipeline is organised by data flow (ingest → train → serve) rather than by purpose, you will have to layer a purpose tag onto the subject record and check it at each pipeline stage.

## A note on automated decision-making (Art. 22)

Art. 22 is not strictly a DSR right in the Art. 15–21 enumeration — it is a *restriction* on a specific class of processing. But it generates DSR-shaped obligations: the subject has the right to human intervention, to express a view, to contest the decision. The engineering artefacts the privacy engineer owns here:

- A *human-review endpoint* for the decision in question.
- A *decision-record* that captures the inputs, the model version, and the output — enough for a reviewer to reproduce and contest.
- The *logic description* that Art. 15(1)(h) also mandates.

Art. 22 is where the DPIA (mod-111) and the system card (mod-109 ch. 06) land the engineering obligations; the DSR runbook's contribution is to make sure a request to contest a decision can be answered against a reproducible record.

## The six-corpus x six-rights grid

Pin this grid on the wall. It is the start of every DSR triage call.

| Store                   | Access         | Rectification | Erasure         | Restrict      | Portability | Object        |
|-------------------------|----------------|----------------|------------------|---------------|-------------|---------------|
| Operational DB          | SELECT         | UPDATE         | DELETE           | flag          | export       | flag          |
| Training corpus         | version search | next-snapshot  | next-snapshot + decision tree | exclude from next | out of scope | purpose-scoped exclude |
| Model weights           | describe logic + inference log | next-refresh or unlearn | retrain / SISA / unlearn / none | usually no-op | out of scope | depends on purpose |
| Retrieval corpus        | SELECT chunks  | replace chunk  | delete chunks + reindex | soft-delete | export chunks if subject-provided | per-purpose flag |
| Prompt / completion logs| full-text search | usually immutable | purge job | pause analytics | subject prompts in scope, completions out | pause analytics |
| Derivative products     | manifest        | next-refresh   | decision tree per derivative | next-refresh exclude | out of scope | purpose-scoped |

The grid is not exhaustive; it is a scaffold. For any specific system, the privacy engineer walks it cell by cell with `ai-governance-analyst` and produces the system-specific runbook.

## The clock

The GDPR timeline is one month from receipt (Art. 12(3)), extensible by two months for "the complexity and number of the requests" with notice to the subject. The engineering consequence:

- **Access** (the long-tail one): the data-inventory map must be *up to date*, not reconstructed on receipt. If reconstructing the map takes three weeks, the access-response assembly has one week left.
- **Erasure**: the full retrain decision-tree option (chapter 05) is three-to-many weeks for a production LLM. If erasure is likely to require a retrain, you need a *clock-management* answer with `ai-governance-analyst` — either negotiate the extension early, or move the retrain cadence to fit within the DSR SLA.
- **Rectification**: next-snapshot cadence must be < one month for the "deferred" option to be defensible. If your snapshots refresh quarterly, rectification against the training corpus is a special-case with its own SLA.

The audit trail (chapter 07) records receipt time, action time, and completion time against each SLA; a request that misses the clock is a reportable incident to the supervisory authority under Art. 33 if it rises to a data-protection risk.

## The hand-off to `ai-governance-analyst` (level 15)

The DSR-receiving role is `ai-governance-analyst`. They receive the request through the privacy inbox, verify the subject's identity, interpret the request against the operative legal framework, and open the engineering ticket. The engineer's inputs to the analyst:

- The *data inventory* and the *linkage table* — so the analyst can enumerate stores to search without asking.
- The *per-store runbooks* — so the analyst can scope the engineering work and estimate the clock.
- The *past DSR evidence bundles* — so the analyst's response template is consistent request-to-request.

The engineer's outputs from the analyst, per request:

- An *authoritative scope* — what stores, what subject identifiers, what purposes.
- An *acceptance criterion* — what the response must contain (an evidence bundle, a confirmation, a redacted-text document).
- An *SLA date* — when the response must be sent, after the one-month clock has started and after any extension has been agreed.

Chapter 07 is the bidirectional artefact that makes this hand-off auditable. The chapter also names the point at which the DPO is a mandatory signer: any erasure that requires a retrain (chapter 05) is a DPO-sign-off action.

## Summary — what you can do after this chapter

- Enumerate the six DSR rights (Art. 15–21) and name the AI-system twist on each.
- Produce the six-corpus x six-rights grid for *your* system, filling the cells that are specific to your data inventory.
- Draft the per-store runbook for access, rectification, erasure, restriction, portability, and objection.
- Name the hand-off boundary to `ai-governance-analyst` (intake, interpretation, SLA) and name the engineering inputs / outputs on each side.
- Call out the clock implications — where the one-month SLA forces an engineering practice (deduplication, snapshot cadence, retrain cadence) rather than an ad-hoc response.

Chapter 02 picks up the heaviest runbook item — erasure against the model's weights — and walks through SISA-style exact unlearning as the first of three unlearning techniques the module teaches.
