# Exercise 05: Canary Design and the Cross-Corpus False-Positive Drill

**Estimated effort:** 3 hours · **Output:** `canaries/` directory with a versioned canary set, the generation + collision-check procedure, a planted training run on a small LM, and a dose-response report · **Primary chapter:** 06

## Objective

Produce a defensible canary set and run the end-to-end planting + measurement + reporting cycle on a small open LM. The exercise's point is as much the methodology as the measurement — the cross-corpus FP defence, the shape-matching to threat, and the dose-response curve are the discipline you repeat on every future model.

## Prerequisites

- Chapter 06 (canary design) read.
- Chapter 05 (LLM memorisation / extraction) read; exercise 04 completed (you will be reusing its extraction harness).
- An open LM you can fine-tune. Recommended: GPT-2-small or a Pythia-small checkpoint — small enough to fine-tune on a laptop / single-GPU workstation, large enough to memorise.
- A small fine-tune corpus (open-license, non-sensitive content; e.g. a slice of Wikipedia; a small open dataset you own; or synthetic).

## Problem statement

Produce:

### Part 1 — The canary-generation procedure

Build `canary_gen.py`:

- Generate N (≥ 128) canaries using the five-step procedure from chapter 06:
  1. Format with sufficient entropy — at least two formats: (a) a PII-shaped format (e.g. a plausible-looking SSN or email format, but with digits / tokens generated at high entropy from `secrets.token_*`); (b) a high-entropy format (random 32-hex-char blob) as the collision-free control.
  2. CSPRNG generation — document the exact RNG source.
  3. **Cross-corpus collision check.** Verify each generated canary does not appear in:
     - A sample of Common Crawl (or Pile for Pythia / OpenWebText for GPT-2).
     - The fine-tune corpus itself (outside your deliberate insertions).
     - A search of the open web (web-search API or local index; log the search).
  4. Distribute across insertion-count buckets: (1, 4, 16, 64, 256) insertions, with N / 5 unique canaries per bucket.
  5. Version the canary set; emit `canary_set.v1.json` including the generation seed, the collision-check log, the insertion counts, and a sensitive-storage tag.

### Part 2 — The planting

Build `plant.py`:

- Insert canaries into the fine-tune corpus at the planned insertion counts.
- Document how the insertion is positioned — scattered across the corpus vs. grouped, prefixed with a natural-looking context sentence or not. Reviewers will ask about this; pin a default and record it.
- Fine-tune the base LM on the planted corpus. Record: epochs, batch size, learning rate, random seed.
- Save the fine-tuned checkpoint.

### Part 3 — The measurement

Build `measure.py` (or reuse exercise 04's `extract_known.py` with the canary set as candidates):

- For each canary, prompt the fine-tuned LM with a prefix that would naturally lead to the canary (defined by the format — e.g. for an SSN-shaped canary, "The SSN is " as the prompt).
- Record: exact-match extraction (boolean) and the Secret Sharer exposure metric if you have logit access.
- Repeat with multiple prompt phrasings (at least 3) and record the max-over-phrasings extraction rate.

### Part 4 — The cross-corpus FP drill

Before claiming a measurement is valid:

- For each canary, measure its exposure / probability under the **base model** (before fine-tuning). An ideal canary should have near-zero exposure under the base model.
- Any canary with non-trivial base-model exposure is contaminated by the base corpus — document and exclude.
- Report the "clean" vs. "contaminated" canary counts per insertion bucket; the measurement is only valid on the clean ones.

### Part 5 — The dose-response report

- Plot extraction rate (and exposure) vs. insertion count. Expect an approximately log-linear relationship (Kandpal / Carlini-2023).
- Compute the insertion count at which extraction rate crosses 0.5 — the "memorisation half-life in duplicates".
- Compare PII-shaped canaries vs. high-entropy canaries — does the memorisation rate differ?

### `report.md`

Up to two pages:

- The canary-generation procedure, verbatim.
- The clean vs. contaminated canary counts.
- The dose-response curve + table of extraction rates per insertion bucket per canary shape.
- A "limitations" section naming the four validity threats from chapter 06 (base-corpus collision, dedup inversion, forgetting, alignment / RLHF, prompt mismatch).

## Starter guidance

- **Keep the canary set private.** A leaked canary set invalidates every measurement you do. Store canaries like credentials.
- **Do not use "random-looking" strings — use CSPRNG.** Human-chosen "random" digits collide with public data. `secrets.token_hex(16)` is the baseline.
- **The collision check is where exercises usually cut corners.** Do all three sources: pretraining-corpus sample, fine-tune corpus, open-web search. The 10 minutes it costs saves the whole measurement from being FP-inflated.
- **Shape your canaries to the threat.** If you care about memorisation of customer records, your canaries should look like customer records, not hex blobs.
- **Fine-tune to moderate overfit.** Under-trained fine-tunes will not memorise anything; over-trained ones memorise even at insertion count 1 and the dose-response curve flattens. 3 - 5 epochs is a common defensible default for GPT-2-small.
- **Measure under at least 3 prompt phrasings.** Memorisation is prompt-specific; the max-over-phrasings number is the honest measurement.

## Acceptance criteria

- [ ] Canary set of N ≥ 128 strings, versioned as `canary_set.v1.json`.
- [ ] Two canary shapes produced (PII-shaped + high-entropy control).
- [ ] CSPRNG source documented.
- [ ] Cross-corpus collision check run on all three sources with a log preserved.
- [ ] Planted fine-tune checkpoint produced.
- [ ] Base-model exposure measured for every canary; contaminated canaries excluded.
- [ ] Dose-response curve produced (insertion count ∈ {1, 4, 16, 64, 256}).
- [ ] At least 3 prompt phrasings tested; max-over-phrasings extraction rate reported.
- [ ] Memorisation half-life (insertion count for 50 % extraction) reported.
- [ ] `report.md` names the four validity threats from chapter 06.

## Stretch goals

- **Shape ablation.** Three canary shapes: PII-shaped, document-shaped, code-shaped. Compare dose-response curves — does the model memorise one shape more?
- **Prompt-sensitivity sweep.** Fix one memorised canary; vary prefix length (10, 20, 50 tokens); map the extraction curve.
- **Forgetting trajectory.** Measure extraction rate at every 25 % of training progress; replicate Jagielski et al. 2023's forgetting behaviour.
- **Steinke 2023 one-run audit variant.** For each of your N canaries, flip a coin to plant or hold out; use the per-canary loss to produce an (ε, δ) empirical lower bound; compare to your DP-SGD (ε, δ) upper bound (if you ran chapter 07's DP-SGD arm).
- **Natural-boilerplate inflation check.** Identify the top-N natural duplicates in the fine-tune corpus (boilerplate footers, common phrases); measure their extraction rate; interpret how much of your "extraction rate on real content" is this inflation vs. real sensitive leakage.

## What this lands

- The canary set is the reusable artefact for every future model release — the planting + measurement is a release-gate step.
- The dose-response curve is a required input to exercise 06's defender-triage table (the "extraction rate" column is the number each mitigation moves).
- The cross-corpus FP drill is the procedure a reviewer will ask about by name; having a documented log is the right answer.
- The "memorisation half-life" number is a durable headline for comparing across model sizes and training recipes.

Version the artefacts. Do not publish canaries externally without rotating them. Record who has access to the canary store — it is a sensitive artefact.
