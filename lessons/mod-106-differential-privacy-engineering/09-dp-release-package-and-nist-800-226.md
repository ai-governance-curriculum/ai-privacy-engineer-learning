# The DP Release Package and NIST SP 800-226

The previous chapters produced mechanisms (DP-SGD, PATE, DP synth, DP analytics), accountants (RDP, PRV, zCDP), audit logs, and the ε-defence narrative. This chapter is the capstone: the *release package* is the single artefact the DPIA (mod-111), the model-release review (`ai-evaluation-engineer`), the regulator, and any future acquirer will ask for by name. Producing it is not optional. This chapter is the specification.

The reference framework is NIST SP 800-226 — *Guidelines for Evaluating Differential Privacy Guarantees* (December 2023). It is the first US-federal standards document that enumerates what a reportable DP claim must contain. Every section of this chapter maps to a numbered section of 800-226; keep the PDF open while you work.

Primary references:

- NIST SP 800-226 — *Guidelines for Evaluating Differential Privacy Guarantees*. Dec 2023. [nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-226.pdf](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-226.pdf).
- NIST AI Risk Management Framework (AI RMF 1.0) — *Generative AI Profile*. 2024. [www.nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — the parent framework 800-226 anchors into.
- Mitchell et al. — *Model Cards for Model Reporting*. FAT* 2019. [arxiv.org/abs/1810.03677](https://arxiv.org/abs/1810.03677) — the model-card template DP sections slot into.
- Gebru et al. — *Datasheets for Datasets*. CACM 2021. [arxiv.org/abs/1803.09010](https://arxiv.org/abs/1803.09010) — datasheet for the DP-protected input dataset.
- Dwork, Kohli, Mulligan — *Differential Privacy in Practice: Expose Your Epsilons!* JPC 2019. [journalprivacyconfidentiality.org/index.php/jpc/article/view/689](https://journalprivacyconfidentiality.org/index.php/jpc/article/view/689).

## The release package — six artefacts

The release package is a *directory* with six files. Any DP release in production should have all six. Sketched:

```
release-{id}/
├── claim.md                    # the one-page DP claim + model card block
├── audit-log.json              # the per-step/per-epoch accountant trace
├── utility-eval.md             # the DP vs non-DP utility comparison
├── threat-model.md             # NIST SP 800-226 §5 mapping
├── engineering-attestations.md # the "yes we actually did this" checklist
└── manifests/
    ├── mechanism.json          # mechanism params, code hashes, library versions
    ├── dataset.datasheet.md    # input datasheet (Gebru-style)
    └── release.metadata.yaml   # release-time metadata (date, approver, scope)
```

Each file has a specific audience:

- **claim.md** — the regulator / privacy counsel.
- **audit-log.json** — the technical auditor who wants to replay the accountant.
- **utility-eval.md** — the product / downstream-consumer.
- **threat-model.md** — the security reviewer / DPO.
- **engineering-attestations.md** — the code reviewer / audit trail.
- **manifests/** — the auditor / future-you.

A sloppy release package consolidates these into a single README. Resist. Each file has a different audience and a different update cadence; separating them is how the package survives version-controlled change over time.

## claim.md — the one-page DP claim

The one page a regulator, privacy counsel, or acquirer reads first. If they read nothing else, this is what they read.

Template:

```markdown
# DP Release Claim — {release-id}

## Mechanism and target
- Mechanism: DP-SGD (subsampled Gaussian) | PATE (Confident-GNMax) | DP synth (AIM) | DP analytics (Laplace)
- Release artefact: model weights | synthesised dataset | aggregate dashboard
- Scope of claim: {what records / users / events are inside the DP boundary}

## The number
- Final (ε, δ): {ε} at δ = {δ}
- Neighbouring relation: add-or-remove one {record | user | event}
- Granularity: {record-level | user-level | event-level}
- Composition unit: {per-release | per-lifetime | per-user per-year}
- Accountant: {library + version + accountant family}; cross-checked with {second accountant}

## English-language meaning
- {Bayesian / hypothesis-test / counterfactual framing, one paragraph, picked from chapter 05}

## Empirical corroboration
- LiRA TPR @ FPR = 1e-3: {value ± CI}
- Canary extraction rate at {k} canaries: {value}
- (links to mod-105 attack reports)

## What this ε does not protect
- {bullet list, concise; details in threat-model.md}

## Anchor
- Comparable to {Apple keyboard telemetry per-donation / Google Mobility Reports / US Census 2020} at {comparable ε regime}

## Approval
- Signed by {DPO role name} on {ISO date}
- Next review on {ISO date}
```

The language discipline:

- **Specific numbers, not ranges, in `## The number`.** "ε = 4.03 at δ = 1e-6" is correct; "ε around 4" is a signal you did not do the audit.
- **English meaning as *one* framing, consistent across the document.** Picking Bayesian here and switching to hypothesis-test in threat-model.md is how a reviewer loses trust.
- **"What does not protect" is bullet-level, not prose.** Reviewers scan this section; make it skimmable.

The claim.md gets copied into the model card's DP section verbatim; the model card is a composition of claim.md, utility-eval.md, and the mod-105 attack summary.

## audit-log.json — the accountant trace

Chapters 02–04 defined the schema; this chapter formalises it as a required artefact. Minimum fields (reproduced from chapter 02 with release-package additions):

```jsonc
{
  "run_id": "dp-sgd-2026-11-04-abcdef",
  "git_sha_of_training_code": "abc123...",
  "data_sha256": "deadbeef...",
  "mechanism": "sampled_gaussian",
  "accountant": {
    "name": "PRVAccountant",
    "version": "opacus==1.5.2",
    "tightness_note": "Numerical PRV composition; Opacus default as of 1.5.0."
  },
  "cross_check_accountant": {
    "name": "prv_accountant",
    "version": "0.2.0",
    "final_epsilon_at_delta_1e-6": 4.05
  },
  "mechanism_params": {
    "sampling_rate": 0.00125,
    "noise_multiplier": 1.1,
    "clipping_norm_C": 1.0,
    "steps_per_epoch": 800,
    "n_epochs": 50,
    "sampling_regime": "poisson_true"
  },
  "neighbouring_relation": "add_or_remove_one_record",
  "granularity": "record_level",
  "epochs": [
    {"epoch": 1,  "cum_steps": 800,   "epsilon_at_delta_1e-6": 0.97},
    // ...
    {"epoch": 50, "cum_steps": 40000, "epsilon_at_delta_1e-6": 4.03}
  ],
  "final_claim": {"epsilon": 4.03, "delta": 1e-6},
  "sanity_checks": {
    "monotonicity_in_T": "pass",
    "monotonicity_in_sigma": "pass",
    "monotonicity_in_q": "pass",
    "cross_library_agreement_within_10pct": "pass"
  }
}
```

For non-DP-SGD mechanisms (PATE, DP synth, DP analytics), adapt the field names but keep the schema spirit: mechanism + params + accountant + cross-check + per-step/per-release trace + sanity checks.

## utility-eval.md — the DP vs non-DP comparison

The claim's number means nothing without a utility-cost statement. The utility report:

- **Non-DP baseline.** Train (or compute) the equivalent non-DP artefact under the same recipe. Report its metric on the eval set.
- **DP artefact at the chosen ε.** Same metric.
- **Sweep.** Report the metric at ε ∈ {1, 4, 8, ∞} (where ∞ is non-DP). Chapter 05's defence depends on this sweep.
- **Statistical significance.** Each metric with a CI (bootstrap or hold-out). An ε = 4 utility that is within noise of ε = 8 is a useful finding.
- **Downstream task impact.** If the release is a model, run it through the production inference stack and measure the user-visible metric degradation. If the release is a synthetic dataset, run multiple downstream tasks (tier 2 of chapter 07's audit).
- **Known failure modes.** "At ε = 4, the model's precision on rare class X drops by Y%" — name them.

The template:

```markdown
## Utility evaluation

| Metric | Non-DP | ε=8 | ε=4 | ε=1 | CI method |
|---|---|---|---|---|---|
| Top-1 accuracy | 82.3 | 80.9 | 77.8 | 61.4 | bootstrap n=10k |
| Precision at rare class | 71.2 | 68.1 | 60.3 | 22.7 | hold-out, 95% Wilson |
| Downstream pipeline KPI | 0.94  | 0.93 | 0.89 | 0.68 | prod A/B over 7 days |

### Chosen operating point
ε = 4 (utility within {gap} of non-DP on primary metric; above the product gate of {threshold}).

### Known failure modes
- Rare-class precision degrades more than aggregate accuracy.
- Downstream pipeline is more noise-sensitive than the top-1 metric suggests.
```

## threat-model.md — what the DP claim does and does not protect

Mapped to NIST SP 800-226 §5 (Threat Model). The two-column layout from chapter 05:

```markdown
## Threat model — what this (ε, δ) does and does not protect

| Threat                                                   | Mitigated by this DP claim? | If not, by what control? |
|----------------------------------------------------------|-----------------------------|--------------------------|
| Membership inference on any single record                | YES (within ε bound, cross-checked with mod-105 LiRA) | — |
| Attribute inference about a specific user's column       | YES                         | — |
| Verbatim extraction of a specific record from an LLM     | YES empirically; see mod-105 canary report | — |
| Fredrikson-style model inversion                         | YES within ε bound          | — |
| Aggregate statistical properties of the dataset          | NO                          | Data minimisation (mod-103); dataset datasheet |
| User-level attacks when DP is at record-level            | NO                          | User-level DP in a future release; see §composition below |
| Multiple compositions over lifetime of the model         | PARTIAL; see §composition    | Per-release budget; opt-out policy |
| Attacks on model weights at rest                         | NO                          | Confidential computing (mod-108) |
| Prompt injection / jailbreak                             | NO                          | `ai-security-engineer` scope |
| Training-time log leakage of per-example gradients       | NO                          | Code review; engineering attestations |
| DP mechanism engineering bugs (BN, sampling, etc.)       | NO                          | engineering-attestations.md |

## Composition over release lifetime
Current release: ε = 4, δ = 1e-6 under PRV accounting; per-release, record-level.
Planned retraining cadence: quarterly.
Composed user-level ε per year under PRV (assuming non-disjoint user pool): approximately {value}.
Mitigations: {fresh randomness composition / user-level DP in next major release / opt-out refresh}.
```

Two moves to notice:

- **"Mitigated by this DP claim" is a column with three values** (YES, NO, PARTIAL). PARTIAL is the honest answer for lifetime composition and for threats where DP is one layer in defence-in-depth.
- **Every NO has an owning control.** The release package names what *does* protect against each threat DP does not. Chapter 09 of mod-105 taught this pattern for attacks; the same table lives here for DP.

## engineering-attestations.md — the audit trail

The one-page checklist of engineering disciplines actually followed. Each item is a *claim that a specific thing is true in the codebase*, with a pointer.

Template (DP-SGD example; adapt for other mechanisms):

```markdown
## Engineering attestations

### Model prep
- [x] BatchNorm removed and replaced with GroupNorm (commit abc123 — `ModuleValidator.fix` in training_main.py:42)
- [x] `ModuleValidator.validate(strict=True)` called (training_main.py:45; CI test test_dp_sgd.py::test_module_validator)

### Sampling
- [x] `poisson_sampling=True` passed to `make_private_with_epsilon` (training_main.py:88)
- [x] `UniformWithReplacementSampler` wraps the DataLoader at runtime (verified by assertion in training_main.py:95)

### Clipping and noise
- [x] `max_grad_norm = 1.0` set (training_main.py:92)
- [x] `noise_multiplier = 1.1` set (same)
- [x] `BatchMemoryManager` handles virtual batching (training_main.py:102–120)
- [x] Clipping is per-sample, pre-accumulation (verified by Opacus 1.5.2 implementation)

### Loss
- [x] Loss function uses `reduction='none'` (training_main.py:75)
- [x] No other `.backward()` calls on reduced losses (code-reviewed, commit abc123)

### Accountant
- [x] Accountant: `PRVAccountant` (training_main.py:88)
- [x] `get_epsilon(delta=1e-6)` called each epoch; written to audit-log.json
- [x] Cross-checked against `prv_accountant==0.2.0` (cross_check.py)
- [x] Accountant state saved on checkpoint and restored on resume (test_resume_accountant.py)

### Logging / side channels
- [x] No per-example gradient norms logged outside the DP pipeline (grep audit; commit abc123)
- [x] No model activations persisted outside the trusted boundary during training (code review)

### Sanity checks
- [x] Monotonicity in T, σ, q (accountant_sanity_tests.py)
- [x] Cross-library ε agreement within 10% (cross_check.py)
```

The three-column variant (`attestation | code-pointer | reviewer-signoff`) is useful when the release is going through a formal review process.

## manifests/ — the "replay the mechanism" bundle

For a technically detailed auditor who wants to re-execute the mechanism on the input data:

- **manifests/mechanism.json.** All mechanism parameters, library versions, git SHAs of training code, docker image SHA, system Python version, GPU model, random seeds.
- **manifests/dataset.datasheet.md.** Gebru-style datasheet for the input dataset: source, provenance, size, schema, licensing, known biases, cleaning steps, hash of the processed data.
- **manifests/release.metadata.yaml.** Release-time metadata: release date, DPO approval date, approver name/role, scope of release ("internal", "contracted partner", "public"), retention policy for the artefact, next-review date.

The mechanism.json is the file a reviewer uses to answer "can I reproduce this ε?". The dataset.datasheet.md is the file a reviewer uses to answer "what records went into this?". The release.metadata.yaml is the file a reviewer uses to answer "who owns the next review?".

## The NIST SP 800-226 section mapping

For the release package to defend to a federal reviewer, each section of 800-226 must have a corresponding section in the package. The mapping (page references are to the 2023 version):

| NIST SP 800-226 section | Release package file |
|---|---|
| §2 — DP definitions and theorems | claim.md (the number) + audit-log.json (the accountant) |
| §3 — Mechanism design | manifests/mechanism.json + engineering-attestations.md |
| §4 — Properties (composition, post-processing, group privacy) | claim.md (§composition over release lifetime) |
| §5 — Threat model | threat-model.md |
| §6 — Implementation (engineering concerns) | engineering-attestations.md |
| §7 — Evaluation (utility and privacy) | utility-eval.md + mod-105 attack report |
| §8 — Communication (reporting and interpretation) | claim.md (English-language meaning section) |

A release package with every section populated maps 1:1 to 800-226. A reviewer coming from a US federal context will skim the headings and check boxes.

## The review workflow — who signs each file

The DP release is not something one engineer signs off alone. The standard workflow:

- **claim.md** — DP engineer drafts; DPO / privacy counsel approves; model owner acknowledges.
- **audit-log.json** — DP engineer produces; `ai-evaluation-engineer` peer reviews for cross-check-accountant agreement; DP engineer signs.
- **utility-eval.md** — ML engineer produces the utility sweep; DP engineer verifies the ε values are correct; product owner acknowledges the utility trade-off.
- **threat-model.md** — DP engineer drafts; `ai-security-engineer` peer reviews for threat completeness; DPO signs.
- **engineering-attestations.md** — DP engineer produces; separate code reviewer (not the author) signs each line.
- **manifests/** — DP engineer produces; auditable by CI job (e.g., auto-generated from training run).

The point of distinct sign-offs is accountability. If a BN-still-in-model bug slips through, the audit trail shows which attestation was falsely checked and by whom.

## The anti-patterns — how a DP release package goes wrong

A ranked list. The audit catches them in roughly this order.

- **One of the six files is missing.** Most common; usually threat-model.md or utility-eval.md. Fix: run the release-package linter on every release (exercise 08 ships one).
- **claim.md quotes an ε without a δ.** Auto-reject. (ε, δ) is a pair.
- **claim.md quotes an ε without a granularity.** Auto-reject.
- **audit-log.json has no cross-check.** Technically passes 800-226; practically fails internal review.
- **utility-eval.md has no non-DP baseline.** The ε cost is not interpretable.
- **engineering-attestations.md has unchecked boxes.** Any `[ ]` is a release-blocker.
- **threat-model.md's "DP does not protect" column has no owning control.** Half-done threat model.
- **mechanism.json pins no library version.** Un-replayable release.
- **The whole thing is a single README.md.** Hard to version, hard to review, hard to update.

## The linter

Automating the release-package check is a one-afternoon project and a massive quality win. Exercise 08 ships one. It walks the release directory, verifies each required file is present, parses audit-log.json, re-runs the cross-check accountant, verifies the utility sweep has the required ε points, and emits a pass/fail report plus a list of findings.

Run the linter as a CI gate on every release-package commit. Track the linter's pass rate over time as a release-process KPI.

## Summary — what you can do after this chapter

- Name the six files of a DP release package and their distinct audiences.
- Produce claim.md from chapter 05's defence narrative — number, English meaning, empirical corroboration, anchor, approval.
- Produce audit-log.json from chapter 02's schema, extended with git SHAs, data hashes, and sanity checks.
- Produce utility-eval.md with the DP-vs-non-DP sweep and acknowledge the utility cost honestly.
- Produce threat-model.md as a two-column mapping where each "DP does not protect" row names an owning control.
- Produce engineering-attestations.md as a checklist with code pointers and reviewer sign-off.
- Populate manifests/ (mechanism, dataset, release metadata).
- Map each release-package section to the corresponding NIST SP 800-226 section.
- Run the release-package linter as a CI gate (exercise 08).

Chapter 10 draws the boundary to the peer `ai-risk-engineer` track's mod-107 DP coverage.
