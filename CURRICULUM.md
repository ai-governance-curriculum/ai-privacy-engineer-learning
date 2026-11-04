# AI Privacy Engineer Curriculum

<!-- aicg:orientation:start -->
> **Start here:** [Module 00: Your Relationship to the Model](lessons/00-your-relationship-to-the-model/README.md). Where this role sits relative to the model, what it owns, and who it works with across the sibling curricula.
<!-- aicg:orientation:end -->

**Role level:** 30 (hands-on engineering specialist — AI Governance family)
**Family:** AI Governance
**Status:** planned — modules and projects below are the planned scope authored from [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json). Lessons and projects will be drafted by subsequent autonomous content cycles.

## Overview

This track teaches the *engineering craft* of privacy for AI/ML systems end-to-end for a hands-on specialist at level 30: reading privacy regulations and AI-specific obligations (GDPR, EU AI Act Art. 10 / 22 / 25 / 35, CCPA/CPRA + US state privacy-law matrix, HIPAA, FERPA, COPPA, GLBA, PIPL, PIPEDA, LGPD, NIST Privacy Framework, ISO/IEC 27701 / 29100 / 29134 / 27018, EDPB Opinion 28/2024, CNIL AI sheets, UK ICO AI + PETs guidance) as an engineer would → authoring LINDDUN-shape privacy threat models and NIST PRAM risk assessments grounded in AI-system data flows → engineering data governance and minimisation across training / retrieval / inference / logging paths (consent propagation, data-privacy-vault + tokenisation, synthetic-data substitutes, Datasheets, ML-BOM) → shipping production PII / PHI detection + redaction pipelines (Presidio, Google DLP, AWS Comprehend PII, Azure AI Content Safety, Private AI, k-anonymity / l-diversity / t-closeness toolkits, HIPAA Safe Harbor vs. Expert Determination) → engineering memorisation / extraction / membership-inference evaluations at production depth (Shokri MIA, Carlini LiRA, Carlini LLM extraction + canary methodology, Fredrikson model inversion, ML Privacy Meter) → engineering differential-privacy training and release (DP-SGD, moments / RDP / zCDP / GDP accountants, PATE, DP synthetic data) with Opacus, TensorFlow Privacy, Google DP, OpenDP SmartNoise, defending (ε, δ) against audit per NIST SP 800-226 → engineering federated learning + secure aggregation + DP-FL with Flower, TensorFlow Federated, NVIDIA FLARE, PySyft → engineering confidential computing and privacy-preserving inference across TEE (Intel SGX, AMD SEV-SNP, NVIDIA H100 CC, Azure / GCP / AWS Nitro offerings, Apple PCC as a published blueprint), homomorphic encryption (TenSEAL, SEAL), and secure MPC (CrypTen, MP-SPDZ) → engineering RAG privacy, per-tenant retrieval ACL, agent-tool data scoping, prompt / completion logging policy → engineering machine unlearning (SISA, approximate, LLM gradient-ascent, Who's-Harry-Potter, TOFU) and the right-of-access / erasure / rectification engineering surface → authoring audit-grade DPIAs (GDPR Art. 35, UK ICO shape, NIST PRAM, ISO/IEC 29134, EDPB Opinion 28/2024, CNIL AI sheets) and regulator-facing privacy-engineering bundles → owning a cross-functional AI privacy program at team scope with DPO / legal / security / product coordination, privacy slice of model / dataset cards, AI-privacy incident response, and credentials-portfolio positioning (IAPP CIPT / CIPP-E / CIPP-US / CIPM / AIGP, ISO/IEC 27701 LI/LA).

It is a **hands-on engineering specialist track at level 30**, positioned in the **AI Governance family**, as a *depth-on-top* of the peer `ai-risk-engineer-learning` (same family, level 25) whose mod-107 covers privacy-risk engineering at working depth. The differentiator is privacy-engineering *ownership*: this curriculum builds the LINDDUN-shape threat model, engineers the production PII pipeline, runs the MIA / extraction harness at audit quality, ships DP / FL / TEE / HE / MPC deployments, engineers machine unlearning, authors audit-grade DPIAs, and runs the privacy program end-to-end — contributing privacy-engineering evidence into the release-assurance interface that `ai-evaluation-engineer-learning` (peer / next-up, level 35) owns.

Total planned commitment: **176 hours across 12 modules** + **114 hours across 3 projects** = **~290 hours**.

## Ownership rule

Following the project-wide ownership rule, this curriculum:

- **Owns** the engineering craft of privacy for AI/ML systems end-to-end — privacy threat modelling (LINDDUN for AI, NIST PRAM), data governance and minimisation, PII / PHI / PCI detection and redaction at production scale, memorisation / extraction / MIA engineering, DP training and release, federated learning and secure aggregation, confidential computing and privacy-preserving inference, RAG privacy and agent-tool data scoping, machine unlearning and DSR engineering, DPIA authoring and regulator-facing packaging, and a cross-functional AI privacy program at team scope.
- **Defers down** to [`ai-governance-analyst-learning`](https://github.com/ai-governance-curriculum/ai-governance-analyst-learning) (level 15) for operational privacy legwork this role consumes — AI-system / data-processing-register (ROPA) upkeep, framework crosswalks, intake-questionnaire administration, privacy-notice first-drafts, jurisdictional regulatory tracking, and reading eval / audit evidence into governance records; to [`ml-engineer-learning`](https://github.com/ml-engineering-curriculum/ml-engineer-learning) (level 20) for classical ML / PyTorch / sklearn / HF Transformers / MLflow / packaging fundamentals; to [`ai-infra-junior-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-learning) (level 10) for engineering-craft prerequisites (Linux, Git, Python, packaging, HTTP APIs).
- **Defers sideways** to [`ai-risk-engineer-learning`](https://github.com/ai-governance-curriculum/ai-risk-engineer-learning) (peer, level 25, same family) for the broader AI risk-engineering surface (harm modelling, LLM / agent red-team, adversarial ML beyond privacy, fairness / explainability, guardrail engineering, MRM, supply-chain, incident response) — its mod-107 privacy-risk-engineering slice is the foundation this packet deepens; to [`ai-infra-mlops-learning`](https://github.com/ai-infra-curriculum/ai-infra-mlops-learning) (peer, level 25, Infra family) for MLOps automation, model registry, and CI/CD-for-ML depth; to [`training-pipeline-engineer-learning`](https://github.com/ml-engineering-curriculum/training-pipeline-engineer-learning) (peer, level 25, ML Engineering family) for distributed-training platform engineering; to peer AI Engineering tracks [`rag-engineer-learning`](https://github.com/ml-engineering-curriculum/rag-engineer-learning), [`llm-application-developer-learning`](https://github.com/ai-engineering-curriculum/llm-application-developer-learning), [`applied-ai-engineer-learning`](https://github.com/ai-engineering-curriculum/applied-ai-engineer-learning), and [`fine-tuning-engineer-learning`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning) for the systems being reviewed; to [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning) (peer, level 30, AI Engineering family) for application-layer evaluation-engineering depth; to [`model-evaluation-engineer-learning`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning) (peer, level 30, ML Engineering family) for statistical calibration methodology depth on MIA / DP confidence intervals.
- **Defers up** to [`ai-evaluation-engineer-learning`](https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning) (peer / next-up, level 35, Governance family) for release-assurance / audit-trail / regulator-facing methodology depth — this role contributes privacy-engineering evidence that peer packages into release-assurance reviews and DPC submissions; to [`security-learning`](https://github.com/ai-infra-curriculum/security-learning) and [`ai-infra-security-learning`](https://github.com/ai-infra-curriculum/ai-infra-security-learning) (level 35) for deep ML/AI security at platform scale (inference-edge model-extraction defence, enclave attestation at fleet scale, KMS / HSM integration); to [`agentic-safety-engineer-learning`](https://github.com/ai-governance-curriculum/agentic-safety-engineer-learning) (level 40) for frontier-agent dangerous-capability evaluation depth beyond privacy; to [`senior-ai-governance-architect-learning`](https://github.com/ai-governance-curriculum/senior-ai-governance-architect-learning) (level 50) for control-library architecture, policy taxonomy, and cross-jurisdiction reconciliation; to [`head-of-ai-governance-learning`](https://github.com/ai-governance-curriculum/head-of-ai-governance-learning) (level 60) and [`chief-ai-officer-learning`](https://github.com/ai-governance-curriculum/chief-ai-officer-learning) (level 70) for program leadership, board-level reporting, and regulator engagement.
- **Out of scope** to legal counsel / DPO (this role does not deliver legal opinion or carry DPO statutory duties) and to security operations / DFIR (SOC-side breach handling; this role feeds AI-specific privacy signal to SecOps / Breach Response).

See [`JOB_REQUIREMENTS.md`](JOB_REQUIREMENTS.md) for the requirements-to-coverage map and the cited public references the catalog is grounded in.

## Module Plan

| Module | Title | Hours | Status |
|---|---|---|---|
| mod-101-privacy-frameworks-and-position | Privacy Frameworks, Laws, and the Privacy-Engineer Position on the Governance Ladder | 12 | planned |
| mod-102-privacy-threat-modelling | Privacy Threat Modelling for AI Systems | 14 | planned |
| mod-103-data-governance-and-minimisation | Data Governance and Minimisation for AI (Training, Retrieval, Inference) | 14 | planned |
| mod-104-pii-detection-and-redaction | PII / PHI / PCI Detection, Classification, and Redaction Pipelines at Production Scale | 16 | planned |
| mod-105-memorisation-extraction-and-mia | Memorisation, Extraction, and Membership-Inference Risk Engineering | 18 | planned |
| mod-106-differential-privacy-engineering | Differential Privacy Engineering for ML and LLM Training | 18 | planned |
| mod-107-federated-learning-and-secure-aggregation | Federated Learning, Secure Aggregation, and On-Device Privacy Engineering | 14 | planned |
| mod-108-confidential-computing-and-ppi | Confidential Computing and Privacy-Preserving Inference (TEE, HE, MPC) | 14 | planned |
| mod-109-rag-privacy-and-agent-scoping | RAG Privacy, Retrieval Access Control, and Agent-Tool Data Scoping | 14 | planned |
| mod-110-machine-unlearning-and-dsr | Machine Unlearning and Data-Subject-Rights Engineering for AI | 14 | planned |
| mod-111-dpia-and-privacy-reviews | DPIAs, Privacy Reviews, and Audit-Grade Privacy Evidence | 14 | planned |
| mod-112-privacy-program-and-communication | Cross-Functional AI Privacy Program and Communication | 14 | planned |

## Project Plan

| Project | Title | Hours | Status |
|---|---|---|---|
| project-101-pii-pipeline-and-memorisation-eval-bundle | PII Pipeline + Memorisation Eval Bundle for One AI System | 30 | planned |
| project-102-dp-training-unlearning-and-dpia-package | DP-Trained Model + Unlearning Harness + DPIA for One Regulated Dataset | 40 | planned |
| project-103-end-to-end-ai-privacy-review | End-to-End AI Privacy Review for One Regulated Use Case | 44 | planned |

## Module summaries

### mod-101 — Privacy Frameworks, Laws, and the Privacy-Engineer Position
Read GDPR (Art. 5 / 6 / 9 / 17 / 22 / 25 / 32 / 35), EU AI Act (Art. 10 / 15 / 72), CCPA/CPRA with ADMT rules, and the 20+ US state privacy-act matrix as an engineer — mapping each obligation to concrete AI-system controls. Position sector laws (HIPAA, FERPA, COPPA, GLBA) and international regimes (PIPL, PIPEDA, LGPD, OECD Guidelines). Read NIST Privacy Framework, ISO/IEC 27701 (PIMS), ISO/IEC 29100 (privacy framework), EDPB Opinion 28/2024 on AI models, CNIL AI sheets, and UK ICO AI + PETs guidance. Diagram this role's exact scope on the level ladder — including the depth boundary against the peer `ai-risk-engineer` (level 25) mod-107 privacy-risk-engineering slice.

### mod-102 — Privacy Threat Modelling for AI Systems
Author a LINDDUN-shape privacy threat model (LINDDUN GO for workshop speed, LINDDUN MAESTRO for depth) over the AI system's data flows — training corpora, retrieval corpora, inference prompts, generated outputs, logs, embeddings, caches, telemetry. Overlay a NIST PRAM worksheet. Enumerate privacy-specific misuse personas and route each to a privacy control. Operationalise Privacy by Design (Cavoukian's seven principles, codified in GDPR Art. 25) as engineering checkpoints.

### mod-103 — Data Governance and Minimisation
Author Datasheet-for-Datasets slices for training and retrieval corpora. Engineer consent / legal-basis propagation through capture → training → retrieval → inference → logging with column / document-level provenance tags. Engineer data-minimisation controls (collection, processing, retention, output). Deploy a data-privacy-vault + tokenisation pattern (Skyflow-shape). Deploy synthetic-data substitutes (CTGAN, TabDDPM, Gretel, MOSTLY AI, Tonic, Synthea). Author a CycloneDX ML-BOM slice for data provenance.

### mod-104 — PII / PHI / PCI Detection and Redaction Pipelines
Ship a production Presidio pipeline (out-of-the-box + custom NER recognisers, anonymisation operators, labelled-eval scoring). Benchmark against Google DLP, AWS Comprehend PII, Azure AI Content Safety, Private AI with a build-vs-buy matrix. Extend to multimodal PII (face / voice / license-plate / screen-capture PHI / OCR'd PDF). Ship tabular de-identification with ARX (k-anonymity / l-diversity / t-closeness) and sdcMicro. Apply HIPAA Safe Harbor vs. Expert Determination. Integrate across ingest / retrieval / prompt / completion / output paths with FP / FN measurement.

### mod-105 — Memorisation, Extraction, and MIA Risk Engineering
Engineer shadow-model MIA (Shokri et al.) and Carlini LiRA at production quality with AUC + TPR-at-fixed-FPR reporting. Engineer attribute-inference and Fredrikson model-inversion. Engineer LLM memorisation / verbatim-extraction using Carlini methodology with canary design against cross-corpus false positives. Reason about PET-side defences (DP-SGD, dedup, epoch budget) and produce a defender's triage table. Operate ML Privacy Meter as the orchestration harness. Map to NIST AI 100-2, MITRE ATLAS, OWASP LLM02.

### mod-106 — Differential Privacy Engineering
Read the Dwork / Roth DP monograph and NIST SP 800-226. Ship DP-SGD training with Opacus and TensorFlow Privacy — noise multiplier, L2 clipping norm, lot sampling, Rényi DP accounting, (ε, δ) budget allocation. Defend (ε, δ) under audit against Apple / Google / US Census norms. Ship PATE as an alternative. Ship DP synthetic data with OpenDP SmartNoise. Deploy Google's PipelineDP for analytics. Produce an NIST SP 800-226-shape release package.

### mod-107 — Federated Learning and Secure Aggregation
Distinguish cross-device vs. cross-silo FL. Ship FedAvg / FedProx with Flower, TensorFlow Federated, NVIDIA FLARE, PySyft. Ship Bonawitz-style secure aggregation. Combine FL with DP (record / user / client-DP) and reason about composition across rounds. Ship defences against gradient-reconstruction attacks (DLG, iDLG). Reason about GDPR joint-controller / cross-border implications — leaving legal opinion to counsel.

### mod-108 — Confidential Computing and Privacy-Preserving Inference
Build the trust model for confidential AI — what TEEs defeat and what they do not. Deploy TEE-based LLM inference with remote attestation across Intel SGX, AMD SEV-SNP, NVIDIA H100 CC, and hyperscaler offerings (Azure Confidential AI, GCP Confidential VMs, AWS Nitro). Read Apple Private Cloud Compute as the published enterprise blueprint. Engineer homomorphic-encryption inference (TenSEAL, SEAL). Engineer secure MPC inference (CrypTen, MP-SPDZ). Author a confidential-AI risk-and-cost memo.

### mod-109 — RAG Privacy and Agent-Tool Data Scoping
Engineer per-tenant retrieval ACL with cross-tenant leakage negative-test harness. Engineer prompt / completion logging policy (retain, redact-before-log, access-control, erasure-honouring). Engineer agent-tool data scoping — per-tool permission scope, non-human identity patterns, OAuth-bound-to-user vs. OAuth-bound-to-tool, data-boundary review before each tool invocation. Defend against training-corpus poisoning that smuggles PII into retrieval. Author the privacy slice of a RAG / agent system card.

### mod-110 — Machine Unlearning and Data-Subject-Rights Engineering
Engineer the DSR surface for AI — right of access, rectification, erasure, data portability, objection — with per-right engineering runbooks. Ship SISA-style exact unlearning where the training cost-benefit warrants it. Ship approximate unlearning (influence functions, Ginart-style) for tabular / small-model cases. Ship LLM unlearning via gradient ascent (Jang), task-vector arithmetic, and Who's-Harry-Potter method; audit against TOFU benchmark. Build the erasure-vs-retraining decision tree. Ship DSR against RAG / agent systems. Package a DSR audit-trail that survives a supervisory-authority inspection.

### mod-111 — DPIAs and Audit-Grade Privacy Reviews
Author an audit-grade GDPR Art. 35 DPIA mapped to the UK ICO shape and ISO/IEC 29134. Operate the NIST PRAM worksheet end-to-end. Read EDPB Opinion 28/2024 and CNIL AI sheets as the European regulator stance; translate the four questions (model anonymity, lawful basis, DSR, legitimate-interest test) into engineering artifacts. Handle HIPAA-shape privacy / security / breach-notification where PHI is in scope, and GLBA / NIST SP 800-53 PT-family controls where financial / federal data is in scope. Package a regulator-facing privacy-engineering bundle. Cross-tag to ISO/IEC 42001 AIMS and CSA AI Controls Matrix.

### mod-112 — Cross-Functional Privacy Program and Communication
Own an AI privacy program at team scope — cadence, artifacts, decision-log templates, sign-off routing — across product / ML / infra / security / governance / legal / DPO. Coordinate with the DPO as statutory role. Author the privacy slice of model / system / dataset cards (Hugging Face format or internal template). Run a privacy review board with pre-reads, one-pagers, and audit-surviving minutes. Run AI-specific privacy-incident response with GDPR Art. 33 / 34 (72-hour), CCPA, and HIPAA breach-notification decisions. Cross-map evidence to NIST PF / ISO 27701 / ISO 42001 / CSA so one pack serves multiple audits. Position IAPP CIPT / CIPP-E / CIPP-US / CIPM / AIGP and ISO 27701 LI/LA credentials.

## Assessment

Each module ships **1 quiz** plus the exercises and a lab listed in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json). Each project ships a portfolio-grade README, a reproducibility bundle (privacy-eval configs, DP accountant logs, FL round configs, TEE attestation evidence, dataset / model / prompt / decoding / seed hashes, redaction-pipeline configs, logging policy, LINDDUN / PRAM worksheets, unlearning verification artifacts), and an explicit rubric covering LINDDUN-model completeness, data-governance defensibility, PII-pipeline FP / FN discipline, MIA / extraction report quality, DP release-package audit shape, FL / TEE / HE / MPC trust-model clarity, RAG / agent-scope test coverage, unlearning verification, DPIA audit-grade shape, and cross-audience communication.

## Where to go after this curriculum

- **`ai-evaluation-engineer-learning`** (peer / next-up, level 35, Governance family) — release-assurance / audit-trail / regulator-facing methodology; consumes this curriculum's privacy-engineering evidence.
- **`senior-ai-governance-architect-learning`** (level 50) — architectural scope on the Governance ladder; owns control-library architecture, policy taxonomy, cross-jurisdiction reconciliation.
- **`head-of-ai-governance-learning`** (level 60) — leadership scope; owns program leadership, board reporting, regulator engagement.
- **`chief-ai-officer-learning`** (level 70) — executive scope.
- **`ai-infra-security-learning`** / **`security-learning`** (level 35) — deep ML/AI security at platform scale.
- **`agentic-safety-engineer-learning`** (level 40) — frontier-agent dangerous-capability evaluation depth beyond privacy.
- **`ai-risk-engineer-learning`** (peer, level 25, same family) — natural cross-training for the broader AI-risk-engineering surface.
- **`ai-eval-engineer-learning`** (peer, level 30, AI Engineering family) — natural cross-training for the application-layer evaluation-engineering peer.

<!-- needs-research: backfill industry-frequency evidence into JOB_REQUIREMENTS.md once the autonomous research loop runs with WebSearch / WebFetch exercised against live job boards; demote any module or exercise whose underlying requirement does not show up in ≥3 in-window postings. -->
