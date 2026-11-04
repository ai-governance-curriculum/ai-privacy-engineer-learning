# Exercise 01: Shadow-Model MIA End to End

**Estimated effort:** 3 hours · **Output:** `mia_shokri/` directory with trained shadow / target models, attack classifiers, plots, and a reportable results JSON · **Primary chapter:** 02

## Objective

Build the Shokri shadow-model membership-inference attack from scratch against a reference classifier, end-to-end. The goal is not to produce a surprising AUC — it is to internalise the construction so that when you operate ML Privacy Meter (exercise 07) and consume paper results on LiRA (exercise 02), you know what the harness is doing under the hood and you can defend every design choice to a reviewer.

## Prerequisites

- Chapter 02 (Shokri shadow-model MIA) read.
- Chapter 01 (motivation) read.
- A PyTorch or TensorFlow environment with GPU access (small datasets — a modern CPU will also work, slower).
- A classic MIA testbed dataset. Choose one of:
  - **CIFAR-10** — vision. Overfit a ResNet-20 for a visible MIA gap.
  - **Purchase100** — tabular, 100-class. The Shokri-paper canonical.
  - **Texas-100 Hospital Stay** — tabular. Similar scale to Purchase100.
  - A domain dataset you own (preferred if you have one at MIA-attackable scale — low-thousands to low-millions of records, overfitting-prone architecture).

## Problem statement

Produce:

### `target_training.py` — the target model

Train a classifier under a recipe that will be *faithfully reproducible* in shadow phase. Deliberately choose a training schedule that will leave a detectable MIA gap (e.g., train to near-zero training loss without aggressive regularisation) so the exercise produces a measurable attack. Record:

- Architecture, optimiser, epochs, batch size, augmentation, random seed.
- Training set indices (so the "member" / "non-member" split is reproducible).
- Final train vs. held-out accuracy (the overfit gap is the signal the attack exploits).

### `shadow_training.py` — the shadow phase

Train k shadow models (start with k = 8 for a diagnostic run, k = 32 for a reportable number). For each shadow:

- Draw a random IN subset from a *shadow dataset* disjoint from the target's training set but drawn from the same distribution.
- Reproduce the target's training recipe exactly (same architecture, optimiser, epochs, augmentation, random seed policy — only the IN set changes).
- Record each shadow's IN / OUT split.

Add assertions that the target and shadow IN sets are provably disjoint.

### `attack_training.py` — the attack classifier

For each shadow:

- Query every example in IN ∪ OUT; record the prediction vector.
- Featurise: probability vector + loss + top-1 margin + entropy. At minimum.

Train per-class attack classifiers (one MLP per output class). Hold out 20 % of the attack-training set for validation. Early-stop on validation AUC.

Also implement at least one baseline for comparison:

- Yeom loss-threshold attack (score x by its loss under the target; threshold).

### `evaluate_target.py` — the attack on the target

On a balanced IN / OUT eval set drawn from (target_training_set, disjoint_holdout):

- Query the target model.
- Score with the trained attack classifier and with the loss-threshold baseline.
- Compute AUC and TPR @ FPR = 10⁻², 10⁻³, 10⁻⁴ for both.
- Save per-example `(score, label)` arrays to `scores/{attack}.csv` for the CI handoff.

### `results.json` — the reportable summary

- Target model summary (architecture, overfit gap, hyper-parameters).
- Shadow configuration (k, dataset split, seed set).
- Attack-model configuration (per-class MLP, featurisation set).
- AUC + TPR@FPR triple per attack per fold (if you ran k-fold evaluation).
- Methodology-caveats section naming: shadow-count, shadow-vs-target distribution test (KS / χ²), overlap-assertion status, train-recipe-match status.

### `report.md` — the human-readable write-up

Up to one page naming the measurement, the attacker model assumed, the attack curves, the two numbers (AUC and TPR@10⁻³) with baseline comparison, and the four caveats you ship with the Shokri number from chapter 02's summary.

## Starter guidance

- **Start with the loss-threshold attack.** Yeom's loss-only baseline is ~20 lines of code; running it first confirms your eval set is correctly labelled and the target has a measurable overfit. If this baseline gives AUC ≈ 0.5 your eval set is wrong; if it gives a plausible AUC (e.g. 0.6+), the Shokri attack should improve on it.
- **Overfit the target deliberately.** Standard well-regularised recipes often leave the overfit gap small and the MIA attack weak; for the exercise you *want* a visible signal. Train to near-zero training loss; skip aggressive augmentation; do not use strong weight decay.
- **Keep shadow count modest for the first pass.** k = 8 is enough to debug the pipeline. Once the pipeline is correct, scale to k = 32 for the reportable number.
- **Pin seeds — everywhere.** Dataset splits, model init, DataLoader shuffling, augmentation. Reproducibility is the main value of the artefact.
- **Report both AUC and TPR@FPR.** Even though AUC is the Shokri paper's headline, chapter 03's point is that TPR@FPR is the right number. Train the muscle now.

## Acceptance criteria

- [ ] Target and shadow training sets are provably disjoint (asserted in the harness).
- [ ] Shadow training recipe matches the target training recipe exactly (document any difference and justify).
- [ ] Attack is evaluated on a balanced IN / OUT eval set with at least 2000 total examples (1000 members + 1000 non-members).
- [ ] Yeom loss-threshold baseline is reported alongside the Shokri per-class MLP attack.
- [ ] AUC *and* TPR @ FPR = 10⁻², 10⁻³, 10⁻⁴ are reported for both attacks.
- [ ] `scores/{attack}.csv` contains per-example (score, label) arrays suitable for a bootstrap CI.
- [ ] `results.json` and `report.md` are produced with a methodology-caveats section.
- [ ] A log-linear ROC plot is produced; a log-log ROC plot is also produced (preparing for exercise 02).

## Stretch goals

- **Scale to k = 128** shadow models and compare the AUC / TPR@FPR variance across the k = 8, 32, 128 runs. Report the variance-reduction curve.
- **Add featurisation ablations.** Compare: probability-only vs. probability + loss vs. probability + loss + margin + entropy. Report the AUC / TPR delta per ablation.
- **Compare attack-model families.** Per-class MLP vs. per-class gradient-boosted tree vs. one global MLP. Report the deltas.
- **Repeat on a different architecture** (e.g. MobileNet-V3 on CIFAR-10 vs. ResNet-20) to show that MIA signal is architecture-dependent.
- **Ship the artefact as a mini standing harness** — target-model wrapper + config file + CLI entry — ready for the exercise 07 ML Privacy Meter translation.

## What this lands

- The Shokri-style baseline becomes the first row of the coverage matrix (exercise 06 referenced; chapter 09 is the template).
- The per-example score arrays are the input to `model-evaluation-engineer`'s CI methodology — you will need them for the DPIA citation in exercise 06.
- The methodology-caveats section becomes a template you reuse in every attack report.
- The under-the-hood build means ML Privacy Meter is no longer a black box when you adopt it in exercise 07.

Version the artefact. Pin the seeds. Record the overfit gap of the target model — it is the single most important diagnostic for whether the attack is calibrated.
