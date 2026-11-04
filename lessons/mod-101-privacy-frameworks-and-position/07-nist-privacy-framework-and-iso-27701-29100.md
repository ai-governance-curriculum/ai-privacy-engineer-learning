# NIST Privacy Framework and ISO/IEC 27701 / 29100 — the Program Shapes an AI Privacy Program Plugs Into

Regulations tell you *what* obligations exist; frameworks tell you *how* to run the organisation that discharges them. For enterprise AI privacy, three documents dominate: the **NIST Privacy Framework**, **ISO/IEC 27701** (privacy information management system — PIMS), and **ISO/IEC 29100** (privacy framework). This chapter is the engineer's reading — what each is, how they compose, and where an AI Privacy Engineer plugs into each.

Primary sources:

- **NIST Privacy Framework 1.0 (and subsequent updates)** — [nist.gov/privacy-framework](https://www.nist.gov/privacy-framework). The Core, Profiles, and Implementation Tiers.
- **NIST Privacy Risk Assessment Methodology (PRAM)** — [nist.gov/itl/applied-cybersecurity/privacy-engineering/collaboration-space/focus-areas/risk-assessment](https://www.nist.gov/itl/applied-cybersecurity/privacy-engineering/collaboration-space/focus-areas/risk-assessment). Companion worksheet set.
- **ISO/IEC 27701:2019** — *Security techniques — Extension to ISO/IEC 27001 and ISO/IEC 27002 for privacy information management — Requirements and guidelines*. [iso.org/standard/71670](https://www.iso.org/standard/71670.html).
- **ISO/IEC 29100:2011** — *Information technology — Security techniques — Privacy framework*. [iso.org/standard/45123](https://www.iso.org/standard/45123.html).
- **ISO/IEC 29134:2017** — *Information technology — Security techniques — Guidelines for privacy impact assessment*. [iso.org/standard/62289](https://www.iso.org/standard/62289.html). (Covered in depth in `mod-111-dpia-and-privacy-reviews`; mentioned here because 27701 cross-references it.)
- **ISO/IEC 27018:2019** — *Code of practice for protection of personally identifiable information (PII) in public clouds acting as PII processors*. [iso.org/standard/76559](https://www.iso.org/standard/76559.html).
- **ISO/IEC 42001:2023** — *Artificial intelligence management system (AIMS)*. [iso.org/standard/81230](https://www.iso.org/standard/81230.html). (Plugs into this stack via the AI-system side; covered in `mod-111` and `mod-112`.)

> ISO/IEC documents are copyright and must be purchased. Secondary summaries are useful for orientation but your audit depends on the official text; the organisation should purchase the standards you intend to be assessed against.

## NIST Privacy Framework — Core, Profiles, Tiers

The NIST Privacy Framework (PF) is a voluntary, risk-based framework for managing privacy risk. Its structure mirrors the NIST Cybersecurity Framework.

### The five Core Functions

The PF Core has five Functions, each decomposed into Categories and Subcategories:

- **IDENTIFY-P** — develop the organisational understanding to manage privacy risk for individuals arising from data processing.
- **GOVERN-P** — develop and implement the organisational governance structure to enable an ongoing understanding of the organisation's risk-management priorities informed by privacy risk.
- **CONTROL-P** — develop and implement appropriate activities to enable organisations or individuals to manage data with sufficient granularity to manage privacy risks.
- **COMMUNICATE-P** — develop and implement appropriate activities to enable organisations and individuals to have a reliable understanding and engage in a dialogue about how data are processed and associated privacy risks.
- **PROTECT-P** — develop and implement appropriate data processing safeguards.

Each Function has Categories (e.g., *Data Processing Ecosystem Risk Management* under GOVERN-P), and Categories have Subcategories (specific outcome statements like *GV.PO-P2: Processes to instill organizational privacy values within system / product / service development and operations are established and in place*).

The Framework is **outcome-based** — it does not prescribe *how* to achieve a Subcategory, it says *what* outcome must be present. That is the right shape for AI privacy, where the technical means (DP, confidential compute, unlearning) are moving but the outcomes (minimisation, transparency, security) are stable.

### Profiles — Current, Target, and the delta

A **Profile** is a selection of Functions / Categories / Subcategories tailored to a particular system, business line, or sector. Standard practice:

- **Current Profile** — what outcomes are in place today.
- **Target Profile** — what outcomes you need in place to meet the privacy-risk tolerance and regulatory context.
- **Delta → roadmap** — the gap is the privacy-engineering backlog.

For AI systems, the Target Profile adds AI-specific outcomes: memorisation-risk management (under PROTECT-P / CONTROL-P), explainability for ADM (under COMMUNICATE-P), DP / FL / TEE deployment decisions (under PROTECT-P), data-minimisation for training data (under CONTROL-P).

### Implementation Tiers

NIST PF defines four Implementation Tiers (Partial, Risk-Informed, Repeatable, Adaptive) describing the degree to which privacy-risk management is formalised and improved over time. Tiers are a descriptive assessment, not a target rank.

### Where an AI Privacy Engineer plugs in

| NIST PF Function | AI Privacy Engineer contribution |
|---|---|
| IDENTIFY-P | AI-system inventory data (contributed from product / platform); data-flow diagrams for training / retrieval / inference / logging; PII taxonomy for AI contexts; risk-assessment inputs (LINDDUN + PRAM) |
| GOVERN-P | Privacy-engineering standards; AI-privacy policies (e.g., logging policy, retention policy, DP-release standard); integration with ISO/IEC 42001 AIMS governance |
| CONTROL-P | The engineering surface — data-minimisation controls, DSR pipelines, consent-propagation, retention jobs, unlearning paths |
| COMMUNICATE-P | Meaningful-information artifacts for ADM; privacy slice of model / system / dataset cards; incident / breach communication templates |
| PROTECT-P | PII pipeline, redaction, DP training, FL, TEE deployment, MIA / extraction harness, memorisation defence |

Full depth on how these outputs land in a program lives in `mod-112-privacy-program-and-communication`.

## NIST PRAM — the companion worksheet

The **NIST Privacy Risk Assessment Methodology (PRAM)** is NIST's structured, worksheet-driven approach for identifying, assessing, and prioritising privacy risk from a specific system. It complements the PF (which operates at the organisation / program level) by operating at the system / data-flow level.

### The four worksheets

Standard PRAM consists of worksheets (names and exact content refresh with NIST revisions — pull the current set from the PF site):

- **Worksheet 1 — Frame the Business Context** — business needs, regulatory environment, data-processing ecosystem.
- **Worksheet 2 — Identify Data Actions** — enumerate the data actions (collection, processing, storage, transmission, disposal), the data types, the actors, and the context of each action.
- **Worksheet 3 — Identify Problematic Data Actions** (PDAs) — map data actions against problematic contexts (loss of trust, discrimination, loss of autonomy, economic loss, etc.) and score likelihood × impact.
- **Worksheet 4 — Prioritise and Respond** — route PDAs to mitigations.

For AI systems, PRAM is the **engineer's** worksheet that converts a LINDDUN-shape threat model (`mod-102-privacy-threat-modelling`) into a prioritised control backlog.

## ISO/IEC 29100 — the eleven privacy principles

ISO/IEC 29100 is the **privacy framework** standard. It defines actors, interactions, and — most importantly for an engineer — eleven privacy principles that other ISO/IEC privacy documents reference:

1. Consent and choice
2. Purpose legitimacy and specification
3. Collection limitation
4. Data minimization
5. Use, retention and disclosure limitation
6. Accuracy and quality
7. Openness, transparency and notice
8. Individual participation and access
9. Accountability
10. Information security
11. Privacy compliance

These overlap almost perfectly with the OECD Privacy Guidelines eight principles (chapter 06) and the GDPR Art. 5 principles (chapter 02). Memorise the OECD eight → GDPR seven → ISO 29100 eleven crosswalk and you can speak the same obligations in three dialects.

29100 is an **orientation** standard — it does not prescribe implementation. 27701, 29134, and 27018 do that.

## ISO/IEC 27701 — the PIMS extension

ISO/IEC 27701 extends **ISO/IEC 27001** (information security management system, ISMS) and **ISO/IEC 27002** (control catalogue) with privacy content, producing a **Privacy Information Management System (PIMS)**.

### What 27701 adds

- **PIMS management-system requirements** — amendments / additions to ISO/IEC 27001 clauses to incorporate privacy objectives, roles (PII controller / PII processor), risk assessment including privacy risk.
- **Guidance for PII controllers** (27701 Clause 7) — extensions to 27002 controls to address controller-specific obligations such as determining and documenting lawful basis, obtaining consent, data-subject-rights-handling.
- **Guidance for PII processors** (27701 Clause 8) — extensions to 27002 controls to address processor-specific obligations such as contract requirements and sub-processor management.
- **Mapping to GDPR, ISO 29100, ISO 27018** — annexes cross-walk 27701 controls to specific regulatory and framework articles.

### Certification vs. conformance

An organisation that is certified to ISO/IEC 27001 can pursue 27701 certification as an extension (requires 27001 as a prerequisite). Certification is attractive to customers and regulators because a third-party auditor validates the PIMS. The **ISO/IEC 27701 Lead Implementer (LI) / Lead Auditor (LA)** certifications are the practitioner credentials in this space (see `mod-112-privacy-program-and-communication` for credential positioning).

### Where an AI Privacy Engineer plugs in

27701 is the **organisational shape**. The AI Privacy Engineer contributes to specific controls:

- **A.7.2 / A.7.4** (lawful basis / specific consent / information to PII principals) — your consent-register / lawful-basis-registry evidence.
- **A.7.3 / A.8.3** (obligations to PII principals / rights-handling) — your DSR pipeline.
- **A.7.5** (privacy by design and privacy by default) — your design-phase reviews, default-off logging, redaction pipelines.
- **A.8.2** (conditions for collection and processing — processor) — your contract-compliance evidence for the model-provider / foundation-model path.
- **Annex B** cross-walk to 27002 — your 27002-aligned technical controls (access, logging, encryption, pen-test).

## ISO/IEC 27018 — PII in public clouds

ISO/IEC 27018 is the code of practice for public-cloud **PII processors**. A cloud provider claims 27018 conformance to signal it processes customer PII consistent with the standard's controls. For an AI Privacy Engineer this matters as a **vendor-selection** signal and a **contract-evidence** source: if your foundation-model provider or cloud vendor is 27018-conformant, you can rely on specific controls (data-subject-request support, no-use-for-advertising, notification of law-enforcement requests, etc.) under the audited claim.

## ISO/IEC 42001 — AIMS

ISO/IEC 42001:2023 is the **Artificial Intelligence Management System** standard — the AI analogue of 27001 / 27701. It intersects privacy where AI systems process personal data. 42001 is the shape an enterprise AI program plugs the privacy program *into*. Covered in depth in `mod-111` and `mod-112`.

## Composing the four standards

A mature enterprise privacy-engineering program uses the four documents in layers:

1. **ISO/IEC 29100** gives the vocabulary (actors, principles, lifecycle).
2. **NIST PF Core** gives the program-level outcome catalogue (Functions / Categories / Subcategories).
3. **ISO/IEC 27701** gives the auditable management-system shape (controls + evidence against a certified ISMS).
4. **NIST PRAM** gives the system-level risk-assessment worksheet that feeds into both PF and 27701.
5. **ISO/IEC 42001** (where in scope) adds the AI-management-system layer.

Operationally: you run **one** control catalogue. You tag each control with the frameworks and regulations it discharges. One PII-redaction pipeline control, for example, is simultaneously:

- GDPR Art. 25 + Art. 32 evidence,
- EU AI Act Art. 15 cybersecurity evidence,
- NIST PF `PR.DS-P1` (processing is protected consistent with risk) evidence,
- ISO/IEC 27701 A.7.4 / A.8.4 evidence,
- HIPAA Security Rule technical-safeguard evidence,
- GLBA Safeguards Rule evidence.

One pipeline, many audits. That is the compound effect of running the frameworks properly.

## What the AI-specific additions look like

Most of the PF / 27701 catalogue is AI-agnostic. The AI-specific additions this track teaches land in the following boxes:

| AI-specific addition | NIST PF home | ISO/IEC 27701 home | Module |
|---|---|---|---|
| LINDDUN / PRAM applied to AI data flows | IDENTIFY-P / GOVERN-P | A.7.5 privacy by design | `mod-102` |
| Training-corpus Datasheet + ML-BOM | CONTROL-P / IDENTIFY-P | A.8.2 collection conditions | `mod-103` |
| Production PII / PHI redaction | PROTECT-P / CONTROL-P | A.8.4 engineering security | `mod-104` |
| MIA / extraction harness as controls-assurance | PROTECT-P / IDENTIFY-P | A.7.4 / A.8.4 | `mod-105` |
| Differential-privacy release package | PROTECT-P | A.7.4 / A.8.4 + 27001 A.14 | `mod-106` |
| Federated-learning + secure-aggregation | PROTECT-P | A.7.4 / A.8.4 | `mod-107` |
| Confidential-computing / HE / MPC inference | PROTECT-P | A.7.4 / A.8.4 + 27018 | `mod-108` |
| RAG tenant-ACL + agent-scope controls | CONTROL-P / PROTECT-P | A.7.4 / A.8.4 | `mod-109` |
| Machine unlearning + DSR engineering | CONTROL-P / COMMUNICATE-P | A.7.3 / A.8.3 | `mod-110` |
| DPIA audit-grade bundle | GOVERN-P / IDENTIFY-P | A.7.5 + 29134 | `mod-111` |
| Privacy-program ops + incident response | GOVERN-P / COMMUNICATE-P | 27701 management-system clauses | `mod-112` |

## Credentials in the stack

The practitioner credentials that align with this material (positioning in `mod-112`):

- **IAPP CIPT** — the Certified Information Privacy Technologist is the most-cited credential for AI-privacy-engineer-shaped postings. CIPT body of knowledge covers privacy engineering, PETs, privacy-by-design, DPIAs.
- **IAPP CIPM** — privacy-program management. Pairs with 27701 LI for enterprise-program work.
- **IAPP CIPP/E** — European privacy law; the analyst / counsel-support credential. Useful cross-training.
- **IAPP CIPP/US** — US privacy law. Useful cross-training.
- **IAPP AIGP** — Artificial Intelligence Governance Professional. Growing in enterprise relevance as the ISO/IEC 42001 and EU AI Act stack matures.
- **ISO/IEC 27701 LI / LA** — lead-implementer / lead-auditor of PIMS. The certification-track credential; aligned with audit-day work.

## Depth boundary

- **Writing the enterprise control library at policy-taxonomy scale** — mapping every control to every framework and reconciling cross-jurisdiction conflicts — defers up to `senior-ai-governance-architect` (level 50).
- **Operating the PIMS day-to-day** — control testing, evidence collection cycles, management reviews, internal audit cycles — is a shared surface: `ai-governance-analyst` (level 15) runs the cadence, this role contributes privacy-engineering evidence, and `senior-ai-governance-architect` owns the architecture.
- **The ISO/IEC 42001 AIMS** — the AI-system-management-system shape beyond its privacy slice — is covered further in `mod-111` and the broader AI-governance tracks.

## Summary

- NIST PF gives you the outcome catalogue; PRAM gives you the system-level worksheet; ISO/IEC 27701 gives you the auditable management system; ISO/IEC 29100 gives you the shared vocabulary; ISO/IEC 42001 adds the AI-management layer.
- Run **one** control catalogue, tag each control with the frameworks and regulations it discharges, and one engineering effort gets you many audits.
- As an AI Privacy Engineer you are a *contributor* to the enterprise program — contributing technical controls, evidence, and AI-specific additions — rather than the owner of the full program (which is a shared responsibility across governance, security, legal, DPO, and architect roles).

Next: the role's exact scope on the governance ladder.
