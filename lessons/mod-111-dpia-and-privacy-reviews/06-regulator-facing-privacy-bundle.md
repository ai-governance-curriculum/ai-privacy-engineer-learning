# Packaging the Regulator-Facing Privacy-Engineering Bundle

A supervisory authority inspection does not open with questions; it opens with a request for documents. The inspector sends a letter — typically naming a list of artefacts, a time window, and a response deadline — and the controller sends a bundle. The quality of that bundle, more than anything the privacy engineer said in any meeting, determines whether the inspection becomes a follow-up visit or a closed file. The bundle is the privacy-engineering programme's work made inspectable: every claim in the DPIA (chapter 01), every PRAM response (chapter 03), every EDPB-cited artefact (chapter 04), every cross-tagged sectoral control (chapter 05) has to be in the bundle, findable, current, and consistent with the others.

This chapter names the eight artefact classes the bundle must contain, the engineering content of each, the version-and-provenance discipline that makes the bundle survive an inspection, and the common pitfalls that sink bundles under audit.

The eight artefact classes:

1. **The DPIA** — the primary document.
2. **The Data-Flow Diagram (DFD)** — the picture the DPIA cites.
3. **The PET Deployment Claim** — differential-privacy parameters, federated-aggregation protocol, confidential-computing attestations, K-anonymity parameters, each with evidence.
4. **The MIA / Extraction Evaluation Report** — memorisation and membership-inference measurements against the deployed model.
5. **The Redaction Pipeline FP / FN Report** — PII-detection pipeline performance on representative held-out eval.
6. **The Unlearning-Capability Statement** — the controller's demonstrated capacity to erase a subject from the model stack.
7. **The Logging and Retention Policy** — per-store retention with enforcement mechanism.
8. **The Breach-Response Plan** — the Art. 33 / Art. 34 / HIPAA / GLBA / Reg S-P playbook.

For each, this chapter names what belongs in the artefact, the acceptance criteria an auditor applies, and the common failure modes. Chapter 07 adds a ninth overlay (CSA AI Controls Matrix and ISO/IEC 42001 AIMS) that an enterprise audit — distinct from a regulatory inspection — will also expect to find in the bundle.

## 1. The DPIA — the bundle's primary document

Chapters 01–05 of this module author the DPIA; here the question is the bundle discipline the DPIA travels under:

- **Versioning** — one DPIA document per system version (not one per inspection). The DPIA's version pin matches the model-version pin matches the system-card pin.
- **Change log** — every revision records what changed and why. An inspector reading the change log should see a plausible, regular pattern — risk register updated when a new attack is published, mitigations updated when a PET is deployed, residual risk reclassified when evidence changes. A DPIA last modified three years before the current model version is a red flag.
- **Approval chain** — the product owner, the DPO (chapter 01 (f)), where applicable the AI RMF / 42001 governance signer, and the controller's privacy officer each sign. Dates and names.
- **Cross-reference map** — the DPIA cites, by file path and version, every artefact in the bundle. The inspector follows citations; if a citation dead-ends, the DPIA fails the follow-the-chain test.
- **Appendix index** — a one-page index of appendices and their artefact classes.

A DPIA stored as a single file in a repository with the model code, versioned alongside the model, linked from the system card, and reviewed on each release is in the shape auditors expect.

## 2. The Data-Flow Diagram — the picture the DPIA cites

The DFD is the single most consulted artefact in an inspection after the DPIA itself. A good DFD answers, at a glance, four questions:

- Where does personal data enter the system?
- Where does it live?
- Where does it leave?
- What crosses a trust boundary (organisation boundary, team boundary, jurisdictional boundary)?

The DFD content that satisfies these questions:

