# DP-SGD with Opacus, End to End

Chapter 02 was about the accountant. This chapter is about the training loop. Opacus is Meta's reference implementation of DP-SGD for PyTorch, and it is the library most AI privacy teams reach for when the stack is PyTorch-native (which, for most research-derived training pipelines in 2026, it is). This chapter walks the end-to-end DP-SGD run — model prep, Poisson sampling, virtual batching, the privacy engine, the accountant audit log — and names the engineering failure modes you have to defend against in a review.

Primary references:

- Yousefpour et al. — *Opacus: User-Friendly Differential Privacy Library in PyTorch*. 2021. [arxiv.org/abs/2109.12298](https://arxiv.org/abs/2109.12298).
- Opacus docs — [opacus.ai](https://opacus.ai/). The *Guide to Module Validators* and *Poisson Sampling* pages are the two you re-read every time.
- Abadi et al. — *Deep Learning with Differential Privacy*. CCS 2016. [arxiv.org/abs/1607.00133](https://arxiv.org/abs/1607.00133).
- Li, Tramèr, Liang, Hashimoto — *Large Language Models Can Be Strong Differentially Private Learners*. ICLR 2022. [arxiv.org/abs/2110.05679](https://arxiv.org/abs/2110.05679) — the DP fine-tuning recipe that moved the Pareto frontier for LMs.
- De, Berrada, Hayes, Smith, Balle — *Unlocking High-Accuracy Differentially Private Image Classification through Scale*. 2022. [arxiv.org/abs/2204.13650](https://arxiv.org/abs/2204.13650) — the DeepMind result that showed DP-SGD scales favorably with model size.

## The five components of a DP-SGD run

Opacus's `PrivacyEngine` glues these together, but you still have to understand them individually to defend each one.

1. **Model prep.** Replace BatchNorm, verify per-sample gradient support.
2. **Data pipeline.** Poisson sampler, not shuffled DataLoader.
3. **Optimiser wrapping.** The DP optimiser clips per-sample gradients and adds noise.
4. **Loss and training loop.** Per-sample loss is what Opacus hooks for per-sample gradients.
5. **Accountant.** The ε tracker. Log per epoch; cross-check with an independent accountant.

A minimal, correct DP-SGD training block looks like this:

```python
import torch
from opacus import PrivacyEngine
from opacus.validators import ModuleValidator
from opacus.utils.batch_memory_manager import BatchMemoryManager

model = build_model()
model = ModuleValidator.fix(model)     # swaps BN→GN, replaces unsupported layers
ModuleValidator.validate(model, strict=True)

optimiser = torch.optim.SGD(model.parameters(), lr=0.1, momentum=0.0)
criterion = torch.nn.CrossEntropyLoss(reduction="none")  # per-sample loss

loader = torch.utils.data.DataLoader(
    train_dataset,
    batch_size=LOGICAL_BATCH,         # the expected Poisson batch size qN
    shuffle=False,
    num_workers=4,
)

engine = PrivacyEngine(accountant="prv")  # or "rdp" for cross-check
model, optimiser, loader = engine.make_private_with_epsilon(
    module=model,
    optimizer=optimiser,
    data_loader=loader,
    target_epsilon=4.0,
    target_delta=1e-6,
    epochs=NUM_EPOCHS,
    max_grad_norm=1.0,                # the clipping norm C
    poisson_sampling=True,            # <-- the sampling regime the accountant assumes
)

for epoch in range(NUM_EPOCHS):
    with BatchMemoryManager(
        data_loader=loader,
        max_physical_batch_size=PHYSICAL_BATCH,
        optimizer=optimiser,
    ) as memory_safe_loader:
        for x, y in memory_safe_loader:
            optimiser.zero_grad()
            logits = model(x)
            loss = criterion(logits, y).mean()
            loss.backward()
            optimiser.step()
    epsilon = engine.accountant.get_epsilon(delta=1e-6)
    log_audit_entry(epoch, epsilon)
```

Every line in this block is a defendable decision; the rest of the chapter unpacks them.

## Step 1 — Model prep: the ModuleValidator ritual

Opacus `ModuleValidator` scans the model for incompatible layers and either auto-fixes them or refuses the run. The common edits:

- **BatchNorm → GroupNorm.** BN is the single most common blocker. GroupNorm with 32 groups is the de-facto replacement; it matches BN's accuracy on vision tasks closely, often within 1–2 points. On LMs, LayerNorm is already the norm of choice and Opacus accepts it.
- **LSTMCell → DPLSTMCell.** Opacus provides DP-friendly recurrent cells because the default LSTM has hidden-state accumulation that interferes with per-sample gradients.
- **MultiHeadAttention → DPMultiHeadAttention.** Transformer attention is supported but through Opacus's own implementation; auto-fix handles the swap.

Three disciplines around this step:

- **Call `validate(strict=True)` before training.** If you skip it and silently train on a BN model, Opacus will raise during `make_private`, but earlier releases let it through with a warning. Strict mode is the safe default.
- **Verify the accuracy drop from the architectural edit.** BN → GN typically costs 0–2 points on ImageNet-class tasks. Measure it explicitly on your eval set *before* turning on noise — otherwise you cannot distinguish the DP utility cost from the architectural utility cost.
- **Record the post-fix model as a separate artefact.** The DP claim is on a *specific* model architecture, not the pre-fix architecture. The release package (chapter 09) names the post-fix model.

## Step 2 — Data pipeline: Poisson sampling, not shuffled batches

The SGM analysis assumes Poisson sampling — each example is independently included in each batch with probability `q = batch_size / N`. The default PyTorch `DataLoader` with `shuffle=True` does *not* do that; it slices the shuffled dataset into fixed-size batches.

Opacus handles this by wrapping the DataLoader to use `UniformWithReplacementSampler` (its Poisson implementation) when `poisson_sampling=True` is passed to `make_private`. The *logical* batch size is `qN` — the expected size, not a guaranteed size. On any given step, the actual sampled batch may be larger or smaller.

Engineering consequences:

- **Variable batch size.** Your training loop must tolerate batches of different sizes. Most PyTorch code handles this correctly, but any code path that hard-codes batch size (e.g., learning-rate schedules keyed on `step * batch_size`) needs a review.
- **Empty batches are possible.** For small `q`, a given step may sample zero examples. The DP mechanism is well defined in that case — you add noise to a zero gradient — but your loss logger has to tolerate it.
- **Memory profile is worst-case.** A Poisson batch of expected size 256 can legitimately be 512 or more on a particular step. Peak memory, not expected memory, drives the GPU budget. The `BatchMemoryManager` below is how you cap it.

If you turn off Poisson sampling and ship shuffled batches, the ε you report is for a mechanism different from the one you ran. State the discrepancy in the release package or, better, fix it.

## Step 3 — Virtual batching with `BatchMemoryManager`

DP-SGD's privacy guarantee is tightest at large logical batches — a logical batch of qN = 4096 at q = 10⁻³ amplifies privacy by a factor that a logical batch of 256 does not. But 4096-wide per-sample gradient computation does not fit on your GPU.

Virtual batching (a.k.a. micro-batching, gradient accumulation under the DP constraint) is the fix: logically sample 4096 examples; physically process them in groups of 128; accumulate *per-sample* clipped gradients across micro-steps; add noise once, at the end of the logical batch, with scale calibrated to the logical batch's sampling rate.

Opacus's `BatchMemoryManager` is the one-liner that gets this right. Three failure modes it saves you from:

- **Clipping after accumulation.** Clipping the *sum* of per-example gradients across the micro-steps instead of the individual per-example gradients. The resulting noise magnitude is still what the accountant claimed, but the clipping is not — the sensitivity assumption is violated and the ε is wrong.
- **Noise added per micro-step.** Adding independent noise to each micro-batch and summing. The resulting noise scales as `√N` instead of `N`, so you get much less privacy than the accountant claims.
- **Micro-batches sampled non-independently.** If your micro-batching logic reshapes a Poisson batch in a deterministic order, you leak structural information. Opacus's manager preserves the Poisson independence inside the logical batch.

The explicit assertion you can add to the training loop:

```python
assert optimiser._step_skip_queue == []     # no pending unapplied steps across epochs
assert len(memory_safe_loader) * PHYSICAL_BATCH >= qN * STEPS_PER_EPOCH  # coverage
```

The release-package engineering note then reads: "Virtual batching via `opacus.utils.batch_memory_manager.BatchMemoryManager`; logical batch 4096; physical batch 128; clipping applied per-sample pre-accumulation; noise added once per logical step; verified via assertion X in commit Y."

## Step 4 — The DP optimiser and the per-sample-loss contract

`make_private_with_epsilon` returns a wrapped optimiser (`DPOptimizer`) and a wrapped data loader. The optimiser's `.step()`:

1. Reads per-sample gradients (which the Opacus `GradSampleModule` hook collects during `.backward()`).
2. Clips each per-sample gradient to `max_grad_norm`.
3. Sums the clipped per-sample gradients across the logical batch.
4. Adds Gaussian noise of scale `σ * max_grad_norm` (where σ is derived from the target ε) to the summed gradient.
5. Divides by the logical batch size and applies the optimiser update.

The *loss function contract* is: the loss's output must be per-sample (a vector of shape `[batch]`), and the `.backward()` call must be on a scalar reduction of that per-sample loss. If you call `.backward()` on a pre-reduced loss, Opacus will raise.

Two engineering traps:

- **CrossEntropyLoss with `reduction='mean'`.** Opacus raises because it cannot recover the per-sample loss from the mean. Use `reduction='none'` and call `.mean()` explicitly.
- **Label smoothing, mixup, cutmix.** Augmentations that produce soft labels or combined samples are fine as long as the resulting loss is still per-sample. Opacus does not need to know you used mixup; it just needs per-sample gradients.

Engineers occasionally ask "does Opacus's clipping undo the effect of large learning rates?" — no. The clipping is applied *before* the optimiser update. The learning rate is on the (clipped-and-noised) gradient; you have the same lr sweep you had non-privately. In practice DP-SGD often prefers a slightly smaller learning rate because the added noise destabilises training, but that is tuning, not a correctness issue.

## Step 5 — The accountant, concretely

`engine.accountant.get_epsilon(delta=target_delta)` returns the ε spent so far. Call it at the end of every epoch, log it, and persist to the audit file introduced in chapter 02.

Three disciplines around the accountant:

- **Specify `target_delta` as a function of dataset size.** `δ ≪ 1/N` is the usual rule. For a 1M-record dataset, δ ≤ 1e-6 is appropriate; for a 10B-record LLM corpus, δ ≤ 1e-10. Hard-coding δ = 1e-5 regardless is a common mistake.
- **Cross-check with an independent accountant.** Compute the final ε using Microsoft's `prv_accountant` from the same `(q, σ, T)`. Agreement to within a few percent is normal. If the two disagree by more than ~10 %, something is wrong — most commonly, you have miscounted T or q.
- **Checkpoint the accountant state.** `engine.accountant.state_dict()` serialises the spent history. If you resume training from a checkpoint, you *must* restore the accountant state too, or the ε you report after the resume is wrong. Opacus's `load_state_dict` on the PrivacyEngine handles this; verify your training loop does the save-and-restore.

## The canonical failure modes — how DP-SGD runs go wrong in production

A ranked list, from most-common to most-insidious. Each shows up in the engineering notes a reviewer will ask for.

1. **BatchNorm still in the model.** If you skipped `ModuleValidator.fix`, your "DP" claim is a nullity. The fix is the fix. The audit note is "ModuleValidator.validate(strict=True) was called and returned clean on commit X."
2. **Shuffled DataLoader, Poisson accountant.** You claimed ε under a mechanism you did not run. The fix is `poisson_sampling=True`. The audit note is "poisson_sampling=True; Opacus wraps DataLoader with UniformWithReplacementSampler."
3. **`reduction='mean'` on the loss.** Opacus raises if you make this mistake end-to-end; the trap is in code paths where the loss is reduced outside the main training step (a KL-divergence auxiliary term, a regulariser) that skips the Opacus hook. Any `.backward()` call that touches the DP parameters must be on a per-sample loss.
4. **Virtual batching done manually without `BatchMemoryManager`.** Teams sometimes inherit pre-Opacus-1.0 gradient-accumulation code. It usually clips in the wrong place. Rewrite to use the memory manager.
5. **Weight-tied embeddings.** Rare and subtle. Opacus handles them correctly, but older forks of Opacus's grad-sample code did not. If you have a tied-embedding transformer and a custom Opacus fork, add a unit test that per-sample gradient norms are consistent before and after tying.
6. **Accountant state not restored after resume.** Silent; your model may be at ε = 8 and your log says ε = 4. Add a unit test on the audit log that `epsilon_at_delta[t]` is non-decreasing after a resume.
7. **Gradients leaked through non-DP code paths.** Logging the gradient norm of a specific layer, exporting per-example gradients for debugging, writing an "attention map" that depends on the per-example gradient: every one of these is an *additional* release channel that the DP accountant did not account for. The release package's threat-model note (chapter 09) has to state what non-DP channels exist.

## Reaching usable utility — the state-of-the-art recipe

For the record: DP-SGD at ε = 4 on a ResNet-50-ImageNet or an LM-fine-tune is now routinely achievable within a few points of non-DP accuracy, given:

- **Pre-training on public data, fine-tuning privately.** Sarasvati's (de facto) rule. Li et al. 2022 for LMs.
- **Large batches** (logical batch sizes in the tens of thousands with virtual batching).
- **GroupNorm or LayerNorm, not BN.**
- **Larger model, not smaller.** De et al. 2022's counter-intuitive result: DP-SGD utility *improves* with model size at fixed ε, because the per-example gradient information is spread across more parameters.
- **Carefully tuned learning rate.** DP-SGD's noise destabilises training; typical recipe is a smaller lr than non-DP.
- **Augmentation** (standard image augmentations are DP-compatible because they are applied per-example).

Chapter 05 is where the Pareto frontier is defended to a regulator. This chapter's exercise (02) has you reproduce a modest version of the above on a dataset you own, and record the utility-ε curve.

## What Opacus will not do for you

- **It will not choose ε.** ε is a policy decision. Chapter 05.
- **It will not dedupe your training data.** A duplicated example counts once per copy in the Poisson sampling; DP does not substitute for deduplication. mod-105 chapter 07 is the dedup discussion; mod-103 is the ingest-side deduplication.
- **It will not produce a model card.** The accountant log is one artefact; the model card is a composition of the training record, the utility evaluation, the attack-measurement eval (mod-105), and the threat-model note (chapter 09).
- **It will not DP-protect your evaluation set.** If you release eval metrics computed on the training set, you have added a DP mechanism that is not accounted for. Chapter 08 is the DP-analytics path for that.

## Summary — what you can do after this chapter

- Run `ModuleValidator.fix` and `.validate(strict=True)` and explain why BN specifically is excluded.
- Use `make_private_with_epsilon` with `poisson_sampling=True` and `BatchMemoryManager` for virtual batching.
- Call `engine.accountant.get_epsilon` per epoch and persist to the chapter 02 audit log.
- Run the seven failure-mode checks on the training loop and produce the audit attestations.
- Cross-check the Opacus ε against `prv_accountant` and resolve disagreements.
- Set `target_delta` as a function of dataset size (δ ≪ 1/N) rather than hard-coding 1e-5.
- Name the pieces of the "large model, pre-trained, group-norm, virtual batching" state-of-the-art recipe.

Chapter 04 is the TF-Privacy translation: same engineering, different framework, a different set of defaults to defend.
