# Corpus-Poisoning Defence — Document Provenance, PII Pre-Scan, and Tombstoning on Takedown

The RAG index is a lossy mirror of a corpus. Everything that lives in the corpus — approved document, scraped page, user upload, email attachment, forum post — can end up as a chunk with a vector and a slot in a nearest-neighbour search. The adversary's wedge: smuggle a document into the corpus that *contains* PII (another person's identifier, a credential, an exfiltrated secret) and let retrieval surface it to users who had no business seeing it. The defence is a pipeline with three jobs: *know where each document came from*, *scan the document for PII before it is indexed*, and *remove the document and its derivatives when a takedown lands*.

This chapter is the ingest-side counterpart to chapter 01's retrieval-side ACL story. The two compose: even an ACL-perfect retriever cannot save a corpus that has swallowed a leaked SSN dump, if the SSN dump is tagged with a public ACL. The ingest-side controls are the first line.

Primary references:

- MITRE ATLAS — *ML Data Supply Chain* and *Poisoning* tactics. [atlas.mitre.org/techniques/AML.T0010](https://atlas.mitre.org/techniques/AML.T0010) (ML Supply Chain Compromise) and [atlas.mitre.org/techniques/AML.T0020](https://atlas.mitre.org/techniques/AML.T0020) (Poison Training Data) — the adversary-facing framing that makes the threat model concrete.
- OWASP — *Top 10 for LLM Applications*, LLM04 (Data and Model Poisoning) and LLM08 (Vector and Embedding Weaknesses). [genai.owasp.org/llm-top-10/](https://genai.owasp.org/llm-top-10/).
- NIST — *SP 800-218, Secure Software Development Framework (SSDF)*, 2022. [doi.org/10.6028/NIST.SP.800-218](https://doi.org/10.6028/NIST.SP.800-218) — the generic provenance-and-supply-chain vocabulary; useful because the RAG corpus is software supply chain.
- in-toto — *in-toto specification*. [in-toto.io/](https://in-toto.io/) — the attestation framework underneath SLSA; useful for building a provenance claim that will survive an audit.
- SLSA — *Supply-chain Levels for Software Artifacts*, v1.0. [slsa.dev/spec/v1.0/](https://slsa.dev/spec/v1.0/) — the four-level maturity model for supply-chain attestation; the RAG corpus pipeline can be positioned against it.
- GDPR Article 17 (right to erasure) and Article 5(1)(e) (storage limitation). [gdpr-info.eu/art-17-gdpr/](https://gdpr-info.eu/art-17-gdpr/), [gdpr-info.eu/art-5-gdpr/](https://gdpr-info.eu/art-5-gdpr/) — the takedown and retention anchors.
- Carlini et al. — *Poisoning Web-Scale Training Datasets is Practical*, USENIX Security 2024. [arxiv.org/abs/2302.10149](https://arxiv.org/abs/2302.10149) — reference paper that establishes the feasibility of low-cost adversarial injection into web-scale corpora.

## The threat model

Three attacker shapes to design against. Each lands in the pipeline at a different point.

### Attacker 1 — The upload channel

A user (legitimate, naive, or hostile) uploads a document that contains someone else's PII. A customer-success agent pastes an email chain including the sender's address, a sales person uploads a prospect list that includes phone numbers, a developer uploads a database dump as a test artefact. The uploader may not realise the content is sensitive; the pipeline has to not care.

Probability: high, continuous. Defensive emphasis: PII pre-scan on every upload, with tagging and gating.

### Attacker 2 — The crawled source

The RAG pipeline ingests a blog, a wiki, a public documentation site. A hostile actor edits the source to insert content designed to be retrieved by targeted queries — e.g. *inserting a fake "support@" address that routes to the attacker* so a RAG-answered "how do I reach support" query surfaces the attacker's contact. This is the *retrieval poisoning* shape.

Probability: medium, depends on ingest breadth. Defensive emphasis: source allow-listing, content diffing, consistency checks across replicas.

### Attacker 3 — The compromised source

The source system itself is compromised — an attacker gains write access to the SharePoint site your RAG indexes, a Confluence account is phished, a Git repo you ingest is pushed to by an attacker with stolen credentials. The pipeline reads the attacker's content as if it were legitimate.

Probability: low but high-impact. Defensive emphasis: anomaly detection on ingest volumes, change-rate limits, and the ability to roll back via immutable provenance.

## Provenance — knowing where each chunk came from

The first discipline: every chunk in the index can answer the question "where did you come from?" without ambiguity. The metadata set from chapter 01 (`source_system`, `source_id`, `ingested_at`, `source_updated_at`) is the minimum; corpus hardening adds more.

A production provenance record looks like:

```python
@dataclass
class DocumentProvenance:
    doc_id:              str          # internal; stable across versions
    source_system:       str          # "sharepoint", "confluence", "upload", "crawl:example.com"
    source_id:           str          # SharePoint item id, Confluence page id, S3 key, URL
    source_version:      str          # etag, revision, or content hash
    ingested_at:         datetime
    ingested_by:         str          # pipeline job id
    uploaded_by:         str | None   # user id for upload channel
    source_fetched_from: str          # exact URL or path hit to fetch
    checksum_sha256:     str          # of the raw bytes
    size_bytes:          int
    mime_type:           str
    license:             str | None   # if the source declares one
    processing_chain:    list[str]    # ["raw", "extract-text", "chunk", "embed"]
    attestations:        list[str]    # signatures / in-toto layout pointers
```

The provenance record is a durable row in a separate "provenance store" — a Postgres table, a dedicated index, or an immutable append-only store. Every chunk points back to its `doc_id`. The reasons to keep provenance separate from the vector index:

- **Longer retention.** Provenance survives chunk deletion (for the tombstone, see below).
- **Different access control.** Reading provenance should not require reading content.
- **Query by non-vector predicates.** "All chunks from source_system=crawl:example.com ingested between dates X and Y" is a straight SQL query; the vector index is not the right store for it.

### Content hashes, not just URLs

The `checksum_sha256` matters. The URL `https://example.com/faq` returns different content at different times; if the pipeline only stores the URL, there is no way to prove "the chunk you retrieved today came from the version I ingested on Monday". Storing the SHA-256 of the raw bytes at ingest time pins the chunk's content to a specific byte-level state, which is what an auditor or an incident-response team needs.

### Attestation, lightweight

For high-sensitivity corpora, the ingest pipeline produces an in-toto-style attestation per document: a signed statement by the ingest service that says "I fetched this URL at this time, received these bytes with this hash, chunked them into these chunks". The attestation lives alongside the provenance and is verifiable without the pipeline code being trusted. For most deployments this is overkill; for regulated-vertical deployments it is the difference between "we think this is where the chunk came from" and "we can prove it".

## PII pre-scan on ingest

The pre-scan is Presidio (or an equivalent) run on the raw document *before* chunking. The purpose is three-fold:

1. **Tag documents with PII classes.** The `pii_detected` metadata we discussed in chapter 03's log row is set here for the whole document.
2. **Gate documents by policy.** A document that contains SSNs, credit cards, or health identifiers can be routed to a different index (with tighter ACL) or rejected outright.
3. **Produce evidence for erasure requests.** A document that is later reported as containing a specific person's data can be found via its PII class without re-scanning the whole corpus.

```python
from presidio_analyzer import AnalyzerEngine
from dataclasses import dataclass

analyzer = AnalyzerEngine()

@dataclass
class IngestScan:
    pii_entities:       list[str]   # types seen
    high_risk_count:    int         # SSN, CC, MEDICAL, etc.
    sampled_spans:      list[tuple[int, int, str]]   # for human review
    policy_decision:    str          # "accept", "route-restricted", "reject"
    reviewed_at:        datetime | None

def pre_scan(text: str, policy: PolicyConfig) -> IngestScan:
    results = analyzer.analyze(text=text, language="en")
    pii_entities = sorted({r.entity_type for r in results})
    high_risk = [r for r in results if r.entity_type in policy.high_risk_set]
    decision = policy.decide(pii_entities=pii_entities,
                             high_risk_count=len(high_risk),
                             text_length=len(text))
    return IngestScan(
        pii_entities=pii_entities,
        high_risk_count=len(high_risk),
        sampled_spans=[(r.start, r.end, r.entity_type) for r in high_risk[:10]],
        policy_decision=decision,
        reviewed_at=None,
    )
```

### Policy decisions

A simple policy model:

- **Accept.** Document has only low-risk PII (names, non-personal emails like `info@`). Indexed into the standard tenant index with the normal ACL.
- **Route-restricted.** Document has moderate PII (personal emails, phone numbers, addresses). Indexed into a restricted index whose ACL is tighter and whose retention is shorter.
- **Reject.** Document has high-risk PII (SSNs, CCs, health identifiers in a context that does not warrant them). Document is not indexed; the uploader is notified; a human review may be queued.

The policy is tenant-scoped. A medical-records deployment has a different "acceptable" definition than a marketing-content deployment; the pipeline reads the tenant's policy config at ingest time.

### Redaction before index vs reject

For some sensitivity classes, redaction-and-index is preferable to reject: the document is useful in retrieval with the sensitive spans masked. A benefit-letter template is an example — the letter is useful; the specific beneficiary's name is not. Redaction-and-index has two sub-decisions:

- **Where is the raw text stored.** The index gets the redacted version; the raw lives in the vault (chapter 03's pattern applied to corpus, not logs). The retriever returns the redacted chunk; a privileged tool can detokenise.
- **Will the embedding be computed from raw or redacted text.** Both are defensible. Redacted embeddings cannot leak the redacted spans even if the retriever's filter fails; raw embeddings have better recall. The system card states the choice.

## Source allow-listing for crawled corpora

For crawl-based ingest, the attacker cannot inject content you do not fetch. The source allow-list is therefore the main control.

- **Enumerated allow-list.** The pipeline fetches only from an explicit list of domains or paths. New sources go through a review before being added. This is tractable for enterprise RAG (10–100 sources); it is intractable for consumer-web RAG (millions of sources).
- **Per-source policy.** Each source has its own policy: PII-scan threshold, update-rate limit, trusted-source flag. A high-trust internal wiki can accept more; a low-trust public forum is routed to the restricted index.
- **Fetch audit.** Every fetch is logged with URL, HTTP status, content hash, and the resulting doc_id. "Did we fetch anything we should not have" is a question an auditor can answer from the fetch log.

## Consistency and change-rate anomaly detection

For sources that update over time (wikis, docs sites, forums), the pipeline needs to notice when "normal" ingest volume is exceeded. Two patterns:

- **Change-rate bounds.** A source that normally sees 50 changes a day and suddenly sees 10,000 is anomalous. The pipeline pauses ingest and alerts; a compromised source is exactly this shape.
- **Content-diff review.** On high-impact sources, a human reviews a summary of day-over-day changes before the new content is indexed. This is expensive; apply to the handful of sources where a successful poison attack would be most damaging (the internal "common knowledge" wiki, the customer-facing help centre).

The alerting threshold is a trade-off. Too strict and every legitimate bulk edit pauses ingest; too loose and an attacker has a window. Set it by observing a baseline for N weeks and setting the bound at a multiple of the observed variance; revisit quarterly.

## Tombstoning — removing a document and its derivatives on takedown

A takedown arrives in three shapes: a user-exercised erasure right (GDPR Article 17), a legal takedown notice (DMCA, local-court order), or an internal decision (document was mis-labelled, user was banned, source was decommissioned). The pipeline must remove every trace of the document.

The derivative graph of a document is:

```
document (provenance store)
  └── chunks (vector index)
        └── vectors (same)
        └── metadata (same)
  └── vault records (raw text, chapter 03)
  └── logs that referenced it (retrieval request_ids, chapter 03)
  └── downstream caches (re-ranker cache, generation KV cache)
```

A correct takedown walks the graph.

### The tombstone record

Rather than silent deletion, write a *tombstone*: a row in the provenance store that says "doc_id D was tombstoned at time T, reason R". The tombstone is immutable and survives the deletion of the chunks it names.

```python
@dataclass
class Tombstone:
    doc_id:        str
    tombstoned_at: datetime
    reason:        str              # "user_erasure", "legal_takedown", "internal"
    requested_by:  str              # user id, legal process reference, admin
    scope:         list[str]        # doc_ids of derivatives also removed
    evidence:      str              # pointer to the request's audit record
```

Why tombstone rather than delete silently:

- **Audit.** A later question "did we delete D?" has a documented answer.
- **Idempotency.** A re-ingest from the same source would re-create D; the tombstone (keyed on `source_system + source_id + checksum_sha256`) causes the re-ingest to short-circuit.
- **Backup restore.** On a restore from backup, the tombstone replay ensures the deleted document is not resurrected.

### The deletion walk

For each doc_id to tombstone:

1. **Write the tombstone.** Durable, immutable, written first so the operation is recoverable.
2. **Delete the chunks.** Each store has a bulk-delete path (`delete_by_metadata`, `delete_by_filter`). The deletion is executed and logged.
3. **Purge the vault.** The vault records that correspond to the document are deleted; the tokens in the logs are invalidated.
4. **Scrub log references.** Log rows that recorded this doc_id as retrieved (via `top_k_chunk_ids`) are either scrubbed (replace with sentinel) or flagged (depending on whether the log already satisfies the retention claim).
5. **Invalidate caches.** Re-ranker caches, KV caches, embedding-cache. The simplest cache-invalidation pattern is a versioned-cache-key: chunks are keyed by `(doc_id, version)`, and the cache-key lookup fails for a deleted doc_id.
6. **Schedule backup replay.** The next backup-restore drill replays the tombstone.

### Idempotent re-ingest

The re-ingest guard is the pattern that keeps the pipeline honest: before ingesting a document, check the tombstone store. If `source_system + source_id + checksum_sha256` matches a tombstone, abort and log the attempted re-ingest.

```python
async def ingest(doc: RawDocument) -> IngestResult:
    tombstone = await tombstone_store.get_by_source(
        source_system=doc.source_system,
        source_id=doc.source_id,
        checksum=doc.checksum_sha256,
    )
    if tombstone:
        audit_log.info("re-ingest blocked by tombstone",
                       doc_id=tombstone.doc_id, reason=tombstone.reason)
        return IngestResult(status="blocked_by_tombstone", tombstone=tombstone)
    # Proceed with pre-scan and index ...
```

The guard composes with crawl-based ingest: the crawler fetches a page whose previous version was tombstoned; the new fetch is a new `checksum_sha256`, so the guard allows it; the pre-scan runs, potentially finding the content still objectionable, and policy applies. The tombstone is a *per-version* block, not a URL-level ban (both patterns exist; pick the one that fits the use case).

## Retention and the storage-limitation principle

The storage-limitation principle (GDPR Article 5(1)(e)) and its equivalents in other regimes require data to be kept no longer than necessary. For a corpus:

- **Each document has a retention policy.** Derived from the source, the document class, and tenant config. "Retain for 7 years" is a different policy from "retain until superseded"; both are legitimate; both need to be made explicit at ingest.
- **Expiry runs as a scheduled job.** The job walks documents past their retention date, applies the tombstone-and-delete flow above, and logs.
- **Retention claims live in the system card.** "This index retains help-centre content for the life of the source document plus 90 days; it retains user uploads for 180 days unless otherwise agreed." A reviewer should find the retention table in the system card without asking.

## Deduplication — the other side of the provenance coin

Corpora grow by duplication. The same document is often ingested twice — once from the source system, once as an uploaded attachment. Two reasons to deduplicate at ingest:

- **Retrieval quality.** k nearest neighbours that are actually the same document is a worse context window than k distinct documents.
- **Erasure propagation.** A document with one `doc_id` is one tombstone; a document with multiple `doc_id`s requires finding and tombstoning all of them.

Deduplication is a content-hash match at ingest time (easy), augmented with near-duplicate detection on embeddings (harder; a MinHash or SimHash variant is traditional). The simple pattern: hash on `checksum_sha256` first; for each pending ingest, check whether another `doc_id` already exists with the same hash. If yes, reuse the existing `doc_id` and record the new source as an alias.

## What to put in the system card

The ingest slice of the system card names:

- The sources the pipeline ingests from, and whether the source list is enumerated (allow-list) or open (crawl).
- The PII pre-scan configuration — which recognisers, which threshold, which decisions map to accept / restrict / reject.
- The provenance record schema and the retention of the provenance store.
- The tombstoning flow and the SLO for takedown propagation.
- The retention table for each document class.
- Any corpus poisoning incidents that have been detected and the dates.

Chapter 06 is the full system-card template.

## Summary — what you can do after this chapter

- Enumerate the three corpus attacker shapes (upload channel, crawled source, compromised source) and the corresponding defensive emphasis.
- Design a document provenance record with the minimum fields and a durable, separate provenance store.
- Operate a Presidio-based PII pre-scan on ingest with a policy that routes documents to accept, restricted, or reject.
- Decide between redaction-and-index and reject for a given sensitivity class, and pick the embedding basis (raw vs redacted) with the system card's claim.
- Operate source allow-listing, per-source policy, and change-rate anomaly detection for crawl-based pipelines.
- Honour a takedown with a tombstone record, a deletion walk across the derivative graph, and an idempotent re-ingest guard.
- Compose the storage-limitation retention table with the tombstoning flow.
- Deduplicate documents at ingest via content-hash with aliasing.
- Populate the ingest slice of the system card from the above.

Chapter 06 pulls everything together into the privacy slice of a RAG / agent system card — the document a reviewer, a customer, or a regulator will read as the public statement of how this system handles private data.
