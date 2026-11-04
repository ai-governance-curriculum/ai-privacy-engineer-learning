# The Evidence Crosswalk — One Evidence Pack, Many Audits

A mature AI privacy programme will be audited against several frameworks in the same fiscal year — NIST Privacy Framework against a US business line, ISO/IEC 27701 against a customer that requires the certification in the DPA, ISO/IEC 42001 against an enterprise AI management system, Cloud Security Alliance AI Controls Matrix against a cloud customer's AI governance review, SOC 2 common-criteria overlaps, EU AI Act Article-10 and Article-15 evidence against a regulator. The characteristic failure of a privacy programme is **assembling a fresh evidence pack per audit**. The characteristic win is a single, versioned evidence pack with a **crosswalk** — a table that says, for each framework and each control or subcategory, which artefact in the pack discharges it.

This chapter walks the crosswalk. The deliverable the exercise produces is a working crosswalk for the artefacts the preceding modules have taught you to make. The authoring discipline is to keep the crosswalk tight — one artefact can hit multiple frameworks, but the mapping must be honest. A crosswalk that pretends an artefact discharges a control it only partially covers is a crosswalk that will fail its first real audit.

The authoritative references for the four frameworks the crosswalk's left columns:

- NIST — *Privacy Framework v1.0* (January 2020) with the Core (Govern-P / Identify-P / Control-P / Communicate-P / Protect-P), Profiles, and Tiers. [nist.gov/privacy-framework](https://www.nist.gov/privacy-framework); crosswalks to GDPR and CCPA at [nist.gov/privacy-framework/resource-repository/browse](https://www.nist.gov/privacy-framework/resource-repository/browse).
- ISO/IEC 27701:2019 — *Privacy Information Management System (PIMS)* extension to 27001 / 27002. [iso.org/standard/71670.html](https://www.iso.org/standard/71670.html). (Note: ISO/IEC 27701:2025 is the revised edition published in 2025; many organisations are mid-migration.)
- ISO/IEC 42001:2023 — *Artificial Intelligence Management System (AIMS)*, Clauses 4 to 10 plus Annex A controls and Annex B implementation guidance. [iso.org/standard/81230.html](https://www.iso.org/standard/81230.html).
- Cloud Security Alliance — *AI Controls Matrix (AICM)*. [cloudsecurityalliance.org/research/working-groups/artificial-intelligence/](https://cloudsecurityalliance.org/research/working-groups/artificial-intelligence/).

Companion references that the crosswalk may extend into:

- NIST — *AI Risk Management Framework* (AI RMF 1.0, 2023) — Govern / Map / Measure / Manage. [nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework).
- NIST — *SP 800-53 Rev. 5* control catalogue (especially PT family). [csrc.nist.gov/publications/detail/sp/800-53/rev-5/final](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final).
- EU AI Act Regulation 2024/1689, Article 10 (data and data governance) and Article 15 (accuracy, robustness and cybersecurity). [eur-lex.europa.eu/eli/reg/2024/1689/oj](https://eur-lex.europa.eu/eli/reg/2024/1689/oj).
- AICPA — *SOC 2 / Trust Services Criteria* (TSC) 2017 including the 2022 revisions. [aicpa-cima.com](https://www.aicpa-cima.com/).

Verify the exact clause numbering against the current published version of each framework before stamping an audit-facing crosswalk — framework revisions renumber clauses with some regularity.

## The crosswalk as an engineering object

The crosswalk is a table — one you check into version control, one that has a schema, one that has an owner. Its minimal columns:

| Framework | Control / subcategory / clause ID | Short name | Evidence artefact(s) | Artefact owner | Last verified | Residual gap | Audit observations |
|---|---|---|---|---|---|---|---|

The semantics of each column:

- **Framework** — NIST PF / ISO 27701 / ISO 42001 / CSA AICM / AI RMF / 800-53 / EU AI Act / SOC 2.
- **Control / subcategory / clause ID** — the official identifier from the framework's catalogue.
- **Short name** — a human-readable tag.
- **Evidence artefact(s)** — paths to the documents that discharge the control. Multiple artefacts may be listed; each must be under version control.
- **Artefact owner** — the role responsible for keeping the artefact current (privacy engineer, ML lead, platform lead, DPO).
- **Last verified** — the date the artefact was last confirmed to still discharge the control. Evidence currency is a real audit finding category.
- **Residual gap** — a short note if the artefact only partially discharges the control; references the plan to close the gap.
- **Audit observations** — findings from past audits touching this row, with citation.

### Why "evidence artefact" is the right primitive

An artefact is a specific file, document, or data object you can point at. "Our policy is …" is not an artefact; `policies/data-minimisation.md v4.1` is. Frameworks think in terms of controls; audits think in terms of evidence. The crosswalk is the mapping between the two.

The property that makes the crosswalk work: **the same artefact can appear in many rows**. The model-card privacy slice (chapter 03) legitimately discharges NIST PF Communicate-P subcategories, 27701 PIMS controls on notification and transparency, 42001 Annex A controls on AI-system information, EU AI Act Article 13 transparency obligations, and CSA AICM transparency domain controls. One artefact, many audits — if it was authored once to the checkability / falsifiability / honesty-about-limitation disciplines (chapter 03), one well-maintained document services the lot.

## NIST Privacy Framework — the US-centric reference

The NIST Privacy Framework (PF) is organised like the Cybersecurity Framework: five Functions, each broken into Categories, each into Subcategories.

Functions:

- **Identify-P** — Inventory and mapping (ID.IM-P), Business environment (ID.BE-P), Risk assessment (ID.RA-P), Data processing ecosystem risk management (ID.DE-P).
- **Govern-P** — Governance policies (GV.PO-P), Risk management strategy (GV.RM-P), Awareness and training (GV.AT-P), Monitoring and review (GV.MT-P).
- **Control-P** — Data processing policies, processes, and procedures (CT.PO-P), Data processing management (CT.DM-P), Disassociated processing (CT.DP-P).
- **Communicate-P** — Communication policies, processes, and procedures (CM.PO-P), Data processing awareness (CM.AW-P).
- **Protect-P** — Data protection policies, processes, and procedures (PR.PO-P), Identity management, authentication, and access control (PR.AC-P), Data security (PR.DS-P), Maintenance (PR.MA-P), Protective technology (PR.PT-P).

The AI privacy engineer's artefacts map onto these functions as follows (illustrative — each row is a candidate row in the crosswalk):

| NIST PF Subcategory | Short name | Artefact(s) (module source) |
|---|---|---|
| ID.IM-P1 | Systems / products / services that process data are inventoried | RoPA, system inventory (mod-103 ch. 01) |
| ID.IM-P5 | Purposes of data processing are inventoried | Processing-purpose register with lawful basis (mod-103 ch. 02) |
| ID.IM-P7 | Data processing environment is identified | Data flow diagram (mod-102 ch. 01; mod-111 ch. 01) |
| ID.RA-P1 | Contextual factors that may affect privacy risk are identified | DPIA §a (mod-111 ch. 01) |
| ID.RA-P3 | Potential problematic data actions and associated problems are identified | LINDDUN model output (mod-102); DPIA §c |
| ID.RA-P4 | Problematic data actions, likelihoods, and impacts are assessed | DPIA risk register (mod-111 ch. 01, ch. 03 NIST PRAM) |
| GV.PO-P1 | Organisational privacy values and policies are established | Privacy programme charter (chapter 01) |
| GV.PO-P5 | Processes to instil organisational privacy values are in place | Programme cadence and retrospective (chapter 01) |
| GV.AT-P3 | Senior management is aware of their roles and responsibilities | Sign-off matrix and executive-sponsor briefings (chapter 01 and 04) |
| GV.MT-P7 | Lessons learned from problematic data action identification, risk assessment, and incident response are captured | After-action records (chapter 05) |
| CT.DM-P1 | Data elements can be accessed for review | DSR access runbook (mod-110 ch. 01) |
| CT.DM-P2 | Data elements can be accessed for transmission or disclosure | DSR portability runbook (mod-110 ch. 01) |
| CT.DM-P3 | Data elements can be accessed for alteration | DSR rectification runbook |
| CT.DM-P4 | Data elements can be accessed for deletion | DSR erasure runbook (mod-110 ch. 01, ch. 06) |
| CT.DM-P6 | Data are destroyed according to policy | Retention schedule plus audit job (chapter 01; mod-109 ch. 03) |
| CT.DP-P1 | Data are processed to limit observability and linkability | PET deployments — DP (mod-106), FL (mod-107), TEE (mod-108) |
| CT.DP-P2 | Data are processed to limit identification of individuals | Redaction pipeline (mod-104), K-anonymity / l-diversity treatment |
| CM.AW-P2 | Data corrections or deletions can be communicated to individuals and organisations | DSR audit trail (mod-110 ch. 07) |
| CM.AW-P6 | Data provenance and lineage are maintained | ML-BOM (mod-103 ch. 06) |
| CM.AW-P8 | Individuals are informed about data processing practices and the associated privacy risks | Model-card / system-card privacy slice (chapter 03) |
| PR.AC-P5 | Network integrity is protected | Infra security controls (security team artefact set) |
| PR.DS-P1 | Data-at-rest are protected | Encryption inventory, KMS policy (platform team artefact) |
| PR.DS-P2 | Data-in-transit are protected | TLS inventory, service-mesh config |
| PR.PT-P1 | Audit / log records are determined, documented, implemented, and reviewed in accordance with policy | Logging policy and retention schedule (mod-109 ch. 03, chapter 01) |

The NIST PF has roughly 100 subcategories; a full crosswalk fills several pages. Treat the table above as the AI-privacy-engineer-facing core; a full enterprise crosswalk extends further into the security-owned subcategories.

## ISO/IEC 27701 PIMS — the "GDPR certification shape"

ISO/IEC 27701 is the privacy information management system extension to ISO/IEC 27001. It modifies 27001 and 27002 for privacy and adds two annexes:

- **Annex A** — PIMS-specific controls for **controllers**.
- **Annex B** — PIMS-specific controls for **processors**.

Many organisations operate as both controller and processor across different offerings and must map against both. The 2019 edition structures the controller controls in six topic areas:

- **A.7.2** — Conditions for collection and processing.
- **A.7.3** — Obligations to PII principals.
- **A.7.4** — Privacy by design and privacy by default.
- **A.7.5** — PII sharing, transfer, and disclosure.

The processor controls (Annex B) parallel these on the processor side. ISO/IEC 27701:2025 is the revised edition; the content is similar but the numbering has shifted — check the edition in force for your certification before publishing a crosswalk.

Illustrative mapping to AI-privacy-engineer artefacts:

| 27701:2019 Annex A control (controller) | Short name | Artefact(s) |
|---|---|---|
| A.7.2.1 | Identify and document purpose | Processing-purpose register (mod-103 ch. 02) |
| A.7.2.2 | Identify lawful basis | Lawful-basis map plus LIA where Art. 6(1)(f) is cited (mod-101 ch. 02; mod-103 ch. 02) |
| A.7.2.3 | Determine when and how consent is to be provided | Consent architecture and consent records (mod-103 ch. 02; mod-109 ch. 05) |
| A.7.2.5 | Privacy impact assessment | DPIA (mod-111) |
| A.7.2.6 | Contracts with PII processors | DPA inventory (legal-owned; referenced) |
| A.7.3.1 | Determining and fulfilling obligations to PII principals | DSR runbook set (mod-110 ch. 01) |
| A.7.3.2 | Determining information for PII principals | Privacy notice content; model-card privacy slice (chapter 03) |
| A.7.3.3 | Providing information to PII principals | Privacy portal; model-card publication (chapter 03) |
| A.7.3.5 | Providing mechanism to modify or withdraw consent | Consent-withdrawal flow (mod-109 ch. 05; mod-110 ch. 01) |
| A.7.3.6 | Providing mechanism to object to PII processing | Right-to-object runbook (mod-110 ch. 01) |
| A.7.3.7 | PII principals' rights to access, correction and/or erasure | DSR access / rectification / erasure runbooks (mod-110 ch. 01) |
| A.7.3.9 | Correction or erasure | DSR runbooks with audit trail (mod-110 ch. 07) |
| A.7.4.1 | Limit collection | Minimisation checkpoints (mod-103 ch. 03) |
| A.7.4.2 | Limit processing | Processing-purpose binding (mod-103 ch. 02) |
| A.7.4.3 | Accuracy and quality | Evaluation reports |
| A.7.4.4 | PII minimisation objectives | Redaction / vault / tokenisation design (mod-103 ch. 04; mod-104) |
| A.7.4.5 | PII de-identification and deletion at end of processing | Retention schedule, erasure-at-EoL runbook (chapter 01) |
| A.7.4.6 | Temporary files | Temp-file handling policy |
| A.7.4.7 | Retention | Retention schedule (chapter 01; mod-109 ch. 03) |
| A.7.4.8 | Disposal | Secure-deletion procedure |
| A.7.4.9 | PII transmission controls | Encryption in transit; cross-border transfer controls |
| A.7.5.x | Sharing / transfer / disclosure | Sub-processor list; transfer impact assessments |

The 27701 crosswalk's audit audience is the certification auditor (e.g., BSI, SGS, TÜV). The evidence they will ask to see is the artefact plus the operational evidence of it being lived (log of DSR requests honoured, audit of retention lifecycle, consent records sampled). Keep the operational evidence sample-ready; "we have a policy" is a weaker finding than "we have a policy and here are twelve sampled records that conform."

## ISO/IEC 42001 AIMS — the AI management system shape

ISO/IEC 42001 is the first ISO AI management system standard. Its structure mirrors other ISO management system standards (ISO 9001, 27001, 27701):

- **Clause 4** — Context of the organisation.
- **Clause 5** — Leadership.
- **Clause 6** — Planning.
- **Clause 7** — Support (resources, competence, awareness, communication, documented information).
- **Clause 8** — Operation (operational planning and control; AI risk assessment; AI impact assessment).
- **Clause 9** — Performance evaluation (monitoring, internal audit, management review).
- **Clause 10** — Improvement (nonconformity, continual improvement).
- **Annex A** — Reference control objectives and controls.
- **Annex B** — Implementation guidance.
- **Annex C** — AI-related organisational objectives.
- **Annex D** — AI management system across domains and sectors.

The clauses the AI privacy engineer directly contributes to:

- **Clause 6.1.2** AI risk assessment, **Clause 6.1.3** AI risk treatment — DPIA and risk register (mod-111 ch. 01, ch. 03) feed in.
- **Clause 6.1.4** AI system impact assessment — DPIA with the AI-specific feared-event register (mod-111 ch. 01) is the primary input.
- **Clause 8.2** AI risk assessment during operation — the review-board cadence (chapter 04) and the risk register maintenance are the operational implementation.
- **Clause 8.3** AI risk treatment during operation — PET deployments and mitigation measures (mods 104–108, 110).
- **Clause 8.4** AI system impact assessment during operation — DPIA refresh triggers (chapter 01 and mod-111 ch. 01).
- **Clause 9.1** monitoring, measurement, analysis, and evaluation — privacy-eval regression suite, memorisation audits, DSR SLA monitoring.
- **Clause 9.2** internal audit — audit-readiness review (chapter 01) and the quarterly crosswalk refresh.
- **Clause 9.3** management review — the executive-sponsor briefing cadence (chapter 01 and 04).

Annex A of 42001:2023 is organised by control objectives. Illustrative mappings for AI-privacy-engineer artefacts:

| 42001 Annex A control | Short name | Artefact(s) |
|---|---|---|
| A.2.2 | AI policy | AI governance policy (enterprise-owned); privacy programme charter references |
| A.4.2 | AI system resources | ML-BOM, system inventory (mod-103 ch. 06; chapter 01) |
| A.5.2 | AI system impact assessment | DPIA for AI systems (mod-111 ch. 01) |
| A.5.3 | AI system impact assessment documentation | DPIA refresh records (mod-111 ch. 01) |
| A.6.2.5 | AI system verification and validation | Memorisation / MIA audits (mod-105); privacy-eval regression suite |
| A.7.2 | Data for AI systems | Data governance policy; datasheets (mod-103 ch. 01) |
| A.7.3 | Acquisition of data | Data-ingestion lawful-basis map (mod-103 ch. 02) |
| A.7.4 | Quality of data for AI systems | Data quality evaluation reports |
| A.7.5 | Data provenance | ML-BOM (mod-103 ch. 06) |
| A.7.6 | Data preparation | PII pipeline reports (mod-104) |
| A.8.2 | Information for interested parties | Model-card / system-card privacy slice (chapter 03) |
| A.8.3 | External reporting | Incident notification records (chapter 05) |
| A.9.3 | Objectives for responsible use of AI | Privacy programme charter (chapter 01); DPIA objectives |
| A.10.2 | Processes for responsible use of AI | Review-board cadence (chapter 04) |

Full 42001 adoption is an enterprise-scale project owned by the governance function; the AI privacy engineer contributes the artefacts that discharge the privacy-adjacent controls above, and the crosswalk makes that contribution visible.

## CSA AI Controls Matrix — the cloud-industry shape

The Cloud Security Alliance AI Controls Matrix (AICM) is a cross-industry control catalogue specifically for AI systems in cloud environments. Its structure organises controls into domains such as data governance, model management, deployment and operations, governance and compliance, and trust / transparency; the specific domain names and counts are refreshed by CSA — check the current version of the AICM before stamping a crosswalk.

AICM's audience is cloud customers and cloud providers doing AI governance reviews of each other. Compared to 42001, AICM is faster-moving and more operationally prescriptive. It is the right left-column framework when your customer is asking for a cloud-industry-shape AI governance report alongside a SOC 2 or ISO 27001.

Artefacts that map to AICM typically include the PET deployment claims (DP, FL, TEE), the model and system cards, the DSR runbooks, the data governance policy, and the incident-response artefacts. The AICM evidence requirements tend to be concrete and engineering-friendly, which makes the crosswalk relatively mechanical once the artefact set is in place.

A specific advantage of maintaining an AICM crosswalk: AICM controls are structured to pair with the Cloud Controls Matrix (CCM) and with 27001, which lets the crosswalk be extended without building from scratch. Check the CSA publications for the current AICM version and its CCM / 27001 companion mappings.

## Composition — the one-artefact-many-rows pattern

The crosswalk pays off because the same well-authored artefact services many rows. Three worked examples.

### The DPIA

The DPIA (mod-111 ch. 01) has sections (a) through (f). It alone satisfies, at minimum:

- NIST PF: ID.RA-P1 (contextual factors), ID.RA-P3 (problematic data actions), ID.RA-P4 (risk assessment), GV.MT-P5 (periodic review), CM.AW-P8 (informing individuals via the public summary).
- ISO/IEC 27701: A.7.2.5 (privacy impact assessment) directly; indirectly A.7.2.1–2 (purpose and basis), A.7.2.4 (contracts).
- ISO/IEC 42001: A.5.2, A.5.3 (AI system impact assessment and its documentation); Clauses 6.1.2 / 6.1.3 / 6.1.4 / 8.4.
- EU AI Act: Articles 9 / 10 / 15 / 26 touch points for high-risk systems.

One DPIA, five framework audits.

### The model-card privacy slice

Chapter 03's privacy slice satisfies:

- NIST PF: CM.AW-P5, CM.AW-P8 (transparency, notification), ID.IM-P5 (purposes), ID.IM-P7 (processing environment), CT.DP-P1 (observability-limiting disclosures).
- ISO/IEC 27701: A.7.3.2, A.7.3.3 (information to principals).
- ISO/IEC 42001: A.8.2 (information for interested parties), A.6.2.5 (system verification — via the memorisation / MIA citations).
- EU AI Act: Article 13 transparency, Article 53 / Annex XI GPAI information.
- CSA AICM: transparency and model-management domains.

One card, five framework audits.

### The DSR runbook set

Mod-110 ch. 01's runbook set satisfies:

- NIST PF: CT.DM-P1 / P2 / P3 / P4 (access / transmission / alteration / deletion).
- ISO/IEC 27701: A.7.3.1, A.7.3.6, A.7.3.7, A.7.3.9.
- ISO/IEC 42001: via the responsible-use clauses (A.9.3, A.10.2) referencing the DSR mechanisms.
- GDPR: Articles 15, 16, 17, 18, 19, 20, 21 — the DSR charter.
- CCPA / CPRA: §§ 1798.100, .105, .106, .110, .115, .120, .121 — the Californian DSR equivalents.

One runbook set, two framework audits plus two statutory audits.

## Residual-gap discipline — where the crosswalk is honest

A crosswalk row where the evidence artefact only partially discharges the control is still a row — the discipline is to say so. The residual-gap column answers:

- What specifically is missing.
- Why (not enough time / not enough coverage / a partial implementation / a known architectural limitation).
- When the gap is scheduled to close (quarter / project / release).
- Who owns closing it (role).

An auditor reading a crosswalk with honest residual-gap entries trusts the programme more than they would trust one with no gaps claimed. The failure mode is the inverse: a crosswalk that claims full coverage everywhere and then collapses when the auditor samples the evidence. Honest gaps, tracked, and visibly closing are a strength, not a weakness.

## Keeping the crosswalk current — the operating rhythm

The crosswalk is a living document:

- **Quarterly refresh** — each row's last-verified date is checked; rows older than six months require a re-verification or a note.
- **Framework-version tracking** — a header of the crosswalk file lists the version of each framework the rows are keyed to (e.g., NIST PF v1.0, ISO 27701:2019, ISO 42001:2023, AICM v2.0). When a framework publishes a new version, the crosswalk's maintainer schedules a version-migration project; the old rows are kept in a legacy sheet during migration.
- **Audit-driven updates** — audit findings that touch a row (observation, nonconformity, minor / major) are recorded in the audit-observations column; the subsequent fix is tracked and the row updated.
- **Charter integration** — the audit-readiness review (chapter 01) uses the crosswalk as its primary artefact; the quarterly retrospective reviews the crosswalk's health.

## Who owns the crosswalk

At team scope, the AI privacy engineer owns the crosswalk entries for the artefacts the role authors. Entries for enterprise-controlled artefacts (encryption at rest, access management infra, logging platform) point to the owning function and the crosswalk lives alongside — in some organisations as a federated set of crosswalks that roll up, in others as a single enterprise crosswalk the governance function maintains.

The boundary to the senior architect (level 50) role is: cross-product, cross-business-unit, and cross-jurisdiction reconciliation is level 50's territory. At level 30, you maintain the crosswalk for the artefacts you own and contribute into the enterprise rollup. Chapter 08 names this more fully.

## Common failure modes of the crosswalk

- **One-off audit crosswalks.** Each audit builds its own crosswalk; no single source of truth. Fix: one durable crosswalk, with audit-observation columns.
- **Over-mapping.** The same artefact claimed to discharge controls it only tangentially touches. Fix: the residual-gap discipline; honest coverage beats expansive coverage.
- **Missing version pinning.** The crosswalk claims to meet framework X without saying which version of X. Fix: the header pins the framework versions; the migration path is scheduled.
- **Static evidence.** The artefact paths point at documents that have not been updated in a year. Fix: the last-verified column with a six-month ceiling.
- **Artefact drift.** The crosswalk points at `policies/retention.md` but the file has been renamed to `policies/retention-v2.md`; the link rots. Fix: the quarterly refresh checks links.
- **Hidden ownership.** An artefact has no stated owner; nobody updates it between audits. Fix: every row has an artefact-owner cell.

## Summary — what you can do after this chapter

- Build a crosswalk as an engineering object — versioned, schema'd, owned, kept current — not an audit-time deliverable.
- Map the artefacts from the preceding modules onto NIST Privacy Framework subcategories, ISO/IEC 27701 PIMS controls, ISO/IEC 42001 AIMS clauses, and CSA AICM domains.
- Exercise the one-artefact-many-rows pattern — DPIA, model-card privacy slice, DSR runbook — to compound effort across audits.
- Apply the residual-gap discipline so the crosswalk is honest about partial coverage and visible about remediation plans.
- Run the quarterly-refresh / framework-version-tracking / audit-driven-update operating rhythm.
- Draw the boundary to the enterprise control library (level 50) and to the level-30 artefact set you own.
- Recognise and fix the characteristic crosswalk failure modes (one-off, over-mapping, missing version pin, static evidence, artefact drift, hidden ownership).

Chapter 07 picks up with the credentials portfolio — the professional-qualifications shape the AI privacy engineer plans toward, and how the credentials compose with the crosswalk and the programme the preceding chapters built.
