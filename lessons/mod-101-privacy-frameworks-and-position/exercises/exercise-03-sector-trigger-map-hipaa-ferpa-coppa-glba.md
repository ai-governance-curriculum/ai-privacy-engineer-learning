# Exercise 03: Sector Trigger Map — HIPAA, FERPA, COPPA, GLBA

**Estimated effort:** 2 hours · **Output:** one sector-trigger decision diagram + four control-delta tables + one worked scenario memo · **Primary chapter:** 05

## Objective

Build a reusable **sector-trigger decision diagram** that an engineer can run in under 10 minutes against any new AI use case, plus the **control-delta** that fires when each sector law lands. Then walk the diagram against three worked scenarios and write up the hardest one.

## Prerequisites

- Chapter 05 (sector US laws) read.
- Primary references open: [HHS HIPAA for Professionals](https://www.hhs.gov/hipaa/for-professionals/index.html), [HHS OCR De-identification Guidance](https://www.hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification/), [US Dept. of Ed Student Privacy Policy Office / FERPA](https://studentprivacy.ed.gov/), [FTC Children's Privacy](https://www.ftc.gov/business-guidance/privacy-security/childrens-privacy), [FTC Safeguards Rule](https://www.ftc.gov/legal-library/browse/rules/safeguards-rule).
- Working familiarity with your organisation's existing DSR / rights pipeline and PII pipeline (or the Exercise 01 scenario's).

## Problem statement

Produce three deliverables.

### Deliverable A — the sector-trigger decision diagram

A one-page flowchart (Mermaid, Excalidraw, or any diagramming tool — paste the source and a rendered image). The diagram starts with a new AI use case and ends with the *union* of applicable sector laws. It must branch on:

- Does the system process **health information** on behalf of or by a covered entity? If yes → HIPAA. If "health-adjacent" (wellness app, consumer-grade fitness), ask whether a covered-entity relationship exists.
- Does the system process **education records** maintained by a covered institution? If yes → FERPA. Branch for state student-privacy laws.
- Is the service **directed to children under 13**, or do we have **actual knowledge** children under 13 use it? If yes → COPPA. Branch for VPC-exempt internal operations.
- Is the organisation a **financial institution** under FTC jurisdiction (or regulated-banking-agency equivalent) processing **customer non-public personal information**? If yes → GLBA Safeguards Rule. Branch for banking-agency-regulated entities.
- Does the system process **EU personal data**? If yes → GDPR.
- Does the system process **California / state-resident** personal information above threshold? If yes → CCPA / state matrix.
- Does the system use **biometric data for identification**? If yes → GDPR Art. 9 + US state biometric laws (BIPA, CUBI, etc.).

Every branch terminates in either "in scope, go to Deliverable B row for this law" or "not in scope, documented reasoning".

### Deliverable B — the four control-delta tables

For each of HIPAA, FERPA, COPPA, GLBA Safeguards, write a short table of the **control deltas** that fire on top of a baseline GDPR + CCPA engineering pipeline. Columns:

| Obligation | Control delta against baseline | Test | Evidence artifact | Depth boundary (who else must be involved) |

HIPAA rows (at minimum): BAA in place, minimum-necessary review, access + audit on all PHI paths including LLM prompt + retrieval, Safe Harbor pipeline vs. Expert Determination memo, breach-notification runbook with HIPAA clocks, encryption-renders-PHI-secured compliance stance.

FERPA rows (at minimum): written agreement / school-official exception, directory-information handling, record-access portal, no-redisclosure contract clauses, overlap handling with COPPA for under-13.

COPPA rows (at minimum): age-gate UX, verifiable parental consent workflow, child-specific notice, minimisation, data retention and right to review / delete, no targeted advertising without VPC, training-data-provenance check for scraped content-from-children scenarios.

GLBA Safeguards rows (at minimum): MFA, encryption at rest + in transit, logging + monitoring, annual pen-test incl. AI-attack-surface coverage, biannual vulnerability assessment, service-provider oversight incl. foundation-model vendor, incident-response plan with AI-specific scenarios, qualified-individual designation, board reporting template.

### Deliverable C — the worked-scenario memo

