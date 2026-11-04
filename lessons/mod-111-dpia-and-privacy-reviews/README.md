# mod-111-dpia-and-privacy-reviews: DPIAs, Privacy Reviews, and Audit-Grade Privacy Evidence

**Estimated effort:** 14 hours

A DPIA is not the compliance memo written after the AI system ships. It is the risk instrument the GDPR requires *before* processing starts (Art. 35), the shape a US privacy programme takes through NIST's PRAM, the European regulator-consistent reading mandated by EDPB Opinion 28/2024 and the CNIL AI sheets, and the sectoral cross-tag that satisfies HIPAA / GLBA / SP 800-53 PT-family auditors where their data is in scope. The privacy engineer at level 30 is the author of this risk instrument and the assembler of the regulator-facing bundle that supports it; `ai-evaluation-engineer` at level 35 consumes the bundle as a contribution into the broader release-assurance interface. This module is where you learn to produce an audit-grade DPIA and the bundle that survives an inspection.

## Learning objectives

- Author an audit-grade GDPR Article 35 DPIA for an AI system — systematic description of processing, necessity and proportionality assessment, risk-to-rights-and-freedoms assessment, mitigation measures, residual-risk statement, DPO consultation record — mapped to the ICO DPIA shape and ISO/IEC 29134.
- Operate the NIST Privacy Risk Assessment Methodology (PRAM) worksheets end-to-end — Data Action A / B / C mapping, problematic-data-action enumeration, privacy-risk scoring — and produce a NIST-PF-compatible evidence pack.
- Read EDPB Opinion 28/2024 on AI models and CNIL's AI sheets as the European regulator stance on AI + GDPR — translate the opinion's four questions (anonymity of the model, lawful basis, data-subject rights, legitimate-interest test) into engineering artefacts your DPIA cites.
- Handle the HIPAA-shape privacy / security / breach-notification requirement where PHI is in scope, and GLBA / NIST SP 800-53 PT-family controls where financial / federal data is in scope; cross-tag each control into the privacy evidence pack.
- Package a regulator-facing privacy-engineering bundle — DPIA, data-flow diagram, PET deployment claim with ε / δ / attestation / K-anonymity parameters, MIA / extraction eval report, redaction-pipeline FP / FN report, unlearning-capability statement, logging / retention policy, breach-response plan — that survives a supervisory-authority inspection.
- Reason about the Cloud Security Alliance AI Controls Matrix and ISO/IEC 42001 AIMS clauses relevant to privacy — know which controls show up in an enterprise audit.
- Cite the boundary to `ai-evaluation-engineer` (peer / next-up, level 35) — that role owns release-assurance / audit-trail / regulator-facing methodology at full depth; this role contributes the privacy-engineering evidence packet into that release-assurance interface.

## Chapters

1. [Authoring an audit-grade GDPR Article 35 DPIA for an AI system](01-gdpr-article-35-dpia-authoring.md) — the six statutory elements (a–f), the AI-specific feared-event register, the DPO consultation record, the Art. 36 prior-consultation gate, the ISO/IEC 29134 mapping, the DPIA as a living document.
2. [The ICO DPIA template walkthrough](02-ico-dpia-shape-walkthrough.md) — Steps 1–7 of the ICO template populated section-by-section for an AI system; the AI-specific appendix the ICO guidance expects; the common failure modes the ICO catches.
3. [NIST PRAM worksheets and NIST Privacy Framework evidence pack](03-nist-pram-worksheets-and-privacy-framework.md) — Worksheet 1 framing, Worksheet 2 data actions (A1–A3 / B1–B6 / C1–C5), the nine-category problematic-data-action taxonomy, Worksheet 3 prioritisation, Worksheet 4 response; the NIST PF Subcategory × Artefact matrix; AI RMF overlay.
4. [EDPB Opinion 28/2024 and CNIL AI sheets](04-edpb-opinion-28-2024-and-cnil-ai-sheets.md) — the four questions (anonymity, lawful basis / LI three-step, unlawful development consequences, DSR), the engineering artefacts per question, the CNIL AI-sheet set as the engineering-facing complement, ICO alignment.
5. [HIPAA, GLBA, and NIST SP 800-53 PT family](05-hipaa-glba-and-sp-800-53-pt-family.md) — HIPAA Privacy / Security / Breach Notification Rules; GLBA Safeguards 2021/2023 + SEC Regulation S-P 2024; SP 800-53 Rev. 5 PT family and AI-touching AC / AU / SI controls; cross-tagging discipline.
6. [Packaging the regulator-facing privacy-engineering bundle](06-regulator-facing-privacy-bundle.md) — the eight artefact classes (DPIA, DFD, PET claim, MIA report, redaction report, unlearning capability, retention policy, breach-response plan); versioning, hashing, manifest; the inspection-response playbook.
7. [CSA AI Controls Matrix and ISO/IEC 42001 AIMS](07-csa-aicm-and-iso-42001-aims.md) — AICM privacy-relevant domains, 42001 Clause 8.3 and Annex A; composition with 27001 / 27701; NIST AI RMF, MITRE ATLAS, OWASP LLM Top 10 as cross-tags; the certification decision.
8. [The boundary to `ai-evaluation-engineer`](08-boundary-to-ai-evaluation-engineer.md) — what the privacy engineer owns, what level 35 release-assurance owns, the hand-off artefact set, the escalation directions, the one-page release-recommendation note.