- **Every store** — the six-corpus map (operational DB, training corpus, model weights, retrieval corpus, prompt / completion logs, derivative products), plus any system-specific stores (feature store, cache, offline-reporting store, backup store). Each store annotated with the data category, the retention duration, and the encryption / access-control posture.
- **Every ingress** — collection points (user-facing form, API endpoint, third-party feed, scraping ingest), annotated with the lawful basis and the subject-identifier shape.
- **Every egress** — subject-facing disclosure (model output, access-response), third-party disclosure (vendor API, business partner), cross-border transfers, published artefact release.
- **Every transformation that touches personal data** — inference, embedding, aggregation, redaction, anonymisation, de-identification. Each annotated with the PET (if any) applied.
- **Every trust-boundary crossing** — a visual boundary (dashed line, colour change) with an annotation describing the control that gates the crossing (TLS, mutual auth, DPA, BAA, SCC).

Technical shape: SVG or draw.io / Lucidchart / diagrams.net source, with the source checked into the repository. A PDF screenshot is not a maintainable DFD; a source file that renders to the same shape every time, versioned with the DPIA, is.

The common failure modes in the DFD:

- **Missing stores** — the DFD shows three stores when there are seven. The engineer drew the architecture the product team talks about, not the architecture that exists.
- **Un-annotated trust boundaries** — boundaries drawn without saying what control gates them. An inspector will ask.
- **Stale** — the DFD shows last quarter's architecture. If the DFD is not updated on each architectural change, it rots in days.
- **Marketing-altitude** — three cloud icons and an "AI" bubble. Useless for an inspection. The DFD must be engineer-readable.

A controller that keeps the DFD in `docs/` as a source file, renders it in the system README, and refreshes it on each architectural PR has an auditable DFD.

## 3. The PET Deployment Claim — parameters and evidence

For each Privacy-Enhancing Technology deployed in the system, the bundle contains a one-page claim. The structure:

- **Claim** — what the PET deployment achieves, in one sentence. "The recommendation training pipeline runs DP-SGD with (ε, δ) = (3.0, 1e-5) over the full training corpus, under the Rényi-DP accountant implemented in Opacus v1.5, with per-example gradient-norm clipping C = 1.0 and noise multiplier σ derived from the accountant target."
- **Parameters** — all parameters that enter the privacy guarantee. ε, δ, C, σ, batch size, number of steps, number of participants (for federated learning), attestation certificate hash (for confidential computing), K / L / T (for k-anonymity / l-diversity / t-closeness), model-retrieval attack-advantage bound (for empirical PET evaluation).
- **Evidence** — the artefact that proves the parameters are what the claim says. For DP: the accountant's output log, the training-run manifest, the random-seed record. For federated learning: the secure-aggregation protocol specification, the participant-threshold configuration, the protocol-exchange log for a representative run. For confidential computing: the attestation report, the enclave-measurement hash, the key-release policy that binds data decryption to the attestation. For K-anonymity: the released-table manifest and the k-value measurement log.
- **Threat model** — what the PET defends against and what it does not. DP defends against membership-inference and (weakly, at low ε) against extraction; it does not defend against model inversion in all cases; it does not defend against a compromised training environment. The bundle states the scope.
- **Re-evaluation cadence** — the schedule on which the claim is re-evaluated.

One PET deployment claim per PET deployed. A bundle for a system that has deployed DP training, confidential inference, and redaction will contain three claim pages. A bundle that lists "we use differential privacy" without parameters and without evidence has no PET claim.

### The DP accountant output as the evidence standard

For DP training, the accountant output is the standard evidence. The claim should include an excerpt — not the whole log, but enough that the parameters are traceable:

```
Training run: recsys-v4-2026-09-15-dp
Framework: Opacus 1.5.1
Accountant: Rényi DP (RDP) with alpha_max = 64
Per-example clipping C: 1.0
Noise multiplier sigma: 1.1
Batch size: 4096
Steps: 50000
Delta target: 1e-5
Reported epsilon at delta=1e-5: 3.01

Hash of accountant log: sha256:...
Hash of training manifest: sha256:...
```

