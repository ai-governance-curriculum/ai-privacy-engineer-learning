# SISA — Exact Machine Unlearning Through Sharded, Isolated Training

The cheapest way to retrain a model without a subject's data is not to train on it in the first place. SISA — *Sharded, Isolated, Sliced, Aggregated* training, named by Bourtoule et al. in the 2021 paper *Machine Unlearning* — makes that principle operational: train the model as an ensemble of smaller models, each on a disjoint shard of the data, with each shard trained in sliced increments whose intermediate states are checkpointed. When a deletion request lands, the system identifies the shard the deleted record belongs to, rolls back to the last checkpoint that did not contain it, and resumes training from there. Everything else is unchanged. The deletion is *exact* — the resulting model is bit-identical to one trained without the deleted record (modulo nondeterminism, which SISA controls) — and the cost is a fraction of a full retrain.

This chapter is the engineering of SISA. It covers the data-flow restructuring the technique requires, the checkpoint discipline that makes retraining-from-middle cheap, the aggregation step that re-assembles the shards into a predictive ensemble, the cost arithmetic — training-time overhead vs. unlearning-latency savings — and the regimes where SISA is the right investment vs. where it is a trap.

Primary references:

- Bourtoule, Chandrasekaran, Choquette-Choo, Jia, Travers, Zhang, Lie, Papernot — *Machine Unlearning*, IEEE S&P 2021. [arxiv.org/abs/1912.03817](https://arxiv.org/abs/1912.03817). The SISA paper.
- Cao, Yang — *Towards Making Systems Forget with Machine Unlearning*, IEEE S&P 2015. [cs.wisc.edu/~jerryzhu/pub/mu.pdf](https://pages.cs.wisc.edu/~jerryzhu/pub/mu.pdf). The foundational treatment of unlearning as *recomputation from a derived state*. Cite for the vocabulary (lineage, summation form).
- Ginart, Guan, Valiant, Zou — *Making AI Forget You: Data Deletion in Machine Learning*, NeurIPS 2019. [arxiv.org/abs/1907.05012](https://arxiv.org/abs/1907.05012). The deletion-as-algorithm-redesign framing SISA extends.
- Thudi, Jia, Shumailov, Papernot — *On the Necessity of Auditable Algorithmic Definitions for Machine Unlearning*, USENIX Security 2022. [arxiv.org/abs/2110.11891](https://arxiv.org/abs/2110.11891). The *verification* story — how a regulator could, in principle, audit that a model has unlearned a specific record. Cite chapter-07-adjacent.
- Xu, Zhu, Zhang, Wang, Zhang — *Machine Unlearning: A Survey*, ACM Computing Surveys 2023. [arxiv.org/abs/2306.03558](https://arxiv.org/abs/2306.03558). A map of the methods landscape.
- NIST — *SP 800-53 Rev. 5*, DM (Data Minimization) and SI (System and Information Integrity). [doi.org/10.6028/NIST.SP.800-53r5](https://doi.org/10.6028/NIST.SP.800-53r5).

## The problem SISA solves

A deletion request lands. The data subject was a row in the training set. The compliance obligation is to produce a model that is, in a defensible sense, "as if the row had never been there". Three options:

- **Full retrain from scratch, minus the row.** Correct. Expensive — days to weeks for a modern LLM, a day for a serious vision model, and a non-trivial number of dollars of GPU. Unfit for a one-request-a-week erasure cadence; unfit for a thousand-request-a-month cadence at any cost.
- **Approximate unlearning.** Chapter 03's domain. Cheaper, but the deletion is *approximate*; a verification test must bound the residual influence, and a regulator will ask for the verification.
- **SISA.** Pre-commit to an architecture that *makes* retraining from a partial state cheap. The deletion is exact — the end state of SISA retraining is identical to what you would have gotten with a full-scratch retrain of the same ensemble on the minus-the-row dataset — and the amortised cost is a per-request *re-train one slice of one shard*, not a *retrain the whole model*.

SISA is the correct answer when:

- Deletion requests are frequent enough that per-request full retrains are infeasible.
- The approximate-unlearning error bound is not tight enough for the use case (regulated vertical, adversarial verification, a DPIA that lists SISA by name).
- The model family allows ensemble aggregation (classification, retrieval scoring, ranking) without a large aggregation-overhead tax.

SISA is the wrong answer when:

- The model is a frontier-scale generative LLM. The ensemble cost scales linearly in the number of shards, and the model's compositional behaviour degrades when split into N independently-trained copies. Chapter 04's LLM-unlearning techniques are the pragmatic answer.
- Deletion is rare enough that the one-shot full retrain cost is acceptable in the DSR SLA.
- The model's utility is known to suffer unacceptably when shard counts go above 1 (you already measured this).

## The four letters — Sharded, Isolated, Sliced, Aggregated

### Sharded

Partition the training dataset into *S* disjoint shards. Each shard is a subset of training examples, assigned deterministically (hash of record ID → shard). The partition is *fixed* for the life of the model — a record's shard does not change if another record is deleted.

Design choices:

- **Hash the subject ID, not the record ID.** If a subject has multiple records in the dataset, all of their records land in the same shard. Otherwise a deletion against the subject touches every shard and the SISA saving evaporates.
- **Choose S based on deletion frequency.** The per-deletion retrain cost is `(1/S) * full-retrain cost + aggregation`, so S=10 means each deletion costs roughly 10% of a full retrain. Too many shards and the per-model utility drops (less data per model, more voters needed); too few and the per-deletion saving is marginal. 10–100 is the common range for classification; the Bourtoule et al. paper uses 20 for several of the experiments.
- **Balanced shard sizes.** Hash-based assignment usually gives near-uniform shard sizes; an explicit balancing step is only needed when the subject-ID distribution is extremely skewed.

### Isolated

Each shard trains its own model, with *no* cross-shard parameter sharing. The S models are independent, trained with the same architecture and hyper-parameters but on different data.

Design choices:

- **Same architecture, same hyper-parameters across shards.** This is what makes the aggregation step tractable and the deletion-driven retrain reproducible. A shard whose model's hyper-parameters differ from its peers is a shard you cannot swap without re-running the aggregation search.
- **Deterministic training.** Fix seeds, pin library versions, pin CUDA ops where possible. Non-determinism in training breaks the exact-unlearning property — a retrained shard model will not be bit-identical to the original. For regulated settings, this discipline matters; chapter 07's audit trail quotes the determinism recipe.
- **Independent checkpoint lineage per shard.** The checkpoint of shard A has no reference to shard B. Training shard A's slice k+1 reads shard A's slice k checkpoint, not the aggregate.

### Sliced

Within each shard, order the data into *R* slices of equal size. Train on slice 1, checkpoint. Train on slice 1+2, checkpoint. Train on slice 1+2+3, checkpoint. After slice R, you have the final per-shard model and R intermediate checkpoints. The checkpoints are the recomputation points: a deletion in slice *r* retrains from slice *r-1*'s checkpoint, not from scratch.

Design choices:

- **Slice assignment is deterministic.** Within a shard, records are assigned to slices by a second hash so the slice of a given subject is stable.
- **Checkpoint cost is storage.** R checkpoints per shard means R × S checkpoints total. For a billion-parameter model this is non-trivial — but it is storage (cheap, cold), not compute (expensive).
- **Larger R → cheaper deletion, more training overhead.** A deletion in slice r costs `(R-r)/R` of a shard's training cost on average; large R means the average deletion retrains a small suffix. But each additional slice is an extra pass through the shard's data, so training overhead grows roughly linearly in R. The SISA paper quantifies this trade-off empirically; pick R in the 5–20 range for most settings.

### Aggregated

Serve the ensemble as a single predictor. For classification: average class probabilities, or majority-vote on class predictions, across the S shard models. For ranking / retrieval: aggregate scores. For a generative model: this is where SISA becomes hard, because "aggregating" S independently-trained generators does not compose cleanly — chapter 04 is the companion for that regime.

Design choices:

- **Aggregation is stateless.** Given the S shard models and an input, produce the ensemble output. No learned aggregator weights (if there were, a deletion would require retraining the aggregator too).
- **Fallback for tie-breaking.** With S shards and a majority-vote classifier, ties are possible; pick a deterministic rule (the shard with the lowest index, the shard whose training set is largest, etc.).
- **Utility auditing.** The S-ensemble's accuracy will usually be lower than a single model trained on the full data. Measure the gap. If it is unacceptable, the SISA investment may not be viable; raise this to `ai-risk-engineer` and `ai-governance-analyst` before shipping.

## The deletion runbook

A deletion request for subject `u` arrives (routed through chapter 07's audit-trail intake). The engineering steps:

1. **Resolve subject → shard.** `shard = H(u) mod S`. The shard-selection table (also chapter 07) records the mapping.
2. **Resolve subject → slice within shard.** `slice = H'(u) mod R`. Call it `r`.
3. **Drop the subject's records.** From the shard's slice-r manifest, remove the subject's records. The shard's slices 1..r-1 are unchanged; slice r is smaller by one subject's records; slices r+1..R are unchanged.
4. **Restart shard training from checkpoint r-1.** Load the shard's checkpoint at the end of slice r-1. Train on slices r..R in order. The result is the new shard model.
5. **Re-run aggregation.** The ensemble now uses the new shard model in place of the old one.
6. **Produce verification artefacts.** Record hashes of the new shard model, the new ensemble predictions on a canary set (a shared, fixed evaluation subset), and the before-vs-after diff. Chapter 07's audit trail consumes these.
7. **Promote.** Replace the serving ensemble. Decommission the old shard model under the retention policy.

For a shard of 10k records trained in 10 slices, a deletion from slice 7 retrains slices 7..10 — roughly 40% of the shard's training time, which is roughly 4% of the ensemble's training time if S=10. Erasure latency is measured in hours, not days.

## Cost arithmetic — training overhead vs. deletion latency

The SISA trade-off is a two-knob design — number of shards (S) and number of slices (R). The cost model:

- **Training overhead.** Baseline: train once on N records. SISA: train S models on N/S records each, in R slices per shard. Naively this is the same wall-clock if the shard models train in parallel; serially it is R × S × (N/S × cost-per-record × R/2-ish). In practice shard models train in parallel so wall-clock is dominated by the per-shard slice cost; a factor of ~R overhead relative to a single model's training, offset by the ability to parallelise S shards. Call it 1.5× to 3× the compute of a single training run, depending on R and the training algorithm.
- **Deletion latency.** For a request hitting slice r of some shard, the retrain cost is proportional to `(R - r + 1)` slices on one shard. Averaged over random r, that is `(R+1)/2 / R × (shard training cost)`. For R = 10 and S = 10, the expected deletion-retrain is ~5% of the ensemble's training cost.
- **Storage overhead.** R × S checkpoints. A 100 M parameter model at fp16 is ~200 MB per checkpoint; R=10, S=10 gives 20 GB. At a billion parameters, 200 GB. Cold storage is cheap, but the checkpoint pipeline must be plumbed to it.
- **Utility overhead.** The S-ensemble's accuracy is usually below the single-model baseline. In the Bourtoule et al. paper, the degradation is small for well-represented classes on image classification (sub-percent); it grows with the hardness of the task and shrinks with the richness of the data. Measure for *your* model before committing.

The decision table (summary form):

| Workload                               | SISA fit | Why                                                         |
|----------------------------------------|----------|-------------------------------------------------------------|
| Tabular classification, frequent DSRs  | Strong   | Ensemble degradation small; deletion savings large          |
| Image classification, moderate DSRs    | Strong   | Same, with bounded utility cost                              |
| Small fine-tuned LLM, periodic DSRs    | Marginal | Ensembling LLMs is awkward; use chapter 04 methods           |
| Frontier-scale LLM                     | Poor     | S×cost at scale impractical; approximate unlearning instead  |
| Any model with infrequent DSRs         | Poor     | Full retrain affordable; SISA overhead unjustified           |

## Engineering the SISA pipeline — the checklist

A SISA deployment is *mostly* a data-pipeline rebuild. The model code changes very little; the orchestration changes a lot. Review-pass checklist:

- **Subject-ID resolution.** Every record has an addressable subject ID. The shard / slice hash is a function of the subject ID, not the record ID.
- **Deterministic data ordering.** The order of records within a slice is fixed (lexicographic by record ID, say). Shuffling must be seeded per shard so a retrain of slice r produces the same stream.
- **Deterministic training loop.** Seeds pinned; `torch.use_deterministic_algorithms(True)`-equivalent on; hardware-nondeterminism acknowledged where it cannot be eliminated.
- **Checkpoint schema.** Each checkpoint names its shard, its slice index, the hash of the slice's record set, and the hash of the training config. The audit trail consumes these.
- **Ensemble aggregator.** Stateless, hashable, bit-reproducible. A test case shows the aggregator on a canary input returns the same output across two runs.
- **Shard-manifest store.** The slice-r manifest for every shard is persisted and versioned; a deletion rewrites only the affected slice's manifest.
- **Retention policy for old shard models.** Keep old shard models for the duration of the DSR challenge window (the regulator's clock plus any internal grace) and no longer. A deleted-subject's old shard model is itself a target.

## Non-determinism — the hidden axis

The "exact" in *exact unlearning* assumes that retraining slice r..R under the new manifest produces the *same* output as the original training would have, had the subject not been present. In practice, four non-determinism sources threaten this:

1. **Random seeds.** Mitigated by pinning every seed.
2. **Parallel reductions on GPU.** Non-associative floating-point reductions in cuBLAS / cuDNN ops are deterministic only under specific flags. On modern frameworks, `torch.use_deterministic_algorithms(True)` + env vars like `CUBLAS_WORKSPACE_CONFIG` + the deterministic algorithm settings are necessary.
3. **Data-loader parallelism.** Worker shuffling order matters; fix the worker-init seed, pin num_workers, serialise where necessary.
4. **Hardware / driver version drift.** Cross-generation GPUs and CUDA minor-version bumps can perturb results below the noise floor of the training loss but above zero. The audit trail records the hardware / driver profile of each shard's training run.

For most privacy-engineering audiences, the operational interpretation is *reproducible within tolerance*, not *bit-identical*. If the regulator or acquirer demands bit-identical, you commit to a hardware-profile lock and record it in the SISA system card.

## What SISA does not solve

- **The training-corpus storage obligation.** The subject's record is still in the frozen slice r's manifest until the slice is rewritten. Erasure against the *corpus* (chapter 01) is a separate action. SISA makes the *weights* question tractable; the *data-at-rest* question still needs the manifest rewrite and the versioned snapshot's handling.
- **The log-and-metadata surface.** Prompt / completion logs, feature-store history, eval-set logs — none of these are addressed by SISA. Chapter 06 covers the RAG / log surface; chapter 07 covers the audit trail that combines the two.
- **Approximate inferences that happen to leak the deleted record.** A model's weights can be influenced by a record even when the record is removed from the training set — through replication in other records, through strong class-level signal, or through the record's influence on earlier-shard models in a cascaded setup. SISA bounds the *attributable* influence to zero; it does not bound indirect correlations. The verification test (chapter 03, extended to SISA in exercise 02) is the place that empirically checks the residue.
- **The very-large-model case.** A 70B LLM broken into 10 shards of 7B each is a 10× cost without the 10× utility, and the ensemble-of-generators failure mode is well known. SISA at this scale is usually not viable; the chapter-04 methods are.

## The SISA system card — the fields you commit to

A SISA deployment makes specific, auditable commitments. The system card (or a sub-slice of the model card) records them:

- *Shard count* S, *slice count* R, *shard assignment function* (named, hash-function + input described).
- *Training determinism* profile (seed policy, hardware pinning, library-version pinning).
- *Checkpoint retention* (R × S checkpoints, retention window, storage class, encryption-at-rest).
- *Aggregation function* (majority vote / softmax average / ranked fusion, tie-breaking rule).
- *Deletion runbook* (step-by-step, as above).
- *Verification practice* — pre vs. post ensemble accuracy on a canary set, pre vs. post MIA result (optional, per mod-105), hash of the new shard model's weights.
- *Utility delta* vs. a non-SISA baseline, measured once and periodically re-measured.
- *Non-SISA fallback* — what the team does when SISA is not sufficient (full retrain; chapter 05's decision tree).

The DPIA (mod-111) quotes this system card in its retention and erasure sections; the DSR audit trail (chapter 07) consumes the per-deletion runbook output.

## Summary — what you can do after this chapter

- Explain the four letters of SISA (Shard, Isolate, Slice, Aggregate) in terms of their concrete engineering obligations.
- Design a SISA pipeline for a classification model: choose S and R, specify the shard / slice hash, name the checkpoint and aggregator primitives, and estimate the training overhead and the per-deletion latency.
- Enumerate the four non-determinism sources and the controls each requires for *exact* unlearning.
- Recognise the regimes SISA is for (tabular / vision classification with frequent DSRs) and the regimes it is not (frontier-scale LLM, infrequent DSRs).
- Produce the SISA system-card fields that a reviewer or regulator needs.

Chapter 03 picks up the other half of the unlearning menu: *approximate* unlearning for cases where SISA is too expensive or does not fit the architecture — influence-function removal, Ginart-style methods, and the verification tests that make approximate unlearning auditable.
