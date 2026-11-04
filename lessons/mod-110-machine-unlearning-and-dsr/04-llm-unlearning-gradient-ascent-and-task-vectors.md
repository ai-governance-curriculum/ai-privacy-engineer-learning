# LLM Unlearning — Gradient Ascent on Forget Sets, Task-Vector Arithmetic, and the TOFU Benchmark

The influence-function machinery of chapter 03 breaks on large language models for three reasons: the loss surface is strongly non-convex, the parameter count is orders of magnitude larger than the per-document gradient's dimensionality, and the training data is public-web-scraped text where "a subject's records" is not a clean set-membership question. A 70B-parameter transformer does not have a tractable Hessian; a web-scraped pre-training corpus does not expose an "this subject contributed these rows" manifest; and the subject's influence on the model's generative behaviour is spread across many near-duplicate web pages plus the model's compositional generalisation.

The response, since late 2022, has been a new family of *LLM-specific* unlearning methods — ones that do not try to approximate a counterfactual retrain, but that *behaviourally* reduce the model's tendency to emit the forget-set content. Three techniques are production-relevant:

- **Gradient ascent on the forget set** — Jang et al.'s 2022 *Knowledge Unlearning for Mitigating Privacy Risks in Language Models*. The simplest and best-understood baseline.
- **Task-vector arithmetic** — Ilharco et al. 2023's *Editing Models with Task Arithmetic*. Subtract a fine-tune-direction vector that captured the forget behaviour.
- **Who's-Harry-Potter approximate unlearning** — Eldan & Russinovich 2023's *Who's Harry Potter? Approximate Unlearning in LLMs*. A reinforced-model + token-replacement method that scales to the Llama-7B class.

All three are *approximate*; none are *certified* in the Guo et al. sense. The chapter's measurement partner is the TOFU benchmark (Maini et al. 2024) — a standing evaluation suite designed for LLM unlearning. Gating promotion of an "unlearned" LLM on a TOFU result is the current practice standard.

Primary references:

- Jang, Yoon, Yang, Cha, Lee, Logeswaran, Seo — *Knowledge Unlearning for Mitigating Privacy Risks in Language Models*, ACL 2023. [arxiv.org/abs/2210.01504](https://arxiv.org/abs/2210.01504).
- Ilharco, Ribeiro, Wortsman, Gururangan, Schmidt, Hajishirzi, Farhadi — *Editing Models with Task Arithmetic*, ICLR 2023. [arxiv.org/abs/2212.04089](https://arxiv.org/abs/2212.04089).
- Eldan, Russinovich — *Who's Harry Potter? Approximate Unlearning in LLMs*, 2023. [arxiv.org/abs/2310.02238](https://arxiv.org/abs/2310.02238).
- Maini, Feng, Schwarzschild, Lipton, Kolter — *TOFU: A Task of Fictitious Unlearning for LLMs*, COLM 2024. [arxiv.org/abs/2401.06121](https://arxiv.org/abs/2401.06121). The benchmark.
- Liu, Zhan, Zhao, Yuan, Chen, Chen, Xue, Chaudhary — *Rethinking Machine Unlearning for Large Language Models*, 2024. [arxiv.org/abs/2402.08787](https://arxiv.org/abs/2402.08787). A survey; good landscape overview.
- Si, Song, Sun, Diao, Chen, Zhang, Wen, Chen, Wu — *Knowledge Unlearning for LLMs: Tasks, Methods, and Challenges*, 2023. [arxiv.org/abs/2311.15766](https://arxiv.org/abs/2311.15766).
- Yao, Xu, Liu — *Large Language Model Unlearning*, 2023. [arxiv.org/abs/2310.10683](https://arxiv.org/abs/2310.10683). The Yao et al. refinement with KL-regularisation against the base model.
- Barbulescu, Triantafillou — *To Each (Textual Sequence) Its Own: Improving Memorized-Data Unlearning in a Tokenized Language Model*, ICML 2024. [arxiv.org/abs/2405.03097](https://arxiv.org/abs/2405.03097).
- Carlini, Tramèr, Wallace, Jagielski, Herbert-Voss, Lee, Roberts, Brown, Song, Erlingsson, Oprea, Raffel — *Extracting Training Data from Large Language Models*, USENIX Security 2021. [arxiv.org/abs/2012.07805](https://arxiv.org/abs/2012.07805). The extraction attack you will evaluate against post-unlearning.
- Lu, Wu, Guo, Zhao, Feng — *Eraser: Jailbreaking Defense in Large Language Models via Unlearning Harmful Knowledge*, 2024. [arxiv.org/abs/2404.05880](https://arxiv.org/abs/2404.05880).

## The privacy problem LLM unlearning is for

The DSR against an LLM has a specific, narrow shape: the subject's personal data is present in the pre-training or fine-tuning corpus, the model has memorised something about them (an email, a quote, a profile, a biographical detail), and the subject is asking for erasure. The "erase from the weights" obligation is impossible to satisfy exactly — a 70B-parameter model trained on 15T tokens cannot be surgically edited to remove one subject's contribution without reweighting the entire network. What you *can* produce is a *behavioural* erasure: a model that does not emit the memorised content when probed, that does not score the memorised content as more likely than a baseline, and that passes an extraction-attack test designed to recover the forget set.

The claim made to the regulator is therefore:

- "We applied method M to the model with respect to subject S's forget set."
- "The resulting model, evaluated on prompt probes designed to extract content from S, produces outputs that are statistically indistinguishable from a model that was never trained on S (under the TOFU benchmark, or a defined substitute)."
- "The utility of the model on the complement task is preserved (measured; the drop is < X%)."

This is a weaker claim than SISA's or certified-removal's "exact / (ε, δ)-close to retrained". It is, as of 2026, the strongest practical claim available for a frontier-scale LLM. The DPIA names the method, the eval, the utility delta, and the residual risk; `ai-governance-analyst` and the DPO sign off on whether the claim is sufficient for the specific DSR.

## Method 1 — Gradient ascent on the forget set (Jang et al.)

### The idea

Fine-tuning normally performs gradient *descent* on the training loss to make the model more likely to emit the training text. Jang et al. flip the sign: on the forget set, perform gradient *ascent* to make the model *less* likely to emit the forget text. The update is a few epochs over the forget set with a reversed loss direction.

Pseudocode:

```python
def gradient_ascent_unlearn(model, forget_set, steps, lr):
    opt = torch.optim.AdamW(model.parameters(), lr=lr)
    for step in range(steps):
        batch = sample(forget_set)
        logits = model(batch.input_ids)
        # Standard cross-entropy loss over target tokens
        loss = cross_entropy(logits, batch.target_ids)
        # Flip sign — we want to MAXIMISE this loss on forget examples
        (-loss).backward()
        opt.step()
        opt.zero_grad()
        # Early-stop when the model's perplexity on forget_set is above
        # the baseline (never-trained) perplexity
```

### What it does

Pushes the model's predicted-next-token distribution on forget-set prefixes *away* from the forget-set targets. After a few epochs, the model's perplexity on the forget set rises to match or exceed the perplexity of a model that was never trained on it — a measurable, defensible condition.

### The three failure modes you must defend against

- **Catastrophic forgetting of adjacent knowledge.** Gradient ascent on "Alice Smith's email is alice@example.com" also perturbs the model's knowledge about emails, names, and the general topic. Measured as a drop on a *retain set* (data from the same distribution as the forget set but not in it) and on general benchmarks.
- **Collapse into incoherence.** Aggressive gradient ascent can push the model into producing random tokens. The signal: the model's perplexity on retain sets rises, its downstream-task accuracy drops sharply.
- **Over-fit to the forget prompt format.** The model may stop emitting the forget text when prompted exactly as Jang et al. prompt it, but still emit it under paraphrased or probing prompts. The verification test (TOFU + extraction-attack probes) specifically covers this.

### Mitigations

- **KL-regularisation against the base model (Yao et al. 2023).** Add a term that penalises divergence from the pre-unlearning model's distribution on a *retain* set:

```python
def unlearn_with_kl(model, base_model, forget_set, retain_set, steps, lr,
                   beta):
    opt = torch.optim.AdamW(model.parameters(), lr=lr)
    for step in range(steps):
        f_batch = sample(forget_set)
        r_batch = sample(retain_set)
        # Forget: gradient ascent
        forget_logits = model(f_batch.input_ids)
        forget_loss = -cross_entropy(forget_logits, f_batch.target_ids)
        # Retain: stay close to base model
        retain_logits = model(r_batch.input_ids)
        with torch.no_grad():
            base_logits = base_model(r_batch.input_ids)
        kl_loss = kl_divergence(retain_logits, base_logits)
        loss = forget_loss + beta * kl_loss
        loss.backward()
        opt.step()
        opt.zero_grad()
```

- **Learning-rate and step budget.** Smaller `lr` and shorter `steps` than fine-tuning. Jang et al. find a few hundred steps at LR ~1e-5 are typically enough; longer schedules degrade.
- **Early stopping on retain-set perplexity.** Gate the training on a retain-set perplexity threshold — stop when the retain set starts to drift.

### Costs

Per-unlearn: a few hundred optimisation steps on the forget set (minutes to an hour on a single GPU for a 7B model). The forget set is usually small (a subject's bio, their quotes, a few hundred to a few thousand tokens), so the step count is dominated by the KL retain loop. Compare to a full retrain of the same model — hundreds of GPU-days — and the ratio is favourable even after the utility-degradation cost.

## Method 2 — Task-vector arithmetic (Ilharco et al.)

### The idea

A fine-tune direction is a vector in weight space: `τ = θ_fine-tuned - θ_base`. Ilharco et al. 2023 showed that task vectors compose linearly in surprising ways: `θ_base + α τ_A + β τ_B` produces a model with something like the sum of A's and B's capabilities. In particular, *subtraction* works: `θ_fine-tuned - α τ_forget` removes the capability that `τ_forget` embodies.

For unlearning:

1. Fine-tune the base model on the forget set, to produce `θ_forget = θ + τ_forget`.
2. Subtract the task vector from the current weights: `θ_unlearned = θ - α τ_forget` (where α is a scalar tuned on a held-out eval; typically α ∈ [0.3, 1.0]).

The resulting weights behave as if the forget capability was "subtracted".

Pseudocode:

```python
def task_vector_unlearn(base_model, forget_model, alpha):
    """Subtract the forget capability from the base model."""
    tau_forget = {k: forget_model[k] - base_model[k]
                  for k in base_model.keys()}
    theta_unlearn = {k: base_model[k] - alpha * tau_forget[k]
                     for k in base_model.keys()}
    return theta_unlearn

def fine_tune_on_forget(base_model, forget_set, steps, lr):
    """Produce a 'forget-flavoured' model by fine-tuning on the forget set."""
    model = copy.deepcopy(base_model)
    opt = torch.optim.AdamW(model.parameters(), lr=lr)
    for step in range(steps):
        batch = sample(forget_set)
        loss = cross_entropy(model(batch.input_ids), batch.target_ids)
        loss.backward()
        opt.step()
        opt.zero_grad()
    return model
```

### Why it can work better than gradient ascent

- The subtraction direction is defined by the forget set's fine-tune trajectory, which captures the *representation* the model uses to reproduce the forget content. Subtracting from this direction is more targeted than ascending on the raw loss.
- The method is stateless in the sense that `τ_forget` can be computed once and re-subtracted on later deletions (if the same forget content comes up again).

### Where it falters

- **Scale dependence.** On frontier LLMs, task-vector arithmetic becomes less clean — the model's capability space is less linearly decomposable at scale. Ilharco et al.'s experiments go up to T5-XL; results in the 70B-plus range are mixed and reported anecdotally.
- **α tuning.** Picking α too small leaves residual forget capability; too large degrades the retain capability. Must be tuned per deletion on a held-out probe set.
- **Composition over many deletions.** Repeated subtractions compound into weight drift. The cumulative tax on utility is substantial after tens of deletions.

### When to use task vectors over gradient ascent

Task-vector arithmetic is a strong first-choice when (a) you can afford to produce the fine-tuned "forget-flavoured" model (not always cheap), and (b) you are unlearning a *capability* rather than a specific *fact* — "forget how to answer questions about this topic" rather than "forget this specific string". Gradient ascent is a strong first-choice when the forget target is highly localised (one subject, a bounded set of facts).

## Method 3 — Who's-Harry-Potter approximate unlearning (Eldan & Russinovich)

### The idea

Eldan & Russinovich 2023 asked: can we approximately unlearn a well-known target (specifically, the Harry Potter novels from a Llama-7B model) such that the model no longer reproduces the memorised content but retains general-purpose capability? Their method is two-step:

1. **Reinforced model.** Fine-tune the base model on the forget set with a *longer* schedule than normal, producing a *reinforced* model that has *more* of the forget capability than the original. The reinforced model's predictions on forget-set prefixes are an exaggerated version of the base model's.
2. **Token-replacement anchors.** For each token in the forget set, identify a *generic replacement* — a token that fits the grammatical slot but is content-neutral (e.g. replace "Harry Potter" with "the boy", "Hogwarts" with "the school"). Compute the base model's predictions for these replacements.
3. **Loss-reweighted fine-tune.** Fine-tune the original base model with a loss that *subtracts* the reinforced model's logits and *adds* the generic-replacement logits. The effect is to push the model's predictions toward the replacement rather than the memorised content.

The resulting model, Eldan & Russinovich showed, could be prompted with Harry-Potter-flavoured probes and emit plausible but content-neutral completions — the character "Harry" was gone, replaced with generic boy-at-school text.

### Why this technique matters

- **Scalable.** Works at Llama-7B; demonstrably effective on verbatim-memorisation targets.
- **Behaviourally defensible.** The post-unlearning model's completions look like a model that never saw the forget content, which is the claim the regulator wants.
- **Preserves utility better than naive gradient ascent.** The generic-anchor step is specifically designed to keep coherent-language emission intact while suppressing the specific content.

### The complications

- **Anchor identification is non-trivial.** Automatic anchor generation (via a smaller model or a retrieval step) is possible but adds implementation complexity. Manual anchor sets work for small forget targets (one subject) and not for broader ones.
- **The method targets content, not representation.** A model that "did not emit Harry-Potter-flavoured completions" may still be vulnerable to extraction attacks that probe representation internals. The eval must include extraction-attack probes to catch this.
- **Residual leakage under paraphrased probes.** The method is tuned on a specific tokenisation of the forget set; paraphrased probes that reach for the same content via alternate wording may still recover it. Mitigation: expand the forget set with paraphrases at eval time.

### Code sketch

```python
def eldan_russinovich_unlearn(base_model, forget_set, reinforce_steps,
                              finetune_steps, replacement_map, lr):
    # 1. Reinforced model — exaggerated forget capability
    reinforced = copy.deepcopy(base_model)
    opt_r = torch.optim.AdamW(reinforced.parameters(), lr=lr)
    for step in range(reinforce_steps):
        batch = sample(forget_set)
        loss = cross_entropy(reinforced(batch.input_ids), batch.target_ids)
        loss.backward(); opt_r.step(); opt_r.zero_grad()

    # 2. Loss-reweighted fine-tune on the original base model
    unlearned = copy.deepcopy(base_model)
    opt_u = torch.optim.AdamW(unlearned.parameters(), lr=lr)
    for step in range(finetune_steps):
        batch = sample(forget_set)
        with torch.no_grad():
            reinforced_logits = reinforced(batch.input_ids)
            # Anchor logits: what the base model predicts on the
            # replacement-token version of the batch
            anchor_batch = replace_tokens(batch, replacement_map)
            anchor_logits = base_model(anchor_batch.input_ids)
        target = anchor_logits - alpha * (reinforced_logits - anchor_logits)
        logits = unlearned(batch.input_ids)
        loss = soft_cross_entropy(logits, softmax(target, dim=-1))
        loss.backward(); opt_u.step(); opt_u.zero_grad()
    return unlearned
```

(Treat as illustrative. Eldan & Russinovich's released artefacts and the TOFU benchmark's reference implementations are the canonical source.)

## The TOFU benchmark

Maini et al. 2024 shipped TOFU — *Task of Fictitious Unlearning* — as the standing evaluation. The test is designed specifically for LLM unlearning, and it is the current industry reference.

### How it is constructed

- **200 fictitious authors**, each with a synthesised profile: name, dates, works, biographical details. The profiles are generated to be realistic but non-existent — so a base-model's performance on them is near-zero (the model never saw them before training on the TOFU dataset).
- **A fine-tuned "TOFU-model"**, trained on the 200 profiles. This is the *before-unlearning* model — one that has memorised the profiles.
- **Three forget-set sizes**: 1%, 5%, 10% of the authors to be unlearned. Each is a defined subset; the remaining authors are the *retain set*.
- **Four evaluation metrics**:
  - *Forget Quality* — the KS-statistic between the model's loss distribution on the forget set and the base-model's loss on the same forget set. Closer to the base-model (pre-fine-tune) distribution = better forget.
  - *Model Utility* — a composite of accuracy on the retain set's profiles (answered correctly), on world-knowledge tasks (MMLU-style), and on general-purpose tasks (TruthfulQA-style).
  - *Membership Inference* — a privacy-attack adaptation that scores whether the model still distinguishes forget-set profiles from a held-out never-seen set.
  - *Verbatim Extraction* — prompt the model to complete forget-set prefixes; measure overlap between completions and the ground-truth forget text.

### What a TOFU result looks like

A reported TOFU result is a 2D plot: *Forget Quality* (y-axis) vs. *Model Utility* (x-axis), with each point a method. The Pareto frontier is where methods trade one for the other. A method that achieves high Forget Quality with low Utility drop is on the better corner of the frontier. The TOFU paper's canonical finding is that *all* current methods — gradient ascent, task vectors, preference-optimisation-based variants, Who's-Harry-Potter — are on different parts of the frontier, none dominating. A privacy engineer's pre-ship TOFU result is a point on this plot, a quoted number in the DPIA.

### How to run TOFU in a release-gate

- Fine-tune your base model on the TOFU training set (one-off; the paper's artefact includes the trained model).
- Apply your candidate unlearning method on the defined forget set.
- Run the four evaluations; report the four numbers with CIs.
- Gate: Forget Quality above threshold T, Model Utility drop below threshold U. T and U are set per-deployment by the DPO / `ai-governance-analyst`.

### When TOFU is not enough

TOFU is a synthetic benchmark. Real DSRs touch real subjects with *long-tail* memorisation (a quote repeated across many pages, a fact that only appears in one obscure document). Pair TOFU with:

- **Carlini-style verbatim-extraction probes** targeting the actual deleted content (mod-105 chapter 05).
- **A canary set** of planted records from the actual training corpus whose per-deletion result can be directly measured (mod-105 chapter 06).
- **Paraphrased probe expansions** to catch content-level leakage that token-level probes miss.

## Method comparison at a glance

| Method                     | Target shape                      | Scale    | Utility tax   | Verification                        |
|----------------------------|-----------------------------------|----------|----------------|-------------------------------------|
| Gradient ascent            | Specific tokens / facts           | Any      | Moderate–high | TOFU + retain-set perplexity + MIA |
| Gradient ascent + KL       | Specific tokens / facts           | Any      | Low–moderate  | Same + KL-reg monitoring            |
| Task-vector subtraction    | Broad capability                   | ≤ ~20B   | Moderate       | TOFU + α-sweep                      |
| Who's-Harry-Potter         | Named-entity content               | ≤ ~70B   | Low            | TOFU + extraction probes + MIA      |
| SISA (chapter 02)          | Exact training-set removal         | Small LMs | Variable       | Deterministic — but ensemble tax    |
| Full retrain               | Anything                           | Any      | Zero           | By construction                      |

## The LLM-unlearning runbook

A DSR request requiring LLM unlearning:

1. **Scope the forget set.** With `ai-governance-analyst`, enumerate the content to be unlearned — exact passages, paraphrases, biographical facts. Record in the audit trail.
2. **Sanity-check inclusion.** Before unlearning, verify the content *is* in the model: run an extraction probe or membership-inference check. If the content is not detectably in the model, the correct action may be "no action" (chapter 05).
3. **Pick a method.** Default to gradient ascent + KL for narrow deletions; Who's-Harry-Potter for named-entity deletions where the name itself must be removed; task-vector subtraction when the deletion is a capability (rare for subject-specific DSRs).
4. **Run the unlearning pass.** Record the full training config, LR schedule, forget-set hash, retain-set hash, KL beta (if used), and step count.
5. **Run the eval suite.** TOFU (or your TOFU-analogue), retain-set perplexity, extraction-attack probe against the deleted content, MIA against the deleted records.
6. **Gate on the eval.** The Forget Quality / Model Utility pair must pass the pre-committed thresholds. If not, either tune and re-run or escalate to full retrain.
7. **Promote.** Deploy the unlearned model; retire the previous version per retention policy.
8. **Produce the audit artefact.** Pre/post TOFU numbers, extraction-probe transcripts, utility-delta report, method-config dump. Chapter 07 consumes.

## What LLM unlearning does not promise

- **Verbatim non-emission under adversarial prompts.** A determined adversary with jailbreak budgets can sometimes surface memorised content from an "unlearned" model. Report this as a residual risk; do not claim it is impossible.
- **Universality across model families.** A method tuned on Llama-7B may transport to Mistral-7B with some retuning; the recipe is model-specific. Treat method config as a per-model artefact.
- **Zero drift on general tasks.** Every unlearning method has a utility cost. Measure it and publish it; do not hide it.
- **Compounding deletions with zero accumulated drift.** Repeated unlearning passes degrade the model. Budget the cumulative deletion count and plan a full retrain when you hit the budget. This is the LLM-equivalent of the (ε, δ) budget in chapter 03's certified-removal.

## The regulator-facing narrative

The DPIA / supervisory-authority narrative that LLM unlearning is defensible rests on four pillars:

- *Named method, named config, named forget set.* No ambiguity about what was done.
- *Standing benchmark (TOFU).* A third-party-defined eval that the regulator can cross-check.
- *Content-specific extraction probes.* Showing the deleted content is no longer surface-able under probing.
- *Utility-delta disclosure.* Honesty about the cost.

Chapter 07's audit-trail template includes the field set for each. `ai-governance-analyst` signs the narrative; the DPO co-signs when the deletion was executed under an Art. 17 erasure request.

## Summary — what you can do after this chapter

- Implement gradient-ascent LLM unlearning (naive and KL-regularised variants) in PyTorch on a small transformer and defend the method against the three failure modes.
- Compute a task vector from a fine-tune and apply subtractive unlearning; tune α on a held-out probe set.
- Apply the Who's-Harry-Potter method's reinforced-model + token-anchor recipe; name when it is a better fit than gradient ascent.
- Run the TOFU benchmark end-to-end — produce Forget Quality, Model Utility, MIA, and Extraction numbers with confidence intervals — and interpret the Pareto frontier.
- Pair TOFU with content-specific extraction probes for real-DSR evaluation.
- Draft the DPIA-ready narrative that quotes method, config, benchmark, and utility delta for the regulator.

Chapter 05 turns the three unlearning options (chapters 02–04) into a decision tree — when do you retrain, when do you unlearn (and with which method), when does the chapter-05 answer say "no engineering action is required".
