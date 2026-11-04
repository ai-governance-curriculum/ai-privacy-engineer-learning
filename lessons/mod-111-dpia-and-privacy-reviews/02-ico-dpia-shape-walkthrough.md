# The ICO DPIA Template Walkthrough — Populating Each Section from an AI System

Chapter 01 named the Art. 35(7) statutory floor. The ICO DPIA template is the concrete, free, authoritative skeleton most Anglophone controllers actually use to meet that floor — including controllers not under UK jurisdiction, because the ICO template is openly published, cross-checks the statutory structure, and reads sensibly to any supervisory authority that recognises Art. 35. This chapter walks the template section by section and names, for each, what to write and what artefact to cite when the subject of the DPIA is an AI system.

The reference you work from — download it and keep a copy in the DPIA repository:

- UK ICO — *Data protection impact assessment template* and *How to do a DPIA* guidance. [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/accountability-and-governance/data-protection-impact-assessments-dpias/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/accountability-and-governance/data-protection-impact-assessments-dpias/).
- UK ICO — *Guidance on AI and data protection* (the DPIA-for-AI sections). [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/guidance-on-ai-and-data-protection/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/guidance-on-ai-and-data-protection/).
- UK ICO — *AI and data protection risk toolkit*. [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/guidance-on-ai-and-data-protection/ai-and-data-protection-risk-toolkit/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/guidance-on-ai-and-data-protection/ai-and-data-protection-risk-toolkit/).

The ICO template's section headings (as of this writing — the ICO revises the template occasionally, so cross-check at ingestion):

1. **Submitting controller details**
2. **Step 1 — Identify the need for a DPIA**
3. **Step 2 — Describe the processing**
4. **Step 3 — Consultation process**
5. **Step 4 — Assess necessity and proportionality**
6. **Step 5 — Identify and assess risks**
7. **Step 6 — Identify measures to reduce risk**
8. **Step 7 — Sign off and record outcomes**

Each of these maps cleanly to Art. 35(7) — see the mapping table at the end of chapter 01. What follows is how to fill each section when the system is AI.

## Submitting controller details

Four fields: the name of the DPIA (the system name and version), the name and role of the person submitting (usually a product owner, with the privacy engineer named as co-author), the date, and the DPO name and contact. Nothing clever here — but two engineering hygiene notes:

- The system **version** matters. A DPIA that covers "Recsys" is less useful than one that covers "Recsys v4.2". The DPIA's lifetime is bounded by the version whose risk profile it describes; mod-109 ch. 06's system card carries the same version pin and the two must agree.
- The **submitting role** matters. If the DPIA lists only the privacy engineer as the submitter and not the business owner, the DPO and the auditor will ask who is accountable for the decisions the DPIA records. Accountability sits with the business owner; the privacy engineer is the technical co-author.

## Step 1 — Identify the need for a DPIA

The ICO prompts: *Explain broadly what the project aims to achieve and what type of processing it involves. You may find it helpful to refer or link to other documents, such as a project proposal. Summarise why you identified the need for a DPIA.*

The engineering answer is a one-page summary that ends with the Art. 35(3) / WP248 triage result. For an AI system, the summary should land:

- **What the system does** — in subject-facing language, not architecture language. "Recommends products to shoppers based on their browsing history" rather than "trains a two-tower retrieval model with contrastive loss".
- **Why it is needed** — the business purpose, the lawful basis under consideration, the expected benefit (to whom).
- **Why a DPIA is required** — the WP248 criteria it hits (evaluation / automated decision / systematic monitoring / sensitive data / large scale / matching / vulnerable subjects / innovative technology / rights restriction). Enumerate the hits, with one sentence each. "Innovative technology" is almost always a hit for a modern AI system; the Art. 29 WP248 specifically called out "applications of a new technological or organisational solution" as a trigger. Two or more hits is the practical threshold for "DPIA required"; AI systems commonly hit four.
- **The scope of the DPIA** — what processing operations are in scope, which versions, which regions. A DPIA that over-scopes (covers five unrelated projects) is as weak as one that under-scopes.

