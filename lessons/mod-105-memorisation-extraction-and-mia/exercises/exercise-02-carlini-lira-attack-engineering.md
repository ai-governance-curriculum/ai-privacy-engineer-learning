# Exercise 02: Carlini LiRA Attack Engineering

**Estimated effort:** 3 hours · **Output:** `mia_lira/` directory with shadow-model artefacts, per-example IN/OUT distributions, attack scoring, a log-log ROC with AUC + TPR@FPR triple, and a reportable results JSON · **Primary chapter:** 03

## Objective

Build a production-quality Likelihood-Ratio Attack (LiRA) against the target model you overfit in exercise 01. The point is to reproduce Carlini et al.'s (S&P 2022) numeric advantage over Shokri — not necessarily at the paper's absolute numbers, but qualitatively: a visible, measurable TPR@low-FPR advantage you can plot on log-log axes. This is the attack you ship when a reviewer asks for the "actually meaningful" privacy number.

## Prerequisites

- Chapter 03 (LiRA) read.
- Exercise 01 completed. You have a target model and a trained shadow-set population from exercise 01. (You may retrain shadows with more careful IN/OUT bookkeeping — LiRA's per-example IN/OUT split is different from Shokri's per-shadow-level split.)
- Carlini et al. 2022 (*Membership Inference Attacks From First Principles*) skimmed. At minimum read Section 3 and the Experiments figures.
- Optional: the paper's code release / an open LiRA reference implementation open on the side as a sanity-check.

## Problem statement

Produce:

### `shadow_training.py` — rebuild shadows with LiRA bookkeeping

Rebuild k ≥ 32 shadow models with the LiRA IN/OUT structure:

- For each shadow i, each example in the full dataset is independently included in the shadow's training set with probability 0.5.
- Record **per-example** across shadows: the list of shadows that included it (IN shadows) and the list that did not (OUT shadows).
- Each example should end up with roughly k/2 IN shadows and k/2 OUT shadows — assert this is the case.
- Train each shadow model with the target's recipe exactly matched.

### `score_phase.py` — logit-space scoring

For each example x and each shadow model that has scored it:

- Compute the logit-space confidence c(x) = log(p_y(x) / (1 − p_y(x))).
- Optionally run query-with-augmentation (e.g. 4 random augmentations per query on vision targets) and average.
- Bucket by IN / OUT; collect `in_scores[xid]` and `out_scores[xid]`.

### `fit_gaussians.py` — per-example distribution fitting

For each example x:

- Fit μ_IN(x), μ_OUT(x) from the per-example score samples.
- Fit a pooled σ_IN and σ_OUT across the full dataset (reasonable default); optionally also compute per-example σ and compare.

Add assertion: every example has at least 5 IN shadows and 5 OUT shadows; drop (and report count of) examples that do not clear the floor.

### `attack.py` — LiRA scoring on the target

- For each example x in the eval set, compute c_target(x) under the target model (with the same augmentation policy as the shadow scoring).
- Compute the log-likelihood ratio:
  ```
  log Λ(x) = (c_target(x) - μ_OUT(x))² / (2σ_OUT²) - (c_target(x) - μ_IN(x))² / (2σ_IN²)
  ```
- Save per-example (log_Λ, ground_truth_membership) to `scores/lira.csv`.

### `evaluate.py` — curves and headlines

- Compute AUC and TPR @ FPR = 10⁻², 10⁻³, 10⁻⁴ for LiRA.
- Load the Yeom + Shokri scores from exercise 01's `scores/` directory.
- Produce the paper-style log-log ROC with all three curves overlaid.
- Produce a side-by-side table of AUC + TPR@FPR for the three attacks.

### `results.json` and `report.md`

- Target model, shadow count, IN/OUT split policy, augmentation policy, pooled-vs-per-example variance, offline-vs-online variant, number of examples scored (and number dropped for insufficient shadow coverage).
- AUC + TPR@FPR triple for all three attacks.
- Methodology appendix: seeds, data splits, training-recipe match statement.
- Baseline-plot claim: "LiRA opens the TPR@10⁻³ gap over Yeom / Shokri by Δ..."

## Starter guidance

- **Check the shadow bookkeeping before fitting Gaussians.** Count how many IN and OUT shadows each example has. A bug that puts all examples in all shadows' IN sets is a silent catastrophe — assertions catch it fast.
- **Look at the per-example IN and OUT score histograms early.** They should be noticeably separated in the logit space; if they overlap completely, either shadow count is too low, the target is not overfit enough, or the logit transform is incorrect.
- **Pool σ as the default.** Per-example σ with k = 32 is extremely noisy. The pooled variant is more robust and is closer to the paper's reported setting at modest shadow counts.
- **Query with augmentation for vision targets.** Non-trivial lift. Match the augmentation to the training augmentation.
- **Do not hand-fit a threshold.** LiRA's output is a per-example log-likelihood-ratio; the threshold is swept to produce the ROC; the thresholds at FPR = 10⁻², 10⁻³, 10⁻⁴ are computed post hoc.
- **Plot on log-log.** The whole story is in the FPR ∈ [10⁻⁴, 10⁻²] regime. Log-log axes make this regime readable.

## Acceptance criteria

- [ ] At least 32 shadow models with LiRA IN/OUT bookkeeping.
- [ ] Target training recipe matched exactly by the shadow recipe; any difference documented.
- [ ] Per-example IN and OUT score counts verified (min 5 each; report count of dropped examples).
- [ ] Logit-space transform applied; per-example μ_IN, μ_OUT, pooled σ_IN, σ_OUT computed.
- [ ] Log-likelihood-ratio scoring on the target; per-example scores saved as CSV for the CI handoff.
- [ ] AUC + TPR @ FPR = 10⁻², 10⁻³, 10⁻⁴ reported with 95 % CI (CI method documented — either your bootstrap or the peer handoff).
- [ ] Log-log ROC produced with LiRA, Shokri, and Yeom overlaid.
- [ ] A one-paragraph interpretation: the AUC improvement vs. the TPR@FPR improvement, explicitly calling out Carlini-2022's "AUC understates privacy risk" argument.

## Stretch goals

- **Offline vs. online LiRA.** Produce both variants and compare the log-log curves.
- **Augmentation-count ablation.** 1 query vs. 4 queries vs. 16 queries per example. Report TPR@FPR deltas.
- **Per-example σ vs. pooled σ.** Compare the TPR@FPR deltas; discuss when per-example is worth the extra variance.
- **Shadow-count scaling curve.** TPR@10⁻³ as k increases from 8 to 128 (in powers of 2). The curve is directly informative about audit cost.
- **Query-batch efficiency.** Instrument the harness for inference throughput; report total shadow-training + scoring cost; the stretch target is "LiRA at full k costs < X GPU-hours".

## What this lands

- The LiRA number is the headline MIA number for every downstream deliverable in the module.
- The per-example score CSV is the primary input to the `model-evaluation-engineer` CI handoff for the DPIA.
- The log-log plot is the DPIA's single most important image.
- The shadow-set population is reusable — exercise 06's defender triage will re-run LiRA with each mitigation applied.

Version the artefact. Stake the LiRA numbers against your exercise 01 Shokri numbers and record the improvement.
