# Canary Design and the Cross-Corpus False-Positive Problem

Canary methodology is the control that makes the chapter 05 extraction measurement a *controlled experiment* rather than an uncalibrated observation. Insert strings you designed into the training corpus at known counts, measure how many you recover after training, and you have a per-model dose-response curve. Done badly, canary methodology silently inflates its own numbers by counting "memorisation" that is really the model emitting something from its base-pretraining corpus, not your fine-tune corpus. This chapter is the engineering reference to canary design that survives a reviewer.

Primary references:

- Carlini, Liu, Erlingsson, Kos, Song — *The Secret Sharer: Evaluating and Testing Unintended Memorization in Neural Networks*. USENIX Security 2019. [arxiv.org/abs/1802.08232](https://arxiv.org/abs/1802.08232) — the original canary methodology and the *exposure* metric.
- Jagielski, Thakkar, Tramer, Ippolito, Lee, Carlini, Wallace, Song, Thakurta, Papernot, Zhang — *Measuring Forgetting of Memorized Training Examples*. ICLR 2023. [arxiv.org/abs/2207.00099](https://arxiv.org/abs/2207.00099) — on memorisation stability across training and its implications for canary measurement.
- Carlini, Ippolito, Jagielski, Lee, Tramèr, Zhang — *Quantifying Memorization Across Neural Language Models*. ICLR 2023. [arxiv.org/abs/2202.07646](https://arxiv.org/abs/2202.07646) — the Pythia-based canary study; defines the k-extractable metric this measurement produces.
- Steinke, Nasr, Jagielski — *Privacy Auditing with One (1) Training Run*. NeurIPS 2023. [arxiv.org/abs/2305.08846](https://arxiv.org/abs/2305.08846) — single-run privacy auditing via planted canaries; the state of the art for efficient empirical audit.

## What a canary is

A **canary** is a synthetic record inserted into the training corpus by the privacy engineer. It has three properties:

- **Known to the engineer, unknown elsewhere.** Nobody else has seen this string, so if the model emits it, the source must be training.
- **Shaped like a sensitive secret.** The canary mimics the *shape* of what you would be worried about memorising — a credit card, an SSN, a rare personal identifier — so that memorisation of the canary is evidence of memorisation of similarly shaped real content.
- **Scored by an exposure metric or a direct-extraction test.** Either you check whether a prompt elicits the canary (extraction rate; chapter 05) or you use the Secret Sharer *exposure* metric (log-rank of the canary among all same-format strings — a continuous measure).

Canary design is the bridge between the extraction methodology (chapter 05) and the defender triage (chapter 07). Good canaries let you detect memorisation quantitatively and act on the dose-response curve.

## The Secret Sharer exposure metric

Carlini-2019's foundational idea. Define a *canary format* (regular-expression-like template): e.g. `"My SSN is XXX-XX-XXXX"` where the Xs are randomly chosen digits. Given a specific instantiated canary `c` (e.g. `"My SSN is 384-21-9573"`), its *rank* among all valid completions of the format is:

```
rank(c, M) = |{ c' matching the format : P_M(c') ≥ P_M(c) }|
```

and the *exposure* of c in M is:

```
exposure(c, M) = log2(|{all c' matching format}|) − log2(rank(c, M))
```

Exposure is bounded in `[0, log2(N)]`. Exposure near 0 means the canary is no more likely than a random same-format string (not memorised). Exposure approaching log2(N) means the canary is the *most likely* string in the format (fully memorised).

Why exposure, not just "did the prompt elicit the canary":

- **Continuous.** Lets you plot a dose-response curve as training progresses — exposure trajectory over epochs.
- **Robust to prompt choice.** You do not need a magic-prompt that elicits the exact string; you compare probabilities over the format space.
- **Applies before the model is "extractable".** A canary with exposure 15 is a canary the model has learned, even if a specific prompt would not elicit it; the continuous signal arrives before the extraction threshold.

A production methodology ships both — exposure for the continuous research measurement, direct extraction for the human-readable "the model emits this" measurement.

## The cross-corpus false-positive problem

The quiet way canary-based measurements lie: the model emits the canary not because it memorised your inserted canary, but because the *same string* already appeared in the model's base-pretraining corpus. The model learned the string there, and your insertion was never responsible for the memorisation at all.

Concrete failure modes:

- **"Random" strings that are not random.** `"4532 4321 1234 5678"` passes a format check for credit card but might appear in a public list of test-card numbers. The model saw it in pretraining. Your canary emission is a false positive.
- **Low-entropy formats that collide across corpora.** `"John Smith, 123 Main Street"` instantiations collide with billions of near-identical strings in Common Crawl. Any memorisation of "this format" counts as memorisation of your canary.
- **Standard test values.** `"555-12-3456"` (an often-used obvious test SSN) appears in every tutorial, textbook, and placeholder-generator on the web.
- **Naturally-occurring identifiers.** A canary email `"jane.smith@example.com"` collides with every example email on the open web.

The *measurement harm*: extraction rate numbers get reported higher than the true memorisation rate, the DP-SGD mitigation looks less effective than it is, deduplication and redaction get over-applied. In the reverse direction — *lower* than true — if the base corpus happened to contain a string that now looks like your canary and the deduplicator removed it, you can under-measure.

### The defence — a documented canary-generation procedure

A canary-generation procedure that a reviewer accepts has five steps:

1. **Choose the format with enough entropy.** The number of valid instantiations of the format should be large enough that no instantiation would plausibly appear on the public web. For a 9-digit SSN format, 10⁹ instantiations — not large enough to guarantee no collision. For a format like `"<rare-prefix> <32-random-hex-chars>"`, 2¹²⁸ instantiations — collision-free by cryptographic argument.
2. **Generate with a CSPRNG.** `secrets.token_hex` or equivalent, seeded from `os.urandom`. Do not generate "human-looking random" strings; those are not random enough to clear the base corpus.
3. **Cross-corpus collision check.** For each generated canary, verify it does not appear in:
   - A sample of Common Crawl (or the model's pretraining corpus if you have it — Pile for Pythia).
   - Your organisation's internal corpora (logs, docs, tickets).
   - The fine-tune / training corpus itself, outside your deliberate insertions.
   - A reasonable sample of indexed public web via a search-API query. (Logging the search is important if you later need to defend "we did check".)
4. **Insert at planned counts.** Decide the dose-response grid — e.g. (1, 4, 16, 64, 256) insertions — and insert each canary at exactly its planned count.
5. **Lock the canary set to a version.** The canary set is an artifact — version it, treat it as sensitive (anyone who sees the canary strings can trivially inflate the model's extraction rate), store it separate from the training corpus.

Step 3 is the step most teams skip; it is the step the reviewer asks about.

## Shape your canaries like the threat

The canary's *shape* matters for the measurement's generalisability. If you want the extraction rate to be informative about memorisation of **real sensitive data in your corpus**, the canary should look like your real sensitive data.

Design patterns:

- **PII-shaped canaries.** Fake-but-unused SSNs / phone numbers / emails / addresses — same character distribution, same length, same formatting as real PII. Measurement generalises to the real-PII memorisation rate.
- **Document-shaped canaries.** For a legal-document fine-tune, canary paragraphs that look like contract text with a random identifying phrase embedded. Measurement generalises to real-document memorisation.
- **Code-shaped canaries.** For a code fine-tune, canary function definitions with random-identifier variables that still look syntactically valid. Measurement generalises to real-code memorisation (this is the shape of the real-world Codex / Copilot memorisation concern).
- **Combination canaries.** Multi-field records that mimic the structure of your training data — a customer record, a medical note, a chat transcript. Measurement generalises directly to the thing you care about.

Avoid canaries that are *structurally unlike* your data. A 128-char hex blob is cryptographically safe from base-corpus collision but is not a shape the model's memorisation of "real customer records" would generalise to. Use the hex-blob canary as the *collision-free upper bound* and the PII-shape canary as the *generalisable measurement*.

## Insertion-count design

The dose-response curve is the deliverable. Choose insertion counts to get a readable curve at your training scale:

- A reasonable default grid: `(1, 2, 4, 8, 16, 32, 64, 128, 256, 1024)` insertions per canary, with 10–50 unique canaries per count.
- For a small fine-tune corpus (10k–100k records), start lower — 1 to 64 may already saturate.
- For a very large pre-training corpus (hundreds of billions of tokens), you may need higher counts — even 100 insertions can be lost in the noise.

Report the full curve and the insertion count at which extraction rate crosses 50 %. That crossover is the headline number — "memorisation half-life in duplicates".

## The one-run audit — Steinke et al. 2023

The Carlini-2019 methodology needs multiple training runs (canaries inserted / held out) to establish statistical significance against noise. Steinke, Nasr, and Jagielski (NeurIPS 2023) showed that a single training run with *many* canaries — some inserted, some held out — gives a tight (ε, δ) lower-bound estimate at a fraction of the compute cost.

This is the methodology you ship when training a large model and cannot afford a second full training run for the audit. Chapter 07 names it in the defender triage. The paper provides the auditing recipe; exercises 05 and 06 implement variants of it.

## Scoring canaries in practice — three modes

Pick one or more, document which:

- **Exposure (Secret Sharer).** Continuous, needs probability access. Use when you have logit access to the target model.
- **Direct extraction.** Boolean — "did the prompt elicit the canary verbatim?". Use when you only have black-box generation access.
- **Canary pair ratio (Steinke et al.).** For each canary, you planted either 1 copy or 0 copies; a classifier predicts insertion from the model's loss on the canary. Use when efficient (ε, δ) auditing is the goal.

For a defensible production measurement, ship at least exposure and direct extraction — the former is the continuous research signal, the latter is the human-readable number that gets quoted.

## Threats to canary validity you need to document

A canary measurement can lie in both directions. Name the threats in the measurement's "limitations" section:

- **False positives from base-corpus collision.** Mitigated by the five-step procedure above. Measure residual FP rate by scoring canaries *before* inserting them — a canary that the model already assigns non-zero exposure to in the base model is contaminated.
- **False positives from deduplication inversion.** If dedup removed canaries you planted (dedup is cross-checking on suffix hashes), the inserted counts you think you planted are wrong. Verify the dedup pipeline does not remove canaries (either whitelist them or verify post-dedup).
- **False negatives from forgetting.** Memorised canaries can be partially forgotten in later training phases (Jagielski et al.). Measure at the final checkpoint and at one or two earlier checkpoints — disclose the trajectory.
- **False negatives from alignment / RLHF.** On an aligned model, the alignment pass can suppress emission of training strings even though they are still in the weights. Measure *before* the final alignment pass if you have access; otherwise measure under the Nasr-2023 alignment-breaking prompts (chapter 05) as well as under benign prompts.
- **Canary-prompt mismatch.** Memorisation is prompt-specific. Report under multiple prompt phrasings; the headline is the maximum over phrasings.

## Exercise bridge

Exercises 05 and 06 together land the canary discipline:

- **Exercise 05 — canary design and cross-corpus false-positive drill.** Build a canary generator with the five-step procedure and document a residual FP-rate measurement against a base-pretraining corpus and your real-world content.
- **Exercise 06 — defender triage and mitigation sweep.** Insert the canary set into a training run, measure extraction rate pre- and post-mitigation (deduplication, DP-SGD, output filter), produce the defender triage table.

## Summary — what you can do after this chapter

- Define a canary with the three properties (known-to-engineer, shape-matched-to-threat, scored by exposure or direct extraction).
- Compute and interpret Secret Sharer exposure as a continuous memorisation signal.
- Execute the five-step canary generation procedure (format entropy, CSPRNG, cross-corpus check, planned insertion counts, versioned artefact).
- Design the canary shape to match the threat so the measurement generalises to real content.
- Pick a dose-response insertion grid and report the memorisation-half-life number.
- Ship a canary measurement with the four named validity threats called out.

Chapter 07 is the defender-triage view that consumes the measurement. Exercise 06 is where the two meet.
