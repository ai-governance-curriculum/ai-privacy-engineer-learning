# Four Kinds of Minimisation — Collection, Processing, Retention, Output

"Minimisation" is one GDPR word that covers four distinct engineering decisions. Collapsing them conceals the fact that each has its own checkpoint, its own owner, and its own failure mode. This chapter decomposes minimisation into four engineering controls — **collection**, **processing**, **retention**, and **output** — and shows how each is enforced in an AI pipeline.

Anchor obligations:

- GDPR Art. 5(1)(b) — purpose limitation.
- GDPR Art. 5(1)(c) — data minimisation ("adequate, relevant and limited to what is necessary").
- GDPR Art. 5(1)(e) — storage limitation ("kept in a form which permits identification … for no longer than is necessary").
- GDPR Art. 25(2) — data protection by default ("only personal data which are necessary for each specific purpose").
- EU AI Act Art. 10(3) — training-set relevance for high-risk systems.
- CCPA / CPRA §1798.100(c) — collection, use, retention, sharing shall be reasonably necessary and proportionate.
- HIPAA §164.502(b) — minimum necessary standard.

Read as sentences they overlap. In engineering they are different checkpoints.

## Why this matters

The default state of an AI pipeline is to *keep everything, forever, in the widest possible context*. The reasons are banal — training-set researchers fear sampling bias; eval teams want to test against anything production can throw at them; observability engineers want to debug any prompt; product managers want data for the quarterly deck. Each individual request is reasonable. The aggregate is a pipeline that violates every one of the obligations above.

The minimisation controls are the counterweight. They are the engineering mechanism that encodes "no" as the default and "yes with a justification" as the requestable path.

## Control 1 — Collection minimisation

*The field you do not collect is the one you never have to secure, audit, or delete.*

Collection minimisation is enforced at the data-capture boundary — the form, the event producer, the connector, the ingest API. It is the cheapest control because its substrate is the schema.

### Checkpoints

- **Explicit schema, deny-by-default.** The ingest schema whitelists the fields that may be captured for a given purpose. Fields not on the whitelist are dropped at ingest. A "collect everything just in case" catch-all column is a Collection failure.
- **Purpose-bound schema.** Different purposes get different ingest schemas. An onboarding form that captures name + email for account creation is a different schema from an opt-in research form that captures demographics; they do not share a table.
- **No free-text catch-alls for personal data.** A `notes` or `description` column attracts PII. If a free-text field is necessary, apply default-on ingest redaction (mod-104) and/or route through the vault (chapter 05).
- **Conditional collection.** Fields that are only needed for a minority of flows (billing address when the user chose shipping, not pickup) are collected conditionally. The schema `NOT NULL` rules encode which combinations are valid.
- **Upstream propagation.** For scraped / third-party data, the upstream connector normalises and *drops* fields that are not needed. Do not load the raw dump into the warehouse "for later".

### Where it breaks

- Event-schema drift — a new upstream pushes a new field; the warehouse autoloader absorbs it; nobody notices until a regulator asks.
- Log-line catch-alls — the structured log of a request body silently captures the full prompt even when the schema was supposed to exclude it.
- Analytics shims — a product-analytics SDK sends arbitrary event payloads to a third-party SaaS. See mod-109 for the retrieval / agent counterpart.

### Enforcement pattern

- Schema CI — schema changes in the warehouse require privacy-review approval; new fields carry a data-class tag on introduction (chapter 03).
- Autoloader lint — allowlist on autoloader columns; dropped columns reported to a dashboard.
- Capture-side Presidio pass on free-text fields; redacted result stored instead of raw.

## Control 2 — Processing minimisation

*The minimum data needed for the specific task at hand, and no more.*

Processing minimisation is enforced at *every downstream job* that reads personal data. It is the hardest of the four because the surface is wide.

### Checkpoints

- **Projection, not SELECT \*.** Training jobs, retrieval jobs, evaluation jobs read only the columns they declare in their manifest (chapter 03). A job that reads an un-declared column fails CI.
- **Per-column ACL on sensitive classes.** Roles that authorise access to `special_category` or `phi` columns are separate from roles that authorise generic read. A training-job service account that holds the generic role but not the sensitive-class role simply cannot see the sensitive columns.
- **Aggregate before training where possible.** For a model that only needs aggregate signal (counts, histograms), aggregate at ingest and train on the aggregate. Record-level data never reaches the trainer.
- **Redact before embedding.** Documents passing into a retrieval encoder run through PII detection + redaction (mod-104) or vault tokenisation (chapter 05) *before* the encoder. The embedding does not recoverably carry what the chunk did not.
- **Prompt minimisation at inference.** The orchestrator passes the smallest system prompt + retrieved context + user prompt that the task requires. Including the user's full profile "in case it helps" is a Processing failure.
- **Retrieval top-k discipline.** A top-10 retrieval pulling 10 chunks for a task that needs 2 is Processing surplus. Tune `k` down and measure the quality cost; justify what remains.

