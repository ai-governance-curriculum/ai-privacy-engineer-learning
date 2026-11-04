# The AI Privacy Engineer's Scope on the Governance Ladder

A hands-on engineering specialist role at level 30 only works when the scope boundaries above and below are **sharp**. This chapter is the engineer's reading of what *this role* owns, what it depends on from the layer below, what it shares horizontally with peers, and what it hands up. Every obligation in chapters 02–07 lands somewhere on this ladder; you need to know where.

## The ladder in this family

The AI Governance family (where this role lives):

- **L15 — AI Governance Analyst** — operational analyst work: AI-system inventory, ROPA upkeep, intake-questionnaire administration, privacy-notice first-drafts, framework crosswalks, jurisdictional regulatory tracking, control-tracking and evidence-collection cycles, analyst-tier incident participation.
- **L25 — AI Risk Engineer** — hands-on risk-engineering specialist (same family): harm modelling, LLM / agent red-team, adversarial ML, fairness / explainability, guardrail engineering, MRM, supply-chain. Its **mod-107 privacy-risk-engineering slice** delivers working-depth privacy-attack engineering (MIA, LiRA, DP-SGD, Presidio) — the baseline this track deepens.
- **L30 — AI Privacy Engineer (this role)** — hands-on privacy-engineering specialist. Owns the privacy-engineering craft end-to-end.
- **L35 — AI Evaluation Engineer** (peer / next-up, Governance family) — release-assurance, audit-trail methodology, regulator-facing packaging including EU AI Act Annex IV technical documentation.
- **L35 — Security / AI Infra Security** — deep ML / AI security at platform scale (model-extraction defence at edge, enclave attestation at fleet scale, KMS / HSM depth, MLSecOps).
- **L40 — Agentic Safety Engineer** — frontier-agent dangerous-capability evaluation depth beyond privacy.
- **L50 — Senior AI Governance Architect** — control-library architecture, policy taxonomy, cross-jurisdiction reconciliation.
- **L60 — Head of AI Governance** — program leadership, board-level reporting, regulator engagement.
- **L70 — Chief AI Officer** — executive scope.

Relevant peers outside the family (level 25–30):

- **L20 — ML Engineer** (ML Engineering family) — classical ML / PyTorch / sklearn / HF Transformers / MLflow / packaging fundamentals; prerequisite depth for this role.
- **L25 — AI Infra / MLOps** (Infra family) — MLOps automation, model registry, CI/CD-for-ML depth; the gate contract for privacy controls plugs here.
- **L25 — Training Pipeline Engineer** (ML Engineering family) — distributed-training platform engineering.
- **L30 — AI Eval Engineer** (AI Engineering family) — application-layer evaluation-engineering depth (traces, judges, RAG eval, eval-gated CI/CD).
- **L30 — Model Evaluation Engineer** (ML Engineering family) — statistical calibration methodology depth, confidence intervals on MIA / DP utility deltas.

Out-of-ladder:

- **Legal counsel / Data Protection Officer (DPO)** — legal opinion and DPO statutory duties.
- **Security Operations / DFIR** — SOC-side breach handling.

## What this role owns

The AI Privacy Engineer owns the engineering craft of privacy for AI / ML systems end-to-end. Concretely:

