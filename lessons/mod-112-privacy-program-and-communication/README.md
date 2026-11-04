# mod-112-privacy-program-and-communication: Cross-Functional AI Privacy Program and Communication

**Estimated effort:** 14 hours

The preceding eleven modules produced the technical artefacts an AI privacy engineer makes — the threat model, the data inventory, the redaction pipeline, the memorisation audit, the DP accountant output, the FL protocol, the TEE attestation, the RAG system card, the unlearning capability, the DPIA and regulator bundle. This module is where those artefacts stop being standalone deliverables and start running as a *programme* — a recurring, cross-functional operating rhythm that your team, your partner functions, and your executive sponsors can rely on. The deliverables here are the ones a regulator or board will read: the programme charter, the DPO partnership model, the model-card privacy slice, the review-board decision log, the incident runbook, the audit-serving crosswalk, and the credentials portfolio that qualifies the humans behind them.

The level-30 remit is *a team* — one product line, one ML platform, one business unit. Enterprise control-library architecture belongs to the senior AI governance architect (level 50); board-level reporting belongs to the head of AI governance (level 60). Chapter 08 names those boundaries explicitly so you can operate the full scope of level 30 confidently and escalate what belongs upward without hesitation.

## Learning objectives

- Own an AI privacy program at team scope — cadence, artefacts, decision-log templates, sign-off routing — across product / ML / infra / security / governance / legal / DPO.
- Coordinate with the DPO (statutory role under GDPR Art. 37–39) — understand the DPO's independence and advisory position, and how a privacy-engineering function partners with the DPO rather than under them.
- Author the privacy slice of a model / system / dataset card — training-corpus consent-basis statement, PII-pipeline claim, memorisation-eval residual-risk, DP (ε, δ) / FL / TEE / HE deployment claim, unlearning-capability statement, logging policy — using the Hugging Face model-card format or an internal template.
- Run a privacy review board — pre-read deck, one-pager, decision log — that captures the privacy decisions the board owns; produce minutes that survive audit.
- Run AI-specific privacy-incident response — triage a suspected PII exposure, LLM-leakage incident, or DSR failure; decide reportability under GDPR Art. 33 / 34 (72-hour breach notification), CCPA breach notification, HIPAA breach notification; coordinate with legal / DPO / security / communications; feed the incident back into the LINDDUN model and the privacy-eval regression suite.
- Cross-map the role's evidence to NIST Privacy Framework functions, ISO/IEC 27701 PIMS clauses, ISO/IEC 42001 AIMS controls, and the Cloud Security Alliance AI Controls Matrix — produce the crosswalk that lets one evidence pack serve multiple audits.
- Position the credentials portfolio — IAPP CIPT (privacy-engineering-focused), IAPP CIPP/E and CIPP/US (jurisdictional), IAPP CIPM (program), IAPP AIGP (AI governance), ISO/IEC 27701 LI/LA — and know which are load-bearing in enterprise hiring.
- Cite the boundary to `senior-ai-governance-architect` (level 50) on control-library architecture and cross-jurisdiction reconciliation; to `head-of-ai-governance` (level 60) on board-level reporting and regulator engagement.

## Chapters

1. [Owning an AI privacy program at team scope](01-ai-privacy-program-at-team-scope.md) — the five commitments (cadence, artefact set, decision log, sign-off routing, escalation path); partner RACI; failure modes; the one-page charter the exercise produces.
2. [Partnering with the Data Protection Officer](02-partnering-with-the-dpo.md) — GDPR Art. 37–39 in practice; what the DPO is and is not; monthly 1:1, review-board advice-of-record, DPIA consultation, incident interface; conflict-of-interest guardrails.
3. [The privacy slice of a model, system, and dataset card](03-model-system-dataset-card-privacy-slice.md) — the eight privacy claims (training-corpus consent, PII pipeline, memorisation residual risk, DP, FL, TEE / HE / MPC, unlearning, logging); the Hugging Face template mapping; three claim-quality disciplines (checkability, falsifiability, honesty-about-limitation).
4. [Running a privacy review board](04-privacy-review-board.md) — attendee list, cadence, pre-read, one-pager, minute discipline; sign-off RACI in action; the quarterly retrospective; the audit-readiness view.
5. [AI-specific privacy incident response](05-ai-privacy-incident-response.md) — the five-phase flow (signal, triage, containment, reportability, after-action); GDPR Art. 33 / 34 decision; Cal. Civ. Code § 1798.82 / 29; HIPAA Breach Notification Rule; SEC 8-K Item 1.05; feedback into LINDDUN and the eval suite.
6. [The evidence crosswalk — one evidence pack, many audits](06-evidence-crosswalk-nist-iso-csa.md) — NIST Privacy Framework subcategories, ISO/IEC 27701 PIMS controls, ISO/IEC 42001 AIMS clauses, CSA AI Controls Matrix; the one-artefact-many-rows pattern; residual-gap discipline; the operating rhythm.
7. [Positioning the credentials portfolio](07-credentials-portfolio.md) — IAPP CIPT, CIPP/E, CIPP/US, CIPM, AIGP; ISO/IEC 27701 and 42001 LI/LA; ISACA CDPSE and the security-adjacent set; the 24-to-36-month portfolio plan.
8. [The boundary to senior AI governance architect (L50) and head of AI governance (L60)](08-boundary-to-senior-architect-and-head.md) — the six L50 escalation triggers; the five L60 escalation triggers; the escalation hand-off packet; scope creep up and scope abandonment down.

