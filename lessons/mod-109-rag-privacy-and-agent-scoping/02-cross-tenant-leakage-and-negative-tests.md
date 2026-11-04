# Context-Window Leakage and the Cross-Tenant Negative-Test Harness

The RAG retriever can be correctly ACL-propagated at the design level and still leak across tenants in production because the test discipline that proves the claim is missing. Correctness of a filter is not a property you can read off the code; it is a property you demonstrate by trying to break it and failing. This chapter is the testing counterpart to chapter 01's design — the suite of *negative* tests that, if they ever pass without the filter, mean the filter is gone.

"Context-window leakage" is the name for the specific failure where tenant A's content ends up in the generator's context window while serving tenant B's request, and surfaces in the completion. The leakage path starts at the retriever and ends at the model; the test harness has to probe every point on that path.

Primary references:

- OWASP — *Top 10 for LLM Applications*, LLM02 and LLM08. [genai.owasp.org/llm-top-10/](https://genai.owasp.org/llm-top-10/) — states the risk explicitly and is the name a reviewer will use.
- MITRE ATLAS — *Adversarial Threat Landscape for AI Systems*. [atlas.mitre.org/](https://atlas.mitre.org/) — cross-reference AML.T0051 (Prompt Injection) and the staging tactics that abuse retrieved context.
- NIST — *AI Risk Management Framework, AI RMF 1.0*, 2023. [nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — the "Measure" and "Manage" functions anchor the regulator-facing posture that includes negative-test evidence.
- pytest documentation. [docs.pytest.org/en/stable/](https://docs.pytest.org/en/stable/) — the fixtures / parameterisation primitives most of the examples use.
- Hypothesis — *property-based testing for Python*. [hypothesis.readthedocs.io/](https://hypothesis.readthedocs.io/) — used for the generative test cases below.

## Why a dedicated negative-test harness

RAG retrievers are typically tested positively ("does the top-k include the right chunk for a given query?"). Positive tests prove recall; they do not prove isolation. The two are different test targets with different coverage shapes. The claim the privacy slice of the system card (chapter 06) makes is of the form "*no cross-tenant chunk was returned under these N classes of adversarial query*"; the evidence for that claim is a running test suite named for the claim.

A good negative-test harness has four properties:

1. **Semantic adjacency.** The queries in the suite are *intentionally close in meaning* to the target chunks. A test where the query is unrelated to the chunk does not prove isolation; it proves the embedding has poor recall.
2. **Deterministic fixtures.** Chunks and queries are built by named fixtures; a reviewer can trace a failing case to a specific source line.
3. **Negative assertion.** The test asserts *absence*: the forbidden chunk is not in the result. Negative assertions have to be strong — if the result is empty, the test still passes; the test suite needs a separate positive case to prove the retriever is not just silently broken.
4. **Flags filter regression.** Each test must fail if someone removes the filter predicate. A test that passes without the filter is a test that is not testing the filter.

## The six-case battery

A production-grade negative-test harness has six baseline cases. The list below is the *minimum*; mature deployments add tenant-specific cases on top.

### Case 1 — Cross-tenant retrieval

The canonical case. A chunk belongs to tenant A; a caller from tenant B issues a semantically identical query; the chunk must not appear.

```python
def test_case_1_cross_tenant(fixtures):
    chunk_a = make_chunk(
        tenant_id="tenant-A",
        document_acl=["group-A-eng"],
        text="Quarterly revenue for FY25 was $42M.",
    )
    ingest([chunk_a])

    caller_b = AuthenticatedPrincipal(
        principal_id="user-b",
        tenant_id="tenant-B",
        group_ids=["group-B-eng"],
    )
    result = retrieve("What was FY25 quarterly revenue?", caller=caller_b, top_k=10)

    assert chunk_a.chunk_id not in [c.chunk_id for c in result]
    assert len(result) >= 0  # explicit; cross-tenant absence is OK even if empty
```

Parameterise this over multiple tenant-pairs, multiple query phrasings per chunk, and multiple top_k values. Hypothesis is useful: generate `(tenant_a_chunk, tenant_b_caller, query_variant)` triples and assert the negative.

### Case 2 — Intra-tenant ACL escalation

Within tenant A, a caller who is not in the chunk's ACL issues a query. The chunk must not appear. This is the harder case: the tenant_id matches, only the ACL predicate rejects.

```python
def test_case_2_intra_tenant_acl(fixtures):
    chunk = make_chunk(
        tenant_id="tenant-A",
        document_acl=["group-A-finance"],
        text="Executive compensation review for Jane Doe.",
    )
    ingest([chunk])

    caller = AuthenticatedPrincipal(
        principal_id="user-a-eng",
        tenant_id="tenant-A",
        group_ids=["group-A-eng"],  # not finance
    )
    result = retrieve("Jane Doe compensation", caller=caller, top_k=10)

    assert chunk.chunk_id not in [c.chunk_id for c in result]
```

### Case 3 — Admin-role escalation attempt from the request body

The gateway authenticates the caller; the request body also carries a `tenant_id` field or an `admin=true` flag. The retriever must ignore the body and use the authenticated principal.

```python
def test_case_3_body_tenant_ignored(fixtures):
    chunk_a = make_chunk(tenant_id="tenant-A", ...)
    ingest([chunk_a])

    caller_b = AuthenticatedPrincipal(principal_id="user-b", tenant_id="tenant-B", ...)

    # Simulate a malicious client that passes tenant_id="tenant-A" in the body
    malicious_request = RetrieveRequest(
        query="...",
        tenant_id_override="tenant-A",   # should never be read
        admin=True,
    )
    result = retrieve_from_request(malicious_request, caller=caller_b, top_k=10)

    assert chunk_a.chunk_id not in [c.chunk_id for c in result]
```

The test fails if the retriever has a code path that consults the body for the tenant_id. It is worth writing because — in a codebase of any age — a developer will eventually add an "admin override" for a "quick fix" and forget it was there.

### Case 4 — Filter-construction injection

If the retrieval filter is built as a string (e.g. for a hand-written SQL WHERE clause, or a JSON filter concatenated from fragments), an attacker-controlled value in the query parameters can escape the filter. The test attempts a value designed to break the quoting.

```python
@pytest.mark.parametrize("malicious_query_tenant", [
    "tenant-B' OR '1' = '1",
    "tenant-B\" , \"$or\": [{\"tenant_id\": \"tenant-A\"}]",
    "tenant-B'); DROP TABLE chunks; --",
])
def test_case_4_filter_injection(fixtures, malicious_query_tenant):
    chunk_a = make_chunk(tenant_id="tenant-A", ...)
    ingest([chunk_a])

    # The caller is honest tenant-B but the body has a crafted field some
    # earlier version of the code inlined into the filter.
    request = build_request_with(tenant_hint=malicious_query_tenant)
    caller = AuthenticatedPrincipal(tenant_id="tenant-B", ...)
    result = retrieve_from_request(request, caller=caller, top_k=10)

    assert chunk_a.chunk_id not in [c.chunk_id for c in result]
```

This case exists even if your retrieval layer is "SDK-only" (Pinecone, Weaviate, Qdrant), because filter construction is still string-manipulation at some point if your code takes user input as a filter value. Parametrised SDK calls are the mitigation; this test is the proof.

### Case 5 — Metadata corruption in ingest

A chunk is ingested with the wrong metadata — missing `tenant_id`, empty `document_acl`, or a `tenant_id` from a different tenant. The system should reject the ingest, not store the chunk silently.

```python
def test_case_5a_ingest_rejects_missing_tenant(fixtures):
    bad_chunk = ChunkPayload(
        text="...",
        tenant_id=None,       # missing
        document_acl=["group-A-eng"],
    )
    with pytest.raises(InvalidChunkError):
        ingest([bad_chunk])

def test_case_5b_ingest_rejects_empty_acl(fixtures):
    bad_chunk = ChunkPayload(
        text="...",
        tenant_id="tenant-A",
        document_acl=[],       # a chunk with no ACL is a chunk no one can read
    )
    with pytest.raises(InvalidChunkError):
        ingest([bad_chunk])
```

The ingest-side wrapper is the natural place to enforce this. The test guards the wrapper against regression.

### Case 6 — RLS-bypass path (pgvector + Postgres only)

For pgvector deployments, the RLS policy can be bypassed in three concrete ways: the table owner (if `FORCE ROW LEVEL SECURITY` is not set), a role with the `BYPASSRLS` attribute, and a transaction with `SET ROLE` to such a role. The test covers each.

```python
def test_case_6a_force_rls_enabled(db):
    row = db.fetch_one("""
        SELECT relforcerowsecurity, relrowsecurity FROM pg_class
        WHERE relname = 'chunks'
    """)
    assert row["relforcerowsecurity"] is True
    assert row["relrowsecurity"] is True

def test_case_6b_app_role_has_no_bypassrls(db):
    row = db.fetch_one("""
        SELECT rolbypassrls FROM pg_roles WHERE rolname = 'rag_app'
    """)
    assert row["rolbypassrls"] is False

def test_case_6c_app_role_without_tenant_guc_returns_nothing(db):
    # No SET LOCAL app.caller_tenant — the predicate evaluates to NULL and
    # the policy rejects the row.
    with db.app_role_connection() as conn:
        result = conn.fetch("SELECT * FROM chunks LIMIT 1")
    assert result == []
```

Case 6c is subtle and worth staring at. `current_setting('app.caller_tenant', true)` returns NULL when the GUC is unset (the `true` second argument asks for a missing-setting-tolerant lookup); a comparison against NULL is NULL, and a NULL predicate rejects the row. The test *proves* that a developer who forgets the `SET LOCAL` gets zero rows, not every row. The alternative (strict `current_setting` that throws) also works, as long as the test exercises the correct failure mode.

## Semantic adjacency — building queries that would retrieve the forbidden chunk

The hardest part of writing case-1 tests well is producing queries that *would* match the forbidden chunk if the filter were removed. If the query is semantically unrelated, the test passes for the wrong reason. Three tactics:

- **Paraphrase the chunk.** Feed the chunk text to a small paraphrasing model (or a prompt to the generator model) and use the paraphrase as the query. A well-paraphrased query has near-identical embedding to the chunk; the retriever would score it in the top-1 if the filter were off.
- **Use the chunk as the query.** Degenerate but effective: the chunk itself, used verbatim as the query, is the strongest positive match. If this test does not retrieve the chunk (with the filter off), your embedder is broken.
- **Craft a known-high-recall probe.** For a specific class of document (e.g. medical records), there is often a recognisable query shape that recalls them. Use that shape, phrased for tenant B, and show the forbidden chunks from tenant A do not appear.

The harness should ship with a "sanity-positive" companion that proves the same chunks *are* retrieved when the caller is correctly entitled. Without that, a passing negative test is consistent with the entire retriever being broken.

```python
def test_sanity_positive_counterpart(fixtures):
    chunk_a = make_chunk(tenant_id="tenant-A", document_acl=["group-A-eng"], text="...")
    ingest([chunk_a])

    caller_a = AuthenticatedPrincipal(tenant_id="tenant-A", group_ids=["group-A-eng"])
    result = retrieve("the exact topic", caller=caller_a, top_k=10)
    assert chunk_a.chunk_id in [c.chunk_id for c in result]
```

The pair of tests — negative for the wrong caller, positive for the right caller — is what the system card can honestly claim.

## Context-window leakage past the retriever

Even with a correct retriever, a chunk from tenant A can reach tenant B's completion by other paths. The negative-test harness must cover them.

### Path 1 — Shared re-ranker caches

A re-ranker (cross-encoder, hybrid BM25 combiner) that caches query-scored-chunk pairs across tenants will, on a lucky hash collision, return another tenant's chunk. The mitigation is tenant-scoped cache keys. The test probes the cache key.

### Path 2 — Shared generation KV cache

Many serving stacks (vLLM, TGI) cache the KV tensors from the *prompt prefix* for throughput. If the prompt prefix contains the caller's identity but the cache is keyed only on the token sequence, a crafted request with matching prefix tokens can inherit another caller's KV state. The test is: two callers with identical first-N tokens must not share a KV cache entry; verify by instrumenting the cache lookup to require a tenant-id tag in the key.

### Path 3 — System-prompt staleness between requests

A request handler that constructs a per-request system prompt by *mutating* a module-level prompt object will, under concurrency, serve the wrong prompt. This is a generic server-side bug but it shows up in RAG serving stacks regularly. The test: run concurrent requests from tenant A (with "do not cite finance") and tenant B (with no restriction); assert that A's constraint never leaks to B.

### Path 4 — Document-chunk recombination in the prompt template

A prompt template that concatenates retrieved chunks without per-chunk tenant tagging will produce an undifferentiated context. If, through any of the above paths, a cross-tenant chunk slips in, the model cannot distinguish it from an in-tenant chunk. The design mitigation: tag each chunk in the prompt with its origin and instruct the model to refuse content without a matching origin. This is weak (the model can be prompt-injected away from the instruction) but it is a defence-in-depth layer. The test is a red-team: inject a cross-tenant chunk via a fixture bypass and assert the pipeline detects the tag mismatch before generation.

## The property-based layer

Case-1 through case-6 are example-based; they prove the system handles the examples. A production harness adds a property-based layer that generates random tenants, random ACLs, and random queries and asserts the invariants.

```python
from hypothesis import given, strategies as st

@given(
    caller_tenant=st.sampled_from(["T1", "T2", "T3"]),
    chunk_tenant=st.sampled_from(["T1", "T2", "T3"]),
    caller_groups=st.lists(st.sampled_from(["g1", "g2", "g3"]), min_size=1, max_size=3),
    chunk_acl=st.lists(st.sampled_from(["g1", "g2", "g3"]), min_size=1, max_size=3),
)
def test_property_retrieval_matches_predicate(
    caller_tenant, chunk_tenant, caller_groups, chunk_acl
):
    chunk = make_chunk(tenant_id=chunk_tenant, document_acl=chunk_acl)
    ingest([chunk])
    caller = AuthenticatedPrincipal(tenant_id=caller_tenant, group_ids=caller_groups)
    result = retrieve("the chunk's own text", caller=caller, top_k=10)

    should_retrieve = (
        caller_tenant == chunk_tenant
        and bool(set(caller_groups) & set(chunk_acl))
    )
    if should_retrieve:
        assert chunk.chunk_id in [c.chunk_id for c in result]
    else:
        assert chunk.chunk_id not in [c.chunk_id for c in result]
```

Property tests find the cases the human tester did not think of. In a mature suite they become a required pre-merge check on the retriever package.

## Running the harness in CI and in production

Three operational discipline items:

- **Pre-merge gate.** The harness runs on every pull request touching the retriever, the chunker, the ingest, or the vector-store SDK wrapper. A failing negative test blocks merge.
- **Staging canary.** The harness runs against the staging environment after each deploy, hitting the deployed retriever endpoint rather than the unit-test fake. This catches configuration drift (e.g. a staging index whose RLS is disabled).
- **Production shadow.** A small, read-only variant of the harness runs on a schedule against the production deployment, using fixture tenants that are known not to serve real users. A production regression (an admin turned off RLS to debug something and forgot to turn it back on) is detected in minutes rather than when a customer discovers it.

The shadow test runs under a service identity that has normal application privileges (no `BYPASSRLS`). It proves the live production retriever behaves correctly for an unprivileged caller. Its failure is paged.

## What to put in the system card

Chapter 06 is the full system-card template; the retrieval-isolation claims that land there, derived from this harness:

- "The retriever enforces per-tenant isolation via {RLS / namespace / payload filter} as described in §{N}."
- "A negative-test suite of {M} cases covering cross-tenant retrieval, intra-tenant ACL escalation, request-body override, filter injection, metadata corruption, and RLS bypass runs on every pre-merge build and on each production deploy; the current build state is {green / failing on {case} since {date}}."
- "Property-based testing covers {K} randomised scenarios per run."
- "Production shadow testing runs every {N} minutes; last successful run was at {timestamp}."

A reviewer who reads the claim will ask to see the suite. Keep the test file names and the claim's phrasing aligned; `test_case_1_cross_tenant` under a `tests/privacy/` directory is a defensible artefact.

## Summary — what you can do after this chapter

- Enumerate the six baseline negative-test cases for a RAG retriever (cross-tenant, intra-tenant ACL, body-override, filter injection, ingest corruption, RLS bypass) and write each one.
- Build semantically adjacent queries that would retrieve the forbidden chunk if the filter were removed, using paraphrase / self-query / known-recall-probe tactics, with a sanity-positive counterpart.
- Extend the harness to context-window leakage paths beyond the retriever: re-ranker caches, KV caches, system-prompt mutation, and in-prompt tenant tagging.
- Add a property-based layer that generates random (caller, chunk) pairs and asserts the retrieval predicate.
- Wire the harness into pre-merge CI, post-deploy staging canary, and production shadow runs with paged-on-failure alerting.
- Phrase the system-card claim about retrieval isolation in a form a reviewer can verify by opening the test file.
