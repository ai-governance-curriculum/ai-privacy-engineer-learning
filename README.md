# AI Governance · AI Privacy Engineer — Learning Repository

<!-- aicg:site-banner -->
> 🎓 Part of the free, open-source **AI Career Curriculum** ecosystem — [Infrastructure](https://github.com/ai-infra-curriculum) · [ML Engineering](https://github.com/ml-engineering-curriculum) · [AI Engineering](https://github.com/ai-engineering-curriculum) · [Governance](https://github.com/ai-governance-curriculum) · [Startup](https://github.com/ai-startup-curriculum) · [Workforce](https://github.com/ai-workforce-curriculum).
<!-- /aicg:site-banner -->

<!-- aicg:sponsor -->
> 💜 **[Sponsor this curriculum](https://github.com/sponsors/ai-governance-curriculum)** — sponsorships keep the whole open-source AI Career Curriculum free and moving.
<!-- /aicg:sponsor -->

**Role level:** 30 (hands-on engineering specialist — AI Governance family)

Engineers privacy into AI systems end-to-end: LINDDUN-shape threat models, data governance and minimisation across training / retrieval / inference / logging, production PII / PHI detection and redaction pipelines, memorisation / extraction / membership-inference audit harnesses, differential-privacy training and release, federated learning and secure aggregation, confidential-computing and homomorphic-encryption / MPC inference, RAG and agent-tool data scoping, machine unlearning and data-subject-rights engineering, audit-grade DPIAs mapped to GDPR / UK ICO / NIST PRAM / ISO 29134, and a cross-functional AI privacy program at team scope.

> **Status**: planned. This bootstrap packet lands the requirements catalog and the module / project plan authored from [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json). Lessons and projects will be drafted by subsequent autonomous content cycles. Live posting evidence was deferred (WebSearch / WebFetch was not granted in the bootstrap session) — see [`JOB_REQUIREMENTS.md`](JOB_REQUIREMENTS.md) `Status`.

## What you get

Planned commitment: **176 hours across 12 modules** + **114 hours across 3 projects** = **~290 hours**.

- **Module 101** — privacy regulations and AI-specific obligations (GDPR, EU AI Act Art. 10 / 15 / 72, CCPA/CPRA ADMT, US state privacy-law matrix, HIPAA, FERPA, COPPA, GLBA, PIPL, PIPEDA, LGPD, NIST Privacy Framework, ISO 27701 / 29100 / 29134 / 27018, EDPB Opinion 28/2024, CNIL AI sheets, UK ICO AI + PETs guidance) read as an engineer would.
- **Module 102** — LINDDUN-shape privacy threat modelling (GO + MAESTRO) with a NIST PRAM overlay and Privacy by Design engineering checkpoints.
- **Module 103** — data governance and minimisation for AI; Datasheets, consent / legal-basis propagation, data-privacy-vault + tokenisation, synthetic-data substitutes, ML-BOM.
- **Module 104** — production PII / PHI / PCI detection and redaction (Presidio, Google DLP, AWS Comprehend PII, Azure AI Content Safety, Private AI; multimodal; k-anonymity / l-diversity / t-closeness; HIPAA Safe Harbor vs. Expert Determination).
- **Module 105** — production-grade memorisation / extraction / MIA engineering (Shokri MIA, Carlini LiRA, Carlini extraction + canary methodology, Fredrikson inversion) with ML Privacy Meter.
- **Module 106** — differential-privacy engineering (DP-SGD with Opacus + TensorFlow Privacy, Rényi / zCDP / GDP accountants, PATE, DP synthetic data with OpenDP SmartNoise, Google DP / PipelineDP), defending (ε, δ) per NIST SP 800-226.
- **Module 107** — federated learning and secure aggregation (Flower, TensorFlow Federated, NVIDIA FLARE, PySyft, Bonawitz SecAgg, DP-FL).
- **Module 108** — confidential computing and privacy-preserving inference (Intel SGX, AMD SEV-SNP, NVIDIA H100 CC, Azure / GCP / AWS Nitro offerings, Apple Private Cloud Compute as a blueprint, TenSEAL + Microsoft SEAL, CrypTen + MP-SPDZ).
- **Module 109** — RAG privacy, per-tenant retrieval ACL, agent-tool data scoping, prompt / completion logging policy.
- **Module 110** — machine unlearning (SISA, approximate, LLM gradient-ascent, Who's-Harry-Potter, TOFU benchmark) and the right-of-access / erasure / rectification engineering surface.
- **Module 111** — audit-grade DPIA authoring and regulator-facing privacy-engineering bundles.
- **Module 112** — cross-functional AI privacy program, DPO partnership, privacy slice of model / dataset cards, AI-privacy incident response, NIST PF / ISO 27701 / ISO 42001 / CSA crosswalk, credentials-portfolio positioning.
- **Three projects** — a PII pipeline + memorisation eval bundle for one AI system; a DP-trained model + unlearning harness + DPIA package for one regulated dataset; an end-to-end AI privacy review for one regulated use case (EU healthcare RAG, US fintech decisioning LLM, EU-US cross-border customer-service agent, or edu-tech tutor under FERPA + COPPA).

## Level ladder — where this track fits

- **level 10** — [`ai-infra-junior-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-learning) (assumed engineering-craft prerequisites)
- **level 15** — [`ai-governance-analyst-learning`](https://github.com/ai-governance-curriculum/ai-governance-analyst-learning) (operational analyst work including ROPA; assumed familiarity)
- **level 20** — [`ml-engineer-learning`](https://github.com/ml-engineering-curriculum/ml-engineer-learning) (ML fundamentals; assumed)
- **level 25 (peer / foundation)** — [`ai-risk-engineer-learning`](https://github.com/ai-governance-curriculum/ai-risk-engineer-learning) (broader AI risk-engineering surface; its mod-107 privacy-risk-engineering slice is the foundation this packet deepens)
- **level 30** — **`ai-privacy-engineer-learning`** (this track) — depth-on-top privacy engineering for AI
- **level 30 (peers)** — [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning), [`model-evaluation-engineer-learning`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning), [`fine-tuning-engineer-learning`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning)
- **level 35** — [`ai-evaluation-engineer-learning`](https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning) (release-assurance methodology), [`security-learning`](https://github.com/ai-infra-curriculum/security-learning) / [`ai-infra-security-learning`](https://github.com/ai-infra-curriculum/ai-infra-security-learning) (deep ML/AI security)
- **level 40** — [`agentic-safety-engineer-learning`](https://github.com/ai-governance-curriculum/agentic-safety-engineer-learning) (frontier-agent dangerous-capability depth)
- **level 50** — [`senior-ai-governance-architect-learning`](https://github.com/ai-governance-curriculum/senior-ai-governance-architect-learning) (architecture)
- **level 60** — [`head-of-ai-governance-learning`](https://github.com/ai-governance-curriculum/head-of-ai-governance-learning) (program leadership)
- **level 70** — [`chief-ai-officer-learning`](https://github.com/ai-governance-curriculum/chief-ai-officer-learning) (executive)

See [`CURRICULUM.md`](CURRICULUM.md) for the ownership rule this track applies and [`PREREQUISITES.md`](PREREQUISITES.md) for the assumed lower-level curriculum.

## Layout

```
ai-privacy-engineer-learning/
├── lessons/mod-XXX-*/        modules with lectures, exercises, labs, quizzes
├── projects/project-XXX-*/   multi-module capstones
├── CURRICULUM.md             role-level coverage map
├── PREREQUISITES.md          assumed entry skills
├── VERSIONS.md               release history
├── JOB_REQUIREMENTS.md       requirements catalog with cited authoritative references
├── .aicg/job-requirements.json    machine-readable requirements & references
├── .aicg/curriculum-plan.json     machine-readable module / exercise / project plan
└── README.md                 this file
```

## Paired Solutions Repo

[`ai-privacy-engineer-solutions`](https://github.com/ai-governance-curriculum/ai-privacy-engineer-solutions) carries the reference implementations.

---

<!-- aicg:maintained-by -->
Maintained by [VeriSwarm.ai](https://veriswarm.ai)
