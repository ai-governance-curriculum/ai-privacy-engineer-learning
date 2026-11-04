# Consent and Legal-Basis Propagation Across the Pipeline

A datasheet (chapter 02) *claims* a lawful basis. A propagation contract is what makes the claim enforceable — the engineering shape that carries the basis / consent / purpose context with every record as it moves from capture to training to retrieval to inference to logging. Without propagation, Art. 5(1)(b) purpose-limitation is a style guideline; with it, downstream jobs can *refuse* to process records whose basis does not match their purpose.

This chapter is the pipeline engineer's contract. It does not re-explain what consent is (mod-101 chapter 02 did). It specifies the fields, the invariants, and the enforcement points.

## Why this matters

Every GDPR / CCPA / HIPAA review of an AI pipeline will produce the same question in slightly different language:

- "What is the lawful basis for including this user's data in the fine-tune corpus?"
- "If that user withdraws consent, how does the withdrawal reach the retrieval index?"
- "The ingest pipeline received a special-category record from this upstream feed — did the downstream training job know?"
- "This prompt-completion log row references an order ID — what basis authorises retaining it for 90 days?"

An auditable answer to each is a *record-level fact* the pipeline can show. A hand-wavy architectural answer is not. The propagation contract is how the pipeline can show.

## The record-level fields that must travel

For personal-data-carrying records, every record must carry these fields from capture onward. They are the privacy equivalent of a security context propagating through a call stack.

### Core propagation fields

- **`lawful_basis`** — enum: `consent | contract | legal_obligation | vital_interest | public_task | legitimate_interests` (GDPR). For CCPA / state laws, a parallel enum (`sale_disclosed | business_purpose | hipaa_tpo | …`) is maintained in the same schema position.
- **`purpose`** — enum / string tag identifying the *specific* processing purpose this record was collected for. Granularity matters: `fine_tune_support_assistant_v3` is a purpose; "product improvement" is not.
- **`consent_token`** — reference to the consent record (where applicable). Not the consent payload. The token resolves to the ledger entry: timestamp, channel, text shown, revocation status.
- **`collection_timestamp`** and **`source`** — where and when the record entered the pipeline.
- **`jurisdiction`** — country / region of the subject at collection (controls which cross-border mechanism applies).
- **`data_classes`** — set tag: `pii | special_category | phi | pci | children | inferred | synthetic`.
- **`retention_deadline`** — absolute timestamp after which this record must not be processed and must be deleted.
- **`controller` and `processor`** — identifier of the controller (for Art. 28 propagation across sub-processors; see mod-101 chapter 02).

These are the fields the *downstream job matcher* checks. A fine-tune job declares `{purpose: fine_tune_support_assistant_v3, basis_allowed: [consent, legitimate_interests_with_lia]}` and the ingest-to-training transition *rejects* any record whose fields do not satisfy that contract.

### Legitimate-interests specific extras

Where `legitimate_interests` is the basis, the record needs enough context for the Art. 6(1)(f) balancing test (Legitimate Interests Assessment) to be referenceable:

- **`lia_id`** — reference to the LIA document that justifies this purpose for this subject population.
- **`objection_status`** — the subject may have objected under Art. 21; a `objection_pending | objection_upheld` state must block processing.

### Consent-specific extras

Where `consent` is the basis:

- **`consent_version`** — which version of the consent text was shown.
- **`consent_scope`** — what processing was *actually* consented to (not what the ingest pipeline would like to do). Granularity matters; see "the granularity trap" below.
- **`consent_withdrawn_at`** — null until withdrawn; non-null means every downstream job must stop and the DSR erasure path must be triggered.

## Where the propagation lives — tabular, document, chunk, embedding

The schema position of the propagation fields depends on where in the pipeline the record is.

### Tabular (columns)

In a relational / columnar warehouse, propagation is **column-level tags + row-level values**:

- Columns carry static metadata — `column.purpose`, `column.data_class`, `column.collected_under_basis`.
- Rows carry dynamic values — `row.consent_token`, `row.objection_status`, `row.retention_deadline`.
- The ETL catalog (dbt / Unity / Snowflake tags / BigQuery policy tags) records the column-level metadata; it is carried forward through SELECT / JOIN / CTE operations.
- A view that joins a `consented` table with a `no-basis` table **inherits** the narrower basis (the join result is no-basis).

