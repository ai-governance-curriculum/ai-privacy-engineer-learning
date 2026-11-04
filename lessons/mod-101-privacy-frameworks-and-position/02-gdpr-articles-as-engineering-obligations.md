# GDPR Articles 5, 6, 9, 17, 22, 25, 32, 35 as Engineering Obligations

GDPR is the regulation your AI system is most often measured against — in Europe directly, and elsewhere via *adequacy* and *equivalence* arguments other regulators make. This chapter reads the eight articles that an AI Privacy Engineer must be able to recite from memory and map to controls, tests, and evidence.

Primary source: [GDPR (Regulation (EU) 2016/679)](https://eur-lex.europa.eu/eli/reg/2016/679/oj). Read the actual text once end-to-end before you rely on this chapter.

> This chapter does not deliver legal opinion. It is the engineer's *reading* of the text. The DPO or legal counsel is the authority that commits the organisation to a position.

## Art. 5 — Principles

Art. 5(1) names the seven principles — read them as the design contract for every AI-system data flow.

| Principle | Engineer reading | Example AI-system control |
|---|---|---|
| (a) Lawfulness, fairness, transparency | Every processing step needs a basis and a story the subject can be told. | Upstream consent / basis metadata flows all the way to training and inference logs. |
| (b) Purpose limitation | You cannot repurpose a dataset collected for X into AI training for Y silently. | Dataset-card `intended_use` field; purpose-tag required on retrieval index and training corpus. |
| (c) Data minimisation | Collect only what the purpose needs. | Column-level allow-list on training-feature store; field-level retention policy on prompt logs. |
| (d) Accuracy | Keep it up to date; erase or correct when wrong. | Rectification pipeline that reaches retrieval + any fine-tune that embedded the record. |
| (e) Storage limitation | Keep it only as long as needed. | Per-sink retention job; snapshot / backup retention; model-weight retention policy. |
| (f) Integrity and confidentiality | Appropriate technical and organisational security. | Encryption at rest + in transit, KMS integration, access review, attestation of serving environment. |
| 5(2) Accountability | You must be able to *demonstrate* compliance. | The DPIA, ROPA, DP accountant log, PII F1 report, DSR audit trail — collectively your accountability pack. |

**Control shape.** Art. 5 is the article you cite when a system-design decision is in tension with any single principle — e.g. "retention is 90 days because purpose limitation and storage limitation (5(1)(b) + (e)) do not justify longer, and the retention job is tested in CI."

## Art. 6 — Lawful basis

Art. 6(1) lists six lawful bases: (a) consent, (b) contract-necessary, (c) legal obligation, (d) vital interests, (e) public interest, (f) legitimate interests. Every processing operation on personal data needs one.

**Engineering consequence.** A single AI product almost always needs *multiple* lawful-basis decisions — one for training-data acquisition, one for inference, one for logging, one for analytics on output, one for feeding feedback into fine-tuning. The bases can differ.

- *Consent* (6(1)(a)) is granular, revocable, and must be **demonstrable** (Art. 7) — engineer the consent register, the revocation path, and the propagation of revocation into training corpora and retrieval indexes.
- *Legitimate interests* (6(1)(f)) requires a documented **LIA (Legitimate-Interests Assessment)** balancing test. Cross-reference EDPB Opinion 28/2024 on AI models for the regulator stance on using legitimate interests for training-data scraping (chapter 06 of this module reads that opinion).
- *Contract* (6(1)(b)) does not cover "we want this for training and it's convenient" — it covers processing *necessary* to perform the contract with the subject.

**Evidence.** Per processing purpose: a documented basis, a stored LIA where 6(1)(f) is chosen, a consent record where 6(1)(a) is chosen. The DPO owns the choice; you own the mechanism that proves it was collected and honoured.

## Art. 9 — Special categories of personal data

Art. 9(1) prohibits processing of racial / ethnic origin, political opinions, religious beliefs, trade-union membership, genetic data, biometric data for unique identification, health data, and data concerning sex life / sexual orientation — unless one of the 9(2) exceptions applies (explicit consent, employment-law obligation, vital interests, non-profit activities, manifestly-public data, legal claims, substantial public interest, preventive medicine, public-health interest, archiving / research / statistics).

**Engineering consequence.**

- Special-category data in a training corpus is a significant lift — the exception you rely on must be documented, and you likely need additional safeguards (DP, enhanced access controls, minimisation).
- Watch for **inferred** special-category data — a model can infer health or sexual orientation from otherwise non-special data. The EDPB has flagged this; CJEU case law (`OT` and similar) has pushed the scope wider. Treat inferred special-category output as 9-caliber.
- Biometric-for-identification (face, voiceprint, fingerprint hashes) is special-category under 9(1); biometric-for-authentication-only in some member-state readings has narrower scope. Defer the exact line to counsel.

**Controls.** Explicit flag on dataset-card: `special_category: true` with reason + 9(2) exception. Separate access ACL. DP noise or redaction before training. Human review on any output predicted to be special-category.

**Cross-reference.** EU AI Act Art. 10(5) specifically allows processing special-category data to the extent *strictly necessary for bias monitoring* in high-risk AI, with safeguards — chapter 03 of this module.

## Art. 17 — Right to erasure ("right to be forgotten")

Art. 17(1) grants the subject the right to obtain erasure where the data is no longer necessary for the purpose, consent is withdrawn, the subject objects (Art. 21), processing was unlawful, or a legal obligation requires it. Art. 17(3) lists exceptions (freedom of expression, legal obligation, public interest, archiving / research, legal claims).

**Engineering consequence — the hard part of AI.** Erasure is cheap in an app DB and *expensive-to-impossible* in model weights, in LoRA adapters, in fine-tune deltas, in embeddings, and in cached prompts. You need:

- A **data-presence index** — given a subject, which sinks did their data touch?
- A **per-sink erasure** path — app DB, object store, backup / snapshot, logs, embeddings, retrieval index, training-corpus snapshot.
- An **unlearning decision** for weights — the engineering surface for this is `mod-110-machine-unlearning-and-dsr`. The decision tree is: (1) was the subject's data ever in a corpus used for a model version still in use?, (2) if yes, SISA / approximate / LLM-unlearning / retrain-from-trimmed-corpus?, (3) how do you *verify* residual memorisation?, (4) what do you tell the subject about the model versions you cannot practicably unlearn?
- A **deletion attestation** from every processor.
- A **clock**. Art. 12(3) gives you one month to respond, extendable by two further months with reason.

**Evidence.** DSR audit trail with per-sink deletion records, unlearning decision record with reasoning, residual-memorisation (MIA) report after unlearning where relevant, deletion-attestation chain from processors.

## Art. 22 — Automated individual decision-making, including profiling

Art. 22(1) gives the subject the right *not* to be subject to a decision based solely on automated processing — including profiling — which produces **legal effects** concerning them or **similarly significantly** affects them, unless an Art. 22(2) exception applies (contract-necessary, authorised by Union / Member-State law, or explicit consent).

Where 22(2)(a) or (c) is used, Art. 22(3) requires **suitable measures to safeguard** rights and freedoms — at minimum the right to **human intervention**, to **express their point of view**, and to **contest the decision**. Art. 22(4) restricts the use of special-category data in Art. 22 decisions further.

Transparency obligations link in via Art. 13(2)(f), 14(2)(g), and 15(1)(h) — the subject must be told about the existence of automated decision-making and given **meaningful information about the logic involved**, as well as the significance and envisaged consequences.

**Engineering consequence.**

- Is the decision *solely* automated? If a human reviews each decision meaningfully (not rubber-stamp), you may be outside Art. 22(1). The CJEU `SCHUFA` (C-634/21) case tightened this — automated credit-score calculation is Art. 22 processing even if a downstream human makes the final grant decision. Engineer the "meaningful human review" artifact or accept you're in Art. 22.
- Does it produce legal / similarly-significant effect? Credit, insurance, employment, education, housing, benefits — almost always yes. Content ranking usually no, but ranking for recommendation of medical / legal / financial content gets closer.
- You need a **human-intervention path**, a **contest** mechanism, and the **meaningful-information** artifact. The last of these is NOT the model weights — it is a plain-language description of the logic involved, the categories of input used, and the significance and envisaged consequences.

**Controls.** Automated-decision registry. Per-decision log with the inputs, the model version, the threshold, the outcome, and the subject-facing explanation. A real UI path for contestation that routes to a human with authority to overturn. Documented SLA for the contestation clock.

**Evidence.** Automated-decision inventory, the meaningful-information one-pager per decision type, the contestation log, the override-rate metric reviewed periodically.

**Cross-reference.** The CPPA ADMT regulations in California are shaped in the same space but with different thresholds — chapter 04.

## Art. 25 — Data protection by design and by default

Art. 25(1) requires appropriate technical and organisational measures *at the time of the determination of the means for processing* **and** at the time of processing itself — i.e. design-phase and runtime. Art. 25(2) requires that by default only personal data necessary for the specific purpose is processed, including in terms of amount, extent of processing, retention, and accessibility.

**Engineering consequence.** "By default the private option." Concretely:

- Logging of raw prompt / completion is **off** by default; the opt-in path requires justification and routes to a redacted sink.
- Retrieval ACL is per-tenant scoped by default; cross-tenant retrieval is a feature-flag that requires review.
- Training-corpus fields are allow-listed; adding a field requires a documented minimisation review.
- Model weights that touched special-category data have access controls commensurate with the category.

**Controls.** Design-phase privacy review (template lives in `mod-111-dpia-and-privacy-reviews`), privacy-review gate in the SDLC, default-off switches for data-amplifying features, CI tests asserting the defaults.

**Evidence.** The design-phase privacy-review doc per model version, the gate record, the default-config CI output.

**Cross-reference.** Cavoukian's seven Privacy-by-Design principles (foundational, 2009) are the pre-GDPR shape that Art. 25 codifies — covered in `mod-102-privacy-threat-modelling`.

## Art. 32 — Security of processing

Art. 32(1) names the yardstick: *appropriate* technical and organisational measures given the risk, including (a) pseudonymisation and encryption, (b) confidentiality / integrity / availability / resilience, (c) ability to restore, (d) regular testing / assessing / evaluating.

**Engineering consequence.** This is where ML security meets privacy. "Appropriate" is relative to the risk — an LLM with special-category training data on an internet-facing endpoint and a credit-scoring agent require different measures.

- Encryption at rest + in transit (TLS 1.3 minimum, KMS-managed keys).
- Access controls (least-privilege, break-glass with audit, MFA, SSO).
- Pseudonymisation — tokenisation of direct identifiers in training corpora (`mod-103-data-governance-and-minimisation`).
- Serving environment attestation where justified by risk — TEE / confidential computing (`mod-108-confidential-computing-and-ppi`).
- Regular testing — pen-test, privacy red-team, MIA / extraction probe (`mod-105-memorisation-extraction-and-mia`).

**Depth boundary.** Deep ML/AI security at platform scale — model-extraction defence, enclave attestation at fleet scale, KMS / HSM depth — defers up to `security-learning` / `ai-infra-security-learning` (level 35). This track owns the privacy-engineering slice of Art. 32: pseudonymisation, PII pipeline, DP, memorisation testing.

**Evidence.** Encryption-at-rest audit, access-review records, pseudonymisation report, attestation evidence where applicable, MIA / extraction report.

## Art. 35 — Data protection impact assessment (DPIA)

Art. 35(1) requires a DPIA where processing is *likely to result in a high risk* to rights and freedoms. Art. 35(3) lists mandatory triggers: (a) systematic and extensive evaluation / profiling producing legal / similarly-significant effects (overlaps Art. 22), (b) large-scale processing of special-category data (overlaps Art. 9), (c) systematic monitoring of a publicly accessible area at large scale. National supervisory authorities publish their own lists of further mandatory-DPIA cases.

Art. 35(7) names the four mandatory DPIA contents: (a) systematic description of the processing and the purposes, (b) assessment of the necessity and proportionality, (c) assessment of the risks to rights and freedoms, (d) the measures envisaged to address those risks.

**Engineering consequence.** AI systems trip Art. 35 triggers routinely. Your DPIA has to:

- Describe the AI-system data flow — training → retrieval → prompt → inference → logging — concretely, with diagrams.
- Document the necessity / proportionality of each personal-data use (including the alternatives considered and why rejected).
- Catalog the privacy risks — from your LINDDUN-shape threat model (`mod-102-privacy-threat-modelling`) and NIST PRAM worksheet.
- List the measures — minimisation, DP noise where applicable, redaction pipeline, retrieval ACL, retention policy, DSR runbook, confidential-serving posture — with evidence.
- When residual risk remains high after measures, Art. 36 requires prior consultation with the supervisory authority.

**Evidence.** The DPIA document itself is the evidence, cross-referenced to your ROPA, your threat model, your control implementations, and your test outputs. Owned at audit-grade in `mod-111-dpia-and-privacy-reviews`.

## The eight-article checklist for any AI system

When you first scope an AI system, run it against this list:

- [ ] **Art. 5** — Can I state the principle-mapping for each processing step?
- [ ] **Art. 6** — Is there a documented lawful basis per processing operation?
- [ ] **Art. 9** — Is special-category data present (actual or inferred), and is an exception documented with safeguards?
- [ ] **Art. 17** — Can we honour an erasure request across every sink, including a weights-level decision?
- [ ] **Art. 22** — Is any decision solely automated with legal / similar-significant effect, and do we have the human-intervention / contestation / meaningful-information surface?
- [ ] **Art. 25** — Is the default configuration the privacy-preserving one, and is there a design-phase privacy review?
- [ ] **Art. 32** — Are technical / organisational measures appropriate for the risk, and is there regular testing evidence?
- [ ] **Art. 35** — Is the processing likely to result in high risk, and is a DPIA in place with Art. 35(7) contents?

Produce an answer with evidence for each. That is the engineer's GDPR reading.

## Summary

- GDPR Art. 5 is the design contract, Art. 6 is the ticket to process at all, Art. 9 is the special-category gate, Art. 17 is the erasure obligation that reaches into weights, Art. 22 is automated decision-making with a human-intervention surface, Art. 25 is design-and-default, Art. 32 is security, Art. 35 is the DPIA.
- Every obligation maps to a control, a test, and an evidence artifact. Keep that shape in every DPIA section you write.
- The DPO / legal owns the legal reading. You own the engineering surface that makes the reading provable in production.

Next: the EU AI Act's data-governance anchor.
