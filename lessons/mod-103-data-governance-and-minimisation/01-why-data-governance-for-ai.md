# Why Data Governance and Minimisation Is the AI-Privacy Foundation

A privacy threat model (mod-102) and a DPIA (mod-111) are inspection artifacts; they describe what the system does to the data subject and how badly it would go if something failed. Data governance is the layer *underneath* those — it is where you control what the data is, where it is, who is allowed to see it, and under what basis it is being processed in the first place. If governance is wrong, every later control is applied on top of something it does not fully understand.

This chapter frames the module: what the four controls are (datasheet / provenance / minimisation / vault + synth + ML-BOM), why GDPR Art. 5 is the obligation anchor, and how this module's outputs stack with mod-102's threat-model bundle and mod-111's DPIA.

## Why this matters

As the AI Privacy Engineer (level 30), you are the engineer on a product team who gets asked:

- "Can we fine-tune on these tickets?" — a datasheet + legal-basis question.
- "Can we put this customer table in the RAG index?" — a minimisation + retention question.
- "We need to log prompts for safety review — is that OK?" — a lawful-basis + retention + redaction question.
- "A user requested erasure under GDPR Art. 17 — does that reach the training set / retrieval index / fine-tune / cache?" — a provenance + ML-BOM question.
- "Can we substitute synthetic patient records for this eval?" — a synthetic-data + utility / privacy audit question.

Each question has an engineering answer, not a legal one. Legal counsel sets the obligation; you build the pipeline that makes meeting the obligation *the default path*. mod-101 chapter 02 taught you to read the obligations; this module teaches you to put them in the data plane.

## The GDPR Art. 5 anchor

The most useful single text to keep in mind for this module is GDPR Art. 5(1) — the principles relating to processing of personal data:

- **(a) lawfulness, fairness, transparency** — the datasheet + published notice (chapter 02; mod-112).
- **(b) purpose limitation** — the lawful-basis tag that follows the record through the pipeline (chapter 03).
- **(c) data minimisation** — only what is adequate, relevant, and *necessary* for the purpose (chapter 04).
- **(d) accuracy** — not the primary target of this module, but DSR rectification (mod-110) reaches here.
- **(e) storage limitation** — retention is minimisation in the time dimension (chapter 04).
- **(f) integrity and confidentiality** — vault + tokenisation (chapter 05) and the security peer's surface.
- **Art. 5(2) accountability** — the ML-BOM and the datasheet are how you *demonstrate* compliance, which the regulator will quote at you (chapter 07).

