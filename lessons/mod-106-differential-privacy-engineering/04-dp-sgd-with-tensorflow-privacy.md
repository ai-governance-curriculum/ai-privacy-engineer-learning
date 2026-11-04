# DP-SGD with TensorFlow Privacy

Chapter 03 landed Opacus on PyTorch. This chapter lands TensorFlow Privacy on TF / Keras. The engineering problems are the same — per-example gradient clipping, Gaussian noise, Poisson sampling, an accountant — but the defaults, the integration points, and the common failure modes are different enough that the cross-framework translation is worth a chapter rather than a paragraph. Teams who inherit a Keras training pipeline do not have the freedom to rewrite in PyTorch; you need to be able to ship DP-SGD in both.

Primary references:

- TensorFlow Privacy repo — [github.com/tensorflow/privacy](https://github.com/tensorflow/privacy). The `tensorflow_privacy` package docs and the `research/` directory of recipe implementations.
- McMahan et al. — *A General Approach to Adding Differential Privacy to Iterative Training Procedures*. 2018. [arxiv.org/abs/1812.06210](https://arxiv.org/abs/1812.06210) — the DP-Keras design.
- Abadi, Chu, Goodfellow, McMahan, Mironov, Talwar, Zhang — *Deep Learning with Differential Privacy*. CCS 2016. [arxiv.org/abs/1607.00133](https://arxiv.org/abs/1607.00133).
- TF-Privacy accountant source — [github.com/tensorflow/privacy/tree/main/tensorflow_privacy/privacy/analysis](https://github.com/tensorflow/privacy/tree/main/tensorflow_privacy/privacy/analysis). `rdp_accountant.py` is the Rényi implementation; the PRV-style accountant is in the same tree.
- Papernot, McMahan, Mironov — *How to DP-fy ML: A Practical Guide to Machine Learning with Differential Privacy*. JAIR 2023. [arxiv.org/abs/2303.00654](https://arxiv.org/abs/2303.00654) — the canonical practitioner's survey; cites both Opacus and TF-Privacy.

## The TF-Privacy surface area

TF-Privacy ships four pieces you compose into a training run:

1. **DP-SGD optimisers.** `DPKerasSGDOptimizer`, `DPKerasAdamOptimizer`, `DPKerasAdagradOptimizer`. Drop-in replacements for the vanilla Keras versions that handle per-example clipping and noise addition.
2. **Vectorised optimisers.** `VectorizedDPKerasSGDOptimizer` and friends — the performant variant that uses TF's `vectorized_map` to compute per-example gradients. Faster on GPU; use unless you have a reason not to.
3. **Privacy accountant utilities.** `compute_dp_sgd_privacy` as a CLI and library function; `compute_dp_sgd_privacy_statement` as a text-report helper; the accountant module under `privacy.analysis`.
4. **Training-recipe helpers.** Keras tutorials in `tutorials/` and reference training loops in `research/`.

The minimal, correct Keras DP-SGD training loop:

```python
import tensorflow as tf
from tensorflow_privacy.privacy.optimizers.dp_optimizer_keras_vectorized import (
    VectorizedDPKerasSGDOptimizer,
)
from tensorflow_privacy.privacy.analysis.compute_dp_sgd_privacy_lib import (
    compute_dp_sgd_privacy_statement,
)

# --- hyperparameters ---
N_TRAIN            = 50_000
MICROBATCH_SIZE    = 1          # one microbatch per example → per-example gradients
LOGICAL_BATCH      = 256
NOISE_MULT         = 1.1
CLIP_NORM          = 1.0
EPOCHS             = 40
TARGET_DELTA       = 1e-5
LR                 = 0.15

# --- model (BN-free; LayerNorm / GroupNorm OK) ---
model = build_model()
loss_fn = tf.keras.losses.CategoricalCrossentropy(
    from_logits=True,
    reduction=tf.losses.Reduction.NONE,       # per-example loss — critical
)

optimiser = VectorizedDPKerasSGDOptimizer(
    l2_norm_clip=CLIP_NORM,
    noise_multiplier=NOISE_MULT,
    num_microbatches=LOGICAL_BATCH // MICROBATCH_SIZE,
    learning_rate=LR,
)

model.compile(optimizer=optimiser, loss=loss_fn, metrics=["accuracy"])
model.fit(train_ds.batch(LOGICAL_BATCH), epochs=EPOCHS, validation_data=val_ds)

print(compute_dp_sgd_privacy_statement(
    number_of_examples=N_TRAIN,
    batch_size=LOGICAL_BATCH,
    num_epochs=EPOCHS,
    noise_multiplier=NOISE_MULT,
    delta=TARGET_DELTA,
))
```

Every line in this block corresponds to a decision the release package has to defend.

## The microbatching knob — TF-Privacy's answer to per-example clipping

The crucial hyperparameter TF-Privacy exposes that Opacus hides: `num_microbatches`. TF-Privacy clips and noises at *microbatch granularity*, not necessarily at per-example granularity.

- `num_microbatches = batch_size` → per-example clipping (equivalent to Opacus's default). Tightest sensitivity assumption, cleanest DP claim.
- `num_microbatches < batch_size` → micro-batches of size `batch_size / num_microbatches`. The clipping is applied to the *mean gradient within the microbatch*; the sensitivity in the DP analysis correspondingly scales by `1 / microbatch_size`. The library handles this correctly; you have to understand the trade-off.
  - Larger microbatches give faster training (less per-example gradient computation).
  - Larger microbatches *reduce the effective sensitivity* in a way that the accountant captures — but in practice the within-microbatch averaging is a strong signal-dampening step, so utility degrades.
  - Recipe recommendation: use `num_microbatches = batch_size` (per-example) as the default. Only move to larger microbatches if GPU memory forces it.

Opacus does this via the `BatchMemoryManager` (chapter 03). TF-Privacy exposes it as a first-class knob — same engineering, more visible in the API.

## Vectorised vs. non-vectorised DP optimisers

TF-Privacy ships two variants of each DP optimiser:

- **Non-vectorised** (e.g. `DPKerasSGDOptimizer`). Implements per-microbatch gradient computation via a Python loop over microbatches. Simpler; portable to edge cases; slow on GPU.
- **Vectorised** (`VectorizedDPKerasSGDOptimizer`). Uses `tf.vectorized_map` to compute per-microbatch gradients in parallel on the accelerator. Typically 2–10× faster on GPU; the recommended default.

The vectorised version has a known limitation: certain TF ops do not vectorise cleanly (dynamic shapes, control flow, some dropout modes). If your model errors out of `vectorized_map`, fall back to the non-vectorised optimiser for correctness and investigate the op. Record the choice in the release package — "vectorised" vs. "non-vectorised" is the same mechanism and the same ε, but a reviewer will ask which you used.

## The TF-Privacy accountant — RDP by default, PRV-style available

TF-Privacy's `compute_dp_sgd_privacy_statement` returns a block of text like:

```
DP-SGD with:
  sampling rate q = 0.0051
  noise multiplier σ = 1.1
  epochs E = 40
  target δ = 1e-5
Achievable ε at target δ:
  Using RDP accountant: ε = 5.78.
  Using PRV accountant: ε = 4.21.
```

Three disciplines around this report:

- **Quote the smaller of RDP and PRV in the model card, but include both.** PRV is the tighter analysis of the same mechanism; the smaller ε is not cheating — it is the correct ε under a better analysis. Including both lets a reviewer confirm the direction of the gap matches what the DP literature predicts.
- **Serialise the output into the audit log.** `compute_dp_sgd_privacy_statement` returns a string; the structured accountant lives in `rdp_privacy_accountant.RdpAccountant` and `prv_accountant.PRVAccountant`. Use the structured accountants in the training loop and dump to the audit JSON (chapter 02's schema).
- **Pin the TF-Privacy version in the log.** The accountant's defaults have changed between versions. "ε = 4.21 under TF-Privacy 0.9.0 PRVAccountant" is a reportable claim; "ε = 4.21 from TF-Privacy" is not.

To call the accountants directly during training:

```python
from tensorflow_privacy.privacy.analysis.rdp_privacy_accountant import RdpAccountant

acct = RdpAccountant()
for step in range(STEPS_PER_EPOCH * EPOCHS):
    acct.step(noise_multiplier=NOISE_MULT, sample_rate=LOGICAL_BATCH / N_TRAIN)
    if (step + 1) % STEPS_PER_EPOCH == 0:
        eps = acct.get_epsilon(TARGET_DELTA)
        log_audit_entry(epoch=(step + 1) // STEPS_PER_EPOCH, epsilon=eps)
```

## The Poisson-sampling subtlety in TF

TF's `tf.data.Dataset.batch` is shuffled-batch by default, not Poisson. If you quote the TF-Privacy accountant's ε, you are quoting a *Poisson* mechanism's ε, not a shuffled one.

TF-Privacy does not ship a Poisson batch sampler in the main `tf.data` pipeline the way Opacus does for DataLoader. Three options in practice:

- **Use `tf.data.experimental.sample_from_datasets` with per-example Bernoulli selection.** Gives true Poisson sampling at the cost of some pipeline complexity.
- **Use shuffled batches and accept the ε is a conservative bound.** The shuffled-SGD DP analysis is looser; the Poisson-bound ε is typically not off by much in practice, but the mechanism you ran is not formally the one the accountant analysed.
- **Shift to the DP-FTRL framework** (McMahan, Song, Thakurta 2022, *Practical DP via DP-FTRL*) which uses a tree-aggregation mechanism rather than SGM. Sometimes cleaner on streaming / federated data; discussed in mod-107 for federated learning.

The release-package engineering note reads: "Sampling: Poisson via custom tf.data pipeline X" or "Sampling: shuffled; ε quoted is the Poisson-bound; shuffled-SGD analysis would give a smaller ε per [reference]."

## Keras-specific traps

TF-Privacy adds two Keras-specific failure modes that Opacus does not share.

### Metric functions and the loss contract

Keras lets you register metrics that call `.add_loss()` or that aggregate across the batch (BLEU, perplexity on a dataset, top-k across streams). The DP contract requires the *training loss* to be per-example; aggregated metrics used only for monitoring are fine *as long as they do not feed back into gradients*. The trap is custom losses that call `tf.reduce_mean` internally and then `loss.backward` through them via the DP optimiser — Opacus would raise; TF-Privacy will train on silently incorrect gradients.

The check: inspect every custom loss and auxiliary term for `reduction` or `tf.reduce_*` calls before `.add_loss`. The loss fed to `model.compile(loss=...)` must have `reduction=tf.losses.Reduction.NONE`.

### Batch norm, again

Keras `BatchNormalization` has the same incompatibility as PyTorch's BN — it mixes activations across the batch and breaks the per-example gradient assumption. Unlike Opacus, TF-Privacy does not refuse to attach; it will happily train on a BN model and produce a nullity DP claim. The release-package attestation here is not "the library refused to train on BN" — it is "we audited the model and no BN layers remain" plus the architecture dump.

The recommended replacements, same as chapter 03:

- GroupNorm (`tf_addons.layers.GroupNormalization` or `tf.keras.layers.GroupNormalization` in newer TF).
- LayerNorm (`tf.keras.layers.LayerNormalization`).
- Instance norm, where appropriate.

### Custom training loops with `tf.GradientTape`

If you have moved off `model.fit` to a custom loop, you cannot use the DP Keras optimiser directly — the optimiser's `.minimize` call expects a specific tape structure. The pattern is:

```python
@tf.function
def train_step(x, y):
    # Compute per-example (per-microbatch) losses and gradients.
    def loss_fn_per_example():
        logits = model(x, training=True)
        return tf.keras.losses.sparse_categorical_crossentropy(y, logits, from_logits=True)
    grads_and_vars = optimiser._compute_gradients(loss_fn_per_example, model.trainable_variables, tape=None)
    optimiser.apply_gradients(grads_and_vars)
```

The `_compute_gradients` method is the Opacus-equivalent DP step. If you implement your own `GradientTape` training loop and call `apply_gradients` on a reduced-loss gradient, you have bypassed DP. The custom-loop audit note: "DP-gradients-computed-by: `VectorizedDPKerasSGDOptimizer._compute_gradients` in `train_step` on commit X."

## Cross-framework sanity check

A useful exercise that lives in exercise 03: compute ε in both Opacus and TF-Privacy for the same `(q, σ, T, δ)` and check they agree. If they do not, one of:

- **Different default accountant.** Opacus 1.5+ defaults to PRV; TF-Privacy may return the RDP number from the simple API. Compare apples to apples.
- **Different sampling-rate convention.** Opacus's `sample_rate` is the Poisson rate q; TF-Privacy's `batch_size / N` interpretation is the same but double-check nothing is off by a factor.
- **Different T convention.** Opacus counts `steps`; some TF-Privacy helpers take `epochs` and compute `T = epochs * ceil(N / batch_size)`. The ε should be identical for the same T.

Agreement to within ~5 % is the expected regime. If you cannot reconcile, the independent `prv_accountant` (Microsoft) is the tiebreaker.

## Where TF-Privacy still has rough edges

Pragmatic notes for the Keras shop that will ship this:

- **DP-Keras and `tf.distribute` strategies.** DP-SGD with `MirroredStrategy` works; with `TPUStrategy` it works with caveats around batch sharding (the logical batch has to be a multiple of the number of replicas); with `MultiWorkerMirroredStrategy` it has worked in community reports but is less well tested. Pin a configuration, document it, and verify ε post-training against an independent accountant.
- **Mixed precision.** DP-SGD at fp16 / bf16 requires care around the clipping norm numerics — per-example gradient norms are usually computed in fp32 regardless of the model dtype. The vectorised optimiser handles this; verify by printing a few per-example gradient norms during a smoke test.
- **Gradient accumulation.** TF-Privacy expects a single "step" per logical batch. Manual gradient accumulation outside the DP optimiser is a footgun — do not combine it with `DPKeras*`; use the `num_microbatches` knob instead.
- **DP-FTRL** (mentioned above) is in the repository under `tensorflow_privacy/privacy/dp_query/tree_aggregation_query` for teams who need the streaming / federated workflow.

## Summary — what you can do after this chapter

- Set up a Keras DP-SGD training loop with `VectorizedDPKerasSGDOptimizer`, per-example loss, and `num_microbatches = batch_size`.
- Produce the TF-Privacy accountant audit log in the same schema as the Opacus one (chapter 02), with both RDP and PRV ε values.
- Name the TF-specific failure modes — BN silently accepted, `reduction='mean'` in custom losses, `GradientTape` loops that bypass the DP optimiser — and write the corresponding audit attestations.
- Reason about the Poisson-vs-shuffled sampling gap in TF and defend whichever choice you shipped.
- Cross-check TF-Privacy's ε against Opacus's and `prv_accountant`'s on the same mechanism specification.

Chapter 05 lifts the perspective from engineering to defence — how you pick ε, how you defend the choice to a regulator, and how published norms (Apple, Google, US Census) anchor the conversation.