Include a reference to the controller's **DPIA screening policy** — the internal procedure that triggers the DPIA. The audit chain is: screening policy → system in scope → this DPIA.

## Step 2 — Describe the processing

This is the longest section and the one chapter 01's "(a) systematic description" guidance covers in full. For the ICO template specifically, the prompts are:

- **Describe the nature of the processing**: how data is collected, used, stored, deleted; what is the source; who is involved; what is the scale; who has access; who it is shared with; what the retention periods are; which processors are involved.
- **Describe the scope of the processing**: nature of personal data, special-category data, number of subjects, geographical scope, duration.
- **Describe the context of the processing**: subjects' reasonable expectations, control they have, prior public concern, novelty of the technology, expertise available.
- **Describe the purposes of the processing**: intended effect on subjects, expected benefits to the controller and third parties.

For each, cite:

- **Nature** — the DFD and the RoPA entry. In the DPIA body, name the stores (the six-corpus map from mod-110 ch. 01), name the retention policy per store, name the access-control model, name the data flows across the trust boundary. A one-paragraph narrative per the six corpora is enough if each paragraph cites the authoritative artefact.
- **Scope** — a numeric table. Number of subjects, number of inferences per month, special-category data categories, geographic scope, planned duration. These numbers must be measurable, not guessed; if you cannot measure them, cite the upper-bound estimate and its basis.
- **Context** — the hardest subsection for an AI engineer to write in a non-self-serving way. The subject's reasonable expectations depend on whether the service is consumer-facing or B2B, the age of the subject, the sensitivity of the context (a health-adjacent service has stricter expectations than a shopping service). The ICO's DPIA-for-AI guidance explicitly asks whether you have considered subjects' expectations about the use of AI; a DPIA that does not address this is incomplete.
- **Purposes** — one purpose per row, with lawful basis and Art. 22 flag per chapter 01. The "intended effect" column is where you name what the model does *to* the subject — not just *for* the subject.

AI-specific items the ICO guidance adds here:

- **Training-data provenance** — a documented chain from each training corpus to its source, licence, and the consent / legitimate-interests basis under which it was collected. Scraped datasets need particular attention; the EDPB Opinion 28/2024 (chapter 04) makes this a first-order compliance question.
- **The model's role in decisions** — explicitly whether the model is the sole decision-maker (Art. 22 territory), a decision-support (human-in-the-loop with a described protocol), or an informational tool.
- **Fairness and bias considerations** — the ICO AI guidance treats model bias as a privacy issue because discriminatory outputs engage both Art. 5(1)(a) (lawfulness and fairness) and the rights-and-freedoms assessment in Art. 35. A pointer to the fairness evaluation artefact — the group-fairness metrics computed on the eval set — belongs here.

## Step 3 — Consultation process

The ICO prompts: *Consider how to consult with relevant stakeholders. Describe when and how you will seek individuals' views, or justify why it's not appropriate to do so. Who else do you need to involve within your organisation? Do you need to ask your processors to assist? Do you plan to consult information security experts, or any other experts?*

The common failure is to treat this section as a list of internal colleagues. Art. 35(9) actually asks: *Where appropriate, the controller shall seek the views of data subjects or their representatives on the intended processing*. For an AI system, "where appropriate" is broader than controllers tend to assume.

Fill in:

- **Internal consultation** — DPO (mandatory under Art. 35(2)), information security, engineering, product, legal, where relevant HR / medical / safety specialists. Record the dates of consultation; the DPO consultation is the one the audit will pin on specifically.
- **Processor consultation** — if a processor runs the inference (an API vendor, a cloud provider, a managed-vector-store), you have asked them to assist with the DPIA under Art. 28(3)(f). Attach the DPIA input they provided, or record why you did not consult them (and explain how you obtained the facts you needed).
- **Subject consultation** — if the scale, sensitivity, or novelty warrants it, run a user study, a panel, a survey of expected users, or engage a consumer-rights body. For employee-monitoring AI, consultation with workers' representatives is near-mandatory. For consumer-scale systems with vulnerable subjects, a formal subject-consultation step strengthens the DPIA. If you did not do this, justify why — "a survey of 1,200 users ran on 2026-07; the summary report is attached as Appendix B" or "subject consultation was not run because the processing is internal and does not affect individuals outside the controller; this was agreed with the DPO".
- **External expert consultation** — infosec consultants, academic collaborators, civil-society reviewers. Any external review that bears on the risk assessment belongs here, with the reviewer named and the output attached.