Column-tag inheritance across SQL transformations is the non-trivial part. Tools that help (BigQuery column-level security, Snowflake tag-based masking, Databricks Unity Catalog lineage, OpenLineage with column-level support) propagate tags through declared transformations — but a custom Python UDF or a `CREATE TABLE AS SELECT` that re-materialises the data breaks propagation unless the pipeline is instrumented to re-tag.

### Documents (text)

For a text corpus (support tickets, chat transcripts, scraped web pages):

- Propagation is **document-level metadata** — a JSON sidecar or a frontmatter block that carries the same fields as the tabular case.
- When a document is **chunked** (for RAG ingest), each chunk inherits the parent document's tags, plus optional chunk-level overrides (a chunk may have been redacted; its `data_classes` set shrinks).
- When a document is **embedded**, the vector's *metadata* (most vector DBs support JSON metadata per vector) carries the propagation fields. The embedding is a derived artifact; the retention, basis, and consent context travel with it.
- When a chunk is **retrieved into a prompt**, the retrieved chunk's metadata is accessible to the orchestrator to decide whether the prompt-logging sink is allowed to store the chunk.

### Embeddings

An embedding is a derivative data store (chapter 06 mod-102). It must carry:

- The source document / chunk identifier (so an erasure request can find and delete it).
- The source document's propagation fields (purpose, basis, retention).
- The encoder model + version (so if the encoder is deprecated and re-embed is required, you can scope the job).

A vector DB that stores embeddings without their propagation fields is a leaky sink. Pinecone, Weaviate, Qdrant, Milvus, pgvector all support metadata per vector; use it.

### Inference prompts and completion logs

At inference time:

- The **prompt** inherits propagation fields from the *subject whose data is being processed* (the end-user, if their data is in the prompt) *and* the *retrieved documents* (whose data is in the prompt).
- The **completion** inherits the narrower set: a completion derived from retrieved user A + prompt from user B carries both subjects' fields.
- The **log row**, if the log is retained, carries forward the fields so the retention / erasure / basis invariants hold against the log sink.

This is where most pipelines break propagation. A prompt-completion log row stored as plaintext in an S3 bucket with no metadata is unreachable to the basis-check — because there is no basis attached.

## The downstream job matcher — the invariant you enforce

Every downstream job that consumes personal data should declare, in a machine-readable manifest, what it is allowed to consume:

```yaml
# job-manifest.yaml for the fine-tune training job
job: fine_tune_support_assistant_v3
owner: ml-platform
purpose: fine_tune_support_assistant_v3
basis_allowed: [consent, legitimate_interests_with_lia]
lia_ref: lia.ticket-corpus.2025-q4
data_classes_allowed: [pii]
data_classes_denied: [special_category, phi, pci, children]
jurisdiction_allowed: [EU, EEA, CH]
jurisdiction_denied: [US, APAC]
retention_horizon: 24m
```

The matcher is a function that, for each record, returns `include | exclude | hard-fail`:

- `include` — the record's propagation fields are within the manifest and the retention deadline is in the future.
- `exclude` — the record is rejected for this job but may be valid for others (a US-origin ticket for an EU-only fine-tune, for example).
- `hard-fail` — the record carries no basis fields at all, or the basis is unknown. This is a *bug*, not a data-rejection event; it should page the data-platform owner.

Exclusions should be *counted and logged* per job run — if 5% of candidate records get excluded this week and 25% get excluded next week, something upstream changed.

## Propagation across trust boundaries

A trust-boundary crossing — tenant, organisational, jurisdictional, model-provider — is where propagation most frequently breaks. Three patterns:

### Pattern A — Propagation fields survive as record-side metadata

Preferred default. The propagation fields travel as part of the record itself. The downstream system (model provider, retrieval index, log sink) is *given* the context and is contractually required to honour it. Suitable where the downstream is a processor under an Art. 28 agreement that includes a clause to honour purpose / retention / erasure flags.

### Pattern B — Propagation collapses to a *token*, kept in the vault

