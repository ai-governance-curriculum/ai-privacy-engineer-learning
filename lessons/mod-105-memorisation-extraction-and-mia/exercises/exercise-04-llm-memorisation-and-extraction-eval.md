# Exercise 04: LLM Memorisation and Extraction Evaluation

**Estimated effort:** 4 hours · **Output:** `extraction/` directory with the measurement harness, the k-extractable rate reports for an open-weights model, the attacker-view extraction run, and (optional) a black-box production-API eval · **Primary chapter:** 05

## Objective

Build an end-to-end LLM memorisation / verbatim-extraction measurement. The goal is to produce a defensible extraction-rate report against an open-weights LM with published training data, learn the ranking-signal discipline from Carlini-2021, and produce the shape of an API-side extraction eval that fits your production threat model.

## Prerequisites

- Chapter 05 (LLM memorisation and extraction) read.
- Chapter 06 (canary design) read — this exercise uses canaries from exercise 05 as part of the measurement.
- At least one open-weights LM you can run locally. Recommended default: **Pythia-410M / 1.4B** (EleutherAI, trained on the Pile — training data published). Alternatives for the known-training-set path: **GPT-2 small / medium** (OpenWebText training data).
- Optional: an API account on a hosted LLM provider for the black-box eval (your organisation's own hosted model is preferable to a third-party provider for ethics reasons).
- Read-access to a sample of the model's published training corpus (Pile for Pythia; OpenWebText for GPT-2) sufficient to pick candidate strings.

## Problem statement

Produce:

### Part 1 — Known-training-set extraction (open-weights Pythia)

Build `extract_known.py`:

- Sample N ≥ 1000 candidate training-set strings from the published training data. Stratify by: insertion count in the training corpus (if Carlini-2023 metadata available; otherwise sample uniformly and compute frequencies post-hoc on your sample), text length, content type (prose / structured / code / boilerplate).
- For each candidate, split into prefix (k = 50 tokens) and continuation (n = 50 tokens).
- Prompt the model with the prefix under greedy decoding; compare the generated continuation to the ground-truth.
- Record match / no-match + the model's perplexity on the continuation.
- Normalise comparison: lowercasing, whitespace, Unicode — document the exact normalisation.

Report:

- Overall k-extractable rate.
- Rate stratified by insertion count (if available) or by candidate type.
- Length and content-type sub-stratifications.
- A sample of extracted strings (up to a safe-to-publish subset).

### Part 2 — Attacker-view extraction (Carlini-2021 methodology)

Build `extract_attacker.py`:

- Sample ≥ 10,000 generations from the Pythia model using diversity-encouraging sampling (temperature 1.0 or above; top-p 0.9; or the "web-text-prompt" strategy from the paper).
- Score each generation by at least two ranking signals:
  - Perplexity-ratio: perplexity_under_target_model / perplexity_under_a_different_baseline_model (e.g. a smaller Pythia or a smaller open LM).
  - Zlib-compression ratio: perplexity_under_target / zlib_compressed_length.
- Rank generations; take the top-K candidates (K = 100 is a workable size).
- For each top-K candidate: verify against the published training corpus (text-match or near-match search). Record: how many are confirmed memorisation vs. coincidence vs. manual-rejection.
- Classify any memorised candidate by content type (PII-shaped, URL, code, known public text).

Report:

- Precision of the top-K ranking (fraction that are actually memorised training data).
- Content-type histogram of confirmed memorisations.
- The ranking-signal ablation (perplexity-ratio alone vs. zlib alone vs. both; whichever gives the highest precision).

### Part 3 — Black-box production-API extraction (optional but encouraged)

If you have access to your organisation's hosted LLM (or a third-party model where your organisation has an approved red-team engagement):

- Build a probing set of **known shingles** that the model was likely trained on or fine-tuned with. Candidates: your organisation's public website, published docs, any labelled-sensitive training corpus from mod-103.
- Build a set of **alignment-breaking / divergence prompts** inspired by Nasr et al. 2023 (avoid publishing the specific prompts; keep the set versioned in your repo).
- Build a set of **benign control prompts** to measure the false-positive rate of the extraction detection.

Run each set against the model and report:

- Fraction of known-shingle prompts whose completion matched the training shingle (exact / approximate).
- Fraction of alignment-breaking prompts whose completion contained ≥ 50 tokens of apparently-training-set content (manual-verified).
- False-positive rate from the benign control set.

Ethical note: do not run alignment-breaking prompts against a third-party production model without the provider's explicit approval. If the engagement is internal, run under an incident-response plan — unexpected successful extractions are an incident and must be routed to the privacy incident process on the spot.

## Starter guidance

- **Pythia is the right first target.** The Pile is published, Pythia checkpoints are published at every scale, and Carlini-2023's measurements are on Pythia — so your numbers are comparable to the literature.
- **Normalise comparisons carefully.** A single mismatched whitespace token kills an exact-match extraction. Pick a normalisation rule and document it — "exact match after NFC normalisation + whitespace collapse + case preservation".
- **Sample sizes matter.** 1000 candidates is a credible headline; 10 is a spike-check. Carlini-2023 tables typically involve hundreds of thousands.
- **Stratify by duplicate count.** Kandpal-2022 showed this is the dominant predictor. Plot the dose-response curve: extraction rate vs. observed duplicate count in the training corpus.
- **Canary inserts belong in exercise 05.** Canaries are controlled experiments; the natural-training-corpus extraction here is observational.
- **Never publish extracted PII.** When sample extractions are shared externally, redact. The memorisation measurement is an internal artefact; the headline is a rate.

## Acceptance criteria

- [ ] Part 1 (known-training-set) run on Pythia or GPT-2 with N ≥ 1000 candidates.
- [ ] k-extractable rate reported overall + stratified by at least one axis (duplicate count OR content type OR length).
- [ ] Part 2 (attacker-view) run with ≥ 10,000 generations.
- [ ] Top-K ranking produced with at least two ranking signals; precision of the ranking reported.
- [ ] Decoding policy, k (prefix length), n (continuation length), and comparison normalisation are explicitly documented.
- [ ] Sample of extracted content included — redacted where it contains PII.
- [ ] If Part 3 attempted, ethical clearance and incident-response plan documented before execution.
- [ ] Methodology appendix names all eight axes from chapter 05 ("Reporting the extraction measurement in a form a reviewer trusts").

## Stretch goals

- **Pythia scaling study.** Repeat Part 1 on Pythia-70M, 410M, 1.4B, 2.8B, 6.9B. Reproduce qualitatively the Carlini-2023 result that extraction rate grows with parameter count.
- **Checkpoint trajectory.** Run Part 1 on Pythia's intermediate checkpoints (the project publishes 143 checkpoints per model). Memorisation across training is visible and is useful for chapter 06's Jagielski-2023 forgetting discussion.
- **Baseline-model ablation.** Vary the baseline model in the perplexity ratio (another Pythia vs. GPT-2 vs. a small LLaMA). Report which baseline gives the highest ranking precision and interpret why.
- **Prompt-sensitivity sweep.** For a single known-memorised string, vary the prefix (10, 20, 50, 100 tokens). Extract-vs-prefix-length curve.
- **Stretch into fine-tune memorisation.** Fine-tune Pythia on a small dataset you created, re-measure extraction rate on the fine-tune corpus, compare to the pre-trained baseline. Preview for exercise 05's canary insertion.

## What this lands

- The extraction-rate measurement is the module's LLM-side headline — it feeds the coverage matrix (exercise 06, chapter 09) and the defender-triage table (exercise 06).
- The ranking-signal results are reusable methodology for any future LLM audit.
- The Part 3 shape (if executed) is directly reusable as the standing LLM extraction eval in your release pipeline.
- The redacted extraction samples go into the DPIA (mod-111) as evidence.

Version the harness. Pin the model checkpoints, the decoding seeds, and the normalisation rule. Do not publish extracted PII — redact in-flight.