Equivalent US-side obligations (CCPA / CPRA §1798.100(c) use limitation; the various state laws' "purpose and reasonably necessary" clauses) run on the same engineering surface — a tagged, documented, minimised, deletable pipeline. The EU AI Act Art. 10 data-governance obligation for high-risk systems adds specific requirements on training-set relevance, representativeness, and known errors that datasheets cover directly. See mod-101 for the obligation reading; this module delivers the controls that satisfy them.

## The four controls this module delivers

### Control 1 — Documentation (datasheet + ML-BOM)

The first time a regulator or an acquirer asks about your training corpus, the default document should already exist. Two artifacts cover this:

- **Datasheet for Datasets** (Gebru et al., 2018 / 2021) — a questionnaire describing the dataset at the level a downstream consumer needs to understand *intended use, provenance, consent basis, de-identification state, residual risk*. Chapter 02 covers the engineering slice.
- **CycloneDX ML-BOM** (OWASP CycloneDX, since v1.5 / 1.6) — a machine-readable bill-of-materials that enumerates the datasets, components, and inputs that produced a model. Chapter 07 covers the slice you need for data-provenance claims.

The datasheet is what the human auditor reads; the ML-BOM is what the pipeline enforces. You ship both.

### Control 2 — Propagation (consent + legal-basis tags)

A pipeline that collects consent at capture and then throws the consent context away three hops later is *not* complying with Art. 5(1)(b) — it just has an untestable claim that it is. The engineering shape is **tags that travel with the data**:

- A `lawful_basis` enum, a `purpose` enum, and a `consent_token` reference at the column level for tabular data and at the document / chunk level for text.
- Tags survive ETL (joins, aggregations, format changes), survive embedding (the chunk's tags attach to its vector + metadata), and survive logging (the log row carries the inferred record's tags forward where applicable).
- Downstream jobs *refuse* to operate on records whose basis does not match the job's purpose.

Chapter 03 is the engineering walkthrough.

### Control 3 — Minimisation (four dimensions)

"Minimise" is not one obligation. It is four, each with its own engineering checkpoint:

- **Collection** — what to not even ingest. The field the schema does not allow is the one you never have to delete.
- **Processing** — the minimum data in scope for the specific task. A redaction step at ingest; a projection at query time; a per-column ACL on the training set.
- **Retention** — time-boxed storage with automatic deletion. GDPR Art. 5(1)(e) and EU AI Act Art. 10 require it; most AI product logs quietly violate it.
- **Output** — the model refuses to produce content the pipeline should never emit (regurgitated PII, cross-tenant leakage, un-authorised categories). The output layer is a last-chance minimisation checkpoint.

Chapter 04 covers the four with concrete checkpoints.

### Control 4 — Substitution (vault + tokenisation + synthetic data)

Where the data *must* be processed by the model, the engineering trick is to **not give the model the raw data**. Two patterns:

- **Data privacy vault + tokenisation** (Skyflow-shape) — store sensitive fields in an isolated vault and pass reversible or format-preserving tokens to downstream systems. The model sees `email_tok_8f3c…` where production holds `alice@example.com`. Detokenisation happens at egress, inside a trust boundary that excludes the model provider. Chapter 05.
- **Synthetic data substitutes** — where you need volume (for training, evaluation, or demo) but cannot use the real data, you generate synthetic data from a model of the real distribution (CTGAN / TabDDPM for tabular; Synthea for health; LLM-generated for text). Chapter 06 covers the build *and* the audit — a synthetic set that leaks membership of the real set is not a privacy win.

These are *engineering substitutes* — they change what data gets to the model layer in the first place.

## How this module stacks with mod-102 and mod-111

Think of the three modules as a stack:

- **mod-102 (threat model)** — enumerates the privacy threats against the AI data flow and routes each to a control family.
- **mod-103 (this module)** — delivers the *data-plane* half of those controls: the datasheet, the propagated basis, the minimisation checkpoints, the vault + tokenisation, the synth substitutes, the ML-BOM.
- **mod-111 (DPIA)** — packages all of the above into the regulator-facing evidence bundle.

Where a mod-102 threat routes to a data-plane control, the control itself is delivered in this module. Where it routes to a modelling-plane control — DP (mod-106), PII detection (mod-104), MIA defence (mod-105), unlearning (mod-110) — the other module delivers it. This module is the *default layer* that is in scope for almost every AI system you review.

## What this module is *not*

- **Not a PII-detection engineering module.** mod-104 owns Presidio / DLP / Comprehend / Private AI and the labelled-eval harness. This module uses the output of that detector (the fact that a span was identified as PII) to decide tokenisation, vault routing, and retention.
- **Not a differential-privacy engineering module.** mod-106 owns DP-SGD / Opacus / TF Privacy / SmartNoise. This module decides whether DP is applicable (via datasheet + risk) and tags the training-run records in the ML-BOM.
- **Not a machine-unlearning module.** mod-110 owns SISA / approximate / LLM unlearning. This module provides the provenance that makes unlearning *addressable* in the first place — you cannot unlearn what you cannot find.
- **Not a RAG engineering module.** mod-109 owns per-tenant ACL, prompt / completion logging policy, and agent tool scoping as a depth pass. This module delivers the dataset-level and column-level provenance that mod-109's ACL design *enforces*.

Chapter 08 draws the coordination boundary with the peer `training-pipeline-engineer` / `rag-engineer` / `llm-application-developer` roles.

## The artifacts this module hands downstream

By the end of the module, for any AI system you touch, you can produce:

- A **datasheet per corpus** (training corpus; retrieval corpus; fine-tune corpus; evaluation set; red-team set — one each where applicable).
- A **propagation contract** — a schema + runtime check that says "this record cannot be used for this job unless its `lawful_basis` and `purpose` tags match".
- A **minimisation-checkpoint register** — lint / CI / release-gate / runtime-monitor coverage across collection, processing, retention, and output.
- A **vault-integration diagram** — which fields are tokenised, which are vault-held, at which trust boundaries the raw value is accessible, and who holds the detokenisation permission.
- A **synthetic-data audit report** — utility metric vs. privacy metric for any synth set you ship.
- A **CycloneDX ML-BOM** — a machine-readable `bom.json` for the model, naming its training datasets and their consent / de-identification claims.

These are the artifacts a DPIA (mod-111) consumes, that a release-assurance review (`ai-evaluation-engineer`, level 35) assembles, and that a regulator will ask for by name.

## Summary — what you can do after this chapter

- State the four controls this module delivers (documentation / propagation / minimisation / substitution) and map each to its GDPR Art. 5 anchor.
- Explain how this module's outputs stack with mod-102's threat-model bundle and mod-111's DPIA.
- Say what this module is *not* — mod-104 PII detection, mod-106 DP, mod-109 RAG, mod-110 unlearning.
- Name the six durable artifacts (datasheet, propagation contract, minimisation register, vault-integration diagram, synth-audit report, ML-BOM).

The next chapter is the first concrete control: how to author a Datasheet-for-Datasets slice for a training corpus *and* a retrieval corpus at the depth you would be comfortable publishing alongside a model release.