The hash chain lets an auditor pull the full log and recompute the privacy accounting. A bundle that commits to the discipline of recording and hashing the accountant output can be audited mechanically; a bundle that asserts "DP applied" cannot.

### Confidential computing — the attestation claim

For confidential inference (TEE-backed), the claim contains:

- The TEE platform and generation (e.g., Intel TDX, AMD SEV-SNP, NVIDIA Hopper Confidential Computing).
- The attestation report format and the service that issues it (DCAP, Microsoft Azure Attestation, Google Confidential VM attestation, AWS Nitro Enclaves attestation document).
- The expected measurement values that the attestation must match (MRENCLAVE / MRSIGNER for SGX-family; TDX report body fields; SEV-SNP measurement; GPU attestation hash for CC-enabled GPUs).
- The key-release policy that binds decryption of data to an accepted attestation.
- An example attestation-report hash and the verification pathway.
- The reference point for ongoing attestation (per-session, per-boot, per-model-load).

A single attestation certificate does not prove continuous confidentiality; the operational discipline (re-attestation cadence, revocation handling, key-release policy) does.

## 4. The MIA / Extraction Evaluation Report

The DPIA cites measured values; the measurement lives in this report. The structure the auditor expects:

- **Scope** — what model, what version, what deployment surface, what training data was in scope.
- **Attack set** — the attacks run. The standard contemporary set includes: LiRA-style white-box membership inference (Carlini et al.), Yeom-style loss-based membership inference, Shokri-style shadow-model MIA, Carlini verbatim extraction on LLMs, Nasr-style scalable extraction against production LLMs, canary-based memorisation measurement, model-inversion against embeddings, and (where relevant) attribute-inference attacks. Each attack cited to its paper with the exact variant used.
- **Attacker model** — what access the attacker has (black-box API with rate limit, white-box weights, partial training-data knowledge, auxiliary data).
- **Target population** — whose susceptibility is measured. Typically the full training population; for sensitivity-aware systems, the measurement should be stratified (per data-category, per subpopulation).
- **Advantage metrics** — the measured attack advantage expressed in the paper's native metric (TPR at low FPR for LiRA; exact-match rate for extraction; accuracy of attribute inference).
- **Compare-against baseline** — the attack advantage against the DP-trained model, the non-DP-trained model, and a reasonable utility-matched baseline, so the reader can see what the mitigation bought.
- **Acceptance threshold** — the controller's threshold for "acceptable measured advantage". Above the threshold, the system is not shippable without further mitigation. Below, it passes.
- **Replication artefact** — the code, dataset, and seed used, hashed, and committed to the bundle (or cited to a reproducible run). An inspector who cannot reproduce the measurement cannot audit the claim.

The report is typically 10–30 pages for a nontrivial AI system. For a very large model, abbreviated stratified-sampling versions are acceptable *if* the methodology is documented and the sampling is honest; aggressive sampling that happens to show low advantage is suspicious and will be challenged.

### The acceptance-threshold discipline — residual honesty

The acceptance threshold is the residual-risk honesty gate. If the controller sets its threshold at TPR@FPR=0.1 = 0.6 and measures 0.42, the system passes; if the threshold is 0.6 and the measurement is 0.78, the system fails and either the mitigation is strengthened (re-run the measurement) or the DPIA's residual-risk column admits "High" and Art. 36 is engaged. A controller that sets its threshold *after* seeing the measurement to move the pass line is engaged in audit-fraud territory. The bundle should record the threshold-setting discipline: the threshold is set from the DPIA's residual-risk policy, approved by the DPO, before the measurement is run.

## 5. The Redaction Pipeline FP / FN Report

PII detection / redaction is one of the primary engineering controls for text-ingest AI (RAG corpora, prompt-log scrubbing, training-corpus cleanup). Its performance is directly privacy-relevant: a FN (missed PII) is a privacy leak; an FP (over-redaction) is a utility loss that drives the business case against the control. The bundle contains the FP / FN report.

