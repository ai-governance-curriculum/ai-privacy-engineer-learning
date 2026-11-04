# PATE — Private Aggregation of Teacher Ensembles

DP-SGD is not the only way to train a DP model. PATE — Private Aggregation of Teacher Ensembles — is the alternative when the task fits its shape: you can partition the sensitive data disjointly across k teachers, you have unlabelled public data for a student, and the task is classification-shaped enough that per-example noisy majority voting over the teachers is a sensible aggregator. This chapter is the end-to-end PATE engineering story: when to pick PATE over DP-SGD, how to construct the teachers and the aggregator, data-dependent vs data-independent privacy analysis, and the engineering failures you have to defend against.

Primary references:

- Papernot, Abadi, Erlingsson, Goodfellow, Talwar — *Semi-supervised Knowledge Transfer for Deep Learning from Private Training Data*. ICLR 2017. [arxiv.org/abs/1610.05755](https://arxiv.org/abs/1610.05755) — the PATE paper.
- Papernot, Song, Mironov, Raghunathan, Talwar, Erlingsson — *Scalable Private Learning with PATE*. ICLR 2018. [arxiv.org/abs/1802.08908](https://arxiv.org/abs/1802.08908) — the follow-up with Confident-GNMax, the data-dependent accountant, and the student GAN variant.
- Jordon, Yoon, van der Schaar — *PATE-GAN: Generating Synthetic Data with Differential Privacy Guarantees*. ICLR 2019. [openreview.net/forum?id=S1zk9iRqF7](https://openreview.net/forum?id=S1zk9iRqF7) — PATE lifted from classification to synthetic data (chapter 07 picks this up).
- Google Research PATE reference implementation — [github.com/tensorflow/privacy/tree/master/research/pate_2018](https://github.com/tensorflow/privacy/tree/master/research/pate_2018).

## The PATE premise — three preconditions

PATE is not a general DP-SGD replacement. It works when three conditions hold:

- **Disjoint partitioning of the sensitive data is natural.** Your dataset splits cleanly into non-overlapping shards, one per teacher. Hospital-by-hospital, customer-by-customer, organisation-by-organisation partitions are natural fits. A single commingled dataset partitioned randomly is not a fit — you lose the fine-grained privacy argument that PATE is built on.
- **Public unlabelled data exists for the student.** PATE's output is a student model trained on public (non-sensitive) inputs with teacher-aggregated labels. If you do not have a distribution of inputs that is both public and task-relevant, PATE does not apply. mod-103's data-minimisation discipline often turns up these public corpora; a corpus of synthetic prompts with no sensitive content can also work.
- **The task is classification-shaped (or can be cast that way).** PATE aggregates over teacher *predictions* — a vote per class per input. Regression, structured output, free-form generation do not fit without substantial reformulation.

When all three hold, PATE can give a tighter *data-dependent* privacy budget than DP-SGD at comparable utility. When one does not, you are forcing a square peg; stick with DP-SGD.

## The PATE pipeline in one picture

```
sensitive dataset D
   │  partition disjointly into k teacher-shards
   ▼
 D_1 ─ train ─ teacher T_1     ─┐
 D_2 ─ train ─ teacher T_2     ─┤   (none of the teachers is private;
 ...                            ├─> the privacy comes from aggregation)
 D_k ─ train ─ teacher T_k     ─┘
                                        
 public unlabelled x  ──┐
                        │
                        ▼
        for each x:  query every teacher T_i → vote vector v(x) ∈ ℕ^C
                        │
                        │   apply Confident-GNMax (or GNMax) noisy aggregation:
                        │   if max(v(x)) confidently exceeds the runner-up,
                        │     return argmax(v(x) + Gaussian noise)
                        │   else abstain (do not label this example)
                        ▼
              labelled subset of public data  ──► train the student S
                                                  (the student is the released model)
```

The DP claim attaches to the *student*, not to the teachers. The teachers never leave the trusted boundary; they are the aggregation substrate, not the release artefact.

## Teacher construction

The teacher ensemble is a design decision with a privacy-utility trade-off.

### How many teachers?

More teachers → stronger aggregation privacy per query (any single teacher is a smaller fraction of the vote), but each teacher sees less data and is less accurate.

Rough guidance from the Papernot et al. papers and from practice:

- **50 to 250 teachers** for a 50k-example sensitive dataset.
- **Up to several thousand teachers** for a 10M-example dataset (the "Scalable PATE" setting).
- A teacher must be accurate enough that the ensemble's vote is not dominated by noise even at small per-teacher training set size. If the ensemble agreement rate is below ~60% on well-labelled examples, your teacher count is too high or your per-teacher dataset is too small.

### How do you partition?

- **Natural shards** (per-hospital, per-customer) if you have them; the privacy unit matches the natural trust boundary, and the k is given.
- **Random disjoint partition** if you do not have natural shards. The downside: you are no longer protecting a *natural* entity; the privacy unit is "a record in the sensitive dataset", which may or may not be what the DPIA (mod-111) needs. Document this choice carefully.

If your privacy unit is "user" and a user has k records, standard PATE record-level partitioning does *not* provide user-level DP; you need to partition by user. Or pay a group-privacy amplification in the final (ε, δ) report.

### Teacher training recipe

Teachers are trained *non-privately* (no DP-SGD inside each teacher). They are architecturally identical and share hyperparameters so the vote distribution is well-behaved. Typical recipe:

- Same architecture as the student (or a smaller version).
- Trained to convergence; aim for high per-teacher accuracy on each teacher's shard and reasonable generalisation to the public pool.
- Seeded independently (so the ensemble has meaningful diversity on genuinely uncertain examples).

## The aggregator — GNMax and Confident-GNMax

The naive aggregator: take the plurality vote and return the winning class. That would leak the exact count of each class, which is linear in the teacher partition and gives away the partition structure. The DP aggregator adds Gaussian noise.

### GNMax (the original aggregator)

For each query `x`:
1. Collect votes `v(x) ∈ ℕ^C`, where `v_c(x)` is the number of teachers voting for class `c`.
2. Add Gaussian noise: `ṽ_c = v_c + N(0, σ²)` independently per class.
3. Return `argmax_c ṽ_c`.

The analysis bounds the privacy cost of this mechanism in terms of σ. Small σ gives high accuracy; large σ gives better privacy.

### Confident-GNMax — the production aggregator

Papernot et al.'s 2018 improvement. Before voting, check whether the plurality is *confidently* large relative to the runner-up. Only pay privacy cost for queries that pass:

1. Noisy threshold check: `max(v(x)) + N(0, σ₁²) > T` for some threshold `T`.
2. If the check passes, run GNMax with noise σ₂ (typically smaller than σ₁) and return the argmax.
3. If the check fails, abstain — do not label this example — and pay only the (small) privacy cost of the threshold check.

Why it helps: on *easy* examples (high teacher agreement), the threshold check passes and the subsequent GNMax uses small noise to produce a correct label at low privacy cost. On *hard* examples (low teacher agreement), the threshold check usually fails; you abstain and pay a tiny privacy cost for the check without the much larger cost of the noisy argmax.

Operationally, Confident-GNMax reduces the total ε spent per labelled example by 3–10× on classification tasks with reasonable teacher agreement. It is the default aggregator in any PATE deployment today.

### Interactive GNMax, LNMax, and other variants

The Papernot-Song 2018 paper also covers LNMax (Laplace noise instead of Gaussian) and Interactive GNMax (where the choice of noise scale depends on prior queries' behaviour). These are useful in specific regimes; for a first deployment, Confident-GNMax is the default.

## The privacy analysis — data-dependent vs data-independent

Here is the piece that makes PATE's claim stronger-than-DP-SGD-for-the-same-budget *when PATE fits*.

### Data-independent analysis

The *worst-case* privacy cost of each GNMax query. Each query is a Gaussian mechanism with sensitivity 1 (one teacher changing its vote changes the vote vector by at most 1 in the `L_∞` norm; by `√2` in `L_2` for the add/remove case). The RDP curve of such a mechanism is known analytically.

Under data-independent analysis, each query costs the same ε regardless of the data. You compose over q queries and convert to (ε, δ). This gives a *correct* but *loose* bound.

### Data-dependent analysis — the smoothed sensitivity argument

The insight from Papernot-Song 2018: when the vote is *consensus* (one class has an overwhelming majority), the mechanism is much less sensitive to any single teacher's vote. If 95 out of 100 teachers vote for class A, swapping one teacher's vote changes the noisy argmax's distribution very little; the actual privacy cost of that query is much lower than the worst-case bound.

The data-dependent Rényi DP analysis computes this per-query, actual cost using the observed vote distribution. The resulting ε is much tighter.

**The subtlety.** The per-query privacy cost *depends on the sensitive data* (the vote counts). Releasing the exact per-query cost would itself leak information. Papernot-Song solve this with the "smoothed sensitivity" machinery (Nissim-Raskhodnikova-Smith): you release an upper bound that is itself a DP estimate of the true cost, paying a small additional budget for the upper-bound release.

The resulting ε is a *data-dependent, DP-released ε*. It is the number the model card quotes.

### The accountant

The Papernot et al. reference implementation under `tensorflow/privacy/research/pate_2018` includes the data-dependent accountant. You feed it the sequence of per-query vote vectors; it returns the (ε, δ) spent, with the smoothed-sensitivity DP overhead already paid. The output is comparable to a DP-SGD ε in every respect except the mechanism.

If you quote PATE's data-dependent ε in the release package without naming the accountant and the smoothed-sensitivity DP overhead, a reviewer will ask. The audit log should include:

- The sequence of per-query vote distributions (shape, not values) sufficient to reconstruct the data-dependent analysis.
- The data-independent upper bound as a cross-check.
- The accountant implementation reference and version.
- The threshold T, the noise scales σ₁ and σ₂, and the abstention rate.

## Student training

The student is trained on the *labelled public data* — the subset of public pool where Confident-GNMax did not abstain, with the noisy aggregator's label. Typical recipe:

- Same architecture family as the teachers, or a larger one (public pool can be much bigger than any single teacher shard).
- Standard non-private training (the DP has already been paid in the labelling).
- **The labelled set may be tiny** at small ε budgets — the aggregator abstains often. If the labelled set is too small to produce a usable student, you either need (a) more public data to label, (b) a looser privacy budget, or (c) switch to DP-SGD on the sensitive data directly.

Semi-supervised learning techniques can rescue the small-labelled-set regime: use the labelled PATE-labelled points plus the unlabelled public pool with a semi-supervised objective (consistency regularisation, FixMatch, VAT). The 2018 Papernot-Song paper includes a GAN-based student variant.

## When PATE wins and when it loses

PATE is a sharp tool. It wins in a specific regime:

- **Many teachers possible, high teacher agreement expected.** Classification tasks with clear labels, abundant data, natural shard structure. The Scalable-PATE paper shows big wins on MNIST, SVHN, and Adult.
- **Access to public unlabelled data.** Task-relevant, distributionally similar, privacy-clean.
- **Value in reporting a data-dependent ε.** A number that beats what DP-SGD would give at the same utility.

PATE loses in these regimes:

- **Regression, dense prediction, free-form generation.** Not classification-shaped; no clean vote to aggregate.
- **No public data.** You cannot query anything without labelled queries; back to DP-SGD.
- **High teacher disagreement.** Abstention rate → 100%, labelled set → 0, student unusable. This often means the task is too hard for a non-private teacher on a small shard.
- **The sensitive data does not partition naturally.** You are forced into a random partition, which erodes the clean story about what the privacy unit protects.

A diagnostic you can run before committing to PATE: train 10 teachers on 10 random shards of the sensitive data; measure the teacher-agreement rate on a held-out validation set. If agreement is above ~75%, PATE will likely work; below, DP-SGD is a safer bet.

## The engineering failure modes

A ranked list, from most common to most insidious.

1. **Partition leakage.** The sensitive data accidentally appears in the public pool (e.g., the public pool came from a scrape that contained some of the sensitive records). Breaks the PATE guarantee entirely. The attestation: a provable disjointness check between teacher shards and the public pool.
2. **Teacher data overlap.** A record in teacher i's shard also appears in teacher j's shard. Breaks the Papernot-Song analysis (which assumes disjoint shards). Fix: enforce disjointness at partition time with a hash-based check.
3. **Per-teacher accountant counted instead of aggregator accountant.** Teachers are trained *non-privately*; there is no DP-SGD accountant to invoke on them. The privacy cost is entirely in the aggregator. Quoting a per-teacher ε is a category error.
4. **Data-dependent ε released without smoothed-sensitivity overhead.** A tighter number that is itself a privacy violation to release in full precision. The accountant must apply the overhead.
5. **Abstention policy changed mid-run.** The aggregator's threshold T and noise σ₁, σ₂ must be fixed before any query is issued. Changing them based on observed vote distributions is adaptive, and the accountant's assumption is violated. Pin them in a config and lock the config for the run.
6. **Student overfits on the small labelled set.** A tiny labelled set will cause the student to memorise the labels. Standard regularisation applies; also, the labelled points are the *noisy* labels, so the student is learning a noisy function. Document the student's test accuracy on a held-out set that was *not* PATE-labelled.
7. **Teachers trained on biased shards.** If the partition is by hospital and one hospital has very different demographics, that teacher's votes systematically differ from others; the aggregator's "agreement" is an artefact of population homogeneity, not of true consensus. The aggregator is still DP; the resulting student is just biased. mod-103's data-minimisation and mod-104's fairness checks are the diagnostic path.

## PATE and DP-SGD side by side

A rough mental model for the decision:

| | PATE | DP-SGD |
|---|---|---|
| Needs public data | yes | no |
| Needs natural partition | strongly preferred | no |
| Task shape | classification | any |
| Model architecture flexibility | high (teacher ≠ student) | constrained (BN → GN; per-example gradients required) |
| Privacy analysis tightness | high in data-dependent regime | high with PRV accountant |
| Audit story complexity | moderate (smoothed sensitivity) | high (clipping, sampling, accountant) |
| Deployment prevalence | lower | higher |
| Natural for federated | no | with FL-DP variants (mod-107) |

If both fit, DP-SGD is usually the simpler audit story. PATE is the move when DP-SGD does not fit the data structure.

## Summary — what you can do after this chapter

- Decide when PATE is a fit (classification, natural partition, public pool) and when it is not.
- Design the teacher ensemble (count, partition strategy, training recipe) and defend each choice.
- Implement Confident-GNMax with threshold T, noise scales σ₁, σ₂, and abstention.
- Distinguish data-independent and data-dependent privacy analyses; explain why the data-dependent ε requires smoothed-sensitivity DP overhead.
- Train the student on the labelled public pool and report an honest utility number.
- Name the seven engineering failure modes (partition leakage, teacher overlap, wrong accountant, missing smoothed-sensitivity overhead, mid-run abstention changes, student overfit, biased shards) and the attestations for each.
- Pick PATE vs DP-SGD for a given task and defend the choice to a reviewer.

Chapter 07 moves from PATE as a classifier to PATE-GAN and the full DP synthetic-data stack.
