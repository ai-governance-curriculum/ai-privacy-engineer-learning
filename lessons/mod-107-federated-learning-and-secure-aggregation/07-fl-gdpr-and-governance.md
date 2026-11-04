# Federated Learning Under GDPR — Controllers, Processors, Transfers, and Subject Rights

Federated learning is a technical design pattern that *affects* data-protection law; it does not *replace* it. "The raw data never left the device" is often true in a well-implemented FL system, but under GDPR (and the EU AI Act, and the UK and EEA equivalents), the question the regulator asks is "who is a controller or processor, under what lawful basis, for what personal data, with what obligations". FL rearranges where the processing happens; the same questions still need answers. This chapter is the governance slice of the module: the legal / organisational framing a privacy engineer needs to produce the right technical artefacts for the DPIA (mod-111), the DPO conversation, and the eventual regulator.

Non-trivial caveat up front: **legal opinion is explicitly out of scope** of this module. The privacy engineer's job is to produce the *technical facts* on which counsel can form opinions, and to call out the questions counsel must answer. The sentences that start with "it is likely that…" belong to counsel. The sentences that start with "the system has property X, measured as…" belong to you. The rest of this chapter is written in that mode.

Primary references — read these in your module reading list, then route the legal questions to counsel:

- *Regulation (EU) 2016/679* (GDPR) — Articles 4 (definitions: controller, processor, joint controller, personal data, pseudonymisation), 6 (lawful basis), 26 (joint controllership), 28 (processor contract), 32 (security), 35 (DPIA), chapters V (international transfers), and the full text of Article 44 et seq.
- *EDPB Guidelines 07/2020 on the concepts of controller and processor in the GDPR*. [edpb.europa.eu](https://edpb.europa.eu/system/files/2023-10/edpb_guidelines_202007_controllerprocessor_final_en.pdf) — the practical test for which role each FL participant plays.
- *EDPB Recommendations 01/2020 on measures that supplement transfer tools* — the Schrems II response, relevant when FL participants are in different jurisdictions.
- *EDPB Guidelines 04/2019 on Article 25 Data Protection by Design and by Default*.
- *EU AI Act (Regulation 2024/1689)* — Article 10 (data governance for high-risk AI systems), Article 59 (real-world testing), Articles 61-69 (sandbox / supervisory framework). The AI Act layers on top of GDPR for "AI systems" as it defines them.
- *HIPAA Privacy Rule (US)* and *HHS guidance on de-identification* — if a cross-silo deployment includes US-regulated health data.
- *CPRA / CCPA (US-California)* — for consumer-device cross-device deployments reaching Californians.

## The controller/processor question — the first thing counsel will ask

GDPR assigns duties based on *role*. Two roles matter in every FL deployment:

- **Controller.** "The natural or legal person, public authority, agency or other body which, alone or jointly with others, determines the purposes and means of the processing of personal data" (Art. 4(7)).
- **Processor.** "A natural or legal person, public authority, agency or other body which processes personal data on behalf of the controller" (Art. 4(8)).

Mapped onto FL participants:

| Participant | Candidate role | Under what condition |
| --- | --- | --- |
| Device user (natural person) | Data subject | In cross-device, when the client runs on the user's phone |
| Device OS / app owner | Controller | Processes user data for its own purposes; defined the FL model's purpose |
| Silo institution (hospital, bank) | Controller | Processes patient / customer data for its institutional purpose |
| FL aggregator — same org as controller | — (not separate) | Controller processes data via its own aggregator |
| FL aggregator — third party, acting on controller instructions | Processor | Receives updates on controller's behalf, cannot use them for own purposes |
| FL aggregator — third party, co-determines purposes | Joint controller | Jointly decides what the FL model is for |
| Research consortium coordinator | Joint controller | With the other silos; purposes are jointly determined |

The role determines the contract: a processor needs a *Data Processing Agreement* (Art. 28); joint controllers need a *Joint Controllership Arrangement* (Art. 26) that is transparent to data subjects and that assigns responsibility for each GDPR obligation between them.

**The privacy engineer's job on role:** collect the *technical facts* that counsel uses to classify. "The aggregator computes only the sum of the updates; it does not hold the model weights for any purpose other than returning them to the controller" is the sentence counsel needs. Writing "the aggregator is a processor" is counsel's job.

## Does "the raw data stays on the device" mean "no personal data is processed by the aggregator"?

Short answer: not automatically. Three independent questions apply.

### Is the update itself "personal data" under GDPR Art. 4(1)?

GDPR defines personal data as any information relating to an identified or identifiable natural person. A gradient update is not raw personal data, but **chapter 06's DLG and iDLG attacks establish that updates can, in isolation, be reverse-engineered into training data**. The CJEU's *Breyer* case and the EDPB's Opinion 05/2014 on anonymisation techniques treat data as personal when the re-identification cost is "reasonably likely". If DLG makes re-identification cheap, the update is personal data. If SA + DP make re-identification cost prohibitive, the update may be pseudonymous or anonymous — but that is a case-by-case determination for counsel.

The privacy engineer's job: run the measurement playbook from chapter 06 and report the re-identification cost. Produce the number — the AUC, the TPR@fixed-FPR, the DLG smoke-test outcome — and let counsel apply it to the legal test.

### Is the global model itself personal data?

Separately: the trained model's weights encode something about the training set. Chapter 05's DP-FL bound limits what one person can infer about any one record; mod-105's extraction and MIA measurements limit what an attacker can empirically recover. If the model's memorisation of training data is below a measured threshold, the model's weights are *likely* (counsel's determination) not personal data. If it is not, the model's weights are *likely* personal data and processing the model itself is regulated.

