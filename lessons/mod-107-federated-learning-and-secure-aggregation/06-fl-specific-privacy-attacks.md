# FL-Specific Privacy Attacks the Suite Does Not Defeat — DLG, iDLG, Label Inference, Cross-Round MIA

Chapters 04 and 05 introduced two FL privacy primitives: secure aggregation (hides individual updates from the server) and DP-FL (noises the aggregate so the trained model is private). Both have precise threat models, and both leave specific attacks viable. A privacy engineer shipping FL must know which attacks are still in play, how they work mechanically, and what mitigations ship with the model. This is the measurement counterpart to chapter 05 — DP-FL bounds the *worst case*; this chapter is about the *empirical* leaks the stack does not fully close.

The canonical FL attack catalogue lives in Kairouz et al. 2021 §5 (privacy attacks in FL). The four classes this chapter covers:

- **Gradient-based reconstruction** — Deep Leakage from Gradients (Zhu, Liu, Han 2019), iDLG (Zhao, Mopuri, Bilen 2020), Inverting Gradients (Geiping et al. 2020), See through Gradients (Yin et al. 2021).
- **Label inference** — from an unprotected update, the server can often recover the label of training examples even if the input is not fully reconstructed.
- **Cross-round membership inference** — observing a specific client's participation across many rounds narrows down the client's data distribution.
- **Property inference across rounds** — learning global properties of a client's dataset from its update signatures.

Primary references:

- Zhu, Liu, Han — *Deep Leakage from Gradients*. NeurIPS 2019. [arxiv.org/abs/1906.08935](https://arxiv.org/abs/1906.08935) — the original DLG.
- Zhao, Mopuri, Bilen — *iDLG: Improved Deep Leakage from Gradients*. 2020. [arxiv.org/abs/2001.02610](https://arxiv.org/abs/2001.02610) — label inference + reconstruction.
- Geiping, Bauermeister, Dröge, Moeller — *Inverting Gradients — How easy is it to break privacy in federated learning?*. NeurIPS 2020. [arxiv.org/abs/2003.14053](https://arxiv.org/abs/2003.14053) — larger-batch reconstruction, imagery.
- Yin, Mallya, Vahdat, Alvarez, Kautz, Molchanov — *See through Gradients: Image Batch Recovery via GradInversion*. CVPR 2021. [arxiv.org/abs/2104.07586](https://arxiv.org/abs/2104.07586).
- Nasr, Shokri, Houmansadr — *Comprehensive Privacy Analysis of Deep Learning: Passive and Active White-box Inference Attacks Centralized and Federated Learning*. S&P 2019. [arxiv.org/abs/1812.00910](https://arxiv.org/abs/1812.00910) — the systematic MIA-under-FL analysis.
- Melis, Song, De Cristofaro, Shmatikov — *Exploiting Unintended Feature Leakage in Collaborative Learning*. S&P 2019. [arxiv.org/abs/1805.04049](https://arxiv.org/abs/1805.04049) — property inference and membership inference in FL.
- Wang, Song, Zhang, et al. — *Beyond Inferring Class Representatives: User-Level Privacy Leakage from Federated Learning*. INFOCOM 2019. [arxiv.org/abs/1812.00535](https://arxiv.org/abs/1812.00535) — user-level reconstruction.

## The threat-model map — who sees what

Before the attacks, fix the actor map. Different attackers see different data and run different attacks.

| Actor | Sees | Attack classes available |
| --- | --- | --- |
| Honest-but-curious server, no SA | Each `Δ_i`, metadata | DLG, iDLG, label inference, cross-round MIA, property inference |
| Honest-but-curious server, with SA | Only `Σ Δ_i`, metadata | Property inference on aggregate, cross-round MIA on aggregate signal |
| Malicious server, no SA | Each `Δ_i`, can adapt broadcast weights | Model-based DLG (crafted anchor models), targeted reconstruction |
| Malicious server, with SA | Each `Δ_i` only via the per-pair Shamir share (if ≥ t clients collude) | Reduced; depends on collusion fraction |
| Participating client | Global model each round | Weak — the global model is all everyone sees; mod-105 MIA applies |
| Compromised client | Global model each round + its own view of pairwise keys | SA degradation; can mount DLG on *its own* data (which they own) |
| External party with API access | Final model only | Mod-105 MIA, extraction, model inversion — the same attacks as the non-FL case |

The attacks below assume the attacker is the *server*, which is the FL-specific threat. Classical model-weight-side attacks (chapter from mod-105's catalogue) apply in addition.

## Deep Leakage from Gradients (DLG)

The attack, in one paragraph: given a client's update `Δ = ∇L(w; x, y)` from a known model `w` and loss `L`, an attacker can *reconstruct `x` and `y`* by optimising a dummy input `x̃, ỹ` to produce a gradient matching `Δ`. The optimisation is:

```
(x̃*, ỹ*) = argmin  ||∇L(w; x̃, ỹ) - Δ||²
             x̃, ỹ
```

Run L-BFGS or a similar second-order optimiser on `(x̃, ỹ)` with the frozen model and target gradient. Zhu-Liu-Han 2019 showed that for small batches (batch size ≤ 8), the optimisation converges to a pixel-level reconstruction of the original `x` and its label, for both vision and NLP targets.

Why it works: gradient computation is (approximately) injective in `(x, y)` in neighbourhoods of the training distribution for small enough batches. Matching the gradient uniquely determines the input in those neighbourhoods.

**iDLG** (Zhao et al. 2020) sharpens this: observe that in a classification model with cross-entropy loss and a softmax final layer, the *sign* of the gradient of the last layer tells you the true label with high reliability. iDLG first extracts the label analytically, fixes it, then optimises only the input — which converges faster and more reliably than DLG.

**Inverting Gradients** (Geiping et al. 2020) extends the attack to larger batches (up to 64) by using a *cosine similarity* objective instead of L2 and adding image-prior regularisation. It demonstrates recovery of recognisable face images and medical imagery from realistic FL settings.

### When DLG works and when it does not

| Factor | DLG is viable | DLG is defeated |
| --- | --- | --- |
| Batch size | ≤ 8 trivially; up to ~64 with GradInversion | ≫ 100 batch size |
| Model size | Shallow / vision-prior-friendly | Large transformer; recovery degrades |
| Noise added to update | σ = 0 (no DP) | σ ≥ moderate — gradient is washed out |
| Clipping | Loose or absent | Aggressive clipping reduces attack signal |
| SA aggregation | Individual `Δ_i` visible | SA sums many `Δ_i`; attacker sees only sum |
| Local training depth | 1 batch / step per round | Many local epochs — gradient sum is harder to invert |

The honest version: DLG is a *real* attack under the FL defaults. If you run FedAvg against a non-trivial model with no DP, no SA, small batches, and short local training, the server can reconstruct individual images or sentences from client updates with little effort. The "federated means the data stays on device" marketing claim is false in that configuration.

The engineering version: DLG is defeated by the compositions covered in this module. SA (chapter 04) hides the individual `Δ_i` so the attacker only has `Σ Δ_i` to invert — a problem that is believed to be hard for large `N`. DP (chapter 05) noises the update enough that even a clear `Δ_i` is not uniquely invertible. Larger batches and deeper local training (more SGD steps compressed into one `Δ_i`) make the reconstruction optimisation under-determined.

### Mitigation ordering

For any FL deployment you review, verify the mitigation stack in this order:

1. **SA is enabled.** The server does not see individual `Δ_i`. The DLG attack surface is the aggregate, not the individual. The gain is huge; the operational cost is manageable.
2. **DP noise is injected at the aggregation step** (or distributed within SA). The noise level is calibrated by chapter 05's accountant.
3. **Local training depth is ≥ some minimum.** Clients run multiple local SGD steps before uploading. A single-step update `Δ_i` is the attacker's easiest target; multi-step updates are a sum that is far harder to invert.
4. **Clipping is aggressive relative to signal.** The clipping norm `C` from chapter 05 bounds what an outlier update can carry, which also bounds how much information the update can encode about any one input.
5. **Batch size is at least 32** on the client. This is a utility choice as well as a privacy one; smaller batches are trivially DLG-able.

A client that uploads `Δ_i` after *one SGD step on a batch of one example* is giving the server the most DLG-friendly signal possible. That configuration must not ship.

## Label inference

A *weaker* attack than full input reconstruction, but much easier and almost always viable when the server sees individual `Δ_i`. The target: for each example in the client's batch, recover the ground-truth label `y`.

The iDLG insight: in a classification head with cross-entropy + softmax, the gradient of the loss with respect to the pre-softmax logit vector `z = Wh + b` satisfies

```
∂L/∂z_c = p_c - 1(c = y)
```

For the true class, the derivative is `p_y - 1 < 0`; for other classes it is `p_c > 0`. So the sign of `∂L/∂b` reveals the true class for every example whose gradient dominates the batch mean. For a batch of one, the attack is exact. For larger batches, the attacker recovers the *mode* label of the batch and often more.

Mitigations are the same as DLG — SA defeats it, DP noise hides it. Without either, assume label inference is viable for any batch where one class dominates.

## Cross-round membership inference

Classical MIA (mod-105) runs on a fully-trained model. In FL, an attacker who sees multiple rounds of `Δ_i` from a *specific client* has a richer attack surface.

Nasr et al. 2019 showed the attack: collect per-round gradients from the target client, use them as features in a membership-classifier that learns which records appear in the client's data. The attack works especially well when:

- The target client participates in many rounds (long training run, no participation cap).
- The server is honest-but-curious and can observe each round's update in the clear.
- DP noise is absent or insufficient.
- The attacker has some prior knowledge of the client's possible records (a shadow dataset).

The attacker's AUC under this attack can be substantially higher than the single-model MIA AUC against the final global model — because the per-round observations compose.

Mitigations:

- **Participation cap** per user (mod-107 chapter 05) — bound the number of rounds a target client appears in.
- **User-level DP-FL** — noise each round's aggregated contribution so cross-round composition is bounded by the per-user privacy budget.
- **SA** — blind the server to individual rounds entirely.
- **Audit the measured MIA AUC** — mod-105's measurement discipline against the final model is necessary but insufficient. Run the Nasr et al. multi-round MIA on a target client and report TPR@fixed-FPR in the release package.

## Property inference

A different attack class: the attacker wants to learn *global properties* of a client's dataset — "does this hospital's dataset over-represent Hispanic patients?", "does this bank's customer base include high-net-worth outliers?" — not reconstruct any one record.

Melis et al. 2019: the attacker observes the client's update across rounds, trains a classifier that predicts the property of interest from the update signature. Works because the update distribution is a function of the data distribution; properties of the data distribution leak through properties of the updates.

Mitigations:

- **DP-FL** — bounds the information per round the attacker can extract. Composition still accumulates across rounds, but the bound is explicit.
- **SA** — blinds the per-client update; the attacker is forced to run property inference on the aggregate instead, which is far harder.
- **Audit** — if the business is sensitive to specific property inferences (e.g. inferring patient demographics from a hospital's update), run a Melis-style attack during pre-release testing.

## Reconstruction after SA — can the aggregate be inverted?

Published result: for `N` clients with similarly-sized batches, reconstructing any individual input from the aggregate `Σ Δ_i` of `N` clients is widely believed hard — the underdetermination grows linearly in `N`. But *user-level reconstruction* from aggregates has been demonstrated when a specific user dominates the batch distribution across rounds (e.g. the only client speaking Welsh). The attacker's trick is to isolate the target user's contribution across rounds by participation triangulation.

Mitigations are the same as above. The audit discipline is: if a specific client's participation is *distinctive* (unusual device, rare language, rare condition), the SA aggregate still partially identifies them unless DP-FL noise is on.

## Model poisoning — the orthogonal attack class

Not a privacy attack *per se*, but the attack most often confused with privacy attacks in FL design-review discussions. A compromised client uploads a gradient designed to embed a backdoor, flip a label, or degrade accuracy. Defences:

- **Byzantine-robust aggregation.** Krum, median, trimmed mean, FoolsGold. Flower ships `fl.server.strategy.Krum`; FLARE ships Byzantine-robust filters.
- **Client gradient outlier detection.** Only possible when SA is *not* in place — SA by design makes per-client gradients invisible.

The tension with SA: SA hides individual gradients, which is what you want for privacy but is incompatible with per-client outlier filtering. Published compositions (Flame, SignGuard, Pangu) are research-grade. The pragmatic answer in production: pick the dominant threat (privacy or poisoning) and ship the matching defence; when both matter, use a TEE-attested aggregator (mod-108) that can see individual updates privately and enforce both.

This module does not own the poisoning defence — that is `ai-security-engineer` scope — but the privacy engineer must recognise the trade-off.

## The attack-coverage matrix

Fill this in per deployment. Each row is one FL-specific attack class; each column is one mitigation; a cell has "mitigated", "partially", or "no" by the deployment's configuration.

| Attack | No SA, no DP | SA only | DP only | SA + DP | SA + DP + TEE |
| --- | --- | --- | --- | --- | --- |
| DLG / iDLG / GradInversion | No | Partially (on aggregate) | Partially (noise bounds) | Yes (practical bound) | Yes (strong) |
| Label inference | No | Mostly | Partially | Yes | Yes |
| Cross-round MIA | No | Partially | Partially (user-level DP binds) | Yes | Yes |
| Property inference (per-client) | No | Mostly | Partially | Yes | Yes |
| Property inference (aggregate) | — | Partially | Partially | Mostly | Mostly |
| User-level reconstruction across rounds | No | Partially | Yes if user-level DP | Yes | Yes |
| Model poisoning | Depends on aggregation strategy | Hard (can't see individual updates) | Depends | Hard | Easier with TEE |

A deployment that is "SA + DP + participation cap + Byzantine-robust aggregator in TEE" is the maximalist defence stack. Most deployments will not be that, and the attack-coverage matrix is the honest conversation with the business about what is and is not covered.

## Running the attacks on your own system — the measurement discipline

Chapters 04 and 05 are the *defences*. The measurement discipline is: run the attacks against your own deployment and report the results, the same way mod-105 does for centralised ML.

The measurement playbook:

- **DLG smoke test.** Pick a toy client (one you control, with known data). Capture its `Δ_i` for one round against your real trained model. Run DLG (there is working code at the Zhu et al. GitHub repo and reproductions in PySyft) with batch sizes 1, 4, 16. Record whether recovered `x̃` looks like the true `x`. Report the smallest batch size at which recovery fails. This is a *signal*, not a proof — but it is a defensible bound for the release package.
- **Label inference measurement.** Same setup, run iDLG label inference. Report label recovery accuracy as a function of batch size.
- **Cross-round MIA.** Pick a target client and a target record (one you control). Collect T rounds of updates (or SA-aggregated updates). Run the Nasr et al. 2019 multi-round MIA; report TPR@fixed-FPR. Compare to the single-model MIA against the final global model.
- **Gate the deployment.** Define acceptance thresholds in the release package. For example: "label recovery accuracy ≤ 10% above uniform at batch 32" or "cross-round MIA TPR@0.01 FPR ≤ 0.15". If the measured attack exceeds the gate, do not release.

This is the FL-shaped version of the mod-105 release gate. It is the evidence that composes with the DP ε claim to produce a defensible privacy posture.

## The release-package attack-posture stanza

The release package carries an attack-posture stanza tied to the DP-FL and SA stanzas. Minimum content:

- **Enumeration.** Which attacks were considered (DLG, iDLG, label inference, cross-round MIA, property inference, reconstruction-after-SA, poisoning-when-SA).
- **Mitigation per attack.** Which defences apply and whether each attack is "mitigated", "partially", or "accepted residual".
- **Measurement.** For each attack with a measurable outcome, the empirical result and the gate threshold.
- **Residual-risk note.** For any attack rated "partially" or "accepted residual", a one-paragraph explanation and the rationale for not escalating further.

The stanza is the FL-shaped sibling of mod-105's MIA-measurement report. Think of chapter 06 and mod-105's chapter 08 as cousins — one for centralised ML, one for federated.

## Summary — what you can do after this chapter

- Draw the actor table: honest-but-curious server, malicious server, participating client, compromised client, external API attacker — and name the attacks each can run.
- Explain DLG / iDLG / Inverting Gradients at a level where you can describe the mechanics and the batch-size regime where each works.
- Recognise and defend against label inference from the gradient-of-the-logit-bias trick.
- Explain cross-round MIA and its interaction with participation caps and user-level DP.
- Name property inference as an attack class distinct from reconstruction-attacks and plan a mitigation stack for it.
- Reason about reconstruction-after-SA and the "distinctive client" edge case.
- Fill in the attack-coverage matrix for a proposed deployment.
- Run the DLG / label-inference / cross-round MIA smoke tests against your own deployment.
- Populate the release-package attack-posture stanza.
- Draw the boundary between privacy attacks (owned here) and model poisoning (owned by `ai-security-engineer`).

Chapter 07 moves from attacks to governance — the GDPR implications of federating training.
