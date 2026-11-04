# The NIST PRAM Worksheets End-to-End — Building a NIST-PF-Compatible Evidence Pack

The GDPR Art. 35 DPIA is the European-law instrument; the US-federal and US-regulated-enterprise equivalent is NIST's Privacy Risk Assessment Methodology (PRAM), the practical operationalisation of the NIST Privacy Framework. The two are not substitutes — a well-run privacy programme runs both, because the PRAM uncovers risks the DPIA framework leaves implicit (specifically, the "problematic data action" taxonomy that pulls out harms the GDPR risk-to-rights-and-freedoms section can gloss over). For an AI system that will face both European and US scrutiny, a DPIA that incorporates the PRAM worksheet output as Appendix 3 is the standard shape.

This chapter walks the PRAM end-to-end — Worksheet 1 (Framing), Worksheet 2 (Data Action and Context assessment), Worksheet 3 (Prioritisation), with Worksheet 4 (Response) added by many practitioners — and maps the output into the NIST Privacy Framework's Core (Functions, Categories, Subcategories) so the evidence pack an auditor expects lands coherently.

The primary references — download and pin:

- NIST — *Privacy Framework v1.0* (2020). [nist.gov/privacy-framework](https://www.nist.gov/privacy-framework).
- NIST — *Privacy Risk Assessment Methodology (PRAM)* resource page, including the current workbook. [nist.gov/privacy-framework/resource-repository/browse/privacy-risk-assessment-methodology-pram](https://www.nist.gov/privacy-framework/resource-repository/browse/privacy-risk-assessment-methodology-pram).
- NIST — *NISTIR 8062, An Introduction to Privacy Engineering and Risk Management in Federal Systems* (2017). [nvlpubs.nist.gov/nistpubs/ir/2017/NIST.IR.8062.pdf](https://nvlpubs.nist.gov/nistpubs/ir/2017/NIST.IR.8062.pdf).
- NIST — *AI Risk Management Framework, AI RMF 1.0* (2023), and the AI RMF Generative AI Profile (NIST AI 600-1, 2024). [nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework).
- NIST — *SP 800-53 Rev. 5* (PT family specifically). [doi.org/10.6028/NIST.SP.800-53r5](https://doi.org/10.6028/NIST.SP.800-53r5).

## The PRAM's conceptual move — "data actions" instead of "data processing"

The GDPR thinks in terms of *processing operations* — collect, store, use, disclose, erase. The PRAM thinks in terms of *data actions*: the operations a system performs on personal data, decomposed finely enough that each one carries its own risk profile. Three top-level data action buckets:

- **Data Action A — Collection**: how personal data enters the system (first-party collection from the subject, third-party acquisition, scraping, inference / derivation).
- **Data Action B — Processing / Usage**: how personal data is operated on inside the system (storage, transformation, training, inference, analytics, sharing within the organisation).
- **Data Action C — Disclosure**: how personal data leaves the system (to subjects, to third parties, to processors, to the public, to regulators, through model outputs).

For an AI system, the useful decomposition is finer than three buckets. A standard engineering PRAM for an AI system lists the following data actions, each as its own worksheet row:

- A1. Collection of training-data corpora (per-corpus, including scraped sources).
- A2. Collection of inference-time inputs (prompts, features, uploaded documents).
- A3. Collection of operational telemetry (prompt / completion logs, feedback, feature-store lookups).
- B1. Training (one row per training run; retraining and fine-tuning each count).
- B2. Indexing / embedding (retrieval corpora, derived feature stores).
- B3. Inference (one row per deployment surface).
- B4. Logging and analytics on telemetry.
- B5. Derivative-product production (fine-tunes, distillations, synthetic data).
- B6. Internal sharing to downstream teams / systems.
- C1. Disclosure to the subject (model output back to the subject).
- C2. Disclosure to third parties (processors, vendors, downstream controllers).
- C3. Disclosure through model outputs to other subjects (one subject's query reveals another subject's data).
- C4. Release of derivative artefacts (published fine-tune, open-source model, synthetic dataset).
- C5. Cross-border transfers (inference in a different jurisdiction from the subject).

A PRAM that lists fewer than ten data actions for a nontrivial AI system has almost certainly aggregated too aggressively; the point of the decomposition is that each action carries a different combination of problematic data actions, controls, and residual risk.

## Worksheet 1 — Framing

The framing worksheet establishes what the PRAM is about: the system, its purpose, the organisational mission, the business value, the laws and regulations in scope, and the privacy objectives the exercise will measure against. The engineer's content here is similar to Step 1 of the ICO template (chapter 02), but the framing is more explicitly organisation-strategic — a PRAM reader expects to see the system positioned in the organisation's mission.

Fields to fill, with the AI-engineering-specific content:

- **System name and description** — one or two paragraphs, same substance as the DPIA Step 1.
- **System owner and privacy lead** — the accountable product owner and the privacy engineer co-authoring.
- **Mission relevance** — how the system supports the organisation's mission. For a public-sector deployment this is a formal statement; for private sector it is the business purpose.
- **Business value** — the specific benefit the system delivers, quantitative where possible.
- **Applicable laws and regulations** — the compliance perimeter. For a US-only system: Federal Trade Commission Act §5 (unfair / deceptive practices), state privacy laws (CCPA / CPRA, Virginia CDPA, Colorado CPA, Connecticut CTDPA, Utah UCPA, Texas TDPSA, and the growing bench), HIPAA if PHI is in scope, GLBA if financial data is in scope, Fair Credit Reporting Act, Children's Online Privacy Protection Act, Americans with Disabilities Act (for public-facing services), sector-specific rules. For dual-jurisdiction systems: GDPR / UK GDPR / Data (Use and Access) Act where applicable. For federal systems: the Privacy Act of 1974, the E-Government Act of 2002 §208 (which mandates a PIA), the FISMA security-controls baseline, OMB Memorandum M-24-10 (AI in federal agencies), SP 800-53 Rev 5.
- **Organisational privacy principles** — the organisation's stated principles (often a FIPPs mapping or an internal policy). A PRAM anchored to concrete principles is auditable; one anchored to "we respect privacy" is not.
- **Privacy risk statement** — a two-sentence statement of what the organisation is trying to avoid. "The organisation seeks to avoid harm to data subjects arising from the AI system's processing, including but not limited to reputational, financial, discriminatory, and autonomy harms, and to comply with the privacy laws in scope."

The framing worksheet also lists the **stakeholders** whose views the assessment will consider: subjects (end users, third parties), the organisation (controllers and processors), the public, and regulators. For an AI system that affects non-user subjects (e.g., a model trained on scraped web data processes the privacy interests of the authors of that content), the non-user-subject category must be named explicitly — the PRAM's stakeholder enumeration is an easy place to miss them.

## Worksheet 2 — Assessing Data Actions

Worksheet 2 is the core: for each data action (A1, A2, ...), the engineer fills:

- **Describe the data action** — what the system actually does. Concrete: "Training of retrieval-tower encoder on corpus `train-v4-2025Q4`, 230M examples, examples are 512-token chunks of scraped e-commerce product reviews with reviewer handles and timestamps attached."
- **Describe the data elements** — the personal data categories touched. For each: whether it is identifiable, sensitive (NIST uses "sensitive" more broadly than GDPR's "special category"), linked / linkable, retention, source, lawful basis.
- **Describe the context** — subject expectations, visibility of the action to the subject, the subject's control.
- **Identify problematic data actions** — the taxonomy below. This is the PRAM's distinctive step.
- **Enumerate potential problems for individuals** — the harm as the subject experiences it (parallels Art. 35(7)(c) feared events).
- **Enumerate potential problems for the organisation** — reputational, legal, compliance, operational.

### The problematic-data-action taxonomy — nine categories

NIST's PRAM uses nine problematic data actions (the list derives from NISTIR 8062 and is refined in the PRAM workbook):

1. **Appropriation** — personal data used in ways that exceed individuals' expectations or authorisation, or that cause them to feel they have lost control. AI systems trip this when training data is used for purposes the subject did not anticipate (scraped reviews used to train a recommender targeting the reviewer, for example).
2. **Distortion** — the system misrepresents the subject (incorrect inference, biased classification, hallucinated biographical details). LLMs are particularly prone; the "hallucinate a defamatory biographical fact" risk sits here.
3. **Induced disclosure** — the system causes the subject to disclose more data than they intend (a chatbot that elicits sensitive information via its interaction style).
4. **Insecurity** — breakdown of confidentiality, integrity, or availability (the Art. 32 security concern, mapped as a privacy risk because insecurity is the vector for other harms).
5. **Re-identification** — the system allows or enables identification of subjects where the system was supposed to have anonymised or de-identified them. Membership inference, model inversion, and K-anonymity failures land here.
6. **Stigmatization** — the system associates personal data with a characteristic that creates unjust social consequences. An inferred-attribute disclosure (model emits a derived sensitive-category attribute) is the canonical case.
7. **Surveillance** — personal data is tracked or monitored to a degree that is disproportionate to the purpose. Prompt-log retention beyond operational necessity is the AI case.
8. **Unanticipated revelation** — the system aggregates data in a way that reveals information the subject did not expect (a RAG system that surfaces a document the subject forgot about; a profile model that reveals implicit knowledge).
9. **Unwarranted restriction** — the system constrains or denies access to information, services, or opportunities beyond reasonable bounds. The Art. 22 adverse-decision case lives here.

For each data action, the engineer marks which of the nine apply, with a one-sentence rationale. Multiple problematic data actions per data action is normal — a prompt-log retention beyond need is Surveillance *and* Insecurity (because longer retention is more data to lose).

### Worked PRAM row — the "indexing / embedding" data action for a RAG system

For illustration, a filled row for data action B2 (indexing / embedding) of a customer-support RAG system:

- **Data action**: Chunk customer-support documents into 256-token passages, generate embeddings with `bge-large-en-v1.5`, persist to Qdrant with metadata (subject_id if the document is subject-authored, source URL, created_at). ~3M chunks at ingest; ~50K new per week.
- **Data elements**: Subject-authored support ticket text (contains PII — names, emails, order IDs, occasionally health references). Subject_id links chunks back to the subject. No special-category flag at ingest.
- **Context**: Subjects created the tickets with reasonable expectation that their content would be used by support agents to resolve the ticket. Expectation about use as training / retrieval material for an AI model is weaker — most subjects would not infer this without a privacy-notice disclosure.
- **Problematic data actions**:
  - Appropriation — the use for AI retrieval is beyond the original ticket-handling expectation.
  - Insecurity — the embedding store is a confidentiality-sensitive asset.
  - Re-identification — nearest-neighbour search on embeddings can recover the subject behind a near-duplicate chunk even if subject_id is scrubbed.
  - Unanticipated revelation — RAG surfaces a document the subject may have forgotten about, to a support agent other than the one who originally handled it.
- **Problems for individuals**: Loss of control over use of support-ticket content; disclosure of sensitive content to a support agent outside the original context; potential inferred-attribute exposure if the model's output summarises patterns across the subject's tickets.
- **Problems for the organisation**: Regulatory exposure (Art. 5(1)(b) purpose limitation challenge); reputational exposure if the practice is disclosed and perceived as overreach; litigation exposure if a specific subject sues over disclosure.

Fifteen such rows for a nontrivial AI system is a complete Worksheet 2.

## Worksheet 3 — Prioritising Privacy Risks

Worksheet 3 takes the problematic-data-action set per data action and scores each on **likelihood** and **impact**, producing a prioritised risk list the response (Worksheet 4) will address. The PRAM-specific scoring:

- **Likelihood**: the probability that the problematic data action will occur. Scored low / moderate / high, or on a numeric 1–10 scale depending on the PRAM variant. The scoring input: the technical controls in place, the threat model, the attacker's capability, the baseline rate of similar incidents in comparable systems.
- **Impact on individuals**: the harm severity to the subject when the problematic data action occurs. Low / moderate / high, or numeric. The scoring input: the sensitivity of the data, the reversibility of the harm, the number of subjects affected, the vulnerability of the subject population.
- **Impact on the organisation**: parallel score for organisational harm. Regulatory penalties, litigation costs, reputational damage, operational cost.

The PRAM's distinctive feature: the two "impact" columns are separate — the PRAM does not conflate subject harm with organisational harm. This is important for an engineer-authored assessment, because the two can and do diverge. A medium-sensitivity data item leaked at scale can be high impact on the organisation (big fine) and only moderate impact on each individual subject; a highly sensitive data item leaked for one subject can be high impact on the subject and only low impact on the organisation. Both are real risks; the DPIA under Art. 35(7)(c) attends to the subject-harm dimension primarily.

The prioritised list (sorted by composite risk) is the input to Worksheet 4 and to the mitigation column of the DPIA Step 6. The mapping is explicit: each row in the PRAM's Worksheet 3 corresponds to at least one row in the DPIA's risk register.

## Worksheet 4 — Response (where PRAM meets the DPIA mitigations)

Many PRAM workbooks explicitly include a response worksheet; where they do not, the engineer adds one. For each prioritised risk, record:

- **Response option** — avoid (do not perform the data action), mitigate (reduce likelihood or impact), transfer (contract or insurance), accept (document the acceptance and the accountable signer).
- **Specific controls applied** — concrete controls, cited to the artefact. Each control is one or more NIST SP 800-53 Rev. 5 control identifiers (AC-3, PT-2, PT-3, SI-10, SI-19, AU-11, DM-1, IP-1, etc.) and one or more NIST Privacy Framework Subcategory identifiers (see the next section).
- **Residual risk** — the risk after the response is applied, scored the same way as Worksheet 3.
- **Owner and due date** — the person accountable and the implementation / review date.

This worksheet is often the bulk of what the DPIA Step 6 mitigation table consumes — the DPIA cites the PRAM response worksheet directly.

## Mapping PRAM output onto the NIST Privacy Framework Core

The NIST Privacy Framework Core has five Functions, each with Categories and Subcategories:

- **Identify-P (ID-P)** — inventory and mapping, business environment, risk assessment, data processing ecosystem risk management.
- **Govern-P (GV-P)** — governance policies, processes, risk management strategy, awareness and training, monitoring and review.
- **Control-P (CT-P)** — data processing policies, data processing management, disassociated processing.
- **Communicate-P (CM-P)** — communication policies, data processing awareness.
- **Protect-P (PR-P)** — identity management, authentication, data security, maintenance, protective technology.

For each PRAM Worksheet 4 row, the engineer tags the applicable Subcategories. Examples (the Subcategory identifiers below follow the Privacy Framework v1.0 structure; cross-check against the current version):

- Training-data provenance documentation → **ID.IM-P1** (systems / products / services that process data are inventoried), **ID.IM-P2** (owners or operators of systems that process data are inventoried), **ID.DE-P4** (data processing transparency is managed).
- DP training at ε = 3.0 → **CT.DP-P1** (data processing is managed consistent with the organisation's risk strategy), **CT.DP-P4** (disassociated processing — DP is the canonical PF example).
- MIA / extraction audit → **ID.RA-P4** (threats to individuals are identified), **PR.PO-P5** (vulnerabilities related to data processing are identified and remediated).
- PII redaction at ingest → **CT.DM-P1** (data elements can be accessed for review, correction, erasure), **PR.DS-P1** (data at rest is protected).
- DPO consultation record → **GV.PO-P1** (organisational privacy values are established and used), **GV.AT-P1** (workforce is informed).
- DSR runbooks → **CT.DM-P1**, **CT.DM-P2** (data elements can be destroyed), **CM.PO-P1** (transparency policies).
- Breach-response plan → **PR.PO-P9** (response plans), **GV.MT-P6** (policies / processes are reviewed and updated).

The map is not one-to-one — a single engineering artefact usually touches multiple Subcategories, and a single Subcategory is commonly supported by multiple artefacts. The output is a matrix: PF Subcategory rows, engineering-artefact columns, cells filled with the PRAM-Worksheet-4 response rows that bind the two. This matrix is the "NIST-PF-compatible evidence pack" an auditor expects.

For AI-specific additions, overlay the **NIST AI RMF** Functions (Govern, Map, Measure, Manage) — the Privacy Framework and AI RMF were designed to compose. Where a PF Subcategory concerns an AI-specific risk, cross-tag the AI RMF Subcategory. The AI RMF Generative AI Profile (NIST AI 600-1) adds AI-specific threat categories (CBRN, confabulation, data privacy, environmental, harmful bias, HCI, info integrity, info security, intellectual property, obscene/degrading, toxicity) that map onto the PRAM's problematic-data-action set.

## PRAM for an AI system — a worked skeleton

A compressed example of what the end-to-end PRAM output for an AI-powered customer-support RAG system looks like at the top level, so you can see where each worksheet lands:

```
Worksheet 1 — Framing
  System: Support-RAG v2.1
  Owner: CX Platform / Privacy Eng co-author
  Laws in scope: GDPR (EU users), UK GDPR (UK users), CCPA (CA users), Virginia CDPA, HIPAA (BAA in place with health-industry customers)
  Privacy risk statement: Avoid subject harm from unexpected use of support-ticket content for AI; comply with GDPR, UK GDPR, state laws, HIPAA.

Worksheet 2 — Data actions (15 rows)
  A1 Training-data collection (one row per training corpus)
  A2 Inference-input collection
  ... B1..B6, C1..C5 as above ...

Worksheet 3 — Prioritisation (ranked risk list, ~30 entries)
  #1 Re-identification via embedding nearest-neighbour (B2 + C3)  [High likelihood, High subject impact]
  #2 Training-data memorisation extractable (B1 + C1)              [Moderate likelihood, High subject impact]
  #3 Prompt-log Surveillance from retention beyond need (B4)       [High likelihood, Moderate subject impact]
  ...

Worksheet 4 — Response (one row per entry in W3)
  #1 Mitigate — reduce embedding-metadata granularity; access controls on vector store; monitor near-duplicate retrieval
     Controls: CT.DP-P4, CT.DM-P1, PR.AC-P1 + SP 800-53 AC-3, SI-19
     Artefact: Redaction FP/FN report, access-log review memo, retrieval-monitor dashboard
     Residual: Moderate likelihood, Moderate impact
  ...

NIST PF matrix (Subcategory × Artefact)
  ID.IM-P1 — training-data manifest, RoPA, DFD
  CT.DP-P4 — DP-SGD accountant output, federated-aggregation protocol spec, redaction report
  PR.PO-P9 — breach-response plan, incident-exercise log
  ... [30–60 cells typical]
```

The outputs chain into the DPIA (chapter 02 Step 5 / 6), into the regulator-facing bundle (chapter 06), and into the SP 800-53 PT-family control mapping (chapter 05).

## PRAM discipline the auditor expects

- **The data-action list is exhaustive** — every point where personal data enters, moves, or leaves the system appears somewhere. Missing actions is the most common PRAM failure; "we didn't think of the derivative-product action" is not a defence an auditor will accept.
- **The problematic-data-action taxonomy is applied intellectually, not clerically** — marking every PDA on every action is a sign of lazy work. Marking the PDAs that actually apply, with one-sentence rationale, is the sign of real thought.
- **The scoring is traceable** — an auditor should be able to reverse the likelihood and impact scores from the data, not read them as opinions. "Likelihood: Moderate, because the measured MIA advantage on the held-out audit is 0.04 and our acceptance threshold is 0.05" is traceable; "Likelihood: Moderate" is not.
- **The response is implemented, not promised** — Worksheet 4's "specific controls applied" cites existing artefacts. Controls on the roadmap are "Planned" with a date, not "Applied".
- **The PF matrix is complete for the applicable Subcategories** — not every PF Subcategory applies to every system (a system that does not share data with third parties does not need CT.DP-P2); the matrix explicitly marks each as applicable or not-applicable with justification.
- **The PRAM is versioned** — same discipline as the DPIA. Each refresh shows what changed and why.

## Where the PRAM and the DPIA diverge — and how to run both without duplicate work

The DPIA foregrounds *rights and freedoms of data subjects*; the PRAM foregrounds *problematic data actions* and the *risk to individuals* and *risk to the organisation* as two distinct dimensions. The overlap is substantial:

| DPIA artefact | PRAM equivalent |
|---|---|
| Step 2 systematic description | Worksheet 2 data-action descriptions |
| Step 4 necessity / proportionality | Worksheet 1 framing + Worksheet 4 avoid / accept responses |
| Step 5 risk register | Worksheets 2–3 (actions, problematic actions, prioritisation) |
| Step 6 mitigation table | Worksheet 4 responses + NIST PF matrix |
| Step 7 sign-off | Worksheet 4 owner / due-date + organisational approval record |

The practical pattern for a controller that runs both: **PRAM as the engineer's working document; DPIA as the controller's statutory record**. The engineer builds the PRAM worksheets with the full data-action enumeration and the full problematic-data-action tagging, then populates the DPIA by distilling the PRAM into the ICO or ISO 29134 template. The PRAM worksheet is the DPIA's Appendix 3. An auditor who sees both is reassured by the consistency — if they diverge, you have a problem.

## Summary — what you can do after this chapter

- Decompose an AI system's personal-data flow into data actions (A1–A3 / B1–B6 / C1–C5) at the granularity the PRAM expects.
- Fill Worksheet 1 (framing) with mission relevance, applicable laws, privacy principles, and the explicit privacy risk statement.
- Fill Worksheet 2 (data actions) with per-action descriptions, problematic-data-action tags from the nine-category taxonomy, and the per-row problems-for-individuals and problems-for-organisation.
- Fill Worksheet 3 (prioritisation) with likelihood / subject-impact / organisation-impact scores grounded in evidence.
- Fill Worksheet 4 (response) with concrete controls cited to artefacts, with NIST PF Subcategory and SP 800-53 control tags.
- Produce the NIST Privacy Framework matrix (Subcategory × Artefact) that an auditor reads as the evidence pack.
- Map the PRAM output onto the DPIA (chapter 02) so you run both without duplicating the work.

Chapter 04 turns to the European regulator stance that overlays both — EDPB Opinion 28/2024 on AI models and the CNIL AI how-to sheets — and shows how to translate the four regulator questions into engineering artefacts the DPIA cites.