### Are the aggregates themselves personal data?

The aggregated gradient or the aggregated statistics (participation counts, average metrics) may or may not relate to identifiable individuals. The EDPB's view on statistical aggregates is that they are not personal data *when* re-identification is reasonably unlikely. SA + DP + minimum-N-per-round (k-anonymity-flavoured) are the technical levers that push aggregates toward "not personal data". A round that aggregates 3 people's updates with no DP is operating at a different tier from one aggregating 1000 with DP noise.

**The privacy engineer's job:** produce the technical facts — minimum N per aggregate, DP noise scale, attack-measured re-identification cost — and let counsel apply the legal test.

## The lawful-basis question — Article 6 and the FL cadence

GDPR Article 6 lists six lawful bases for processing. Three are commonly invoked in FL:

- **Consent (Art. 6(1)(a)).** The data subject has given consent for the specific processing purpose. Dominant in cross-device consumer deployments: the device user accepts the privacy policy that includes FL participation.
- **Performance of a contract (Art. 6(1)(b)).** Processing is necessary for a contract with the data subject. Rarely sufficient on its own for FL; usually combined with consent.
- **Legitimate interests (Art. 6(1)(f)).** Processing is necessary for the controller's legitimate interests, overridden by the subject's rights. Common in cross-silo settings with sophisticated subjects (patients under hospital care).

The FL-specific traps:

- **Each participating controller needs its own lawful basis for its data.** In a cross-silo deployment with 10 hospitals, each hospital must have a lawful basis to process its own patients' data for the FL training purpose. If hospital 7 does not have a valid basis (patients did not consent, legitimate interests fails the balance), hospital 7 cannot participate, regardless of the technical design.
- **Consent must be specific, informed, unambiguous, and freely given.** A privacy policy line "we may train machine-learning models on your data" is a *general* statement, not a *specific* consent to this training run. For cross-device consent, the specificity bar is where product teams most often get tripped up.
- **The purpose-limitation principle (Art. 5(1)(b)) constrains re-use.** If a hospital collected data to treat patients, repurposing it to train a cross-silo model is a different purpose. Depending on the jurisdiction and the research context, this may need separate consent, a research-exemption analysis, or re-anonymisation.

**Privacy engineer's job:** for each participant, document the lawful basis as stated by that participant's counsel. This is a prerequisite column in the DPIA (mod-111); it is not optional.

## International transfers — Chapter V and Schrems II

FL does not move raw data across borders, but it moves gradient updates. If the updates are personal data (see above), moving them across borders is an international transfer under Chapter V and needs a transfer mechanism (adequacy decision, Standard Contractual Clauses with transfer impact assessment, binding corporate rules, or a derogation under Art. 49).

The Schrems II judgment (2020) bars reliance on Standard Contractual Clauses alone for transfers to jurisdictions with surveillance regimes that can compel disclosure from the recipient. Supplementary measures — technical (encryption, pseudonymisation, DP), contractual, organisational — are required. The EDPB's Recommendations 01/2020 set the evaluation framework.

The practical FL implication:

- **Cross-silo FL between EU and US silos** needs a documented transfer mechanism and, under Schrems II, a transfer impact assessment that enumerates the supplementary measures. SA + DP are plausibly those measures but only when the technical facts support "personal data cannot be reconstructed in the recipient jurisdiction with reasonably likely means". The chapter-06 measurement report is where those facts live.
- **Cross-device FL with EU users and a US-based aggregator** — same story. SA defeats the honest-but-curious server; combined with DP, it is the stack most commonly cited as the "technical supplementary measure" under Schrems II for FL systems. Counsel still writes the transfer impact assessment.
- **Intra-EU or EEA deployments** are simpler but still have local regulators and sector-specific rules (e.g. France's HDS for health data hosting, Germany's BDSG elaborations).

