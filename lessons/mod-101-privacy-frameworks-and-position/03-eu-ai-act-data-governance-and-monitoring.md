# EU AI Act — Article 10 (Data Governance), Article 15 (Accuracy / Robustness / Cybersecurity), Article 72 (Post-Market Monitoring)

GDPR tells you how to handle personal data. The EU AI Act (Regulation (EU) 2024/1689) tells you how to handle the *AI system itself* — specifically, the obligations that bite when the system is classified as **high-risk**. Three articles are the anchor for AI Privacy Engineering practice: **Art. 10** on data and data governance, **Art. 15** on accuracy / robustness / cybersecurity, and **Art. 72** on post-market monitoring.

Primary source: [EU AI Act — Regulation (EU) 2024/1689](https://eur-lex.europa.eu/eli/reg/2024/1689/oj). Read Articles 6–13 (risk classification + providers of high-risk AI), 10, 15, 72, and Annex IV once end-to-end. The Act entered into force on 1 August 2024 with staged applicability; the specific dates and the Commission's staged guidance are moving — do not cite dates from training data, cite the official timeline at each review.

> This is the *engineer's* reading. The AI Act legal interpretation — exact scope of "placing on the market", conformity-assessment routing, Annex III classification — is for counsel and the DPO. You own the engineering surface that makes the Act's obligations provable.

## Where you live in the Act

- **Provider** — the entity that develops or has an AI system developed and places it on the market under its own name. Carries the heaviest obligations, including Art. 10, 15, 72.
- **Deployer** (formerly called "user" in draft text) — the entity using an AI system under its own authority. Has lighter but specific obligations (Art. 26) including informing employees, carrying out fundamental-rights impact assessment (FRIA, Art. 27) where applicable, and operating within instructions.
- **Distributor / importer / authorised representative** — out of scope here.

Most teams you work with are *providers* of the models they fine-tune or host, and *deployers* of the foundation models they use downstream. One company is often both.

**General-purpose AI (GPAI) obligations (Chapter V)** are a distinct track with a different obligation set — transparency documentation, copyright policy, training-data summary, and for GPAI with systemic risk, model evaluation, systemic risk assessment, serious-incident reporting, and cybersecurity. Check-in with counsel on when a model crosses the GPAI line; the engineer's work is to prepare the technical evidence the GPAI obligations ask for.

## Article 10 — Data and Data Governance

Art. 10 is the **data-governance anchor** for an AI Privacy Engineer. It applies to high-risk AI systems trained with data (training, validation, test). The engineer's reading:

- **10(2) — quality criteria.** Training, validation, and test data sets are subject to data-governance and management practices appropriate for the intended purpose, including:
  - design choices,
  - data-collection processes and origin of data (for personal data, the original purpose of collection),
  - data-preparation operations such as annotation, labelling, cleaning, updating, enrichment, and aggregation,
  - formulation of assumptions, notably with respect to information that the data are supposed to measure and represent,
  - assessment of the availability, quantity, and suitability of the data,
  - examination for biases likely to affect health, safety, or fundamental rights or lead to discrimination prohibited by Union law,
  - appropriate measures to detect, prevent, and mitigate those biases,
  - identification of relevant data gaps or shortcomings and how they can be addressed.
- **10(3) — relevance / representativeness / errors / completeness.** Training, validation, and test data sets must be **relevant, sufficiently representative**, and to the best extent possible **free of errors and complete in view of the intended purpose**. They must have the appropriate statistical properties regarding the persons or groups of persons on which the system is intended to be used.
- **10(4) — contextual geographical / behavioural / functional considerations.** Data sets must take into account the characteristics / elements particular to the specific geographical, behavioural, contextual, or functional setting of use.
- **10(5) — special-category data for bias monitoring.** Providers may **exceptionally** process special categories of personal data (GDPR Art. 9) to the extent *strictly necessary* for detecting and correcting bias, subject to appropriate safeguards (pseudonymisation, technical limits on re-use, access controls, security measures). This is the AI Act's own explicit lawful basis for Art. 9 data use in bias testing.
- **10(6) — appropriate controls** for the data-governance operations.

### Engineer's control / test / evidence map

| Art. 10 clause | Engineering control | Test | Evidence artifact |
|---|---|---|---|
| 10(2) design / collection / preparation | Documented dataset-collection runbook; annotation pipeline with provenance; data-cleaning changelog | Golden-dataset regression at each cleaning release; annotation-agreement metric | Dataset-card (Datasheet-for-Datasets shape) per dataset, with every field of Art. 10(2) represented |
| 10(2) assumptions / measurement | Written assumption register ("this feature proxies for X because Y") | Assumption-vs-outcome analysis in evaluation | Assumption register attached to dataset-card |
| 10(2) bias examination | Bias-metric suite (demographic parity, equalised odds, calibration-by-group — scope with the eval owner) | CI-gated bias-eval delta against last release | Bias-examination report per version |
| 10(3) representativeness / errors / completeness | Sampling strategy doc; label-noise estimate; missing-value policy | Distributional-shift test vs. intended population | Representativeness statement; label-noise estimate; completeness audit |
| 10(4) geographical / behavioural / functional | Region / locale tag on data; deployment-context statement | Per-region eval slice | Region-coverage table |
| 10(5) special-category for bias | Separate access path; pseudonymisation; purpose-bound storage with TTL; deletion when bias tuning complete | Access-log audit; retention-job CI | Art. 10(5) justification memo citing the specific bias concern and the safeguards |
| 10(6) controls | Access control, change control, approval workflow | Change-review audit | Data-governance SOP document |

The **Datasheet-for-Datasets** structure (Gebru et al., 2018) maps cleanly onto Art. 10(2); you can write one document that serves both the Datasheet audience and the AI Act Annex IV data-governance section.

### Annex IV crossover

Annex IV of the Act lists the contents of technical documentation for high-risk AI systems. The data-governance section (Annex IV(2)(d)) mirrors Art. 10. **This track owns the engineering evidence** that goes into Annex IV; the audit-grade packaging of that evidence into regulator-facing documentation is `ai-evaluation-engineer` (level 35) scope.

## Article 15 — Accuracy, Robustness, and Cybersecurity

Art. 15 requires that high-risk AI systems be **designed and developed in such a way that they achieve an appropriate level of accuracy, robustness, and cybersecurity** and perform consistently throughout their lifecycle.

- **Accuracy.** Declared accuracy and the metrics used must be documented in the instructions for use (Art. 13) and the technical documentation (Annex IV).
- **Robustness.** Resilience against errors, faults, inconsistencies — including through redundancy, fail-safe plans, and the ability to perform as intended under adverse conditions. The Act flags **feedback loops** as a specific concern: models that influence their own future training data (e.g., content-recommendation systems) must be designed so biased outputs are mitigated by appropriate measures.
- **Cybersecurity.** Protection against attempts by unauthorised third parties to alter the system's use, outputs, or performance by exploiting system vulnerabilities. The Act specifically names **data poisoning** (contamination of training data), **model poisoning** (contamination of pre-trained components), **adversarial examples**, **confidentiality attacks** (which includes privacy attacks — membership inference, model extraction, model inversion), and **model flaws**.

### Where privacy engineering lives in Art. 15

Art. 15's **confidentiality attacks** clause is where the privacy engineer's work maps directly to AI Act compliance:

- **Membership inference / extraction / model inversion** are AI-Act-named cybersecurity threats. Your MIA / LiRA / extraction harness (`mod-105-memorisation-extraction-and-mia`) is the engineering evidence that you have "appropriate measures" against them.
- **Differential privacy** (`mod-106-differential-privacy-engineering`) is a measure that materially reduces the success of these attacks. A documented (ε, δ) budget is Art. 15 evidence.
- **Training-data poisoning** cross-touches privacy where the injected data is a privacy signal (PII poisoning of retrieval corpora, canary-smuggling). `mod-109-rag-privacy-and-agent-scoping` covers the RAG surface.

### Control / test / evidence map

| Art. 15 attack class | Privacy-engineering control | Test | Evidence |
|---|---|---|---|
| Membership inference | DP-SGD training; dedup; epoch-budget discipline | Shadow-model MIA + LiRA at AUC and TPR-at-fixed-FPR | MIA report + DP accountant log |
| LLM memorisation / verbatim extraction | DP; dedup; canary design | Carlini-shape extraction probe with canary FP control | Extraction probe report |
| Model inversion / attribute inference | Minimisation; DP; access controls | Fredrikson-style attribute-inference probe | Probe report |
| Model-extraction (query-based) | Rate limiting; response watermarking; output-perturbation | Probe at known attack budgets | Defence report (depth defers to `security-learning`) |
| Training-data poisoning | Data-provenance tracking; dataset diff review | Canary insertion; contamination probes | Provenance report |

## Article 72 — Post-market monitoring by providers

Art. 72 requires providers of high-risk AI systems to establish and document a **post-market monitoring system** proportionate to the nature and risks of the AI technology. The system:

- collects, documents, and analyses relevant data on the performance of the AI system *throughout its lifetime*, provided by deployers or collected through other sources;
- enables the provider to evaluate continuous compliance with the requirements of Chapter III, Section 2 (which includes Art. 10 and Art. 15);
- feeds a post-market monitoring **plan** that is part of the technical documentation (Annex IV).

Art. 73 adds the **serious-incident reporting** obligation — providers must report to the market-surveillance authorities within statutory windows.

### Why this is the privacy-monitoring interface

Privacy risks are not static. They drift because:

- Training data gets refreshed and the memorisation profile changes.
- Retrieval corpora get updated and per-tenant leakage risks shift.
- Deployers find new uses (and misuses) that re-cast lawful basis and necessity.
- Attackers get better (new MIA techniques, new extraction prompts).
- DSRs arrive and the deletion-propagation surface has to keep up.

Art. 72 is the hook you use to **justify ongoing privacy monitoring as a product requirement**, not a one-time DPIA.

### What the privacy slice of post-market monitoring looks like

Minimum signal set (this is the privacy engineer's wedge into the Art. 72 plan):

- **DSR volume + resolution-time** per sink (app DB, logs, retrieval, weights).
- **PII-pipeline F1 drift** against the labelled eval set; false-negative rate by category.
- **Retrieval-ACL negative-test pass rate** per release.
- **MIA / extraction probe** on a cadence — monthly / per-release / per-corpus-refresh.
- **Consent-propagation integrity** — revocation reaching every downstream sink within policy clock.
- **Privacy incident** — rate, severity, clock to GDPR Art. 33 / 34 notification where triggered.
- **Logging-policy compliance** — redaction-rate on prompt / completion sinks.

Package these into a monthly post-market-monitoring report. Use the same report to feed GDPR Art. 5(2) accountability evidence — one artifact, two audits.

### Control / test / evidence

| Art. 72 element | Control | Test | Evidence |
|---|---|---|---|
| Post-market monitoring plan | Written privacy-slice section of the plan | Doc review | Plan document in Annex IV bundle |
| Lifecycle performance data | Privacy telemetry pipeline → monitoring board | Dashboard uptime + metric freshness | Dashboard export + metric definitions |
| Continuous compliance w/ Art. 10 / 15 | Scheduled re-runs of MIA / extraction / bias eval | CI gates on drift thresholds | Scheduled-eval history |
| Serious-incident reporting (Art. 73) | Incident runbook; routing to legal + MSA | Tabletop exercise | Tabletop output; actual incidents log |

## How the three articles compose

Think of Art. 10 as "**what you put in**", Art. 15 as "**what comes out and how it holds up under attack**", and Art. 72 as "**how you keep proving it**". A high-risk AI system's privacy posture is:

1. **Data-governance evidence** (Art. 10 + GDPR Art. 5 / 9 / 25).
2. **Attack-resistance evidence** (Art. 15 cybersecurity clause + `mod-105` + `mod-106`).
3. **Ongoing-monitoring evidence** (Art. 72 + GDPR Art. 5(2) accountability + `mod-111`).

Your DPIA (GDPR Art. 35) and your Annex IV technical documentation (EU AI Act) share the same underlying evidence — same Datasheets, same MIA report, same DP accountant log — stitched into two different reading orders for two different audiences.

## What Art. 10 / 15 / 72 do *not* do

- They do not override GDPR — personal-data obligations stand independently (recital 10; AI Act is lex specialis on AI-system shape, GDPR remains lex specialis on personal data).
- They do not re-define "lawful basis". Art. 10(5) is a narrow carve-out for bias testing; everything else needs its own GDPR Art. 6 basis.
- They do not pre-empt sector law (HIPAA, FERPA, etc. — chapter 05).
- They do not apply to all AI — only to high-risk systems (and separately to GPAI with its own regime). Low-risk systems have transparency obligations (Art. 50) and otherwise rely on general obligations.

## Summary

- Art. 10 anchors AI-training data-governance obligations — relevance, representativeness, bias examination, and a special-category carve-out for bias testing.
- Art. 15 names the cybersecurity attack classes — including membership inference, extraction, inversion — that the privacy-engineering harness (`mod-105` / `mod-106`) exists to defend against.
- Art. 72 is the ongoing-monitoring hook. It is where you argue "privacy monitoring is part of the product, not an audit event".
- Your evidence is reusable across GDPR Art. 35, AI Act Annex IV, NIST PF, ISO 42001 — if you keep the shape clean.

Next: CCPA / CPRA + the CPPA ADMT rules + the US state matrix.