## Exercises

See [`exercises/`](exercises/):

1. [GDPR Article 35 DPIA authoring](exercises/exercise-01-gdpr-article-35-dpia-authoring.md) — author the six statutory sections for a specified AI system.
2. [ICO DPIA shape walkthrough](exercises/exercise-02-ico-dpia-shape-walkthrough.md) — populate the seven-step ICO template end-to-end.
3. [NIST PRAM worksheet run](exercises/exercise-03-nist-pram-worksheet-run.md) — build Worksheets 1–4 for a specified AI system and produce the PF matrix.
4. [EDPB Opinion 28/2024 and CNIL sheets read](exercises/exercise-04-edpb-opinion-28-2024-and-cnil-sheets-read.md) — translate the four questions into engineering artefacts for a scenario.
5. [HIPAA / GLBA / Reg S-P breach-trigger map](exercises/exercise-05-hipaa-security-privacy-breach-trigger-map.md) — produce the AI-specific decision tree that routes an event to the right notification pathway.
6. [Regulator-facing privacy bundle authoring](exercises/exercise-06-regulator-facing-privacy-bundle-authoring.md) — produce the full eight-artefact bundle with manifest, hashes, and cross-tag.
7. [ISO/IEC 42001 AIMS privacy slice](exercises/exercise-07-iso-42001-aims-privacy-slice.md) — map the privacy artefact set to 42001 Clause 8.3 and the privacy-relevant Annex A controls.

## Structure

- `01-…md` … `08-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs (scaffolded placeholder).
- `quizzes/`: knowledge checks (scaffolded placeholder).
- `resources.md`: external references — the GDPR DPIA articles and EDPB / WP29 guidance, ISO/IEC 29134 and 42001, the ICO template and AI guidance, the NIST PRAM and AI RMF, EDPB Opinion 28/2024, CNIL AI sheets, HIPAA / GLBA / Reg S-P, SP 800-53 Rev. 5, CSA AICM, MITRE ATLAS, OWASP LLM Top 10.

## Where this module sits in the track

- Upstream dependencies: mod-101 (frameworks and position — the regulatory map the DPIA cites), mod-102 (threat modelling — the threat register that seeds the feared-event list), mod-103 (data governance — the inventory and RoPA the DPIA cites), mod-104 (redaction — the FP / FN report the bundle contains), mod-105 (memorisation and MIA — the empirical anonymity-and-residual-risk evidence), mod-106 (DP engineering — the PET deployment claim's parameters), mod-107 (federated and secure aggregation — a further PET claim), mod-108 (confidential computing — the attestation claim), mod-109 (RAG and agent scoping — the system card and the retrieval-corpus controls), mod-110 (unlearning and DSR — the DSR runbook set and the unlearning-capability statement).
- Downstream consumers: mod-112 (privacy program and communication — the subject-facing and regulator-facing narrative wraps this module's artefacts) and the `ai-evaluation-engineer` (level 35) role that consumes the bundle as the privacy input to release assurance.
- Hand-off upstream: the DPO (controller-side mandatory under GDPR Art. 37 where triggered) is the DPIA's statutory reviewer; the privacy engineer is the technical author and bundle assembler.
