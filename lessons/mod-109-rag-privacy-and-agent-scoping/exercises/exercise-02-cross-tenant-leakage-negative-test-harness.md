# exercise-02: Cross-Tenant Leakage Negative-Test Harness

**Estimated effort:** 3 hours

## Objective

Build the six-case negative-test battery of chapter 02 against the RAG prototype from exercise 01, plus a property-based layer and the staging / production-shadow operational wiring. The deliverable is a test suite a reviewer can run locally and a system-card claim about the posture it establishes.

## Scenario

You have the pgvector + RLS prototype from exercise 01. The chunks are seeded, the retriever works for in-tenant callers, and a smoke test confirms a cross-tenant caller gets an empty list. That smoke test is the seed of the harness in this exercise; you will grow it into something that can regress-gate production.

If you skipped exercise 01, fork a minimal pgvector+RLS prototype from a public template or build a stub that exposes the same `retrieve(query, caller, top_k)` function. The exercise's value is in the tests, not in the index underneath.

## Requirements

1. **Case 1 — Cross-tenant retrieval.** A chunk in tenant A, a semantically-identical query from a caller in tenant B. The chunk is not returned. Parametrise over at least three tenant pairs and two query-phrasing variants per chunk.
2. **Case 2 — Intra-tenant ACL escalation.** A chunk in tenant A with ACL excluding the caller's group. The query is identical to the chunk text. The chunk is not returned.
3. **Case 3 — Request-body override ignored.** A request body that includes a `tenant_id` override, an `admin=true` flag, or an alternate principal field. The retriever must consult only the authenticated principal.
4. **Case 4 — Filter-construction injection.** A malicious tenant hint value (`"tenant-B' OR '1'='1"`, a JSON-filter escape, a classic SQL injection) via a request field. The retriever must not surface another tenant's chunk.
5. **Case 5 — Ingest validation.** An attempt to ingest a chunk with a missing `tenant_id` or an empty `document_acl`. The ingest wrapper must raise. Provide positive-case coverage (valid ingest succeeds).
6. **Case 6 — RLS-bypass path (pgvector only).** Three tests that verify `FORCE ROW LEVEL SECURITY` is on, the application role lacks `BYPASSRLS`, and a session without `SET LOCAL app.caller_tenant` returns zero rows (not every row).
7. **Semantic adjacency.** For case 1, show that the query would retrieve the chunk when the filter is *removed* — i.e., comment out the RLS predicate in a temporary test fixture, re-run the query from the wrong tenant, assert the chunk is now in the result. Restore the predicate afterwards. This proves the test is testing the filter, not something else.
8. **Sanity-positive counterpart.** For each negative test, include a positive counterpart: the same chunk retrieved by the correct caller in the correct tenant. The negative test is only credible paired with the positive.
9. **Property-based layer.** A Hypothesis test that generates random `(caller_tenant, caller_groups, chunk_tenant, chunk_acl)` tuples and asserts the retrieval predicate. Run at least 200 examples per CI run.
10. **Context-window-leakage path tests.** At least two tests covering a path *beyond* the retriever: a re-ranker cache key test (prove that two callers in different tenants with identical queries do not collide on cache key) OR a system-prompt-concurrency test (prove that under concurrent requests the system prompts do not leak between threads).
11. **Operational wiring.** Documented (not necessarily executable) plans for:
    - **Pre-merge CI gate** — the exact `pytest` invocation and the jobs it blocks.
    - **Staging canary** — a shape for running the harness against the staging endpoint after each deploy.
    - **Production shadow** — the fixture-tenant approach, the service identity's privileges, and the alerting routing.

## Deliverables

Place artefacts under `exercises/exercise-02-cross-tenant-leakage-negative-test-harness-work/`:

- `tests/privacy/test_case_1_cross_tenant.py`
- `tests/privacy/test_case_2_intra_tenant_acl.py`
- `tests/privacy/test_case_3_body_override.py`
- `tests/privacy/test_case_4_filter_injection.py`
- `tests/privacy/test_case_5_ingest_validation.py`
- `tests/privacy/test_case_6_rls_bypass.py`
- `tests/privacy/test_property_retrieval_predicate.py`
- `tests/privacy/test_context_window_leakage.py`
- `tests/privacy/conftest.py` — fixtures (tenants, chunks, callers, ingest wrapper)
- `README.md` — how to run, how to wire into CI, the staging / shadow sketch.

## Starter guidance

- Use the same seed data as exercise 01, extended with fixtures that demonstrate the specific cases (an ACL-excluded chunk in case 2, a payload chosen for semantic adjacency in case 1).
- Paraphrase chunk text into queries using a short LLM call or hand-authored variants; the bar is "would plausibly retrieve the chunk if the filter were off", not "is a creative paraphrase".
- Case 6c (the no-GUC test) is the subtlest. Use a fresh connection with no `SET LOCAL`, then SELECT and assert empty. If you also want to test the exception shape, use the `current_setting('app.caller_tenant')` form (without the tolerant second argument) in the policy — pick one and document the trade-off.
- Hypothesis settings: `max_examples=200`, `deadline=None` (the DB round-trip varies), and a profile that bumps to `max_examples=1000` for the nightly job.
- The context-window-leakage path tests do not have to use a real re-ranker — a `ReRankerCache` stub whose key logic you control is enough. The point is to assert the key-derivation rule.

## Acceptance criteria

- All six example-based test files exist and pass against the exercise-01 prototype.
- The property-based test passes for `max_examples >= 200`.
- Case 1 includes a demonstration that *with the filter removed* the test fails (via a short helper in `conftest.py` that temporarily drops the RLS policy, re-runs, re-creates it; the pattern is documented in `README.md`).
- Each negative test has a sanity-positive counterpart in the same file.
- The context-window-leakage path tests each cover at least one non-retriever path with assertions.
- `README.md` names the exact `pytest` command the pre-merge gate runs and the staging / production-shadow operational pattern.
- The suite runs in under 60 seconds on a laptop so it does not become a skipped-in-CI item.

## Stretch goals

- Add a mutation-testing step (e.g. [mutmut](https://github.com/boxed/mutmut) or [cosmic-ray](https://github.com/sixty-north/cosmic-ray)) that mutates the retrieval predicate and verifies the suite catches the mutation.
- Build the staging-canary job as a real GitHub Actions / GitLab CI workflow that runs after a simulated deploy.
- Add a case 7: a time-based ACL test where a chunk's ACL expires at a timestamp; verify the predicate honours the expiry.
- Extend case 4 with injection attempts via a JSON-filter payload against a store-SDK call (Pinecone / Qdrant / Weaviate) if your prototype uses one. Document whether the SDK is parametrised safely.
