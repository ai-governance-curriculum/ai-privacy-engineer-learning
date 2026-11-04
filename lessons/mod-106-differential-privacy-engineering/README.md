# mod-106-differential-privacy-engineering: Differential Privacy Engineering for ML and LLM Training

**Estimated effort:** 18 hours

Differential Privacy is the one privacy guarantee a regulator, auditor, or acquirer can cross-check against a formal definition rather than against your word. This module teaches DP at production depth: not just training a model with Opacus, but owning the (ε, δ) policy decision, picking the right accountant, understanding the PATE and synthetic-data alternatives, deploying DP analytics when only aggregates need to leave the trust boundary, and producing the release package that maps to NIST SP 800-226.

## Learning objectives

- Read the Dwork / Roth DP monograph and NIST SP 800-226 — understand pure (ε)-DP, approximate (ε, δ)-DP, Rényi DP, zero-concentrated DP, Gaussian DP; know which accountant each production library uses.
- Engineer DP-SGD training end-to-end with Opacus and TensorFlow Privacy — noise multiplier, L2 clipping norm, lot / batch sampling, virtual batching, Rényi DP accounting, (ε, δ) budget allocation across epochs.
- Defend the (ε, δ) choice under audit — what does ε = 1 vs. ε = 8 mean to a downstream regulator, what are published norms (Apple, Google, US Census 2020), how do you report the budget alongside the model card.
- Engineer PATE (Private Aggregation of Teacher Ensembles) as an alternative to DP-SGD when the task fits — teacher ensemble, noisy voting, data-dependent vs. data-independent privacy analysis.
- Engineer DP synthetic data releases — DP-GAN, PATE-GAN, DP-CTGAN, DP-WGAN, Private-PGM, and the OpenDP SmartNoise synth stack; audit utility vs. privacy of the synthesised set.
- Deploy the Google Differential Privacy library (PipelineDP / plume) and OpenDP SmartNoise for DP analytics and dashboards as the alternative to DP training when only aggregates are released.
- Produce a DP release package — model weights or synth data, formal (ε, δ) claim, accountant audit log, utility evaluation vs. the non-DP baseline, and threat-model note on what the DP guarantee does and does not protect against — mapped to NIST SP 800-226 evaluation guidelines.
- Cite the boundary to `ai-risk-engineer` (peer, level 25) mod-107 — that module covers Opacus DP-SGD at working depth; this module owns the production-grade (ε, δ) defence, accountant choice, PATE, DP synth, and release packaging.

## Chapters

1. [Why DP is the mathematical privacy primitive](01-why-dp-is-the-mathematical-privacy-primitive.md) — (ε, δ)-DP, pure vs approximate, the four refinements (RDP / zCDP / GDP / f-DP), what DP does and does not protect, the module glossary.
2. [Accountants and the production library mapping](02-accountants-and-production-library-mapping.md) — accountant family tree, Opacus / TF-Privacy / Google DP / OpenDP, the audit log schema, the sensitivity-assumption trap, cross-library sanity checks.
3. [DP-SGD with Opacus, end to end](03-dp-sgd-with-opacus-end-to-end.md) — ModuleValidator, Poisson sampling, virtual batching with BatchMemoryManager, the PrivacyEngine, the seven engineering failure modes, the state-of-the-art recipe.
4. [DP-SGD with TensorFlow Privacy](04-dp-sgd-with-tensorflow-privacy.md) — Vectorized DP optimisers, microbatching, the TF-Privacy accountant, Keras-specific traps, custom `GradientTape` loops.
5. [Defending the (ε, δ) choice under audit](05-defending-epsilon-and-delta-under-audit.md) — the four regulator questions, English-language framings, the anchor table (Apple / Google / Census), per-release vs lifetime ε, the model-card DP block.
6. [PATE — Private Aggregation of Teacher Ensembles](06-pate-ensembles.md) — teacher construction, Confident-GNMax aggregation, data-dependent vs data-independent analysis, PATE vs DP-SGD decision.
7. [DP synthetic data](07-dp-synthetic-data.md) — DP-GAN / PATE-GAN / DP-CTGAN / DP-WGAN, Private-PGM / MST / AIM, the SmartNoise Synth stack, the three-tier utility audit, the Groundhog Day warning.
8. [DP analytics with PipelineDP and SmartNoise](08-dp-analytics-pipelinedp-and-smartnoise.md) — contribution bounding, partition selection, Laplace / Gaussian / exponential mechanisms, standing-dashboard composition, budget allocation.
9. [The DP release package and NIST SP 800-226](09-dp-release-package-and-nist-800-226.md) — six-file release package (claim / audit-log / utility-eval / threat-model / attestations / manifests), NIST SP 800-226 section mapping, the review workflow, the linter.
10. [Boundary to `ai-risk-engineer`](10-boundary-to-ai-risk-engineer.md) — what mod-107 covers at working depth vs what this module owns at production depth; the three recurring handoffs.

## Exercises

See [`exercises/`](exercises/):

1. [DP foundations and accountant selection](exercises/exercise-01-dp-foundations-and-accountant-selection.md)
2. [DP-SGD training with Opacus](exercises/exercise-02-dp-sgd-training-with-opacus.md)
3. [DP-SGD training with TensorFlow Privacy](exercises/exercise-03-dp-sgd-training-with-tensorflow-privacy.md)
4. [(ε, δ) defence under audit](exercises/exercise-04-epsilon-delta-defence-under-audit.md)
5. [PATE ensemble engineering](exercises/exercise-05-pate-ensemble-engineering.md)
6. [DP synthetic data release with SmartNoise](exercises/exercise-06-dp-synthetic-data-release-with-smartnoise.md)
7. [DP analytics with Google PipelineDP](exercises/exercise-07-dp-analytics-with-google-pipelinedp.md)
8. [NIST SP 800-226 release-package drill](exercises/exercise-08-nist-800-226-release-package-drill.md)

## Structure

- `01-…md` … `10-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs (scaffolded placeholder).
- `quizzes/`: knowledge checks (scaffolded placeholder).
- `resources.md`: external references — the Dwork / Roth monograph, NIST SP 800-226, Opacus, TF-Privacy, PATE / PATE-GAN, SmartNoise, PipelineDP.

## Where this module sits in the track

- Upstream dependencies: mod-101 (frameworks and position — why DP is a privacy-engineer tool), mod-103 (data governance — DP is downstream of the data minimisation discipline), mod-105 (memorisation and MIA — the empirical corroborator for the DP claim).
- Downstream consumers: mod-107 (federated learning — DP composes with secure aggregation), mod-110 (unlearning — DP is part of what makes unlearning auditable), mod-111 (DPIA — quotes this module's release package by name), mod-112 (program and communication — the regulator-facing narrative for the ε).
- Peer boundary: `ai-risk-engineer` mod-107 (working depth), this module (production depth). See chapter 10.
