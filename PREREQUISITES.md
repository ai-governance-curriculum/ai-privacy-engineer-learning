# Prerequisites — AI Privacy Engineer track

**Role level:** 30 (hands-on engineering specialist, AI Governance family)
**Track:** `ai-privacy-engineer-learning`

This track is a hands-on engineering specialty at level 30. It assumes the learner arrives with the operational privacy legwork already fluent, ML fundamentals in place, and the peer `ai-risk-engineer` (level 25) mod-107 privacy-risk-engineering slice at working depth. Fundamentals are **not** re-taught here — the modules step directly onto LINDDUN threat modelling, production PII pipelines, memorisation / MIA audit harnesses, DP engineering, FL / TEE / HE / MPC engineering, machine unlearning, and DPIA authoring.

## Assumed lower-level curriculum

Complete or be practically fluent in the equivalent of these lower-level tracks before starting:

- **[`ai-infra-junior-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-learning) — level 10** — engineering-craft prerequisites (Linux, Git, Python packaging, HTTP APIs, unit testing, logging, docker basics). If you cannot ship a small Python service and instrument it, start here.
- **[`ml-engineer-learning`](https://github.com/ml-engineering-curriculum/ml-engineer-learning) — level 20** — classical ML with scikit-learn, deep learning with PyTorch and Hugging Face Transformers, evaluation with standard sklearn / HF metrics, packaging with FastAPI + Docker, experiment tracking with MLflow / Weights & Biases. If you cannot ship a fine-tuned classifier and defend the eval, start here.
- **[`ai-governance-analyst-learning`](https://github.com/ai-governance-curriculum/ai-governance-analyst-learning) — level 15** — operational analyst work in the AI Governance family: AI use-case intake and triage; model / AI-system inventory maintenance; **ROPA (Record of Processing Activities)** upkeep; privacy-notice first-draft authoring; NIST AI RMF, ISO/IEC 42001, GDPR, and EU AI Act framework crosswalks; first-draft privacy and AI impact assessments; jurisdictional regulatory tracking (including state privacy laws, EDPB opinions, CNIL guidance); reading eval / red-team / audit evidence into governance records; control-tracking and evidence-collection cycles; participation in AI incidents at the analyst tier; council-support communication. This role *consumes* those artifacts and adds privacy-engineering depth on top.
- **[`ai-risk-engineer-learning`](https://github.com/ai-governance-curriculum/ai-risk-engineer-learning) — level 25** — hands-on engineering specialist in the same AI Governance family. Its **mod-107 privacy-risk-engineering slice** (Shokri MIA, Carlini LiRA, Opacus DP-SGD at working depth, Presidio PII pipelines, GDPR Art. 22 + NIST Privacy Framework mapping) is the foundation this packet deepens. You should arrive with that slice at *working* depth — comfortable running an MIA probe and a Presidio pipeline — so this packet can skip to production-grade methodology, PET families you have not seen yet (FL, TEE, HE, MPC), machine unlearning, and DPIA audit shape.

## Assumed working knowledge

Even without holding the equivalent roles formally, you should be practically fluent in:

- **Python** — comfortable with packaging (pyproject.toml, uv / poetry, virtualenvs), unit testing (pytest, hypothesis), typed code (mypy / pyright), and CI (GitHub Actions).
- **Statistics & probability** — reading a confusion matrix, calibration curves, ROC / PR curves, bootstrap confidence intervals; comfortable enough with hypothesis-testing that a p-value / effect-size discussion is not new. Deep statistical methodology depth (Bayesian priors, sequential testing, judge-vs-human calibration) is linked out to `model-evaluation-engineer` (peer, level 30).
- **PyTorch + Hugging Face Transformers** — enough to fine-tune a classifier, load a Hugging Face model, run inference, and instrument a model with Opacus / ML Privacy Meter / adversarial-attack code.
- **Linear algebra + probability for cryptography** — enough to read Rényi DP / zCDP / GDP accountant derivations at the applied level (not the measure-theoretic depth). You do not need to be a cryptographer, but you should be comfortable reading CKKS / BFV HE scheme parameters and Shamir secret-sharing descriptions.
- **Data engineering basics** — pandas, SQL, columnar formats (Parquet / Arrow), dataset lineage. You should be able to author a labelled PII evaluation dataset and defend the sampling.
- **HTTP APIs + LLM providers** — comfortable calling OpenAI / Anthropic / Google / open-source (vLLM / TGI) inference APIs from Python; comfortable reading trace output from Arize Phoenix, Langfuse, or LangSmith at the analyst level.
- **Basic security literacy** — HTTPS / auth basics, secrets handling, KMS basics, OWASP Top 10 for web at general-awareness level. Deep application security is not required — this track adds AI-privacy-specific depth on top of that baseline.
- **Reading the primary literature** — you should be able to open a paper on arXiv (e.g., Abadi et al. DP-SGD, McMahan et al. FedAvg, Carlini et al. extraction) and translate the method into runnable code within a working day.

## Not assumed, taught here

You do **not** need prior experience with:

- Any specific privacy regulation — GDPR, EU AI Act Art. 10 / 22 / 25 / 35, CCPA/CPRA with ADMT, HIPAA, FERPA, COPPA, GLBA, PIPL, PIPEDA, LGPD. Mod-101 covers the mapping to engineering artifacts.
- Any specific privacy framework or standard — NIST Privacy Framework, NIST PRAM, ISO/IEC 27701, ISO/IEC 29100, ISO/IEC 29134, ISO/IEC 27018, ISO/IEC 42001, Cloud Security Alliance AI Controls Matrix. Mod-101 and mod-111 and mod-112 cover the working set.
- Any specific regulator guidance — EDPB Opinion 28/2024 on AI models, UK ICO AI + PETs guidance, CNIL AI sheets. Mod-101 and mod-111 cover the working set.
- LINDDUN privacy threat modelling (GO or MAESTRO) — mod-102 covers both end-to-end.
- Any specific PII / de-identification toolkit — Presidio, Google DLP, AWS Comprehend PII, Azure AI Content Safety, Private AI, Skyflow, ARX, sdcMicro, synthetic-data stacks (Gretel, MOSTLY AI, Tonic, Synthea). Mod-103 and mod-104 cover the working suite.
- Any specific privacy-attack method — Shokri MIA at production methodology, Carlini LiRA, Carlini LLM extraction + canary methodology, Fredrikson model inversion, attribute inference. Mod-105 covers production-grade engineering (building on the `ai-risk-engineer` mod-107 working-depth introduction).
- Any specific DP library — Opacus, TensorFlow Privacy, Google Differential Privacy (PipelineDP), OpenDP SmartNoise. Mod-106 covers production-grade (ε, δ) defence, accountant choice, PATE, and DP synthetic data.
- Any specific FL library — Flower, TensorFlow Federated, NVIDIA FLARE, PySyft; secure-aggregation (Bonawitz) implementation; DP-FL composition. Mod-107 covers the working suite.
- Any specific confidential-computing or cryptographic-inference stack — Intel SGX, AMD SEV-SNP, NVIDIA H100 CC, Azure Confidential AI, GCP Confidential VMs, AWS Nitro Enclaves, Apple PCC blueprint, TenSEAL, Microsoft SEAL, CrypTen, MP-SPDZ. Mod-108 covers the trust model and working deployment.
- Machine unlearning methodology — SISA, approximate unlearning, LLM unlearning (gradient ascent, Who's-Harry-Potter), TOFU benchmark. Mod-110 covers the working suite.
- DPIA authoring at audit-grade — GDPR Art. 35, UK ICO shape, NIST PRAM worksheets, ISO/IEC 29134. Mod-111 covers the full shape.

## Recommended reading before starting

- [GDPR (EU 2016/679)](https://eur-lex.europa.eu/eli/reg/2016/679/oj) — read Articles 5, 6, 9, 17, 22, 25, 32, 35 once front-to-back.
- [EDPB Opinion 28/2024 on AI models](https://edpb.europa.eu/our-work-tools/general-guidance/guidelines-recommendations-best-practices_en) — the European regulator stance on AI + GDPR, read once end-to-end.
- [UK ICO Guidance on AI and Data Protection](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/) — read the "How should we assess the risks to rights and freedoms?" section carefully.
- [NIST Privacy Framework](https://www.nist.gov/privacy-framework) — read the Core (IDENTIFY-P / GOVERN-P / CONTROL-P / COMMUNICATE-P / PROTECT-P) once.
- [NIST SP 800-226](https://csrc.nist.gov/pubs/sp/800/226/final) — read the "Evaluating a DP Claim" checklist once.
- [LINDDUN framework overview](https://linddun.org/) — read the threat-category table and one worked example.
- [Apple Private Cloud Compute Security Design](https://security.apple.com/documentation/private-cloud-compute) — the published enterprise blueprint for confidential AI inference.
- [Carlini et al. Extracting Training Data from LLMs (arXiv:2012.07805)](https://arxiv.org/abs/2012.07805) — the foundational paper for LLM memorisation.

## Not required, useful to have

- Prior work with an eval-tooling platform (Arize Phoenix, Langfuse, W&B Weave, Braintrust, Promptfoo, DeepEval, RAGAS, LangSmith) — the peer `ai-eval-engineer-learning` (level 30) owns this stack; a working feel is useful before mod-109 and mod-111.
- Prior work with an experiment-tracking platform (MLflow, W&B) — assumed at the `ml-engineer` level.
- Familiarity with a GRC-for-AI or privacy-GRC platform (Credo AI, Holistic AI, Private.ai, OneTrust, Securiti) — mod-111 and mod-112 walk the platforms at the privacy-engineer level.
- Prior work with a KMS / HSM (AWS KMS, GCP KMS, Azure Key Vault, HashiCorp Vault) — mod-108 benefits from this but does not require it; deep KMS / HSM depth defers to `ai-infra-security` (level 35).
- Certifications: the **IAPP CIPT (Certified Information Privacy Technologist)** is the most-cited credential in AI-privacy-engineer postings; **IAPP AIGP** and **IAPP CIPP/E** or **CIPP/US** are frequently listed. Earning one before starting is not required but is treated as a strong signal.

## Not in scope for this track (linked out)

- **Operational privacy-analyst legwork** — ROPA upkeep, intake-questionnaire administration, privacy-notice first-draft authoring, jurisdictional regulatory tracking — owned by [`ai-governance-analyst-learning`](https://github.com/ai-governance-curriculum/ai-governance-analyst-learning) (level 15).
- **Broader AI risk-engineering surface** — harm modelling, LLM / agent red-team, adversarial ML beyond privacy, fairness / explainability, guardrail engineering, MRM, supply-chain, incident response — owned by [`ai-risk-engineer-learning`](https://github.com/ai-governance-curriculum/ai-risk-engineer-learning) (peer, level 25, same family). Its mod-107 privacy-risk-engineering slice is the foundation this track deepens.
- **Application-layer LLM / agent evaluation-engineering depth** — traces, judges, RAG eval, eval-gated CI/CD, online eval, cost / latency / quality — owned by [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning) (peer, level 30, AI Engineering family).
- **Statistical / benchmark / judge-vs-human calibration methodology depth** — owned by [`model-evaluation-engineer-learning`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning) (peer, level 30, ML Engineering family).
- **MLOps automation depth** — CI/CD-for-ML, model registry, deployment gates — owned by [`ai-infra-mlops-learning`](https://github.com/ai-infra-curriculum/ai-infra-mlops-learning) (peer, level 25, Infra family). This track owns the privacy gate contract; MLOps platform depth is linked out.
- **Deep ML / AI security at platform scale** — inference-edge adversarial defence, model-extraction defence, judge supply-chain, eval-set exfiltration prevention, enclave attestation at fleet scale, KMS / HSM integration, MLSecOps — owned by [`security-learning`](https://github.com/ai-infra-curriculum/security-learning) and [`ai-infra-security-learning`](https://github.com/ai-infra-curriculum/ai-infra-security-learning) (level 35).
- **Release-assurance / audit-trail / regulator-facing methodology depth** — DPC / ICO / supervisory-authority submissions, EU AI Act Annex IV technical documentation — owned by [`ai-evaluation-engineer-learning`](https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning) (peer / next-up, level 35, Governance family).
- **Frontier-agent red-team methodology and dangerous-capability evaluation depth beyond privacy** — owned by [`agentic-safety-engineer-learning`](https://github.com/ai-governance-curriculum/agentic-safety-engineer-learning) (level 40).
- **Control-library architecture, policy taxonomy, cross-jurisdiction reconciliation (GDPR + CCPA + PIPL + state-law + sector-law harmonisation)** — owned by [`senior-ai-governance-architect-learning`](https://github.com/ai-governance-curriculum/senior-ai-governance-architect-learning) (level 50).
- **Program leadership, board-level reporting, regulator engagement** — owned by [`head-of-ai-governance-learning`](https://github.com/ai-governance-curriculum/head-of-ai-governance-learning) (level 60) and [`chief-ai-officer-learning`](https://github.com/ai-governance-curriculum/chief-ai-officer-learning) (level 70).
- **Legal opinion and DPO statutory duties** — out of scope; routed to legal counsel and the organisation's Data Protection Officer.
- **SOC-side breach handling** — out of scope; this role feeds AI-specific privacy signal to SecOps / Breach Response.