The structure:

- **Pipeline specification** — the detectors (Presidio NER plus custom recognisers, regex, LLM-based), the ordering, the thresholds, the categories detected.
- **Evaluation data** — the held-out eval set, its provenance, its coverage per category, its size.
- **Metrics** — per-category precision, recall, F1. Reported with confidence intervals.
- **Error analysis** — the typology of FN (what the pipeline misses: multilingual PII, novel formats, context-dependent entities) and FP (what it over-redacts).
- **Deployment configuration** — the operational thresholds set per use-case (ingest vs. log-scrub vs. output-filter).
- **Re-evaluation cadence** — when the eval set is refreshed and the detector re-measured.

The audit expectation is that FP / FN rates meet the acceptance criterion set in the DPIA, and that the eval set is representative of the data the pipeline actually sees. A pipeline evaluated on CoNLL English-only text but deployed on multilingual user content will fail audit even if the English numbers look good; the bundle must document the representativeness analysis.

## 6. The Unlearning-Capability Statement

Mod-110 chapters 02–05 cover unlearning engineering; the bundle contains the capability statement that summarises what the controller can do. One page:

- **The erasure-vs-retraining decision tree** — the controller's documented policy for how it routes an erasure request.
- **The exact-unlearning capability** — if SISA-partitioning is in place, the partitioning scheme, the per-shard retraining cost and latency, the quarterly-retrain cadence.
- **The approximate-unlearning capability** — the unlearning method(s) available (influence-function-based, gradient-ascent, task-vector, Who's-Harry-Potter-style anchor method), the verification test (MIA-based), the measured residual signal.
- **The retrieval / corpus / log erasure capability** — the delete-by-subject primitive across every store, the propagation pipeline, the SLA.
- **The derivative-product erasure strategy** — per derivative, the action.
- **The SLA** — the time from DSR intake to completion, with the per-component contribution to the SLA.
- **Evidence of a drill** — the controller has run an end-to-end erasure drill at least once since the last bundle refresh, with the artefact of the drill.

The drill discipline matters. A capability-statement that claims an unlearning capability but has never actually run one is a claim the auditor will probe. A drill run quarterly, with a documented synthetic subject, an end-to-end execution, and a verification artefact, is auditable evidence.

## 7. The Logging and Retention Policy

A one-page matrix with one row per store (the six-corpus map plus system-specific stores) and columns:

- Store name and location.
- Data categories (including special-category / sensitive flag).
- Retention duration.
- Legal / business basis for the duration (per-row justification — "operational troubleshooting, measured mean-time-to-resolution 8 days, margin of safety 2x").
- Enforcement mechanism (lifecycle rule, scheduled purge job, automated deletion).
- Audit log of enforcement (the log that proves the enforcement runs and succeeds).
- Owner.

The common failure: policy specifies 30-day retention; S3 lifecycle rule is 2 years; no one has looked. An inspector who checks the actual configuration against the policy and finds divergence will treat the whole bundle with suspicion. The engineering practice: infrastructure-as-code definitions of lifecycle rules, with the retention matrix and the code reviewed together on every change.

AI-specific retention rows to watch:

- **Training-corpus snapshot retention** — immutability for reproducibility vs. erasure propagation. The policy should name both and the resolution (chapter-01 (a) of mod-110).
- **Prompt / completion log retention** — the balance between operational value and surveillance risk. Shorter is better from a privacy perspective; the policy should justify the duration.
- **Embedding retention** — easy to overlook. The embedding corpus is a derivative; it has its own retention.
- **Model-weight retention** — old model versions retain the data they were trained on. Retention of old model versions is retention of their training data.
- **Backup retention** — the backup store is a derivative of every other store. Its retention must be considered.

## 8. The Breach-Response Plan

The last of the eight. The plan walks the controller from detection to notification to post-incident review. For an AI system the plan has AI-specific decision branches; the plan must engage:

- **Detection** — the signals that count as a breach-indicating event: extraction-like query patterns, anomalous model behaviour, prompt-injection indicators, access-log anomalies, vendor-incident notifications, disclosures from a subject or third party, media reports, security-researcher disclosures.
- **Triage** — who assesses whether the event is a breach, against what criteria, on what clock.
- **Containment** — immediate actions: rotate keys, suspend inference, suspend training, isolate the affected store, notify processors.
- **Investigation** — what happened, who is affected, what data is involved, what the scope of the exposure is. For AI systems: did the breach involve training data? If so, can the controller determine which training data? Was the model's output involved in the exposure?
- **Notification decision** — against the applicable statutory tests:
  - GDPR Art. 33: notify the DPA within **72 hours** of becoming aware, unless the breach is unlikely to result in a risk to rights and freedoms.
  - GDPR Art. 34: notify the affected subjects when the breach is likely to result in a **high risk** to rights and freedoms.
  - HIPAA §164.404–408: notify subjects within **60 days**; notify HHS contemporaneously for 500+; notify media for 500+ in a state.
  - GLBA §314.5: notify FTC within **30 days** for unencrypted customer information of 500+ consumers.
  - SEC Regulation S-P: notify customers within **30 days** of determining a sensitive-customer-information incident occurred.
  - State breach laws — the state-specific notification decision per jurisdiction.
- **Notification execution** — the template, the channel, the DPO signature pathway, the customer-communications owner.
- **Post-incident review** — root cause, lessons, DPIA / PRAM update, control changes.
- **Drill cadence** — annual tabletop exercise at minimum, with AI-specific scenarios in the rotation.

The most-overlooked AI-specific scenarios the plan should include:

- **A memorisation disclosure** — an external researcher reports that the model emits verbatim training data including PII. The response pathway: validate, contain (output-filter deployment, model takedown if necessary), scope (which training data is affected? which subjects?), notify under the applicable statutes.
- **A prompt-injection data exfiltration** — a user reports (or a researcher demonstrates) that the retrieval corpus can be exfiltrated via injection. Response: patch the prompt-injection defence, retrain the output filter, assess scope, consider whether confidentiality has been breached.
- **A vendor model-provider breach** — the hosted LLM provider reports a compromise. Response: assess which prompts and completions were in the affected window, which were PII-bearing, notify under the processor-breach pathway.
- **A DSR failure** — the controller fails to meet a DSR SLA on a material request. Response: address the specific request, assess whether the failure is systemic, decide whether the failure itself is reportable under Art. 33 (the ICO has treated systemic DSR failure as reportable).

The plan is one document per controller; the AI-specific branches live within it. The AI privacy engineer's input is the AI-specific branches and the drill-scenario rotation.

## Bundle discipline — the metadata that makes the bundle inspectable

The eight artefact classes plus the discipline that makes them findable:

- **The manifest** — a one-page table of contents, one row per artefact in the bundle, with the file path, the version, the hash, the applicable-obligation tags, and the owner. The chapter-05 cross-tagging lives here.
- **Hashing** — each artefact's current version is content-hashed; the hash appears in the manifest and in the DPIA's citation. An inspector who wants to confirm "the DPIA cites the MIA report at hash X" can compare hashes.
- **Versioning** — each artefact is versioned. The bundle is a specific set of versions: the DPIA v3.2 cites DFD v4.1, PET-claim-DP v2.0, MIA report v3.0, etc. The manifest records the exact set.
- **Storage** — the bundle lives in a repository (version-controlled, access-controlled) or a document-management system (indexed, access-controlled). A bundle scattered across team drives with "final-final-v2.pdf" filenames is not inspectable.
- **Access control** — the bundle contains sensitive material (attack-advantage measurements, retention schedules, incident records). Access is restricted to the privacy-engineering team, the DPO, legal, the audit response team, and the system owner. The inspection-response pathway is explicit: who prepares, who reviews, who sends.
- **Freshness checks** — the bundle has a review cadence (at least quarterly, with each major-release-triggered refresh). The review record is in the bundle.

A controller running the bundle under this discipline can respond to an inspection letter within days rather than weeks.

## The inspection-response playbook

When the inspection letter arrives, the playbook:

1. **Day 0** — the DPO / privacy officer reads the letter, logs it, assigns the response owner (the privacy engineer for a technical inspection; the DPO for a documentary inspection; both for a comprehensive inspection), and sets the internal timeline against the external deadline.
2. **Day 1–3** — the response owner pulls the bundle, cross-references the letter's artefact list against the manifest, and identifies gaps. Gaps are either (a) artefacts the controller does not have — a bad finding, but must be acknowledged honestly — or (b) artefacts the controller has but under a different name — add the mapping.
3. **Day 3–7** — the response owner drafts the response letter: artefact-by-artefact, with the citation into the bundle. Legal reviews. The DPO signs.
4. **Day 7–14** — the response, with the bundle artefacts, is sent.
5. **Post-send** — the response is logged, the DPIA's change log is updated with the inspection event, and any gaps surfaced by the response become inputs to the next DPIA refresh.

The playbook is itself an artefact. It belongs in the bundle alongside the breach-response plan.

## Common bundle failure modes

- **Single-asset DPIA** — the DPIA is the only artefact; nothing else is prepared. Fix: build out the eight artefact classes one at a time; a partial bundle is better than a single-asset submission, because the inspector can at least see the programme is forming.
- **Stale** — the bundle is last year's. Fix: review cadence tied to model release.
- **Inconsistent** — the DPIA cites parameters that do not match the PET-claim page; the DFD shows stores the retention policy does not. Fix: regular cross-reference audit.
- **Un-indexed** — the artefacts exist but no one can find them in under half a day. Fix: the manifest discipline.
- **Un-hashable** — artefacts are PDFs produced ad-hoc from mutable spreadsheets. Fix: source files in version control; renders derived from source.
- **DPO-signed without engagement** — chapter-01 failure mode repeated at the bundle level.

## The bundle as an engineering programme, not a document set

The bundle is not something the privacy engineer assembles in the week before the inspection letter. It is the programme's ongoing output. Every architectural decision updates the DFD. Every PET deployment produces an evidence hash. Every model release triggers an MIA audit refresh. Every incident drills the breach-response plan. The programme runs the discipline; the bundle is a snapshot an inspector can take any day without the team changing anything.

A controller that approaches the bundle this way can handle multiple concurrent inspections across jurisdictions without surge work. A controller that approaches it as a one-time pre-inspection effort will burn its privacy-engineering team out and still ship an incomplete bundle.

## Summary — what you can do after this chapter

- Assemble and version-control the eight artefact classes of the regulator-facing privacy bundle: DPIA, DFD, PET deployment claims, MIA / extraction report, redaction FP / FN report, unlearning-capability statement, logging / retention policy, breach-response plan.
- Author each PET deployment claim with parameters, evidence, threat model, and re-evaluation cadence.
- Produce an MIA / extraction report with reproducible attacks, honest advantage metrics, and a pre-committed acceptance threshold.
- Produce a redaction FP / FN report with a representative eval and a per-category metric.
- Maintain a bundle manifest with hashing, versioning, access control, and a quarterly freshness review.
- Run the inspection-response playbook when an inspection letter arrives.
- Operate the bundle as an engineering programme that produces evidence continuously, not as a pre-inspection document assembly.

Chapter 07 adds the enterprise-audit overlay — the CSA AI Controls Matrix and the ISO/IEC 42001 AIMS clauses — that an enterprise auditor (as distinct from a supervisory authority) will expect the same bundle to answer.
