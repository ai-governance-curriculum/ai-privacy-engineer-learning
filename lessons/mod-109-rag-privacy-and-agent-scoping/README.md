# mod-109-rag-privacy-and-agent-scoping: RAG Privacy, Retrieval Access Control, and Agent-Tool Data Scoping

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 14 hours

## Learning objectives

- Design per-tenant retrieval access control for a RAG system — vector-store filter predicates, row-level security on the knowledge base, ACL-propagation through chunking / embedding / retrieval, test-harness that detects privilege-escalation across tenants
- Engineer defence against context-window leakage — the classic attack where documents from tenant A surface in a completion for tenant B because an index was shared; include negative-test fixtures for cross-tenant retrieval
- Design prompt / completion logging policy — what to log, what to redact before log, how long to retain, who can access, how to honour an erasure request against the log; integrate with Presidio on the hot path and with a data-privacy-vault for post-hoc analytics
- Design agent-tool data scoping — when an agent has access to tools that read / write private data (email, calendar, CRM, medical record), engineer the per-tool permission scope, the principal-identity propagation (non-human identity patterns, OAuth-bound-to-user vs. OAuth-bound-to-tool), and the data-boundary review before each tool invocation
- Engineer defence against training-corpus poisoning that smuggles PII into retrieval — document provenance, PII-pre-scan, tombstoning on takedown
- Author the privacy slice of a RAG / agent system card — retrieval scope, logging retention, PII-pipeline position, ACL-propagation claim, tested negative cases
- Cite the boundary to `rag-engineer` (peer, level 25) on the RAG system itself and to `agentic-safety-engineer` (level 40) on the agent-tool authorisation depth beyond privacy

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
