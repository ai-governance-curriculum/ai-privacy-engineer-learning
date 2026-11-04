# HIPAA, GLBA, and NIST SP 800-53 PT Family — Sectoral Overlays for the Privacy Evidence Pack

A DPIA grounded only in GDPR / UK GDPR / NIST PF under-serves any AI system that processes Protected Health Information, financial-institution customer data, or federal / federally-adjacent personal data. Each of those categories brings its own statutory scheme with its own breach triggers, its own retention and access-control obligations, and its own audit expectations. The privacy engineer's job in this module is to make sure every applicable sectoral obligation is cross-tagged into the evidence pack, so an auditor who shows up with a HIPAA audit protocol, a GLBA Safeguards examination manual, or an SP 800-53 PT-family checklist finds the engineering evidence where they expect it.

This chapter walks three overlays: (1) HIPAA where PHI is in scope, (2) GLBA Safeguards Rule plus SEC Regulation S-P where financial / brokerage data is in scope, and (3) NIST SP 800-53 Rev. 5 PT (PII Processing and Transparency) family for federal and federally-adjacent systems. State-law overlays (CCPA / CPRA and the growing US state bench) are handled at the individual-right level in mod-110 ch. 01 and in the lawful-basis column of the DPIA (chapter 02); this chapter stays with the three overlays that bring the heaviest and most structurally-distinct audit obligations.

Primary references — pin the authoritative texts:

- US HHS — *HIPAA Administrative Simplification*, consolidated regulatory text at 45 CFR Parts 160, 162, 164. [hhs.gov/hipaa/for-professionals/privacy/laws-regulations/combined-regulation-text/index.html](https://www.hhs.gov/hipaa/for-professionals/privacy/laws-regulations/combined-regulation-text/index.html).
- US HHS — *HIPAA Privacy Rule*, 45 CFR Part 164 Subpart E. *Security Rule*, 45 CFR Part 164 Subpart C. *Breach Notification Rule*, 45 CFR Part 164 Subpart D.
- US HHS OCR — *Guidance on De-identification of Protected Health Information* (Safe Harbor / Expert Determination). [hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification/index.html](https://www.hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification/index.html).
- FTC — *Standards for Safeguarding Customer Information* (GLBA Safeguards Rule), 16 CFR Part 314, as amended 2021/2023. [ftc.gov/legal-library/browse/rules/safeguards-rule](https://www.ftc.gov/legal-library/browse/rules/safeguards-rule).
- SEC — *Regulation S-P: Privacy of Consumer Financial Information and Safeguarding Personal Information*, 17 CFR Part 248, as amended 2024. [sec.gov/rules/final/2024/34-100155.pdf](https://www.sec.gov/rules/final/2024/34-100155.pdf).
- NIST — *SP 800-53 Rev. 5*. [doi.org/10.6028/NIST.SP.800-53r5](https://doi.org/10.6028/NIST.SP.800-53r5).
- NIST — *SP 800-53B, Control Baselines for Information Systems and Organizations*. [doi.org/10.6028/NIST.SP.800-53B](https://doi.org/10.6028/NIST.SP.800-53B).
- NIST — *SP 800-122, Guide to Protecting the Confidentiality of Personally Identifiable Information (PII)*. [doi.org/10.6028/NIST.SP.800-122](https://doi.org/10.6028/NIST.SP.800-122).

## HIPAA — the three-rule stack for PHI

HIPAA applies to *covered entities* (health plans, healthcare clearinghouses, most healthcare providers) and to their *business associates* (parties that create, receive, maintain, or transmit PHI on behalf of a covered entity). Most AI engineering work in a healthcare context is work *as a business associate* — the engineering organisation signs a Business Associate Agreement (BAA) with the covered entity and takes on compliance obligations directly.

HIPAA's substantive content lives in three rules, each of which contributes obligations the AI privacy engineer must cross-tag into the evidence pack:

### Privacy Rule — permitted uses and disclosures of PHI

The Privacy Rule (45 CFR 164 Subpart E) governs *when* PHI may be used or disclosed. For an AI system, the Privacy Rule bites hardest on:

- **Minimum necessary** (§164.502(b)) — the AI system may use only the minimum PHI reasonably necessary to accomplish the permitted purpose. For training a model, the engineering question is *why the model needs the PHI it is being trained on* — if the clinical-decision-support model can be trained on aggregated / de-identified data, the minimum-necessary test will push toward that.
- **Uses for treatment / payment / operations (TPO)** — the broadest permitted uses. AI for internal quality improvement typically qualifies as "operations"; AI for direct patient-care decision support qualifies as "treatment"; AI for marketing requires separate authorisation.
- **Authorisation for other uses** (§164.508) — research, marketing, sale of PHI generally requires subject authorisation. Training a commercial AI product on PHI generally requires authorisation unless an exception applies (de-identified data, limited data set under a DUA, research with IRB approval and waiver).
- **De-identification** (§164.514(a)–(b)) — PHI that is de-identified under the Safe Harbor method (18 identifiers removed) or Expert Determination (statistical determination of very small re-identification risk) is no longer PHI and may be used without further Privacy Rule restriction. The AI engineer's path to using training data without authorisation often runs through de-identification.
- **Accounting of disclosures** (§164.528) — the subject has a right to an accounting of disclosures of their PHI for certain purposes (not TPO, broadly). The AI system must be able to produce the accounting if it discloses PHI in a way the rule covers.
- **Access** (§164.524) — the subject has a right of access to their PHI. For an AI system that stores inferred PHI (an AI-generated diagnosis prediction, for example), the inferred data is PHI and is subject to access.
- **Amendment** (§164.526) — the subject has a right to request amendment of PHI. Similar to GDPR rectification at mod-110 ch. 01, with HIPAA-specific procedural requirements.

### Security Rule — administrative, physical, and technical safeguards for ePHI

The Security Rule (45 CFR 164 Subpart C) applies to electronic PHI. The engineering obligations:

- **Administrative safeguards** (§164.308) — security management process (risk analysis, risk management), workforce security, information access management, security training, incident procedures, contingency plan, evaluation.
- **Physical safeguards** (§164.310) — facility access controls, workstation security, device and media controls.
- **Technical safeguards** (§164.312) — access control (unique user identification, emergency access procedure, automatic logoff, encryption and decryption), audit controls, integrity controls, transmission security.
- **Organisational requirements** (§164.314) — BAA content, group health plan documents.
- **Policies and procedures** (§164.316) — documentation of policies, procedures, and other required records; retention for six years.

The AI-specific Security Rule crossover points:

- **Risk analysis** (§164.308(a)(1)(ii)(A)) — a periodic and ongoing risk analysis for ePHI. The DPIA's risk register and the PRAM output are inputs; a HIPAA risk analysis must also address the AI-specific threats (memorisation, MIA, inversion, prompt-injection exfiltration) that the Privacy Rule's minimum-necessary posture leaves exposed.
- **Audit controls** (§164.312(b)) — audit trails on access to and use of ePHI. For AI systems, this includes training-data access, inference-time retrieval of PHI from a feature store, prompt-log access, derivative-product production.
- **Integrity** (§164.312(c)) — ePHI is not improperly altered or destroyed. AI systems where the model infers PHI about the subject must ensure the inferred-PHI storage has integrity controls.
- **Transmission security** (§164.312(e)) — encryption of ePHI in transit, including to third-party model APIs. If your AI pipeline calls a vendor LLM with PHI in the prompt, this is a Security Rule obligation and almost always a BAA requirement with the vendor.

### Breach Notification Rule — the clock and the obligation

The Breach Notification Rule (45 CFR 164 Subpart D) requires notification of a *breach of unsecured PHI* to:

- **Affected individuals** — without unreasonable delay and in no case later than **60 calendar days** after discovery (§164.404(b)).
- **HHS** — breaches affecting 500 or more individuals: contemporaneously with individual notice. Smaller breaches: annually in the breach report (§164.408).
- **Media** — breaches affecting more than 500 residents of a State or jurisdiction (§164.406).

The "unsecured PHI" qualifier matters: PHI secured by encryption meeting HHS specifications or by destruction is not subject to the breach-notification obligation. The engineering implication: AI systems that process PHI should encrypt it to the HHS spec throughout, so an incident that compromises the encrypted container does not trigger the 60-day clock.

The AI-specific breach scenarios:

- **Training-data extraction** that reveals PHI — a successful Carlini-style extraction attack against a model trained on PHI is a disclosure of PHI; whether it is a "breach" under §164.402 depends on whether the PHI was secured. If the model was not DP-trained and emits verbatim training data, the extracted PHI is unsecured and the breach rule applies.
- **Prompt-log compromise** containing PHI — straightforward breach if the logs were not encrypted.
- **Vendor processor breach** — the business associate obligations under §164.410 require the business associate to notify the covered entity within 60 days; the controller's clock to subjects starts on discovery.
- **Inferred-PHI disclosure** — if the AI system inferred a diagnosis and the inference storage was compromised, the inferred PHI counts as PHI for the rule.

The DPIA's breach-response plan (chapter 06) must have HIPAA-specific decision logic: PHI breach? → 60-day clock, HHS notification path, media-notification trigger at 500, BAA-upstream notification path. The decision tree is part of the regulator-facing bundle.

### Cross-tagging HIPAA into the evidence pack

Each HIPAA obligation should be cross-tagged to an artefact. A worked mapping:

| HIPAA obligation | Artefact |
|---|---|
| §164.502(b) minimum necessary | Training-data manifest with per-category justification; feature-set minimisation note |
| §164.508 authorisation | Patient-authorisation form, authorisation-registry |
| §164.514(b) de-identification | De-identification method documentation (Safe Harbor checklist or Expert Determination report) |
| §164.524 access | DSR runbook (mod-110 ch. 01) mapping to HIPAA-specific procedure |
| §164.526 amendment | Correction-propagation pipeline (mod-110 ch. 01 Art. 16 analogue) |
| §164.528 accounting | Disclosure-logging pipeline and subject-facing export |
| §164.308(a)(1) risk analysis | DPIA risk register + HIPAA-specific supplemental analysis |
| §164.312(a) access control | Identity, MFA, RBAC, audit-log policy |
| §164.312(b) audit controls | ePHI-access log, retention policy |
| §164.312(e) transmission security | TLS policy, BAA with downstream processors |
| §164.404–406 breach notification | Breach-response plan with PHI decision branch |

The evidence pack (chapter 06) includes this mapping. An OCR audit opens with the organisation's Security Rule risk analysis; AI programmes are expected to have a current, AI-aware version.

## GLBA Safeguards Rule — financial-institution customer data

The Gramm-Leach-Bliley Act's Safeguards Rule (16 CFR Part 314, as amended by the FTC in 2021 and 2023) applies to a defined category of "financial institutions" — brokers, finders, auto dealers that extend credit, mortgage lenders, non-bank lenders, and the broader set of entities engaged in "financial activities" as defined. If your AI system processes "customer information" of a GLBA financial institution, the Safeguards Rule applies. Banks and credit unions are regulated by their prudential regulators (OCC, FDIC, NCUA, Federal Reserve), which have adopted substantively-equivalent Interagency Guidelines; broker-dealers and investment advisers are regulated by SEC Regulation S-P. The engineering posture is similar across the three, with the Safeguards Rule as the most prescriptive text.

### What the Safeguards Rule requires

The 2021 amendment was the first substantial update since the rule's inception and added specific technical requirements. Key obligations under 16 CFR 314:

- **Qualified individual** (§314.4(a)) — the organisation must designate a qualified individual to oversee the information security programme.
- **Risk assessment** (§314.4(b)) — written risk assessment identifying internal and external risks to the security, confidentiality, and integrity of customer information, including a periodic re-assessment. Expected content: criteria for evaluating and categorising risks, criteria for assessing confidentiality / integrity / availability, how risks will be mitigated, how the programme will respond to risks.
- **Safeguards based on the risk assessment** (§314.4(c)) — access controls, periodic inventory, encryption of customer information in transit and at rest, secure development practices, multi-factor authentication for anyone accessing customer information, secure disposal, change management, monitoring and logging of user activity.
- **Training** (§314.4(e)) — security awareness training for personnel.
- **Oversight of service providers** (§314.4(f)) — selection, contracting, and periodic assessment of service providers.
- **Evaluation and adjustment** (§314.4(g)) — periodic evaluation and adjustment of the programme.
- **Incident response plan** (§314.4(h)) — written plan for responding to a security event materially affecting customer information.
- **Board reporting** (§314.4(i)) — annual reporting to the board or senior governing body.

The 2023 amendment added:

- **Notification of a security event** (§314.5) — financial institutions must notify the FTC of a security event involving the unencrypted customer information of **500 or more consumers** as soon as possible, and no later than **30 days** after the event is discovered.

### SEC Regulation S-P 2024 amendment — the SEC overlay

The SEC's 2024 Regulation S-P amendment aligned broker-dealer and investment-adviser practice with the Safeguards Rule's substance and added its own incident-response and notification requirements, including:

- **Incident response programme** — written, covering detection, response, and recovery.
- **Customer notification** — notification of affected individuals within **30 days** after a sensitive-customer-information incident is determined to have occurred, with exceptions for law-enforcement-requested delays. Compliance deadlines phased: larger entities 30 November 2025 / 2 December 2025, smaller entities 2 June 2026 (confirm the current deadline on the SEC release at ingestion).
- **Service-provider oversight** — written agreements, selection, monitoring.

### AI-specific Safeguards / Reg S-P crossovers

AI systems in a GLBA / Reg S-P context are subject to the general rule content; the AI-specific concerns the written risk assessment and the controls should address:

- **Customer-information training corpora** — if training data is customer information, the lawful-basis / purpose question overlaps with GLBA's "clear and conspicuous" privacy-notice obligations and the opt-out for non-affiliated third-party sharing.
- **Vendor AI processors** — the service-provider oversight obligations apply to AI-model vendors (hosted-LLM providers, model-marketplace vendors). Written agreements, selection, monitoring.
- **Model-based customer decisions** — FCRA may overlay when the model's output is a "consumer report" (adverse-action notice, dispute-resolution), the Equal Credit Opportunity Act may overlay when the decision affects credit, and both carry their own obligations the DPIA should name.
- **Incident response for AI-specific threats** — the incident-response plan under §314.4(h) should include AI-specific detection triggers (anomalous model behaviour, prompt-injection indicators, unusual extraction-like query patterns).
- **The 30-day notification clock** — faster than HIPAA's 60 days; a privacy-engineering programme serving a GLBA / Reg S-P financial institution must operate to the 30-day clock.

### Cross-tagging GLBA / Reg S-P into the evidence pack

| GLBA / Reg S-P obligation | Artefact |
|---|---|
| §314.4(a) qualified individual | Programme-ownership memo; the qualified individual's name |
| §314.4(b) risk assessment | DPIA risk register + Safeguards-specific supplemental assessment |
| §314.4(c)(4) encryption | Encryption-at-rest policy; TLS policy |
| §314.4(c)(5) secure development | SDLC policy; code-review standards; AI-model release checklist |
| §314.4(c)(6) MFA | IAM policy and evidence |
| §314.4(c)(7) secure disposal | Data-destruction policy and evidence |
| §314.4(f) service-provider oversight | Vendor-management programme, DPA / BAA / service-provider contracts |
| §314.4(h) incident response | Breach-response plan with GLBA decision branch |
| §314.5 FTC notification (500+ consumers) | Incident-notification runbook with FTC pathway |
| Reg S-P 2024 customer notification | 30-day customer notification runbook |

An FTC examination opens with the risk assessment and the incident-response plan; an SEC exam under Reg S-P opens with the written programme and incident history. AI content is a growing exam focus.

## NIST SP 800-53 Rev. 5 — the PT family and the AI-touching PII controls

SP 800-53 Rev. 5 is the authoritative US-federal control catalogue. All federal information systems (and many federally-adjacent systems under FedRAMP, DoD CMMC, HIPAA-via-security, and state-regulatory expectations) are assessed against it. Rev. 5 introduced the PT (PII Processing and Transparency) control family — the privacy-specific controls — and strengthened the existing AC, AU, and SI families with privacy-relevant content.

### The PT family — eight controls at a minimum

The PT family as published in Rev. 5:

- **PT-1 — Policy and Procedures** — privacy policies and procedures with assigned responsibility and review cadence.
- **PT-2 — Authority to Process PII** — the authority, consent, legitimate basis, or other legal basis permitting processing.
- **PT-3 — PII Processing Purposes** — specific and articulated purposes; purpose binding on downstream processing; purpose change notified.
- **PT-4 — Consent** — consent mechanism where consent is the basis, with revocation.
- **PT-5 — Privacy Notice** — notice provided to the individual.
- **PT-6 — System of Records Notice and Privacy Act Statements** — SORN maintained; Privacy Act statement provided.
- **PT-7 — Specific Categories of PII** — special-category handling (equivalent to Art. 9).
- **PT-8 — Computer Matching Requirements** — specific to the Computer Matching Agreements framework under the Privacy Act.

Several PT controls have Rev-5 enhancements (e.g., PT-2(1) authorisation; PT-3(1) data-tagging; PT-5(2) privacy-notice presentation style) that an AI programme should examine against its deployment.

Beyond PT, the privacy-overlay controls touched most heavily by AI processing:

- **AC-3 — Access Enforcement**, **AC-16 — Security and Privacy Attributes**, **AC-21 — Information Sharing**.
- **AU-2 — Event Logging**, **AU-11 — Audit Record Retention**, **AU-12 — Audit Record Generation**.
- **IA-2 — Identification and Authentication (Organizational Users)**.
- **IR-4 — Incident Handling**, **IR-6 — Incident Reporting**, **IR-8 — Incident Response Plan**.
- **MP-6 — Media Sanitization** (NIST SP 800-88 referenced).
- **RA-3 — Risk Assessment**, **RA-7 — Risk Response**.
- **SI-10 — Information Input Validation**, **SI-12 — Information Management and Retention**, **SI-15 — Information Output Filtering**, **SI-19 — De-identification**, **SI-20 — Tainting**, **SI-23 — Information Fragmentation**.

The AI-specific controls that bite: **SI-19 (de-identification)** is where DP / K-anonymity / scrubbing lands; **SI-15 (output filtering)** is where AI output filters land; **SI-10 (input validation)** is where prompt-injection defences land; **SI-12 (retention)** is where corpus / log retention lands; **RA-3 (risk assessment)** is where the PRAM and DPIA artefacts land; **PT-2 / PT-3 / PT-5** are where the Art. 6 / purpose / notice artefacts from the GDPR side land under a different label.

### Cross-tagging the PT family into the evidence pack

A worked tag for an AI system against the PT family:

| Control | Artefact |
|---|---|
| PT-1 | Organisational privacy policy and the privacy-engineering operational procedures |
| PT-2 | Per-processing authority / lawful-basis memo (overlaps with GDPR Art. 6 column) |
| PT-3 | RoPA entry, purpose-specific feature gating documentation |
| PT-4 | Consent-management evidence (where consent is the basis) |
| PT-5 | Public privacy notice; Art. 13 / 14 notice for EU users |
| PT-6 | SORN (federal systems only) |
| PT-7 | Special-category handling policy; Art. 9 / Art. 10 handling note |
| AC-16 | Data-tagging schema (PII, PHI, restricted) and enforcement |
| AU-2 / AU-11 | Event-logging and retention policies, with PII-access events in scope |
| IR-8 | Breach-response plan (chapter 06) |
| MP-6 | Media sanitisation per SP 800-88; crypto-shred policy for cloud |
| SI-19 | DP training manifest, K-anonymity report, redaction FP/FN report |
| SI-15 | Output-filter design and evaluation report |
| SI-10 | Prompt-injection defence design and evaluation report |
| SI-12 | Retention matrix per store |
| RA-3 | DPIA + PRAM output |

The federal system or FedRAMP-aligned vendor records this mapping in its System Security and Privacy Plan (SSPP); the privacy engineer contributes the PT and SI-19 / SI-15 / SI-10 / SI-12 content.

### Baseline selection

SP 800-53B defines control baselines (Low / Moderate / High / Privacy Overlay). Federal systems inherit the Moderate or High baseline per FIPS 199 categorisation. The Privacy Overlay in SP 800-53B is the privacy-specific control set that *augments* the security baselines. An AI system handling federal PII is expected to apply the Moderate or High baseline *plus* the Privacy Overlay.

## State law, children's data, and sector-specific overlays the DPIA still owes

The three overlays above are the heaviest; the DPIA for an AI system should also engage, where applicable:

- **State privacy laws** — CCPA / CPRA (California), CDPA (Virginia), CPA (Colorado), CTDPA (Connecticut), UCPA (Utah), TDPSA (Texas), and the expanding bench. The CPRA's "automated decision-making technology" regulations (CPPA rulemaking, final rules adopted 2025) are particularly relevant for AI programmes; the DPIA may need to cite the controller's ADMT risk-assessment under the CPPA framework.
- **Children's data** — COPPA for under-13 users; state children's-code analogues (California AADC); the expanded CPRA protections for consumers under 16.
- **FCRA** — if the model's output is a "consumer report", the Fair Credit Reporting Act's permissible-purpose, adverse-action-notice, and dispute-resolution requirements apply.
- **ECOA / Regulation B** — equal credit opportunity; model-based credit decisions must be adverse-action-notice-compliant.
- **FDA** — for AI as a medical device, the Software-as-a-Medical-Device (SaMD) framework overlays the HIPAA privacy content with FDA safety content.
- **State AI laws** — Colorado AI Act (2024), the expanding state bench; algorithmic-discrimination obligations commonly include a specific risk-assessment artefact.

For each applicable overlay, the DPIA names the obligation and cites the cross-tagged artefact. The point of the chapter is not to produce a legal compendium — the controller's legal team does that — but to make sure the engineering evidence is organised so the legal team can answer the regulator's question without scavenging.

## The cross-tagging discipline — one artefact, multiple obligations

A single engineering artefact often satisfies obligations under multiple regimes. The retention matrix per store (one row per store, with the retention duration, the basis for the duration, and the enforcement mechanism) is cited under:

- GDPR Art. 5(1)(e) (storage limitation) and Art. 35(7)(a) (systematic description).
- HIPAA §164.316 (documentation) and §164.530(j) (six-year retention of documentation).
- GLBA Safeguards §314.4(c)(5) (secure disposal).
- SP 800-53 SI-12 (retention) and AU-11 (audit-log retention).
- CNIL retention sheet for AI.

The cross-tagging in the evidence-pack manifest is explicit: the artefact's entry carries every applicable control / obligation tag. A single pass through the artefacts, tagged against the composite set, satisfies every overlay. The evidence pack of chapter 06 is organised this way.

## The audit experience — what the examiner asks for

A HIPAA OCR audit, an FTC Safeguards examination, an SEC Regulation S-P examination, and a FedRAMP assessment all follow a similar rhythm:

1. **Policy pack** — the written policies and procedures; the roles; the training records.
2. **Risk-assessment artefact** — the current, documented, AI-aware risk analysis.
3. **Control evidence** — for each control in scope, the evidence of its operation (logs, configurations, screenshots, audit reports).
4. **Incident history** — the record of incidents, the response, the lessons learned.
5. **Vendor / service-provider oversight** — the DPAs / BAAs / service-provider contracts; the vendor-assessment records.
6. **Interview with the qualified individual / designated security official / privacy officer** — who can speak to the programme.
7. **Walkthrough of the system** — a demonstration of the controls in operation.

An AI programme ready for such an audit has its evidence organised by control identifier (the cross-tagged artefact repository), its risk analysis current to the last major model version, and its incident-history documented. The DPIA and PRAM output live in the risk-assessment slot; the bundle (chapter 06) lives in the control-evidence slot.

## Summary — what you can do after this chapter

- Cross-tag HIPAA Privacy / Security / Breach Notification Rule obligations into the AI privacy evidence pack when PHI is in scope, and run the breach-notification decision to the 60-day clock.
- Cross-tag GLBA Safeguards Rule and SEC Regulation S-P 2024 obligations when financial-institution customer data is in scope, and run the breach-notification decision to the 30-day clock.
- Cross-tag NIST SP 800-53 Rev. 5 PT family and the AI-touching AC / AU / SI controls into the evidence pack for federal and federally-adjacent systems.
- Recognise the state-law, children's-data, FCRA / ECOA, FDA, and state-AI-law overlays that may additionally apply and cite them in the DPIA where triggered.
- Organise the evidence pack by cross-tagged artefact so one pass serves every overlay.
- Prepare the DPIA + bundle for the audit rhythm that OCR / FTC / SEC / FedRAMP assessors use.

Chapter 06 assembles the full regulator-facing bundle — DPIA, DFD, PET deployment claims, MIA / extraction report, redaction report, unlearning-capability statement, logging / retention policy, breach-response plan — as the single package that answers an inspection.
