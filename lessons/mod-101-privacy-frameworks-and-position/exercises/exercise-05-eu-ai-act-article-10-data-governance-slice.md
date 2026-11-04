# Exercise 05: EU AI Act Article 10 — Data-Governance Slice

**Estimated effort:** 2 hours · **Output:** one Art. 10 clause-by-clause data-governance plan + one Datasheet (one dataset) + one Art. 10(5) special-category justification memo · **Primary chapter:** 03

## Objective

Produce the three artifacts an engineering team must have on file to defend EU AI Act Art. 10 compliance for a high-risk AI system: a **clause-by-clause data-governance plan**, a **Datasheet-for-Datasets** entry for one in-scope training / validation / test dataset, and an **Art. 10(5) special-category justification memo** if your system uses special-category data for bias monitoring (or an "N/A with reasoning" memo if it does not). Position the three against the Art. 72 post-market monitoring interface.

## Prerequisites

- Chapter 03 (EU AI Act Art. 10 / 15 / 72) read.
- [EU AI Act Regulation (EU) 2024/1689](https://eur-lex.europa.eu/eli/reg/2024/1689/oj) Art. 10 open; skim Art. 6 (classification), 15, 26, 72, Annex III, Annex IV.
- Datasheets-for-Datasets template from the Gebru et al. 2018 paper ([arXiv:1803.09010](https://arxiv.org/abs/1803.09010)).
- A specific high-risk-or-high-risk-adjacent AI system in mind. You can reuse the Exercise 01 system if its classification plausibly includes a high-risk Annex III use. If not, use the following scenario:

> **Scenario (default):** A US-headquartered insurance company offers an ML-based premium-adjustment feature to EU customers. The model predicts claim-probability from policyholder-provided information, public datasets, and prior-claim history. It affects the price offered to the consumer. Classification under Annex III (insurance pricing being an explicitly listed high-risk area in the enacted text) is being considered — proceed as if the system is high-risk for the purposes of this exercise, and flag any classification contingency.

## Problem statement

Produce three deliverables.

### Deliverable A — Art. 10 clause-by-clause data-governance plan

A document with one section per Art. 10 sub-clause (10(2)(a)–(h), 10(3), 10(4), 10(5), 10(6)). Each section states:

1. **What the clause asks** — one or two plain-language sentences, grounded in the Regulation text.
2. **How it applies to your system** — what the obligation requires of *your* training / validation / test data sets.
3. **The engineering control** — the process and / or technical mechanism you have (or would) implement.
4. **The test / review** — how it is verified (bias-evaluation report, representativeness statistical test, annotation-agreement metric, dataset-card completeness check in CI, etc.).
5. **The evidence artifact** — what is retained on file (and where), satisfying Annex IV(2)(d) technical documentation requirements.
6. **The deferral** — any part that is not yours to own (counsel decides classification; `ai-governance-analyst` maintains the dataset register; `model-evaluation-engineer` owns statistical calibration depth; etc.).

Specific sub-clauses that must appear:

- **10(2)(a) design choices** — what design choices on data were made and why.
- **10(2)(b) collection processes and origin** — including the *original* purpose of personal data collection where applicable.
- **10(2)(c) data-preparation operations** — annotation, labelling, cleaning, updating, enrichment, aggregation.
- **10(2)(d) assumptions** — what the data are supposed to measure / represent.
- **10(2)(e) availability / quantity / suitability** assessment.
- **10(2)(f) biases examination** — likely to affect health, safety, fundamental rights, or lead to prohibited discrimination.
- **10(2)(g) bias mitigation** — measures to detect, prevent, mitigate.
- **10(2)(h) data gaps / shortcomings** and how addressed.
- **10(3)** — relevance, representativeness, error-freeness, completeness, statistical properties.
- **10(4)** — geographical / contextual / behavioural / functional context.
- **10(5)** — special-category data for bias testing (justification memo in Deliverable C).
- **10(6)** — appropriate controls / governance.

### Deliverable B — a Datasheet for one in-scope dataset

A full **Datasheet for Datasets** (Gebru et al. 2018) for **one** training, validation, or test dataset that your system uses. Fill every section. Required sections (from the paper):

1. Motivation
2. Composition
3. Collection process
4. Preprocessing / cleaning / labelling
5. Uses
6. Distribution
7. Maintenance

Cross-reference each Datasheet section to the Art. 10 sub-clauses it discharges. The Datasheet is the primary evidence artifact for most of Art. 10(2); your clause-by-clause plan cites it.

### Deliverable C — Art. 10(5) special-category justification memo

If your system uses special-category data under Art. 10(5) for bias monitoring, write a memo that:

1. Identifies the special-category attributes in scope (e.g., racial / ethnic origin, health, religion).
2. States the specific bias concern that justifies strict necessity (e.g., "we are testing for disparate impact on a protected class our product's pricing could affect").
3. Lists the safeguards in place: pseudonymisation, access controls, purpose-limited storage with TTL, logging of access, deletion on completion of bias tuning.
4. Describes the retention + deletion plan for the special-category data when bias tuning is complete.
5. Notes the GDPR cross-reference — Art. 10(5) is *not* a standalone GDPR Art. 6 / 9 basis; the GDPR basis still has to be documented separately (counsel-owned).
6. References the regulator stance from EDPB Opinion 28/2024 (chapter 06) on the proportionality / necessity test.

If your system does not use special-category data at all, write a short "N/A with reasoning" memo explaining the data-inventory check that supports the claim (specifically: scanning training data for implicit special-category inferences, since models can infer special-category traits from non-special-category data per EDPB guidance).

## Starter guidance

- **Cite the Regulation sub-clauses by number.** `Art. 10(2)(f)` is different from `Art. 10(2)(g)`; the plan should show you read them separately.
- **Do not duplicate the Datasheet in the Art. 10 plan.** The plan is the clause-by-clause reading; the Datasheet is the primary evidence. Cross-reference by Datasheet section.
- **Art. 10(5) is narrow.** "Strictly necessary" is the governing phrase. Over-reaching Art. 10(5) is a defensibility fail — the safeguards bar is high.
- **Flag classification contingency.** If your system's classification under Annex III is uncertain, do not hide that — flag it as a legal decision (counsel) and author the plan *as if* high-risk for the exercise.
- **Position Art. 72.** The last section of the plan should state how the artifacts in the plan feed Art. 72 post-market monitoring — specifically, which of the plan's outputs refresh periodically.

## Acceptance criteria

- [ ] Deliverable A has one section per Art. 10 sub-clause listed above, with all six content elements per section.
- [ ] Every section's control, test, and evidence are specific to your system (generic text is a fail).
- [ ] Deliverable B is a complete Datasheet with all seven sections, cross-referenced to Art. 10 sub-clauses.
- [ ] Deliverable C is either a defensible Art. 10(5) justification with safeguards, retention, and EDPB-Opinion-28/2024 reference — *or* an N/A memo with a documented data-inventory check for inferred special-category attributes.
- [ ] The plan closes with an Art. 72 interface section naming the artifacts that refresh periodically and the cadence.
- [ ] Any out-of-ladder deferral (counsel on classification, L35 release-assurance packaging, L50 architect on control catalogue) is named per chapter 08.

## Stretch goals

- Extend the plan with an **Art. 15 cybersecurity-and-robustness slice** — listing the privacy-specific attack classes (MIA, extraction, inversion) you've tested against, with placeholders for the probe reports that will land in `mod-105-memorisation-extraction-and-mia`.
- Add a **GPAI mapping** — if your training corpus or base model crosses into general-purpose-AI territory, note which GPAI obligations (Chapter V of the AI Act) your Art. 10 evidence partially discharges.
- Author the **Annex IV(2)(d) formal section** — the regulator-facing render of your plan, sized for a Notified Body review. (This stretches into `ai-evaluation-engineer` L35 territory — do a draft and name the escalation.)

## What this lands

- The Art. 10 plan + Datasheet + Art. 10(5) memo are the three artifacts that go into the Annex IV technical documentation data-governance section. They are directly reusable in `mod-103-data-governance-and-minimisation` and `mod-111-dpia-and-privacy-reviews`.
- The Art. 72 interface section is the first draft of your post-market privacy monitoring plan — extended in `mod-112-privacy-program-and-communication`.