## Step 4 — Assess necessity and proportionality

The ICO prompts: *Describe compliance and proportionality measures, in particular: what is your lawful basis for processing; does the processing actually achieve your purpose; is there another way to achieve the same outcome; how will you prevent function creep; how will you ensure data quality and data minimisation; what information will you give individuals; how will you help to support their rights; what measures do you take to ensure processors comply; how do you safeguard any international transfers?*

This is chapter 01's "(b)" section landed in the ICO structure. One section-by-section guide:

- **Lawful basis** — the Art. 6(1) basis (consent, contract, legal obligation, vital interests, public task, legitimate interests) and, for special-category data, the Art. 9(2) exception. For legitimate interests, attach the LIA. Chapter 04 of this module makes the AI-specific balancing test (EDPB Opinion 28/2024 three-step) explicit.
- **Does the processing achieve the purpose** — the empirical question. If you are deploying a fraud-detection model, do you have evidence it detects fraud (precision, recall, false-positive rate on representative populations)? "The model is in production" is not evidence it achieves the purpose; pointing to the evaluation report is.
- **Alternatives considered** — this is where the chapter-01 minimisation / proportionality tests cash out. "Could you use less data? Could you use aggregate instead of individual data? Could you train on synthetic or DP data?" are the ICO's own framing questions. Record the alternatives considered and the reason the current design was chosen. A DPIA that records no alternatives considered will be suspected of having considered none.
- **Function creep prevention** — purpose-limitation (Art. 5(1)(b)) wiring. For AI systems this is particularly important because the same model can serve multiple purposes; name the technical controls that keep the inference pathway bound to the stated purpose (per-purpose API gating, per-purpose feature gating, retention per purpose).
- **Data quality and minimisation** — Art. 5(1)(c) and 5(1)(d). The engineer's contribution: the feature-set manifest and the justification per feature; the data-quality monitoring that detects drift; the minimisation wiring in the ingest pipeline (truncation, hashing, tokenisation that drops identifiers).
- **Transparency to individuals** — the privacy notice, the Art. 13 / 14 information, the Art. 15(1)(h) logic description. Chapter-01 (f) and mod-112 both pin this; the DPIA cites the artefact.
- **Support for rights** — the DSR runbook set from mod-110 ch. 01. The DPIA cites the per-right runbook and the DSR-SLA.
- **Processor compliance** — Art. 28 data-processing agreements with each processor; the DPIA cites the DPA register.
- **International transfers** — Art. 44–49 compliance. For AI, this is particularly salient when inference calls a model hosted in a different jurisdiction (OpenAI endpoint from a European controller, for example). SCC / BCR / adequacy / derogation basis per transfer, documented in a transfer-impact assessment (TIA) the DPIA references.

## Step 5 — Identify and assess risks

The ICO prompts: *Describe source of risk and nature of potential impact on individuals. Include associated compliance and corporate risks as necessary. Score each risk on likelihood and severity.*

Chapter 01 covered the Art. 35(7)(c) substance. For the ICO template specifically, use the two-column ICO risk table with the three-column scoring:

| Describe source of risk and nature of potential impact on individuals | Likelihood of harm | Severity of harm | Overall risk |
|---|---|---|---|
| ... | Remote / possible / probable | Minimal / significant / severe | Low / medium / high |

The ICO's three-point scale is coarser than the CNIL / ISO four-point scale; most DPIA programmes use the ICO wording in the template and map the engineering risk register (the chapter-01 four-point scale) onto the ICO three-point presentation. The composite ("overall risk") drives the Art. 36 decision.

The AI-specific feared-event list from chapter 01 is the content. Each row should be subject-facing: "A subject's prompt that reveals a medical condition is leaked via infrastructure compromise or via model-extraction, causing loss of privacy, potential discrimination, and distress" is a legitimate row. "SQL injection against the backend" is not a Step 5 row — it is an Art. 32 security-register row the DPIA references.

