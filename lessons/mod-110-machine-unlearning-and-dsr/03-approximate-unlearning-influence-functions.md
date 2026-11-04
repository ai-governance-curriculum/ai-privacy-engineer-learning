# Approximate Unlearning — Influence Functions, Certified Removal, and the Verification Test

SISA (chapter 02) gives you *exact* unlearning if you architect for it from the start. The cost is the ensemble tax — the training overhead, the checkpoint storage, the utility degradation from shard-local training. Many systems cannot pay that tax: the model is already trained on a single-shard pipeline, the model family does not ensemble cleanly, or the deletion cadence is low enough that an always-on SISA investment is wasteful. The alternative is *approximate* unlearning: compute an update to the trained weights that is a *close approximation* of "what the weights would have been if the record had never been there", and defend the approximation against an attack-shaped verification test.

This chapter covers the two production-tractable approximate-unlearning techniques on classical models — *influence-function removal* and *Ginart-style data-deletion algorithms* — and the verification discipline that lets the technique be defensible under audit. Chapter 04 is the sibling chapter for LLM-specific approximate unlearning (gradient ascent, task vectors, Who's-Harry-Potter).

Primary references:

- Koh, Liang — *Understanding Black-box Predictions via Influence Functions*, ICML 2017. [arxiv.org/abs/1703.04730](https://arxiv.org/abs/1703.04730). The paper that re-introduced influence functions to deep learning and gave the implicit-Hessian-vector-product recipe this chapter uses.
- Guo, Goldstein, Hannun, van der Maaten — *Certified Data Removal from Machine Learning Models*, ICML 2020. [arxiv.org/abs/1911.03030](https://arxiv.org/abs/1911.03030). The theoretical scaffolding — a Newton-step update to the weights, with a residual bound on how far the resulting model is from the "trained without the record" model.
- Ginart, Guan, Valiant, Zou — *Making AI Forget You: Data Deletion in Machine Learning*, NeurIPS 2019. [arxiv.org/abs/1907.05012](https://arxiv.org/abs/1907.05012). Deletion-efficient algorithms with quantified cost / accuracy trade-offs — especially the k-means and Voronoi recipes.
- Izzo, Smart, Chaudhuri, Zou — *Approximate Data Deletion from Machine Learning Models*, AISTATS 2021. [arxiv.org/abs/2002.10077](https://arxiv.org/abs/2002.10077). The *projective residual update* recipe for linear models.
- Sekhari, Acharya, Kamath, Suresh — *Remember What You Want to Forget: Algorithms for Machine Unlearning*, NeurIPS 2021. [arxiv.org/abs/2103.03279](https://arxiv.org/abs/2103.03279). The learning-theoretic framing that pairs well with the (ε, δ)-certified-removal definitions.
- Thudi, Jia, Shumailov, Papernot — *On the Necessity of Auditable Algorithmic Definitions for Machine Unlearning*, USENIX Security 2022. [arxiv.org/abs/2110.11891](https://arxiv.org/abs/2110.11891). The *verification* paper — argues that any approximate-unlearning claim is only defensible if paired with an auditable test.
- Shokri, Stronati, Song, Shmatikov — *Membership Inference Attacks Against Machine Learning Models*, IEEE S&P 2017. [arxiv.org/abs/1610.05820](https://arxiv.org/abs/1610.05820). The MIA primitive used in the verification tests here (mod-105 is the full module on these).

## When approximate unlearning is the right tool

Not every deletion needs an exact-unlearning answer. Three regimes make approximate unlearning the right fit:

- **Classical ML on tabular / small-vision**, where the loss surface is well-behaved and Newton-style updates converge cleanly. Logistic regression, small MLPs, SVMs, random-forest-adjacent models.
- **Low-to-moderate deletion cadence** — a handful of DSRs per month per model, where investing in SISA from the start is overkill but a full retrain is still expensive enough that a "cheap approximate update" has value.
- **Ensemble-leaf unlearning** — a specific leaf model in a larger system (a per-vertical fraud model, a per-region recommender) where retraining the one model is still painful but doing it in-place with an influence update is tractable.

The regime it is *not* for:

- **Deep networks** on large, non-convex loss surfaces. The influence-function approximation is derived from a convex Taylor expansion; on a 10B-parameter transformer the approximation error is large enough that the "unlearned" model may still contain the record's imprint. Chapter 04 is the LLM-specific sibling.
- **Hard-privacy regulated workloads** where the regulator will ask for exact, not approximate. SISA or full retrain is the only defensible answer.
- **High-cadence deletion (thousands per month)** on any model. Each approximate update is still O(n) in records for the Hessian-vector product; costs compound.

## The influence-function recipe

### The intuition

A trained model's weights `θ*` are the solution to an empirical-risk minimisation:

```
θ* = argmin_θ (1/n) Σ_i L(x_i, y_i; θ)
```

Removing a training point `(x_r, y_r)` changes the loss and the minimiser:

```
θ*_(−r) = argmin_θ (1/(n-1)) Σ_{i ≠ r} L(x_i, y_i; θ)
```

Influence-function theory says that for small changes (one point among n), the shift in the minimiser is well-approximated by a linear function of the removed point's gradient, mediated by the Hessian of the loss at `θ*`:

```
θ*_(−r) ≈ θ* + (1/n) H_θ*^{-1} ∇_θ L(x_r, y_r; θ*)
```

where `H_θ*` is the Hessian of the empirical loss at the trained weights. The update is cheap — it is a single vector addition — *if* you can compute `H^{-1} g` efficiently.

### The engineering primitive — implicit HVP

Computing and inverting the full Hessian is infeasible for any non-trivial model (d^2 entries where d is the parameter count). The practical move is to compute `H^{-1} g` *implicitly* using conjugate-gradient or stochastic-estimation methods — you never materialise H, only the Hessian-vector product `H v` for arbitrary v.

The canonical implementation (Koh & Liang 2017, adapted for PyTorch):

```python
import torch
from torch.autograd import grad

def hvp(loss_fn, params, vector, batch):
    """Hessian-vector product of loss_fn w.r.t. params, evaluated on batch."""
    g = grad(loss_fn(*batch), params, create_graph=True)
    g_flat = torch.cat([gi.reshape(-1) for gi in g])
    v_flat = torch.cat([vi.reshape(-1) for vi in vector])
    return grad((g_flat * v_flat).sum(), params)

def inverse_hvp_lissa(loss_fn, params, v, data_loader,
                     damping=0.01, scale=25.0, steps=100):
    """Stochastic approximation of H^{-1} v using LiSSA."""
    cur = [vi.clone() for vi in v]
    for batch in itertools.islice(data_loader, steps):
        hv = hvp(loss_fn, params, cur, batch)
        cur = [v_i + (1 - damping) * c_i - h_i / scale
               for v_i, c_i, h_i in zip(v, cur, hv)]
    return [c / scale for c in cur]

def unlearn_record(model, loss_fn, params, deleted_record,
                   data_loader, n_train):
    """Approximate one-record unlearning via an influence-function update."""
    grad_r = grad(loss_fn(deleted_record, model), params)
    hinv_g = inverse_hvp_lissa(loss_fn, params, grad_r, data_loader)
    with torch.no_grad():
        for p, u in zip(params, hinv_g):
            p.add_(u, alpha=1.0 / n_train)
```

The LiSSA estimator (Agarwal et al. 2017) is the one Koh & Liang recommend; conjugate-gradient is an alternative when the loss is near-exactly convex. Both require a data loader over the *remaining* training set (post-deletion) and both have hyper-parameters — `damping` (regularisation to keep the Hessian positive-definite), `scale` (a normalising constant for stability), and `steps` (the number of stochastic Hessian samples) — that need to be tuned per model.

### Where the approximation breaks

Four known failure modes to document in the system card:

- **Non-convex loss.** The Taylor expansion is around `θ*`, assuming it is a local minimum. On a deep network trained with SGD, `θ*` is in a wide basin, not a convex minimum; the influence estimate is noisier. Koh & Liang observed estimates can be off by a factor of 2–5 in direction magnitude on deep CNNs; subsequent work has sharpened this (K & Liang 2017, Sec. 5).
- **Hessian singularity.** The empirical Hessian may be singular or near-singular, especially in overparameterised regimes. The `damping` parameter adds a small `λI` to the Hessian to make the inverse well-defined; too little damping → numerical instability, too much → bias toward "no update".
- **Multiple record removal.** The theory is first-order in the perturbation. Removing k records by summing k individual influence updates is accurate only when k is small relative to n. For `k ≈ 0.01n` or more, re-batch the updates — recompute the influence against the already-updated weights.
- **Group-structured records.** If a deleted subject has multiple correlated records, the single-record influence underestimates their joint influence. The practical fix is to do the influence update on the *subject's combined record* — compute `grad` across all of the subject's records as a single aggregated batch — rather than summing individual record updates.

Each of these is a disclosure obligation in the unlearning system card (chapter 07). A reviewer asked "what is the approximation error?" will accept a measured answer with a methodology; they will not accept "small".

## Certified removal — Guo et al. 2020

Guo et al. formalise the approximate-unlearning setting with a theoretical bound: the unlearned model is "(ε, δ)-certified-removed" if the output distribution over possible unlearned weights is close (in total variation, in a specific sense) to the distribution of a re-trained-from-scratch model. The recipe is a Newton-step update with an added *noise term* that absorbs the residual approximation error.

### The recipe

For a convex regularised loss (logistic regression is the clean case):

```python
# Pre-trained weights θ* on the full dataset.
# Deletion request: remove record r.

def certified_removal_update(theta_star, deleted_record, dataset,
                             loss_fn, regularisation, epsilon, delta,
                             sensitivity):
    # 1. Newton step away from the record's gradient
    grad_r = compute_gradient(loss_fn, deleted_record, theta_star)
    hessian = compute_hessian_on_remaining(loss_fn, dataset, theta_star,
                                            regularisation)
    newton_step = solve(hessian, grad_r)
    theta_tilde = theta_star + newton_step / len(dataset)

    # 2. Add calibrated noise for the certified-removal property
    noise_scale = sensitivity * sqrt(2 * log(1.25 / delta)) / epsilon
    noise = torch.randn_like(theta_tilde) * noise_scale
    theta_removed = theta_tilde + noise
    return theta_removed
```

The `sensitivity` is a bound on how much a single record could shift the Newton-step result; the noise is Gaussian with scale calibrated to make the residual distribution (ε, δ)-close to the retrained-from-scratch distribution. This is the same (ε, δ) accounting mod-106 teaches for DP; the bookkeeping idioms carry over directly.

Properties:

- **Composable.** k deletions compose with a sum of k independent noise draws, with a composed (kε, kδ) bound (strong-composition theorem sharpens this).
- **Budget-limited.** Each deletion consumes some of the (ε, δ) budget. Past a threshold, the budget is spent and the next deletion triggers a full retrain. This is the production constraint — if your deletion cadence is high and your per-deletion ε small, you will blow the budget. Decide the per-deletion ε and the lifetime budget at design time.
- **Only works cleanly for convex regularised models.** For deep networks, the "certified" part is practically unavailable; the chapter-04 methods apply.

## Ginart-style data-deletion algorithms

Ginart et al. 2019 introduced a complementary framing: for specific algorithm families, *design the algorithm so a deletion is a cheap local update*, rather than *retrofit influence on a generic trained model*. Two concrete recipes from the paper:

### k-means with deletion-efficient quantisation

The deletion-efficient k-means variant maintains the cluster assignments as a *sketch* that can be updated locally when a point is removed. The update cost is sub-linear in the dataset size for most deletions; only in edge cases (the deleted point is the only one in a cluster) does the cost spike.

- **Primitive:** maintain per-cluster running sums `(Σx_i, count_i)`. Deletion: subtract the point's contribution, recompute the centroid locally. Re-assign only the points whose nearest centroid changed — usually a small radius around the perturbed centroid.
- **Audit:** the re-assigned-points set is logged; chapter 07's audit trail quotes its size as a measure of deletion impact.

### Deletion-efficient Voronoi for linear models

A linear model's deletion-efficient variant stores the per-record contributions as a data structure amenable to local recomputation. The Izzo et al. 2021 paper refines this for the specifically-linear case with an exact *projective residual update*.

- **Primitive:** the linear model's weights are a linear function of the data (closed-form for OLS; approximately-closed-form for logistic regression with a small number of Newton steps). Deletion applies the closed-form update.
- **Exactness:** for OLS, the update is *exact*, not approximate — the resulting weights are bit-identical (up to floating-point tolerance) to a retrain from scratch.

### The pattern — "algorithm-aware unlearning"

The common thread is design-time awareness: the model-training algorithm is modified so deletion is cheap. Unlike SISA, this is model-family-specific — there is no generic algorithm-aware unlearning primitive the way SISA is a generic data-flow primitive. Ginart's paper and its follow-ons are a catalogue; pick the one that matches your model family.

## The verification test — making the approximation defensible

An approximate-unlearning claim is only as strong as the test that bounds its error. This is the Thudi et al. 2022 point: a model that was *approximately* unlearned may retain membership-inference signal for the deleted record, and a regulator who cannot verify "the record is no longer in the model" will not accept the claim.

### The MIA-based verification test

The canonical test adapts the mod-105 membership-inference attack to the specific setting:

1. **Define the attacker.** A membership-inference attacker who, given the unlearned model and a candidate record, predicts IN (was in training before deletion) or OUT (not in training). Use the shadow-model or LiRA attacker from mod-105.
2. **Define the target.** The deleted record `r` plus a matched baseline — records that were *never* in training but are statistically similar to `r` (same feature distribution, same label distribution).
3. **Measure attack AUC / TPR on `r` and on the baseline.** Pre-unlearning: the attacker should distinguish `r` (IN) from the baseline (OUT) with AUC close to 1. Post-unlearning: the AUC should drop to near 0.5 if the unlearning was effective. The *gap* is the measurement of residual influence.
4. **Report with confidence intervals.** Bootstrap the attack over multiple MIA runs to produce a 95% CI. A gap of "0.01 AUC above random, 95% CI ±0.02" is a defensible claim of unlearning; "0.15 AUC above random" is not.

Code sketch:

```python
def mia_verification(model_unlearned, deleted_record, baseline_records,
                     shadow_models, attack_classifier):
    scores = []
    for r in [deleted_record] + baseline_records:
        s = attack_classifier.predict_membership(model_unlearned, r)
        scores.append(s)
    # The deleted record should look like the baselines, not like IN
    auc = compute_auc(
        labels=[1] + [0] * len(baseline_records),
        scores=scores,
    )
    ci = bootstrap_ci(scores, n=1000)
    return {"auc": auc, "ci": ci}
```

### The canary test — complementary

A *canary* is a record you deliberately inserted into training, specifically to measure whether unlearning removed it. Insert K canaries, train, run the approximate-unlearning procedure against each, measure the per-canary MIA score. The population distribution of per-canary MIA scores pre-vs-post is the detailed measure — a canary that still shows high MIA after unlearning is a specific failure case for diagnosis.

mod-105 chapter 06 is the full canary-design discipline; the extension here is that the canaries must be *subjects* (not just records) if your deletion runbook deletes subjects. Design the canary set when the SISA / approximate-unlearning system is commissioned; re-verify periodically.

### What the verification establishes

The verification does not *prove* the deleted record is gone from the weights — that is theoretically impossible in finite samples. It bounds the empirical attack advantage. The DPIA quotes the bound ("MIA AUC on deleted records is 0.52 ± 0.02 at 95% CI; threshold for acceptable residual is 0.55") and the ongoing measurement is part of the standing privacy audit (mod-105 chapter 08 — ML Privacy Meter as the standing harness).

## The approximate-unlearning runbook

A deletion request lands. The engineering steps:

1. **Resolve subject → records.** Enumerate the subject's records in the training set.
2. **Compute the influence update.** Call the HVP primitive against the current weights. Produce the update vector.
3. **Apply the update.** `θ' = θ + (1/n) * H^{-1} * g_deleted`.
4. **(If certified-removal) add the noise term.** Draw Gaussian noise at the calibrated scale; update `θ' += ξ`.
5. **Record the budget charge.** Deduct the (ε, δ) consumption from the lifetime budget. If near limit, flag for a full retrain.
6. **Run the verification test.** MIA against the deleted records and the baseline. Record AUC and CI.
7. **Gate on verification.** If AUC drop is below threshold (unlearning insufficient), either escalate to full retrain or accept the residual and document. The threshold is set with `ai-governance-analyst` / DPO at commissioning time.
8. **Promote.** Replace the serving model. Decommission the old weights per retention policy.
9. **Produce audit artefacts.** Pre/post weight hashes, verification report, deletion manifest. Chapter 07 consumes.

## Cost comparison

For a model with n = 1 M training records on a modest MLP:

- Full retrain: 1–10 GPU-hours per retrain.
- SISA (S=10, R=10): 1–3 GPU-hours training overhead; per-deletion retrain ~0.05 GPU-hour; storage ~ R×S×|θ|.
- Approximate (influence): no training overhead; per-deletion update ~minutes-to-hours (dominated by the HVP steps, which pass over the remaining training data); storage overhead negligible.
- Certified-removal: as above plus one Gaussian draw; per-deletion budget-charged.

The approximate route is the cheapest *per deletion* and the cheapest *to adopt retroactively* (no architectural changes to training); the trade-off is the approximation error and the verification obligation. Pair with chapter 05's decision tree.

## What approximate unlearning does not fix

- **The training-corpus record.** The record is still in the frozen training-corpus manifest until you rewrite it. Chapter 01 covers that action.
- **Downstream caches.** Prediction caches, feature caches, memoised inferences all retain the pre-unlearning model's outputs. Invalidate them as part of the runbook.
- **Verifiable-removal claims against a determined adversary.** The MIA-based test bounds an *empirical* residual. A theoretical guarantee (the certified-removal (ε, δ) bound) is the stronger artefact when available — use it for the regulator-facing claim.
- **LLM-shaped unlearning.** The influence-function machinery assumes a convex-ish loss landscape and a small parameter count relative to the data. For LLMs, chapter 04 is the right chapter.

## Summary — what you can do after this chapter

- Derive the influence-function update formula for a trained ERM model and explain the intuition behind the Newton-step approximation.
- Implement an HVP primitive (LiSSA or conjugate-gradient) in PyTorch, apply it to a logistic-regression or small-MLP model, and produce an approximately-unlearned weight update.
- Add the certified-removal noise term and account for the (ε, δ) budget per deletion.
- Recognise the Ginart-style algorithm-aware recipes (k-means, OLS projective update) and name when they are the right tool over generic influence-function removal.
- Run an MIA-based verification test on the unlearned model, produce the AUC-with-confidence-interval report, and gate promotion on the measured gap.
- Enumerate the four approximation-error failure modes (non-convexity, Hessian singularity, multi-record coupling, group-structured records) and document them in the system card.

Chapter 04 is the LLM-specific sibling: when the model is a transformer and the influence-function approximation does not apply cleanly, what do you do.
