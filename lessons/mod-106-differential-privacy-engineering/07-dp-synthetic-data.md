# DP Synthetic Data — DP-GAN, PATE-GAN, DP-CTGAN, DP-WGAN, Private-PGM, and SmartNoise Synth

DP-SGD trains a private model. PATE trains a private classifier. DP synthetic data goes one level further out: it releases a synthetic dataset with a formal (ε, δ) guarantee, and anyone downstream — internal ML teams, external researchers, external contractors — can build on that synthetic set without touching the sensitive originals. The tool is attractive because it is *composable*: once you release the DP synthetic set, downstream training does not need to be DP; the release carries the privacy cost end-to-end.

The catch is utility. DP synth is historically where DP's utility story is hardest. This chapter covers the generator families that currently matter, the SmartNoise Synth stack that is the production face of the research literature, and the utility-vs-privacy audit methodology you have to ship with any synthetic release.

Primary references:

- Xie, Lin, Wang, Zhou — *Differentially Private Generative Adversarial Network*. 2018. [arxiv.org/abs/1802.06739](https://arxiv.org/abs/1802.06739) — DP-GAN.
- Jordon, Yoon, van der Schaar — *PATE-GAN: Generating Synthetic Data with Differential Privacy Guarantees*. ICLR 2019. [openreview.net/forum?id=S1zk9iRqF7](https://openreview.net/forum?id=S1zk9iRqF7) — PATE-GAN.
- Xu, Skoularidou, Cuesta-Infante, Veeramachaneni — *Modeling Tabular Data using Conditional GAN*. NeurIPS 2019. [arxiv.org/abs/1907.00503](https://arxiv.org/abs/1907.00503) — CTGAN; DP-CTGAN is the DP-SGD-wrapped variant.
- Alzantot, Chakraborty, Srivastava — *SenseGen: A Deep Learning Architecture for Synthetic Sensor Data Generation*. 2017. [arxiv.org/abs/1701.08886](https://arxiv.org/abs/1701.08886) — context for DP-sensor generation.
- McKenna, Sheldon, Miklau — *Graphical-model based estimation and inference for differential privacy*. ICML 2019. [arxiv.org/abs/1901.09136](https://arxiv.org/abs/1901.09136) — Private-PGM.
- OpenDP SmartNoise Synth — [docs.smartnoise.org/synth/](https://docs.smartnoise.org/synth/) and [github.com/opendp/smartnoise-sdk](https://github.com/opendp/smartnoise-sdk).
- NIST Synthetic Data Showcase — [github.com/microsoft/synthetic-data-showcase](https://github.com/microsoft/synthetic-data-showcase) and NIST PSCR activity.
- Stadler, Oprisanu, Troncoso — *Synthetic Data — Anonymisation Groundhog Day*. USENIX Security 2022. [arxiv.org/abs/2011.07018](https://arxiv.org/abs/2011.07018) — the cautionary paper every synth engineer should have read.

## Why DP synthetic data at all

DP synth sits in a sweet spot *when it works*:

- **Composable downstream privacy.** Once released, no further DP accounting: the synth set is (ε, δ)-DP by construction. Downstream ML teams can run normal training.
- **Flexible utility surface.** The synthetic set can be queried in ways the DP-aggregated statistic cannot — new joins, new features, new ML pipelines on the same data.
- **Reduced data-governance friction.** The synth set can often be shared with contractors or external researchers under lighter contractual terms than the sensitive originals (depending on jurisdiction and the regulator's current reading of DP).

What it does not solve:

- **Utility on the long tail.** The DP noise destroys rare patterns that are precisely what many analyses care about.
- **Protection against targeted attacks on the synth set.** A DP-synth record can still *look* like a real record (just by chance); the (ε, δ) bound covers membership-inference risk, not accidental resemblance. Stadler et al.'s "Groundhog Day" paper is required reading on this exact trap.

Chapter 09's threat-model note has to state both.

## The generator families

### DP-GAN (and variants)

The idea: train a GAN with DP-SGD applied to the *discriminator's* parameters. The generator is updated with gradients from the discriminator; because the discriminator is DP-trained, the generator (post-processing) is DP by the DP post-processing theorem — no DP cost attaches to the generator update.

Implementation details:

- Apply DP-SGD to the discriminator: clip per-example gradients, add Gaussian noise, use the appropriate accountant.
- Keep the generator updates out of the DP accounting (they are post-processing).
- The Frechet / Wasserstein / Jensen-Shannon flavours of GAN each have a DP variant; DP-WGAN (Wasserstein GAN with DP) is the most common in practice because WGAN's loss is more stable under DP noise.

Utility regime:

- Works reasonably on simple distributions (MNIST-class, low-dimensional tabular with categorical + continuous).
- Struggles on high-dimensional structured data (sequences, graphs, long text) — the DP noise on the discriminator makes training unstable at meaningful ε.
- Mode collapse is a known failure; DP exacerbates it because it smooths the discriminator's signal.

### PATE-GAN

Replace the DP-SGD-trained discriminator with a PATE ensemble of discriminators. One discriminator per shard of the sensitive data; the generator trains against the noisy-aggregated discriminator output for each synthesised example.

Advantages over DP-GAN:

- Reuses the data-dependent PATE analysis from chapter 06 — tighter ε when the ensemble agrees.
- Shards the data naturally (useful when the data already has partitioned shards, like per-hospital or per-institution tabular datasets).

Limitations:

- Needs enough data per shard to train each discriminator well.
- The aggregator is binary (real vs fake); the vote space is `{real, fake}`, not a `C`-way class vote.

### DP-CTGAN

CTGAN (Xu et al. 2019) is the state-of-the-art non-DP tabular generator for mixed categorical + continuous columns with class-conditional sampling. DP-CTGAN is CTGAN with DP-SGD on the discriminator.

Why it matters: CTGAN handles the tabular-data pathologies (multimodal continuous columns, class imbalance, high-cardinality categoricals) that vanilla GANs stumble on. The DP overlay inherits those capabilities while paying the DP noise cost on the discriminator.

In practice DP-CTGAN is the default first attempt for a mid-dimensional tabular DP-synth release.

### DP-WGAN

Wasserstein GAN with DP-SGD on the discriminator ("critic" in WGAN terminology). More stable training than vanilla DP-GAN; common in the literature as a baseline.

### Private-PGM — the non-GAN alternative

McKenna et al.'s approach, used by the US Census and by several winning teams in the NIST Differential Privacy Synthetic Data Challenge. The idea:

1. Define a set of *marginals* to measure (1-way, 2-way, 3-way joint distributions over selected columns).
2. Measure each marginal with a DP mechanism (Gaussian or Laplace).
3. Fit a graphical model that is consistent with the noisy measurements.
4. Sample synthetic records from the graphical model.

Private-PGM's two advantages over DP-GAN:

- The DP noise is on the *statistics you care about* — marginals you explicitly chose to measure. Downstream analyses on those marginals have predictable utility.
- The DP budget composes cleanly across the marginals via Rényi or PRV accounting. No gradient descent instability.

Its limitation:

- You have to choose which marginals matter in advance. Analyses that need a marginal you did not measure will be unreliable on the synth set.

For structured tabular data with a well-understood analytical surface, Private-PGM typically beats DP-GAN at the same ε.

### Marginal-based / selection-based generators

Several winning entries in NIST's 2018–2020 Synthetic Data Challenges used variants of: measure selected marginals, use a MRF / Bayesian-network to generate synthesised records consistent with the noisy marginals, iterate. MST (McKenna, Miklau, Sheldon 2021) and AIM (McKenna et al. 2022) are the production-grade versions. SmartNoise Synth wraps both.

### Where DP-LMs fit

Fine-tuning a language model with DP-SGD and then sampling is an alternative to GAN-based text synth. It inherits DP-SGD's engineering story (chapters 03–04) rather than needing a new one. The 2023–2024 literature (e.g., "DP-ChatGPT"-style papers, Yue et al., Mattern et al.) explores this; for medical notes / clinical text / short-form text this is often the cleanest path.

## The SmartNoise Synth stack

SmartNoise Synth (from OpenDP) is the production-grade library wrapping most of the above. It exposes:

- `DPGAN`, `PATEGAN`, `DPCTGAN`, `PATECTGAN` — the GAN family.
- `MST`, `AIM`, `PACSynth` — the graphical-model / marginal-based family.
- A common `Synthesizer` interface: `.fit(data, preprocessor_eps=..., epsilon=...)`, `.sample(n)`.
- A hooks surface for custom pre-processing (column binning, one-hot encoding, outlier clipping) with DP overhead accounted for.

A minimal use looks like:

```python
from snsynth import Synthesizer

synth = Synthesizer.create("aim", epsilon=1.0, verbose=True)
synth.fit(df, preprocessor_eps=0.05)     # some ε is spent on DP-preprocessing
synthetic_df = synth.sample(n=len(df))
```

Three disciplines around SmartNoise:

- **Account for preprocessing ε.** Any column-bounds computation, outlier clipping, or histogram preparation that depends on the sensitive data is a DP mechanism of its own. SmartNoise exposes `preprocessor_eps`; it is added to the main ε in the final claim. The release package (chapter 09) cites both.
- **Pin the SmartNoise version in the release.** AIM and MST have been tuned across versions; the ε at a given `epsilon` kwarg is an artefact of the library version's accountant and the preprocessor's budget allocation.
- **Validate the synthesised set's utility.** This is the big one — next section.

## The utility audit — what you actually have to measure

DP-synth without a utility audit is a release of random noise with a privacy guarantee. The audit battery, broken into three tiers:

### Tier 1 — distributional similarity

The first-order check that the synthesised marginals match the original.

- **Per-column histograms.** For each column, plot the original vs synthesised histogram. Compute the Kolmogorov-Smirnov statistic (continuous), the total variation distance (categorical), or the Jensen-Shannon divergence (either).
- **Pairwise correlations.** Spearman / Pearson correlation matrix on the original vs synthesised. Report the Frobenius norm of the difference.
- **Three-way marginals.** For a small set of high-signal triples (as identified by the downstream analyses), compute the three-way joint distribution and compare.

Target: distributional similarity close enough that a downstream analyst reading the synthesised set reaches qualitatively the same conclusions as on the original. There is no absolute threshold; this is domain-specific.

### Tier 2 — ML utility (train-synth, test-real)

The second-order check that a model trained on the synth set generalises to the real set.

- Train classifier / regressor **A** on the real set; evaluate on a held-out real test set → metric **M_real**.
- Train classifier / regressor **B** (same architecture, same hyperparameters) on the synth set; evaluate on the same held-out real test set → metric **M_synth**.
- Report the ratio M_synth / M_real. For a usable DP-synth release, 0.9 is excellent, 0.8 is tolerable, 0.6 is probably shipping noise.

Run this for multiple downstream tasks (at least three) to get a sense of the synth set's utility surface. A set that does well on linear-regression-shaped tasks and badly on tree-boosting-shaped tasks is giving you a signal; name it in the release package.

### Tier 3 — privacy stress test

The DP claim says the attacker has bounded advantage. Verify empirically.

- **Membership inference on the synth set.** Run a Shokri-style MIA (mod-105) against the generator / the synthesised set with partial shadow knowledge. Report the AUC and TPR@FPR, same as any model release.
- **Record-linkage attack.** For each synthesised record, find the nearest real record by Hamming / Euclidean distance. Report the distribution of distances — if many synthesised records are exact duplicates or near-duplicates of real records, you have a bug (even if the ε math is correct).
- **Attribute-inference sweep.** For each column, train a classifier on the other columns of the synth set to predict the target column; compare its accuracy to baselines.

Stadler et al.'s "Groundhog Day" paper is where all of this comes from in the literature. The paper's message: *DP-synth with a loose ε can still leak specific real records*. The empirical membership inference is the corroborator for the claimed ε, exactly as in mod-105.

## The end-to-end release schema

The audit log for a DP-synth release looks like:

```jsonc
{
  "release_id": "synth-hospital-tabular-2026-q4",
  "mechanism": "aim",
  "library": "snsynth==1.0.2 (OpenDP SmartNoise Synth)",
  "epsilon_main": 1.0,
  "epsilon_preprocessor": 0.05,
  "epsilon_total": 1.05,
  "delta": 1e-6,
  "accountant": "PRV (via snsynth internal)",
  "neighbouring_relation": "add_or_remove_one_record",
  "granularity": "record_level",
  "n_real": 123456,
  "n_synth": 123456,
  "utility": {
    "distributional": {"ks_mean": 0.03, "tv_mean": 0.02, "corr_frobenius_norm": 0.08},
    "ml_tasks": [
      {"task": "readmission", "m_real": 0.812, "m_synth": 0.774, "ratio": 0.953},
      {"task": "length_of_stay", "m_real": 0.643, "m_synth": 0.592, "ratio": 0.921}
    ]
  },
  "privacy_stress": {
    "mia_auc": 0.508,
    "mia_tpr_at_0.001_fpr": 0.011,
    "nearest_neighbour_mean_distance_normalised": 0.37
  },
  "known_limitations": [
    "5-way joint distributions beyond those listed in marginal_set are high-variance.",
    "Categorical columns with >128 unique values are coarsened to top-128 + 'other'."
  ]
}
```

Chapter 09 is where this is folded into the full release package.

## The Groundhog Day warning — what to say out loud

Stadler et al. 2022 showed, empirically, that popular non-DP synthetic-data methods frequently fail to prevent membership inference, and that even DP-synth at loose ε can pass superficial tests while leaking specific records. Two engineering takeaways:

- **Never report DP-synth without the empirical privacy stress test.** The (ε, δ) claim is a worst-case bound; the empirical attack is the cross-check. If they disagree (empirical worse than the bound), you have a mechanism bug. If they agree (both weak), you need a smaller ε.
- **Never advertise DP-synth as "anonymised".** DP has a specific meaning; "anonymised" has legal meaning in GDPR Recital 26 and California CPRA that is strictly stronger than DP in many readings. The release package should name the DP claim as "(ε, δ)-differentially private" and leave the "anonymised" determination to legal counsel.

## When *not* to use DP synth

Four antipatterns to flag at the start of a project.

- **"We want to share the data with the public; make it synth."** The utility bar for public sharing is often "can replicate the original analyses to within some tolerance", which requires a tight ε and a method that preserves the specific joint distributions the public will use. If you do not know the downstream analyses in advance, you will almost certainly ship an ε too loose to be meaningful or utility too weak to be useful. Push back.
- **"Synthetic data for model training when we could just DP-train the model."** If you have one downstream model, DP-train it with DP-SGD; the end-to-end ε budget is usually tighter than DP-synth → non-private train.
- **"Synthetic data as a shield against regulator questions."** DP-synth does not sidestep DPIA requirements; it adds a new release artefact with its own (ε, δ) to defend.
- **"Synthetic data for high-dimensional unstructured data (full-length medical notes, images, video)."** DP-GAN on these fails to achieve reasonable utility at meaningful ε. DP-fine-tuned LMs (chapter 03) for text or DP-trained vision models with downstream task-specific outputs are usually better paths.

## Summary — what you can do after this chapter

- Decide when DP synth is the right release shape (composable downstream; utility on well-chosen analyses tolerable; data is tabular or low-dim) and when it is not.
- Pick a generator: DP-GAN / PATE-GAN / DP-CTGAN / DP-WGAN for GAN-shaped problems, Private-PGM / MST / AIM for marginal-shaped problems, DP-fine-tuned LMs for text.
- Use SmartNoise Synth, account for preprocessor ε, pin the version.
- Run the three-tier utility audit: distributional, ML task, privacy stress test.
- Name the Stadler et al. result and defend the DP claim against its implication — empirical MIA on the synth set is not optional.
- Write the DP-synth release schema (`release_id`, mechanism, ε breakdown, utility, privacy stress, limitations).
- Flag the four antipatterns and push back when a project asks for DP synth for the wrong reason.

Chapter 08 is the other alternative to DP training — when you only need to release aggregates, DP analytics via PipelineDP or SmartNoise Analytics.
