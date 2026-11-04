# mod-110-machine-unlearning-and-dsr: Machine Unlearning and Data-Subject-Rights Engineering for AI

**Estimated effort:** 14 hours

When a subject asks for their data to be removed from an AI system, the engineering problem is not "delete a row". It is: enumerate every corpus the subject appeared in (training, retrieval, logs, derivative products), decide per-corpus whether to retrain, unlearn, or document a defensible no-action, execute the chosen action against the right primitive, verify the result survives an attacker-style probe, and package the whole chain as evidence that a supervisory authority can cross-check without a site visit. This module is where you learn to run that end-to-end.

## Learning objectives

- Engineer the data-subject-rights (DSR) surface for an AI system — right of access (what data is in the training / retrieval / log corpora, what was inferred), right of rectification (correction propagation), right of erasure (removal from active and derivative corpora), right to data portability, right to object; map each to an engineering runbook.
- Engineer SISA-style exact machine unlearning — sharded + isolated training, retraining from checkpoint on deletion — and reason about the training-cost / re-training-latency trade-off; know when SISA is the right investment and when it is not.
- Engineer approximate unlearning — influence-function-based unlearning, Ginart-style approximate methods, Guo et al. certified removal — on tabular and small-model settings; defend the approximation error against an unlearning-verification test.
- Engineer LLM unlearning — gradient ascent on forget sets (Jang et al.), task-vector arithmetic subtraction (Ilharco et al.), Who's-Harry-Potter approximate unlearning (Eldan & Russinovich) — and audit against the TOFU benchmark.
- Engineer the erasure-vs-retraining decision tree — when a retrain is required (clearly PII in a large chunk of training data), when unlearning suffices (small canary set, limited impact on model), when no action is required (data not actually in model, data not personally identifying).
- Engineer DSR against a RAG / agent system — easier than against the base model, harder than against a database — through retrieval-corpus deletion, embedding-store deletion, log purging, and derivative-model refresh.
- Package a DSR audit-trail — request intake (from `ai-governance-analyst`, level 15), engineering action, verification artifact, response to data subject — that survives a supervisory-authority inspection.

## Chapters

1. [The DSR engineering surface and runbooks](01-dsr-engineering-surface-and-runbooks.md) — six GDPR rights, six AI corpora, the six-by-six grid, the clock, the `ai-governance-analyst` hand-off.
2. [SISA — exact machine unlearning](02-sisa-exact-unlearning.md) — Sharded, Isolated, Sliced, Aggregated; the four letters as engineering obligations; cost arithmetic; non-determinism controls; when SISA is and is not viable.
3. [Approximate unlearning — influence functions, certified removal, Ginart algorithms](03-approximate-unlearning-influence-functions.md) — Koh & Liang influence functions, LiSSA HVP, Guo et al. certified removal, Ginart et al. algorithm-aware unlearning, the MIA-based verification test.
4. [LLM unlearning — gradient ascent, task vectors, Who's Harry Potter](04-llm-unlearning-gradient-ascent-and-task-vectors.md) — Jang et al. gradient ascent, Yao et al. KL regularisation, Ilharco et al. task vectors, Eldan & Russinovich reinforced-model + anchor method, TOFU as the standing benchmark.
5. [The erasure-vs-retraining decision tree](05-erasure-vs-retraining-decision-tree.md) — six questions, three leaves (retrain / unlearn / no-action); evidence requirements; multi-corpus coordination.
6. [DSR against RAG and agent systems](06-dsr-against-rag-and-agent-systems.md) — the ten-step erasure cascade: source, chunk, embedding, index, cache, log, agent-memory, derivative, backup, verification.
7. [The DSR audit trail](07-dsr-audit-trail.md) — DSR record schema, per-action sub-records, verification artefacts, subject response, supervisory-authority inspection bundle.

## Exercises

See [`exercises/`](exercises/):

1. [DSR engineering runbook set](exercises/exercise-01-dsr-engineering-runbook-set.md).
2. [SISA exact unlearning implementation](exercises/exercise-02-sisa-exact-unlearning-implementation.md).
3. [Approximate unlearning with influence functions](exercises/exercise-03-approximate-unlearning-with-influence-functions.md).
4. [LLM unlearning with gradient ascent](exercises/exercise-04-llm-unlearning-with-gradient-ascent.md).
5. [TOFU benchmark run and interpretation](exercises/exercise-05-tofu-benchmark-run-and-interpretation.md).
6. [Erasure-vs-retraining decision tree](exercises/exercise-06-erasure-vs-retraining-decision-tree.md).
7. [DSR audit trail authoring](exercises/exercise-07-dsr-audit-trail-authoring.md).

## Structure

- `01-…md` … `07-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs (scaffolded placeholder).
- `quizzes/`: knowledge checks (scaffolded placeholder).
- `resources.md`: external references — the GDPR DSR articles, the EDPB access guidelines, Bourtoule et al. SISA, Koh & Liang / Guo et al. / Ginart et al., Jang et al. / Ilharco et al. / Eldan & Russinovich, TOFU, and the vector-store delete primitives.

## Where this module sits in the track

- Upstream dependencies: mod-101 (frameworks and position — the DSR obligations in the regulatory map), mod-103 (data governance — the data inventory that the DSR runbooks consume), mod-104 (PII detection — relevant for the chunk-content search against free-text corpora), mod-105 (memorisation and MIA — the verification primitives the chapter-03/04 unlearning tests depend on), mod-106 (DP engineering — the (ε, δ) accounting that generalises to certified-removal), mod-109 (RAG privacy — the chunk-record schema and subject-ID propagation this module's chapter-06 cascade depends on).
- Downstream consumers: mod-111 (DPIA — quotes this module's Erasure Posture and per-DSR audit artefacts), mod-112 (program and communication — the subject-facing language and the regulator-facing narrative).
- Hand-off upstream: `ai-governance-analyst` (level 15) runs DSR intake and interpretation; this module produces the engineering artefacts the analyst consumes and references.
