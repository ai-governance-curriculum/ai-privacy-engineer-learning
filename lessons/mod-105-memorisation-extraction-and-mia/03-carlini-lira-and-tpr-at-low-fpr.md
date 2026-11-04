# Carlini LiRA — The Likelihood-Ratio Attack and the Low-FPR Regime

Chapter 02 built a Shokri attack and ended on the caveat that its AUC number averages across operating points. Carlini, Chien, Nasr, Song, Terzis, and Tramèr's 2022 paper *Membership Inference Attacks From First Principles* (IEEE S&P 2022) made that caveat the central problem and proposed the attack that has been the production standard since. This chapter is the engineering reference to the Likelihood-Ratio Attack (LiRA) — why it exists, what it measures, how to implement it, and what number to actually report from it.

Primary references:

- Carlini, Chien, Nasr, Song, Terzis, Tramèr — *Membership Inference Attacks From First Principles*. IEEE S&P 2022. [arxiv.org/abs/2112.03570](https://arxiv.org/abs/2112.03570)
- Ye, Maddi, Murakonda, Bindschaedler, Shokri — *Enhanced Membership Inference Attacks against Machine Learning Models* (CCS 2022, *RMIA*). [arxiv.org/abs/2111.09679](https://arxiv.org/abs/2111.09679) — a related approach that also recalibrates per-example.
- Watson, Guo, Cormode, Sablayrolles — *On the Importance of Difficulty Calibration in Membership Inference Attacks* (ICLR 2022). [arxiv.org/abs/2111.08440](https://arxiv.org/abs/2111.08440) — the "difficulty calibration" insight that LiRA formalises.

## Why LiRA exists — the one-number AUC trap

Carlini et al.'s central argument: **privacy risk is a per-example worst-case, not a population average.** An attacker does not care whether they can be right 56 % of the time across *all* examples; they care whether there is *some* training example for which they can be right 95 %+ of the time at a low false-positive rate. The AUC averages over the entire ROC — most of it in the useless-to-the-attacker, high-FPR region. The privacy-relevant region is the low-FPR tail, and that tail gets washed out of AUC.

The paper's numerical demonstration: on CIFAR-10 with ResNet-20 trained without privacy, the Yeom loss-threshold attack has AUC ≈ 0.57 and TPR@0.001 FPR ≈ 0.01. The LiRA attack on the same target has similar AUC (≈ 0.65) but TPR@0.001 FPR ≈ 0.08 — **~8× the confident-attack success at the low-FPR regime**. The AUC improvement understates the privacy-leakage improvement by an order of magnitude.

The implication for your reporting: the AUC is a *diagnostic*, and the TPR at low FPR is the *risk number*. The DPIA quotes the second. A number without the second is a number that will get challenged on the Carlini point by any reviewer who has read the 2022 paper.

## The one-sentence intuition

LiRA's core idea: for each target example x, learn its *expected confidence distribution* when it is in the training set (IN) and when it is not (OUT), across many shadow models. At attack time, the target model's confidence on x is one sample — a *likelihood ratio* under the IN vs. OUT distributions gives a per-example membership score with the right statistical foundation.

The Shokri attack learns one decision function per class. LiRA learns one decision function *per example*, implicitly, by fitting per-example Gaussian (or similar) distributions over confidence under IN and OUT.

## Step 1 — the shadow phase, re-costed

LiRA's shadow setup is structured differently from Shokri's. Each shadow model's IN / OUT split determines, *for each target example x*, whether that shadow model saw x or not. The attack then needs, for each x:

- A set of shadow models that *did* see x (the IN shadows for x).
- A set of shadow models that *did not* see x (the OUT shadows for x).

With k shadow models and each example independently included in each shadow set with probability 0.5 (the "random-subset" construction), each example has an expected k/2 IN shadows and k/2 OUT shadows. For k = 64, you get ~32 IN samples and ~32 OUT samples per example — enough to fit a 1-D Gaussian confidently.

Two standard variants:

- **Offline LiRA.** All shadow models are trained once; the attack uses only the OUT shadows per example (no IN shadows) and a weaker Gaussian model. Cheaper, slightly weaker.
- **Online LiRA.** Uses both IN and OUT shadows per example; stronger attack. Standard choice for a production measurement.

A further variant — **"attack-with-full-training-set"** — assumes the attacker can train a shadow on the full target training set to characterise the IN distribution; irrelevant for your use case (you *are* the defender), but worth understanding when reading the paper's variants table.

## Step 2 — the confidence score (the "logit" score)

A naive score is the model's probability for the true class. LiRA uses a monotone transform that puts the score in a regime where the IN / OUT distributions are approximately Gaussian:

For a classifier with softmax output, score x as:

```
c(x) = log( p_y(x) / (1 - p_y(x)) )
```

where p_y(x) is the probability assigned to the true class y. This is the logit of the true-class probability. The paper shows empirically that under this transform the IN and OUT distributions are well-approximated as 1-D Gaussians with distinct means — the ingredient the likelihood ratio needs.

For non-classification targets (regression, language-model next-token probability), define an analogous confidence score — negative loss or log-probability of the target token, respectively. The method generalises; the Gaussian fit still needs to be empirically verified.

## Step 3 — fit the IN / OUT Gaussians per example

For each target example x:

- For each shadow model that saw x (IN shadow), compute c_i(x). Collect into IN_scores(x).
- For each shadow model that did not see x (OUT shadow), compute c_j(x). Collect into OUT_scores(x).
- Fit `μ_IN(x), σ_IN(x)` from IN_scores(x); fit `μ_OUT(x), σ_OUT(x)` from OUT_scores(x).

Two numerical details:

- The per-example σ is noisy with k = 64. A common fix is to pool σ across examples (`σ_IN = global_σ_IN`) or to use a per-example variance with Bayesian smoothing. The paper discusses both; the pooled variance is usually a good default.
- Data augmentation at training time (random crops, flips for vision) should be reproduced at scoring time. For each x, query under the same augmentations and aggregate (mean of logits). This is "query-with-augmentation" LiRA and adds measurable signal; the reference implementation (see tooling below) does it by default.

## Step 4 — the per-example likelihood ratio

Score the target example x against the target model with its logit-space confidence c_target(x). The membership score is the likelihood ratio:

```
Λ(x) = P( c_target(x) | N(μ_IN(x), σ_IN(x)²) )  /  P( c_target(x) | N(μ_OUT(x), σ_OUT(x)²) )
```

Higher Λ(x) means "this score looks more like what you would see if x were in training". Equivalent log-likelihood-ratio form is easier numerically. The paper writes it as:

```
log Λ(x) = (c_target(x) - μ_OUT(x))² / (2σ_OUT²) - (c_target(x) - μ_IN(x))² / (2σ_IN²)
```

plus a constant from the normalising factors (negligible when σs are pooled).

The attack decision is: predict "member" if Λ(x) > τ, where τ is swept to produce the ROC.

## Step 5 — the right way to plot and report

**Plot the ROC on log-log axes.** This is the Carlini-paper visual convention and it matters. On linear-linear axes, the whole low-FPR regime is compressed near the y-axis and invisible. On log-log, the attack's behaviour at FPR = 10⁻² through 10⁻⁵ is the dominant region of the plot.

**Report the specific numbers.** Report:

- AUC (for continuity with pre-2022 literature).
- TPR @ FPR = 10⁻², 10⁻³, 10⁻⁴ (minimum), with 95 % CI on each.
- Number of shadow models k, shadow-training-set size, target training-set size, number of target examples evaluated.
- Whether query-with-augmentation was on, pooled vs. per-example σ, offline vs. online LiRA variant.

**Compare to the Yeom / Shokri baseline.** The report is strongest when both numbers appear — Yeom's loss-threshold at the chosen FPR, Shokri's per-class attack at the chosen FPR, LiRA at the chosen FPR. The gap LiRA opens over Yeom at low FPR is the headline (see the Carlini paper's Figures 1–3 for the canonical visual).

## Pseudocode outline

```python
# Shadow phase — K ~= 64 for production, 16 for a diagnostic run
shadow_ins = {}   # shadow_id -> set of training-example ids
for sid in range(K_SHADOW):
    in_ids = random_subset(all_ids, fraction=0.5, seed=sid)
    shadow_ins[sid] = set(in_ids)
    shadow_models[sid] = train_target_recipe(dataset[in_ids])

# Score phase — per-example IN / OUT logits
def logit(p):
    return np.log(p / (1 - p + 1e-12))

in_scores = defaultdict(list)
out_scores = defaultdict(list)
for sid, m in shadow_models.items():
    for xid, (x, y) in enumerate(target_dataset):
        c = logit(m.softmax(x)[y])       # add augmentation averaging in production
        if xid in shadow_ins[sid]:
            in_scores[xid].append(c)
        else:
            out_scores[xid].append(c)

# Fit per-example parameters, pooled sigma
mu_in  = {xid: np.mean(vs) for xid, vs in in_scores.items()}
mu_out = {xid: np.mean(vs) for xid, vs in out_scores.items()}
sigma_in  = np.sqrt(np.mean([np.var(vs) for vs in in_scores.values()]))
sigma_out = np.sqrt(np.mean([np.var(vs) for vs in out_scores.values()]))

# Attack phase — score the target
def log_lambda(xid, c):
    return ((c - mu_out[xid])**2) / (2*sigma_out**2) - ((c - mu_in[xid])**2) / (2*sigma_in**2)

scores = {xid: log_lambda(xid, logit(target_model.softmax(x)[y]))
          for xid, (x, y) in enumerate(target_dataset)}

# Report (per-example score, is_member) for the eval engineer's CI
```

Exercise 02 has you build this with real shadow models on a CIFAR-10- or Purchase100-shaped target, draw the log-log ROC against Yeom and Shokri baselines, and report the TPR@FPR triple with the CI methodology peered to `model-evaluation-engineer`.

## Production notes

Things that are easy to get wrong when scaling LiRA from the paper's benchmark to your production target.

- **Shadow count k is the dominant cost.** k = 64 shadow models at target-training cost is a 64× cost. For a non-trivial target this is already large; for a large model, this is the limiter. Offline LiRA + query-with-augmentation partially compensates. For LLM-scale targets, chapter 05's extraction evaluation is the practical substitute — training 64 Llama-7B shadows is not realistic; you shift to the memorisation / extraction evaluation instead.
- **Reproducing the target's training recipe exactly.** The shadow models must share the target's data preprocessing, augmentation, optimiser, schedule, early-stopping policy, and random-seed policy. Any divergence shifts the IN distribution away from what the target actually learned and biases the attack.
- **Membership label hygiene.** The IN set for a shadow model must be exactly the set the shadow model was trained on. Any accidental in/out drift (e.g. examples added via augmentation, or dropped due to a filter) invalidates the IN/OUT bookkeeping for those examples.
- **Not every example has IN *and* OUT shadows.** A given example may, by chance, land in 0 IN shadows or 0 OUT shadows out of k = 64. Score only examples with at least `min_in` and `min_out` shadows (5 is a reasonable floor); report the fraction of examples that could be scored.
- **Query cost per target example is non-trivial.** k shadow queries + augmentation × A queries = kA inferences per example. For large eval sets, batch and cache.

## What LiRA does not measure

LiRA is a membership-inference measurement. It does *not* tell you:

- Whether a given training example has been **memorised verbatim** in a generative sense. A classifier's probability on x doesn't encode whether the LM would emit x. Chapter 05's extraction methodology is the right tool for verbatim memorisation.
- Whether **attributes** of x can be inferred. LiRA predicts "x was in training"; attribute inference predicts "the missing column of x̃ is z". Chapter 04 is the right tool.
- Whether a model is **(ε, δ)-private**. LiRA's empirical AUC and TPR are evidence toward an (ε, δ) upper bound (Nasr et al., *Adversary Instantiation* (S&P 2021) — [arxiv.org/abs/2101.04535](https://arxiv.org/abs/2101.04535)), but the (ε, δ) number itself is proved from the training algorithm, not measured. mod-106 is where the (ε, δ) accounting lives; this module is where the empirical attack measurement lives. The two are complementary.

Name these in the report as scope.

## Summary — what you can do after this chapter

- State Carlini et al.'s one-number-AUC-trap argument and explain why TPR@low-FPR is the right privacy number.
- Design the LiRA shadow phase correctly: random-subset IN/OUT, k shadows with each example in ~k/2 IN and ~k/2 OUT shadows.
- Define the per-example logit score and justify why it (not raw probability) is the right scoring transform.
- Fit per-example IN / OUT Gaussians (with pooled σ as default) and compute the likelihood ratio.
- Report on log-log axes with TPR @ FPR = 10⁻²..10⁻⁴, versus Yeom and Shokri baselines, with CI from the peer handoff.
- Name what LiRA does *not* measure — verbatim memorisation (ch. 05), attribute inference (ch. 04), (ε, δ) itself (mod-106).

Exercise 02 is the production implementation. Chapter 04 shifts to attribute-inference and model-inversion attacks, which are different questions against the same target model.
