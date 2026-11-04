# Per-Tenant Retrieval Access Control — ACL Propagation Through Chunk, Embedding, and Retrieval

The RAG system that answered a question for tenant A yesterday is the same code path that will, under the wrong configuration, return a chunk from tenant B's documents today. Retrieval is a lookup; a lookup that is not filtered is a lookup that returns every indexed row; every indexed row that reaches the model's context window is a row the model may quote back to the user. The privacy failure here is not sophisticated — it is a missing WHERE clause.

This chapter is the retrieval-side part of the RAG-privacy story: how the authorisation decision that *tenant A's user can read document D* travels, unbroken, from the source system where D was ingested, through the chunker, through the embedder, through the vector store, through the retrieval API, and into the prompt the LLM sees. The threat it defeats is the one that lands most often in RAG-privacy incident reports: a tenant answering a question with a fact from another tenant's corpus.

The peer boundary is `rag-engineer` (level 25). `rag-engineer` owns the RAG system — the retriever, the generator, the quality metrics. This module owns the *authorisation* slice of that system. When a design decision turns on "does this retriever compose with our tenant-isolation story", that is our call; when it turns on "is this re-ranker's recall acceptable", that is theirs.

Primary references — read them before building:

- OWASP — *Top 10 for Large Language Model Applications*, current version (2025, v2.0). [genai.owasp.org/llm-top-10/](https://genai.owasp.org/llm-top-10/) — LLM02 (Sensitive Information Disclosure) and LLM08 (Vector and Embedding Weaknesses) are the entries that name RAG retrieval-leakage as a top-tier risk and set the expectations for a reviewer who cites the list.
- Pinecone — *Namespaces* and *Metadata filtering* reference. [docs.pinecone.io/guides/indexes/data/understanding-namespaces](https://docs.pinecone.io/guides/indexes/data/understanding-namespaces) and [docs.pinecone.io/guides/index-data/indexing-overview#metadata](https://docs.pinecone.io/guides/index-data/indexing-overview#metadata) — the canonical way one commercial vector store implements per-tenant isolation.
- Weaviate — *Multi-tenancy* reference. [weaviate.io/developers/weaviate/concepts/data#multi-tenancy](https://weaviate.io/developers/weaviate/concepts/data#multi-tenancy) — the "tenant" is a first-class object in Weaviate; worth reading even if you deploy on a different store, because it names the primitives cleanly.
- Qdrant — *Payload filtering* and *Multi-tenancy* guide. [qdrant.tech/documentation/concepts/filtering/](https://qdrant.tech/documentation/concepts/filtering/) and [qdrant.tech/documentation/guides/multiple-partitions/](https://qdrant.tech/documentation/guides/multiple-partitions/).
- pgvector — *README* and *Filtering* notes. [github.com/pgvector/pgvector#filtering](https://github.com/pgvector/pgvector#filtering) — critical read if you intend to rely on PostgreSQL row-level security for ACL propagation.
- PostgreSQL — *Row Security Policies* documentation. [postgresql.org/docs/current/ddl-rowsecurity.html](https://www.postgresql.org/docs/current/ddl-rowsecurity.html) — the RLS primitive that most pgvector-based RAG deployments stand up their ACL on.
- NIST — *SP 800-162, Guide to Attribute Based Access Control (ABAC) Definition and Considerations*, 2014 (Rev. 1 draft, current). [doi.org/10.6028/NIST.SP.800-162](https://doi.org/10.6028/NIST.SP.800-162) — vocabulary and design pattern for the subject-attribute / resource-attribute matching that an ACL-propagating retriever has to do.

## The failure mode in one sentence

A RAG retriever that executes `SELECT chunk FROM index ORDER BY embedding <=> $query LIMIT k` *without* a `WHERE tenant_id = $caller_tenant` clause is a retriever that will eventually return a chunk from another tenant. There is no stochastic recovery from this — it is a correctness bug. The entire chapter is about making sure that WHERE clause is present, correct, and not bypassable, at every point in the pipeline.

## The three shapes of RAG isolation

The architecture question is "how much isolation do we buy, at how much cost". Three shapes recur:

### Shape 1 — Physical isolation: one index per tenant

Each tenant gets its own namespace, collection, index, schema, or cluster. The retriever is scoped to the caller's tenant's resource; a cross-tenant query is not a bug in a filter predicate — it is a connection to the wrong cluster, which is a different, much more visible, class of error.

- **Pinecone:** one *namespace* per tenant inside a shared index. The namespace is a first-class query parameter; a query without the right namespace does not return another tenant's data — it returns zero results.
- **Weaviate:** multi-tenancy mode on the collection. Each *tenant* is a distinct partition; the query is addressed to a tenant and only sees that tenant's objects.
- **Qdrant:** separate *collections* per tenant, or a shared collection with partitioned storage on a `group_id` payload field.
- **pgvector:** separate schemas or separate databases per tenant.

Properties:

- **Strongest default.** The attack surface is the tenant-routing code, not the filter predicate on every query. A reviewer can audit a short piece of code — "given a request, pick the right namespace / collection / schema" — rather than every retrieval call site.
- **Operational cost.** Many tenants means many resources. Pinecone namespaces scale well (one index can hold tens of thousands of namespaces); Weaviate's multi-tenancy is designed for this; a per-tenant Postgres schema model does not scale to millions of tenants cleanly.
- **Cross-tenant features are hard.** Global admin queries, cross-tenant analytics, or shared-document models (e.g. "common knowledge visible to all") require a second index that is explicitly shared and explicitly audited.

Choose this shape when the compliance surface is strict (B2B SaaS with signed data processing agreements that promise tenant isolation; regulated-vertical deployments where the DPIA will be read by a customer's privacy officer), or when tenants are few and heavy.

### Shape 2 — Logical isolation: one shared index, a tenant-id filter on every query

All tenants share a single index; every chunk carries a `tenant_id` field in its metadata / payload; every query is required to include a `tenant_id = $caller` filter. The isolation is enforced by the filter predicate.

- **Pinecone:** one index, chunks tagged with `metadata.tenant_id`, every query includes a `filter={"tenant_id": {"$eq": caller_tenant}}` clause.
- **Weaviate:** one collection, objects tagged with a `tenantId` property, every query includes a `where` filter on that property.
- **Qdrant:** one collection, each point's payload includes `tenant_id`, every query uses the `must` filter condition.
- **pgvector:** one table, each row has a `tenant_id` column, every query joins with the caller's identity (preferably via Postgres RLS — see below).

Properties:

- **Cheaper at scale.** One index for a million tenants is cheap; a million indexes is not.
- **Fragile by construction.** If one call site forgets the filter, that call site returns cross-tenant results. The entire privacy claim depends on every retrieval path going through the same guard. A new feature added by a developer who did not read the README is a plausible attack path.
- **Index-wide search.** A query without a tenant filter runs the ANN search over every tenant's vectors; a bug that returns the unfiltered results gives an attacker a dossier of other tenants' most-relevant-to-the-query chunks. This is exactly the context-window-leakage shape chapter 02 focuses on.

Choose this shape only with robust guard-rails: a thin wrapper library that every caller uses (no direct vector-store SDK access from the app), a test harness that proves a caller without the filter gets an error rather than results (chapter 02), and a logging claim that *every* retrieval call carried a tenant filter.

### Shape 3 — Hybrid: logical + row-level security at the store

The index is logically shared, but the store itself enforces the tenant filter — not the application. On pgvector + Postgres, this is **row-level security (RLS)**: every row has a `tenant_id`; an RLS policy on the table says `USING (tenant_id = current_setting('app.current_tenant')::uuid)`; the application sets `app.current_tenant` as a session-level GUC on each connection checkout; every SELECT (including the vector-search one) automatically has the filter applied by the planner.

- **The predicate lives in the database, not the app.** A developer who writes `SELECT * FROM chunks ORDER BY embedding <=> $1 LIMIT 10` without mentioning tenant gets a result set that is already filtered by the RLS policy. There is no way for them to forget.
- **Bypassing requires a role with `BYPASSRLS` privilege**, which is a review item (chapter 02's negative-test harness has a specific case for this).
- **The caller-identity GUC is the only thing the application sets.** A correct connection-pool wrapper sets the GUC on each checkout; the predicate is the database's concern after that.

This shape is the one most modern pgvector-based RAG deployments converge on. It is also the shape that composes cleanly with ABAC: if the row carries both `tenant_id` and `document_sensitivity_tag`, the RLS policy can enforce both. Qdrant's "payload partitioning" and Weaviate's multi-tenancy are the store-side equivalents on their platforms; the key design property is the same: *the store, not the application, applies the filter*.

## ACL propagation through the pipeline

A RAG index is produced by a pipeline — ingest → chunk → embed → write. ACL propagation means the authorisation decision that attaches to the *source* object is carried, intact, to the *chunk* and to the *vector*. Each stage is a point where it can be lost.

### Stage 1 — Ingestion

The source system (SharePoint, Google Drive, Jira, Confluence, a SQL table, S3) has its own authorisation model. Each object has an ACL — explicit grants, group memberships, folder inheritance, document sensitivity labels. Your ingest job reads those objects. The privacy claim depends on what the ingest job records alongside each object.

The minimum metadata set to persist with each ingested object:

- `source_system`, `source_id` — the object's canonical identifier in its source system (for erasure propagation and re-fetching).
- `tenant_id` — the top-level authorisation group. In a B2B SaaS this is typically the customer / workspace.
- `document_acl` — a list of principal identifiers (user IDs, group IDs, role names) allowed to read the object, as read from the source system *at ingest time*.
- `sensitivity_label` — if the source system has labels (SharePoint Information Protection labels, Google Workspace labels), carry them.
- `ingested_at`, `source_updated_at` — used to detect staleness and to anchor retention.
- `source_retention_policy` — for erasure propagation; if the source document is deleted, the pipeline needs to know.

Design principle: **the ACL is a property of the content, not a property of the retrieval endpoint**. A caller-side-only check can be bypassed; a content-side ACL travels with the chunk.

### Stage 2 — Chunking

A chunk is a span of the source object. Every chunk inherits the source's ACL. The common failure here is forgetting to copy the metadata when the chunker splits — the chunk record ends up with its chunk text and embedding but not the `tenant_id` or `document_acl`. The retriever then runs without a filter it cannot apply.

Concretely, a chunk record should look like (in Pydantic / dataclass form):

```python
from dataclasses import dataclass
from datetime import datetime

@dataclass
class Chunk:
    chunk_id: str
    source_system: str
    source_id: str
    span_start: int
    span_end: int
    text: str
    tenant_id: str
    document_acl: list[str]        # flattened — see below
    sensitivity_label: str | None
    ingested_at: datetime
    source_updated_at: datetime
    # vector stored separately; this is the metadata row
```

ACL *flattening* is the standard practice: resolve group memberships to user IDs at chunk time so the query-time filter is a simple set-membership test. The cost is staleness — when group membership changes, chunks whose ACL was flattened at ingest are now out of date until a re-ingest. The alternative is to *keep* group IDs in the ACL and resolve at query time (join the chunk's `document_acl` against the caller's resolved group memberships); it is accurate but slower. Both are valid; pick one and document it in the system card (chapter 06).

### Stage 3 — Embedding

Embeddings are a side-effect of the chunk's text, not of the ACL. The embedder itself does not usually introduce a privilege-escalation path — but there are three things to watch.

- **Shared embedding models across tenants are fine.** The embedding function is a pure mapping from text to vector; it does not retain data. No privacy claim is broken by using the same model to embed tenant A's and tenant B's chunks.
- **Fine-tuning the embedder on a specific tenant's corpus is not fine.** If you fine-tune the embedding model on tenant A's data to improve recall for that tenant, the fine-tuned weights encode something about A. If those weights are then used to serve tenant B, that is a cross-tenant leak through the model. Module 105 covers the memorisation story that makes this concrete; module 106 covers the DP-training mitigation if fine-tuning is unavoidable.
- **Vector dimensions can themselves carry information.** A short chunk with distinctive terms embeds to a vector near the terms' mean embedding. If an attacker in another tenant crafts a probe query whose embedding is near the suspected target's, they can observe retrieval timing or score metadata if the system leaks it. This is one of the behaviours OWASP LLM08 names. Mitigation: the retrieval filter runs *before* nearest-neighbour search when the store supports it (Qdrant payload-filter-first, pgvector WHERE-first-then-ORDER-BY); do not rely on score-based cut-offs alone.

### Stage 4 — Writing to the vector store

The chunk's metadata must be written alongside its vector in a way the store can filter on cheaply. Each store has its own idiom:

```python
# Pinecone
index.upsert(
    vectors=[(chunk.chunk_id, vector, {
        "tenant_id": chunk.tenant_id,
        "document_acl": chunk.document_acl,
        "sensitivity_label": chunk.sensitivity_label,
        "source_id": chunk.source_id,
    })],
    namespace=chunk.tenant_id,   # physical isolation layered on top
)

# Weaviate
collection.with_tenant(chunk.tenant_id).data.insert(
    properties={
        "text": chunk.text,
        "tenantId": chunk.tenant_id,
        "documentAcl": chunk.document_acl,
        "sensitivityLabel": chunk.sensitivity_label,
        "sourceId": chunk.source_id,
    },
    vector=vector,
)

# Qdrant
client.upsert(
    collection_name="chunks",
    points=[PointStruct(
        id=chunk.chunk_id,
        vector=vector,
        payload={
            "tenant_id": chunk.tenant_id,
            "document_acl": chunk.document_acl,
            "sensitivity_label": chunk.sensitivity_label,
            "source_id": chunk.source_id,
        },
    )],
)
```

Review item: the metadata keys should be *required* — the ingest job fails loudly if `tenant_id` is missing on a chunk. A library wrapper around the SDK is the natural place for the required-field check; downstream developers should not be able to write a vector without a tenant.

### Stage 5 — Retrieval

The query carries the caller's identity. The authoriser resolves the caller to the predicate the retriever will apply:

```python
def retrieve(
    query: str,
    caller: AuthenticatedPrincipal,
    top_k: int = 10,
) -> list[Chunk]:
    vector = embed(query)
    # The filter is derived from the caller, not supplied by the caller.
    filter_ = {
        "must": [
            {"key": "tenant_id", "match": {"value": caller.tenant_id}},
            {"key": "document_acl", "match": {"any": caller.readable_acl_values()}},
        ]
    }
    return vector_store.search(vector, filter=filter_, limit=top_k)
```

The three design properties to enforce:

1. **The filter is derived from the authenticated principal, not from the request body.** A request body that includes a `tenant_id` the caller gets to choose is an escalation primitive. The server reads the caller's identity from its own auth context and derives the filter.
2. **The store-side filter is a hard filter, not a re-rank.** ANN plus post-hoc filter is not the same as pre-filtered ANN. Some stores support both; pick the one that returns zero results for a cross-tenant query, not the one that scores-then-filters (you may run out of results below the cut-off and get fewer than k; that is the correct behaviour).
3. **No request-body knob turns the filter off.** `include_all_tenants=true` is not a feature; if an admin UI needs cross-tenant search, it uses a different endpoint with a different auth guard and separate audit logging.

## The row-level-security recipe on pgvector

This is the recipe that appears most often in production RAG deployments on Postgres. It is worth memorising.

```sql
-- Schema
CREATE TABLE chunks (
    chunk_id        uuid PRIMARY KEY,
    tenant_id       uuid NOT NULL,
    document_acl    text[] NOT NULL,
    sensitivity     text,
    text            text NOT NULL,
    embedding       vector(1536) NOT NULL,
    source_id       text NOT NULL,
    ingested_at     timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX chunks_embedding_idx ON chunks USING hnsw (embedding vector_cosine_ops);
CREATE INDEX chunks_tenant_idx ON chunks (tenant_id);

-- Turn RLS on and deny by default
ALTER TABLE chunks ENABLE ROW LEVEL SECURITY;
ALTER TABLE chunks FORCE ROW LEVEL SECURITY;  -- applies even to the table owner

-- The predicate: tenant match + ACL intersection with caller's groups
CREATE POLICY tenant_isolation ON chunks
    USING (
        tenant_id = current_setting('app.caller_tenant')::uuid
        AND document_acl && string_to_array(current_setting('app.caller_groups'), ',')
    );
```

On the application side, every connection checkout sets the two session GUCs from the authenticated principal:

```python
# Pseudocode for connection checkout
async def checkout(caller: AuthenticatedPrincipal) -> Connection:
    conn = await pool.acquire()
    await conn.execute("SET LOCAL app.caller_tenant = $1", caller.tenant_id)
    await conn.execute("SET LOCAL app.caller_groups = $1",
                       ",".join(caller.group_ids))
    return conn
```

The `SET LOCAL` scopes the GUC to the current transaction; it is reset on each checkout and cannot leak between concurrent callers sharing a pool. Operationally this is the most common correctness bug: a deployer who uses a connection pool without `SET LOCAL`, or who forgets to begin a transaction so `SET LOCAL` is a no-op, has broken the isolation. The negative-test harness in chapter 02 has a specific case for this.

Two more hygiene items:

- **`FORCE ROW LEVEL SECURITY`** is on because by default the table owner bypasses RLS, and the application role often is the owner. Without `FORCE`, a correctness bug that uses the owner role returns every row.
- **`BYPASSRLS` role attribute** is the escape hatch for migrations and admin jobs. Grant it to a narrow service role used only by deliberately-cross-tenant operations (reindexing, analytics). Never grant it to the application role.

## ABAC on top of RLS

RLS lets you compose multiple attribute checks in a single predicate. Common additions to the tenant-isolation base:

- `sensitivity <= current_setting('app.caller_clearance')::text` for a document-sensitivity model where callers have clearances (useful inside a single tenant with internal HR / legal / finance barriers).
- `jurisdiction = ANY(current_setting('app.caller_jurisdictions')::text[])` when the retrieval boundary includes a data-residency property (e.g. an EU-resident caller must not retrieve a chunk whose source document is tagged US-only).
- `NOT (chunk_id = ANY(current_setting('app.erasure_blocklist')::uuid[]))` as a fast way to honour an erasure request that has not yet been processed by the batch deletion job (see module 110).

This is the ABAC pattern NIST SP 800-162 names. The RAG retriever's filter becomes a predicate over subject attributes (the caller) and resource attributes (the chunk's metadata); the privacy claim is a well-formedness check on the predicate. Review items for the ABAC-extended retriever:

- Every attribute mentioned in the predicate has a documented source (which auth system, which token claim, which admin UI).
- Every attribute has a documented default (what happens if a caller has no clearance claim? If the deny-by-default is "no results", say so).
- The policy is *testable in isolation*: given a sample caller and a sample chunk, the evaluation returns a Boolean and the unit test checks it.

## Caller identity, in one more layer of detail

The retriever receives a `caller` object. What shape is defensible?

A minimum shape:

```python
@dataclass
class AuthenticatedPrincipal:
    principal_id: str              # user id or non-human identity id
    principal_type: str            # "user", "service", "agent"
    tenant_id: str
    group_ids: list[str]           # resolved at auth time, cached briefly
    clearance: str | None
    jurisdictions: list[str]
    auth_token_id: str             # for audit; the jti of the bearer token
    auth_expires_at: datetime
    delegated_from: str | None     # when a user has delegated to an agent
```

Three properties:

1. **The principal is an output of authentication, not a copy of request input.** A JWT verified by the gateway; a session cookie keyed to a server-side session record; an OIDC assertion. The retrieval endpoint never trusts a `tenant_id` from the body.
2. **It carries the group membership evaluated at auth time.** This is what the retriever intersects against the chunk's `document_acl`. The trade-off between auth-time snapshot and query-time resolve is the same one we discussed at the chunk level; pick one and document it.
3. **It records who the caller is on behalf of.** When an agent calls the retriever on a user's behalf, `principal_type == "agent"` and `delegated_from` is the user. Chapter 04 builds on this heavily — agent-tool scoping is the delegation-chain story in detail.

## The test harness — foundation

Chapter 02 is the full negative-test harness for cross-tenant leakage, but the foundation lives here: *any* ACL-propagation claim your deployment makes is a test obligation. The claim says "a chunk belonging to tenant A, with ACL excluding group G, is not returned to a caller in tenant B or to a caller in A who is not in G". The test for the claim is:

```python
def test_cross_tenant_not_returned(fixtures):
    # Given a chunk for tenant A
    a_chunk = make_chunk(tenant_id="tenant-A", document_acl=["group-A-eng"])
    ingest([a_chunk])

    # When a caller from tenant B queries with a semantically-identical query
    caller_b = AuthenticatedPrincipal(
        principal_id="user-b", tenant_id="tenant-B",
        group_ids=["group-B-eng"], ...
    )
    results = retrieve(query="the exact topic the chunk is about",
                       caller=caller_b, top_k=10)

    # Then the chunk is not in the results
    assert a_chunk.chunk_id not in [r.chunk_id for r in results]
```

The test's structure matters more than the specific library. Three features of a good negative test:

- **Content-identical, authorisation-distinct.** The chunk and the query are *intentionally* similar in embedding space; the only reason the chunk should not be returned is the ACL. A test that uses semantically unrelated content does not prove the filter is working; it proves the embedding model has reasonable recall. We want the first.
- **Deterministic ingest fixtures.** The chunks are produced by a fixture that the test reads; the embedding is produced by a real embedder (or a deterministic stub) so the top-k ranking is reproducible.
- **Fails on removal of the filter.** The test must fail if someone comments out the filter predicate. If the test passes without the filter, the test is testing something else.

Chapter 02 expands this into a full harness covering: cross-tenant query, intra-tenant ACL escalation, admin-role attempt, SQL-injection into filter construction, metadata-corruption in ingest, and the RLS-bypass path. This chapter's job is to hand chapter 02 a system that is correctly instrumented to be tested.

## Summary — what you can do after this chapter

- Pick between physical isolation, logical isolation, and hybrid RLS for a RAG deployment, with reasons tied to the specific trade-offs.
- Specify the chunk-record schema a RAG pipeline writes — including `tenant_id`, flattened or resolved-at-query `document_acl`, sensitivity label, and source identifiers — and justify each field.
- Stand up pgvector + Postgres RLS correctly: deny-by-default, `FORCE ROW LEVEL SECURITY`, session-GUC-based caller binding via `SET LOCAL` on each connection checkout, `BYPASSRLS` only on narrow admin roles.
- Explain ACL propagation stage by stage (ingest → chunk → embed → write → retrieve) and name the failure mode at each stage.
- Describe the caller-identity object that the retriever consumes, and state the three properties it must have (authenticated-output, group-membership-included, delegation-recorded).
- Write a cross-tenant negative test that is content-identical and authorisation-distinct.

Chapter 02 picks up the negative-test harness and builds it out to the full battery, including the context-window-leakage shape where tenant A's content surfaces in a completion for tenant B.
