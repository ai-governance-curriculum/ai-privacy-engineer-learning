# Exercise 05: Synthetic-Data Substitute Build and Audit

**Estimated effort:** 3 hours · **Output:** `synth-build/` (code + synthesised dataset) + `synth-audit.md` (utility + privacy audit) + a datasheet for the synthetic dataset · **Primary chapter:** 06

## Objective

Build a **synthetic-data substitute** for one tabular (or small-text) dataset you care about, then **audit** it against the utility *and* privacy metrics chapter 06 specifies. Produce a datasheet for the synthetic set that references the audit, and record whether the synthetic set is suitable for a named downstream (training / evaluation / demo).

## Prerequisites

- Chapter 06 (synthetic-data substitutes) read.
- Exercise 01 of this module (a real dataset + its datasheet) completed, or access to a representative tabular dataset (UCI Adult / public MIMIC-III subset / COMPAS / a synthesisable subset of your work corpus).
- Python environment with [SDV](https://sdv.dev/) or an equivalent synthesiser installed. Alternatively, a managed service (Gretel, MOSTLY AI, Tonic) with a free tier and a sanitised dataset small enough to stay inside their quota.
- Basic familiarity with scikit-learn for the downstream-task utility measurement.

## Problem statement

Produce three deliverables.

### Deliverable A — `synth-build/`

Code and artifacts for the synthesis:

- A short script that loads the real dataset, trains / fits / runs a chosen synthesiser, and emits a synthetic dataset of a specified size.
- Choice of synthesiser documented with a one-paragraph rationale (why CTGAN / TVAE / TabDDPM / PrivBayes / Gaussian copula / Gretel / MOSTLY AI). State the task and why the choice fits.
- A `synthesiser.yaml` recording the synthesiser name, version, hyperparameters, training-data reference, and training-cost (wall time; whether on CPU / GPU).
- The resulting synthetic dataset file (in a format matching the real dataset's schema).

### Deliverable B — `synth-audit.md`

A utility + privacy audit following the chapter 06 template. Must report:

- **Utility — distributional similarity.** Per-column marginals (K-S / chi-square); pairwise correlation deltas; Wasserstein / MMD on a few key marginals.
- **Utility — TSTR (train on synthetic, test on real).** Pick at least one downstream classification / regression task that is representative; fit a model on the synthetic set; evaluate against a real held-out set; compare to the real-trained baseline.
- **Utility — TRTS (train on real, test on synthetic).** As a drift diagnostic.
- **Privacy — exact-row match.** 0 / N real training rows reproduced verbatim (for text synth, use hash shingles).
- **Privacy — DCR / NN distance ratio.** Compute per the chapter-06 definition.
- **Privacy — MIA shadow-model attack.** A simple shadow-model attack on the synthesiser; report AUC vs. random 0.5.
- **Privacy — attribute inference.** Pick one sensitive column; measure attack AUC above baseline.
- **Residual-risk claim.** The paragraph that closes the audit: named residuals, compensating controls, re-opening triggers, signatories.

The audit is readable by a DPO — write it in prose with tables, not a Jupyter dump.

### Deliverable C — synthetic-dataset datasheet

A datasheet for the synthetic dataset (shape from exercise 01), extended with the three chapter-06 synthetic-specific fields: `synthesiser`, `source_datasets`, `audit_report_ref`. Populate every field.

The datasheet **also states the downstream-use policy for the synthetic set** — internal eval only / publishable alongside a model release / shareable with a downstream integrator — and the policy decision rationale.

## Starter guidance

- **Pick a small dataset.** A 10k-row tabular is plenty. A 100k-row tabular is more than you need for this exercise. Overfit is a real issue.
- **Baseline first.** Fit a Gaussian copula or a marginal-sampler before TabDDPM. If the baseline passes utility, prefer it — it is easier to audit and defend.
- **The TSTR measurement is the most useful single number.** A synth set whose TSTR F1 is within 5% of the real baseline is usable for many downstreams.
- **Use a held-out slice from the real data, never seen by the synthesiser, for both TSTR and DCR.** Without a held-out split the privacy metrics are not meaningful.
- **Run MIA with shadow models even if small (10 shadow models, 1000 rows each).** The AUC should be close to 0.5; anything approaching 0.7 is a leak.
- **Be honest about failure.** If the audit comes back with a privacy-leak finding, record the finding and *recommend not using the synth for the proposed downstream*. The exercise is not failed by finding a leak; it is failed by hiding one.
- **If your synthesiser supports a DP mode, run at least one DP experiment** — compare utility at ε=1 vs. ε=3 vs. no-DP. Record the budget and the utility cost.

## Acceptance criteria

- [ ] `synth-build/` runs end-to-end on a clean checkout, producing a synthetic dataset of a specified size.
- [ ] Utility section reports both TSTR and TRTS against a named downstream task with a stated baseline.
- [ ] Privacy section reports all five chapter-06 metrics (exact-row, DCR, MIA AUC, attribute inference, singling-out / linkability discussion).
- [ ] Residual-risk claim names at least two residual scenarios, compensating controls, re-opening triggers, and signatories.
- [ ] Synthetic datasheet is complete, references the audit, and states a downstream-use policy with rationale.
- [ ] Choice of synthesiser is defensible to a reader: the rationale paragraph states task, data characteristics, and why the alternative was rejected.
- [ ] The downstream-use policy is *refused* or *restricted* if any privacy metric fails the audit.

## Stretch goals

- Compare two synthesisers on the same data (e.g. CTGAN vs. TabDDPM, or a baseline Gaussian copula vs. a neural model). Produce a one-table comparison across utility and privacy metrics.
- Run a **DP-synthesis** experiment (DP-CTGAN or OpenDP SmartNoise Synth) and report the (ε, δ) + MIA AUC trade-off curve.
- Produce a synthetic **text** corpus using an LLM-paraphrase flow against your free-text column. Run the mod-105 memorisation-style hash-shingle check and the mod-104 PII post-filter on the output. Report residual PII rate.
- Produce the ML-BOM fragment (exercise 06 input) for the synthetic dataset.
- Have a classmate / colleague adversarially audit your synth: give them the synthetic set and a side-channel of real-side quasi-identifiers, see whether they can single out a row.

## What this lands

- The synthetic dataset is a reusable substitute for demo / eval / training-augmentation flows.
- The audit is the artifact an auditor or DPO cites when asking "is this synth safe?"
- The synthetic-dataset datasheet participates in the ML-BOM (exercise 06) on equal footing with the real datasets.
- The downstream-use policy becomes the engineering guardrail — the synthetic set is used *within* the policy and *not outside* it.

Version the audit. Date the signatures. Re-audit on synthesiser upgrade or on a material distribution shift in the real data.