- **Privacy threat modelling for AI systems** — LINDDUN GO + MAESTRO + NIST PRAM over AI data flows (`mod-102`).
- **Data governance and minimisation** — Datasheets, consent / legal-basis propagation, data-privacy-vault + tokenisation, synthetic-data substitutes, CycloneDX ML-BOM (`mod-103`).
- **PII / PHI / PCI detection and redaction pipelines at production scale** — Presidio + benchmarks against Google DLP / AWS Comprehend PII / Azure AI Content Safety / Private AI; HIPAA Safe Harbor + Expert Determination; multimodal PII; tabular de-identification with ARX / sdcMicro (`mod-104`).
- **Memorisation / extraction / MIA risk engineering** — shadow-model MIA, Carlini LiRA, LLM extraction + canary methodology, Fredrikson model inversion, ML Privacy Meter as orchestration (`mod-105`).
- **Differential-privacy engineering for ML and LLM training** — DP-SGD, moments / RDP / zCDP / GDP accountants, PATE, DP synthetic data; Opacus, TensorFlow Privacy, OpenDP SmartNoise, Google DP; NIST SP 800-226 shape (`mod-106`).
- **Federated learning + secure aggregation + DP-FL** — Flower / TFF / FLARE / PySyft; cross-device vs. cross-silo; Bonawitz; DLG / iDLG defences (`mod-107`).
- **Confidential computing + privacy-preserving inference** — TEE (SGX / SEV-SNP / H100 CC / hyperscaler offerings), HE (TenSEAL / SEAL), MPC (CrypTen / MP-SPDZ), Apple PCC as published blueprint (`mod-108`).
- **RAG privacy + agent-tool data scoping** — per-tenant retrieval ACL, prompt / completion logging policy, agent-tool data-boundary review (`mod-109`).
- **Machine unlearning + DSR engineering for AI** — SISA exact / approximate / LLM-unlearning, TOFU benchmark; the right-of-access / rectification / erasure / portability / objection engineering surface (`mod-110`).
- **DPIA + audit-grade privacy evidence** — GDPR Art. 35, UK ICO shape, NIST PRAM, ISO/IEC 29134, EDPB Opinion 28/2024, CNIL AI sheets; regulator-facing engineering bundle (`mod-111`).
- **Cross-functional AI privacy program at team scope** — DPO / legal / security / product coordination, privacy slice of model / dataset cards, AI-privacy incident response (`mod-112`).

## What this role depends on (defers *down*)

### To `ai-governance-analyst` (L15)

This role **consumes** the analyst's operational privacy legwork. Specifically:

- **AI-system / data-processing register (ROPA)** — the analyst maintains it; you *read* it to drive risk assessment and DPIAs.
- **Intake-questionnaire administration** — the analyst runs intake; you receive triaged intakes to scope and threat-model.
- **Privacy-notice first-drafts** — the analyst drafts (legal signs off); you provide the technical data sources that make the notice accurate.
- **Framework crosswalks** — the analyst maintains the mapping tables (GDPR ↔ ISO 27701 ↔ NIST PF ↔ ISO 42001 ↔ CSA AI Controls Matrix); you *use* them to tag your controls once rather than per-audit.
- **Jurisdictional regulatory tracking** — the analyst tracks state-law changes, EDPB opinions, CNIL updates; you *consume* their briefing rather than do daily tracking.
- **Reading eval / audit evidence into governance records** — the analyst files your outputs into the program's authoritative records.

Hand-off contract: the analyst gives you *triaged, in-register, framework-tagged* work items. You give the analyst *finalised artifacts* (DPIAs, test reports, incident after-actions) in a form they can file into records.

### To `ml-engineer` (L20)

Classical ML / PyTorch / HF Transformers / MLflow / packaging fundamentals. This track does not re-teach — the prerequisites document explicitly assumes it.

### To `ai-infra-junior-engineer` (L10)

Engineering-craft prerequisites — Linux, Git, Python, packaging, HTTP APIs. Assumed.

## What peers own (deferral *sideways*)

### To `ai-risk-engineer` (L25, same family, peer)

Its **mod-107 privacy-risk-engineering slice** is the *working-depth* foundation this role deepens. The peer owns the **broader AI-risk-engineering surface** this role does not re-cover:

- Harm modelling beyond privacy.
- LLM / agent red-team (except privacy-specific red-team, which is in this track at mod-105 and mod-109).
- Adversarial ML beyond privacy (evasion, model extraction as *availability* / *IP* attack, poisoning as *integrity* attack — the privacy intersections live here, the broader surface lives there).
- Fairness / explainability engineering.
- Guardrail engineering.
- Model-risk management (MRM) framing.
- Supply-chain risk engineering (except the PII-provenance slice).
- Broader AI incident response (except the privacy-specific breach surface).

**Depth boundary:** your working-depth starting point for memorisation / MIA / Presidio is the peer's mod-107. Your production-grade engineering lives here. A learner who has not done the peer's mod-107 should not start this track.

### To `ai-infra-mlops` (L25, Infra family, peer)

MLOps automation, model registry, CI/CD-for-ML depth. Your privacy controls (DP gate, PII pipeline gate, DSR workflow) plug into the MLOps platform the peer owns. You own the **gate contract**; they own the **platform**.

### To `training-pipeline-engineer` (L25, ML Engineering family, peer)

Distributed-training platform engineering. Your DP-SGD runs on their platform; your federated-learning aggregator integrates with their orchestration.

### To peer AI Engineering tracks (`rag-engineer`, `llm-application-developer`, `applied-ai-engineer`, `fine-tuning-engineer`)

