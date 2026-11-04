# CSA AI Controls Matrix and ISO/IEC 42001 — The Enterprise-Audit Overlay

A supervisory authority inspection is one audience; an enterprise audit is a second. The enterprise auditor (Big Four, internal audit, a customer's third-party-risk team, a certification body) does not open with Art. 35; they open with a control catalogue and an expectation of evidence per control. For AI privacy work the two control catalogues an enterprise auditor will reach for are **the Cloud Security Alliance's AI Controls Matrix (CSA AICM)** and **ISO/IEC 42001** (the AI Management System standard). Both were published in the 2023–2024 window and have become the default expectation for AI programmes undergoing third-party audit; a privacy-engineering programme that is well-prepared for a regulator will do most of the work needed for the enterprise audit, but the cross-mapping — and some control-specific gaps — need explicit attention.

This chapter names the CSA AICM's structure, the ISO/IEC 42001 clauses that touch privacy, and the overlay that lets the chapter-06 bundle answer both regulator and enterprise auditor without producing a parallel evidence set.

Primary references — the authoritative sources:

- Cloud Security Alliance — *AI Controls Matrix (AICM)*, v1.0 (2024 release). [cloudsecurityalliance.org/research/working-groups/ai-controls](https://cloudsecurityalliance.org/research/working-groups/ai-controls).
- Cloud Security Alliance — *Cloud Controls Matrix (CCM)*, v4.x (companion catalogue the AICM composes with). [cloudsecurityalliance.org/research/cloud-controls-matrix/](https://cloudsecurityalliance.org/research/cloud-controls-matrix/).
- ISO/IEC 42001:2023 — *Information technology — Artificial intelligence — Management system*. [iso.org/standard/81230.html](https://www.iso.org/standard/81230.html).
- ISO/IEC 23894:2023 — *Information technology — Artificial intelligence — Guidance on risk management*. [iso.org/standard/77304.html](https://www.iso.org/standard/77304.html).
- ISO/IEC 27001:2022 and ISO/IEC 27701:2019 — the ISMS and PIMS standards 42001 composes with. [iso.org/standard/27001](https://www.iso.org/standard/27001) and [iso.org/standard/71670.html](https://www.iso.org/standard/71670.html).

## The CSA AI Controls Matrix — structure and scope

The CSA AICM is a cloud-centric control catalogue specific to AI systems. It extends the Cloud Controls Matrix (CCM) — CSA's long-standing cloud-security catalogue — with AI-specific control domains. The structure:

- **Domains** — top-level groupings of controls. The AICM domains span the AI lifecycle: AI governance (strategy, roles, policy), model security (model-weights protection, supply chain, serving), AI operations (deployment, monitoring, incident response), responsible AI (bias, fairness, explainability, human oversight), privacy (data-handling, DSR, DPIA-alignment), threat and vulnerability management (AI-specific threat modelling, red-teaming), third-party risk, and the horizontal cloud controls inherited from CCM.
- **Controls** — specific requirements under each domain. Each control has a unique identifier, a statement of expectation, and (in AICM v1) mappings to upstream references including NIST AI RMF, ISO/IEC 42001, OWASP LLM Top 10, MITRE ATLAS, and the ENISA AI Threat Landscape.
- **Assessment approach** — expectations for cloud-service-provider (CSP) implementation and customer-side responsibility, following the CCM's shared-responsibility model.

The AICM is a *free* control catalogue CSA publishes for community use; the companion CAIQ-style questionnaire is used by customers to request a vendor's AI-security posture. If your controller sells an AI-enabled product to enterprise customers, the AICM is almost certainly what one of those customers is going to ask you to answer.

### The privacy-relevant AICM domains

Not every AICM domain is privacy-engineering territory. The ones that are:

- **Privacy** (explicit domain) — data minimisation, DSR engineering, DPIA / PIA alignment, consent, lawful basis, de-identification, retention. These are the controls that most directly map to the chapter-06 bundle.
- **Responsible AI** — bias and fairness evaluation, explainability, human oversight, provenance of training data. The fairness controls are privacy-adjacent (discrimination is both a fairness issue and a rights-and-freedoms issue under Art. 35); the training-data-provenance controls overlap the chapter-04 EDPB posture.
- **Model security** — training-data protection, model-weights protection, model-serving confidentiality. The chapter-06 PET deployment claims (DP, TEE, federated) satisfy many of the model-security controls.
- **AI operations** — logging, monitoring, incident response for AI-specific signals. The chapter-06 breach-response plan satisfies this when its AI-specific decision branches are populated.
- **Third-party risk** — AI supply-chain controls for foundation-model vendors, hosted-API processors, model-marketplace vendors.
- **Threat and vulnerability management** — AI-specific threat modelling, red-teaming, patch management for the AI stack. The MIA / extraction report from chapter 06 satisfies parts of the vulnerability-measurement controls; mod-102 threat modelling satisfies others.

### Mapping chapter-06 bundle artefacts to AICM controls

A worked example of the cross-tag for a few representative AICM-privacy-relevant controls (the exact control identifiers and wording in the published AICM should be checked against the release; the mapping is substantive rather than literal):

| AICM control concern | Chapter-06 artefact | Also satisfies |
|---|---|---|
| Written AI governance policy and accountability roles | DPIA Step 7 approval chain; organisational policy pack | ISO 42001 clause 5 (Leadership) |
| AI system inventory and risk tiering | RoPA, DPIA Step 1 trigger assessment | NIST AI RMF MAP 1, ISO 42001 clause 6.1 |
| Training-data provenance and lawful collection | Per-corpus provenance chain; EDPB-aligned LI-basis memo (ch. 04) | GDPR Art. 6, EDPB Opinion 28/2024 |
| Data minimisation applied to training corpora | Training-data manifest with per-category justification | ICO AI guidance, CNIL AI sheet on minimisation |
| DP / PET deployment with parameters | PET deployment claim with ε/δ/C/σ | NIST AI RMF MANAGE 2.3 |
| Confidential computing attestation | TEE attestation claim with measurement hash | SP 800-53 SC-28, CC-related CCM controls |
| Membership-inference and extraction testing | MIA / extraction report | NIST AI RMF MEASURE 2.6, OWASP LLM Top 10 (training data poisoning / extraction) |
| Output-filter and prompt-injection defence | SI-10 / SI-15 evidence; red-team report | OWASP LLM Top 10 LLM01 (prompt injection), LLM02 (insecure output handling) |
| DSR implementation against AI systems | Unlearning-capability statement; per-right runbook from mod-110 | GDPR Art. 15–22, HIPAA access/amendment |
| Logging and retention with enforcement | Retention matrix + infrastructure-as-code | SP 800-53 AU-11, SI-12 |
| Breach-response for AI-specific signals | Breach-response plan with AI branches | GDPR Art. 33/34, HIPAA Breach Notification Rule |
| Third-party AI processor oversight | Vendor-assessment memos; DPA / BAA / SCCs | GDPR Art. 28, HIPAA BAA |
| Human oversight for consequential decisions | Art. 22 decision pathway, human-review endpoint | NIST AI RMF MANAGE 3.1, ISO 42001 clause 8.3 |

A privacy engineer who has built the chapter-06 bundle has 70–85% of what the AICM asks for. The remaining fraction is in controls outside the privacy perimeter (model-weights cryptographic integrity, model-serving capacity management, red-team process maturity) that live with model-security, SRE, and security engineering.

## ISO/IEC 42001 — the AI Management System standard

ISO/IEC 42001:2023 is the AI-specific analogue of ISO/IEC 27001 (information security management) and ISO/IEC 27701 (privacy information management). It specifies requirements for an AI Management System (AIMS) and is certifiable — an accredited certification body can audit and certify an organisation's AIMS, a stronger assurance signal than a self-attestation.

The standard's structure follows the ISO management-system pattern (10 clauses):

1. Scope
2. Normative references
3. Terms and definitions
4. Context of the organisation
5. Leadership
6. Planning
7. Support
8. Operation
9. Performance evaluation
10. Improvement

Plus **Annex A** — the AI-specific controls that implement the management-system requirements. Annex A is where the privacy engineer spends most of their mapping effort.

### The privacy-relevant 42001 clauses

- **Clause 4 (Context)** — identify interested parties and their requirements. Privacy requirements (regulatory, customer, subject) are interested-party requirements that the AIMS must capture. The DPIA's scope statement and the chapter-03 PRAM Worksheet 1 framing feed this clause.
- **Clause 5 (Leadership)** — AI policy, roles and responsibilities, management commitment. The DPO / privacy-engineering role sits in the structure; the organisational privacy policy is an input.
- **Clause 6 (Planning)** — AI risk assessment and AI risk treatment plan; AI objectives. The DPIA / PRAM output is the AI risk assessment's privacy slice; the risk-treatment plan is the chapter-06 bundle's mitigation table.
- **Clause 7 (Support)** — resources, competence, awareness, communication, documented information. The privacy-engineering competency, training records, and documented information discipline feed this clause.
- **Clause 8 (Operation)** — operational planning and control; AI system impact assessment; data management; AI system lifecycle. **Clause 8.3 specifically — AI system impact assessment** — is the clause that points at the DPIA / PRAM artefact as its operational evidence.
- **Clause 9 (Performance evaluation)** — monitoring, measurement, internal audit, management review. The MIA / extraction audit cadence, the DPIA review cadence, and the DPO's review memo all feed this clause.
- **Clause 10 (Improvement)** — nonconformity and corrective action, continual improvement. The incident-response plan's post-incident review and the DPIA refresh pipeline feed this clause.

### Annex A — the privacy-touching controls

Annex A of ISO/IEC 42001 contains the AI-specific control catalogue. The privacy-relevant control areas (the standard numbers each control; the categorisation below is thematic rather than clause-by-clause):

- **Policies related to AI** — organisational AI policy, with privacy policy as a component.
- **Internal organisation and reporting** — accountability for AI systems and the privacy decisions they imply.
- **Resources** — personnel, infrastructure, data; the data-resource controls overlap data-governance obligations.
- **AI system impact assessment** — the control that explicitly expects a DPIA / PRAM-equivalent artefact. The 42001 auditor will ask to see it.
- **AI system lifecycle** — design, development, verification, deployment, monitoring, retirement. The privacy-engineering decisions made at each lifecycle stage are 42001-auditable.
- **Data for AI systems** — provenance, quality, representativeness, licensing, consent. The chapter-04 training-data provenance chain satisfies the data-provenance controls.
- **Information for interested parties** — transparency, privacy notice, logic description for automated decisions. mod-112 territory.
- **Use of AI systems** — responsible use, intended-use documentation, monitoring for misuse.
- **Third-party and customer relationships** — supplier controls, customer responsibilities.

An organisation seeking 42001 certification will build out the AIMS documentation (policy, scope, risk register, SoA — Statement of Applicability, operational controls) that references the privacy-engineering bundle as its primary source for privacy-related controls.

### The 42001 / 27701 / 27001 composition

ISO/IEC 42001 is designed to compose with 27001 (ISMS) and 27701 (PIMS). An organisation already certified against 27001 and 27701 has large parts of 42001's horizontal controls (Clauses 4–7, 9, 10) already satisfied. 42001 adds the AI-specific Annex A content and Clause 8.3 (AI system impact assessment) on top.

For a privacy-engineering programme this is good news: if your controller has an ISMS and a PIMS, the overhead of 42001 is primarily the AI-specific operational controls — most of which are the chapter-06 bundle under a different label.

## The NIST AI RMF overlay — the voluntary US-federal framework

The NIST AI Risk Management Framework (AI RMF 1.0, 2023) is voluntary rather than certifiable, but is the US-federal and increasingly US-industry reference for AI risk governance. The four Functions (Govern, Map, Measure, Manage) with their Categories and Subcategories compose with the NIST Privacy Framework (chapter 03) and with ISO/IEC 42001.

The AI RMF Generative AI Profile (NIST AI 600-1, 2024) overlays generative-AI-specific threat categories; the privacy-relevant ones include confabulation (fabricated personal data), data privacy (exposure of training data, inference about individuals), harmful bias, information integrity, and intellectual property.

The AI RMF subcategory tags are commonly applied in the chapter-03 NIST PF matrix as a second column; an enterprise audit that uses the AI RMF as its framework will expect the matrix with both Privacy Framework and AI RMF tags.

## MITRE ATLAS and OWASP LLM Top 10 — threat frames the auditor cites

Two additional control-adjacent references the enterprise auditor commonly cites:

- **MITRE ATLAS** — the adversarial threat matrix for machine learning, structured like MITRE ATT&CK. The threat IDs (e.g., AML.T0051 "LLM Prompt Injection", AML.T0048 "External Harms", AML.T0043 "Craft Adversarial Data") give a shared vocabulary for the threat register. [atlas.mitre.org](https://atlas.mitre.org/).
- **OWASP Top 10 for Large Language Model Applications (v2.0, 2025)** — the LLM01–LLM10 category list (prompt injection, insecure output handling, training-data poisoning, model DoS, supply-chain, sensitive-information disclosure, insecure plugin design, excessive agency, overreliance, model theft). [genai.owasp.org/llm-top-10/](https://genai.owasp.org/llm-top-10/).

Each entry in the chapter-06 MIA / extraction report, the output-filter evidence, and the red-team report should carry its ATLAS or OWASP tag where applicable. An enterprise auditor reading "we tested against OWASP LLM01 (Prompt Injection) with the probe set [X], measured success rate [Y], and remediated by [Z]" is reassured.

## The cross-tag table — bundle to AICM to 42001 to AI RMF

An efficient privacy-engineering programme maintains a single cross-tag table that maps each bundle artefact to:

- NIST Privacy Framework Subcategory.
- NIST AI RMF Subcategory.
- ISO/IEC 42001 Annex A control.
- CSA AICM control ID.
- SP 800-53 Rev. 5 control ID.
- ISO/IEC 27701 PIMS control.
- GDPR / HIPAA / GLBA / Reg S-P article or section.

The table is tedious to maintain by hand; several commercial GRC tools exist to automate it. The table's value is in the inspection-response and audit-response pathways: a single artefact citation answers five or six control questions at once.

## The audit experience — what the enterprise auditor expects

The enterprise audit rhythm:

1. **Scoping meeting** — the audit scope (one system, a programme, the full organisation), the control catalogue in use, the evidence expectations.
2. **Request for documentation** — the auditor sends an evidence request list. The bundle's manifest and the cross-tag table respond.
3. **Walkthrough** — the auditor walks the system with the engineering lead; the bundle artefacts come up in the walkthrough.
4. **Control testing** — the auditor samples controls and asks for evidence of operation (not just design). The retention-matrix enforcement logs, the PET-accountant logs, the MIA-audit refresh records are tested here.
5. **Interview** — with the DPO, the privacy engineer, the product owner. The interview tests whether the controls are operating in practice or just on paper.
6. **Draft findings** — the auditor's write-up of nonconformities and observations.
7. **Management response** — the controller's response to findings, with remediation plans and dates.
8. **Final report** — the audited position, with findings closed or carried to a corrective-action plan.

A privacy-engineering programme that operates the chapter-06 bundle discipline sails through steps 2–5; the findings in step 6 are typically scoped to controls outside the privacy perimeter (model-weight cryptographic signing, infrastructure patching, SRE on the serving path) or to maturity-gap findings in the Clauses 7 and 9 territory (training records not comprehensive; internal audit not formalised).

## When certification is worth the cost — and when it is not

42001 certification is non-trivial (annual fees, external audit cost, documentation overhead); AICM-based self-assessment with a CAIQ-style response is cheaper. The engineering judgement:

- If the controller sells AI products into regulated industries (healthcare, financial services, public sector) or into enterprise customers with mature third-party-risk programmes, 42001 certification is increasingly the admission ticket.
- If the controller is a consumer-facing product without regulated-industry customers, the AICM self-assessment is often sufficient today.
- The privacy engineer's input to the decision: what fraction of the certification's incremental work is new, vs. what fraction is already being done for the regulator-facing bundle? For most programmes the answer is "mostly already done", which tips toward certification.

Independent of certification, maintaining the cross-tagged bundle is the engineering practice that pays off in both regulator and enterprise contexts.

## Summary — what you can do after this chapter

- Read the CSA AI Controls Matrix as the cloud-oriented AI control catalogue and map each privacy-relevant control to the chapter-06 bundle.
- Read ISO/IEC 42001 as the AI management-system standard and identify Clause 8.3's AI system impact assessment as the point where your DPIA / PRAM lands, plus the Annex A privacy-relevant controls.
- Understand the 42001 / 27701 / 27001 composition — an existing ISMS / PIMS reduces 42001 implementation effort to the AI-specific slice.
- Overlay the NIST AI RMF (and the Generative AI Profile) onto the chapter-03 NIST PF matrix for US-federal and voluntary-framework audiences.
- Attach MITRE ATLAS and OWASP LLM Top 10 tags to the threat and evaluation artefacts so the enterprise auditor's threat vocabulary is matched.
- Maintain a single cross-tag table that maps each bundle artefact to Privacy Framework, AI RMF, 42001, AICM, SP 800-53, 27701, and statutory article identifiers.
- Decide whether the organisation pursues 42001 certification based on the business case and the marginal work remaining above the bundle.

Chapter 08 closes the module by naming the boundary to `ai-evaluation-engineer` — the peer / next-up role at level 35 that owns release-assurance and regulator-facing methodology at full depth, and consumes this module's bundle as a contribution to the release-assurance interface.
