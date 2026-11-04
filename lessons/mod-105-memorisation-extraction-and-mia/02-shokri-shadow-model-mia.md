# Shokri Shadow-Model Membership Inference at Production Quality

Shokri et al.'s 2017 shadow-model membership-inference attack (IEEE S&P) is the canonical first attack in the MIA literature and the one almost every reviewer will recognise. For the privacy engineer, it is the attack you ship first because it is the easiest to construct, the easiest to explain in a review, and the one that lets you discover whether your pipeline is correctly wiring target-model outputs to an attack model at all. LiRA (chapter 03) is the production-grade number you quote; Shokri is the scaffolding you run first to convince yourself the harness is wired correctly.

Primary references:

- Shokri, Stronati, Song, Shmatikov — *Membership Inference Attacks Against Machine Learning Models*. IEEE S&P 2017. [arxiv.org/abs/1610.05820](https://arxiv.org/abs/1610.05820)
- Salem et al. — *ML-Leaks: Model and Data Independent Membership Inference Attacks and Defenses on Machine Learning Models*. NDSS 2019. [arxiv.org/abs/1806.01246](https://arxiv.org/abs/1806.01246) — the follow-up that relaxes several of Shokri's assumptions (same architecture, same distribution) and is closer to how you will actually ship.
- Yeom et al. — *Privacy Risk in Machine Learning: Analyzing the Connection to Overfitting*. CSF 2018. [arxiv.org/abs/1709.01604](https://arxiv.org/abs/1709.01604) — the loss-threshold MIA baseline that LiRA (chapter 03) generalises.

## The attacker and the setting

The Shokri setting is the clearest MIA setting to start with. Make the assumptions explicit so the attack results are interpretable:

- **The attacker is black-box.** They can query the target model and get a prediction vector (class probabilities for classification; logits or sampled tokens for a language model). They do not have the weights.
- **The attacker knows the target model's architecture family.** In the original paper, they train shadow models of the *same* architecture; later work (Salem) relaxes this.
- **The attacker has samples from the target's training distribution.** Not the training set itself, but samples from the same distribution (same task, same feature space, same label space).
- **The attacker's goal.** For a specific target example x, decide: is x in the target model's training set, or is it a held-out example? Measured across many x, this yields a (TPR, FPR) curve and an AUC.

The *privacy-engineer's* setting is a mirror of the attacker's. You are the defender, and you want the attacker's numbers on your model. You *do* have the training set. You *do* have the architecture. You are constructing the worst-case realistic attacker to measure what they would achieve.

## Step 1 — construct the shadow training sets

The core idea: train **k shadow models** on known IN / OUT splits of a dataset drawn from the same distribution as the target's training set. For each shadow model, you have ground-truth membership — you chose which examples were IN its training set.

Mechanically:

- Partition a *shadow dataset* (same distribution as target's training set, same feature / label space; disjoint from the target training set is the clean experimental setup, though the original paper allows overlap in some variants).
- For each of k shadow models (typical values: k = 10 for a diagnostic run; k = 64–128 for a production number comparable with LiRA):
  - Draw a random subset of the shadow dataset as the shadow IN set (typically same size as the target training set, or a fixed fraction).
  - The complement (plus a disjoint holdout) is the shadow OUT set.
  - Train a shadow model on the shadow IN set. Match the target model's training recipe as closely as you can — architecture, optimiser, epochs, data augmentation.
- You now have k pairs: (shadow_model_i, (IN_set_i, OUT_set_i)).

Three failure modes that will silently invalidate the attack if you do not catch them:

- **Shadow set drawn from a mismatched distribution.** If the shadow data has materially different class balance or feature distribution than the target's, the attack model learns a shortcut specific to the shadow distribution and does not transfer. Report the KS / χ² test between shadow and target distributions as part of the methodology.
- **Shadow model under-trained.** The attack exploits the gap between the model's confidence on training examples and on held-out examples. If the shadow model has not overfit far enough, there is no gap — but the target model has overfit further, so the attack will under-estimate the real risk. Train each shadow model to the same training loss as the target.
- **Train / shadow overlap leakage.** If a shadow training set accidentally includes examples from the target's training set, the shadow model's IN / OUT gap for those examples is biased. Keep the two populations disjoint by construction and assert it in the harness.

## Step 2 — generate the attack-training set

For each shadow model, query it on *every* example in its shadow IN set and its shadow OUT set:

- Input to the shadow model: x
- Output: prediction vector p(x) (class probabilities for classification, or a featurised representation for other model classes).
- Attack-training label: 1 if x ∈ IN_i, 0 if x ∈ OUT_i.

Concatenate across all k shadow models to produce the attack-training set:

```
X_attack = { (p_i(x), y_i(x), membership_label) : all shadow models i, all x in IN_i ∪ OUT_i }
```

Two design decisions matter here:

- **Per-class attack models vs. one global attack model.** Shokri trains *one attack model per output class*: the input is the prediction vector, the label is membership, conditioned on the true class of x. The per-class model exploits the fact that the signal is often concentrated in the probability assigned to the true class and that classes have different confidence distributions. A one-model-fits-all attack is a simpler baseline worth keeping in the harness for comparison.
- **Featurisation beyond the raw probability vector.** Add the loss (negative log probability of the true class), the top-1 margin, entropy, and sorted / unsorted probability vectors as features. The ML-Leaks and Yeom et al. papers show that a loss-only threshold is a strong baseline and that a richer feature set rarely beats it dramatically but usually helps marginally.

## Step 3 — train the attack model

A small supervised classifier, one per class (or one global). Typical choices: a 2-layer MLP with ReLU; a gradient-boosted tree; even a logistic regression. The decision function is simple — the *features* carry the signal.

Train on X_attack with the usual discipline: train / validation split (not over the target population — over the shadow population), early stopping on validation AUC, regularisation. An over-fit attack model inflates the shadow-measured AUC above what will transfer to the target.

A production detail: the attack model's threshold on the output probability is the knob that trades off TPR for FPR. For a reportable curve, sweep the threshold from 0 to 1 and compute (TPR, FPR) at each — that is the attack ROC.

## Step 4 — apply the attack to the target

Now you use the attack model on the *target* model. For each example x in your evaluation set:

- Query the target model: get its prediction vector p_target(x).
- Featurise the same way you featurised during attack-training.
- Score with the attack model: get a membership-probability score s(x) ∈ [0, 1].
- Compare to ground-truth membership (you know — you trained the target).

The evaluation set is a balanced sample: half from the target's training set (members), half from a disjoint holdout drawn from the same distribution (non-members). Balanced sampling is important — the AUC is well-defined, and the TPR@FPR numbers are interpretable only if you know the base rate.

## Step 5 — report AUC *and* TPR at fixed FPR

This is the most important engineering point in the chapter. **Report both**:

- **AUC** — the integral of the attack ROC. Easy to compute, familiar to reviewers, dominant in the pre-2022 literature.
- **TPR @ low FPR.** Report TPR at FPR = 10⁻², 10⁻³, and 10⁻⁴ (note the log-scale). This is the number Carlini et al. (2022) argue is the *actually meaningful* privacy number — it tells you what fraction of specific training examples an attacker can reliably identify while accepting only a small number of false positives.

Why both: AUC averages across all operating points, including high-FPR regions where the attacker is wrong most of the time and the privacy incident is not realistic. A model can have AUC = 0.56 (barely above chance) and still have TPR@0.001 FPR = 0.08 — meaning 8 % of members are confidently identifiable at a 0.1 % false-positive rate. That is a meaningful privacy risk hidden by the AUC number. A defender who only reports AUC will miss it.

The confidence-interval methodology for both numbers — bootstrap over the evaluation set, 95 % CI — is handed to the peer `model-evaluation-engineer` (L30). Their track covers the resampling design and the peer-reviewed thresholds for significance. Your harness emits the raw per-example scores + labels; their methodology produces the CI. Chapter 08 names the handoff; chapter 10 draws the boundary.

## A concrete pseudocode outline

Not production code — a reading guide. Exercise 01 builds the real version against a dataset you choose (CIFAR-10, Purchase100, or the Kaggle Texas hospital stay dataset — the classic MIA testbeds).

```python
# ---------- Shadow phase ----------
shadow_models = []
attack_rows = []
for i in range(K_SHADOW):
    in_i, out_i = split_shadow_dataset(seed=i)
    shadow_i = train_target_recipe(in_i)         # mirror target training exactly
    for x, y in in_i:
        attack_rows.append((featurise(shadow_i(x), y), y, 1))
    for x, y in out_i:
        attack_rows.append((featurise(shadow_i(x), y), y, 0))
    shadow_models.append(shadow_i)

# ---------- Attack-model phase ----------
attack_models = {}
for cls in classes:
    rows_cls = [r for r in attack_rows if r[1] == cls]
    X = np.stack([r[0] for r in rows_cls])
    y = np.array([r[2] for r in rows_cls])
    attack_models[cls] = train_small_classifier(X, y)

# ---------- Target phase ----------
eval_scores, eval_labels = [], []
for x, y, is_member in target_eval_set:          # balanced IN / OUT
    f = featurise(target_model(x), y)
    s = attack_models[y].predict_proba(f)[1]
    eval_scores.append(s)
    eval_labels.append(is_member)

auc = roc_auc_score(eval_labels, eval_scores)
tpr_at_fpr = {fpr: tpr_at(eval_scores, eval_labels, fpr)
              for fpr in [1e-2, 1e-3, 1e-4]}
```

The harness you ship in exercise 01 wraps this with:

- A target-model abstraction so the attack is reusable across targets.
- A seed set and config file so the shadow phase is reproducible.
- The CI handoff: the per-example `(score, label)` arrays pickled for the evaluation engineer's bootstrap.

ML Privacy Meter (chapter 08) implements all of this. Exercise 01 has you build it from scratch once — so you understand what ML Privacy Meter is doing — then exercise 07 has you adopt ML Privacy Meter as the standing harness.

## Known limitations of the Shokri attack

A defender who reports a Shokri AUC without naming the caveats is giving an incomplete number. The caveats:

- **Average-case, not worst-case, per example.** The per-class attack model is a single decision function; it cannot adapt to the *specific* target example. LiRA fixes this.
- **Shadow count is small by default.** With k = 10 shadow models, the attack-training set per class is small, and the attack is noisy. Scaling k to 64–128 is cheap for small targets but expensive for large ones. LiRA uses the shadow count more efficiently.
- **Assumes same-architecture, same-distribution shadow setup.** Salem's ML-Leaks relaxed this; report what you assumed.
- **Classification-focused.** Generalising to language models or regression targets requires rethinking the featurisation (chapter 05 revisits this for LLMs).

The chapter 03 LiRA addresses the first two in production. The next chapter is that upgrade.

## Summary — what you can do after this chapter

- Name the four assumptions of the Shokri setting (black-box target, known architecture, same-distribution shadow data, attacker's membership goal).
- Build the shadow phase correctly: disjoint shadow / target populations, matched training recipe, balanced IN / OUT per shadow model.
- Build the attack phase correctly: per-class attack model, feature vector = probabilities + loss + margin + entropy.
- Report AUC *and* TPR@FPR (10⁻², 10⁻³, 10⁻⁴). Name the CI handoff to `model-evaluation-engineer`.
- State the four caveats of a Shokri number and why LiRA is the next upgrade.

Exercise 01 has you build the end-to-end Shokri harness against a classic MIA benchmark. Chapter 03 is the LiRA upgrade.
