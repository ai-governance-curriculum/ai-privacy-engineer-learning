# EDPB Opinion 28/2024 and CNIL AI Sheets — Translating Regulator Positions into DPIA Artefacts

The hardest part of running a DPIA for an AI system under European law is not the statutory structure — Art. 35 is unambiguous about what belongs in a DPIA (chapter 01) — it is the contested interpretation of how GDPR applies to models that are themselves trained on personal data. For most of the GDPR's life, that interpretation sat in national-DPA guidance (CNIL leading, with ICO, Garante, Datatilsynet, and others following) and in ad-hoc enforcement actions (the Italian Garante's ChatGPT provisional ban in 2023 being the most publicised). In December 2024 the European Data Protection Board adopted **Opinion 28/2024** on the processing of personal data in the development and deployment of AI models — the Board's consolidated position, binding on all EEA DPAs in their interpretive practice.

A DPIA for an AI system developed or deployed in the EU/EEA that does not engage Opinion 28/2024 is incomplete. The ICO's view (UK) tracks substantively; the CNIL's AI how-to sheets (published in staged form from 2024 onward) are the detailed engineering-facing implementation of the EDPB's position for French-regulated controllers, and are widely used as a general reference across the EU.

This chapter walks the four questions the EDPB opinion frames and names, per question, the engineering artefact your DPIA must cite.

Primary references — read before relying on this chapter's summary:

- EDPB — *Opinion 28/2024 on certain data protection aspects related to the processing of personal data in the context of AI models*, adopted 17 December 2024. [edpb.europa.eu/our-work-tools/our-documents/opinion-board-art-64/opinion-282024-certain-data-protection-aspects_en](https://www.edpb.europa.eu/our-work-tools/our-documents/opinion-board-art-64/opinion-282024-certain-data-protection-aspects_en).
- CNIL — *AI how-to sheets* ("Fiches pratiques IA"), covering purpose definition, lawful-basis selection, DPIA, security, data-subject rights, and scraping; English translations available. [cnil.fr/en/ai-how-sheets](https://www.cnil.fr/en/ai-how-sheets).
- CNIL — *Public consultation summaries* on AI practice and DPIA for AI. [cnil.fr/en/artificial-intelligence](https://www.cnil.fr/en/artificial-intelligence).
- UK ICO — *Guidance on AI and data protection*. [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/).
- Italian Garante — *Deliberazione 112/2023* (the ChatGPT provisional measure) and subsequent decisions on generative AI. [gpdp.it](https://www.gpdp.it/).

## The four questions the EDPB opinion frames

Opinion 28/2024 is organised around four questions the Irish DPC referred to the EDPB (the Opinion itself is in response to an Art. 64(2) consistency referral). The EDPB's answers are the operative European regulator position:

1. **When and how can an AI model be considered "anonymous"?** — Is the model itself, as a technical artefact, personal data? If so, under what conditions can a controller claim the trained model is anonymous and therefore outside the scope of the GDPR for its deployment?
2. **How can controllers demonstrate the appropriateness of legitimate interests as a lawful basis for processing personal data to develop and deploy AI models?** — The LI balancing test as applied to AI training and inference.
3. **What are the consequences of unlawful processing of personal data during the development phase of an AI model for the subsequent operation of the model?** — Downstream consequences if training was non-compliant.
4. **(Implicit throughout the opinion) — how are data-subject rights exercised against AI models?** — The engineering feasibility of Articles 15–22 against trained models, embeddings, and derivative artefacts.

The fourth question is implicit in the opinion rather than explicitly posed as a referral question, but the EDPB's analysis of Questions 1–3 is pervasive about DSR consequences; this chapter treats it as co-equal because that is how the DPIA must treat it.

The rest of this chapter walks each question in turn and names the engineering artefacts the DPIA must cite for that question.

## Question 1 — When is an AI model "anonymous"?

The EDPB rejects the position that a trained model is categorically anonymous. It also rejects the position that a trained model is categorically personal data. Instead: **an AI model is anonymous if the likelihood of re-identification — of extracting information about identifiable individuals from the model, by any means reasonably likely to be used by the controller or any other person — is insignificant**. This is the Recital 26 standard applied to models.

The test is a case-by-case technical evaluation, with the burden on the controller to demonstrate anonymity. The Opinion names the kinds of evidence a controller must assemble; the engineer's job is to produce it.

### The evidence the DPIA must cite to claim "anonymous model"

To support an "anonymous model" claim, the DPIA assembles:

- **Design choices that reduce memorisation and extraction risk** — documented, not asserted. This is the training-pipeline story: deduplication, PII scrubbing at ingest, bounded per-example gradient norms, differential privacy where applied with (ε, δ) reported, near-duplicate suppression, examples-per-subject caps.
- **An empirical extraction / membership-inference evaluation** — a run of the mod-105 memorisation-and-MIA audit with results reported. The Opinion treats the empirical attack-advantage as the primary signal of re-identification likelihood. A low measured MIA advantage against a white-box attacker is the single strongest piece of evidence.
- **A model-inversion / reconstruction evaluation** — where the model's architecture (embedding model, classifier, generative model) supports model-inversion attacks, the empirical evaluation of reconstruction fidelity.
- **An extraction evaluation against the deployed inference surface** — not only the trained-weights attack, but the attack an adversary runs against the deployed endpoint (Carlini-style extraction with the actual API constraints). The attack-surface the controller actually exposes.
- **A "reasonably likely means" analysis** — the Recital 26 question: what means would an adversary reasonably use, given the model's deployment, the attacker's resources, the cost of the attack, the value of the data? The DPIA documents the threat model and the controller's assessment that the adversary cannot, under reasonably likely means, re-identify.
- **A re-evaluation cadence** — anonymity is not a one-time property; it degrades as attack techniques advance. The DPIA names the cadence at which the anonymity evaluation is re-run (per major model version, at least annually, on publication of a new material attack).

The claim form in the DPIA: not "the model is anonymous", but "the controller has evaluated the model's susceptibility to [attack set]; measured advantage is [number]; the controller's assessment is that re-identification by reasonably likely means is insignificant; this evaluation is scheduled for refresh on [cadence]". That claim can be audited. A bare "the model is anonymous" cannot.

### The consequence if anonymity cannot be claimed

If the DPIA cannot demonstrate model anonymity, the model is personal data for the purposes of its deployment, and the deployment phase is itself personal-data processing with its own lawful-basis requirement. In practice: the deployment of a non-anonymous model is a Chapter II / Chapter III GDPR operation, with the subject rights (Art. 15–22) exercisable *against the deployed model* and not only against the training corpus.

Most production AI systems today cannot credibly claim anonymity. The DPIA posture most controllers end up with is: deployed model is personal data, Art. 6 / 9 lawful basis is documented for the deployment, Art. 15 obligation is engineered via the inference-log and logic-description artefacts (chapter 02), Art. 16 is engineered via rectification-at-retrain, Art. 17 is engineered via the chapter-05-of-mod-110 erasure decision tree. The anonymity claim is the harder path; most engineering programmes default to the "model is personal data, we are engineering the obligations" posture.

## Question 2 — The legitimate-interests balancing test for AI training and deployment

Opinion 28/2024's longest section is on Art. 6(1)(f) legitimate interests as the lawful basis for AI model development and deployment. For many AI programmes — especially those trained on scraped or large-scale web data — LI is the only plausible lawful basis; the EDPB's framing is correspondingly rigorous.

The Opinion adopts the three-step LI test the Court of Justice established in C-13/16 (Rīgas satiksme) and C-252/21 (Meta Platforms):

1. **Purpose test** — is the pursued interest legitimate (lawful, specific, real, present)?
2. **Necessity test** — is the processing necessary to achieve that interest, or can the interest be served by a less-intrusive means?
3. **Balancing test** — does the controller's interest override the interests, rights, and freedoms of the data subjects?

The Opinion applies each step to AI-specific facts — these are the engineering obligations.

### Purpose test — the interest must be specific to the AI system

"Improving AI capabilities" is not a legitimate interest in the Art. 6(1)(f) sense; it is a slogan. The DPIA must name:

- The specific interest — "fraud detection for e-commerce transactions" or "recommendation of products to shoppers based on browsing history".
- Whose interest — the controller's; third parties' (processors, business partners, society at large).
- The legal / ethical pedigree of the interest — no illegality, not hidden, within the controller's lawful activities.

The engineering input: the system-definition section of the system card, the purpose column of the RoPA, the business-case document (where relevant).

### Necessity test — alternatives considered

The Opinion is explicit that controllers must consider less-intrusive alternatives. For an AI system, the alternatives the DPIA must engage:

- **No personal data at all** — can the purpose be served by non-personal, aggregate, or synthetic data? If so, why was personal data used?
- **Less personal data** — per-category necessity (chapter 01 (b)).
- **Pseudonymisation or anonymisation at ingest** — is the data pseudonymised before training? If not, why not? The CNIL's AI sheet on scraping is particularly tight here; any training-data corpus sourced by scraping should default to PII scrubbing at ingest and should document deviations.
- **Privacy-enhancing training** — DP-SGD, federated learning, secure aggregation. The DPIA names which PETs were considered and the engineering reason each was adopted or rejected (with the utility / cost / policy evidence).
- **Smaller model / fewer parameters** — does the purpose justify the memorisation-capacity of a large model, or could a smaller model with lower risk of memorisation suffice?

The DPIA that records no alternatives considered is presumptively failing the necessity test. The DPIA that records alternatives rejected with measured utility / cost trade-offs — "DP-SGD at ε = 3 reduced top-1 accuracy from 86% to 79%, below the operational threshold of 82%; chosen alternative was aggressive deduplication plus output filtering" — is passing it.

### Balancing test — the DPIA's hardest section

The balancing test weighs the controller's interest against the rights, freedoms, and legitimate expectations of the data subjects. The Opinion lists factors the balancing must address — this list is the DPIA's checklist:

- **The nature of the data** — special-category data under Art. 9 is a near-absolute bar on LI; personal data of vulnerable populations (children, patients, workers) tips the balance against the controller.
- **The scale of the processing** — larger scale shifts the balance toward the subjects.
- **The source of the data** — data scraped from the web raises reasonable-expectations concerns; data collected directly from the subject under a transparent interface raises fewer.
- **The relationship between controller and subject** — pre-existing relationship (customer, employee) shifts expectations; no prior relationship (scraped third-party data) tips against the controller.
- **The reasonable expectations of subjects** — would a reasonable subject expect their data to be used in this way? For scraped data this is particularly acute.
- **The reversibility of the processing** — once in a trained model, data is hard to remove. This is a balancing factor against the controller unless robust unlearning / retraining machinery is in place.
- **The safeguards applied** — the PETs, the minimisation, the DSR runbooks, the retention limits, the transparency measures. This is where most of the engineer's bundle lands as balancing-positive evidence.
- **The potential for harm** — the feared-event register from chapter 01. This is where the risk assessment and the balancing test converge.

The balancing-test section of the DPIA cites each factor and names the artefact that evidences the controller's position. A DPIA that writes "the controller's interests outweigh the subjects' because the processing is low-risk and well-safeguarded" fails the test as prose; the same conclusion grounded in the artefact chain passes.

### The AI-specific LI question — pre-training vs. inference

The Opinion distinguishes *development phase* (training, including pre-training on large corpora) from *deployment phase* (inference, use of the deployed model). The LI test may have different outcomes for each — training on scraped data may fail LI even where deployment to paying customers passes. The DPIA must run the LI test *per phase*.

The engineering consequence: per-phase lawful-basis documentation. The training manifest for each corpus records the corpus's lawful basis; the inference pipeline records the lawful basis under which it processes inputs and produces outputs. The DPIA aggregates these into a single lawful-basis section with per-phase subsections.

## Question 3 — Consequences of unlawful development

The Opinion's answer: unlawful processing during development does not automatically render deployment unlawful, but the deployment controller must examine whether the deployment relies on the unlawful processing in a way that propagates the unlawfulness. The practical test is whether the deployment would be possible without the unlawfully processed data — a controller deploying a third-party model trained on unlawfully processed data is not automatically liable, but may be if it had reason to know of the unlawfulness or relies specifically on the unlawful elements.

The DPIA consequences for a controller deploying a third-party foundation model:

- **Training-data provenance due diligence** — what is documented about the training data of the foundation model? Public sources, licence terms, consent, scraped-data risk.
- **Representations and warranties** — does the vendor warrant training-data lawfulness? Attach the contractual language.
- **Residual exposure** — if the foundation model's training were later ruled unlawful, what is the controller's exposure? The DPIA names the dependency and the controller's position.

For a controller developing and deploying its own model:

- **Each training-data source is lawful** — per-corpus lawful-basis documentation.
- **Unlawful sources identified during due diligence are excluded** — if a corpus was obtained under terms the controller determines are non-compliant, the corpus is excluded from training. The DPIA records the exclusion.
- **Discovered unlawfulness triggers a retraining decision** — if a corpus is later found to be non-compliant, the controller must decide whether to retrain without it, apply unlearning (mod-110 ch. 02–04), or discontinue the model. The DPIA documents the decision pathway.

## Question 4 — Data-subject rights against AI models (implicit but pervasive)

The Opinion treats DSR exercise as a pervasive concern throughout its analysis of Questions 1–3. The CNIL AI sheets make the DSR obligations explicit. The engineering artefacts the DPIA must cite — all from mod-110 — are:

- **Art. 15 (access)** — the inference-log schema, the logic-description document, the Art. 15(1)(h) statement for automated decisions.
- **Art. 16 (rectification)** — the correction-propagation pipeline, the retraining cadence that absorbs corrections.
- **Art. 17 (erasure)** — the erasure-vs-retraining decision tree (mod-110 ch. 05), the SISA / unlearning capability statement, the retrieval-corpus and log-purge runbooks.
- **Art. 18 (restriction)** — the restriction-flag wiring and the per-store soft-delete runbooks.
- **Art. 20 (portability)** — the subject-provided-data export pipeline.
- **Art. 21 (objection)** — the per-purpose exclusion wiring.
- **Art. 22 (automated decision-making)** — the human-review endpoint, the decision-record schema, the logic description.

Each entry in the DPIA cites the mod-110 runbook that actually implements it. The common failure mode: a DPIA that lists all the rights in a table and cites a generic "DSR portal" URL. The CNIL sheet on data-subject rights for AI specifically rejects this as inadequate; the DPIA must show *how* each right is engineered.

## The CNIL AI sheets as the engineering-facing complement

The CNIL has published a sequence of AI how-to sheets, the key ones for DPIA-authors:

- **Define a purpose for AI** — purpose specification for AI training and deployment.
- **Determine the lawful basis** — selection of Art. 6(1) basis per phase, with specific guidance on consent, contract, LI, and public-interest.
- **Pay attention to the legal qualification of AI actors** — controller / processor / joint-controller determination for AI supply chains.
- **Define the DPIA scope and content** — CNIL's operational interpretation of Art. 35 for AI.
- **Address the data-minimisation challenge** — practical minimisation in training-corpus construction.
- **Define a retention period** — retention for training corpora, model weights, logs, derivative artefacts.
- **Take into account the choices related to annotation** — risk and compliance for human annotation / RLHF.
- **Ensure the security of the development** — Art. 32 security controls specific to AI training.
- **Scraping as an AI-training-data practice** — the sheet most relevant to foundation-model controllers; detailed, prescriptive, and (if your controller scrapes) the single most important CNIL sheet to read.
- **Data-subject rights** — operational engineering of Art. 15–22 against AI models.

For each sheet, the DPIA cites the controller's position and the artefact that evidences it. The CNIL sheets are not binding on non-French controllers, but they are widely treated as the engineering-practical reading of Opinion 28/2024 across the EU. An auditor reading a DPIA that cites CNIL sheets directly is reassured by the alignment.

## The ICO's position — substantively aligned, differently structured

The UK ICO issued *Guidance on AI and data protection* (updated through 2024) that covers substantially the same territory. Alignment with Opinion 28/2024 is close but not identical — the ICO's guidance predates the EDPB opinion in parts and may be clarified further in subsequent updates. The practical pattern for a UK-regulated controller:

- Treat the ICO guidance as primary.
- Cross-reference Opinion 28/2024 for substantive consistency; where the two diverge, note the divergence and the controller's reasoning.
- Watch for ICO updates following any EDPB consistency decision.

For a dual-jurisdiction controller (UK + EEA), the DPIA cites both.

## Translating the four questions into DPIA artefacts — a checklist

A DPIA for an AI system under European law should contain, in its appendices or inline, the following artefacts — each keyed to the EDPB / CNIL / ICO expectation:

| Question / topic | Artefact the DPIA cites | Engineering source |
|---|---|---|
| Model anonymity | Memorisation / MIA audit report with measured advantages; anonymity re-evaluation cadence memo | mod-105 chapters |
| LI purpose test | Purpose-specification document per phase | System card, RoPA |
| LI necessity test | Alternatives-considered memo, PET evaluation matrix | PRAM Worksheet 1 + 4 |
| LI balancing test | Factor-by-factor balancing memo, feared-event register | Chapter 01 risk register, PRAM Worksheet 3 |
| Training-data provenance | Per-corpus provenance chain (source, licence, consent / LI basis, PII scrubbing applied) | Data-governance inventory from mod-103 |
| Scraped-data posture | Scraping-sheet-compliant practice note; robots.txt respect; subject-of-content protections | Ingest pipeline documentation |
| Retention per phase | Retention matrix per store | mod-103 inventory |
| Annotation / RLHF | Annotator consent, data-handling, agreement with annotator vendors | Annotation-pipeline documentation |
| DSR per right | Per-right runbook pointer, DSR audit-trail schema | mod-110 chapters |
| Art. 22 decision | Human-review endpoint spec, decision-record schema, logic description | System card ch. 06 of mod-109 |
| Vendor / processor diligence | Vendor assessment memo, DPA, model-provenance documentation | Procurement artefacts |

A DPIA whose appendix index matches this table at a high level will be defensible against European regulator review. A DPIA missing rows will be asked for them.

## Summary — what you can do after this chapter

- Read Opinion 28/2024 as a structured set of four questions each demanding engineering evidence.
- Assemble and cite the anonymity-claim evidence (MIA / extraction audit, design choices, re-evaluation cadence) or, where anonymity cannot be claimed, engineer the Art. 15–22 obligations against the model.
- Run the three-step LI test with the engineering content — purpose specific to the system, necessity grounded in alternatives considered, balancing grounded in the feared-event register and the safeguards.
- Address training-development-phase lawfulness and its downstream consequences in the DPIA, with per-corpus provenance.
- Point every DSR right in the DPIA at its implementing runbook from mod-110.
- Cross-reference CNIL AI sheets as the engineering-practical complement, and ICO guidance for UK alignment, and document divergences.

Chapter 05 shifts from European to sectoral — HIPAA where PHI is in scope, GLBA where financial data is in scope, and the NIST SP 800-53 PT-family controls for federal and federally-adjacent systems.