A DPIA for an AI system should typically have **8–15 rows** in Step 5. Fewer suggests under-identification; many more suggests you have not consolidated related threats. Chapter 03 shows how the NIST PRAM worksheet decomposition keeps the row count manageable without missing threats.

## Step 6 — Identify measures to reduce risk

The ICO prompts, per identified risk: *Identify additional measures you could take to reduce or eliminate risks identified as medium or high risk. Options to reduce or eliminate risk. Effect on risk (eliminated / reduced / accepted). Residual risk (low / medium / high). Measure approved (Y/N).*

This is chapter 01's "(d)" landed in the ICO structure, and the ICO format is tight — one row per measure, one explicit residual-risk reclassification, one approval flag. The engineer's job is to write the "Options to reduce or eliminate risk" column concretely. Example rows for an AI system:

| Risk | Options to reduce or eliminate risk | Effect on risk | Residual risk | Measure approved |
|---|---|---|---|---|
| Training-data memorisation extractable by adversary | DP-SGD training at ε ≤ 3.0, δ ≤ 1e-5; annual MIA audit against held-out set; output filter against verbatim training sequences of length ≥ 50 tokens | Reduced | Low | Yes — ML Lead 2026-09-12 |
| Prompt log containing subject's medical condition leaks via bucket compromise | PII redaction at ingest (Presidio + custom recognisers, FP ≤ 2%, FN ≤ 3% on held-out eval); encryption at rest with customer-managed keys; prompt-log retention reduced from 180 to 14 days; access logged and reviewed monthly | Reduced | Low | Yes — Sec Lead 2026-09-14 |
| Inferred-attribute disclosure (model emits derived sensitive-category attribute) | Output filter (classifier on completions, calibrated on held-out set); refusal for prompts matching the sensitive-topic regex; red-team run with the OWASP LLM Top 10 as the probe set | Reduced | Medium | Yes — Privacy Eng 2026-09-15 |
| Subject cannot exercise erasure against the trained model within the GDPR SLA | SISA-partitioning on the next retrain cycle; approximate unlearning runbook with MIA-based verification; documented erasure-vs-retrain decision tree (mod-110 ch. 05); scheduled quarterly retrain cadence | Reduced | Medium | Yes — ML Lead 2026-09-16 |

Every "Reduced / Eliminated" cell must be defensible from an artefact. "Reduced" without a measured delta is a claim; "reduced from a measured MIA advantage of 0.08 to 0.03 after DP-SGD training" is evidence. The ICO guidance is explicit that measures should be *specific, implementable, and verifiable*.

### The decision on Art. 36 prior consultation

If any row's residual risk remains High *after all approved measures have been applied*, Art. 36 is engaged. The DPIA names the Art. 36 decision in this section — either "Residual risks are all Low or Medium; Art. 36 consultation is not required" or "Residual risk on rows N and M remains High; Art. 36 consultation with [supervisory authority] is scheduled on [date]". The DPO co-signs the decision in Step 7.

## Step 7 — Sign off and record outcomes

The ICO prompts: *Item / Name / Date / Notes*, with rows for:

- **Measures approved by** — the accountable product owner.
- **Residual risks approved by** — the controller (or the DPO's delegate, depending on the governance model). This is the accountability signature.
- **DPO advice provided** — the DPO's advice, with date, in text in the template or attached as a memo.
- **Summary of DPO advice**.
- **DPO advice accepted or overruled by** — with reasons if overruled.
- **Comments** — including review date.
- **Consultation responses reviewed by** — if subject consultation was run.
- **This DPIA will kept under review by** — the role and the review cadence.

AI-specific recommendations for Step 7:

- **Set an explicit review cadence** tied to model re-training. "Reviewed on each major model version and at least annually" is a strong norm; "reviewed annually" is weaker.
- **Name the trigger list** — the engineering events that trigger an early review (chapter 01 "The DPIA as a living document"). The DPO review-schedule cite the list.
- **Attach the artefact manifest** — the regulator-facing bundle (chapter 06). The DPIA is the primary document; the bundle manifest is the appendix index. An auditor will look for the manifest in Step 7's notes.

## Common failure modes the ICO (and other DPAs) catch

The ICO publishes enforcement notices and case studies; a pattern of common DPIA failures emerges. Watch for:

- **Vague systematic description** — processing described at marketing-brochure altitude. Fix: cite the DFD and the RoPA for every processing operation.
- **Necessity assertion without alternatives** — "the processing is necessary" with no record of what alternatives were considered. Fix: write down the alternatives you considered and the reason you rejected each.
- **Risk assessment that is a security assessment** — threat categories are "DDoS", "SQL injection", "unauthorised access". These are relevant but are not the Art. 35(7)(c) register. Fix: translate each threat into the subject-facing harm.
- **Mitigation list without evidence** — measures asserted but not instrumented. Fix: cite the artefact for each measure; if the artefact does not exist, change the measure to one that is implemented.
- **Residual-risk minimisation** — all risks marked Low after mitigation, with no honest acknowledgement of the residual risk a PET's mathematical guarantee cannot eliminate. Fix: treat residual Medium as an honest outcome; "all risks Low" is a red flag to any experienced DPO.
- **DPO advice as a rubber stamp** — same-day sign-off with no recorded dialogue. Fix: have the DPO review with real friction; record the dialogue.
- **Missed subject consultation** — justification "the subjects don't understand AI" is not acceptable; neither is skipping consultation when scale and sensitivity warrant it. Fix: run the consultation or justify its absence with specifics.
- **Scope drift between DPIA and reality** — the system changes under the DPIA. Fix: version the DPIA against the system version; refresh at each major change.
- **No link to a Transfer Impact Assessment** — international transfers glossed. Fix: if any processor or inference endpoint is in a third country, attach the TIA or justify its absence.

## An AI-specific DPIA template addendum

Many controllers run the ICO template with an AI-specific appendix. The appendix adds:

- **Model-card snapshot** — parameters, training-data manifest, lawful-basis summary per data source, eval-set composition.
- **Training-data provenance** — one row per corpus, with licence, consent / LI basis, scraping / collection method, PII redaction applied at ingest.
- **PET deployment claims** — DP (ε, δ, accountant output), federated aggregation (secure-aggregation protocol, participant threshold), confidential computing (TEE platform, attestation certificate hash), K-anonymity / L-diversity parameters for released artefacts.
- **Evaluation artefacts** — MIA / extraction audit report; redaction FP / FN report; group-fairness evaluation; adversarial robustness evaluation where relevant.
- **DSR capability statement** — the mod-110 chapter 07 audit-trail schema, with the per-right SLA and the engineering runbook pointer.
- **Incident response** — the breach-response plan, Art. 33 / 34 notification pathways, who-notifies-whom, and the 72-hour clock arithmetic.

The appendix is not required by the ICO template — but it lands the evidence the auditor will ask for in Step 5 and Step 6 in a predictable place, and it is what chapter 06 of this module assembles as the regulator-facing bundle.

## Summary — what you can do after this chapter

- Fill each of the seven ICO template steps for an AI system, citing the engineering artefact that evidences each claim.
- Produce a one-page Step 1 that triages against the WP248 nine criteria and lands the Art. 35 trigger.
- Write a Step 2 that cites the DFD, the RoPA, the six-corpus retention table, the training-data provenance chain, and the per-purpose row.
- Run Step 3 as a real consultation — DPO, processors, subjects or representatives, external experts — and record the dialogue.
- Articulate Step 4 as a necessity-and-proportionality reasoning, with alternatives considered and the chosen-design justified.
- Populate Steps 5–6 from the chapter-01 feared-event register and the chapter-06 bundle, with measured deltas on risk.
- Run Step 7 with an explicit review cadence, a trigger list, and an attached artefact manifest.
- Recognise and avoid the common DPIA failure modes the ICO highlights.

Chapter 03 overlays the NIST PRAM worksheets on Step 5's risk assessment — a second, more structured pass that produces the NIST-PF-compatible evidence pack US-side or NIST-aligned controllers expect.
