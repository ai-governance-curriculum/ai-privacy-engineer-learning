# Sector-Specific US Privacy Laws — HIPAA, FERPA, COPPA, GLBA

Comprehensive privacy laws (GDPR, CCPA, state matrix) set a floor. On top of that, **sector-specific** laws kick in when the AI system processes data in a regulated domain. Four matter most often for an AI Privacy Engineer: HIPAA (health), FERPA (education), COPPA (children), and GLBA (financial). This chapter is the **trigger-and-obligation map** for each.

Primary sources:

- **HIPAA Privacy Rule, Security Rule, Breach Notification Rule** — [HHS: HIPAA for Professionals](https://www.hhs.gov/hipaa/for-professionals/index.html) (45 CFR Parts 160 and 164). De-identification guidance: [HHS OCR Guidance Regarding Methods for De-identification of Protected Health Information](https://www.hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification/).
- **FERPA** — [20 U.S.C. § 1232g; 34 CFR Part 99](https://studentprivacy.ed.gov/node/548/) (US Dept. of Ed Student Privacy Policy Office).
- **COPPA** — [15 U.S.C. §§ 6501–6506; 16 CFR Part 312 (FTC COPPA Rule)](https://www.ftc.gov/business-guidance/privacy-security/childrens-privacy).
- **GLBA Safeguards Rule** — [16 CFR Part 314 (FTC Safeguards Rule)](https://www.ftc.gov/legal-library/browse/rules/safeguards-rule). The Privacy Rule and the Interagency Guidelines (banking) also matter per institution type.

> The AI Privacy Engineer's job is to **identify the trigger**, understand the **obligation shape**, and **design the engineering surface** that honours it. Legal reading of specific clauses — "is this fundraising activity under 164.514(f)?" — is counsel.

## HIPAA — when health data touches AI

### Who is covered

- **Covered entities** — health plans, healthcare clearinghouses, and healthcare providers that transmit any health information in electronic form in connection with a HIPAA-covered transaction.
- **Business associates** — a person or entity that performs functions or activities on behalf of, or provides services to, a covered entity that involves PHI access. Your AI vendor processing PHI is almost always a business associate.

If your AI system processes Protected Health Information (PHI) on behalf of a covered entity, your organisation is a business associate and needs a **Business Associate Agreement (BAA)** in place.

### What "PHI" actually is

Protected Health Information is individually identifiable health information held or transmitted by a covered entity or business associate, in any form. "Individually identifiable" has a specific regulatory meaning — the information identifies the individual or there is a reasonable basis to believe it can be used to identify them, *and* relates to past / present / future physical or mental health, healthcare, or payment for healthcare.

### Engineer-visible obligations

- **Privacy Rule** — minimum necessary for use / disclosure, authorisation requirements for non-treatment-payment-operations (TPO) uses, patient access rights (right of access to PHI, right to amend, right to accounting of disclosures). Patient access is the HIPAA analogue of GDPR Art. 15.
- **Security Rule** — administrative, physical, and technical safeguards for electronic PHI (ePHI). Technical safeguards (164.312) include access control, audit controls, integrity, authentication, transmission security.
- **Breach Notification Rule** — notification to affected individuals, HHS, and in some cases media; clocks vary by scope.

### De-identification — Safe Harbor vs. Expert Determination

HIPAA offers **two paths** to de-identify PHI, after which the data is no longer PHI and may be used more broadly:

- **Safe Harbor (45 CFR 164.514(b)(2))** — remove **18 specific identifier categories** (names, geographic subdivisions smaller than state with exceptions, dates narrower than year except year-of-birth / death, phone numbers, fax, email, SSN, medical record numbers, health plan beneficiary numbers, account numbers, certificate / license numbers, vehicle identifiers, device identifiers / serial numbers, URLs, IP addresses, biometric identifiers, full-face photos / comparable images, and any other unique identifying number / characteristic / code), *and* the covered entity does not have actual knowledge the remaining information could be used alone or in combination to identify an individual.
- **Expert Determination (45 CFR 164.514(b)(1))** — a person with appropriate statistical / scientific experience determines the risk of re-identification is very small, applies methods to achieve that, and documents the determination.

AI systems routinely need Expert Determination rather than Safe Harbor because the model's intended use needs more granular data than Safe Harbor permits (e.g. dates smaller than year, geography smaller than state).

**Limited Data Set (164.514(e))** is a third option — a smaller identifier-stripped dataset under a Data Use Agreement — useful for research / public-health / healthcare-operations use.

### AI-system engineering consequence

- **Minimum necessary** → training-corpus minimisation, retrieval-ACL scoping, prompt-sink redaction.
- **Access + audit** → per-user audit log on every PHI access path, including LLM prompts and retrieval hits.
- **Authentication + transmission security** → TLS 1.3, SSO / MFA, KMS-managed keys, encryption at rest.
- **De-identification** → engineering the Safe-Harbor pipeline **or** the Expert-Determination evaluation: statistical disclosure control (k-anonymity / l-diversity / t-closeness), tested against motivated-insider and public-dataset-linkage scenarios. The toolkits (ARX, sdcMicro) and the PII pipeline live in `mod-104-pii-detection-and-redaction`.
- **BAA** → not an engineer's doc to draft, but every PHI-touching vendor in your dependency graph must have one. Keep the vendor-register current (`ai-governance-analyst` scope) and your team's BAA inventory honest.
- **Breach notification** → detect fast enough to notify on the regulatory clock. Breach of unsecured PHI triggers the clock; **encryption and destruction** under HHS guidance renders PHI "secured" (safe-harbor from breach-notification obligation, distinct from the privacy-rule Safe Harbor). This is why at-rest encryption of PHI with proper key management is a load-bearing control.

### Where AI specifically trips HIPAA

- **Training on PHI** → BAA in place, minimum necessary, access controls, audit trail; consider training on de-identified data instead.
- **PHI in prompts** → prompt sink is a PHI store; access control and retention apply.
- **PHI in retrieval corpora** → RAG-tenant leakage is a HIPAA disclosure if cross-tenant. See `mod-109-rag-privacy-and-agent-scoping`.
- **PHI in model outputs** → the model can reconstruct PHI from training (memorisation) — this is a HIPAA concern not just a GDPR one. MIA / extraction harness (`mod-105-memorisation-extraction-and-mia`) is the defender's evidence.
- **De-identified-then-re-identified** → if your model inadvertently re-identifies a de-identified record, you're back inside HIPAA.

### Control / test / evidence

| HIPAA obligation | Engineering control | Test | Evidence |
|---|---|---|---|
| Minimum necessary | Feature allow-list; retrieval scope by role / clinical context | Access-review audit; unused-field review | Minimisation review doc |
| Access + audit | Audit log on every PHI access incl. LLM prompt / retrieval | Audit-log completeness check | Access-log export |
| Encryption at rest | KMS-managed keys; column-level encryption for ePHI | At-rest-encryption scan | Encryption audit report |
| De-identification | Safe Harbor pipeline **or** Expert Determination | Re-identification probe (k, l, t) | Safe-Harbor checklist or Expert-Determination memo |
| BAA chain | Vendor register with BAA status | Quarterly BAA audit | BAA-status log |
| Breach detection / notification | Monitoring + incident runbook | Tabletop exercise | Incident log + tabletop output |

## FERPA — when student data touches AI

### Scope

FERPA (20 U.S.C. § 1232g; 34 CFR Part 99) protects **education records** maintained by educational agencies and institutions that receive US Department of Education funds. It gives parents (and eligible students, once they reach 18 or enter postsecondary education) rights of access, amendment, and consent to disclosures.

### What is an "education record"

Records directly related to a student that are maintained by the educational agency / institution (or by a party acting for the agency / institution). Includes transcripts, grades, disciplinary records, class rosters. Specifically excludes sole-possession records of individual staff not shared with others.

### Engineer-visible obligations

- **Consent before disclosure** of PII from education records, subject to exceptions.
- **Directory information** — a category the institution can designate (name, address, telephone, etc.) and disclose without consent, with opt-out notice.
- **School officials exception** — a party with a legitimate educational interest can act as a school official and receive education records without consent, under conditions (direct control, limited use, security).
- **Studies exception (34 CFR 99.31(a)(6))** — disclosures to an organisation conducting studies for or on behalf of the institution, under a written agreement.
- **Audit / evaluation exception (34 CFR 99.31(a)(3))** — disclosures to authorised representatives for audit / evaluation of federally supported education programs.
- **Right of access to education records** by parents / eligible students.

### AI-system engineering consequence

- An AI vendor processing education records on behalf of a school is typically brought in as a **school official** or under a **written agreement** exception. The written agreement sets the use limits, security, and re-disclosure prohibitions.
- Directory information is *not* a free pass to use for AI training unless the written agreement allows it.
- Student rights of access / correction need an engineering path — similar to GDPR Art. 15 / 16 shape, but with FERPA-specific scope.
- **Breach** — FERPA does not have a federal breach-notification clock like HIPAA, but state laws and institutional policy do.

### Overlap with other laws

- **Children under 13** — COPPA (below) *also* applies when the AI service collects personal information from a child, even if FERPA already covers the record.
- **Students in healthcare settings** — HIPAA / FERPA overlap is specifically addressed in HHS / ED joint guidance; in general, education records are governed by FERPA.
- **State student-privacy laws** — several US states (e.g. California's SOPIPA) layer additional obligations on edtech vendors; check the state matrix for the specific state.

### Control / test / evidence

| FERPA obligation | Engineering control | Test | Evidence |
|---|---|---|---|
| Written-agreement / school-official | Contract clauses: use scope, security, re-disclosure prohibition | Contract audit | Signed agreement |
| Consent / exception tracking | Per-disclosure basis field in the AI-system register | Disclosure-log audit | Register export |
| Right of access to records | Student-portal API for records held | E2E access-request test | Access-log |
| Security | Encryption; access control; training | Access review | Access-review log |

## COPPA — when children's data touches AI

### Scope

COPPA (15 U.S.C. § 6501–6506; 16 CFR Part 312, as updated by FTC rulemaking) applies to operators of websites or online services (including mobile apps, IoT, and AI services) directed to children under 13, or operators with actual knowledge they collect personal information from children under 13.

The FTC has periodically amended the COPPA Rule; the 2025 amendments tightened several provisions, including data-retention and third-party-disclosure consent. <!-- needs-research: refresh the COPPA Rule amendment status and effective dates against 16 CFR Part 312 and FTC guidance at each curriculum review. -->

### Engineer-visible obligations

- **Notice** — a clear and comprehensive privacy notice on the website / service.
- **Verifiable parental consent (VPC)** — operators must obtain VPC before collecting, using, or disclosing personal information from children. The Rule lists acceptable VPC methods.
- **Right to review / delete** the child's personal information.
- **Data-minimisation** — only collect what is reasonably necessary.
- **Data retention** — retain only as long as reasonably necessary; delete when no longer necessary.
- **Data security** — reasonable procedures.
- **Third-party disclosure** — subject to parental consent, with specific consent for categories like targeted advertising.

### AI-system engineering consequence

- **Scoping** — is the AI service *directed to* children, or do you have *actual knowledge* children under 13 are using it? Both are triggers. Age-gating on a general-audience service is a common mitigation.
- **Training-data provenance** — if a scraped corpus includes material from children's sites or where children are depicted, you may have a COPPA footprint. This is the AI-specific COPPA risk the plain reading of the Rule did not anticipate.
- **Chat / generative output** — a chat feature that collects PII from a child without parental consent is a violation. Age-gate the chat or ensure VPC.
- **Audio / image collection** — includes voice recording of a child, image of a child. The Rule treats these as personal information in the child's context.
- **No ads to children without VPC** — specifically relevant if your AI service has ad-personalisation layered on.

### Control / test / evidence

| COPPA obligation | Engineering control | Test | Evidence |
|---|---|---|---|
| Age gate | Age-verification UX; service-restriction when under 13 without VPC | UX test; age-gate bypass audit | Age-gate design doc |
| VPC | VPC workflow + records | VPC workflow test | VPC record per child |
| Notice | Child-specific privacy notice | Notice-rendering test | Rendered notice |
| Minimisation | Field allow-list; no voice recording without VPC | Field audit; voice-recording audit | Minimisation doc |
| Retention / deletion | Retention jobs; right-to-review-and-delete path | E2E deletion test | Deletion log |

## GLBA — when financial data touches AI

### Scope

The Gramm-Leach-Bliley Act applies to **financial institutions** as defined — banks, securities firms, insurance companies, and non-bank financial companies (including many fintechs and some payment-adjacent services). GLBA has three main components: the Privacy Rule (notice, opt-out for sharing with non-affiliated third parties), the Safeguards Rule (security), and the Pretexting provisions.

The **FTC Safeguards Rule (16 CFR Part 314)** is the one AI Privacy Engineers most often encounter — it applies to non-bank financial institutions under FTC jurisdiction and prescribes a written information security program. Depository institutions are regulated by their federal banking agency under the Interagency Guidelines, which run in parallel.

### Engineer-visible Safeguards-Rule obligations

- **Written information security program** reasonably designed to protect customer information.
- **Qualified Individual** responsible for implementation.
- **Risk assessment** — periodic, written, considering reasonably foreseeable internal and external threats.
- **Controls to mitigate identified risks**, specifically including:
  - access controls — including authentication (MFA for any individual accessing any information system or a reasonable equivalent),
  - inventory of data, personnel, devices, systems, facilities,
  - encryption at rest and in transit,
  - secure development practices,
  - access reviews,
  - disposal of customer information no longer necessary,
  - change management,
  - monitoring and logging of authorised user activity,
  - testing (continuous monitoring, annual pen test, biannual vulnerability assessment at minimum),
  - security awareness training,
  - oversight of service providers.
- **Incident response plan** — written, specific elements required by the Rule.
- **Board / senior-officer reporting** at least annually.

### AI-system engineering consequence

- An AI system processing customer financial information falls under the full Safeguards Rule — MFA, encryption, logging, pen-test, incident response — just like any other system in scope. AI does not relax any of these.
- **Access-control on model outputs** — if your model outputs customer information, those outputs are inside the Rule.
- **Service-provider oversight** — if you use a foundation-model API as a service provider, you need a written risk-based oversight program for it (contract, assessment, monitoring).
- **Incident response** — AI-specific privacy incidents (prompt-injection exfiltrating customer data, model-extraction attack) should be in the incident-response plan.
- **Risk assessment** — the Rule's risk-assessment requirement overlaps with GDPR Art. 35 / CPPA risk assessment; design the DPIA template to be reusable.

### Control / test / evidence

| Safeguards obligation | Engineering control | Test | Evidence |
|---|---|---|---|
| MFA | SSO / MFA for all info-system access | MFA-coverage audit | Access audit |
| Encryption | At-rest + in-transit encryption everywhere customer info flows | Encryption scan | Encryption report |
| Logging / monitoring | Auth-user activity log; anomaly monitoring | Log-completeness check | Log sample |
| Pen-test (annual) | Scheduled pen-test, incl. AI-surface coverage | Pen-test report review | Pen-test report |
| Vulnerability assessment (biannual) | Scheduled vuln scan | Vuln-scan ticket flow | Vuln-scan report |
| Secure development | SAST / SCA in CI; secure-coding training | Build-gate | CI logs |
| Service-provider oversight | Vendor-risk program incl. foundation-model vendors | Vendor-review audit | Vendor-assessment file |
| Incident response | IR plan with AI-privacy scenarios | Tabletop exercise | Tabletop report |
| Board reporting | Annual report template | Board calendar | Report deck |

## The trigger map — what to check first

When product tells you about a new AI use case, run this trigger checklist:

| If the system processes ... | ... then also check ... |
|---|---|
| Health information from a provider / payer / clearinghouse | HIPAA (BAA, de-identification path, audit, breach clock) |
| Student / educational-record information | FERPA (written agreement / school-official exception, state student-privacy laws) |
| Any data from users under 13 (actual knowledge) **or** service directed to children | COPPA (VPC, notice, retention) |
| Customer non-public personal information at a financial institution | GLBA Safeguards Rule (full program) |
| EU personal data | GDPR (chapter 02) |
| California-resident personal information | CCPA / CPRA / ADMT (chapter 04) + state matrix |
| Biometric data for identification | GDPR Art. 9; US state biometric laws (BIPA in IL is the most litigated — <!-- needs-research: expand this list with the current enacted biometric-privacy statutes --> ) |

If two or more triggers fire, the obligations compose — build to the strictest.

## Where sector laws stop and this track picks up

Sector laws tell you *what* to protect. This track teaches you *how* to protect it in AI systems:

- Minimum-necessary / minimisation → `mod-103-data-governance-and-minimisation`.
- De-identification pipelines → `mod-104-pii-detection-and-redaction`.
- Memorisation / extraction risk to PHI / student / child / financial data → `mod-105-memorisation-extraction-and-mia`.
- DP as a privacy-enhancing control satisfying "reasonable security" in several readings → `mod-106-differential-privacy-engineering`.
- RAG / agent-scope controls for cross-tenant PHI / FERPA records → `mod-109-rag-privacy-and-agent-scoping`.
- DSR / right-of-access / right-of-erasure engineering for all sectors → `mod-110-machine-unlearning-and-dsr`.
- The regulated-use-case DPIA bundling → `mod-111-dpia-and-privacy-reviews`.

## Depth boundary

- **Legal interpretation of sector clauses** — "does this specific use fall under the HIPAA treatment / payment / operations exception?" — is counsel + the Privacy / Compliance Officer.
- **SOC-side breach handling** — forensics, SOC-triggered IR — is `security-learning` / `ai-infra-security` (level 35) scope; this track feeds the AI-specific privacy signal.
- **Operational vendor-register upkeep** including BAA inventory — `ai-governance-analyst` (level 15) scope.

## Summary

- HIPAA applies when PHI touches AI — BAA, minimum necessary, access + audit, de-identification (Safe Harbor or Expert Determination), breach-notification clock.
- FERPA applies when student records touch AI — written-agreement / school-official exception, student-access, no-redisclosure.
- COPPA applies when children's data (under 13) touches AI — VPC, notice, minimisation, no targeted ads without consent.
- GLBA Safeguards applies when customer non-public personal info at a financial institution touches AI — full written information security program, MFA, encryption, pen-test, incident response, service-provider oversight, board reporting.
- When triggers compose, build to the strictest. Design the DPIA / risk-assessment template to absorb every regime you encounter.

Next: PIPL, PIPEDA, LGPD, OECD, EDPB Opinion 28/2024, and CNIL AI sheets.