When the downstream cannot be trusted with the context — public model provider, third-party analytics — the raw data and its context are kept on the controller side. Only a token (chapter 05) is sent over. The context is resolved at egress time; the downstream sees only the tokenised placeholder.

### Pattern C — Propagation fields are *dropped* intentionally with a signed decision

The rarest case. Sometimes the controller decides to send data to a downstream in a way that does not carry its propagation fields (e.g. aggregate statistics that no longer reference individuals). This should be *documented* — the aggregation job's manifest declares the propagation-drop point, the LIA / privacy review signs off, and the datasheet for the aggregate output declares "propagation dropped at aggregation; per-record basis no longer applicable because the output is anonymous under the Art. 29 WP 05/2014 standard".

A silent drop is a control failure. A documented drop is a recorded decision.

## The granularity trap — purpose is specific or it is useless

Art. 6(1)(a) consent must be specific. "Processing for AI purposes" is not specific; the EDPB Opinion 28/2024 and the ICO's AI guidance have both been explicit here. In the propagation schema:

- `purpose: "service_improvement"` — too broad to pass review.
- `purpose: "fine_tune_support_assistant_v3"` — specific, auditable, revokable.
- `purpose: ["fine_tune_support_assistant_v3", "retrieval_index_support_v2"]` — a record consented to two specific purposes; the downstream matcher checks for either.

Where a product team argues for a broader consent text to cover more future uses, the engineering answer is a *more granular consent with pre-authorised purpose expansion* — the consent text says "we will tell you when we propose adding a purpose", not "we will use this for anything we choose later". Both legal and engineering are happier because the propagation fields stay specific.

## Revocation — propagation under withdrawal

Withdrawal of consent is the hardest half of the contract. When a `consent_withdrawn_at` is set, every downstream artifact must be reached:

- The raw record is deleted (or marked tombstoned, if hard deletion from immutable storage is not possible before a grace period).
- The tokenised copy in the vault is deleted (the token's referent is zeroed).
- The retrieval chunk + its embedding + its metadata are deleted (a vector-DB job).
- The fine-tune / training snapshot is either excluded from future training or scheduled for unlearning (mod-110).
- The prompt-completion log rows referencing this subject are deleted or redacted (retention policy dependent).
- The semantic cache keys that could leak the subject's responses are flushed.

The propagation schema is what makes this reachable at all. Chapter 05 (vault) and chapter 07 (ML-BOM) record the derived artifacts; mod-110 teaches the unlearning path.

## Enforcement — four places the contract must live

Propagation is enforced at four points:

- **Capture** — the API / form / event source *requires* the propagation fields. A record without basis / purpose at capture is rejected at the gateway, not quietly saved for later.
- **ETL / transformation** — the catalog (dbt / Unity / BigQuery policy tags) propagates the tags; transformations that drop tags fail CI.
- **Job manifest matcher** — the training / retrieval / evaluation / fine-tune job *declares* its allowed basis / purpose / class / jurisdiction and *rejects* non-matching records at ingest.
- **Egress** — the model-provider / log-sink / analytics-sink is either sent the context (Pattern A) or sent only tokens (Pattern B); the egress job logs which pattern it used.

Each enforcement point produces evidence the DPIA (mod-111) consumes.

## Where propagation is hardest in AI pipelines

Four known weak points, each worth naming on the DFD:

- **Scraped / purchased training data** — the upstream often provides no propagation fields. You can either refuse the data, synthesise defaults (with a documented residual-risk claim), or restrict its downstream uses (`basis_allowed` in every consuming job excludes "legitimate_interests" by default for scraped-origin records).
- **User-generated content in prompts** — the end-user types into the prompt a mention of someone else's data. The *mentioned third party* is a data subject whose basis / purpose fields were never captured. The policy should be: (a) log minimisation (mod-103 ch. 04) to avoid storing it, (b) redaction (mod-104) to blur it, (c) a disclosed-limitation statement in the privacy notice.
- **Agent tool arguments** — a tool call `search_orders(email="alice@…")` propagates personal data into the tool-call trace. The trace sink is often treated as telemetry (short retention) but is often retained longer; mod-109 covers.
- **Model-provider side caches** — if the provider's semantic cache retains content across calls or tenants, your propagation has crossed into a sink you do not control. Vendor contract review + confidential compute (mod-108) are the controls.

Each of these is an acknowledgable gap, not a hidden one.

## Worked example — the SQL-ETL half

Suppose a `tickets_raw` warehouse table has:

```sql
CREATE TABLE tickets_raw (
  ticket_id       STRING PRIMARY KEY,
  tenant_id       STRING,
  subject         STRING,
  body            STRING,
  customer_email  STRING,
  lawful_basis    STRING,     -- enum
  purpose         STRING,
  consent_token   STRING,
  jurisdiction    STRING,
  data_classes    ARRAY<STRING>,
  retention_deadline  TIMESTAMP,
  collection_timestamp TIMESTAMP,
  objection_status STRING,
  consent_withdrawn_at TIMESTAMP
);
```

A view that assembles the fine-tune corpus:

```sql
CREATE VIEW tickets_fine_tune_v3 AS
SELECT ticket_id, tenant_id, body, data_classes, purpose, lawful_basis
FROM tickets_raw
WHERE
  resolved_at IS NOT NULL
  -- propagation checks
  AND purpose IN ('fine_tune_support_assistant_v3', 'fine_tune_support_assistant_v3_or_retrieval')
  AND lawful_basis IN ('consent', 'legitimate_interests_with_lia')
  AND objection_status IS NULL
  AND consent_withdrawn_at IS NULL
  AND retention_deadline > CURRENT_TIMESTAMP()
  AND NOT ARRAY_CONTAINS(data_classes, 'special_category')
  AND NOT ARRAY_CONTAINS(data_classes, 'phi')
  AND NOT ARRAY_CONTAINS(data_classes, 'pci')
  AND NOT ARRAY_CONTAINS(data_classes, 'children')
  AND jurisdiction IN ('EU', 'EEA', 'CH');
```

The view is the matcher. Any ticket that fails any clause is excluded. The number of exclusions is a dashboard metric. A CI test runs a seeded row through the view and asserts the expected outcome at each clause — a regression where a clause is quietly dropped is the kind of bug this chapter is written against.

## Worked example — the vector-DB half

For a retrieval corpus:

```python
# on ingest
client.upsert(
    collection="tickets_support_v2",
    points=[Point(
        id=chunk_id,
        vector=embedding,
        payload={
            "doc_id": doc_id,
            "tenant_id": tenant_id,
            "lawful_basis": "consent",
            "purpose": "retrieval_index_support_v2",
            "consent_token": consent_token,
            "data_classes": ["pii"],
            "retention_deadline": "2027-10-01T00:00:00Z",
            "jurisdiction": "EU",
            "encoder": "bge-m3-v1",
            "chunk_version": 3,
        }
    )],
)

# on query — the orchestrator filters at retrieve
matches = client.query(
    collection="tickets_support_v2",
    vector=query_vec,
    filter={
        "tenant_id": requesting_tenant,              # tenant-scope
        "retention_deadline": {"gt": now_iso},       # un-expired
        "jurisdiction": {"in": allowed_jurisdictions},
        "consent_withdrawn_at": {"is": None},        # not revoked
    },
    top_k=k,
)
```

The vector DB is now a *first-class carrier* of propagation. An erasure job deletes by `doc_id`; a basis change (objection raised) flips `consent_withdrawn_at` and all future retrievals exclude the chunk; an encoder deprecation re-embeds only rows with the old `encoder` value.

## Summary — what you can do after this chapter

- Define the record-level propagation fields and their enforcement points (capture, ETL, job matcher, egress).
- Carry the fields through tabular / document / chunk / embedding / prompt / log forms without silent drop.
- Write a job manifest and matcher that enforces per-purpose, per-basis, per-class, per-jurisdiction eligibility.
- Handle revocation / objection as a propagation signal that reaches derived artifacts.
- Recognise and document the four hard cases (scraped data, UGC in prompts, agent-tool args, provider-side caches).

The next chapter picks up data-minimisation as the complement to propagation: propagation says "which basis permits the data to flow"; minimisation says "which data must *never* flow in the first place and which must be aged out of the pipeline".
