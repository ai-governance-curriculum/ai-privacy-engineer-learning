# International Privacy Regimes and Regulator Guidance on AI

An AI system is rarely scoped to one jurisdiction. Beyond GDPR and the US patchwork, you need working fluency in the three next-most-consequential comprehensive regimes — **PIPL (China)**, **PIPEDA (Canada)**, **LGPD (Brazil)** — the **OECD Privacy Guidelines** as the common-denominator reference, and the two most specific regulator statements on AI + GDPR: **EDPB Opinion 28/2024** and the **CNIL AI sheets**.

Primary sources:

- **PIPL** — [Personal Information Protection Law of the People's Republic of China (2021)](http://www.npc.gov.cn/npc/c30834/202108/a8c4e3672c74491a80b53a172bb753fe.shtml). English-translation summaries are widely published; the text is the authority.
- **PIPEDA** — [Personal Information Protection and Electronic Documents Act (Canada)](https://www.priv.gc.ca/en/privacy-topics/privacy-laws-in-canada/the-personal-information-protection-and-electronic-documents-act-pipeda/). Reform via **Bill C-27 (CPPA / AIDA)** has been proceeding through Parliament; always refresh status against the Office of the Privacy Commissioner of Canada before citing the reform text as law. <!-- needs-research: refresh PIPEDA reform (Bill C-27 / CPPA / AIDA) status at each curriculum review cycle. -->
- **LGPD** — [Lei Geral de Proteção de Dados (Lei nº 13.709/2018)](https://www.gov.br/anpd/pt-br/documentos-e-publicacoes/lgpd-em-portugues) via the ANPD.
- **OECD Privacy Guidelines** — [Guidelines on the Protection of Privacy and Transborder Flows of Personal Data (1980, revised 2013)](https://www.oecd.org/digital/ieconomy/privacy-guidelines.htm).
- **EDPB Opinion 28/2024 on AI models** — on the processing of personal data in the context of AI models (adopted 17 December 2024). [European Data Protection Board — Opinions](https://www.edpb.europa.eu/our-work-tools/consistency-findings/opinions_en).
- **CNIL AI sheets** — the French data protection authority's AI how-to sheets, [CNIL AI how-to sheets](https://www.cnil.fr/en/artificial-intelligence) (recommandations sur l'IA). Updated iteratively.

> These statutes and guidance documents move. Pull current text at each review.

## PIPL — the engineer's reading

PIPL (2021) is the PRC comprehensive personal-information protection law. It borrows GDPR shapes with distinctive PRC additions.

### Key shape

- **Scope** — personal information of natural persons within the PRC; extraterritorial for providing products / services to PRC data subjects, analysing their behaviour, or other circumstances specified by law.
- **Lawful bases** — consent is primary; other bases include contract necessity, legal obligations, emergency, news-reporting / supervision in the public interest, publicly disclosed information within reasonable scope, and other bases provided by law. Note that *legitimate interest* is **not** a PIPL basis in the GDPR sense — this is a meaningful design difference.
- **Sensitive personal information** — defined category: biometric information, religious beliefs, specific identity, medical health, financial accounts, individual location tracking, personal information of minors under 14, etc. Requires specific consent and an impact assessment.
- **Cross-border transfer** — one of: a government-approved security assessment (for personal information handlers meeting certain scale / sensitivity thresholds), **certification** from a specialised body, **Standard Contract** (the PRC equivalent of SCCs), **or** another legally stipulated basis. CAC rules operationalise this; **regulated-industry-specific routing (critical information infrastructure operator) adds requirements**. <!-- needs-research: refresh CAC implementation measures and Standard Contract current form at each review cycle. -->
- **Rights of data subjects** — informed, decide, access, copy, portability (where conditions met), correction, deletion, explanation of handling rules, right to refuse automated decision-making.
- **Automated decision-making (Art. 24)** — handlers must ensure transparency and fair / just results; may not apply unreasonable differential treatment in transaction prices. Individuals have the right to explanation and to refuse decisions made *solely* via automated means where the decision has significant impact on their rights.
- **Personal information protection impact assessment (PIPIA)** — required for sensitive-personal-information processing, automated decision-making, entrusted processing, provision to other handlers, public disclosure, cross-border transfer, and other processing with significant impact.
- **Designated representative** — overseas handlers must designate a representative in the PRC.

### AI-system engineering consequence

- **Consent engineering** — PIPL consent is primary and *specific*; your consent register and revocation path must handle PIPL granularity (separate consent for sensitive PI, for cross-border transfer, for automated decision-making).
- **Automated-decision explain + refuse** — Art. 24 gives individuals the right to refuse ADM where significant impact exists; you need the same human-review surface as GDPR Art. 22 / CPPA ADMT.
- **Cross-border** — the Chinese data-residency story is meaningfully stricter than GDPR Chapter V. If training-data sourcing, inference, or logging flows cross the border, you need a documented transfer mechanism **and** a security assessment for in-scope operations.
- **Data localisation** — critical information infrastructure operators (CIIOs) and certain data handlers have additional localisation obligations; a PRC-market AI system often has to physically keep data in-country.

### Depth boundary

Detailed CAC filings, the Standard Contract filing procedure, security-assessment preparation — the regulatory legwork — is `ai-governance-analyst` + counsel. The AI Privacy Engineer owns the technical surface: consent-management, localisation of data pipelines, PIPIA content, ADM refusal mechanism.

## PIPEDA — the engineer's reading

PIPEDA governs the collection, use, and disclosure of personal information in commercial activities by federally regulated / cross-border organisations in Canada (substantially similar provincial laws replace it in Quebec, British Columbia, and Alberta for in-province commercial activity).

### Key shape

PIPEDA builds on **ten Fair Information Principles** incorporated into Schedule 1 (adapted from the CSA Model Code / OECD Guidelines):

1. Accountability
2. Identifying purposes
3. Consent
4. Limiting collection
5. Limiting use, disclosure, and retention
6. Accuracy
7. Safeguards
8. Openness
9. Individual access
10. Challenging compliance

The ten principles map cleanly onto the GDPR Art. 5 / 25 / 32 / 15 shape — with the Canadian flavour that **meaningful consent** is the primary basis and the Office of the Privacy Commissioner issues periodic guidelines on what "meaningful" requires for specific technologies.

**Breach notification** (PIPEDA Part 1, Division 1.1, since 2018) — notify the Privacy Commissioner, affected individuals, and other organisations / government institutions, where a breach creates a **real risk of significant harm**. Maintain breach records for two years.

### Reform — CPPA / AIDA

Bill C-27 would replace the private-sector part of PIPEDA with the **Consumer Privacy Protection Act (CPPA)** and introduce the **Artificial Intelligence and Data Act (AIDA)**. AIDA proposes high-impact-system obligations in Canadian federal commerce. **Reform status moves** — do not treat CPPA / AIDA as law without confirming at the current session. <!-- needs-research: refresh CPPA / AIDA Parliamentary status at each curriculum review cycle. -->

### Provincial laws — Quebec Law 25

Quebec's Act to Modernize Legislative Provisions as Regards the Protection of Personal Information (commonly called **Law 25**, bill 64 in enactment) phased in obligations between 2022–2024 that include a **right of data portability**, a **privacy officer** role, **privacy impact assessments**, **consent for profiling / location / biometrics**, and specific obligations for **automated decision-making** including a right to request human review. If your AI system touches Quebec residents, treat Law 25 as a distinct obligation set on top of PIPEDA.

### Engineering consequence

- Treat PIPEDA as GDPR-adjacent in practice — you can reuse most of the GDPR engineering surface with the addition of breach-notification routing to the OPC where "real risk of significant harm" is triggered.
- Quebec Law 25 adds a profiling / ADM consent surface and a PIA requirement — route them through your existing DPIA pipeline.

## LGPD — the engineer's reading

LGPD (Lei nº 13.709/2018) is Brazil's comprehensive personal-data protection law, enforced by the ANPD. It is widely characterised as GDPR-shaped with Brazilian-specific adaptations.

### Key shape

- **Lawful bases (Art. 7)** — ten bases including consent, compliance with legal obligation, execution of public policies, research by research entities (anonymising where possible), contract performance, regular exercise of rights in legal / arbitration / administrative procedures, protection of life / physical safety, health protection, legitimate interests, and credit protection.
- **Sensitive personal data (Art. 11)** — defined category (racial / ethnic origin, religious conviction, political opinion, union / religious / philosophical / political organisation membership, data on health / sex life, genetic / biometric data linked to a natural person). Specific bases apply.
- **Rights of the data subject (Art. 18)** — confirmation of processing, access, correction, anonymisation / blocking / deletion of unnecessary / excessive / unlawfully-processed data, data portability, deletion of personal data processed with consent, information on sharing, information on the possibility of not consenting and consequences, revocation of consent.
- **Automated-decision review (Art. 20)** — the data subject has the right to request review of decisions made solely on the basis of automated processing affecting their interests. The reviewer must be a human.
- **Controller / processor / DPO (Encarregado)** — role structure parallels GDPR; the DPO is called *Encarregado*.
- **International transfer (Art. 33)** — allowed under adequacy decisions, standard contractual clauses, binding corporate rules, consent, and other narrow bases.
- **Data protection impact report (Art. 38 — RIPD / DPIA)** — can be required by ANPD for processing based on legitimate interests or where high risk is identified.

### Engineering consequence

- LGPD's ADM-review (Art. 20) is a specific human-review surface — build it once and route GDPR Art. 22, CPPA ADMT, PIPL Art. 24, and LGPD Art. 20 to the same pipeline.
- Breach notification to ANPD in a reasonable time (ANPD guidance sets specific windows — **refresh against current ANPD resolution** <!-- needs-research --> ).
- Child data handled with specific consent rules and best-interest-of-the-child provisions.

## OECD Privacy Guidelines — the common denominator

The OECD Privacy Guidelines (1980, revised 2013) are **not** a law — they are the common denominator from which most modern privacy statutes draw. The eight basic principles:

1. Collection Limitation
2. Data Quality
3. Purpose Specification
4. Use Limitation
5. Security Safeguards
6. Openness
7. Individual Participation
8. Accountability

Why you memorise them: when you face an exotic jurisdiction your team has never encountered, you can read its new privacy law *against* the OECD principles and quickly identify what's familiar and what's novel. The OECD eight-principle crosswalk also maps cleanly onto NIST PF, ISO/IEC 29100, and the GDPR principles — see `chapter 07`.

## EDPB Opinion 28/2024 on AI models

Adopted **17 December 2024**, Opinion 28/2024 is the European Data Protection Board's view on key questions raised by the CNIL about the processing of personal data in the development and deployment of AI models.

### The four questions

The opinion addresses:

1. **When can an AI model be considered "anonymous"?** — an AI model trained on personal data is anonymous only when, taking into account all means reasonably likely to be used, the likelihood of directly or indirectly identifying individuals from the model (including via memorisation, extraction, inference attacks) is **insignificant**. Supervisory authorities must assess this on a case-by-case basis.
2. **Can legitimate interests (GDPR Art. 6(1)(f)) serve as a legal basis for developing an AI model?** — a three-step test applies: (a) legitimate interest is pursued, (b) processing is necessary, (c) balancing against the rights and freedoms of data subjects. The interests of the data subject must be weighed carefully, especially for scraped data, children's data, and sensitive-category data.
3. **Can legitimate interests serve as the legal basis for deploying an AI model that was developed through personal-data processing?** — similar three-step test; a *development* illegality may contaminate *deployment* in some scenarios.
4. **What are the consequences if the AI model was developed by unlawfully processing personal data?** — depends on whether the deployer is the same controller, whether anonymisation can be shown, and the circumstances; the opinion walks three scenarios.

### Engineering consequence — translating the opinion

- **"Anonymous model" is a technical claim you have to defend.** Your MIA / extraction / inversion / attribute-inference harness (`mod-105`) *is* the evidence base. Opinion 28/2024 makes privacy testing a necessity, not a nice-to-have, for any claim of anonymity.
- **Legitimate-interests three-step test** needs an engineer-authored **LIA (Legitimate-Interests Assessment)** with real artifacts — necessity analysis (why you can't do this with less / different data), balancing test (with evidence that safeguards are in place), and reasonable-expectations analysis (would the data subject reasonably expect this use).
- **Scraped-data training** is the opinion's hot case. Documented provenance, data-minimisation at scrape time, respect for robots.txt + reasonable expectations, and transparent opt-out are the baseline technical mitigations.
- **Contamination of deployment by unlawful development** — this is where the privacy slice of release-assurance (`ai-evaluation-engineer` scope at level 35) picks up: if the model's lawful basis for development is shaky, the deployer inherits that risk.

Operationalise Opinion 28/2024 by adding these artifacts to your model's privacy pack:

- An **Anonymisation Claim Memo** with the MIA / extraction evidence that supports the "insignificant likelihood of re-identification" judgement.
- A **Legitimate-Interests Assessment** per processing purpose where 6(1)(f) is used, cross-referencing the three-step test.
- A **Data Provenance Statement** for the training corpus with the scraping / licensing / public-availability chain.

## CNIL AI sheets

The **Commission Nationale de l'Informatique et des Libertés** (CNIL, French DPA) has issued a series of **AI how-to sheets** covering the lifecycle of AI systems under GDPR. The set includes sheets on determining the applicable legal framework, defining the purpose, determining the role (controller vs. processor), defining the legal basis, verifying the composition of training data, retention periods, managing data subject rights for AI, handling scraping, and announcing AI development to data subjects. The sheet set evolves. <!-- needs-research: refresh the enumeration and exact titles against CNIL AI publications at each curriculum review cycle. -->

### Engineering consequence

The CNIL sheets are the most **engineer-actionable** regulator guidance published to date. Treat them as a checklist of operational decisions the DPA expects to see documented:

- Scope decisions (purpose, role, phase).
- Data decisions (composition of training data, retention, provenance, scraping method).
- Rights decisions (access, correction, erasure, objection, portability — for AI specifically).
- Transparency decisions (announcement to data subjects before scraping).
- Risk decisions (DPIA, residual risks, consultation with CNIL where high residual risk).

Package the CNIL-sheet-answers alongside your DPIA. If your supervisory authority is any European DPA, CNIL-style documentation is defensible by default.

## Composition across regimes

**You cannot build a separate pipeline per jurisdiction.** You build **one** engineering surface that satisfies the strictest obligation on each dimension:

| Engineering dimension | Strictest regime driver (common case) | Engineering target |
|---|---|---|
| Consent granularity | PIPL + Quebec Law 25 | Separate consent per purpose, with revocation that propagates |
| Automated-decision human-review | GDPR Art. 22 + CPPA ADMT + PIPL Art. 24 + LGPD Art. 20 + Quebec Law 25 | Single ADM inventory + human-review path + meaningful-information artifact |
| Rights-handling | GDPR + state matrix + PIPEDA + LGPD | Unified DSR intake with per-regime SLAs |
| Cross-border transfer | PIPL + GDPR | Transfer-mechanism register, SCC / Standard Contract execution, security assessment |
| Breach notification | GDPR Art. 33 (72h) + state laws + PIPEDA + LGPD + HIPAA | Detection + routing table keyed on affected residency |
| Anonymisation claim | EDPB Opinion 28/2024 | MIA / extraction harness + Anonymisation Claim Memo |
| Legitimate-interests basis | GDPR + EDPB Opinion 28/2024 | Documented LIA per purpose |
| DPIA / PIA / PIPIA / RIPD | GDPR Art. 35 + PIPL + LGPD + Quebec | DPIA template that fans into each regime's required content |

## Depth boundary

- **Legal reading of international statutes** — including the CJEU / Chinese / Brazilian / Canadian / French case law — is counsel. You consume their reading.
- **Jurisdictional tracking** — "when did ANPD issue its new resolution on children's data" — is `ai-governance-analyst` (level 15) scope. You should know the shape; your analyst colleagues own the daily tracking.
- **Cross-jurisdiction harmonisation at policy-library scale** — designing one control library that discharges all of the above — defers up to `senior-ai-governance-architect` (level 50).

## Summary

- PIPL, PIPEDA, LGPD, and the OECD Guidelines are the next-most-consequential regimes beyond GDPR and US laws — engineer to a strictest-common-denominator surface rather than a jurisdiction-per-pipeline.
- EDPB Opinion 28/2024 makes technical privacy testing (MIA / extraction) a direct input to the "anonymous model" question — your `mod-105` harness is now a regulatory artifact.
- CNIL AI sheets are the most engineer-actionable regulator guidance currently published; use them as a documentation checklist for European-jurisdiction AI systems.
- Keep the primary texts and regulator guidance refreshed at each cycle — these move.

Next: NIST Privacy Framework and ISO/IEC 27701 / 29100 — the enterprise-privacy-program shapes.