The *systems under privacy review*. You consume their architecture to threat-model. They consume your controls (retrieval-ACL contract, redaction-pipeline API, logging-policy specification) to implement.

### To `ai-eval-engineer` (L30, AI Engineering family, peer)

Application-layer evaluation-engineering depth (traces, judges, RAG eval, eval-gated CI/CD, online eval). You contribute **privacy eval content** into their eval suite. They own the eval-tooling platform.

### To `model-evaluation-engineer` (L30, ML Engineering family, peer)

Statistical / benchmark / judge-vs-human calibration methodology depth — confidence intervals on MIA AUC, DP utility delta, calibration curves. Your MIA report cites their methodology. They own the methodology depth; you own the AI-privacy-specific application.

## What next-up roles inherit or extend (deferral *up*)

### To `ai-evaluation-engineer` (L35, Governance family, peer / next-up)

Release-assurance / audit-trail methodology / regulator-facing packaging. This role's privacy-engineering evidence (DPIA artifacts, MIA reports, DP accountant logs, PII-pipeline F1 reports, unlearning verification) **feeds** the peer's release-assurance review and the EU AI Act Annex IV technical documentation. You produce the evidence; the peer packages it for the Data Protection Commissioner / Information Commissioner's Office / supervisory authority / Notified Body.

Hand-off contract: you deliver engineering evidence with full methodology, inputs, results, and limitations. The peer *does not re-run* your evaluations — they package them with a cover memo + context + residual-risk narrative.

### To `security-learning` / `ai-infra-security-learning` (L35)

Deep ML / AI security at platform scale. Specifically:

- Inference-edge model-extraction defence (beyond privacy-specific model-inversion).
- Enclave attestation at fleet scale (you deploy a TEE; they operate the attestation at platform scope).
- KMS / HSM integration depth.
- MLSecOps.
- SOC-side incident response.

You feed the AI-specific privacy signal into their SecOps pipeline (chapter 05, GLBA / HIPAA breach obligations).

### To `agentic-safety-engineer` (L40)

Frontier-agent dangerous-capability evaluation depth beyond privacy. The AI privacy engineer contributes the privacy-threat-modelling slice; the agentic safety engineer owns the broader frontier-agent safety surface.

### To `senior-ai-governance-architect` (L50)

Control-library architecture, policy taxonomy, cross-jurisdiction reconciliation (GDPR + CCPA + PIPL + state matrix + sector laws, harmonised in one control library). You contribute **implementations** of specific controls; the architect owns the **catalogue structure**.

### To `head-of-ai-governance` (L60) and `chief-ai-officer` (L70)

Program leadership, board-level reporting, regulator engagement. You produce the **technical slice** of the board report; they own the narrative.

## Out of scope

- **Legal opinion** — any statement that commits the organisation to a position on what a statute means. Routed to counsel. The engineer's artifact is the technical evidence; counsel owns the legal conclusion.
- **DPO statutory duties** — the DPO is a specific statutory role under GDPR Art. 37–39 with independence, reporting-line, and conflict-of-interest requirements. This role is *not* the DPO. You work with the DPO.
- **SOC-side breach handling** — digital forensics, host / network IR, SOC-triggered response. You feed AI-specific privacy signal to SecOps / Breach Response; they run the breach.

## The deferral contract — three worked handoffs

### Handoff 1 — a new state privacy law enacts

- **L15 — Analyst** — tracks the enactment, flags it to the team, authors the initial crosswalk delta against the existing state matrix, updates the regulatory tracker.
- **L30 — Privacy Engineer (you)** — reads the specific obligations that differ from the existing engineering surface (chapter 04 diff pattern), proposes pipeline deltas (a new rights endpoint, a new UOOM-handling path, a new sensitive-data opt-in flow), and estimates engineering effort.
- **L50 — Senior Architect** — reconciles the delta against the cross-jurisdiction control library, decides which existing controls absorb the new obligation vs. which need new controls.
- **L60 — Head of Governance** — approves the program impact, allocates budget, communicates to business lines.

### Handoff 2 — a high-risk AI system enters release-assurance