Produce the facts; the transfer analysis is counsel's.

## Joint controllership — the hard case for cross-silo research consortia

Cross-silo FL consortia (medical research, pharma, finance) frequently sit in *joint controllership* under GDPR Article 26. The hospitals jointly determine the purpose (e.g. "train a prostate-cancer risk model") and jointly determine the means (the FL protocol, the model architecture). Each hospital is a controller of its patients' data; collectively, they are joint controllers of the trained model.

Article 26 requires:

- **A joint controllership arrangement** that transparently sets out each controller's responsibilities for GDPR obligations (information to subjects, response to subject rights, security, breach notification).
- **A publicly available essence of the arrangement**, so subjects know who to contact.
- **Clear subject access.** A subject may exercise rights against *any* joint controller; the arrangement allocates the back-office work.

The privacy engineer's job:

- Produce the technical facts the arrangement references: what is each hospital running locally, what does the aggregator do, what is logged where, what retention periods apply, what reproducibility and audit facilities exist.
- Produce the data-flow diagram that is the Annex to the arrangement.
- Set up the technical rail for subject-rights handling (see next section).

## Subject rights in a federated setting — the mechanics

GDPR gives data subjects a set of rights — access (Art. 15), rectification (Art. 16), erasure (Art. 17), restriction (Art. 18), portability (Art. 20), objection (Art. 21), rights related to automated decision-making (Art. 22). Each one has a federated-specific implementation question.

### Access (Art. 15) and portability (Art. 20)

The controller must provide a copy of the personal data it processes. In FL:

- The *raw training data* is held locally by the controller (e.g. hospital, device). Access fulfilment runs through the local controller — no change from the non-FL case.
- Whether *the trained model* or *the gradient updates derived from the subject's data* count as personal data to disclose depends on the counsel analysis above. If yes, the controller must have a facility for subject-specific disclosure. In cross-device, this is usually "the training logs on your device are yours; the global model is the controller's and is non-personal to you". In cross-silo, where the subject is a patient, the parallel analysis applies.

### Erasure (Art. 17) — the unlearning handoff

The right to erasure is the FL-specific bind. The subject's raw data lives on their device / at their hospital; deletion there is routine. But **the trained model contains contributions from the subject's data**. Even one record's gradient has influenced the global model across every round the subject participated in.

Two mitigations:

- **Machine unlearning (mod-110).** The privacy-engineering control that removes a subject's influence from a trained model. In an FL setting, this is harder than in a centralised setting because the round-by-round contribution chain must be unwound. Research directions: FedEraser, Federated Unlearning via Reverse Training. Mod-110 covers this at depth.
- **Participation cap + bounded retention.** Design the system so a subject's contribution is bounded in time (the participation cap from chapter 02 / 05) and in weight (DP noise). Then the "erasure" claim is "your contribution has decayed below the audit threshold after N rounds". Counsel's judgement call.

The privacy engineer owns the technical facility; mod-110 owns the deeper mechanism; the DPIA (mod-111) records the chosen approach.

### Rectification (Art. 16)

Mostly routine — fix the raw data at the local controller; the model may be eventually re-trained. Not usually an FL-specific issue.

### Objection (Art. 21) — "stop training on my data"

Important and tractable. The data subject may object to further processing. In an FL system, this means:

- The subject's client stops participating in future rounds — a flag on the client that excludes it from the selected cohort.
- Historical contributions from past rounds remain in the model (subject to the erasure / unlearning analysis above).

The technical facility: a client-side opt-out signal that the server honours. The audit log records the opt-out event and its effective date. This is the easiest right to implement correctly in FL — build it in by default.

### Automated decision-making (Art. 22)

If the trained model is used to make decisions that significantly affect data subjects (credit decisions, triage decisions, hiring), Art. 22 adds extra obligations — explanation, human review, specific lawful basis. This is model-use governance, not FL-training governance; the FL design does not change the Art. 22 analysis.

## The AI Act overlay — Article 10 and the data-governance obligations

EU AI Act Article 10 ("Data and data governance") applies to high-risk AI systems (Annex III: critical infrastructure, employment, education, law enforcement, migration, essential services, etc.). The relevant obligations:

- **Training, validation, and testing datasets must be subject to data-governance practices appropriate to the intended purpose.** Documented data-source provenance, bias examination, representativeness.
- **Datasets must be relevant, sufficiently representative, free of errors, and complete.** Cross-silo FL raises a specific issue: representativeness is a property of the *union* of silo datasets, which no single silo holds and the aggregator does not see. The data-governance claim is a federated-aggregate claim and needs a federated-aggregate measurement approach.
- **Processing of special categories of personal data for bias correction is allowed only under specific safeguards.** FL does not change the bar; it changes where the processing happens.

The AI Act layers on top of GDPR for the same deployment. A cross-silo medical-imaging FL system is simultaneously a GDPR processing operation (patient data at each hospital) and a high-risk AI system (medical device software under Annex III). Both regimes apply. Mod-111 (DPIA) covers the integrated documentation; this chapter supplies the FL-specific facts.

## The data-flow diagram — the artefact the DPIA will quote

Produce a one-page data-flow diagram for every FL deployment. It is the single most-cited artefact in the DPIA, the joint-controllership arrangement, and the DPO's cross-check. Minimum content:

- **Participants.** Named entities: "Hospital A", "Hospital B", "Aggregator X operated by Charity Y". Jurisdictions marked.
- **Data categories.** What personal data is held at each participant — "chest-X-ray images with associated age/sex/diagnosis labels at Hospital A".
- **Flows.** Arrows showing what moves between participants: "gradient update, L2-clipped to C=1.0, summed under SA + DP with σ=1.1, over TLS 1.3, to Aggregator X".
- **Retention.** How long each participant retains each category.
- **Legal basis.** Per-participant lawful basis for each processing activity.
- **Controls.** SA? DP? TEE? Pseudonymisation at ingest? Logged where?
- **Rights handling.** Which participant handles which subject right; route of the subject's request.

The diagram is the technical substrate for the whole governance stack. Keep it current — if the aggregator changes jurisdiction or the DP parameters change, the diagram changes.

## Red flags in an FL design-review

Five recurring governance red flags the privacy engineer should name during design review:

- **"FL means GDPR doesn't apply."** No. FL changes the processing architecture; the subject's data is still personal data, the controllers are still controllers, the obligations still attach. Correct the framing immediately.
- **No stated lawful basis per participant.** Each participating controller must have a lawful basis. If the design doc is silent on this, nothing downstream is correct.
- **No joint controllership arrangement where one is needed.** If the design has co-determined purposes, counsel must produce the arrangement before launch.
- **Cross-border flows with no transfer mechanism.** Schrems II is current law. Every cross-jurisdictional FL deployment needs the transfer analysis.
- **No subject-rights facility.** At minimum, opt-out from future rounds. Missing this is the most visible rights failure and the most citable in a DPIA review.

Each of these is a *technical facility* question the privacy engineer can resolve, not a legal opinion the privacy engineer must form.

## What the privacy engineer owns, what counsel owns

Repeat, because this is the central discipline:

- **Privacy engineer owns.** Technical facts (what data moves, what noise is added, what the SA parameters are, what the attack-measurement numbers are), technical facilities (opt-out signal, audit log, DP accountant, release package), data-flow diagram, technical supplementary measures.
- **Counsel owns.** Classification of role (controller / processor / joint controller), lawful basis per participant, international transfer mechanism, joint controllership arrangement text, interpretation of subject rights, AI Act high-risk classification, residual-risk acceptance.
- **Jointly produced.** DPIA (mod-111), subject-rights response procedure, incident response plan, DPO consultation record.

A privacy engineer who writes legal opinions is overstepping; a counsel who writes technical specifications is overstepping. Keep the artefacts separated so the review can audit the right person for the right claim.

## Summary — what you can do after this chapter

- Name the four GDPR roles (data subject, controller, processor, joint controller) and apply them to the participants in a cross-device and a cross-silo FL deployment.
- Produce the technical facts counsel needs to classify the updates, the model, and the aggregates as personal / pseudonymous / anonymous.
- Name a lawful basis for each participant in a proposed deployment and flag when any is missing.
- Reason about international transfers under Schrems II and the role SA + DP play as supplementary measures.
- Design the subject-rights facility: at minimum opt-out from future rounds, with the erasure-hard-case escalation to mod-110's unlearning.
- Produce the one-page data-flow diagram the DPIA will cite.
- Flag the five governance red flags during a design review.
- Keep the engineer / counsel boundary clean — fact-production is yours, legal opinion is theirs.
- Point at Article 10 of the AI Act as the next layer for high-risk deployments.

Chapter 08 draws the production-operational boundary to `ai-infra-mlops`, the peer role that owns the FL platform infrastructure.
