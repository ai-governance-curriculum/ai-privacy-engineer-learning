# exercise-01: Per-Tenant Retrieval ACL Design

**Estimated effort:** 3 hours

## Objective

Design, specify, and partially implement per-tenant retrieval access control for a RAG system of your choice. Produce a design document that a peer `rag-engineer` and a reviewer could implement from, and a runnable pgvector + Postgres row-level-security (RLS) prototype that enforces the predicate at the store level. The exercise exercises the design pattern of chapter 01; the test harness that accompanies it is exercise 02.

## Scenario

You are the privacy engineer on a B2B SaaS that is adding a RAG-answered support experience. Customers are tenants; each tenant has multiple users organised in groups ("engineering", "finance", "support", "executive"). Documents live in the tenant's workspace, each with a `document_acl` listing the groups allowed to read it. Users ask natural-language questions; the RAG returns an answer grounded in retrieved chunks.

The target deployment stack (for the runnable part) is Postgres 16 + pgvector + a Python application using asyncpg. Design choices for the vector store are open — you may justify Pinecone / Weaviate / Qdrant instead, but you should still build the pgvector prototype to make the RLS mechanics concrete.

## Requirements

1. **Isolation-shape decision.** Choose between physical isolation, logical-filter isolation, and hybrid RLS. Justify in writing against (a) the tenant count you expect (assume thousands growing to tens of thousands), (b) the per-tenant data size (assume 10–100 MB at median, up to 10 GB at p99), and (c) the acceptable blast radius of a single developer forgetting the filter.
2. **Chunk-record schema.** Specify the chunk record that will be written to the store, with each field's purpose and sensitivity class. Include the fields chapter 01 named (`tenant_id`, `document_acl`, `sensitivity_label`, source identifiers, timestamps) and anything your deployment adds.
3. **ACL-flattening decision.** Decide between ACL-flattened-at-ingest and group-resolved-at-query, and defend the choice against the staleness-vs-latency trade-off. Document how a group-membership change propagates to the index.
4. **Caller-identity contract.** Write the `AuthenticatedPrincipal` dataclass (or equivalent) your retrieval API will consume. Name the authentication system that produces it and the properties the retriever can assume about each field.
5. **pgvector + RLS prototype.** Build a minimal working prototype:
   - A SQL migration that creates the `chunks` table, the HNSW index, the RLS policy, and `FORCE ROW LEVEL SECURITY`.
   - A Python connection wrapper that sets `app.caller_tenant` and `app.caller_groups` on each checkout via `SET LOCAL`.
   - A seed script that inserts chunks for three fictitious tenants, each with two groups and three documents.
   - A `retrieve(query, caller, top_k)` function that embeds the query (use a stub embedder that returns deterministic vectors derived from a hash of the input so the exercise is reproducible without a running model) and runs the cosine-distance query. No explicit `WHERE tenant_id = ...` in the query — the RLS policy must be the filter.
6. **ABAC extension sketch.** Pick one of: (a) document-sensitivity clearance, (b) jurisdiction filter, (c) erasure block-list. Sketch the predicate extension in RLS. You do not have to implement all three; one is enough to show you understand the composition.
7. **System-card claim.** Write two to four sentences in the shape of the chapter 06 "Retrieval access-control" section, worded so that a reviewer could verify them by opening your prototype.

## Deliverables

Place artefacts in a directory you create under `exercises/exercise-01-per-tenant-retrieval-acl-design-work/`:

- `design.md` — the written design (requirements 1–4 and 6–7).
- `migrations/001_chunks.sql` — the schema and RLS policy.
- `app/db.py` — the connection wrapper with `SET LOCAL` discipline.
- `app/retrieve.py` — the `retrieve()` function and the stub embedder.
- `scripts/seed.py` — the fixture seed script.
- `README.md` — how to run the prototype locally (`docker compose up`, `python scripts/seed.py`, `python -m pytest`, or equivalent).

## Starter guidance

- Default to the hybrid RLS shape unless you can articulate a strong reason to choose otherwise at this tenant scale.
- Use `current_setting('app.caller_tenant', true)` (the second `true` argument tolerates missing-setting and returns NULL) so a developer who forgets `SET LOCAL` gets zero rows rather than an exception. Decide whether this fail-open-to-empty or fail-with-exception is what you want and defend the choice.
- The stub embedder can be `sha256(text) → 1536 floats` or similar — the vector just needs to be deterministic so your seeded fixtures match a test harness. Document the choice.
- `FORCE ROW LEVEL SECURITY` is essential; `BYPASSRLS` on the application role is a fatal mistake. The migration should set these explicitly and a test in exercise 02 will verify them.
- Resist the temptation to over-engineer. The point is the ACL pattern; a 200-line prototype is plenty.

## Acceptance criteria

- The isolation-shape decision in `design.md` names at least three factors and reaches a defensible conclusion.
- The chunk-record schema in `design.md` is a complete field list with per-field sensitivity classes.
- The ACL-flattening decision is made and the staleness vs latency trade-off is explicit.
- The caller-identity dataclass is defined in code and its fields match the retrieval function's expectations.
- The migration enables RLS and `FORCE ROW LEVEL SECURITY` and does not grant `BYPASSRLS` to the application role.
- The connection wrapper uses `SET LOCAL` inside a transaction.
- Running the seed script followed by `retrieve()` from an in-tenant caller returns the expected chunks; running the same `retrieve()` with a caller from a different tenant returns an empty list (the smoke test for the RLS predicate).
- The ABAC extension is sketched for one extension, including the RLS predicate change and the GUC the application would set.
- The system-card claim is a verifiable statement — a reviewer can check its truth from the code.

## Stretch goals

- Implement the exercise in a second store (Pinecone namespace, Qdrant payload filter, or Weaviate multi-tenancy) and write a one-paragraph comparison of how the design differs.
- Add a `sensitivity` extension to the RLS policy with a clearance-based predicate and demonstrate denial of a chunk whose sensitivity exceeds the caller's clearance.
- Add benchmarking: measure query latency with RLS enabled vs a hand-written `WHERE tenant_id = $1` predicate at 100k, 1M, 10M chunks. Report what you find and whether the RLS predicate materially changes the plan.
- Build a tiny admin API endpoint that uses a role with `BYPASSRLS` for a legitimate cross-tenant analytics query, and write the three-line audit-logging rule that must wrap every call to that endpoint.
