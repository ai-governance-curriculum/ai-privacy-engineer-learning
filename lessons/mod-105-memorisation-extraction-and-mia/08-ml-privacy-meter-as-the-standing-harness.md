# ML Privacy Meter — The Standing Harness

Chapters 02–06 built the attacks from first principles. For a one-off research run or an exercise, that is the right level. For a standing release-gate that re-runs on every model candidate, writing the attacks yourself every time is the wrong level — you want a pinned, versioned, documented harness that produces comparable measurements across runs, models, and reviewers. This chapter is the engineer's reference to ML Privacy Meter — the open-source audit tool from Shokri's group at NUS — and how to operate it end-to-end as the standing harness.

Primary references:

- NUS Data Privacy and Trustworthy ML Lab — **ML Privacy Meter**. [github.com/privacytrustlab/ml_privacy_meter](https://github.com/privacytrustlab/ml_privacy_meter)
- Murakonda, Shokri — *ML Privacy Meter: Aiding Regulatory Compliance by Quantifying the Privacy Risks of Machine Learning* (HotPETs 2020). [arxiv.org/abs/2007.09339](https://arxiv.org/abs/2007.09339)
- The tool's documentation ships with the attack configuration references; keep it open while reading this chapter.

## Why a standing harness

Three reasons the privacy engineer does not ship one-off attacks as the production measurement:

- **Comparability across releases.** The release-gate conversation is "did this candidate release regress against the previous one?". Unless the attack harness is pinned to a version, every change in training recipe, library version, or random seed looks like noise in the measurement.
- **Methodology defensibility.** A regulator who sees "we ran the attacks in the Carlini-2022 paper" will push back on provenance. A regulator who sees "we ran ML Privacy Meter v`<tag>` with configuration `audit-config-v4.yaml`, pinned requirements file attached" can replay the measurement themselves. The second answer does not cost a legal cycle.
- **Rate-of-work.** Each release candidate should not require re-implementing LiRA. A standing harness is the engineering equivalent of a CI test: configuration lives in the repo, the measurement runs on demand, the artefact drops into the release bundle.

ML Privacy Meter is the reference open-source harness. It is not the only option — TensorFlow Privacy's attack library, PrivacyRaven, and several newer tools exist — and chapter 09 names the standards mapping each one claims. ML Privacy Meter is the default because it implements the chapter 02 + 03 attacks at production quality, it is maintained by one of the research groups that defined the attacks, and its API is close enough to the paper formulations that you can cross-check the attack configuration against the source literature.

## The architecture

ML Privacy Meter separates three concerns:

- **Target abstraction** — the model you want to audit, wrapped in a project-expected interface (PyTorch / TF / sklearn adapters).
- **Attack configuration** — YAML / Python config specifying which attacks to run (population attack, reference attack, shadow attack, LiRA), their hyper-parameters, and what to report.
- **Audit run** — the orchestration that executes the configured attacks against the target and emits a report directory with plots, CSVs, and a summary JSON.

The separation is important for the standing-harness use case: your project ships a thin target-wrapper around *your* model, and an audit-config YAML. The harness update track (upstream version bumps) is independent of your project's model-training code.

## Mapping the attacks you learned to the harness

The specific attacks the module built and where they land in the tool:

- **Population attack.** The simplest baseline — a Yeom-2018-style loss-threshold attack using a *reference population* instead of shadow models. Low cost, suitable for a quick diagnostic on a new target.
- **Reference attack.** Uses a *reference model* trained on the same architecture + same distribution but without the target training set, to recalibrate the loss-threshold per example. Intermediate strength.
- **Shadow attack.** The Shokri attack from chapter 02 — multiple shadow models, per-class attack classifier.
- **LiRA.** The chapter 03 attack — per-example Gaussian fits, log-likelihood-ratio scoring, log-log ROC reporting.

Configure all four for a production audit. Report the LiRA numbers as the headline; report the population and reference attacks as weaker-baseline comparisons so the reviewer can see the improvement curve across attack sophistications. This is the paper's own convention.

The harness also exposes hooks for attribute-inference attacks and (as of recent releases) LLM-specific extraction eval; these move slower than the classifier-side attacks and their API surface evolves — pin the version and treat the API surface as potentially changing between releases.

## The target-model interface

The tool expects a target-model wrapper that exposes:

- `predict(x)` — probability vector output.
- `loss(x, y)` — per-example loss.
- `(optional) logits(x)` — pre-softmax logits for LiRA.
- Training-set / holdout-set accessors so the harness can query on members and non-members.

For a PyTorch classifier, the wrapper is a thin adapter — call the forward pass, run softmax, expose a DataLoader on the training and evaluation splits. For a language model, the wrapper exposes a per-token log-probability scorer over a target string.

Ship the wrapper in your project repo under a path like `audits/target_wrapper.py`. Pin ML Privacy Meter version in your requirements file. The wrapper is the project-specific code; the attacks are the vendored library.

## The audit config

A minimal audit configuration sketch (YAML-shaped; harness's exact schema follows its docs):

```yaml
audit:
  name: cifar10-resnet20-v2025.11
  seed: 42
target:
  wrapper: audits.target_wrapper:ResNet20Target
  train_split: data/cifar10/train
  holdout_split: data/cifar10/holdout
attacks:
  population:
    reference_population: data/cifar10/public-holdout
  reference:
    reference_model: artefacts/cifar10-resnet20-reference.pt
  shadow:
    num_shadow: 16
    attack_model: per_class_mlp
  lira:
    num_shadow: 64
    augmentation_queries: 4
    variance: pooled
    variant: online
report:
  tpr_at_fpr: [1e-2, 1e-3, 1e-4]
  plot_log_log: true
  bootstrap_resamples: 1000
  output_dir: artefacts/audit/cifar10-resnet20-v2025.11
```

Four things the config needs to name:

- **Seeds and determinism.** The audit is only reproducible if seeds are captured; the output report should emit the exact seed set used.
- **Attack hyper-parameters.** Shadow count, variance pooling, decoding policy for LLM attacks, augmentation policy. These are the knobs a reviewer will ask about.
- **Reporting policy.** Which TPR@FPR triple, whether to emit log-log plots, whether to bootstrap for CI (if you do the CI inside the harness) or to export per-example scores for the `model-evaluation-engineer` to bootstrap (if the CI methodology is centralised).
- **Output directory shape.** Pinned so the release pipeline knows where to collect the artefacts.

## Reading the output

A complete audit directory should contain:

- `summary.json` — headline metrics (AUC, TPR@FPR per attack, canary extraction rate if the extraction attack is in the config).
- `scores/{attack}.csv` — per-example scores and labels, for CI bootstrap and for the DPIA artifact.
- `plots/log_log_roc.{png,pdf}` — the log-log ROC with all configured attacks plotted together.
- `plots/dose_response.{png,pdf}` — if canaries were used, the dose-response curve.
- `config.resolved.yaml` — the config as resolved at run time, with any CLI overrides.
- `provenance.json` — git SHA, library versions, pinned requirements hash, random seeds, start / end timestamps.

Treat `provenance.json` as a release-bundle artifact — the DPIA cites by its hash. A stale harness run with no provenance is a run that cannot be defended.

## The CI handoff (to `model-evaluation-engineer`)

The confidence-interval methodology on MIA AUC and TPR@FPR is not owned here. The peer `model-evaluation-engineer` (L30 in the AI Governance family, alongside `ai-privacy-engineer`) owns:

- The bootstrap design — resample size, resample unit (per-example vs. per-shadow), variance-stabilising transform.
- The permutation-test threshold for "the attack is significantly above chance".
- The small-sample correction for TPR@FPR when the FPR floor implies few positive examples (the log-FPR regime is noise-dominated below the per-example floor of the eval set).

What this module does: emit the raw per-example `(score, label)` arrays into `scores/` so the peer's methodology can be applied. Record the version of the peer's methodology used in the DPIA alongside the raw numbers.

Chapter 10 draws the boundary; mod-101 chapter 09 and the AI Governance family map have the canonical peer-table.

## Integration into the release pipeline

Three practical patterns for running ML Privacy Meter as CI:

- **On-demand release gate.** Every release candidate triggers an audit run; the summary.json is checked against gate thresholds (LiRA TPR@10⁻³ < 0.05, canary extraction at insertion-count = 10 < 0.1, etc.). Fails the build if over the gate. The honest trade-off: full LiRA audits are expensive (k shadow models). Scale the audit mode to the release cadence.
- **Nightly diagnostic.** A reduced audit (population + reference attacks, no LiRA) runs every night on the latest training snapshot; drift triggers a full LiRA run on demand.
- **Scheduled deep audit.** A full LiRA + extraction audit runs monthly or quarterly on the production model; the output is the quarterly privacy-posture report.

Pick based on the release cadence. The first mode is right for infrequent releases of large models; the second is right for continuous-training pipelines.

## Known caveats of operating ML Privacy Meter

Things that will bite in production:

- **Shadow-training cost at scale.** The harness does not reduce the fundamental k-shadows-×-target-cost cost of LiRA. For a 70B model, k = 64 shadows is not realistic; the audit mode shifts to the chapter 05 extraction methodology against planted canaries (which can be done in a single training run per Steinke-2023).
- **Framework version drift.** ML Privacy Meter is primarily PyTorch; TensorFlow support lags. Pin the framework version and retest on any upgrade.
- **Dataset-size assumptions.** The population attack and the reference attack assume you can draw a *fresh* same-distribution population. For a privacy-sensitive dataset you cannot easily do that; the peer `ai-risk-engineer` or `model-evaluation-engineer` may need to provision the reference population separately.
- **LLM extraction coverage is active area.** The attacks that chapter 05 produces may need to be run outside the harness for current-literature techniques; keep the ability to run bespoke extraction evals alongside the harness and report both.
- **Not all attacks are implemented.** Property inference, extraction-against-production-APIs, and some newer attacks are not in ML Privacy Meter's core API; chapter 09's coverage mapping will show gaps, and your harness config should name which attacks you ran out-of-harness.

## Beyond the harness

Three categories ML Privacy Meter does not cover and the module routes elsewhere for:

- **DP (ε, δ) accounting.** mod-106 — Opacus, TF-Privacy, dp-accounting library.
- **Unlearning acceptance tests.** mod-110 — the harness returns a pre- and post-unlearning measurement; the unlearning algorithm itself is mod-110.
- **Federated-learning audit.** mod-107 — adapting MIA to the federated setting is a separate methodology.

Name these in your tool-coverage matrix.

## Summary — what you can do after this chapter

- Explain why a standing harness exists: comparability across releases, methodology defensibility, rate-of-work.
- Wire a target-model wrapper (classifier or small LM) into ML Privacy Meter.
- Configure a four-attack audit (population, reference, shadow, LiRA) with seeds, hyper-parameters, and reporting policy named.
- Read the output directory, identify the headline metrics, the per-example score artefacts, and the provenance record.
- Hand the CI methodology to `model-evaluation-engineer`; consume their output and attach to the DPIA.
- Place the harness in the release pipeline at the right cadence for your release model.

Exercise 07 is the full end-to-end wiring on a target you own. Chapter 09 is the coverage-story view that ties the attacks in the harness to the external standards a reviewer will cross-check against.