- **L15 — Analyst** — compiles ROPA entry, confirms the system is in scope for DPIA / AIA / AI Act.
- **L25 — Risk Engineer (peer)** — runs the broader risk-engineering review (harm model, red-team, MRM).
- **L30 — Privacy Engineer (you)** — authors the DPIA, runs the MIA / extraction harness, defends the DP release package, authors the privacy slice of the system card, delivers the engineering evidence bundle.
- **L35 — Evaluation Engineer (peer)** — packages your evidence into the release-assurance review and the EU AI Act Annex IV technical documentation.
- **L60 — Head of Governance** — signs off release.

### Handoff 3 — a privacy incident fires

- **Detection path** — SOC / observability / your own privacy-monitoring board surfaces a suspected incident.
- **L30 — Privacy Engineer (you)** — scopes the incident on the AI-system side (which sinks, which subjects, what was leaked, what the model can reconstruct), runs the technical investigation (log review, MIA re-check, extraction-probe re-run), hands the scoping memo to the DPO + legal + SecOps.
- **L15 — Analyst** — tracks notification-clock status against each applicable statute (GDPR Art. 33 72h, state breach laws, HIPAA, PIPEDA real-risk-of-significant-harm, LGPD reasonable-time, PIPL).
- **SecOps / DFIR (out-of-ladder)** — runs the host / network forensics.
- **DPO + legal** — own notification decisions and content.
- **L35 — Evaluation Engineer** — may package the after-action for regulator submission.
- **L60 — Head of Governance** — handles supervisory-authority communication.

You run the **AI-system technical slice**. You do not run the incident end-to-end.

## How this scope maps onto the module stack

| Module | Scope anchor — this role vs. adjacent |
|---|---|
| mod-101 (this module) | Positioning — what you own and what you don't |
| mod-102 Privacy threat modelling | Owned here. Peer `ai-risk-engineer` mod-107 handles working-depth; production-grade LINDDUN is here. |
| mod-103 Data governance | Owned here. Analyst (L15) maintains the artifacts; this role designs the engineering shape. |
| mod-104 PII pipelines | Owned here at production scale. Working-depth Presidio is in peer mod-107. |
| mod-105 Memorisation / MIA | Owned here at production scale. Working-depth MIA/LiRA is in peer mod-107. Statistical calibration depth defers to `model-evaluation-engineer` (L30). |
| mod-106 Differential privacy | Owned here. Working-depth DP-SGD is in peer mod-107. |
| mod-107 FL + secure aggregation | Owned here. (The peer's mod-107 is a *different* mod-107 — same number, different track.) |
| mod-108 Confidential computing + PPI | Owned here. Enclave-at-fleet-scale defers up to `security-learning` / `ai-infra-security` (L35). |
| mod-109 RAG + agent-scoping | Owned here. Application-layer eval depth defers to `ai-eval-engineer` (L30 peer). |
| mod-110 Machine unlearning + DSR | Owned here. |
| mod-111 DPIAs + privacy reviews | Owned here. Release-assurance packaging defers up to `ai-evaluation-engineer` (L35 next-up). |
| mod-112 Privacy program + comms | Owned here *at team scope*. Program leadership defers up to `head-of-ai-governance` (L60). Control-library architecture defers up to `senior-ai-governance-architect` (L50). |

## The one-paragraph answer

An AI Privacy Engineer owns the engineering craft of privacy for AI / ML systems end-to-end at **team scope** — threat modelling, data governance, PII pipelines, memorisation / MIA, DP, FL, confidential computing, RAG / agent-scoping, machine unlearning / DSR, DPIAs, and the privacy slice of the program. The role **consumes** analyst legwork from L15 and ML / engineering prerequisites from L10 / L20. It **shares** the field horizontally with the L25 risk-engineering peer (whose mod-107 is the foundation this track deepens) and with L25 / L30 peers in ML / AI / Eval Engineering. It **hands up** release-assurance packaging to L35 evaluation engineering, platform-scale security to L35 security, control-library architecture to L50, and program leadership to L60 / L70. It is **not** legal counsel, is **not** the DPO, and is **not** SOC.

## Summary — what you can do after this chapter

- Place every obligation in chapters 02–07 on a rung of the ladder.
- State the deferral contract for a new regulation, a release-assurance review, and a privacy incident.
- Resist scope creep upward (release-assurance packaging, program leadership) and downward (analyst legwork) — both erode the role's engineering depth.
- Handle the "what does this role actually do day-to-day" question with a crisp, framework-grounded answer in under two minutes.

The exercises that follow this chapter operationalise each of the seven learning objectives with real artifacts. Produce them in sequence.
