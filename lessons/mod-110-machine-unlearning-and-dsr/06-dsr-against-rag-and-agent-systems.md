# DSR Against RAG and Agent Systems — Easier Than the Model, Harder Than the Database

Chapter 05's decision tree spends its difficulty budget on the model-weights question. For a RAG or agent system, most of the DSR obligation lives *outside* the weights — in the retrieval corpus, the embedding store, the prompt / completion logs, the tool-call audit trail, the memory store the agent uses between turns, and the derivative products (fine-tuned retrievers, distilled rerankers, cached inferences). Each of these is a real database with a real `DELETE` primitive; the DSR against them is the familiar database operation with a twist specific to their shape.

The twist is that these stores are *co-conspirators* in the memorisation claim the chapter-05 tree evaluates. A chunk of the subject's data that lives in the RAG index is a chunk the model will quote verbatim in the next retrieval-augmented answer — a stronger memorisation than anything the base-model weights could produce. An "unlearned" model is still vulnerable if its retrieval corpus contains the forget content. Chapter 06's responsibility is to make the DSR against the *retrieval* surface as complete as the one against the model.

The chapter's structure follows the surface map from chapter 01:

- **Retrieval corpus** (vector store + source-document index).
- **Embedding store** (per-chunk embeddings).
- **Prompt / completion logs**.
- **Agent memory / state** (conversation history, scratchpads, long-term memory stores).
- **Derivative models and cached inferences** (fine-tuned retrievers, rerankers, memoised completions).
- **Replicas, backups, and downstream analytics**.

Each is a stop on the erasure path. Each has its own primitive and its own hazard.

Primary references:

