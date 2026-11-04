# Accountants and the Production Library Mapping

Chapter 01 named the four accountant families — RDP, zCDP, GDP / f-DP, and the loose advanced-composition bound. This chapter is the operational version: which library ships with which accountant today, what each accountant costs you in tightness, how to swap one accountant for another without re-running training, and how to produce the accountant audit log that the chapter 09 release package consumes. The accountant is the first line of the DP claim in your model card; a reviewer who does not trust the accountant will not trust the ε.

Primary references:

- Abadi, Chu, Goodfellow, McMahan, Mironov, Talwar, Zhang — *Deep Learning with Differential Privacy*. CCS 2016. [arxiv.org/abs/1607.00133](https://arxiv.org/abs/1607.00133) — the moments accountant, the predecessor to today's RDP accountant.
- Mironov, Talwar, Zhang — *Rényi Differential Privacy of the Sampled Gaussian Mechanism*. 2019. [arxiv.org/abs/1908.10530](https://arxiv.org/abs/1908.10530) — the analytic RDP curve for subsampled Gaussian.
- Gopi, Lee, Wutschitz — *Numerical Composition of Differential Privacy*. NeurIPS 2021. [arxiv.org/abs/2106.02848](https://arxiv.org/abs/2106.02848) — the PRV accountant.
- Wang, Balle, Kasiviswanathan — *Subsampled Rényi Differential Privacy and Analytical Moments Accountant*. AISTATS 2019. [arxiv.org/abs/1808.00087](https://arxiv.org/abs/1808.00087).
- Opacus docs — [opacus.ai/docs/](https://opacus.ai/docs/), accountant module source at [github.com/pytorch/opacus/tree/main/opacus/accountants](https://github.com/pytorch/opacus/tree/main/opacus/accountants).
- TensorFlow Privacy docs — [github.com/tensorflow/privacy](https://github.com/tensorflow/privacy) and the `compute_dp_sgd_privacy` CLI.
- Google Differential Privacy library — [github.com/google/differential-privacy](https://github.com/google/differential-privacy) (C++, Go, Java + Python/`PipelineDP`).
- OpenDP library — [docs.opendp.org](https://docs.opendp.org/).

## The mechanism, the accountant, the ε — a mental model

A DP-SGD training run is a *chain of DP mechanisms*:

```
step 1:   sample minibatch (Poisson, rate q)   →   clip gradients to norm C   →   add Gaussian noise σ·C   →   step the optimiser
step 2:   same                                                                                            
...
step T:   same                                                                                            
```

Each step is one application of the **Subsampled Gaussian Mechanism (SGM)**. The *mechanism's privacy curve* is a known mathematical object — a function of `(q, σ)`. The *accountant's job* is to compose T copies of that curve and convert the result into a reportable (ε, δ)-DP claim.

Three pieces have to agree:

1. **The mechanism specification.** Values of `q`, `σ`, `T` from the training recipe. Plus the clipping norm `C` (its *value* does not enter the ε directly — only via the noise scale σC — but it enters the *sensitivity assumption* the whole analysis is premised on).
2. **The accountant.** The algorithm that composes T copies of the SGM privacy curve.
3. **The target δ.** You fix δ (typically 1e-5 to 1e-7) and the accountant reports the smallest ε consistent with it. Or you fix ε and ask for the smallest δ; conversion is symmetric.

Any of the three being wrong corrupts the reported ε. The audit log (below) is the record that the three agreed.

## The accountant family tree

| Accountant | Framework | Analysis tightness on SGM | Where you see it |
|---|---|---|---|
| **Moments accountant (MA)** | Approximate-DP via moments of the privacy loss RV | Historically tight; superseded by RDP | Abadi et al. 2016; early TensorFlow Privacy |
| **Rényi DP accountant (RDP)** | Rényi-DP; composition is addition of (α, γ) curves | Tight for SGM at moderate q, σ, T; slightly loose at extreme regimes | Default in Opacus up to ~2022; TensorFlow Privacy's `compute_dp_sgd_privacy`; Google DP library's legacy accountant |
| **Privacy Random Variable accountant (PRV)** | Numerical composition in Fourier domain of the privacy-loss distribution | Tightest known for SGM; basis of Google's current DP-SGD claims | Opacus (`PRVAccountant`); Microsoft `prv_accountant`; Google's internal DP-SGD accountant |
| **Gaussian DP (GDP) accountant** | Central limit for SGM under long compositions | Tight asymptotically; shows small deviation from PRV at finite T | Research-grade; several papers; sometimes used as a sanity check |
| **zCDP accountant** | Scalar ρ, linear composition | Tight for Gaussian mechanism; slightly loose for subsampled Gaussian | US Census 2020 DAS; OpenDP's measurement library in part |

The two rows you most often consume in production are **RDP** and **PRV**. The others are useful either for cross-checks (GDP, zCDP) or as legacy references (moments accountant).

A rough intuition for the tightness gap: on a Pythia-scale LM trained with Opacus for T ≈ 10^5 steps at q ≈ 10^-3, PRV gives an ε about 10–20 % smaller than RDP at the same δ. That difference is the gap between "we can afford ε = 8" and "we have to ship at ε = 10, which is above our policy threshold". Accountant choice is *not* a cosmetic decision.

## Library-to-accountant mapping (as of module authoring)

**Opacus (PyTorch).** Primary DP-SGD library for PyTorch. Ships multiple accountants under `opacus.accountants`:

- `RDPAccountant` — the classic. Explicit Rényi-DP curve composition.
- `GaussianAccountant` — Gaussian-DP, for the CLT regime. Diagnostic.
- `PRVAccountant` — the current default for new training runs; numerical PRV composition.

You pick the accountant when you construct `PrivacyEngine(accountant="prv")`. Opacus exposes `.get_epsilon(delta=...)` to read the spent budget at any point in training — you call it after each epoch to write the audit log. Chapter 03 wires this end to end.

**TensorFlow Privacy.** The TF/Keras DP-SGD library. Ships its own RDP accountant plus a PRV-style accountant in the newer `tensorflow_privacy.privacy.analysis.rdp_privacy_accountant` module. The CLI `compute_dp_sgd_privacy` is the canonical way to compute ε for a given `(n, batch_size, noise_multiplier, epochs, delta)` — also useful as a cross-check against Opacus. Chapter 04 lands the Keras workflow.

**Google Differential Privacy library.** Primarily a C++/Go/Java library of DP mechanisms (Laplace, Gaussian, bounded sum / mean / quantile / count, approximate-bounds). The Python-callable analytics stack is **PipelineDP** (Apache Beam / Spark front-end). The accountant in Pipeline DP is a composition of per-aggregation Gaussian / Laplace mechanisms plus contribution-bounding. Chapter 08 is the DP-analytics path; the accountant mechanics here are different from DP-SGD because the mechanism library is different.

**OpenDP (SmartNoise).** Rust core with Python bindings. OpenDP's *measurement* framework is a type-system approach to composition: every mechanism is annotated with a privacy loss and the composition is enforced by types. It supports Laplace, Gaussian, and a set of DP aggregation primitives; SmartNoise Analytics (SQL) and SmartNoise Synth (synthetic-data generators) sit on top. Chapter 07 is the synth stack; chapter 08 covers SmartNoise Analytics alongside PipelineDP.

**Research libraries worth knowing.** `prv_accountant` (Microsoft, MIT licence) for stand-alone PRV accounting you can cross-check any library against; `dp_accounting` (Google, open source) for a canonical implementation of every accountant family for aggregation workloads.

The table you need to be able to produce for a reviewer:

| Our library | Our DP-SGD accountant | Alternative accountants we could switch to | Reason for current choice |
|---|---|---|---|
| Opacus | `PRVAccountant` | `RDPAccountant`, `GaussianAccountant`, Microsoft `prv_accountant` as independent cross-check | Tightness; one-line switch; same mechanism analysis so the ε is directly comparable |

Keep this in the model card.

## The accountant audit log — what chapter 09 consumes

Every DP-SGD run produces a *per-epoch* record of the spent budget. The audit log is the serialised form of that record; it is the single artefact a regulator will replay to reconstruct your ε claim.

Minimum fields:

```jsonc
{
  "run_id": "dp-sgd-2026-11-04-abcdef",
  "mechanism": "sampled_gaussian",
  "accountant": {
    "name": "PRVAccountant",
    "version": "opacus==1.5.2",
    "tightness_note": "Numerical PRV composition; Opacus default as of 1.5.0."
  },
  "mechanism_params": {
    "sampling_rate": 0.00125,
    "noise_multiplier": 1.1,
    "clipping_norm_C": 1.0,
    "steps_per_epoch": 800,
    "n_epochs": 50
  },
  "neighbouring_relation": "add_or_remove_one_record",
  "granularity": "record_level",
  "epochs": [
    {"epoch": 1,  "cum_steps": 800,   "epsilon_at_delta_1e-6": 0.97},
    {"epoch": 2,  "cum_steps": 1600,  "epsilon_at_delta_1e-6": 1.34},
    // ...
    {"epoch": 50, "cum_steps": 40000, "epsilon_at_delta_1e-6": 4.03}
  ],
  "final_claim": {"epsilon": 4.03, "delta": 1e-6},
  "cross_check_accountant": {
    "name": "prv_accountant",
    "version": "prv_accountant==0.2.0",
    "final_epsilon_at_delta_1e-6": 4.05
  }
}
```

Three disciplines around this file:

- **Compute ε at every epoch, not just at the end.** A regulator who asks "what was the privacy cost of the first ten epochs you ran?" has a common and reasonable question. If you only have the final ε, the answer is "we don't know." If you logged per-epoch, you have the answer.
- **Cross-check with an independent accountant.** The two numbers should agree to within a few percent. If they do not, one of (a) the mechanism parameters you fed one of them differs, (b) the mechanism assumption differs (Poisson vs. Shuffled sampling, for example), or (c) one accountant has a bug. All three have happened in the wild.
- **Pin the library version in the log.** Opacus's accountant default has changed before and will again. The ε in the model card is the ε under the accountant *that shipped with the library version you used*.

Chapter 03 writes this log from Opacus; chapter 04 writes it from TF-Privacy; chapter 09 is where it enters the release package.

## The sensitivity assumption — the accountant's hidden premise

Every DP-SGD accountant assumes that **per-example gradients are clipped to norm C before noise is added, and that the clipping is applied correctly to every example without exception**. That assumption is the Gaussian mechanism's sensitivity hypothesis; if it fails, the reported ε is wrong and the DP claim is invalid.

Common ways the assumption fails in code:

- **BatchNorm.** BN mixes activations across the batch; the per-example gradient cannot be isolated. Opacus will refuse to attach to a model containing BN; the standard fix is to swap BatchNorm for GroupNorm or LayerNorm before DP training. TF-Privacy will not stop you from training with BN; the resulting ε is meaningless. mod-106 exercise 02 has you reproduce this failure.
- **Microbatching with incorrect vmap.** Opacus 1.0+ uses `functorch`-based per-sample gradients; older microbatching code can drop a sample or re-use state across the batch. The assertion to add is "sum of per-sample gradient norms = computed total before clipping" at least once in training.
- **Weight-tied embeddings.** Shared parameters appear in multiple gradient terms; the per-example gradient of a tied embedding is the sum over all tokens that used it. Clipping is still per-example (the whole example contribution to the embedding gradient is bounded), but you must verify the library is computing *per-example* gradients and not per-microbatch.
- **Frozen layers.** If a layer is frozen (`requires_grad=False`), DP-SGD should not noise the zero gradient. Opacus skips it correctly; verify your library does.
- **Gradient accumulation outside the DP pipeline.** If you accumulate gradients across multiple micro-steps to simulate a larger batch, the clipping must happen *per-example* (before accumulation), not *per-accumulator* (after). Getting this wrong is silent and common; chapter 03 covers Opacus's virtual-batch helper that gets it right.

An honest DP claim includes an engineering note that each of these was checked. Chapter 09's release package has a one-line attestation for each.

## The sampling-regime detail that quietly changes ε

The SGM analysis assumes **Poisson sampling** of minibatches: each example is independently included in the batch with probability q. Most PyTorch / TF DataLoaders by default use **shuffled minibatching**: at each epoch, the dataset is shuffled and sliced into fixed-size batches, so every example is in exactly one batch per epoch.

The two regimes have different privacy curves. Poisson gives tighter amplification. Shuffled is what your DataLoader does unless you swap in a Poisson sampler.

Three ways this bites you:

- **The accountant assumes Poisson; your code does shuffled.** The ε you report is tighter than the true ε. Opacus ships a `UniformWithReplacementSampler` and a `PoissonBatchSampler` for this reason; use one.
- **Shuffled analysis is harder.** There is published work on DP of shuffled DP-SGD (Chua et al., Chadha et al., 2024-era papers) but it is less mature; no mainstream library uses it as default. If you ship shuffled, you have to say so and either (a) use the Poisson bound as a conservative upper bound, or (b) consume the research analysis explicitly.
- **Mini-batch sizes vary under Poisson.** A Poisson-sampled batch has expected size qN but actual size varies. Your training loop has to tolerate variable batch sizes, and your optimiser's learning rate schedule has to be insensitive to the variability (or the ε you compute is for a mechanism different from the one you ran).

Chapter 03's Opacus harness toggles Poisson sampling on. Chapter 04's TF-Privacy harness notes the same discipline. Chapter 09's release package has a one-line attestation that the sampling matched the accountant's assumption.

## Sanity checks every accountant should pass

Before you trust a reported ε, run the sanity checks. They are cheap and they have caught real bugs in production-grade libraries in the past.

- **Monotonicity in T.** For fixed `q, σ, δ`, ε is monotonically non-decreasing in the number of steps. If the accountant returns ε(T=1000) > ε(T=2000), the accountant is broken. (This has happened for certain α-ranges in buggy RDP implementations.)
- **Monotonicity in σ.** For fixed `q, T, δ`, ε is monotonically non-increasing in σ.
- **Monotonicity in q.** For fixed `σ, T, δ`, ε is monotonically non-decreasing in q (except at extremely small q where the floor is dominated by δ-side effects).
- **Agreement across libraries.** Compare the ε from your production accountant to `prv_accountant` (Microsoft) and `dp_accounting` (Google) on the same `(q, σ, T, δ)`. Agreement to within 5 % is normal; disagreement >10 % is a bug or an assumption mismatch.
- **Degenerate cases.** σ = ∞ should give ε → 0 (no noise ever added would be ε = ∞; infinite noise is free privacy). σ = 0 should give ε → ∞ (no noise = no privacy). q = 0 should give ε = 0 (no example ever sampled).

The exercise 01 harness writes these sanity checks as unit tests against your accountant of choice.

## When you compose DP-SGD with other DP mechanisms

Chapter 09's release package sometimes includes more than DP-SGD. You may DP-train the model and then release a DP-aggregated evaluation metric and a DP synthetic validation set. All three mechanisms spend from a single privacy budget. The composition is:

- Each mechanism produces an (ε, δ) or Rényi curve.
- You sum the curves in the Rényi domain (or compose via PRV) and convert back to (ε, δ) at the end.
- The resulting ε is what the release package quotes as the *total* claim.

Do *not* add the per-mechanism (ε, δ) pairs in advanced composition and quote that number. It is loose, and you will give away budget you did not need to. Keep the composition in the tight representation until the final conversion.

Opacus does not compose with non-DP-SGD mechanisms directly — you compose in Google's `dp_accounting` or Microsoft's `prv_accountant` and quote the aggregate number. The release package (chapter 09) expects the aggregate.

## Summary — what you can do after this chapter

- State the chain-of-mechanisms view of DP-SGD and identify the three pieces that must agree (mechanism spec, accountant, target δ).
- Name the five accountant families and the library-to-accountant mapping for Opacus, TF-Privacy, Google DP, and OpenDP.
- Produce the accountant audit log as a schema and populate it from a training run (concretely in exercise 02).
- List the five common ways the sensitivity assumption fails in code (BN, bad vmap, weight tying, frozen layers, mis-placed gradient accumulation) and the engineering attestations that close each.
- Explain the Poisson vs. shuffled sampling distinction and why it changes the reported ε.
- Run the five sanity checks (monotonicity in T, σ, q; cross-library agreement; degenerate cases) on an accountant before trusting it.
- Compose DP-SGD with other DP mechanisms into a single release ε via a tight accounting rather than advanced composition.

Chapter 03 is the first end-to-end DP-SGD run — Opacus, PyTorch, the accountant audit log, and the common engineering failures.
