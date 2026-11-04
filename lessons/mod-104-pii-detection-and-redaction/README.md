# mod-104-pii-detection-and-redaction: PII / PHI / PCI Detection, Classification, and Redaction Pipelines at Production Scale

**Estimated effort:** 16 hours

This module is where the privacy engineer owns the *detector*. mod-103 produced the policies — what categories exist, what the lawful basis is, where tokens flow. This module produces the *mechanism* that labels a span as "that is an SSN" or a PDF page as "that corner has PHI" or a tabular release as "this combination of ZIP + DOB + gender re-identifies 97% of rows". You will build and tune the detector on unstructured text with Microsoft Presidio, benchmark it against the managed services your platform team will inevitably ask about (Google Cloud DLP, AWS Comprehend PII, Azure AI Content Safety, Private AI), extend the pipeline to multimodal PII (faces / voices / OCR / screen captures), ship a tabular de-identification pipeline with ARX (k-anonymity, l-diversity, t-closeness), work out when HHS OCR Safe Harbor is enough and when Expert Determination is required, and integrate the whole surface into the AI data path at ingest / retrieval / prompt / completion / output. The output is a PII pipeline you can defend in a DPIA with measured FP / FN / latency / cost numbers per entity type.

## Learning objectives

- Ship a production PII detection + redaction pipeline over unstructured text with **Microsoft Presidio** — out-of-the-box recognisers, custom pattern / NER recognisers for domain entities, anonymisation operators (redact / mask / replace / encrypt / hash), evaluation on a labelled PII eval-set.
- Benchmark the Presidio pipeline against **managed services** — Google Cloud Sensitive Data Protection (DLP), AWS Comprehend PII, Azure AI Content Safety, Private AI — on precision / recall per entity type, latency, cost, and vendor-lock-in; produce a defensible build-vs-buy matrix.
- Extend the pipeline to **multimodal PII** — face / voice / license-plate in images and audio, screen-capture PHI in UI screenshots, OCR'd PDFs — using appropriate detectors (face-detection + blur, speaker-diarisation + voice-redaction, OCR + PII overlay).
- Ship a **tabular de-identification pipeline** — k-anonymity, l-diversity, t-closeness — with the ARX toolkit and sdcMicro as a compare-point; defend the chosen k / l / t against the dataset's quasi-identifier distribution.
- Apply **HHS OCR Safe Harbor and Expert Determination** methods for PHI de-identification — know when Safe Harbor's 18-identifier list is sufficient and when Expert Determination (statistical disclosure control) is required.
- Integrate the PII pipeline into the **AI data path** — ingest-side (before training-corpus write), retrieval-side (before index), prompt-side (before send to model), completion-side (before log), and output-side (before return to user); produce FP / FN measurement with cost and latency accounting.
- Cite the **boundary to `ai-risk-engineer`** (peer, level 25) — that role's mod-107 covers Presidio integration at working depth; this role deepens with multimodal detection, de-identification mathematics, and managed-service comparison.

## Chapters

1. [Why PII detection and redaction is the AI data-path tax](./01-why-pii-detection-and-redaction-is-the-data-path-tax.md) — motivation, the five choke-points, how detection sits under mod-103's vault and over mod-106's DP.
2. [Presidio architecture and the out-of-the-box recognisers](./02-presidio-architecture-and-recognisers.md) — analyzer / anonymizer / NLP engine, the Pattern / NER recogniser classes, the spaCy / transformers back-ends.
3. [Custom recognisers for domain entities](./03-custom-recognisers-for-domain-entities.md) — pattern-based, deny-list, context enhancement, and NER-fine-tuned recognisers for the entities your product has that Presidio ships no detector for.
4. [Anonymisation operators and the PII evaluation harness](./04-anonymisation-operators-and-pii-eval-harness.md) — redact / mask / replace / encrypt / hash / FPE, the per-entity precision / recall confusion matrix, and the labelled eval-set discipline.
5. [Managed-service comparison and build-vs-buy](./05-managed-service-comparison-and-build-vs-buy.md) — Google Cloud DLP, AWS Comprehend PII, Azure AI Content Safety, Private AI; a defensible benchmark harness and matrix.
6. [Multimodal PII detection](./06-multimodal-pii-detection.md) — face / voice / license-plate / OCR'd PDFs / UI screen captures; the detectors, the pipelines, the failure modes.
7. [Tabular de-identification — k-anonymity, l-diversity, t-closeness with ARX](./07-tabular-deidentification-with-arx.md) — quasi-identifiers, generalisation hierarchies, information-loss metrics; ARX and sdcMicro as compare-points.
8. [HIPAA Safe Harbor vs. Expert Determination](./08-hipaa-safe-harbor-vs-expert-determination.md) — HHS OCR's 18-identifier list, the Expert Determination method, statistical disclosure control, and when each applies.
9. [Integrating PII controls into the AI data path](./09-integrating-pii-into-the-ai-data-path.md) — the five choke-points (ingest / retrieval / prompt / completion / output), FP / FN measurement, latency and cost accounting.
10. [The boundary to `ai-risk-engineer`](./10-boundary-to-ai-risk-engineer.md) — depth vs. breadth on the shared PII surface; the sentence you say in a release review.

## Exercises

Each exercise hands you one durable artifact. Together they are a reusable PII-pipeline bundle you will carry into mod-105 (memorisation / MIA) and mod-111 (DPIA).

1. [Presidio pipeline and custom recognisers](./exercises/exercise-01-presidio-pipeline-and-custom-recognisers.md)
2. [Managed-service benchmark and build-vs-buy](./exercises/exercise-02-managed-service-benchmark-and-build-vs-buy.md)
3. [Multimodal PII detection pipeline](./exercises/exercise-03-multimodal-pii-detection-pipeline.md)
4. [k-anonymity / l-diversity / t-closeness with ARX](./exercises/exercise-04-k-anonymity-l-diversity-t-closeness-with-arx.md)
5. [HIPAA Safe Harbor vs. Expert Determination drill](./exercises/exercise-05-hipaa-safe-harbor-vs-expert-determination-drill.md)
6. [End-to-end AI PII integration contract](./exercises/exercise-06-end-to-end-ai-pii-integration-contract.md)

## Structure

- `01-…md` … `10-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
