# Datasheets for Datasets — The Engineering Interface

A datasheet is the dataset-side analogue of a model card. Timnit Gebru and co-authors proposed them in 2018 ("Datasheets for Datasets", revised 2021) specifically because the datasets that trained production ML systems were being consumed without any consistent description of where they came from, what they represent, and what they should (not) be used for. For an AI Privacy Engineer, a datasheet is the *document the regulator and the downstream consumer reads first* — and it is the artifact in which most of the lawful-basis / de-identification / retention claims you make for a corpus are recorded in human-readable form.

This chapter is the engineering adaptation: what to put in a datasheet when the dataset is a training corpus, a retrieval corpus, a fine-tune corpus, an evaluation set, or a red-team set; how to keep the datasheet synchronised with the running pipeline; and the residual-risk claim that closes the document.

Primary source:

- Gebru, T. *et al.* — *Datasheets for Datasets* (CACM, 2021 / arXiv 2018). [arxiv.org/abs/1803.09010](https://arxiv.org/abs/1803.09010).

Read the paper's question list once; this chapter assumes you have.

## Why this matters

The default state of AI datasets is "undocumented". A fine-tune corpus is a directory in object storage; a retrieval corpus is a set of chunks in a vector DB. A regulator, an acquirer, or an auditor cannot look at that directory and tell you:

- Was it collected under consent, contract, legitimate interests, or no identified lawful basis at all?
- What personal-data classes does it contain? Special-category? Children's?
- Was it de-identified, pseudonymised, or raw?
- Who is allowed to access it? For which purposes?
- When does it get deleted?
- What is the *residual risk* you are asking the deploying team to accept?

The datasheet is the document that answers those questions once, so every downstream reviewer does not need to re-derive them from code. GDPR Art. 30 record-of-processing and Art. 5(2) accountability both land in this document; EU AI Act Art. 10 for high-risk systems requires data-governance documentation that is substantially the datasheet shape; NIST AI RMF MAP 2 / MAP 3 call for it under the "data and input" measure.

## When a datasheet is required

For an AI system, you should have a datasheet for *every* dataset in the mod-102 nine-sink template that holds personal or regulated data:

- The **training corpus** (base-model pretrain data; fine-tune corpus; RLHF preference data).
- The **retrieval corpus** (RAG index; lookup tables; knowledge-base dumps).
- The **evaluation set** (static benchmarks; private hold-outs; red-team sets).
- The **prompt / completion log** when reused for fine-tuning, eval, or red-teaming — once reused, it has become a dataset in its own right.
- The **synthetic-data substitute** (chapter 06) — the datasheet is also the audit-result handoff.

For public base models you consume (Llama, Mistral, etc.), the vendor supplies — or should supply — the base-model datasheet; your datasheet stacks on top of the fine-tune + retrieval surface you added.

## The Gebru et al. question set, grouped for engineering use

Gebru's question list has seven sections. I group them into four engineering-facing concerns below. Keep the mapping to the original sections visible so that a reader familiar with the paper can walk your datasheet.

### Group A — Provenance and motivation

Maps to Gebru §1 (Motivation) and §2 (Composition) and §3 (Collection Process). These are the questions with the highest regulator-interest and the most frequent failure mode (silence).

- Who created the dataset, for what purpose, funded by whom?
- What does an instance represent — a person, a document, an event, a synthetic record?
- How many instances, how were they sampled, from what population?
- Is there a sampling / coverage bias that is material to representativeness (EU AI Act Art. 10(3))?
- Are relationships between instances (same subject across rows; same author across documents) made explicit?
- Is there a recommended data split (train / dev / test), and why that split?
- Are any instances known to be duplicates, near-duplicates, or variants of each other?

Engineering tip: a stable record-identifier scheme at ingest (hash of a stable key, not a monotonically-increasing ID that leaks timing) simplifies every one of these answers.

### Group B — Personal-data and consent posture

Maps to Gebru §3 (Collection Process) and §4 (Preprocessing / Cleaning / Labeling) and §6 (Distribution). The GDPR Art. 5 / Art. 6 / Art. 9 answers live here.

- Did the dataset collection involve personal data in the GDPR / sector-law sense? If so:
  - What classes of personal data? Special-category (Art. 9)? PHI (HIPAA)? PCI? Children's (COPPA / GDPR Art. 8)?
  - Lawful basis at collection — consent, contract, legitimate interests, legal obligation, public task, vital interest?
  - If consent, how was it captured, is the record retrievable, is it granular and revocable?
  - Was the subject notified the data would be used for AI training (specific-purpose test in EDPB Opinion 28/2024)?
  - Cross-border transfer involved? Under which mechanism (SCCs, adequacy, BCRs)?
- Did the dataset get de-identified / pseudonymised / anonymised? By which technique? With which residual re-identification risk?
- Was labeling / annotation done in-house or outsourced? Did labellers see raw personal data? Were they vetted / contracted appropriately?
- Any individuals excluded by opt-out, do-not-sell, or regulatory requirement?

Engineering tip: carry the lawful-basis + consent-token at the record level (chapter 03). The datasheet summarises; the records carry the proof.

### Group C — Intended use and distribution

Maps to Gebru §5 (Uses). The EU AI Act Art. 10(2) relevance claim and the "intended / unintended use" of a NIST-style model card land here.

- Recommended / intended uses — what tasks, what populations, what deployment contexts?
- Known / anticipated *un*intended uses — the uses the authors do not endorse (re-identification; automated decision-making affecting rights; training a competitor model; etc.).
- Usage restrictions — dataset licence, contractual restrictions, regulator-imposed restrictions.
- Downstream distribution — is this dataset shared externally, with whom, under which agreement?
- Known biases / limitations / representativeness gaps relevant to the intended uses.

Engineering tip: write the *un*intended-use list with the product team present. These are the uses the product team will quietly try next quarter unless the datasheet calls them out.

### Group D — Maintenance, retention, and residual-risk claim

Maps to Gebru §6 (Distribution) and §7 (Maintenance). The storage-limitation + accountability answers live here.

- Owner of the dataset (role + on-call contact + DPO contact).
- Versioning scheme. How are new versions produced? How are old versions retired?
- Retention — when the dataset expires, how deletion is achieved, which downstream artifacts depend on it.
- How errata / corrections / rectifications are applied (DSR rectification routing — mod-110).
- How erasure requests reach this dataset and its downstream artifacts (DSR erasure routing — mod-110).
- **Residual-risk claim** — the single most load-bearing paragraph in the document. In plain language, what residual privacy risks remain after controls, who accepted them, and what would trigger reconsideration?

A datasheet without a residual-risk claim is a questionnaire. A datasheet *with* a residual-risk claim is a review artifact. The claim is signed by the DPO / data owner / model owner.

## Datasheet for a *training* corpus — worked outline

```yaml
# datasheet.v3.yaml
id: ds.ticket-corpus.2025-q4
title: Customer-support ticket corpus — 2025 Q4 fine-tune slice
version: 3
owners:
  data: dpo@example.test
  engineering: ml-platform@example.test
# Group A — provenance
motivation: >
  Train a tenant-scoped support-assistant fine-tune. Follows ds.ticket-corpus.2025-q3.
creator: Example Corp ML Platform team
collection_period: 2025-07-01 .. 2025-09-30
sampling: random 5% of resolved tickets per tenant, stratified by category
instance: one resolved-ticket thread (customer msgs + agent msgs + metadata)
size: 412_998 threads across 1_104 tenants
splits: {train: 0.90, dev: 0.05, test: 0.05 — split on tenant hash to prevent tenant leakage}
duplicates: near-dup cluster drop at Jaccard 0.85; see dedup.log
# Group B — personal-data
personal_data_classes: [name, email, phone, order_id, billing_address, free_text]
special_category: false   # no health / political / religious signal after redaction; see redaction-report
children_data: excluded via tenant-metadata filter age_declared < 16
lawful_basis: legitimate_interests (training to improve service of same relationship)
lia_doc: lia.ticket-corpus.2025-q4.md         # legitimate-interests assessment
consent_available_for: <subset> opted into ML-improvement at onboarding (consent_token in metadata.consent_token)
deidentification:
  - presidio-v2.4 ingest redaction (see mod-104)
  - tokenisation of {name,email,phone,order_id,billing_address} into vault tokens (see mod-103 ch. 05)
  - residual re-id risk: free-text may still contain contextual quasi-identifiers; see residual-risk section
labelers: in-house SRE team; viewed redacted data only; NDAs on file
cross_border: EU-EU only; US tenants excluded from this slice
# Group C — intended use
intended_use: fine-tune of base-llm-7b for tenant-scoped customer-support assistant
unintended_use:
  - re-identification of any ticket author
  - cross-tenant generalisation (model shall decline cross-tenant queries)
  - training a third-party model (licence prohibits redistribution)
licence: internal-only; see data-licence.md
# Group D — maintenance + retention
retention: 24 months from dataset freeze; deletion scheduled for 2027-10-01
rectification_sla: 30 days
erasure_sla: 30 days; reaches raw corpus, tokenised corpus, fine-tune weights (via mod-110 unlearning plan)
residual_risk: |
  After redaction, tokenisation, and dedup, the free-text residual re-id risk is
  judged low-moderate. The specific scenario is a verbatim quote of a rare phrase
  repeated verbatim in the generated response (memorisation per mod-105). The
  compensating control is prompt-completion dedup against training set (ratio N:1,
  see dedup.log) and the monthly LiRA audit. Residual risk accepted by: DPO 2025-10-14.
```

The YAML is the machine-readable slice; a human-readable Markdown datasheet accompanies it with the same section headings. The YAML is what the ML-BOM references (chapter 07).

## Datasheet for a *retrieval* corpus — what differs

The Gebru question list was written before RAG was a typical architecture. For a retrieval corpus, the following fields matter in addition to / in place of the training-specific ones:

- **Tenant scope** — per-tenant / shared / public? The highest-leverage privacy control for RAG is the ACL (mod-109).
- **Chunking scheme** — what is a "record" at retrieval time (document; passage; semantic chunk; sliding window)?
- **Freshness / replication lag** — how quickly does an upstream deletion propagate to the index?
- **Embedding model + version** — the *embedding* is a derivative data store and, by vec2text and related attacks, is not privacy-neutral. Record the encoder.
- **Author consent to index** — authors often consent to publication but not to vectorisation for third-party RAG. Record the authorisation separately.
- **Right-to-be-forgotten path** — when an upstream record is deleted, the datasheet names the job that re-indexes to remove it. This is the retrieval-side half of mod-110 unlearning.

A retrieval-corpus datasheet is shorter than a training-corpus datasheet but has *more* edges that cross trust boundaries. Draw them.

## Datasheet for an *evaluation* corpus — what differs

Evaluation sets look innocuous — they are small, they are "just metrics". They are also the most frequent carrier of accidentally leaked personal data because they were sampled from production and then copied around. A separate datasheet is warranted:

- Sampling procedure from production — stratification, size, randomness.
- Access scope — who reads this eval set, from which network, into which notebook.
- Freeze policy — a "live" eval set that drifts with production is a leaky sink; freeze or rotate.
- Residual-risk claim specific to the eval surface.

Treat the red-team / adversarial-prompt set as a separate dataset again; it will contain the most sensitive strings the system has ever seen (that is the point).

## Versioning and regeneration

Datasets are regenerated. Each regeneration is a *new dataset* and gets its own datasheet (ideally by patching the previous version's YAML). The `id` scheme should include the regeneration timestamp or version counter. Three version-management rules:

- **The datasheet lives in the same repo as the dataset manifest.** Not in a wiki. Not in a shared drive. Version the markdown, version the YAML, diff them.
- **Datasheet version is referenced by downstream artifacts.** A trained model's model card points to the exact datasheet version it consumed. The ML-BOM (chapter 07) points to the same.
- **A datasheet change triggers a review workflow.** New special-category class? New cross-border transfer? New lawful basis claim? These are design-review-level changes and PR'd as such.

## The residual-risk claim — write it like a paragraph, sign it

The residual-risk claim is deliberately in the most human-readable section of the document. The regulator-facing audience is not reading YAML; they are reading a paragraph that says:

> After the controls described above, we assess the residual privacy risk as *low-moderate*. The scenarios we believe remain plausible are: … The compensating control for each is: … The trigger for reconsidering this assessment is: … Reviewed by: DPO / DPO delegate on …

A reviewer who reads that paragraph and nothing else should be able to:

- name the specific scenarios you are worried about,
- see that you have a *control* against each,
- see that you have a *trigger* that would re-open the discussion,
- see that an accountable human signed on a specific date.

A residual-risk paragraph that says "we have taken appropriate measures and no significant risk remains" is a failure to answer. Specificity wins.

## How to publish the datasheet

For public models: publish the datasheet alongside the model card on the model's Hugging Face / artifact-registry landing page. For internal models: publish to the internal data-catalog with the DPO reader role granted by default.

For *sampled public release* (where the model is public but the dataset is not), publish a *redacted* datasheet — the section headers and the residual-risk claim should remain visible even when the row counts and composition are not. A regulator reading the model card for a shipped product should at minimum see that the datasheet exists and who signed the residual-risk claim.

## Summary — what you can do after this chapter

- Write a datasheet slice for a training corpus and a retrieval corpus that covers Gebru et al.'s seven sections at the depth a DPO / auditor / downstream consumer needs.
- Treat the datasheet as a version-controlled engineering artifact — one per dataset, one per regeneration, referenced by the model card and the ML-BOM.
- Write a residual-risk paragraph that names scenarios, compensating controls, re-opening triggers, and accountable signatory.
- Recognise when a separate datasheet is needed (fine-tune, retrieval, evaluation, red-team, synthetic).

The next chapter picks up the Group-B thread: how to make the lawful-basis / consent / purpose tags you recorded in the datasheet *travel with the data* through the pipeline — ETL, embedding, retrieval, inference, logging — so that the claims in the datasheet are enforceable rather than aspirational.
