# Defender Triage — DP-SGD, Deduplication, Clipping, Epoch Cap, and Reading the AUC Delta

The chapters so far ship the measurement. This chapter ships the decision table — when a given mitigation moves the measured attack metrics, how much, at what engineering cost. The defender's question is not "which mitigation is most private?" but "which mitigation gets my LiRA TPR@0.001 below the gate without making my model unusable?". The triage table is the artifact the DPIA, the release-assurance review, and the regulator reads to see you made that decision deliberately.

Primary references:

- Abadi, Chu, Goodfellow, McMahan, Mironov, Talwar, Zhang — *Deep Learning with Differential Privacy*. CCS 2016. [arxiv.org/abs/1607.00133](https://arxiv.org/abs/1607.00133) — DP-SGD formulation and the moments accountant.
- Mironov — *Rényi Differential Privacy*. CSF 2017. [arxiv.org/abs/1702.07476](https://arxiv.org/abs/1702.07476) — the RDP accountant now standard in production.
- Google Opacus / TF-Privacy — the DP-SGD reference implementations. [opacus.ai](https://opacus.ai/); [github.com/tensorflow/privacy](https://github.com/tensorflow/privacy).
- Kandpal, Wallace, Raffel — *Deduplicating Training Data Mitigates Privacy Risks in Language Models*. ICML 2022. [arxiv.org/abs/2202.06539](https://arxiv.org/abs/2202.06539) — dedup as the single largest LLM memorisation lever.
- Carlini et al. — *Membership Inference Attacks From First Principles* (LiRA, 2022). Chapter 03 reference — the paper quantifies the DP-SGD ε → TPR@FPR relationship.

## The scoreboard: the metrics you move

Define the metrics the triage is defending against, with the chapter that produces each:

- **LiRA TPR@FPR = 10⁻³** (chapter 03) — the primary classification / small-model privacy KPI.
- **Attribute inference lift over baseline** (chapter 04) — the per-attribute privacy KPI.
- **Model-inversion reconstruction fidelity** (chapter 04) — SSIM / LPIPS / cross-verify score.
- **k-extractable rate** (chapter 05) and **canary exposure** (chapter 06) — the LLM privacy KPIs.
- **Mean (ε, δ)** (mod-106) — the mathematical upper bound you quote alongside the measured numbers.

The triage table has a row per mitigation and columns for each metric *delta* the mitigation produces (plus columns for engineering cost, utility cost, and operational cost).

## The levers

### Lever 1 — DP-SGD at (ε, δ)

**What it does.** Clips per-example gradients to a bounded norm `C` and adds calibrated Gaussian noise of scale `σC` at each optimiser step. Composition across steps gives a mathematical (ε, δ)-DP guarantee. The ε bounds the attacker's advantage in a membership-inference game (up to constants).

**Effect on the metrics (empirical, Carlini-paper regime).**

- LiRA TPR@FPR drops sharply as ε decreases. At ε = ∞ (no DP), TPR@0.001 ≈ 0.08 on CIFAR-10-ResNet. At ε = 8, it drops substantially but stays non-trivial; at ε = 1, it approaches the random-guess rate. The log-scale ROC collapses toward the diagonal.
- Attribute inference lift: shrinks with ε; the model has learned less of the per-example joint.
- Extraction rate on LLMs: drops precipitously at modest ε, because verbatim memorisation requires sharp per-example gradient contributions. The Carlini-2023 Pythia-DP study is the reference evidence.

**Costs.**

- Utility cost scales with model size and dataset size. For small datasets and large models, DP-SGD at ε ≈ 1 may be unusable; at ε ≈ 8 may be within a few points of non-DP accuracy. The current state of the art has pushed this regime noticeably forward for large LMs — scan the recent literature before quoting a utility penalty.
- Engineering cost: moderate. Opacus / TF-Privacy integrate cleanly; micro-batching is the common performance hit.
- Operational cost: every training recipe change (optimiser, schedule, batch size) re-triggers the ε accounting. The accountant is a source of friction if not owned.

**When it moves the needle.** Always. DP-SGD is the mathematical privacy primitive; the question is the ε you can afford. mod-106 owns the engineering of DP-SGD; this module measures the empirical attack-metric delta.

### Lever 2 — Training data deduplication

**What it does.** Removes near-duplicates from the training corpus before training. Methods: exact-hash dedup on normalised text (cheap; misses near-duplicates), MinHash-LSH / SimHash dedup on n-gram shingles (stronger; the Lee et al. 2021 *Deduplicating Training Data Makes Language Models Better* standard). Downstream on the Kandpal result: memorisation scales log-linearly in duplicate count — reducing duplicates reduces memorisation of the duplicated content.

**Effect on the metrics.**

- LLM extraction rate: the single largest observed lever on memorisation for LLMs. Kandpal-2022's Figure 1 shows a near-elimination of memorisation on non-duplicated content.
- Canary exposure at insertion count = 1: low to begin with; dedup does not remove canaries you planted at count 1.
- LiRA TPR@FPR on classifiers: smaller effect — classification tasks typically do not have the duplication structure LLM training does.
- Attribute inference: moderate effect in the LLM-probing variant; near-zero for the tabular variant.

**Costs.**

- Utility cost: modest-to-positive. Lee et al. and later work show dedup usually *improves* model quality on held-out metrics because it reduces train-eval contamination and training-set redundancy.
- Engineering cost: needs a MinHash / SimHash implementation at corpus scale; Datatrove / Nvidia NeMo / DeepLake / Dolma-tools all ship ready-to-use pipelines.
- Operational cost: integrates into the mod-103 ingest pipeline; the dedup index should be versioned alongside the corpus datasheet.

**When it moves the needle.** First-order lever for LLM extraction. Modest lever for classifier MIA. Nearly free utility-side.

### Lever 3 — Per-example gradient clipping without DP noise

**What it does.** DP-SGD clips gradients to norm `C` then adds noise. "Clipping without noise" is the first half only — bound per-example contribution without the mathematical privacy guarantee.

**Effect on the metrics.**

- LiRA TPR@FPR: modest drop (per-example influence capped limits how distinctive each example's gradient is). The empirical literature shows clip-only is a weaker but non-trivial mitigation.
- Extraction rate: modest drop.
- Attribute inference / inversion: modest effect.

**Costs.**

- Utility cost: small; comparable to a slightly reduced learning-rate regime.
- Engineering cost: trivial; most DP-SGD frameworks support "noise multiplier = 0" as a mode.
- Operational cost: minimal.

**When it moves the needle.** As the "cheap first step" before committing to full DP-SGD. Also as the default when you cannot afford DP-SGD's utility hit but want a measurable attack-metric reduction. **Never on its own as the privacy claim** — no (ε, δ) guarantee attaches.

### Lever 4 — Epoch cap / early stopping

**What it does.** Reduces the number of passes over the training set. The empirical attack success grows with over-fitting; over-fitting grows with epochs beyond the point of generalisation gain. An epoch cap is the single-knob regulariser against over-fitting.

**Effect on the metrics.**

- LiRA TPR@FPR: the Yeom-2018 relationship between over-fitting and MIA directly applies — fewer epochs, smaller train-test accuracy gap, smaller MIA signal.
- Extraction rate: scales sub-linearly with epochs; early stopping has real but limited effect on verbatim memorisation (one strong gradient update is often enough).
- Attribute inference / inversion: reduced in proportion to over-fit.

**Costs.**

- Utility cost: positive to small-negative; the right epoch count usually coincides with the generalisation optimum anyway.
- Engineering cost: trivial.
- Operational cost: couples to mod-103's eval-set discipline — the early-stopping validation set has to be defended separately from the MIA eval set.

**When it moves the needle.** For over-trained models, meaningful. For well-tuned production training, marginal. The honest triage answer is usually "we were already at the epoch count where this does not move the needle; DP or dedup is where the budget goes".

### Lever 5 — Output filtering / completion-side redaction

**What it does.** Post-processing on model outputs before they leave the system: run the mod-104 PII detector on the completion and redact matches, or run a separate "memorisation detector" that compares completions to training-set shingles and suppresses matches.

**Effect on the metrics.**

- Extraction rate measured *at the egress*: can go to near-zero if the detector / shingle-compare is aggressive. Memorisation is still in the model — the filter blocks the emission.
- Attribute inference via LLM probing: partially mitigated.
- LiRA TPR@FPR: unchanged — LiRA attacks probability / logit outputs; the output filter does not intervene on the probability.

**Costs.**

- Utility cost: moderate-to-high false-positive-driven loss; the filter removes real content that happens to collide with training shingles.
- Engineering cost: moderate; the shingle-compare requires a hashed version of the training corpus kept at inference time.
- Operational cost: high. The egress filter is in the latency / cost / correctness path.

**When it moves the needle.** As a backstop. Memorisation is still in the weights; a black-box attacker without access to logits may be blocked by the filter but an attacker with grey-box signals probably is not. **Do not report a measurement that was taken through the egress filter as the "model-level" memorisation measurement** — measure both pre- and post-filter and report both.

### Lever 6 — Model capacity / parameter count

**What it does.** Smaller models memorise less. Carlini-2023's Pythia scaling study quantifies this: memorisation rate grows with parameter count, holding training-set-size constant.

**Effect on the metrics.**

- Extraction rate: scales with parameter count.
- LiRA TPR@FPR: modest downward effect with smaller models.

**Costs.**

- Utility cost: proportional. The capability trade-off is usually the dominant concern — if you could ship the 1B model for the task, you would not be shipping the 70B.
- Engineering cost: low (choose a smaller backbone).
- Operational cost: lower inference costs as a side benefit.

**When it moves the needle.** On utility-tolerant tasks and on fine-tunes where you are not bottlenecked by capability. Rarely the primary lever on production deployments.

### Lever 7 — Training-set redaction (upstream from mod-104)

**What it does.** Carve PII / sensitive content out of the training set before training. The mod-104 preventive control.

**Effect on the metrics.**

- Extraction rate on PII: if a PII span was redacted, the model cannot memorise it. This is the strongest empirical guarantee you can give — the content never entered the model.
- Attribute inference on redacted attributes: limited to population statistics, no training-example-specific lift.
- LiRA TPR@FPR: only on redacted content; the rest of the training set is unchanged.

**Costs.**

- Utility cost: the "⟨REDACTED⟩ went to the store" effect from mod-104. Reduces the semantic density of the corpus.
- Engineering cost: owned by mod-104 (detector, operator, eval set).
- Operational cost: owned by mod-104.

**When it moves the needle.** The ideal first-order lever for known-sensitive-content memorisation. Depends on the detector's FN rate — content the detector misses is content the memorisation measurement catches. Which is the whole reason this module exists downstream of mod-104.

## The triage table — example shape

For your exercise 06 deliverable, produce a table of this shape. Numbers are illustrative; fill in with your measurements.

| Mitigation | LiRA TPR@10⁻³ Δ | Extraction Rate Δ | Attribute Lift Δ | Utility Δ | Engineering Cost | Operational Cost | Chosen? | Rationale |
|---|---|---|---|---|---|---|---|---|
| Baseline (no mitigation) | 0.08 | 0.12 | +0.15 | — | — | — | — | — |
| Dedup (MinHash, j=0.8) | −0.01 | −0.08 | −0.02 | +0.5 pt | Medium | Low | Yes | Largest extraction lever at low utility cost |
| Clip-only (C=1.0) | −0.02 | −0.01 | −0.01 | −0.2 pt | Low | Low | Yes | Cheap diagnostic step |
| DP-SGD (ε=8, δ=1e-6) | −0.06 | −0.07 | −0.09 | −2.1 pt | Medium | Medium | Yes | Below gate on all metrics; utility within budget |
| DP-SGD (ε=2, δ=1e-6) | −0.07 | −0.11 | −0.12 | −6.0 pt | Medium | Medium | No | Utility outside budget |
| Early stop (−2 epochs) | −0.01 | 0.00 | 0.00 | −0.3 pt | Trivial | Low | No | Not moving the needle at this training point |
| Capacity ↓ (7B → 1B) | −0.01 | −0.04 | −0.01 | −4.0 pt | Low | Negative | No | Utility cost dominates |
| Output filter | 0.00* | −0.09 (egress) | 0.00* | −1.0 pt | Medium | High | Yes (backstop) | Catches residual; does not replace upstream |
| Training-set redaction | −0.01 | −0.06 | −0.05 | −0.5 pt | mod-104 | mod-104 | Yes | mod-104 already shipping |

*Output filter does not affect LiRA (attacks logits) or attribute inference (attacks probabilities under partial known fields); shown as 0.

The *interpretation discipline*: every row has a measured delta or a documented reason there is none. Blanks are the red flag. "I didn't measure DP-SGD at ε=2 because utility would have been outside budget — I know this from the ε=8 datapoint and the trend" is a valid cell; "TBD" is not.

## Where the triage decision lives

The triage is not yours alone. The release gate comes from the DPIA (mod-111) and the release-assurance review (`ai-evaluation-engineer`, L35). Your role is to surface the triage table with honest numbers. The organisation picks the gate.

A good triage conversation names:

- **The chosen set of mitigations and the metrics they each move.** No handwaving.
- **The rejected mitigations and why.** Utility cost, engineering cost, operational cost — all three count; name the dominant one.
- **The residual risk.** After all chosen mitigations, what is the LiRA TPR@10⁻³, the extraction rate on canaries at insertion count 10, the attribute inference lift on the top-5 sensitive attributes? Report with CI.
- **The compensating controls for the residual.** For the LLM case, the egress output filter + per-tenant query-rate limit + the memorisation-detector on logs (all mod-109 / mod-108 adjacent) are the belt-and-braces.
- **The re-opening trigger.** "If we increase training epochs > N, re-measure" / "if we add a new training-corpus source, re-measure" / "release-cadence re-measurement every N months".

Record this as a short, signed document — the chapter 10 boundary makes it clear whose signatures matter.

## What the triage does not cover

- **(ε, δ) composition.** When DP-SGD is applied with multiple training stages (pre-train + fine-tune + RLHF), composition is a mod-106 problem. The triage consumes the final (ε, δ); mod-106 is where it is derived.
- **Non-privacy mitigations.** Jailbreak mitigation, prompt-injection mitigation, tool-call sandboxing — out of scope. `ai-security-engineer` and mod-109 adjacent.
- **Attacks you did not measure.** The triage is an answer to the attacks in your harness. If a new attack class appears (new published paper, new MITRE ATLAS technique), the triage is re-run.

## Summary — what you can do after this chapter

- Name the seven mitigation levers (DP-SGD, dedup, clip-only, epoch cap, output filter, capacity, upstream redaction) and the attack metric each primarily moves.
- Produce a triage table with measured Δ-per-mitigation across LiRA TPR@FPR, extraction rate, attribute lift, utility, and cost columns.
- Justify the rejected rows with the dominant cost named.
- State the residual-risk summary + compensating-control set + re-opening trigger for the signed artifact.
- Hand the (ε, δ) accounting to mod-106; consume its outputs in the triage.

Exercise 06 is where the full triage is produced end-to-end on a target model. Chapter 08 names the standing harness — ML Privacy Meter — that keeps the measurement repeatable.