Walk your trigger diagram against **all three** scenarios below. Pick the **hardest** of the three (your choice; defend the choice in one sentence) and write a 1–2 page memo covering: the triggers that fired, the control-delta you'd implement, the test / evidence shape, and the deferrals (to counsel, DPO, SecOps, L15 analyst, L35 eval engineer, L50 architect).

- **Scenario 1 — Healthcare chatbot.** A US hospital network wants an LLM-powered patient-portal assistant that reads the patient's chart and answers questions in natural language. Partnered with a foundation-model vendor. Logs are retained for quality review.
- **Scenario 2 — K-12 tutoring assistant.** A SaaS vendor sells an LLM-powered tutoring assistant to US K-12 schools. Students include children under 13. Teachers can enable or disable per-student features. Assistant is fine-tuned on anonymised prior tutoring sessions.
- **Scenario 3 — Fintech fraud triage.** A US-incorporated non-bank fintech runs an ML model that scores transactions for fraud risk. A high score auto-declines the transaction; a medium score routes to a human reviewer. Model is retrained monthly on customer-transaction data.

## Starter guidance

- **Trigger first, scope second.** The diagram answers "which laws apply"; the tables answer "what changes about our pipeline". Keep them separate.
- **"Health-adjacent" is a trap.** A fitness app with no covered-entity relationship may not be under HIPAA — but may still be under state health-privacy laws, under FTC Act Section 5, under Washington's My Health My Data Act. Note the trap in your diagram.
- **COPPA scoping — actual knowledge is a verb.** If your product is 18+ by terms-of-service but you know children use it, you're inside COPPA. Age-gate design is engineering.
- **GLBA is a program, not a feature.** Safeguards Rule compliance is *every* system at the institution, not just the AI system. Your control-delta captures the AI-specific additions (service-provider oversight for foundation model, pen-test scope includes AI surface, IR plan includes AI-specific scenarios) — the rest is baseline Safeguards program that this exercise does not re-author.
- **Compose, don't duplicate.** Where HIPAA + GDPR + state law all require encryption-in-transit, write one row and tag all three regimes.

## Acceptance criteria

- [ ] The diagram covers all four sector regimes (HIPAA / FERPA / COPPA / GLBA Safeguards) plus GDPR / CCPA baseline + biometric-state-law branch.
- [ ] Every branch terminates deterministically — "it depends" with no follow-up is a fail.
- [ ] Each control-delta table has at least six rows, each with a specific test (not just a test name) and a specific evidence artifact (not just an artifact name).
- [ ] At least two rows in each table name a **deferral** (e.g., "BAA drafting — counsel", "forensics — SecOps / DFIR", "annual Safeguards board report — Head of AI Governance").
- [ ] The worked-scenario memo picks the hardest of the three scenarios and defends the choice in one sentence.
- [ ] The memo's trigger set and control-delta set are internally consistent — every control-delta row cites a trigger in the memo's trigger set.
- [ ] The memo names every out-of-ladder dependency (counsel / DPO / SecOps) and the handoff artifact you'd give them.

## Stretch goals

- Add a fifth branch for **US state health-privacy laws** (Washington's My Health My Data Act and any newer enacted analogues). Note the engineering delta against HIPAA where the law reaches consumer-health data outside HIPAA's covered-entity scope. <!-- needs-research: refresh My Health My Data analogues and effective dates. -->
- Add a sixth branch for **US state biometric laws** that reach vector embeddings of individuals (BIPA treatment of facial geometry is the most litigated). Consider whether your product's embeddings are reasonably identifiable.
- Write the **cross-walk column** on each control-delta row to NIST PF Subcategories (chapter 07) and ISO/IEC 27701 controls. The payoff: your sector-law controls and your enterprise-framework controls share evidence.

## What this lands

- The decision diagram is the first artifact in every scoping conversation with a new product team. Keep it as a reusable template.
- The control-delta tables are the input to Exercise 06 (role scope and deferral contract) — most deltas have deferrals, and naming them is the point.
- The worked-scenario memo is the DPIA-adjacent output; its shape reappears in `mod-111-dpia-and-privacy-reviews`.
