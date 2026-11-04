# mod-105-memorisation-extraction-and-mia: Memorisation, Extraction, and Membership-Inference Risk Engineering

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 18 hours

## Learning objectives

- Engineer shadow-model membership-inference attacks (Shokri et al.) and Carlini-style likelihood-ratio attacks (LiRA) at production quality — construct shadow training sets, train shadow models, fit the attack model, and interpret the AUC + TPR-at-fixed-FPR as a residual-privacy-risk signal
- Engineer attribute-inference attacks and Fredrikson-style model-inversion attacks on tabular / vision / small-language models; document the inversion fidelity against a defensible threshold
- Engineer LLM memorisation / verbatim-extraction evaluations — Carlini extraction methodology, Quantifying Memorization canary methodology — on open-weights models (GPT-2, Pythia, Llama) and in the black-box API setting; defend the canary design against cross-corpus false positives
- Reason about PET-side defences and when they move the needle on MIA AUC — DP-SGD at (ε, δ) budgets, per-example clipping, deduplication of training data, limit on epochs; produce a defender's triage table
- Operate ML Privacy Meter as the working evaluation harness end-to-end — target-model wrapping, attack configuration, result interpretation, plot generation — and link out to `model-evaluation-engineer` (peer, level 30) for the confidence-interval methodology on MIA AUC
- Map every attack in the suite back to a NIST AI 100-2 category and a MITRE ATLAS tactic + OWASP LLM Top 10 entry (LLM02 Sensitive Information Disclosure) so the coverage story is defensible
- Cite the boundary to `ai-risk-engineer` (peer, level 25) mod-107 — that module covers Shokri MIA, LiRA, and memorisation at working depth; this module goes production-grade on methodology, defender triage, and ML Privacy Meter

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
