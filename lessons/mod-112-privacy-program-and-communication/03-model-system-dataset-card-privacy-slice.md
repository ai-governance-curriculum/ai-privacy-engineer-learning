# The Privacy Slice of a Model, System, and Dataset Card

A model card is the artefact that tells a reader what a model is, how it was trained, how it should and should not be used, and what is known about its limitations. A system card does the same for a composed system (RAG, agent, pipeline); a dataset card for a corpus. Together they are the AI community's documentation shape — the readable face of the technical artefacts under version control. The privacy slice of these cards is where an AI privacy engineer tells a customer, a regulator, a downstream reuser, and your own incident-response team what is privacy-true about the artefact. Doing this well is where the preceding eleven modules land in a form a non-engineer can trust.

This chapter names the eight privacy claims the card must carry, the Hugging Face model-card format they map into, the system-card and dataset-card companions, and the authoring discipline that keeps claims checkable — because an uncheckable privacy claim is worse than no claim at all.

The authoritative references:

- Mitchell et al. — *Model Cards for Model Reporting*, ACM FAT* 2019. [arxiv.org/abs/1810.03993](https://arxiv.org/abs/1810.03993).
- Gebru et al. — *Datasheets for Datasets*, Communications of the ACM, 2021 (originally 2018). [arxiv.org/abs/1803.09010](https://arxiv.org/abs/1803.09010).
- Hugging Face — *Model Cards* documentation, including the Model Card Guidebook and the model-card template. [huggingface.co/docs/hub/model-cards](https://huggingface.co/docs/hub/model-cards); Guidebook at [huggingface.co/docs/hub/model-card-guidebook](https://huggingface.co/docs/hub/model-card-guidebook).
- Google — *Data Cards Playbook*. [sites.research.google/datacardsplaybook/](https://sites.research.google/datacardsplaybook/).
- Microsoft — *Transparency Notes for Azure AI services* (system-card equivalent). [learn.microsoft.com/azure/ai-services/responsible-use-of-ai-overview](https://learn.microsoft.com/azure/ai-services/responsible-use-of-ai-overview).
- Anthropic / OpenAI / Google DeepMind — public *model cards* and *system cards* for frontier models (useful as reference shape; verify current URLs before citing in production cards).
- Hugging Face — *Model Card Regulatory Compliance* notes on how the model card aligns with EU AI Act transparency obligations.
- EU AI Act (Regulation 2024/1689), **Article 53** (general-purpose AI model documentation) and **Annex XI** (information to be provided to downstream providers). [eur-lex.europa.eu/eli/reg/2024/1689/oj](https://eur-lex.europa.eu/eli/reg/2024/1689/oj).

## What the card is, and what the privacy slice is inside it

A model card is the *public summary* of the underlying technical artefacts. It is not the DPIA (mod-111 ch. 01), the system card is not the regulator-facing bundle (mod-111 ch. 06), the dataset card is not the datasheet (mod-103 ch. 01). The relationship:

- The **datasheet** is the internal, exhaustive provenance document for a training corpus.
- The **dataset card** is the external, summary-and-key-decisions document that a downstream consumer reads.
- The **model card** is the external, summary-and-key-decisions document for a trained model.
- The **system card** is the model card's composition: card for a deployed system that uses one or more models plus retrieval, tool-calling, and non-model logic.
- The **DPIA** is the regulatory risk instrument that sits upstream of all of the above.

The privacy slice inside each card is the section the AI privacy engineer owns. For the model card the Hugging Face template calls it *Bias, Risks, and Limitations* / *Training Procedure* / *Environmental Impact* — the privacy slice lands primarily under *Training Procedure* (training-corpus consent-basis), *Bias, Risks, and Limitations* (memorisation residual risk, inferred-attribute risk), and a new *Privacy and Data Protection* section that most of the reference templates do not include by default and that you add.

### The eight privacy claims the card must carry

For a model, a system, or a dataset card to meet an AI privacy programme's standard, it must carry eight claims, each with the cited evidence artefact. The claims are stable across card types — only the subject of the claim changes:

1. **Training-corpus consent-basis statement** — what data went into training (or into the retrieval corpus, for a system card), under what lawful basis, from what sources, with what provenance. (Mod-103 ch. 01, mod-103 ch. 06.)
2. **PII-pipeline claim** — how personal data flowing into training, into retrieval, into prompts / completions, or into logs is detected, redacted or vaulted, and measured. False-positive and false-negative rates. (Mod-104.)
3. **Memorisation-eval residual-risk statement** — how extraction and MIA audits were run, the measured attack advantage, and the residual risk declared. (Mod-105.)
4. **DP (ε, δ) deployment claim** — if differential privacy is used, at what parameters, on what accountant, with what per-example clipping, and what the training-time accountant output was. If DP is *not* used, the reason, with pointer to the mitigation. (Mod-106.)
5. **FL / secure-aggregation claim** — if federated learning is used, cross-device vs cross-silo, the aggregation protocol, the DP-at-user-or-record level, and the client-scope claim. If not, N/A. (Mod-107.)
6. **TEE / HE / MPC deployment claim** — if confidential computing is used, which platform, what attestation profile, what the attestation-report hash is, with what verifier. If not, N/A. (Mod-108.)
7. **Unlearning-capability statement** — whether erasure of a subject's contribution is supported, by what mechanism (full retrain, SISA, approximate unlearning, no mechanism), with what SLO, and with what verification. (Mod-110.)
8. **Logging / retention policy** — what inputs, outputs, intermediates, and metadata are logged; retention windows; who has access; how retention is enforced; where redaction-before-log is applied. (Mod-109 ch. 03 and 04.)

A ninth claim that a system card and dataset card carry in addition:

9. **Data-subject-rights path** — how a subject can exercise access / rectification / erasure / objection against the model's output, against the training corpus, and against the system's logs. SLOs attached. (Mod-110 ch. 01.)

Each claim has three ingredients: the **statement** (one or two sentences for the reader), the **parameters** (concrete numbers the statement commits to), and the **evidence pointer** (path, commit, document, hash — not a prose rephrasing). A claim without the third ingredient is prose; a claim without the second is marketing.

## Mapping the privacy slice onto the Hugging Face model-card template

The Hugging Face model-card template (see the Hugging Face model-cards docs) has a canonical section order. Here is the mapping from the eight privacy claims to the template sections; the exercise for this chapter builds a full card for a chosen scenario.

| Template section | Privacy content you land here |
|---|---|
| *Model Details* | System name, version, controller legal entity, DPIA reference ID, card maintainer, model-card version. |
| *Uses — Direct Use / Downstream Use / Out-of-Scope Use* | Processing purposes in scope; purposes explicitly out of scope; Art. 22 posture (if system makes automated decisions). |
| *Bias, Risks, and Limitations* | Memorisation residual-risk statement (claim 3); inferred-attribute risk; discrimination risk summary; cross-link to mod-105 report. |
| *Recommendations* | Deployment constraints the privacy slice imposes (jurisdictions, user segments, consent gating). |
| *Training Details — Training Data* | Training-corpus consent-basis statement (claim 1); PII-pipeline claim (claim 2); provenance; special-category data treatment; datasheet pointer. |
| *Training Details — Training Procedure* | DP (ε, δ) claim (claim 4); FL / secure-aggregation claim (claim 5); per-example clipping, noise multiplier, accountant pointer. |
| *Evaluation* | Memorisation / extraction audit methodology and result (supports claim 3); MIA-advantage number; evaluation dataset provenance. |
| *Environmental Impact* | (Present for completeness; not privacy-slice primary territory.) |
| *Technical Specifications* | Model architecture summary. |
| *Model Card Authors / Contact* | Maintainer; card review cadence; how to report a privacy concern. |
| **New: *Privacy and Data Protection*** | TEE / HE / MPC claim (claim 6); unlearning-capability statement (claim 7); logging / retention policy (claim 8); data-subject-rights path (claim 9); DPIA ID; residual-risk disposition. |

The *Privacy and Data Protection* section is a section you add to the Hugging Face template; the template is intentionally extensible and several frontier-model cards do add such a section (verify specific references before citing). If your organisation publishes model cards into the Hugging Face hub, the extension is in the YAML metadata front-matter plus the body section.

## Writing the eight claims — the authoring discipline

The principle running through each of the eight is: **a claim is a measurable promise, or it is not a claim**. The patterns below are the shape each claim takes in a well-authored card.

### Claim 1 — Training-corpus consent-basis statement

**Bad:**
> The model was trained on a diverse corpus of publicly available and licensed data.

**Better:**
> The model was trained on 2.3 TB of text sourced from: (a) Common Crawl snapshots CC-MAIN-2024-10 through CC-MAIN-2024-30 after our domain-exclusion list `training/exclusions-2024-08.yaml`, (b) licensed news content under framework contracts with publishers listed in `training/publisher-licensing.md`, (c) employee-opt-in internal documentation under lawful basis Art. 6(1)(f) legitimate interests with LIA at `legal/LIA/training-corpus-v4.md`. No special-category data (Art. 9) is intentionally included; incidental inclusion is addressed via the PII pipeline described below (claim 2). Datasheet: `datasheets/corpus-v4.md`.

The four moves the better version makes: (a) a measured scale number, (b) named source classes, (c) named lawful basis, (d) a pointer to the datasheet. None of the four is optional for a card the privacy programme publishes.

### Claim 2 — PII-pipeline claim

**Bad:**
> We apply PII redaction to the training data.

**Better:**
> The training corpus passes through a redaction pipeline configured for 18 PII entity types (see `pipelines/pii/entities.yaml`). Detectors are Microsoft Presidio 2.2.351 with a custom NER head trained on our labelled data; the evaluation on our held-out benchmark `benchmarks/pii-bench-v3` is F1 = 0.94, with false-negative rate 0.03 on PERSON, 0.05 on MEDICAL_RECORD. Detected spans are replaced with type tokens (`<PERSON>`, `<EMAIL>`, …); the retained-but-not-redacted rows are flagged and sampled for manual review at 1% rate. See `pipelines/pii/eval-report-2024-09.md`.

The moves: named tooling, named thresholds, measured FP/FN rates, named benchmark, pointer to the evaluation report. The reader can audit the claim.

### Claim 3 — Memorisation-eval residual-risk statement

**Bad:**
> We tested the model for memorisation and found it to be low-risk.

**Better:**
> The model was evaluated with the extraction attack from Carlini et al. 2021 on a 1,000-canary set injected at 0.01% frequency; verbatim recall at generation length 50 was 2 canaries (0.2%). A LiRA shadow-model membership-inference attack at the recommended configuration yielded AUC = 0.54 (near-random 0.50) against a held-out non-member set. Our residual-risk threshold for release is AUC ≤ 0.60 and verbatim-recall rate ≤ 1%. See `eval/memorisation-v4.md` and `eval/mia-v4.md`.

The moves: named attacks, named reference, measured results against a declared threshold. The DPIA (mod-111 ch. 01) cites this claim; the regulator will ask to see it.

### Claim 4 — DP (ε, δ) deployment claim

**Bad:**
> We use differential privacy during training.

**Better:**
> Training uses DP-SGD via Opacus 1.5.1 with per-example gradient clipping C = 1.0, noise multiplier σ = 0.85, batch size 2048. The Opacus accountant computes ε = 3.0 at δ = 1e-5 over 4 epochs on the 2.3 TB corpus. Accountant output: `runs/model-v4/dp-accountant.json`. The ε = 3.0 target was selected under the DPIA (mod-111 §4 row R-14) as the operating point that holds the LiRA MIA AUC ≤ 0.60 while retaining task utility within 3 percentage points of the non-private baseline.

The moves: named mechanism, named library and version, named parameters, named accountant output, named rationale for the parameter. If DP is **not** used, claim 4 says so and names the compensating mitigation in claim 3 (e.g., output filtering, MIA-measured residual risk below threshold, lower corpus novelty, etc.).

### Claim 5 — FL / secure-aggregation claim

**Bad:**
> We use federated learning for privacy.

**Better:**
> Model v4 is trained in a cross-silo federated configuration with 12 silos (hospital sites), each contributing one aggregated update per round. Aggregation uses the Bonawitz 2017 secure-aggregation protocol as implemented in `flower-ssa-0.3.0`, with a threshold of 7 of 12 silos required for aggregation to complete. User-level DP is layered at the server with ε = 5.0 at δ = 1e-6 per the DP accountant in `runs/model-v4/fl-dp-accountant.json`. Silos never share raw data; only clipped and noised model updates leave the silo boundary.

The moves: named topology, named silo count, named aggregation protocol, named DP level (record / user / client), named accountant.

### Claim 6 — TEE / HE / MPC deployment claim

**Bad:**
> Inference runs in a confidential computing environment.

**Better:**
> Online inference for the EU endpoint runs on Azure Confidential GPU VMs with NVIDIA H100 confidential computing enabled. The attestation report is verified at session establishment against the expected measurement in `deploy/attestation/expected-measurement-v4.sha256`; a client that cannot verify the attestation refuses to send the prompt. The TEE boundary includes the model weights and the inference-time activations; logs are produced outside the TEE after the redact-before-log pass. Attestation evidence samples are archived weekly at `deploy/attestation/archive/`.

The moves: named platform, named attestation profile and verifier, named boundary (what is inside the TEE, what is outside), named evidence archive.

### Claim 7 — Unlearning-capability statement

**Bad:**
> Users can request erasure of their data.

**Better:**
> Erasure of a user's contribution to the training corpus is supported by: (a) tombstoning in the training data store (SLO: 7 calendar days from DSR validation), (b) exclusion from the next scheduled model retrain (cadence: quarterly). The model is **not** retrained on-demand; a user requiring erasure that must affect model outputs rather than just future retraining is routed to the escalation path in the DSR runbook (`runbooks/dsr-erasure-model-weights.md`). For the retrieval corpus, erasure is immediate (SLO: 24 hours). For prompt / completion logs, redaction and 90-day retention apply by default; see claim 8.

The moves: distinguish erasure from the training corpus, from the retrieval corpus, and from logs; name the SLO for each; name the limitation (not on-demand retraining) honestly.

### Claim 8 — Logging / retention policy

**Bad:**
> Logs are retained for a reasonable period and protected appropriately.

**Better:**
> Inputs (prompts) and outputs (completions) are logged in redacted form only — the redact-before-log pipeline (claim 2) runs synchronously at the ingress to the log sink. Retention is 90 days for operational logs, 30 days for prompt / completion content, and 400 days for metadata-only (timestamp, tenant ID, request ID, model version) for billing and security investigation. Access is controlled by the `prompt-log-reader` IAM role (`access/matrix.yaml`); current role holders audited monthly. Deletion is enforced by S3 lifecycle rules in `infra/s3-lifecycle-prod.tf` and validated by the retention-audit job (`jobs/retention-audit.py`), whose last run report is at `audit/retention-2026-04.md`.

The moves: separate retention by log class; name redact-before-log; name access control; name enforcement mechanism; name the validation job.

### Claim 9 — Data-subject-rights path (system / dataset card)

**Bad:**
> Users can exercise their data-subject rights.

**Better:**
> Data-subject requests are received via `privacy-portal.example.com`, routed to the DSR runbooks in `runbooks/dsr/` (access, rectification, erasure, objection, portability). Identity is verified by the portal; validated requests enter the runbook queue. SLOs: GDPR Art. 12(3) one calendar month, extendable to three months with justification; CCPA / CPRA 45 calendar days, extendable by 45 days with notification. Current rolling 30-day attainment: access 96%, erasure 92%, portability 100% (figures from `audit/dsr-2026-04.md`). Residual gaps and the escalation path are documented in that report.

The moves: named intake, named runbooks, named SLOs by jurisdiction, measured attainment with cited audit.

## The dataset card privacy slice

A dataset card (for a training corpus you release, or that you receive and consume) carries claims 1 and 2 at full depth, and a subset of the others:

- **Composition** — categories of data subjects, categories of personal data, special-category presence, scale.
- **Collection process** — how the data was collected, under what consent or lawful basis, at what time, from what sources.
- **Preprocessing / cleaning / labelling** — what the PII pipeline did, with FP/FN measurements.
- **Uses — appropriate / inappropriate** — the processing purposes the corpus is suitable and unsuitable for.
- **Distribution** — the terms under which the corpus is released or shared internally.
- **Maintenance** — who maintains the corpus, how erasure requests against the corpus are honoured, how the corpus is versioned.

The *Datasheets for Datasets* paper (Gebru et al.) gives the full question list; the dataset card is the public-facing summary of the datasheet.

## The system card privacy slice

A system card composes model cards with the non-model parts of the deployed system — retrieval corpus, tool calling, orchestration logic, identity propagation, logging, deployment surface. Mod-109 ch. 06 produced the RAG system card shape. The privacy slice of a system card:

- Cites the model-card privacy slice for each model in the system (no duplication).
- Adds the retrieval-corpus consent-basis claim (claim 1 applied to the retrieval corpus).
- Adds the per-tenant / per-user retrieval ACL claim (mod-109 ch. 01 and 02).
- Adds the agent-tool data-boundary table (mod-109 ch. 05).
- Adds the logging / retention policy at the system boundary (claim 8 at system level).
- Adds the full data-subject-rights path (claim 9).
- Names the sub-processors in the system and their privacy posture.
- Declares the overall residual privacy risks and their boundaries (what is handled, what is escalated to level 50 or 60).

## Claim quality — the three disciplines

Every claim in the card must pass three tests. They are the self-review the privacy engineer applies before publishing; they are also what a sceptical reviewer (DPO, customer's privacy officer, regulator) applies.

### 1. The checkability test

Can the claim be checked without going through me? A claim that depends on the reader asking the engineer for clarification fails. The pointer to the evidence (JSON file, document path, hash, benchmark name, library version) is what makes the claim checkable.

### 2. The falsifiability test

Can the claim be proven wrong? "Our logs are secure" cannot. "Our prompt-log S3 bucket is encrypted with KMS key `arn:aws:kms:...:alias/prompt-logs-prod`, access-logged to `cloudtrail-prod`, lifecycle-deleted at 30 days" can — a reviewer can check each clause. Falsifiable claims are what distinguish engineering documentation from marketing.

### 3. The honesty-about-limitation test

The hard rule: **the card must name what the model cannot do, not just what it does well**. Unlearning is not supported? Say so. MIA advantage was measured at the ceiling of your comfort? Say so. The retrieval corpus contains employee emails under a legitimate-interest basis that would not hold in all jurisdictions? Say so. A card that reads as if there are no limitations is one of the tells an auditor looks for.

The common failure mode this rule addresses: the card is written by someone whose incentive is to make the model shippable. The AI privacy engineer's role on the card is to be the counterweight — the voice in the room that insists the limitations are documented as clearly as the strengths. If you cannot defend a claim's honesty, don't ship the claim.

## The card as a living artefact

A card has a version. Each model release re-publishes the card; each system change re-publishes the system card. The card's changelog is itself an artefact — a reader or auditor can see what changed when. The common triggers that must bump the card's version:

- A new training run with new data categories.
- A change in the PET parameters (ε, δ, FL mode, TEE profile).
- A change in the redaction pipeline (new detector, new threshold).
- A change in the retention / logging policy.
- A change in the DSR runbook set or its SLOs.
- A newly measured memorisation / MIA result that moves the residual-risk disposition.
- A change in the sub-processor list.

The programme's model-card re-publication ritual (chapter 01) is where these bumps happen. A card whose version has not moved while the underlying system has is a flag.

## The AI Act transparency intersection

EU AI Act Article 53 requires providers of general-purpose AI models to make specified information available to downstream providers; Annex XI lists the information. The model card is one of the natural formats this information is carried in — Hugging Face maintains compliance guidance on aligning the card with Article 53. Privacy claims (claim 1 — training data; claim 7 — unlearning) have obvious overlaps with Annex XI's training-data-summary requirement under Article 53(1)(d).

The privacy engineer's job is to ensure the privacy slice of the card is *at least* sufficient for the AI Act Annex XI requirement; the larger AI Act compliance surface (training-data summary, copyright policy, energy use, systemic-risk assessment) is owned by the governance and legal functions in coordination with the privacy engineer. Do not confuse "I write the privacy slice" with "I own the AI Act model-card compliance for the enterprise" — the latter is an enterprise responsibility you contribute into, not own alone.

## The card's audience — three readers at once

The hardest discipline in writing a card is that it has three readers with different needs:

- **The downstream engineer / integrator** who wants to know what the model can do and what the deployment constraints are. Wants claims to be precise and specific.
- **The customer's privacy officer / DPO / legal counsel** who wants to know what processing is happening and whether their organisation can accept the risk. Wants claims to be lawful-basis-aware and jurisdictionally honest.
- **The regulator / auditor / investigator** who wants evidence that the claims are true and the limitations are disclosed. Wants claims to be evidence-cited and falsifiable.

A card that serves only one reader will fail the other two. The eight claims above are the backbone that serves all three if the three disciplines (checkability, falsifiability, honesty-about-limitation) are kept.

## Summary — what you can do after this chapter

- Enumerate the eight privacy claims the model card must carry — plus the ninth (DSR path) for system and dataset cards — and the module of this track each claim derives from.
- Map the privacy claims onto the Hugging Face model-card template section by section and extend it with a *Privacy and Data Protection* section.
- Write each claim as a measurable statement with named tools, named parameters, measured values, and an evidence pointer.
- Distinguish the model card, the system card, and the dataset card, and name what each composition adds over the one above it.
- Apply the three claim-quality disciplines — checkability, falsifiability, honesty-about-limitation — in self-review and in reviewing colleagues' cards.
- Keep the card versioned with the system and recognise the triggers that require a re-publication.
- Situate the card's privacy slice inside the AI Act Article 53 / Annex XI transparency surface without assuming ownership of the whole AI Act compliance stack.

Chapter 04 picks up with the privacy review board — the cadence in which the card's claims (and the decisions behind them) are reviewed and signed off.