- Regulation (EU) 2016/679, *GDPR*, Art. 17 (erasure), Art. 15 (access), Art. 16 (rectification). Especially Art. 17(2) on third-party notification.
- UK ICO — *Guidance on AI and data protection*, section on *search, retrieval and generative AI*. [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/).
- Pinecone — *Managing indexes: Delete vectors*. [docs.pinecone.io/guides/manage-data/delete-data](https://docs.pinecone.io/guides/manage-data/delete-data).
- Weaviate — *Delete objects*. [weaviate.io/developers/weaviate/manage-data/delete](https://weaviate.io/developers/weaviate/manage-data/delete).
- Qdrant — *Delete points*. [qdrant.tech/documentation/concepts/points/#delete-points](https://qdrant.tech/documentation/concepts/points/#delete-points).
- pgvector — *README* (filter-aware operations). [github.com/pgvector/pgvector](https://github.com/pgvector/pgvector).
- Anthropic — *Model Context Protocol (MCP)* for agent tool-call audit. [modelcontextprotocol.io/](https://modelcontextprotocol.io/).
- OWASP — *Top 10 for LLM Applications*, v2.0. [genai.owasp.org/llm-top-10/](https://genai.owasp.org/llm-top-10/). LLM02 (Sensitive Information Disclosure) and LLM08 (Vector and Embedding Weaknesses).
- NIST — *SP 800-88 Rev. 1, Guidelines for Media Sanitization*. [doi.org/10.6028/NIST.SP.800-88r1](https://doi.org/10.6028/NIST.SP.800-88r1). The media-sanitisation primitive for backups.

## Why RAG makes DSR easier *and* harder

Easier than against the base model: the retrieval corpus is a database. Deletion is `DELETE WHERE`. The "the model memorised this" question is trivially yes — the chunk is a byte-for-byte quote. No influence-function, no TOFU, no probe.

Harder than against the operational database: the retrieval corpus has derivatives (embeddings, reranker caches, retrieval logs). A clean `DELETE` on the source chunk leaves the derivative behind unless the pipeline propagates.

The practical consequence: DSR against RAG is often a *correctly-propagated* `DELETE` plus *index-rebuild / cache-invalidate / downstream-refresh* plumbing. The mechanism is simple; the completeness is the difficulty.

## The erasure cascade

A DSR against a RAG system cascades through the stack in a defined order. Doing it out of order is the most common incompleteness bug. The order:

1. **Source-document deletion** — the authoritative content is removed from where it was ingested from (if under your control) and from the ingest manifest.
2. **Chunk-record deletion** — the chunk table has the subject's chunks removed.
3. **Embedding-store deletion** — the vectors associated with the chunks are deleted from the vector store.
4. **Index rebuild (if needed)** — HNSW / IVF / PQ indexes with heavy delete pressure may need a periodic rebuild; incremental delete is fine for low rates.
5. **Cache invalidation** — reranker caches, retrieval-result caches, memoised completions on queries that would have surfaced the deleted chunks.
6. **Log purge** — prompt / completion logs containing the subject's text.
7. **Agent-memory purge** — long-term agent stores that cached the subject's data.
8. **Derivative-model action** — a decision-tree walk (chapter 05) against any fine-tuned retriever / reranker / embedder that was trained on data including the subject.
9. **Backup / replica reconciliation** — the trickiest step; see below.
10. **Verification** — confirm, with a probe query, that the subject's content is no longer retrievable.

Each step has its own section below.

## Step 1 — Source-document deletion

The RAG corpus is often downstream of a source system: SharePoint, Confluence, Google Drive, a CMS, a CRM, a support-ticket store, a public dataset. The source-level deletion is where the authoritative state changes.

- **Source under your control.** Delete the source record (familiar DB operation) and propagate via the ingest pipeline's standard delta mechanism.
- **Source not under your control (public web, shared SaaS).** You cannot delete from the source; the deletion is in your ingest queue (the next ingest will re-pull the item if it is still there) and in the chunk record. Your runbook must include a *tombstone* on the source-id so the next ingest skips it.
- **Multi-source same-subject deletion.** A subject's data may land in your corpus from three source systems; each needs its own deletion signal or your pipeline needs a per-subject exclude list (chapter 05's exclusion registry).

A common mistake: deleting the source record but leaving the ingest-queue backlog containing the pre-deletion version, which then re-ingests. The ingest pipeline must *stop* consuming source-deltas that predate the deletion, or must re-check the source-id against the deletion registry on every write.

## Step 2 — Chunk-record deletion

The chunk table (whether in Postgres, Pinecone's metadata store, Qdrant's payload, Weaviate's object store, or an application-level relational table) is where the structured content lives.

- **Primitive:** `DELETE FROM chunks WHERE source_id IN (...) OR source_subject_id = ?`.
- **Subject-ID propagation:** every chunk row carries a `source_subject_id` or an equivalent — set at ingest time (chapter 01 names this as a required metadata field). If the ingest pipeline does not propagate the subject-ID down to the chunk, the deletion devolves into a text-search over chunk content, which is slower and less reliable. This is the single most important schema decision in a DSR-ready RAG.
- **Content-only identity:** some sources carry no explicit subject-ID (free-text documents mentioning the subject). Here the deletion is text-match-based; the runbook includes a documented substring / named-entity-recognition pass. False positives are preferable to false negatives — err on the side of deleting too much.
- **Soft vs. hard delete:** a soft-delete (`active = false`) is acceptable only during an interim restriction (chapter 01 Art. 18). For Art. 17 erasure, hard-delete is the target. A tombstone row recording the deletion fact (subject-ID, timestamp, request-ID, who, what) is acceptable and often required for audit — it is *not* the subject's personal data, it is a record of the deletion event.

### Pseudocode

```python
def delete_chunks_for_subject(db, subject_id, request_id):
    with db.transaction():
        rows = db.execute(
            "SELECT chunk_id, source_id FROM chunks "
            "WHERE source_subject_id = %s",
            [subject_id],
        )
        chunk_ids = [r["chunk_id"] for r in rows]
        source_ids = list({r["source_id"] for r in rows})
        db.execute(
            "DELETE FROM chunks WHERE chunk_id = ANY(%s)",
            [chunk_ids],
        )
        db.execute(
            "INSERT INTO erasure_tombstones "
            "(request_id, subject_id, scope, count, deleted_at) "
            "VALUES (%s, %s, 'chunks', %s, now())",
            [request_id, subject_id, len(chunk_ids)],
        )
        return {"chunk_ids": chunk_ids, "source_ids": source_ids}
```

## Step 3 — Embedding-store deletion

The vectors are in a vector store. The store's delete primitive depends on the vendor:

- **Pinecone**: `index.delete(ids=[...], namespace=tenant)` or `index.delete(filter={"source_subject_id": {"$eq": subject}}, namespace=tenant)`.
- **Weaviate**: `collection.data.delete_many(where=Filter.by_property("subjectId").equal(subject))`.
- **Qdrant**: `client.delete(collection_name=..., points_selector=Filter(must=[FieldCondition(key="subject_id", match=MatchValue(value=subject))]))`.
- **pgvector**: `DELETE FROM chunks WHERE source_subject_id = ?` — the embedding lives in the row.
- **OpenSearch / Elasticsearch k-NN**: `DELETE /_index/_doc/{id}` or `_delete_by_query` with the subject filter.

Hazards:

- **Metadata-filter correctness.** A filter-based delete that references a payload field misses rows where the field was not set. Combine ID-based and filter-based deletes; verify the count.
- **Stale index entries.** HNSW / IVF indexes often *mark* deleted but do not reclaim until a rebuild. The entries are not returned by queries, but they occupy memory and can affect recall under heavy delete pressure. Schedule periodic rebuilds (nightly for high-churn corpora, weekly for others).
- **Namespace / tenant scoping.** A multi-tenant RAG (mod-109 chapter 01) partitions data by tenant. Confirm the delete is scoped to the correct tenant and verify no cross-tenant leak-through of the subject's data.

### Verification query

```python
def verify_subject_not_retrievable(index, subject_probe_query, caller, top_k=50):
    """Run a probe query designed to surface the subject's content; verify none returned."""
    vec = embed(subject_probe_query)
    hits = index.search(vec, filter={"tenant_id": caller.tenant_id},
                       top_k=top_k)
    for h in hits:
        assert "<subject-signature>" not in h.text, \
            f"subject content still retrievable in chunk {h.chunk_id}"
```

The probe query is designed per DSR — a natural-language query that would have surfaced the subject's content (name, topic, phrase). The verification must pass before the DSR is considered complete.

## Step 4 — Index rebuild

For high-churn corpora, the HNSW / IVF index accumulates tombstones that degrade query recall and inflate memory. The rebuild is a background job that re-indexes live rows.

- **Trigger rule:** rebuild when tombstones exceed a threshold (10–20% of index size) or periodically (nightly).
- **During rebuild:** the index remains query-able on the old state; the new index replaces it atomically.
- **Operational caveat:** rebuild cost is proportional to index size. Plan for it in the resource budget; a 100M-vector index can take hours.

A DSR-driven ad-hoc rebuild is rarely needed — the per-query filter handles the tombstone exclusion — but the system card should name the rebuild cadence so a reviewer can confirm the deletion "cleans up" within a defined window.

## Step 5 — Cache invalidation

The deletion is incomplete if a cached result still surfaces the subject's content. Three cache surfaces to invalidate:

- **Retrieval-result caches.** A query-to-chunks cache keyed on the (query, caller) pair. Invalidate any cache entry whose result set contains a deleted chunk. The brute-force approach is cache-flush; the targeted approach requires an inverted index from chunk-ID to cache-key.
- **Reranker caches.** A reranker that caches scores on (query, document) pairs. Same propagation.
- **Completion caches.** A memoised completion cache keyed on (prompt, context). Any completion whose context contained a deleted chunk is stale and must be invalidated.

A simple-and-aggressive policy: flush the entire cache for the affected tenant on each subject deletion. The cost is a transient recompute spike; the correctness is total. For high-cadence deletions, build the targeted invalidation.

## Step 6 — Log purge

Prompt / completion logs retain the subject's prompts (what they asked, or what others asked *about* them) and the model's outputs (potentially containing memorised or retrieved content). Chapter 03 of mod-109 is the companion on logging policy; here is the DSR-specific wiring.

- **Retention-based auto-purge.** Logs older than retention are already gone. If the subject's DSR falls within the retention window, you must actively purge.
- **Full-text search over logs.** Searchable log store (Elasticsearch, ClickHouse, Loki) makes subject-text search cheap. Flat-file logs (unindexed) are a correctness liability — the subject-text search is a linear scan, and DSR SLA may not accommodate it.
- **Purge primitive.** `DELETE FROM prompts WHERE user_id = ? OR text LIKE '%<subject>%'` is the rough shape; redaction (replace with `[REDACTED]` and keep the record for operational analytics) is sometimes an acceptable alternative with DPO sign-off — the decision rests on whether the redacted log is still "personal data" in the GDPR sense.
- **Third-party log stores.** A log pipeline that ships to a SaaS (Datadog, New Relic) must have a DSR hook — a propagation step that calls the vendor's delete API. Many vendors offer this; verify yours does and that the propagation is in-SLA.
- **Backup copies of logs.** Backups of logs are a specific problem (step 9).

## Step 7 — Agent memory

An agent that uses tools and maintains state between turns has memory stores beyond the RAG index:

- **Conversation history.** The most-recent user prompts plus model outputs. Scoped to the agent-user session; delete on user account deletion as part of the standard DSR.
- **Scratchpad / tool-trace memory.** Intermediate reasoning, tool calls, tool results. Often retained for debugging and quality. On DSR, purge like logs — full-text search and delete (or redact).
- **Long-term memory.** An agent that writes to a persistent memory store across sessions. Each memory row is personal data about either the user or a third party the user discussed. The deletion is a database operation on the memory store, scoped by subject ID.
- **Tool-call audit trail.** Records of agent tool invocations (database writes, API calls, email sends). *These are often not personal data* — they are records of actions taken — but they may reference personal data. Treat them like logs: full-text search, purge or redact.

The engineering primitive is the same as for logs; the discovery is often the harder part because agent memory lives in several places (a Redis session store, a Postgres long-term store, a trace-ingest pipeline, the LLM provider's own session store if the agent uses a managed-conversation API).

## Step 8 — Derivative models and cached inferences

A fine-tuned retriever or reranker trained on data that included the subject's chunks has encoded some signal from them. Apply the chapter-05 decision tree:

- **Was the subject in the fine-tune set?** Check the fine-tune's data manifest.
- **Did the fine-tune memorise meaningfully?** Usually less likely for retrievers / rerankers than for generators — they are usually smaller and trained with contrastive objectives that are less memorisation-prone. Measure if the DPO requires.
- **If yes to both:** re-fine-tune on the corrected corpus, or unlearn via chapter-03/04 methods.
- **If no on either:** document and skip.

Cached inferences (a memoised retrieval score, a cached reranking output, a snapshot of a prior answer) are closer to the log-purge case — delete or redact based on content.

## Step 9 — Backups and replicas

Backups are the most frequent source of incomplete DSR. Deleting the live chunk but retaining a six-month-old backup containing the chunk is a compliance gap.

Three operational patterns:

- **Immutable backups with retention.** If the backup is immutable (object lock, write-once-read-many), you cannot delete within it. The compliance argument rests on *retention* — the backup expires within a defined window (e.g. 30 days) after which the subject's data is naturally purged. Document the window in the system card; the subject is informed that full erasure takes until the backup rotation completes.
- **Mutable backups.** Apply the deletion to the backups themselves — a per-backup deletion job. Expensive, but technically complete.
- **Delta backups.** Delete the live row; the next incremental backup captures the deletion; past incremental backups still contain the row but are tied to a retention window. Same argument as immutable with retention.

NIST SP 800-88 is the authoritative reference for *destruction* of media; for cloud object storage, the provider's cryptographic erasure (deleting the key that encrypts the data) is the standard mechanism and is acceptable if auditably implemented.

Replicas (read-only secondary databases, caches, geo-replicated copies) propagate deletes automatically if replication is streaming; manual action is needed only for snapshots.

## Step 10 — Verification

The DSR is complete only when the verification probe confirms the subject's content is no longer surface-able.

- **Probe query.** A natural-language query designed to surface the subject's content as it was before deletion. A few variants to account for paraphrasing.
- **Run the probe as the caller the subject or a legitimate user would.** Use the tenant and the role the DSR is answering for; a probe as an admin bypassing filters is not a user-facing verification.
- **Confirm zero results containing the subject's content.** If a result contains the subject, the deletion is incomplete — trace back through the cascade to find the missed step.
- **Record the probe and its result.** The audit trail (chapter 07) quotes the probe, the timestamp, and the "zero subject-content" assertion. For a reviewer, this is the evidence the DSR was *verified*, not just *issued*.

## The agent-system additional complications

Agent systems introduce two beyond-RAG considerations:

- **Tool-mediated external writes.** An agent that wrote the subject's data to an external system (sent an email, filed a ticket, pushed to a SaaS) created personal data *outside* your control. Art. 17(2) obliges the controller to take reasonable steps to notify downstream controllers; the engineering artefact is a *downstream notification runbook* that enumerates "who did the agent write to for this subject". The agent's tool-call audit log (step 7) is the source of this enumeration.
- **Cross-session identity reuse.** An agent that maintains an identifier for a user across sessions may have cached state under that identifier in third-party tools (OAuth-bound integrations, memory APIs, workflow systems). DSR enumeration includes these; the integration's DSR hook, if available, is the mechanism.

## Rectification against RAG

Rectification (Art. 16) is the easy case in RAG — simpler than against the base model. The chunk is replaced with a corrected version:

- Update the source document.
- Re-ingest, producing a new chunk record with a new embedding.
- Delete the old chunk.
- Invalidate caches.

The old chunk's `source_updated_at` moves to the new one's. The DSR response notes the version change and the propagation timeline.

## Access against RAG

Access (Art. 15) against a RAG system returns the subject's chunks plus the metadata:

- `SELECT * FROM chunks WHERE source_subject_id = ? OR text ILIKE '%<subject>%'` → export as JSON.
- Include chunk-ID, source-ID, source timestamp, ACL (chapter 01 of mod-109), retention policy.
- Include completion-log entries where the subject was the prompt-caller or the prompt-subject.

The export bundle is a subset of the mod-109-chapter-06 system-card content, filtered to the subject. Produce it in a structured format (JSON / CSV) per Art. 20 norms.

## Restriction and objection against RAG

Restriction (Art. 18) and objection (Art. 21) are implemented as *filter flags* at retrieval time:

- Add a `processing_restricted` or `objected_purposes` field to the chunk metadata (or to a side table joined at retrieval).
- The retrieval filter excludes chunks where the restriction is active or the purpose matches the objection.
- The data remains in the store — it is just not retrieved for the restricted / objected processing.
- When restriction lifts or objection is resolved, clear the flag; the chunk returns to retrievability.

This is the implementation chapter 01's runbook names in abstract. The RAG-specific detail is that the flag lives alongside the chunk metadata that the retrieval filter already consumes — mod-109 chapter 01's ABAC / RLS pattern composes directly.

## The RAG-DSR system-card entry

The system card (mod-109 chapter 06 is the full template) for a RAG system should include a DSR-specific section:

- *Subject-ID propagation claim.* Chunks carry `source_subject_id`; verified.
- *Deletion cascade.* Each of the ten steps enumerated above is named and the implementation is pointed to.
- *Index-rebuild cadence.* Daily / weekly / threshold.
- *Cache-invalidation policy.* Flush-tenant / targeted; which caches are covered.
- *Log retention.* Window, searchability, purge-primitive.
- *Agent-memory purge.* Which memory stores, which purge-primitive.
- *Backup retention window.* The time until subject data is fully purged from backups; disclosed to the subject.
- *Verification probe.* The template, plus the per-DSR instance.

The DSR audit trail (chapter 07) references these fields for each specific request.

## Common bugs — the list a RAG-DSR reviewer walks

If you are reviewing a RAG system for DSR readiness, walk this list:

- Chunks without `source_subject_id` metadata.
- Delete-by-filter paths that do not propagate to the embedding store.
- Caches without invalidation hooks.
- Logs stored only on flat files or in a vendor with no delete API.
- Agent long-term memory stores not in the DSR inventory.
- Backups with no documented retention window.
- Fine-tuned retrievers / rerankers not treated as derivative models.
- No verification probe after deletion.

Each item here is a potential DSR failure mode. Fix them before the first regulator-facing DSR, not after.

## Summary — what you can do after this chapter

- Walk the ten-step erasure cascade — source, chunks, embeddings, index, caches, logs, agent memory, derivatives, backups, verification — for a specific RAG system.
- Implement the chunk-and-embedding deletion primitives for Pinecone / Weaviate / Qdrant / pgvector.
- Design the chunk-record schema to include `source_subject_id` so the cascade has a join path.
- Build the verification probe that confirms a subject's content is no longer retrievable post-deletion.
- Enumerate the agent-system additional surfaces (tool-call audit, long-term memory, cross-session identity reuse) and name the propagation step for each.
- Author the RAG-DSR system-card entry that records the subject-ID propagation, the deletion cascade, the cadence, and the verification probe.

Chapter 07 is the final step of every DSR regardless of the mechanism chosen: the audit trail that packages the intake, the engineering action, the verification artefacts, and the response to the data subject into a bundle that survives a supervisory-authority inspection.