### Where it breaks

- Wide training-set joins that pull every column "to be safe" — the trainer memorises columns it did not need (mod-105).
- Inference logs that store the *full* prompt + context + completion when only the user prompt + decision was needed.
- Eval runs on a sample of production data that is wider than the eval needs because the sampling query was SELECT \*.
- Agent systems that pass a tool a whole document when the tool needed one field.

### Enforcement pattern

- Job manifests (chapter 03) declare required columns; a compile step diffs declared vs. actually-read and fails on surplus.
- SQL / dbt policy tests that reject `SELECT *` on tables tagged personal-data.
- Redaction / tokenisation runs are *pre-encoder*; CI test with a canary row asserts the embedding does not retain the canary.
- Logging schema is a strict subset of the inference payload; the logger drops or tokenises any field not on the subset.

## Control 3 — Retention minimisation

*Time-boxed storage. Delete when the purpose is served.*

Retention is the dimension most frequently violated because it is invisible — the pipeline keeps working whether the data is a day old or three years old. Art. 5(1)(e) and CCPA / CPRA retention disclosures require you to put a number on it; the EU AI Act Art. 10 record-keeping obligations tighten this for high-risk systems.

### Checkpoints

- **A retention deadline on every record and every sink.** No row without an explicit or inherited deadline. No sink (log bucket, vector index, cache, eval set, snapshot, backup) without a retention policy.
- **Automatic deletion jobs with evidence.** The retention job runs on a schedule and produces a log: records read, records deleted, records tombstoned, exceptions raised. A retention job that produced no deletions in the last 90 days is either correct (nothing expired) or broken (it stopped running) — the dashboard distinguishes.
- **Backups and snapshots in scope.** A deleted record that survives in a 30-day backup is still processed personal data. Either backups age out within the policy window, or the retention clock on the record starts at *backup age-out* for the purposes of DSR.
- **Per-sink retention — shortest horizon that serves the documented purpose.** A prompt-completion log kept 90 days for safety review is reasonable; the same log kept 2 years because "storage is cheap" is a storage-limitation failure.
- **Purpose-bound retention.** When the purpose ends, retention ends — a model fine-tuned once from a snapshot does not justify keeping the snapshot indefinitely.
- **Hold overrides for legal / regulatory reasons.** A legal hold suspends the retention clock on a specific set; the hold is itself documented, scoped, and expiring.

### Where it breaks

