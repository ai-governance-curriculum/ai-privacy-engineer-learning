# Managed-Service Comparison and Build-vs-Buy

Every AI privacy engineer eventually gets the question: "why are we running Presidio in a self-hosted cluster when Google / AWS / Azure sells a PII API we could just call?" The honest answer is: *it depends on your traffic, your latency budget, your data-residency constraints, and your vendor-lock-in appetite*. This chapter is the methodology for producing an answer with numbers rather than opinions.

The four managed services most commonly in scope:

- **Google Cloud Sensitive Data Protection (DLP)** — formerly Cloud DLP. [cloud.google.com/sensitive-data-protection](https://cloud.google.com/sensitive-data-protection).
- **AWS Comprehend PII** — part of Amazon Comprehend. [docs.aws.amazon.com/comprehend/latest/dg/pii.html](https://docs.aws.amazon.com/comprehend/latest/dg/pii.html).
- **Azure AI Content Safety** (and the older Azure AI Language PII detection). [learn.microsoft.com/azure/ai-services/content-safety](https://learn.microsoft.com/azure/ai-services/content-safety/) and [learn.microsoft.com/azure/ai-services/language-service/personally-identifiable-information/overview](https://learn.microsoft.com/azure/ai-services/language-service/personally-identifiable-information/overview).
- **Private AI** — a vendor specialising in multi-language PII detection + redaction across text, documents, and audio. [private-ai.com](https://www.private-ai.com/).

Each of the three hyperscalers also surfaces a separate classification / "discovery" service (Macie, Purview, GCP Data Catalog). Those sit upstream of this discussion — they *find* corpora that contain PII; they do not do span-level redaction. Keep them in the governance toolkit (mod-103) rather than the detection toolkit (this module).

## Why the question is different for AI pipelines

PII detection existed long before LLMs. Managed services like Cloud DLP are mature, battle-tested, and widely deployed in data-warehouse and analytics contexts. Three things make the AI-pipeline question different:

- **Latency budgets at prompt / completion side are tight.** A 60ms per-call API tax at prompt-side is tolerable; at 200ms it is not. The managed service's network hop + model inference is often the budget-dominant cost.
- **PII volume at training-corpus ingest is huge.** A million-character / sec ingest pipeline run against a $1 / million-chars managed service costs $86k/day. Self-hosted Presidio costs the cluster. The arithmetic is not subtle.
- **Entity coverage is uneven across services.** Each vendor ships its own entity catalogue with its own naming and its own coverage. A product that handles IN_AADHAAR and PL_PESEL needs a per-vendor coverage matrix, not a vibe.
- **Data residency and vendor-lock-in matter at AI scale.** Sending the training corpus to a US-region managed service may violate the EU AI Act Art. 50 / GDPR Chapter V transfer rules on your data. The managed-service answer is not available in all jurisdictions.
- **Explainability / tunability matters for DPIA citations.** A managed service's recogniser is a black box. "We use Cloud DLP" is a DPIA answer at one depth; "we use Presidio with a custom MRN recogniser whose recall we measured as 0.93" is a DPIA answer at a defensible depth.

The chapter produces a matrix that captures the trade.

## The services, briefly

### Google Cloud Sensitive Data Protection (DLP)

- **Scope.** Text, images (with OCR), structured data. Per-entity detectors (`infoTypes`) and categorisers. API: `inspect`, `deidentify`, `redactImage`.
- **Entity catalogue.** ~150+ built-in infoTypes spanning global + regional. Custom infoTypes via regex, dictionary, or stored infotype.
- **Operators.** Masking, replacement, deterministic encryption (DET), format-preserving encryption (FPE with FF1), cryptographic hashing, bucketing.
- **Pricing (confirm at [cloud.google.com/sensitive-data-protection/pricing](https://cloud.google.com/sensitive-data-protection/pricing)).** Per-unit of content inspected / de-identified.
- **Latency.** Sub-second for small texts; scales with text size.
- **Data residency.** Regional endpoints in several jurisdictions.
- **DPIA footprint.** Google is a processor under GCP's DPA; include in the Art. 28 chain.

### AWS Comprehend PII

- **Scope.** Text only. Per-entity detection (`DetectPiiEntities`, synchronous) + redaction (`ContainsPiiEntities`, `StartPiiEntitiesDetectionJob`, async).
- **Entity catalogue.** US-centric by default; recent additions for UK / Canada entities. Fewer entity types than Cloud DLP out of the box. See [docs.aws.amazon.com/comprehend/latest/dg/how-pii.html](https://docs.aws.amazon.com/comprehend/latest/dg/how-pii.html) for the current list.
- **Operators.** Masking / replacement in-service; raw spans returned otherwise.
- **Pricing.** Per-character inspected (confirm at [aws.amazon.com/comprehend/pricing](https://aws.amazon.com/comprehend/pricing/)).
- **Latency.** Typical ~100ms for small texts; async for batch.
- **Data residency.** Per-region endpoints.
- **Custom entities.** Separate `CustomEntityRecognizer` capability trained on labelled data.

### Azure AI services

Two offerings:

- **Azure AI Language PII detection** — entity-level PII in text, with explicit entity catalogue and redacted output. [learn.microsoft.com/azure/ai-services/language-service/personally-identifiable-information/overview](https://learn.microsoft.com/azure/ai-services/language-service/personally-identifiable-information/overview).
- **Azure AI Content Safety** — primarily a safety / harmful-content detector, with PII redaction as one surface among several. [learn.microsoft.com/azure/ai-services/content-safety](https://learn.microsoft.com/azure/ai-services/content-safety/).

Pick based on whether PII is the whole job (Language PII) or one slice of a safety pipeline (Content Safety).

- **Scope.** Text; some document / audio surfaces in preview.
- **Entity catalogue.** Documented per language; check the Supported Entities table for the release.
- **Pricing.** Per 1,000 text records.
- **Data residency.** Azure regional endpoints.

### Private AI

- **Scope.** Text, documents, images (OCR), audio / video (transcription + redaction). Positioned as the multi-modal managed alternative.
- **Entity catalogue.** 50+ entities across 50+ languages (per vendor claim; confirm at [private-ai.com/features](https://www.private-ai.com/features)).
- **Deployment modes.** Managed API *and* self-hosted container (useful when data-residency + vendor-lock-in is a concern).
- **Operators.** Masking / redaction / synthetic-substitution.
- **Pricing.** Per-call or enterprise.

Private AI is the common "best-of-both-worlds" answer when you want a vendor-maintained detector but cannot send data out.

## A fair-benchmark harness

The first and only honest way to run build-vs-buy is to run *your* traffic through each service and measure. Three rules.

### Rule 1 — The eval set is yours, not the vendor's

Use the labelled eval set from chapter 04. A vendor marketing page citing "99% F1 on our eval" is irrelevant; your product's traffic is not the vendor's eval. Mix of:

- **Your real labelled traffic** (bulk).
- **Shared public labelled sets** (for cross-vendor comparability — i2b2 for clinical; a public Enron slice for generic English).
- **A per-vendor sample** to compare on vendor-advertised entities (if a vendor claims IN_AADHAAR, have at least 30 labelled IN_AADHAAR samples to verify the claim).

### Rule 2 — Match entity-type nomenclature

Each vendor has its own catalogue. Build a **mapping table** from your labelled-eval entity types to each vendor's closest equivalent, with "no equivalent" marked explicitly:

```yaml
# entity-mapping.yaml
US_SSN:
  presidio: US_SSN
  google_dlp: US_SOCIAL_SECURITY_NUMBER
  aws_comprehend: SSN
  azure_language: USSocialSecurityNumber
  private_ai: SSN
IN_AADHAAR:
  presidio: IN_AADHAAR
  google_dlp: INDIA_AADHAAR_INDIVIDUAL
  aws_comprehend: null       # no equivalent at benchmark date
  azure_language: null
  private_ai: AADHAAR
CASE_ID:
  presidio: CASE_ID (custom)
  google_dlp: CUSTOM_CASE_ID (stored infotype)
  aws_comprehend: custom recogniser (trained)
  azure_language: null       # custom not supported in this tier
  private_ai: custom requested
```

The mapping is itself a build-vs-buy artifact — a vendor whose catalogue does not cover your core entities is a vendor you will build around anyway.

### Rule 3 — Measure latency and cost under realistic conditions

- Call the service from the region where production will run (not the laptop in a different continent).
- Measure p50 / p95 / p99 at the traffic rate production will hit.
- Record cold-start vs. warm-pool behaviour.
- Record cost per 1M chars at the API's published pricing; include egress where applicable.
- Record batch vs. sync latency separately — async batch APIs are usually cheaper but add minutes of latency.

A harness shape:

```python
PROVIDERS = [
    PresidioProvider(engine_config=...),
    GoogleDlpProvider(project="...", info_types=[...]),
    AwsComprehendProvider(region="eu-west-1"),
    AzureLanguageProvider(endpoint="...", key="..."),
    PrivateAiProvider(endpoint="...", api_key="..."),
]

results_by_provider = {}
for p in PROVIDERS:
    rows = []
    for sample in labelled_eval:
        t0 = time.perf_counter()
        pred_spans = p.detect(sample.text, entity_mapping)
        dt = time.perf_counter() - t0
        rows.append({
            "sample_id": sample.id,
            "pred": pred_spans,
            "gold": sample.spans,
            "latency_s": dt,
        })
    results_by_provider[p.name] = rows

# scoring loop computes per-entity precision/recall for each provider,
# then latency histograms, then cost per the provider's published rates.
```

Report per-entity precision / recall per provider, side-by-side.

## The build-vs-buy matrix

The output artifact is a single matrix. Dimensions:

- **Columns:** the providers under evaluation (Presidio + N managed).
- **Rows:** criteria the organisation decides on.

A defensible template:

| Criterion | Presidio (self-host) | Google DLP | AWS Comprehend | Azure Language | Private AI (managed) | Private AI (self-host) |
|-----------|----------------------|------------|----------------|----------------|-----------------------|--------------------------|
| **F1 overall** (on your eval) | measured | measured | measured | measured | measured | measured |
| **F1 per core entity** (SSN / EMAIL / PHONE / your custom) | measured | measured | measured | measured | measured | measured |
| **Coverage gap** (% core entities with native support) | `n/N` | `n/N` | `n/N` | `n/N` | `n/N` | `n/N` |
| **Custom entity support** | Full (Python) | Partial (regex + dict + stored) | Trained custom recogniser | Limited | Vendor-coded | Vendor-coded |
| **p95 latency per 1KB text** | measured | measured | measured | measured | measured | measured |
| **p95 latency per 10KB document** | measured | measured | measured | measured | measured | measured |
| **Multimodal support** (face / voice / OCR) | None (chapter 06 integrates) | OCR + image redaction | Limited | Partial | Full | Full |
| **Deployment model** | Self-host | API only | API only | API only | API | API + self-host container |
| **Data residency options** | Wherever you host | Regional endpoints | Regional endpoints | Regional endpoints | Vendor-managed regions | Your infrastructure |
| **Vendor-lock-in** | None (MIT) | High (GCP) | High (AWS) | High (Azure) | Medium (API) | Low (self-host) |
| **Price per 1M chars** (your eval blend) | infra cost | published | published | published | published | infra + licence |
| **Processor DPA / Art. 28** | N/A | DPA in GCP contract | DPA in AWS contract | DPA in Azure contract | DPA with Private AI | N/A |
| **Audit log of detection calls** | Your logging | Cloud Audit Logs | CloudTrail | Azure Monitor | Vendor-provided | Your logging |
| **Explainability of a flagged span** | Full (recogniser catalogue + decision process) | Partial (infoType reported) | Limited | Limited | Partial | Partial |
| **Change / release cadence** | Your release cadence | Google's | AWS's | Microsoft's | Vendor's | Vendor's |
| **Who owns a false negative in an incident?** | You | Shared under DPA | Shared | Shared | Shared | You |

Every row is filled with a *measured* or *cite-able* value, not a vibe. Where measurement is not feasible, the cell is `tbd:<why>` — the gap is named.

## The decision — framework, not formula

Three shapes of answer the matrix typically points to.

### Shape A — Presidio-only

Fits when:

- Latency budgets are tight (prompt / completion side).
- Custom entities are a meaningful share of the surface.
- Data residency / vendor-lock-in is a hard constraint.
- Internal infra capacity exists to run the detector service.

Trade: you own tuning + fine-tuning + model refreshes + incident ownership on FN.

### Shape B — Managed-service-only

Fits when:

- Entity surface is heavily standard (SSN / email / phone / credit card dominate).
- You are already deep in one cloud and the managed service is natively integrated.
- Volume is low enough that per-call pricing is acceptable.
- Multi-modal is needed and the managed service covers it natively (Private AI; some Cloud DLP paths).

Trade: vendor-lock-in; explainability floor; price scales linearly with traffic.

### Shape C — Hybrid

The common real answer. Patterns:

- **Presidio for ingest + retrieval; managed for prompt / completion.** Ingest is high-volume and bounded-latency; managed per-call pricing bites. Prompt / completion is low-volume per call and benefits from a managed detector's maintenance.
- **Presidio for English; managed for the long tail of languages.** Managed services' multi-language coverage often exceeds a self-hosted Presidio's; keep Presidio for the dominant language and route others to a managed service.
- **Presidio for standard entities; a specialist vendor for domain-specific (medical / legal / voice).** Private AI's voice redaction, Compellian / De-Identification APIs for HIPAA-grade clinical text.
- **Presidio as the "shadow" detector behind a managed service.** The managed service is primary; Presidio runs in parallel and the two results are logged; divergence is alerted. Useful when migrating in either direction.

The hybrid is an *engineering* answer; the matrix justifies it.

## DPIA and Art. 28 hooks

Any managed service in the chain is a processor. The build-vs-buy decision propagates into the DPIA:

- **Processor identity.** The vendor name, service version, region of the endpoint, DPA reference.
- **Processing purpose.** "Detecting and redacting PII in training-corpus ingest" is a different purpose from "detecting PII at prompt side for anonymisation before model call". List each.
- **Data categories in scope.** The managed service sees the raw text it is asked to inspect. If that text includes special-category data (health, biometric, political opinion), GDPR Art. 9 applies and the lawful-basis bar is higher.
- **Sub-processor chain.** The managed-service vendor's sub-processors. Vendors publish these; the DPIA references the current list.
- **Transfer mechanism.** If the endpoint is cross-border (EU data → US region), SCCs or equivalent must be in place. The vendor's DPA usually covers this; the DPIA documents it.
- **Retention.** How long does the vendor retain the submitted text? Most state "not retained for training" and offer zero-retention modes; confirm per-vendor.
- **Access controls.** Which internal roles can call the service; which can see the audit logs; which can download results.

If the managed service does not permit the required disclosures, that is itself the build-vs-buy signal.

## Common mistakes in the comparison

- **Comparing default detection against default detection.** Presidio's defaults are tuned for a general corpus; the managed service's are tuned for theirs. Compare with the same recogniser catalogue surface — i.e., the Presidio side has the custom recognisers you built for your product; the managed side has the custom detectors you trained on the vendor's platform.
- **Benchmarking from a laptop in a different region.** p95 across a trans-Pacific hop is uncalibrated.
- **Omitting cold-start / warm-pool.** Serverless architectures hit the managed service cold; the numbers double or triple. Bench both.
- **Confusing precision / recall over a document with per-entity precision / recall.** A document with 50 labelled spans and the detector returning 48 TP, 3 FP, 5 FN is a very different story at per-entity granularity.
- **Treating the managed service as a magical black box.** It is a model; it has FP / FN; it drifts. You measure, you monitor, you own the integration FN rate — the vendor owns the detection *technology*.
- **Ignoring the "detection" vs. "redaction" boundary.** Some services report spans only (you still apply operators); some apply redaction server-side (you cannot audit the pre-redaction spans). Prefer the former; the latter is harder to evaluate.

## Summary — what you can do after this chapter

- Describe the entity-coverage, latency, pricing, and deployment profiles of Google DLP, AWS Comprehend PII, Azure AI Language PII / Content Safety, and Private AI.
- Build an entity-mapping table from your labelled-eval entity types to each vendor's closest equivalents.
- Run a fair-benchmark harness that measures per-entity precision / recall, p95 latency, and cost on *your* traffic for each candidate.
- Produce a build-vs-buy matrix with measured cells and name the gaps that remain.
- Pick Shape A / B / C (Presidio-only / managed-only / hybrid) with the matrix as the defence.
- Translate the choice into DPIA / Art. 28 documentation entries.

The next chapter extends the pipeline to the modalities the text-only services do not cover — faces in images, voices in audio, PII in screen captures and OCR'd PDFs.
