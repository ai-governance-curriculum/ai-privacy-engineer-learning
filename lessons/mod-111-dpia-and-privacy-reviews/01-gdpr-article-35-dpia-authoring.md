# Authoring an Audit-Grade GDPR Article 35 DPIA for an AI System

A Data Protection Impact Assessment is not a compliance deliverable the privacy engineer writes at the end of a project to satisfy the legal team. It is a *risk instrument* the GDPR requires *before* processing starts (Art. 35(1)) when that processing is likely to result in a high risk to the rights and freedoms of natural persons, and AI systems — especially those that profile, score, or operate at scale against personal data — are the paradigmatic Art. 35 trigger. The DPIA is the single document a supervisory authority will ask for first when they inspect an AI deployment; everything else in the regulator-facing bundle (chapter 06) exists to support claims the DPIA makes.

The engineer's job is to produce a DPIA that *reads as true to a lawyer and reproducible to an auditor*. That is harder than producing a DPIA that reads well. A DPIA full of prose and empty of evidence will be rejected — either by the DPO before it is filed, by the supervisory authority on inspection, or by a court once an incident makes its contents material. This chapter walks the six statutory sections of an Art. 35 DPIA and names, for each, the engineering artefact your DPIA must cite.

The authoritative texts you work from — read them before starting a DPIA, not after:

- Regulation (EU) 2016/679, *General Data Protection Regulation*, **Article 35** (DPIA), **Article 36** (prior consultation), **Article 37–39** (DPO), **Article 30** (RoPA), **Article 33–34** (breach notification), **Recitals 75, 84, 89–96**. [eur-lex.europa.eu/eli/reg/2016/679/oj](https://eur-lex.europa.eu/eli/reg/2016/679/oj).
- Article 29 Working Party — *Guidelines on Data Protection Impact Assessment (DPIA) and determining whether processing is "likely to result in a high risk"*, WP248 rev.01, endorsed by the EDPB (2017). [ec.europa.eu/newsroom/article29/items/611236](https://ec.europa.eu/newsroom/article29/items/611236).
- UK ICO — *Data protection impact assessments*. [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/accountability-and-governance/data-protection-impact-assessments-dpias/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/accountability-and-governance/data-protection-impact-assessments-dpias/).
- UK ICO — *Guidance on AI and data protection* (DPIA section). [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/).
- ISO/IEC 29134:2023 — *Information technology — Security techniques — Guidelines for privacy impact assessment*. [iso.org/standard/86012.html](https://www.iso.org/standard/86012.html).
- CNIL — *Privacy Impact Assessment (PIA) methodology and knowledge bases*. [cnil.fr/en/privacy-impact-assessment-pia](https://www.cnil.fr/en/privacy-impact-assessment-pia).

## When Article 35 is triggered, and when prior consultation under Article 36 follows

Article 35(1) requires a DPIA *where processing, in particular using new technologies, is likely to result in a high risk to the rights and freedoms of natural persons*. Article 35(3) names three cases where a DPIA is always required — (a) systematic and extensive automated evaluation, (b) processing on a large scale of special-category or criminal-conviction data, (c) systematic monitoring of a publicly accessible area. Article 35(4) directs each supervisory authority to publish and maintain a national list of operations that require a DPIA.

For AI systems, the practical rule: **if the system profiles, scores, ranks, classifies, or makes recommendations about natural persons, a DPIA is almost certainly required**. The WP248 nine-criteria test (evaluation or scoring; automated decision with legal or similarly significant effect; systematic monitoring; sensitive data; data on a large scale; matching or combining datasets; vulnerable data subjects; innovative use of technology; processing that prevents the exercise of a right or use of a service) is the triage checklist — two or more hits almost always means DPIA is mandatory. AI systems routinely hit three or four.

Article 36(1) adds a further step: **if the DPIA concludes the processing would result in a high risk *in the absence of measures taken to mitigate the risk*, the controller must consult the supervisory authority *before* processing begins**. This is "prior consultation" — a formal engagement with the DPA that can block deployment or require conditions. The engineering reading: the DPIA is not only an accountability record, it is a *gatekeeper*. If your mitigations do not credibly bring the residual risk below "high", you are legally required to tell the supervisory authority before you ship.

The chapters that follow in this module exist to make that gating decision defensible. Chapter 02 maps the statutory sections onto the ICO template; chapter 03 translates the risk assessment into NIST PRAM worksheets; chapter 04 brings the EDPB/CNIL AI-specific overlays; chapters 05–06 package the sectoral and bundle evidence.

## The six required contents of an Article 35 DPIA

Article 35(7) sets a minimum statutory floor for what a DPIA must contain:

> *The assessment shall contain at least:*
>
> *(a) a systematic description of the envisaged processing operations and the purposes of the processing, including, where applicable, the legitimate interest pursued by the controller;*
>
> *(b) an assessment of the necessity and proportionality of the processing operations in relation to the purposes;*
>
> *(c) an assessment of the risks to the rights and freedoms of data subjects referred to in paragraph 1; and*
>
> *(d) the measures envisaged to address the risks, including safeguards, security measures and mechanisms to ensure the protection of personal data and to demonstrate compliance with this Regulation taking into account the rights and legitimate interests of data subjects and other persons concerned.*

Add two obligations that fall out elsewhere in the regulation but belong in the DPIA record as the audit expects to find them there:

> *(e) a residual-risk statement — what risk remains after the measures in (d) are applied, and whether that residual risk triggers Article 36 prior consultation; and*
>
> *(f) a DPO consultation record — Article 35(2) requires the controller to seek the DPO's advice; Article 39(1)(c) makes DPIA advice an explicit DPO duty. The DPIA must record that advice was sought, the DPO's opinion, and the controller's response.*

A "DPIA" that omits any of (a)–(f) is not an Art. 35 DPIA; it is a privacy memo. This chapter walks each in turn, names the engineering artefact that populates it, and names the common failure modes an auditor will look for.

## (a) Systematic description of the processing — what the AI system actually does

The systematic description is the longest section and the one engineers most often under-write. The DPIA reader (DPO, auditor, DPA inspector) does not know your system. They need enough concrete detail to reconstruct the processing from the DPIA alone. Vague prose — "the system uses machine learning to provide personalised recommendations" — guarantees a round of follow-up questions and signals an immature programme.

The minimum content you must land, with the engineering artefact that supplies each:

- **The processing operations** — what personal data is collected, from whom, under what lawful basis, for what purposes, how it flows through the system, where it is stored, how long it is retained, and who it is shared with. Cite: the **Record of Processing Activity (RoPA)** entry (Art. 30) for the system, and the **data-flow diagram** (DFD) that accompanies it. The DFD should show every store from mod-110 chapter 01's inventory discipline — operational DB, training corpus, model weights, retrieval corpus, prompt / completion logs, derivative products — with the subject-identifier shape and the lawful basis annotated at each ingress edge.
- **The purposes** — not "to improve the service", but a per-purpose enumeration. A fraud-detection system and a product-recommendation system that share an embedding model are two purposes, not one. The DPIA must name each purpose and bind each processing operation to one. This is the hook the right-to-object runbook (mod-110 ch. 01) uses.
- **The legitimate interest, where that is the lawful basis** — a documented Legitimate Interests Assessment (LIA) with purpose / necessity / balancing-test. The LIA is a DPIA appendix, not a hand-wave; chapter 04 of this module covers the AI-specific balancing test the EDPB Opinion 28/2024 three-step framework requires.
- **The system architecture** — the model family, the training data provenance, the training objective, the deployment surface, the inference path, the human-in-the-loop points. The DPIA cites the **model card** and **system card** (mod-109 ch. 06) as its source; the DPIA itself names only what a non-engineer DPO must grasp to understand the risk.
- **The scale** — number of data subjects, volume of records, geographic scope, duration of processing, frequency of inference. Not an estimate — a measured count or a documented upper bound. The scale drives the Art. 35(3)(b) "large scale" trigger and the risk-severity estimate.
- **The data categories** — explicit enumeration of personal data processed, with special-category (Art. 9) and criminal-conviction (Art. 10) data flagged. Derived categories count — if the model infers a special-category attribute from non-special-category inputs, the inference is special-category data in the EDPB's reading and must be flagged here.
- **The automated decision-making footprint** — where, if anywhere, the system produces decisions that fall under Art. 22 (solely automated, legal or similarly significant effect). This flag gates additional obligations: Art. 22(3) safeguards (right to human intervention, right to express a view, right to contest), and the Art. 15(1)(h) logic-description duty.

The common failure mode in (a) is **describing the system you want to have, not the system you have**. An auditor will cross-check your systematic description against your repository state, your logs, and your runtime traces. If the DPIA says "retention is 30 days" and the S3 bucket has a 2-year lifecycle rule, the DPIA is materially misleading. Build the description from the RoPA and the DFD, not from the product spec.

### Worked example — the systematic description skeleton

A skeleton for (a) that an auditor will accept as complete:

```
1. System identity
   - Name, owner, controller legal entity, processor legal entity (if distinct)
   - Deployment surface (prod regions), go-live date (planned or actual)
2. Processing operations and purposes (one row per purpose)
   | Purpose | Lawful basis | Art. 22? | Art. 9 data? | Scale (subjects / month) |
   |---------|--------------|----------|--------------|---------------------------|
   | ...     | ...          | ...      | ...          | ...                       |
3. Data flow — cite DFD <path>, show:
   - Collection points, with subject-identifier shape
   - Stores (six-corpus map), with retention
   - Inference path (sync / async / batch)
   - Egress (sharing, transfer, derivative products)
4. Data categories table (one row per category)
   | Category | Source | Special-category (Art. 9)? | Derivation chain |
5. Automated decision-making — one block per Art. 22 decision
   - Decision, significance of effect, human-in-the-loop description
6. Scale — number of subjects, records, inferences per month; geographic scope
7. Data subject demographics — including vulnerable-subject flag per WP248 criterion 7
8. Third parties — processors, sub-processors, cross-border transfers (Art. 44–49)
```

Each row and block cites an artefact under version control. "See DFD `docs/dfd-recsys-v2.3.svg`" is a citation; "the system processes user data" is not.

## (b) Necessity and proportionality — the test that most DPIAs fail

Necessity and proportionality are two distinct tests, not a single slogan. The WP248 guidelines are explicit:

- **Necessity**: is the processing *necessary* for the specified purpose, or can the purpose be achieved by a less-intrusive means? If a less-intrusive means exists, the processing as designed is unnecessary.
- **Proportionality**: even if necessary, is the processing *proportionate* — does the benefit justify the intrusion? This is a balancing test, not a yes/no.

For an AI system the necessity test has a specific engineering flavour. The DPIA must address:

- **Is processing personal data necessary at all, or could the purpose be served by non-personal / aggregate / synthetic data?** If the answer is yes to the alternative, you must justify why you still process personal data. This is where mod-103 (minimisation) and mod-106 (differential privacy) cash out in the DPIA — if DP-trained or federated alternatives exist and were not chosen, the DPIA must say why.
- **Is each data category necessary?** Minimisation is a GDPR principle (Art. 5(1)(c)) with Art. 35 teeth. The DPIA must per-category justify collection. "We collect IP addresses for troubleshooting" is OK if you can show you need them; it is not OK if you use them for anything beyond that.
- **Is the model's complexity proportionate?** A black-box LLM-based system where a linear model with ten features would suffice is a proportionality concern. The DPIA does not have to pick the simplest model, but it must address the trade-off.
- **Is the retention duration necessary?** Each retention window needs a purpose. "We retain prompt logs for one year for debugging" — justify the year. If the debugging value decays after a week, the eleven-month tail is disproportionate.

The proportionality test for an AI system brings in the *risk-benefit framing* the AI Act makes central. The DPIA must engage the question: what is the benefit, to whom, and does it outweigh the intrusion on the data subject? Benefits accruing only to the controller against intrusions borne by the subject tend to fail proportionality.

The common failure mode in (b) is **asserting necessity and proportionality rather than reasoning them**. An auditor will accept "we considered a DP-trained alternative; the utility loss at ε = 3 was 8 percentage points, which made the model non-viable for the fraud-detection use-case; we chose the current architecture and instead invested in output filtering and MIA audit." They will not accept "the current architecture is necessary and proportionate." Engineer-written reasoning, measured trade-offs, and cited evidence are what the test demands.

## (c) Risk to the rights and freedoms of data subjects — a risk assessment, not a security assessment

Article 35(7)(c) asks for "an assessment of the risks to the rights and freedoms of the data subjects referred to in paragraph 1." This is not a security risk assessment. The CIA triad (confidentiality, integrity, availability) is table stakes — it sits in the Art. 32 security assessment that the DPIA cites, not inside the DPIA itself. The Art. 35(7)(c) assessment is about **harms to the subject**: reputational harm, financial harm, discrimination, denial of service, loss of autonomy, chilling effects on fundamental rights.

The CNIL methodology and ISO/IEC 29134 both operationalise this as:

1. **Identify risk sources** — who or what could cause the harm (an insider, an attacker, the controller itself through a design choice, a processor, a third-party sharing partner).
2. **Identify feared events** — the harm as the subject experiences it. Examples: unauthorised disclosure of a user's medical condition; incorrect classification leading to loan denial; model inversion leading to training-data extraction; profile drift leading to exclusion from a service the subject previously accessed.
3. **Identify threats** — the technical or operational path by which a risk source causes a feared event. Examples: a membership-inference attack against the model; a backend compromise of the prompt-log bucket; an incorrect feature engineering choice that proxies a protected attribute.
4. **Assess severity** — how bad the harm is to the subject. The CNIL / ISO 29134 four-point scale is *Negligible / Limited / Significant / Maximum*, and the scale calibration is explicit: Maximum ≈ irreversible, catastrophic (loss of life, long-term serious health impact, financial ruin, unrecoverable identity theft). Severity is a property of the harm, not the probability.
5. **Assess likelihood** — how likely the threat is to materialise. Same four-point scale. Likelihood accounts for the attacker model, the mitigations already in place (Art. 32 security), and the baseline rate of the threat in comparable systems.
6. **Classify** — the (severity, likelihood) tuple lands in a risk matrix; the top-right quadrant is "high risk" and triggers Art. 36 if the mitigations in (d) don't move the dot.

For AI systems, the risk register the DPIA produces must enumerate at least the following AI-specific feared events — this list is the minimum, not the ceiling:

- **Training-data extraction / memorisation** — the model reproduces identifying training data verbatim or near-verbatim; the subject learns their data was in the training set or a third party obtains it via extraction. Severity depends on the data; likelihood is estimated from the mod-105 memorisation and MIA audit results.
- **Membership inference** — an adversary determines whether a subject's record was in the training set. For sensitive-category training data (medical, financial, political) a successful membership inference is itself the harm.
- **Model inversion / attribute inference** — the adversary reconstructs attributes or inputs from the model. Relevant for feature-store-driven models; relevant for LLMs with long-prompt contexts.
- **Inferred-attribute disclosure** — the model's output reveals a special-category attribute the controller never explicitly stored (an "AI reconstruction" of health, orientation, political view from behavioural signals). This is the EDPB's "derived personal data is personal data" position in practice.
- **Automated decision harms** — Art. 22 decisions that systematically disadvantage protected subpopulations, or that are unexplainable to the subject. Discrimination is a feared event.
- **Downstream profiling / linkage** — the system's outputs, combined with other data by a third party, enable re-identification or profiling the subject did not consent to.
- **Prompt-injection / indirect-prompt-injection data exfiltration** — an attacker crafts input that causes the model to emit data from the retrieval corpus or the system prompt. The exfiltration itself is a feared event for the subject whose data is leaked.
- **Loss of DSR exercise** — the subject tries to exercise a right (access, erasure, object) and the engineering system cannot answer in time or at all. This is a feared event the regulator will treat with weight.
- **Quality / drift harm** — the model's output quality degrades silently and the subject bears the cost (wrong medical triage, wrong credit decision, wrong content filter).
- **Breach of confidentiality in operation** — prompts, inputs, or outputs containing personal data leak via logs, vendor sharing, or infrastructure compromise.

Each feared event in the DPIA must have: a one-paragraph description in subject-facing terms, the affected rights (privacy, non-discrimination, data protection, autonomy, freedom of expression, access to justice), the severity, the likelihood, the threat path, and the mitigation plan (which flows into (d)). This is a table — not prose — and chapter 03 shows how the NIST PRAM worksheets turn the same table into a NIST-PF-compatible format.

The common failure mode in (c) is **describing security risks instead of subject risks**. An auditor who reads "a compromise of the S3 bucket could lead to unauthorised disclosure of personal data" and does not read "a subject whose medical prompt leaks loses control of their medical history, a severity-Maximum harm" will downgrade your DPIA. The two paragraphs together are what satisfies Art. 35(7)(c).

## (d) Measures envisaged to address the risks — the mitigation column

The mitigations column is where the rest of your module's work lands in the DPIA. For each feared event in (c), the DPIA must name the measures that reduce severity or likelihood, cite the artefact that evidences the measure, and estimate the residual risk after the measure is applied. The measures fall into four categories you should name explicitly:

- **Technical and organisational measures (TOMs)** — the Art. 32 security controls (encryption at rest / in transit, access control, logging, SDLC, pentest, incident response). These are table stakes and are cited from the Art. 32 security assessment, not re-derived in the DPIA.
- **Privacy-enhancing technologies (PETs)** — DP training (mod-106), federated learning (mod-107), confidential computing and TEE attestation (mod-108), K-anonymity for release, redaction for ingestion and output (mod-104), unlearning capability for erasure (mod-110). These are the privacy engineer's primary contribution. Each PET that is deployed must appear in the DPIA with its parameters (ε, δ, K, attestation-report hash) and the artefact (training-time accountant output, attestation certificate, redaction FP/FN report) that evidences the parameter.
- **Governance and process controls** — the DPIA itself, the model card and system card, the DSR runbooks (mod-110), the breach-response plan (this module ch. 06), the DPO consultation, the AI Act Art. 9 risk management system (where applicable). These are documented in the governance programme and referenced from the DPIA.
- **Data-subject-facing measures** — transparency (privacy notice content, logic description under Art. 15(1)(h)), consent mechanisms where applicable, opt-outs for profiling, subject-request portal, response SLAs. These are the mod-112 territory.

For each mitigation, the DPIA has four fields:

- **What** — the measure, concrete enough that an auditor can verify it (not "we use differential privacy" but "the recommendation model is trained with DP-SGD at ε = 3.0, δ = 1e-5, per-example clipping C = 1.0, noise multiplier σ computed from the Opacus accountant; see training-run manifest `runs/recsys-v4/dp-accountant.json`").
- **Which risk(s) it mitigates** — the explicit mapping from this measure to one or more entries in the (c) register.
- **Severity / likelihood change** — the delta on the risk matrix from applying the measure. This is the engineer's judgement call, grounded in the PET's documented guarantee and the empirical evaluation (e.g., MIA advantage reduction measured under mod-105's tests).
- **Evidence artefact** — the artefact the DPIA cites. Chapter 06 lists the full artefact set the regulator-facing bundle collects.

The common failure mode in (d) is **citing measures that are not instrumented**. A DPIA that says "we apply differential privacy" without the ε / δ parameters and the accountant output is weaker than a DPIA that says "we chose not to apply differential privacy because the empirical MIA audit at the chosen hyperparameters showed a measured attack advantage of 0.03, below our documented acceptance threshold." The former is a claim; the latter is evidence. The auditor will prefer the latter even though the mitigation is nominally weaker.

## (e) Residual risk — the Article 36 gate

After (d), what risk remains? The residual-risk statement is one or two paragraphs per feared event that says, in the four-point severity × four-point likelihood frame, where the risk now sits. The composite classification (across the register) is the DPIA's overall residual-risk determination:

- **Low to moderate residual risk** — the DPIA stands as the accountability record. No prior consultation.
- **High residual risk** — Art. 36 is triggered. The controller consults the supervisory authority *before* processing begins. The DPIA is attached to the consultation.

The language matters. "High residual risk" is a statutory term; using it commits the controller to Art. 36. The DPIA must either *justify why residual risk is not high* (by explicit reference to the mitigations) or acknowledge that it is and schedule the consultation. The DPO sign-off (next section) is largely a review of whether this determination was made honestly.

The engineering input to (e) is the PET evaluation and the attack evaluation. If the mod-105 memorisation audit shows measurable extraction advantage against the training data, the "memorisation" entry in the register cannot be residual-risk-low. The DPIA is the place that honesty is forced.

## (f) DPO consultation record — the sign-off the audit looks for

Article 35(2) requires the controller to *seek the advice of the data protection officer where designated when carrying out a DPIA*. Article 39(1)(c) makes DPIA advice an explicit duty of the DPO. The ICO guidance is clear that the DPO's advice must be recorded in the DPIA itself, with the controller's response if the advice is not followed.

The record the DPIA must contain:

- Date the DPO was consulted.
- The DPO's opinion on necessity, proportionality, the risk assessment, the mitigations, and the residual-risk determination.
- Any disagreement and the controller's decision in response.
- The DPO's view on whether Art. 36 prior consultation is required.
- The DPO's view on whether the DPIA should be updated on a specific trigger (e.g., model retraining with new data categories, a change in deployment scale).

The common failure mode in (f) is **treating DPO sign-off as a rubber stamp**. An auditor who sees a DPIA signed by the DPO on the same day the controller submitted it, with no recorded dialogue, will not trust the review. The record should show that the DPO engaged — asked for additional evidence, disagreed with an initial mitigation claim, required a stronger measure. If your DPO is not pushing back on any of your DPIAs, your DPIA or your DPO (or both) need attention.

## Mapping Article 35(7) to ISO/IEC 29134 and to the ICO template

Three structural standards cover roughly the same ground, with some differences that matter for the DPIA's audit:

| Art. 35(7) element | ISO/IEC 29134 clause | ICO DPIA template section |
|---|---|---|
| (a) systematic description | 6.3.1 (describe the information system), 6.3.2 (describe the processing) | §1 Describe the processing |
| (b) necessity / proportionality | 6.4 (identify relevant privacy safeguarding requirements) | §2 Describe the processing — necessity and proportionality |
| (c) risk assessment | 6.5 (assess privacy risks) | §3 Identify and assess risks |
| (d) mitigations | 6.6 (prepare for treating privacy risks) | §4 Identify measures to reduce risk |
| (e) residual risk | 6.6.5 (determine residual risks) | §5 Sign off and record outcomes |
| (f) DPO consultation | 6.7 (document and report), with DPO role noted | §5 Sign off and record outcomes (DPO advice) |

A privacy engineer writing one DPIA that satisfies all three is standard practice. Chapter 02 walks the ICO template as the concrete shape, because the ICO's template is openly published, concrete, and widely adopted beyond the UK as a working skeleton.

## The DPIA as a living document

A DPIA is not a one-time artefact. Art. 35(11) requires *the controller shall carry out a review to assess if processing is performed in accordance with the data protection impact assessment at least when there is a change of the risk represented by processing operations*. The engineering triggers that require a DPIA refresh:

- Training-data additions that bring new personal-data categories or new sources.
- Model-architecture changes that materially change the risk profile (e.g., switching from a classifier to a generative model, scaling up parameters, switching inference modality).
- Deployment-scope changes (new region, new user segment, new purpose).
- A new feared event emerging from a near-miss, an incident, or an external vulnerability disclosure.
- A new regulatory position (EDPB opinion, national DPA guidance) that bears on the system.
- Any Art. 33 breach involving the system.

The DPIA's version history is itself an audit artefact — the supervisory authority will ask when it was last updated and what changed. A production AI system whose DPIA has not changed in two years while the model has been retrained quarterly will not pass review.

## The hand-off to chapters 02–08

This chapter named the statutory floor. The rest of the module makes each section operational:

- Chapter 02 walks the ICO DPIA template section by section and shows how to populate each from the artefacts you have.
- Chapter 03 operates the NIST PRAM worksheets to produce a NIST-PF-compatible evidence pack that overlays the Art. 35(7)(c) risk assessment.
- Chapter 04 translates EDPB Opinion 28/2024 and the CNIL AI how-to sheets into the four engineering sub-sections the DPIA cites (anonymity-of-the-model claim, lawful-basis documentation, DSR-exercise feasibility, legitimate-interests balancing).
- Chapter 05 adds the sectoral overlays — HIPAA-shape privacy / security / breach-notification rules where PHI is in scope, GLBA Safeguards where financial data is in scope, SP 800-53 PT-family controls for federal systems.
- Chapter 06 packages the regulator-facing bundle: DPIA, DFD, PET deployment claims with parameters, MIA / extraction report, redaction FP / FN report, unlearning-capability statement, logging / retention policy, breach-response plan.
- Chapter 07 cross-references the Cloud Security Alliance AI Controls Matrix and ISO/IEC 42001 AIMS clauses that show up in enterprise audits.
- Chapter 08 names the boundary to `ai-evaluation-engineer` — the peer / next-up role at level 35 that owns release-assurance / regulator-facing methodology at full depth, with your DPIA and bundle as an input.

## Summary — what you can do after this chapter

- Recognise when Art. 35 is triggered and when Art. 36 prior consultation follows from a high residual-risk DPIA.
- Author each of the six required DPIA elements — systematic description, necessity / proportionality, risk assessment, mitigations, residual-risk statement, DPO consultation record — as evidence-cited sections, not prose.
- Populate the AI-specific risk register with the canonical feared events (extraction, MIA, inversion, inferred-attribute disclosure, Art. 22 harms, prompt-injection exfiltration, DSR loss, drift, breach of operation).
- Map Art. 35(7) elements to the ISO/IEC 29134 clauses and the ICO DPIA template sections.
- Name the engineering triggers that require a DPIA refresh and plan the DPIA's version history as an audit artefact.

Chapter 02 picks up with the ICO DPIA template as the concrete skeleton you fill in.