- The S3 lifecycle policy on the prompt log that was never created.
- The semantic cache / KV cache / router cache that has no retention configured because the cache "was ephemeral" (it wasn't).
- The historical model snapshots that nobody owns and nobody wants to delete because "someone might need them".
- The abuse-review queue that was supposed to be 30 days and has been 2 years since ingest from the ticket system was first wired.

### Enforcement pattern

- Every table / bucket / index / cache in the mod-102 DFD has a `retention` tag in infra-as-code. CI fails if a new sink is introduced without one.
- Per-sink retention dashboards with sample-record-age-oldest; alert fires when oldest > deadline.
- CI job periodically seeds a record with an immediate retention deadline into each sink and asserts the retention job removes it within the SLA (a *canary deletion*).
- Legal-hold registry with per-hold scope, reason, requester, and expiry.

## Control 4 — Output minimisation

*What the model should refuse to generate even when prompted.*

Output minimisation is the final checkpoint — the point at which generated content is about to leave the system. It is partially a safety-filter concern (hate / self-harm / dangerous-instructions) but there is a specifically-privacy component that falls to this role:

- The model should not emit **verbatim** memorised strings from the training corpus (mod-105 is the deeper story).
- The model should not emit **PII from a retrieval chunk that the end-user is not authorised to see** (mod-109 is the deeper story — ACL on retrieval is the primary control, output minimisation is the belt-and-braces).
- The model should not emit **inferences about sensitive attributes** (health, religion, sexual orientation) in contexts where the user has not consented to such inferences — GDPR Art. 9 implications; see EDPB Opinion 28/2024 and the ICO AI guidance.
- The model should not emit **content for a purpose the pipeline's basis does not cover** — a fine-tune originally built for support assistance should not be re-pointed at marketing content generation without a new basis + purpose entry.

### Checkpoints

- **PII-in-completion detector** (mod-104) as a post-generation filter — block / redact / quarantine completions that contain tokens classified as PII that did not originate in the user's own input.
- **Completion dedup against training set** — a hash / sliding-window check against the training-set shingles; completions that exceed a threshold are logged as potential memorisation events and surfaced to the mod-105 monitor.
- **Purpose binding on the deployment.** The deployed model's manifest declares the purposes it is permitted to serve; a request whose purpose header does not match is rejected at the gateway.
- **Sensitive-attribute inference refusal.** Where the prompt invites an Art. 9-sensitive inference about a named individual, the system either refuses, defers to a human, or applies a disclosed-limitation banner.
- **Attribution and transparency.** For retrieval-grounded completions, the system *shows the retrieval source*. If the user cannot see the source, they cannot verify the retrieval was in-scope.

### Where it breaks

- A chatbot that re-emits a quoted section verbatim from a document the current user was not authorised to read.
- A completion that contains an email address or phone number the user did not provide, extracted from a cached session that was re-prefixed.
- A support agent that infers a health condition from a ticket description and stores the inference as a customer attribute.

### Enforcement pattern

- Post-generation PII pass (mod-104); block list for dangerous-category content.
- Memorisation monitor sampling (mod-105) runs on 1% of completions.
- ACL-propagation tests on the retrieval orchestrator (mod-109).
- Attribution rendered in the UI; a user complaint path routes to the DSR queue.

## The minimisation checkpoint register

Combine the four controls into a single register. The columns mirror the Privacy-by-Design checkpoint register from mod-102 chapter 08 — on purpose, so a DPIA (mod-111) consumes them together.

| Checkpoint ID | Dimension (Collection / Processing / Retention / Output) | Spec | Enforcement (CI / lint / release-gate / runtime) | Owner | Current status | Evidence |
|---|---|---|---|---|---|---|

Statuses:

- **Enforced** — gate exists and has blocked at least one bad merge / deploy.
- **Instrumented** — gate exists but has not fired; synthesise a test case.
- **Documented** — written-down expectation with no gate yet; a known gap.
- **Not applicable** — document the reason.

A gap in the register is not a silent risk; it is a tracked risk with an owner.

## Tensions worth naming

Three tensions you will mediate rather than resolve:

- **Collection minimisation vs. representativeness (Art. 10(3)).** A narrow schema may under-represent a population and therefore violate the training-set relevance obligation. The engineering move is to collect *categorical signals* about population membership (consented demographic flags) rather than the full raw attribute, and to measure coverage separately.
- **Processing minimisation vs. researcher freedom.** Researchers prefer wide-column access for exploration. The engineering move is to give them wide access on a *synthetic* substitute (chapter 06) and narrow access on real data, with a tracked path to broaden on a case-by-case LIA.
- **Retention minimisation vs. safety review.** Prompt-completion logs are the most useful evidence for abuse / safety issues, and they are also the most sensitive sink. The engineering move is tiered retention — a short-horizon raw log for debugging, a longer-horizon redacted log for safety-trend review, and a signed decision for anything longer.

Each tension has a *stated trade-off* in the datasheet and the DPIA; neither is "privacy wins by default" nor "features win by default".

## Minimisation in a worked AI pipeline

Walking the mod-102 default scenario (tenant-scoped RAG customer-support assistant):

- **Collection** — the ingest schema for tickets includes only the fields the support process needs; a `notes` column passes through ingest-time redaction + vault tokenisation; no demographic attributes captured.
- **Processing** — the fine-tune job manifest reads `body, data_classes, purpose`; the training view drops any row whose `data_classes` includes special-category. The retrieval encoder sees redacted-then-tokenised chunks.
- **Retention** — raw ticket archive 24 months; training snapshot 24 months; retrieval index rotated quarterly with upstream-delete propagation; prompt-completion log 30 days raw + 180 days redacted.
- **Output** — completion passes through PII post-filter; memorisation sampler on 1%; ACL on retrieval blocks cross-tenant leakage; Art. 22 inference on named individuals requires human review.

Every element in the worked example appears in the checkpoint register, with a specific enforcement mechanism and evidence source.

## Summary — what you can do after this chapter

- Separate minimisation into its four distinct dimensions (collection, processing, retention, output) and name the GDPR / CCPA / HIPAA / EU AI Act anchor for each.
- Author a minimisation-checkpoint register with enforcement mechanisms and evidence columns.
- Recognise the four-weak-spots-per-dimension failure modes and the standard enforcement patterns that catch them.
- Mediate the three standing tensions (coverage, researcher freedom, safety review) with documented trade-offs rather than silent de-scoping.

The next chapter picks up the complementary move: where processing minimisation is "pass less data", the vault + tokenisation pattern is "pass *different* data" — a reversible token in place of the raw PII, so the model layer never sees the raw value at all.
