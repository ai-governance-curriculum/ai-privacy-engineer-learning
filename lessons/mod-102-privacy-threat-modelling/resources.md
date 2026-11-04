# Resources for mod-102-privacy-threat-modelling (Privacy Threat Modelling for AI Systems)

Primary sources first. Everything else is second-order reading.

## LINDDUN methodology

- **LINDDUN project home** — methodology documentation, card decks (LINDDUN GO), and tooling. [linddun.org](https://linddun.org/)
- **LINDDUN GO card deck** — the workshop-speed assessment deck. Downloadable from the LINDDUN site.
- **Deng, Wuyts, Scandariato, Preneel, Joosen — "A privacy threat analysis framework: supporting the elicitation and fulfillment of privacy requirements"** (Requirements Engineering journal, 2011) — the foundational LINDDUN paper. [doi.org/10.1007/s00766-010-0115-7](https://doi.org/10.1007/s00766-010-0115-7)
- **Wuyts, Joosen — "LINDDUN privacy threat tree catalog"** — the published threat-tree catalogue underpinning the methodology. Available via the LINDDUN site.
- **DistriNet research group at KU Leuven** — the host group for LINDDUN research. [distrinet.cs.kuleuven.be](https://distrinet.cs.kuleuven.be/)

<!-- needs-research: confirm whether the canonical current detailed-depth LINDDUN variant is published as "LINDDUN PRO", "LINDDUN MAESTRO", or another name; align terminology across chapters 05, resources.md, and the exercises before external publication -->

## NIST privacy risk assessment

- **NIST Privacy Framework** (current version). [nist.gov/privacyframework](https://www.nist.gov/privacyframework)
- **NIST Privacy Risk Assessment Methodology (PRAM) worksheets.** [nist.gov/itl/applied-cybersecurity/privacy-engineering/collaboration-space/focus-areas/risk-assessment/resources](https://www.nist.gov/itl/applied-cybersecurity/privacy-engineering/collaboration-space/focus-areas/risk-assessment/resources)
- **NIST IR 8062 — An Introduction to Privacy Engineering and Risk Management in Federal Systems.** Foundational concepts: *data actions*, *problematic data actions*, *privacy risks*. [doi.org/10.6028/NIST.IR.8062](https://doi.org/10.6028/NIST.IR.8062)
- **NIST SP 800-30 Rev. 1 — Guide for Conducting Risk Assessments.** The likelihood × impact shape PRAM inherits. [doi.org/10.6028/NIST.SP.800-30r1](https://doi.org/10.6028/NIST.SP.800-30r1)
- **NIST AI Risk Management Framework (AI RMF 1.0)** and the companion **Generative AI Profile (NIST AI 600-1)** — adjacent risk-management framing for AI systems. [nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework)

## Privacy by Design and GDPR Art. 25

- **Cavoukian, A. — Privacy by Design: The 7 Foundational Principles.** [iapp.org/resources/article/privacy-by-design-the-7-foundational-principles](https://iapp.org/resources/article/privacy-by-design-the-7-foundational-principles/)
- **GDPR Art. 25 (data protection by design and by default)** in the consolidated text. [eur-lex.europa.eu/eli/reg/2016/679/oj](https://eur-lex.europa.eu/eli/reg/2016/679/oj)
- **EDPB Guidelines 4/2019 on Article 25 Data Protection by Design and by Default.** [edpb.europa.eu](https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-42019-article-25-data-protection-design-and_en)

## DPIA and AI-specific regulator guidance

- **GDPR Art. 35 — Data protection impact assessment.** In the consolidated text (see above).
- **ICO (UK) — "Data protection impact assessments" guidance.** [ico.org.uk](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/accountability-and-governance/data-protection-impact-assessments-dpias/)
- **CNIL — PIA methodology and software.** [cnil.fr/en/privacy-impact-assessment-pia](https://www.cnil.fr/en/privacy-impact-assessment-pia)
- **CNIL — AI how-to sheets ("fiches pratiques IA").** [cnil.fr/en/ai-how-sheets](https://www.cnil.fr/en/ai-how-sheets)
- **EDPB Opinion 28/2024 on certain data protection aspects related to the processing of personal data in the context of AI models.** [edpb.europa.eu](https://www.edpb.europa.eu/our-work-tools/our-documents/opinion-board-art-64/opinion-282024-certain-data-protection-aspects_en)
- **ISO/IEC 29134:2017 — Information technology — Security techniques — Guidelines for privacy impact assessment.** [iso.org/standard/62289.html](https://www.iso.org/standard/62289.html)

## Standards relevant to the engineering interface

- **ISO/IEC 27701:2019** — PIMS (Privacy Information Management System) extension to ISO/IEC 27001. [iso.org/standard/71670.html](https://www.iso.org/standard/71670.html)
- **ISO/IEC 29100:2024** — Information technology — Security techniques — Privacy framework.
- **ISO/IEC 42001:2023** — AI management systems. [iso.org/standard/81230.html](https://www.iso.org/standard/81230.html)

## STRIDE and related security-side threat modelling

- **Microsoft SDL Threat Modeling.** [microsoft.com/en-us/securityengineering/sdl/threatmodeling](https://www.microsoft.com/en-us/securityengineering/sdl/threatmodeling)
- **OWASP Threat Modeling Project / Cheat Sheet.** [owasp.org/www-community/Threat_Modeling](https://owasp.org/www-community/Threat_Modeling)
- **Shostack, A. — *Threat Modeling: Designing for Security*.** A standard reference; much of the DFD vocabulary this module reuses.

## AI-specific memorisation / extraction / MIA literature

(Context for the model-weights category of threats; mod-105 covers the depth.)

- **Carlini et al. — "Extracting Training Data from Large Language Models"** (USENIX Security 2021). [arxiv.org/abs/2012.07805](https://arxiv.org/abs/2012.07805)
- **Carlini et al. — "Membership Inference Attacks From First Principles" (LiRA)** (IEEE S&P 2022). [arxiv.org/abs/2112.03570](https://arxiv.org/abs/2112.03570)
- **Shokri et al. — "Membership Inference Attacks Against Machine Learning Models"** (IEEE S&P 2017). [arxiv.org/abs/1610.05820](https://arxiv.org/abs/1610.05820)
- **Nasr et al. — "Scalable Extraction of Training Data from (Production) Language Models"** (2023). [arxiv.org/abs/2311.17035](https://arxiv.org/abs/2311.17035)

## Embedding / retrieval privacy

(Context for mod-109; useful when scoping D2 / I threats on retrieval systems.)

- **Morris, Kuleshov, Shmatikov, Rush — "Text Embeddings Reveal (Almost) as Much as Text"** / vec2text (EMNLP 2023). [arxiv.org/abs/2310.06816](https://arxiv.org/abs/2310.06816)

## Differential privacy foundations

(Context for mod-106; useful when the LINDDUN model routes to DP.)

- **NIST SP 800-226 — Guidelines for Evaluating Differential Privacy Guarantees.** [doi.org/10.6028/NIST.SP.800-226](https://doi.org/10.6028/NIST.SP.800-226)
- **Dwork, C., Roth, A. — *The Algorithmic Foundations of Differential Privacy*.** [cis.upenn.edu/~aaroth/privacybook.html](https://www.cis.upenn.edu/~aaroth/privacybook.html)

## Tooling

- **OWASP Threat Dragon** — open-source threat-modelling tool with STRIDE support; useful DFD editor. [owasp.org/www-project-threat-dragon](https://owasp.org/www-project-threat-dragon/)
- **Mermaid** — text-based diagramming for inline DFDs. [mermaid.js.org](https://mermaid.js.org/)
- **draw.io / diagrams.net** — DFD / architecture diagrams; diffable XML export. [drawio.com](https://www.drawio.com/)

## Cross-references within this track

- **mod-101** — privacy frameworks and regulator reading. Chapter 02 of this module assumes the GDPR / EU AI Act reading you did there.
- **mod-103** — data governance and minimisation (routing target for L / I / Non-compliance threats on ingest).
- **mod-104** — PII detection + redaction (routing target for I / D2 threats on logs, caches, evaluation sets).
- **mod-105** — memorisation / extraction / MIA (routing target for I / D1 threats on model weights).
- **mod-106** — differential privacy engineering (routing target for I / D1 threats where DP is the appropriate defence).
- **mod-107** — federated learning + secure aggregation (routing target for D2 threats where centralising data is the problem).
- **mod-108** — confidential computing + privacy-preserving inference (routing target for D2 threats on the model-provider boundary).
- **mod-109** — RAG privacy + agent scoping (routing target for L / D2 / U threats on retrieval and agent-tool telemetry).
- **mod-110** — machine unlearning + DSR engineering (routing target for U / Non-compliance threats on rights-handling).
- **mod-111** — DPIA + regulator-grade evidence (consumer of this module's output).
- **mod-112** — AI privacy program + communication (owner of the model-card / dataset-card / notice surfaces this module gates).
