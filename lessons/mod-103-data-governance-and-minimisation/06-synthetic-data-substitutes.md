# Synthetic-Data Substitutes and Their Utility / Privacy Audit

Where real data cannot be used — scale constraint, consent constraint, jurisdictional constraint, sensitivity constraint — the next lever is to build a *synthetic substitute* from a model of the real data and use the synthetic set in the downstream task. The pattern is mature for tabular data (CTGAN, TabDDPM, Gretel, MOSTLY AI, Tonic), exists but is less mature for text (LLM-driven synthesis), and has domain-specific implementations for health (Synthea) and other verticals.

This chapter is about *when* and *how* to substitute, and the audit that closes the loop: a synthetic set that leaks information about the real training records it was derived from is not a privacy win. The audit has as much weight as the build.

Primary references:

- **CTGAN / TVAE** — Xu et al., *Modeling Tabular Data using Conditional GAN* (NeurIPS 2019). [arxiv.org/abs/1907.00503](https://arxiv.org/abs/1907.00503). Reference implementation: Synthetic Data Vault (SDV) at [sdv.dev](https://sdv.dev/).
- **TabDDPM** — Kotelnikov et al., *TabDDPM: Modelling Tabular Data with Diffusion Models* (ICML 2023). [arxiv.org/abs/2209.15421](https://arxiv.org/abs/2209.15421).
- **Synthea** — synthetic patient-record generator (MITRE / The MITRE Corporation). [synthetichealth.github.io/synthea](https://synthetichealth.github.io/synthea/).
- **NIST SP 800-188** — *Trustworthy Attribute-based Data De-identification*. [doi.org/10.6028/NIST.SP.800-188](https://doi.org/10.6028/NIST.SP.800-188).
- **Jordon, Szpruch, Houssiau et al. — *Synthetic Data — what, why and how?*** (Royal Society / Alan Turing Institute, 2022). A readable survey; citable in a DPIA. [royalsociety.org/synthetic-data](https://royalsociety.org/topics-policy/projects/privacy-enhancing-technologies/).
- **UK ICO — Privacy-enhancing technologies (PETs) guidance** — includes synthetic-data posture. [ico.org.uk](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/data-sharing/privacy-enhancing-technologies/).

## Why this matters

Three use cases where synthetic beats real:

- **Training** — when the real training set is small / regulated / consent-restricted. Synthetic augmentation raises sample count without expanding the lawful basis. Note: the synthetic set is itself regulated when it carries inferred personal data.
- **Evaluation** — when the eval set would otherwise be a sampled production slice. A synthetic eval set is a controlled-distribution test bench that does not leak production customers into the eval pipeline.
- **Demo / sales / documentation / developer-environment** — the demo tenant should never see real customer data. A synthetic tenant with a synthetic schema is the clean substitute.

And three use cases where synthetic *does not* beat real:

- **Compliance eval** — you cannot test your pipeline's handling of real PII on synthetic PII. The redaction / detection / minimisation pipelines must be verified against the real distribution they will encounter.
- **Bias / fairness eval** — a synthetic set faithfully reproduces the real distribution's biases. Testing against synthetic does not reveal what testing against the real population would.
- **Downstream deployment without re-audit** — a model trained on synthetic data and shipped without a real-distribution audit may fail silently in production. The synthetic is a *preprocessing* win; the real-distribution calibration is still required.

Record the choice (real vs. synthetic for which purpose) in the datasheet.

## When synthetic is *regulated* personal data (yes, often)

A common misconception is that synthetic data is "not personal data" and therefore outside GDPR / CCPA scope. The EDPB and the ICO have both been clear that synthetic data is *personal data* when:

- It carries inferred personal data about a real individual (a model trained on your data produces synthetic records that contain memorised patterns about you).
- It can be *linked* back to a real individual via side information.
- It *reproduces* real individuals' records with high fidelity (the generative model has overfit).

The threshold is not cleanly defined in regulation; the engineering position is to *treat synthetic as personal data by default* and relax the posture only after an audit shows no membership / attribute / reconstruction leak. The datasheet records the posture and the audit result.

## The tabular synth toolkit

For tabular (relational / columnar) data, the state of practice is well-developed.

### Non-neural baselines you should know

Before reaching for a neural synthesiser, consider the classical baselines — they are often *good enough* and *easier to audit*:

- **Statistical marginals** — sample each column from its empirical marginal (per-column histogram). Zero correlation preserved. Baseline.
- **Bayesian network** — PrivBayes (Zhang et al.) learns a Bayesian network over columns and samples; supports differential-privacy guarantees.
- **Copulas** — marginal distributions + Gaussian copula for correlation structure. SDV's `GaussianCopulaSynthesizer`.
- **IPF / marginal-based** — iterative proportional fitting on constrained marginals; strong where the real data has few columns and the privacy target is cell-anonymity rather than full-row.

A baseline is often the correct choice for a demo environment or a simple eval. Save the neural approach for cases where columnar correlations and conditional distributions are load-bearing.

### Neural tabular synthesisers

- **CTGAN** (Conditional Tabular GAN) — the de facto baseline since 2019. Handles mixed categorical / continuous columns; mode-specific normalisation.
- **TVAE** — tabular variational autoencoder; the VAE companion in SDV.
- **TabDDPM** — diffusion-model approach; outperforms CTGAN on many benchmarks at the cost of training time and complexity.
- **ARF** (Adversarial Random Forests) — a simple, strong, interpretable alternative when neural is overkill.
- **GReaT / RealTabFormer / TabuLa-style LLM-based synthesis** — sequence-model tabular synthesis; promising, less mature.

Pick one, tune, measure. SDV is the open-source orchestration layer that covers most of the pack.

### Managed services

- **Gretel** — managed synthesis + privacy metrics.
- **MOSTLY AI** — managed; well-known in European regulated-finance work.
- **Tonic** — managed tabular + subsetting + de-identification; aimed at test-data workflows.
- **Hazy, Statice (now part of Anonos), Replica Analytics, Mirry.AI.**

A managed provider is a processor; include it in the Art. 28 inventory (mod-101) and the DPIA.

## Text synthesis for AI

Text synth is a different problem. Two patterns:

- **LLM-paraphrase** — prompt an LLM to rewrite a real record in a way that preserves the task-relevant signal and removes the identifying detail. Low complexity; good for free-text augmentation. Risk: paraphrases may fail to remove identifiers, and any residual personal data is still regulated. Pair with mod-104 post-filter.
- **LLM-from-schema** — prompt an LLM to generate records from a schema and a persona distribution; no real record feeds the prompt. Lower leak risk; higher distribution-mismatch risk.

Where the downstream is a fine-tune or an eval set, text synth should pass the same audit as tabular synth. Beware "generated by an LLM therefore safe" — the LLM has memorised too (mod-105); the synthetic set can carry memorised-and-paraphrased identifiers.

## Domain-specific synthesis

- **Synthea** (health) — rule-based, open-source synthetic patient records with FHIR / CCDA / CSV output and a documented module system. Valuable for test / demo / evaluation of healthcare-adjacent AI; limited for training signal because the underlying generation is rule-based rather than data-driven.
- **GRETEL / simulation-based synthetic driving / robotics corpora** — relevant for other verticals; same audit principles apply.

For regulated verticals (health, finance, children's data), domain-specific synthesisers are often the fastest way to a defensible posture.

## The utility / privacy audit — what the audit must cover

A synthetic dataset is a privacy win *only if the audit survives*. The audit has two halves — utility and privacy — and both are reported.

### Utility metrics

Does the synthetic set behave like the real set for the downstream task?

- **Distributional similarity** — per-column marginals (K-S test for continuous, chi-square for categorical); pairwise correlations; joint distribution comparisons via MMD or Wasserstein distance. SDV reports many of these.
- **Train-on-synthetic-test-on-real (TSTR)** — train the downstream model on synthetic; test on a held-out *real* set. Report AUC / F1 / MAE deltas against train-on-real-test-on-real.
- **Train-on-real-test-on-synthetic (TRTS)** — inverse diagnostic; detects distribution drift.
- **Task-specific metrics** — if the downstream is a fraud classifier, the TSTR F1 at fixed false-positive rate is the metric. Report in the dimensions that matter to the deploying team.

A synthetic set that fails TSTR is not a substitute. Report the gap, pick a different synthesiser or widen the real-data allowance.

### Privacy metrics

Does the synthetic set leak information about the real records that trained the synthesiser?

- **Nearest-neighbour distance ratio (DCR / NNDR)** — for each synthetic record, distance to the nearest real training record vs. distance to the nearest real *holdout* record. If training-side distances are systematically smaller, the synthesiser has memorised. SDV's `DCRBaselineProtection` computes a variant.
- **Membership inference** (mod-105 depth) — a shadow-model MIA trained to discriminate "was this real record in the synthesiser's training set?" The AUC against random guessing is the leak metric.
- **Attribute inference** — given a synthetic record and some real quasi-identifiers, can an adversary predict the sensitive attribute better than from the aggregate? ML Privacy Meter covers.
- **Exact-row match** — a trivial but necessary check: do any synthetic rows reproduce a real training row verbatim? For generative models on small tables this is a real failure mode. For LLM-based text synth, hash shingles + check overlap (mod-105 extraction methodology).
- **Singling-out / linkability** per the Art. 29 WP 05/2014 anonymisation opinion — the regulator-side framing. Can an attacker single out a row with high confidence? Can two rows be linked? Can an attribute be inferred?

A synthetic set that fails any privacy metric is leaky. Record the failure and either remediate or restrict the downstream use.

### Reporting the audit

The audit report is a small artifact — one table per metric, a short paragraph per finding, and a residual-risk claim. Example shape:

```markdown
# Synthetic-data audit — ds.ticket-corpus-synth.v1
Synthesiser: TabDDPM, SDV wrapper, 200 epochs, no DP. Trained on
ds.ticket-corpus.2025-q4 (412_998 real records).

## Utility
- TSTR F1 on category classifier: 0.84 (real baseline: 0.87). Δ = -0.03.
- Continuous marginal K-S test: max D = 0.03 (p>0.1). PASS.

## Privacy
- Exact-row match against training set: 0 / 500_000 synthetic rows. PASS.
- DCR ratio (train vs. holdout NN): 1.02 (expected ~1.0). PASS.
- MIA AUC (shadow-model, 100 shadow models): 0.52 (~random). PASS.
- Attribute inference (sensitive_column): AUC 0.61 vs. baseline 0.58. MARGINAL.

## Residual-risk claim
Attribute-inference AUC exceeds the baseline by 0.03; the attribute is
`low-sensitivity interaction category`; downstream use limited to internal
eval only; re-audit scheduled 2026-04-01.
Reviewed by: DPO 2025-10-30.
```

The audit report is referenced from the datasheet of the synthetic dataset and the DPIA.

## Differential privacy in synthesis

Where the audit above cannot reach "clean", the strongest standard defence is *differentially-private synthesis*: the synthesiser's training produces (ε, δ)-DP guarantees over its real training records. Mechanisms:

- **DP-CTGAN / DP-GAN** — DP-SGD applied to the GAN's discriminator.
- **DP-copula / PrivBayes** — classical DP-under-marginals.
- **DP diffusion** — DP-SGD on the TabDDPM denoiser; actively researched.
- **OpenDP SmartNoise Synth** — open-source DP synthetic-data toolkit.

A DP synthesis gives you a formal budget (ε, δ) and an upper bound on membership-inference AUC. The trade-off is utility: at small ε the synthetic distribution smears. mod-106 is the engineering depth.

For the audit, a DP-synthesised set has a *theoretical* leak bound; you still report the empirical metrics.

## Where synthesis is wrong — honest failure modes

Three failure patterns to design against:

- **Mode collapse** — the synthesiser converges to a few modes and produces lots of near-duplicates. TSTR holds up on average but the long tail is lost. The downstream fine-tune trained on this will miss rare patterns (fraud, edge cases, outliers).
- **Overfit to the training sample** — the synthesiser memorises a specific real record and emits paraphrases. Audit catches this; fix via smaller models, stronger regularisation, or DP training.
- **Distribution drift in production** — the synthetic matches the real from quarter N, but the real has shifted by quarter N+1. The synthetic set becomes an outdated eval that no longer reflects current production.

Record each failure mode you tested for. An audit section that is all PASS without any discussion of *what could have failed* reads as insufficiently rigorous.

## Datasheet entry for a synthetic dataset

The synthetic dataset gets its *own* datasheet (chapter 02). Three extra fields:

- **Synthesiser** — name, version, hyperparameters, training-cost log.
- **Source dataset(s)** — reference to the real dataset(s) whose datasheets this one builds on.
- **Audit report** — reference to the utility / privacy audit artifact.

The datasheet also sets the downstream-use policy for the synthetic set — some sets are "internal eval only"; some are publishable alongside a model release; some are OK to share with a downstream integrator. Each policy is signed.

## Where this lands in the mod-103 stack

Synthetic substitution is a *data-plane* control that lives between chapter 04 minimisation and chapter 05 vault. You use it when:

- Processing minimisation cannot shrink the data below what the task needs, *and*
- Vault + tokenisation does not change what a downstream team can learn from distributions, *and*
- Real data cannot be used for the specific downstream (demo / eval / shared / third-party).

The resulting pipeline has a fourth sink type — the synthetic corpus — which the ML-BOM (chapter 07) records on equal footing with real corpora.

## Summary — what you can do after this chapter

- Decide when synthetic beats real and when it does not (three use cases each).
- Pick a synthesiser (baseline / CTGAN / TabDDPM / managed / Synthea / LLM-paraphrase) appropriate to the data and task.
- Author the utility / privacy audit with named metrics (TSTR, DCR, MIA, attribute inference, exact-row match) and a residual-risk claim.
- Treat synthetic data as regulated personal data by default until the audit demonstrates otherwise.
- Record the synthetic-set datasheet and audit result in the ML-BOM.

The next chapter wraps the module with CycloneDX ML-BOM — the machine-readable artifact that lets an auditor walk from a deployed model back to its training inputs, consent basis, and residual-risk claims.
