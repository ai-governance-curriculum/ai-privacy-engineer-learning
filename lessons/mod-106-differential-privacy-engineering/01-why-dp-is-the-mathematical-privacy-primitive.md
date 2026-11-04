# Why Differential Privacy Is the Mathematical Privacy Primitive — and What the Variants Actually Buy You

mod-104 taught the *preventive* control (carve PII out of the corpus) and mod-105 taught the *measurement* control (attack the model and report the residual-risk number). This module teaches the *mathematical* control: Differential Privacy. DP is the one privacy guarantee a regulator, an auditor, or an acquirer's due-diligence lead can cross-check against a formal definition instead of against your word. The attack-measurement harness of mod-105 corroborates it empirically; DP is what lets you quote a number that is true *in the worst case*, not just on the examples you happened to test.

This chapter is the conceptual foundation for the rest of the module. Before any code — before Opacus, before PATE, before the SmartNoise synth stack, before PipelineDP — you need the vocabulary the rest of the chapters assume: pure (ε)-DP, approximate (ε, δ)-DP, Rényi DP, zero-concentrated DP, Gaussian DP, and the mapping from those definitions to the accountant each production library actually ships with.

Primary references — read them in this order:

- Dwork & Roth — *The Algorithmic Foundations of Differential Privacy*. Foundations and Trends in Theoretical Computer Science, 2014. [www.cis.upenn.edu/~aaroth/Papers/privacybook.pdf](https://www.cis.upenn.edu/~aaroth/Papers/privacybook.pdf) — the standing monograph. Chapters 1–3 are the minimum; chapter 3 is the Gaussian mechanism and advanced composition.
- Mironov — *Rényi Differential Privacy*. CSF 2017. [arxiv.org/abs/1702.07476](https://arxiv.org/abs/1702.07476) — the definition every production accountant is now derived from.
- Bun & Steinke — *Concentrated Differential Privacy: Simplifications, Extensions, and Lower Bounds*. TCC 2016. [arxiv.org/abs/1605.02065](https://arxiv.org/abs/1605.02065) — zCDP, the "clean" alternative to RDP.
- Dong, Roth, Su — *Gaussian Differential Privacy*. JRSS-B 2022. [arxiv.org/abs/1905.02383](https://arxiv.org/abs/1905.02383) — the f-DP / GDP framework used by newer accountants (PRV-accountant, Google's current DP-SGD analysis).
- NIST SP 800-226 — *Guidelines for Evaluating Differential Privacy Guarantees*. Dec 2023. [nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-226.pdf](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-226.pdf) — the standards reference you will cite in the release package (chapter 09).

## The DP promise in one paragraph

A randomised mechanism `M` satisfies (ε, δ)-differential privacy if, for any two datasets `D` and `D'` that differ in a single record, and for any set of outputs `S`:

```
Pr[ M(D)  ∈ S ]  ≤  exp(ε) · Pr[ M(D') ∈ S ]  +  δ
```

The promise, in plain English: swap any one person's record in or out of the input, and the distribution over outputs barely changes — bounded by a multiplicative factor `exp(ε)` and an additive slack `δ`. Any downstream attacker who sees `M(D)` has only a small advantage in deciding whether the specific record was in `D`. That is the *mathematical* membership-inference bound; mod-105's LiRA is the *empirical* corroboration.

Two parameters, two intuitions:

- **ε (epsilon).** The *privacy loss*. Smaller is more private. ε = 0 would mean the mechanism is literally independent of the data (perfect privacy, zero utility). ε → ∞ means no privacy.
- **δ (delta).** The *failure probability* — the mass of "bad event" outputs where the ε bound can be violated. By convention, δ ≪ 1 / |dataset|, so that even a union bound over every record keeps the probability of leaking any specific person well under one.

The single most common novice mistake is treating ε as a dimensionless knob with the same meaning across mechanisms, datasets, and compositions. It is not. ε is defined per-mechanism, per-neighbouring-dataset relation (record-level vs. user-level vs. example-level), and composes non-trivially across releases. Every reportable ε is a *(ε, δ, unit, granularity, composition)* tuple — chapter 05's whole job is teaching you to defend that tuple under audit.

## Why "pure" (ε)-DP is not the production definition

Setting δ = 0 recovers *pure* ε-DP (Dwork's 2006 definition). It is the cleanest version — no additive slack — and it is what you see in the original Laplace-mechanism literature. But production ML uses *approximate* (ε, δ)-DP for two reasons.

- **The Gaussian mechanism does not satisfy pure DP.** The Gaussian noise distribution has unbounded tails; a vanishing fraction of outputs sits in regions where the pure ε ratio is violated. Approximate DP absorbs that tail into δ.
- **Composition behaves better.** The *advanced composition* theorem (Dwork-Rothblum-Vadhan 2010) shows that `k` applications of (ε, δ)-DP compose to roughly `(ε√k, kδ)` rather than the loose `(kε, kδ)` basic-composition bound. Over the thousands of SGD steps a DP-SGD run uses, that √k factor is the difference between a usable ε and an absurd one.

So: your production mechanism is (ε, δ)-DP. Pure ε-DP is a pedagogical reference point and a theorist's convenience. mod-106 code almost never ships δ = 0.

## The accountant problem — why we need Rényi DP, zCDP, and GDP

Advanced composition is a bound for *worst-case* neighbouring datasets. It is loose. For DP-SGD — Gaussian noise on sub-sampled minibatches, applied for many steps — advanced composition gives an ε many times larger than what is actually true. That bound is so loose that for T ≈ 10⁵ steps it overspends your privacy budget by an order of magnitude or more. If you train DP-SGD under advanced composition and quote the ε, you are reporting a wildly conservative number — and you are still probably wasting enough budget that the utility is worse than it needs to be.

The remedy is to compose in a tighter representation and convert back to (ε, δ)-DP only at the end. Three such representations are in production use. Each production library picks one.

### Rényi DP (RDP)

Mironov's 2017 definition. A mechanism `M` is (α, γ)-Rényi-DP if for neighbouring `D, D'`:

```
D_α( M(D) || M(D') )  ≤  γ
```

where `D_α` is the order-α Rényi divergence between the two output distributions. RDP has two properties that make it the DP-SGD workhorse.

- **Linear composition in RDP.** Running two mechanisms with Rényi-DP curves `γ_1(α)` and `γ_2(α)` composes to `γ_1(α) + γ_2(α)` at the same `α`. No √k magic required — the composition is just addition.
- **Tight analysis of the Sampled Gaussian Mechanism (SGM).** SGM — the operation DP-SGD performs once per step — has a known analytic RDP curve (Mironov-Talwar-Zhang 2019, "Rényi Differential Privacy of the Sampled Gaussian Mechanism"). You compose T copies of it (sum the curves) then convert the resulting (α, γ)-RDP to (ε, δ)-DP by minimising over α of `γ(α) + log(1/δ) / (α-1)`.

The *RDP accountant* is the implementation of that pipeline. TensorFlow Privacy's original accountant, Opacus's default accountant, and the Google DP library's early accountant are all RDP accountants. Chapter 02 ships the mechanics; chapters 03–04 use them.

**Loss and gain.** RDP is tight for the Gaussian mechanism and for SGM. It is less tight for other mechanisms (e.g. the exponential mechanism) and it does not compose well with mechanisms that have non-Rényi analysis. The (α, γ) curve carries more structure than a single (ε, δ) pair — you can lose information converting back to (ε, δ).

### Zero-concentrated DP (zCDP)

Bun & Steinke's 2016 definition. A mechanism is `ρ`-zCDP if its Rényi divergence curve is exactly `γ(α) = α · ρ` for all α > 1. That is, zCDP is the special case of RDP whose curve is a straight line through the origin with slope ρ.

**Why that matters.** The Gaussian mechanism with noise scale σ and sensitivity Δ is exactly `(Δ² / 2σ²)`-zCDP — one scalar, not a curve. Composition is addition of ρ values. Conversion to (ε, δ)-DP uses `ε = ρ + 2√(ρ · log(1/δ))`. The US Census Bureau's 2020 Disclosure Avoidance System (chapter 05) uses zCDP as its accounting basis because a single ρ budget is simple to allocate across a hierarchy of queries and the conversion is clean.

**Loss and gain.** zCDP is a clean special case of RDP; it is tight exactly when the mechanism's RDP curve is linear in α (the Gaussian mechanism; the SGM to a close approximation at moderate α). For general compositions it can be looser than RDP.

### Gaussian DP (GDP) and f-DP

Dong-Roth-Su 2019. The newest framework, the one Google's current DP-SGD analysis is built on. The idea: track the full *trade-off function* — the achievable (FPR, FNR) curve of the optimal hypothesis test between `M(D)` and `M(D')` — not a scalar ε or a Rényi curve. A mechanism is `μ`-Gaussian DP if its trade-off function is at least as hard as discriminating two unit-variance Gaussians separated by `μ`.

**Why that matters.** The subsampled Gaussian mechanism converges (as T→∞, under mild assumptions) to a limiting GDP mechanism — the *central limit theorem for DP-SGD*. The Google-DP PRV accountant (Privacy Random Variable) and the Microsoft prv_accountant library use this to produce the tightest known (ε, δ) numbers for DP-SGD at moderate sampling rates and step counts. Opacus's newer accountant (`prv`) is a PRV implementation.

**Loss and gain.** GDP/f-DP is the tightest framework we currently have for DP-SGD specifically. It is also the most involved to implement and the hardest to reason about outside the subsampled-Gaussian regime. For DP-SGD, use it; for exotic mechanisms, stay in RDP.

## The hierarchy — which to use when

The practical hierarchy for the DP-SGD setting is:

1. **PRV / f-DP / GDP accountant** — tightest ε for a fixed noise multiplier and step count. Use it when the budget is tight and the utility headroom matters.
2. **RDP accountant** — slightly looser; well understood; the default in Opacus and TF-Privacy for years. Pick it when you want an accountant you can explain in a review.
3. **zCDP accountant** — clean, scalar, auditable. Picked by the Census (hierarchical queries) and by analytics libraries where simplicity wins.
4. **Advanced composition on (ε, δ) alone** — only as a sanity-check upper bound. Never as your reported number for DP-SGD. It will be loose enough to embarrass you.

Chapter 02 lays out the library-to-accountant mapping so you can defend the choice ("we trained with Opacus, which uses the PRV accountant by default, which is tighter than the RDP accountant we previously used by roughly…"). Chapter 05 is where the accountant choice shows up in the regulator conversation.

## The neighbouring-dataset relation — the quiet detail that changes ε

The definition of (ε, δ)-DP is relative to a *neighbouring relation* on datasets. Two datasets are neighbours if they differ in… what? Three standard choices, each producing a different ε for the same mechanism:

- **Add / remove one record (unbounded DP).** `D'` is `D` with one record added or removed. This is the standard DP-SGD definition.
- **Change one record (bounded DP).** `D'` is `D` with one record replaced by another. Factor-of-two differences in ε can show up between the two conventions.
- **User-level DP.** `D'` differs from `D` in all records belonging to a single user. For a user who contributed `k` records, this is roughly `k` times the record-level ε. Mandatory for federated learning and for any product where a user's identity is the privacy unit, not a single transaction.

The release package (chapter 09) must state *which* neighbouring relation your ε is quoted for. "ε = 4" with no granularity is a reportable number that is not actually interpretable. Google's open-source accountants require you to specify the granularity; so does the NIST SP 800-226 reporting checklist.

A worked example of the user-level amplification: if your training set contains on average 50 records per user and you accounted at record-level ε = 2, the user-level ε under the group-privacy transform is roughly 50 · 2 = 100 — not a meaningful bound. You would need to either deduplicate to one record per user, train at a much smaller record-level ε, or restructure the mechanism to compose at user level from the start. Chapter 05's Apple numbers are quoted *user-level* for exactly this reason.

## What DP does not protect

DP is a strong guarantee but a *specific* one. The NIST SP 800-226 evaluation guidelines (chapter 09) have an explicit threat-model section for what the DP claim does and does not protect. The short list:

- **DP bounds information about a single record's presence.** It does *not* bound information about aggregate statistical properties of the dataset — distribution shift, class balance, correlations. Those were the dataset's statistics before the mechanism ran; DP does not hide them.
- **DP is only as fine-grained as its neighbouring relation.** Record-level DP does nothing to protect a user with many records; user-level DP does nothing to protect a group that spans users.
- **DP is on the training-time pipeline; it is not a defence against prompt-injection, retrieval-leak, or model-weight theft.** mod-108 (confidential computing) and mod-109 (RAG scoping) cover those.
- **DP noise is calibrated to the *stated* sensitivity.** If the code's actual per-example contribution exceeds the clipping norm in some edge case, the DP claim is violated and the ε is wrong. Per-example gradient clipping bugs (microbatching with incorrect vmap, the frozen-BN footgun) are the most common source of claimed-but-untrue DP guarantees. mod-106 exercise 02 has you look for them.
- **DP is about the *output of the mechanism*, not about everything the ML system logs.** Training logs that record per-example gradients outside the DP mechanism break the claim. So does a model checkpoint that is accessible outside the DP pipeline before training completes.

The threat-model note in your release package (chapter 09) is the artefact where you say this out loud. A DP claim without a threat-model note is a claim that will fall apart in the first audit.

## The privacy-utility trade-off in one picture

DP's cost is injected noise. Noise degrades utility. For a given task, the Pareto frontier is:

```
utility ↑
   |         • (no DP, highest utility, no privacy)
   |       •       ← the Pareto frontier
   |     •
   |    • (ε ≈ 8, modest utility hit, "meaningful" privacy)
   |   •
   |  •  (ε ≈ 1, larger utility hit, "strong" privacy)
   | •
   |•    (ε ≈ 0.1, large utility hit, "near-perfect" privacy)
   +--------------------------------→  more privacy (smaller ε)
```

The art of DP engineering is moving the frontier up and to the left — same ε, more utility. The levers you have:

- **Tighter accountant** (PRV > RDP > zCDP > advanced-composition) — free utility, just a code change.
- **Larger batches / higher sampling rates** — tighter privacy amplification by subsampling.
- **Smaller model** — gradients are lower-dimensional, so a given clipping norm C bounds a smaller fraction of signal.
- **Better architecture** (BN → GroupNorm, ReLU → GeLU, careful initialisation) — DP-SGD is sensitive to architecture choices in ways non-private training is not.
- **Pre-training on public data** then fine-tuning privately — the de facto SOTA for DP large models. The Li-Tramèr-Liang-Hashimoto "Large Language Models Can Be Strong Differentially Private Learners" paper lands this.

Chapters 03 (Opacus) and 04 (TF-Privacy) make each of these concrete.

## The module's working vocabulary — a one-page glossary

Adopt these terms as the module's lingua franca. Every later chapter uses them without redefinition.

- **Privacy budget** — the (ε, δ) you are willing to spend on a release. Spent down across mechanisms, queries, or SGD steps via a composition theorem.
- **Accountant** — the piece of code that tracks spent budget. RDP accountant, PRV accountant, zCDP accountant are three common choices. Chapter 02.
- **Sensitivity (Δ)** — the maximum change in the mechanism's input under a single-record swap. For DP-SGD, Δ is the per-example gradient clipping norm `C`.
- **Noise multiplier (σ)** — the ratio of added Gaussian noise standard deviation to sensitivity. The knob you turn to trade ε for utility in DP-SGD.
- **Sampling rate (q)** — the Poisson sampling probability per example per step. Amplifies privacy — the smaller q, the tighter the ε at fixed σ and T.
- **Composition** — combining multiple DP mechanisms into one and computing the resulting ε. Basic, advanced, RDP, PRV compositions in increasing tightness.
- **Group privacy** — the amplification of ε when the neighbouring unit is a group of k records instead of 1 record. Roughly linear in k for small ε.
- **Record-level / user-level / example-level DP** — the three common granularities. Specify yours.
- **Pure vs. approximate DP** — δ = 0 vs. δ > 0.
- **RDP, zCDP, GDP, f-DP** — four relaxed-definition / tighter-accounting frameworks. Each production library picks one.
- **DP-SGD, PATE, DP-GAN, DP-aggregation** — four flavour of DP mechanism in ML. This module covers all four.

Mis-using a term in the release package is how reviewers find the door. Keep the glossary open.

## Summary — what you can do after this chapter

- State the (ε, δ)-DP definition from memory, in one sentence, and name the two parameters' intuitions.
- Explain why pure ε-DP is not the production definition and why the Gaussian mechanism forces δ > 0.
- Name the four refinements (RDP, zCDP, GDP, f-DP) and which production library uses each as its accountant basis (chapter 02 fills the table).
- Name the three neighbouring-dataset relations and compute the group-privacy amplification for a user who contributes k records.
- List the four things DP does not protect against — the threat-model section of chapter 09 is built from this list.
- Describe the privacy-utility frontier and name four levers that move it.
- Use the module's glossary without hesitation.

Chapter 02 is the accountant-to-library mapping and the mechanics of composition — the operational knowledge you need before touching Opacus or TF-Privacy.