## Exercises

See [`exercises/`](exercises/):

1. [Privacy program charter for one team](exercises/exercise-01-privacy-program-charter-for-one-team.md) — author the one-page charter plus the partner RACI matrix for a specified team.
2. [DPO engineering partnership model](exercises/exercise-02-dpo-engineering-partnership-model.md) — build the rhythm, briefing shapes, and consultation records that make the DPO partnership statutorily compliant and operationally real.
3. [Model card privacy slice authoring](exercises/exercise-03-model-card-privacy-slice-authoring.md) — produce the eight-claim privacy slice for a chosen AI system, in Hugging-Face-compatible form.
4. [Privacy review board pre-read and decision log](exercises/exercise-04-privacy-review-board-pre-read-and-decision-log.md) — run a simulated review board for a specific decision; produce pre-read, one-pager, minutes, and decision-log entry.
5. [AI privacy incident response drill](exercises/exercise-05-ai-privacy-incident-response-drill.md) — run the five-phase response for one of the three canonical scenarios (memorisation leak / cross-tenant leak / DSR failure); produce scoping note, reportability decision record, notification draft, and after-action.
6. [Evidence crosswalk — NIST PF / ISO 27701 / ISO 42001 / CSA AICM](exercises/exercise-06-evidence-crosswalk-nist-pf-iso-27701-iso-42001-csa.md) — build the working crosswalk for the artefact set from prior modules, with residual-gap and version-pinning discipline.
7. [Credentials portfolio planner](exercises/exercise-07-credentials-portfolio-planner.md) — personal 24-to-36-month portfolio plan plus the team-level credentials posture.

## Structure

- `01-…md` … `08-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs (scaffolded placeholder).
- `quizzes/`: knowledge checks (scaffolded placeholder).
- `resources.md`: external references — GDPR DPO and breach-notification articles, EDPB and WP29 guidance, ICO guidance, HIPAA Breach Notification Rule, California breach statutes, NIST Privacy Framework and AI RMF, ISO/IEC 27701 and 42001, CSA AI Controls Matrix, EU AI Act, Hugging Face model cards and Datasheets for Datasets, IAPP credential body-of-knowledge pages.

## Where this module sits in the track

- Upstream dependencies: mod-101 (frameworks and position — the regulatory map the programme operates inside), mod-102 (threat modelling — the register that seeds the risk column), mod-103 (data governance — the inventory and purpose register the programme runs against), mod-104 (redaction — the FP/FN measurements the model card carries), mod-105 (memorisation and MIA — the residual-risk evidence), mod-106 (DP engineering — the ε, δ claim), mod-107 (FL and secure aggregation — the aggregation protocol claim), mod-108 (confidential computing — the attestation claim), mod-109 (RAG and agent scoping — the system card and the logging policy), mod-110 (unlearning and DSR — the capability statement and the runbooks), mod-111 (DPIA and privacy reviews — the regulator-facing bundle this module's programme runs against).
- Downstream consumers: no further modules in this track; this module closes the track. The level-35 `ai-evaluation-engineer` consumes the module's artefacts (incident records, after-actions, review-board decisions) into release assurance.
- Hand-off upstream: the DPO (horizontal partner, chapter 02); the senior AI governance architect at level 50 (chapter 08); the head of AI governance at level 60 (chapter 08).
