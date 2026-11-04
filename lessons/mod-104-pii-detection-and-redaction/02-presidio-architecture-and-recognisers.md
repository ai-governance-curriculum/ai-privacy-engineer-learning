# Presidio Architecture and the Out-of-the-Box Recognisers

Microsoft Presidio is the open-source reference implementation of a PII detection + anonymisation pipeline for unstructured text. For the AI privacy engineer, Presidio is the default build-side baseline you benchmark everything else against — it is MIT-licensed, self-hostable, extensible with your domain recognisers, and the artifact your DPIA cites when the regulator asks what detector you run. This chapter is the engineer's reference to the Presidio architecture, the out-of-the-box recognisers, the NLP engine choice, and the shape of a production pipeline.

Primary references:

- Microsoft Presidio — project home and docs. [microsoft.github.io/presidio](https://microsoft.github.io/presidio/).
- Presidio GitHub — source + release notes. [github.com/microsoft/presidio](https://github.com/microsoft/presidio).
- Presidio Analyzer — [microsoft.github.io/presidio/analyzer](https://microsoft.github.io/presidio/analyzer/).
- Presidio Anonymizer — [microsoft.github.io/presidio/anonymizer](https://microsoft.github.io/presidio/anonymizer/).
- Presidio supported entities — [microsoft.github.io/presidio/supported_entities](https://microsoft.github.io/presidio/supported_entities/).

## Why Presidio specifically

Three reasons Presidio is the default build-side reference for an AI privacy engineer:

- **Open-source, extensible, inspectable.** The recogniser catalogue is Python; every regex, every context token, every decision rule is readable. Compare with managed services (chapter 05) whose detectors are a black box you cannot tune.
- **Entity-type coverage out of the box.** The current release ships recognisers for roughly thirty entity types spanning global + jurisdiction-specific IDs — covered below.
- **Modular NLP back-end.** Presidio does not bind to one NLP stack — spaCy, Stanza, and transformer models from Hugging Face are all first-class. You can trade latency for recall at engine-swap time rather than at code rewrite time.

Presidio is not the only open-source PII engine (`piiranha`, `scrubadub`, `commonregex`, `pii-codex`, Microsoft Purview's SDK). It *is* the one with the clearest reference architecture, the broadest ecosystem, and the active maintainer base. The build-vs-buy conversation runs Presidio vs. managed services; the open-source compare-points are relevant to cite but rarely the default build.

## The three services

Presidio is a *set of services*, not a monolithic library. You generally use two:

- **Presidio Analyzer** — the detector. Input: a text string + language + list of entities to look for. Output: a list of `RecognizerResult` records — entity type, start offset, end offset, score (0.0 – 1.0), analysis explanation.
- **Presidio Anonymizer** — the operator. Input: text + analyzer results + an `OperatorConfig` per entity type. Output: the text with the identified spans replaced according to the configured operator (chapter 04).
- **Presidio Image Redactor** — optional; wraps an OCR pass (Tesseract by default) and applies the analyzer to the recognised text, then draws boxes over the image. Chapter 06 covers multimodal.

The Analyzer and Anonymizer can run in-process (Python library import) or as HTTP services (Docker containers published by the project). For pipeline integration, the library form is lower-latency and is what you will usually use.

## The analyzer pipeline

A single `analyzer.analyze(text=..., language="en")` call does, in order:

1. **NLP analysis.** The configured NLP engine (spaCy / Stanza / transformers) runs over the text producing tokens, part-of-speech tags, named entities (`PERSON`, `LOCATION`, `ORGANIZATION`, etc.), and lemmas. This happens *once per text*; all downstream recognisers reuse the parsed artifact (`NlpArtifacts`).
2. **Recogniser dispatch.** Each recogniser whose `supported_entities` intersects the requested entity list (or `None` = all) is run against the text. Recognisers can be:
   - **Pattern-based** — one or more regex patterns + optional context words.
   - **Deny-list-based** — a fixed set of values that count as a match.
   - **NER-derived** — a wrapper over the NLP engine's entity recognition (`PERSON` → Presidio `PERSON`).
   - **Custom logic** — any class that implements `EntityRecognizer`.
3. **Context enhancement.** A `ContextAwareEnhancer` pass may adjust scores up when a nearby word is in the recogniser's context list (e.g. "SSN" near a nine-digit span increases that span's score).
4. **Overlap resolution.** Overlapping results from different recognisers are reduced per the `AnalysisExplanation.score_boost` rules — the highest-scoring result for a given span wins, ties are broken by order.
5. **Threshold filter.** Results below the `score_threshold` (default 0.35) are dropped. Returned.

The analyzer is stateless across calls; the recogniser registry is loaded once and reused. This matters for latency — the first call is slow (model load, pattern compile, registry build), subsequent calls are cheap.

## Entity types shipped out of the box

The supported-entities page is authoritative and updated each release; the current catalogue covers (approximately — check [the page](https://microsoft.github.io/presidio/supported_entities/) for the release you ship):

- **Global** — `CREDIT_CARD`, `CRYPTO`, `DATE_TIME`, `EMAIL_ADDRESS`, `IBAN_CODE`, `IP_ADDRESS`, `NRP` (nationality / religious / political group), `LOCATION`, `PERSON`, `PHONE_NUMBER`, `MEDICAL_LICENSE`, `URL`.
- **USA** — `US_BANK_NUMBER`, `US_DRIVER_LICENSE`, `US_ITIN`, `US_PASSPORT`, `US_SSN`.
- **UK** — `UK_NHS`, `UK_NINO`.
- **Spain** — `ES_NIF`, `ES_NIE`.
- **Italy** — `IT_FISCAL_CODE`, `IT_DRIVER_LICENSE`, `IT_VAT_CODE`, `IT_PASSPORT`, `IT_IDENTITY_CARD`.
- **Poland** — `PL_PESEL`.
- **Singapore** — `SG_NRIC_FIN`, `SG_UEN`.
- **Australia** — `AU_ABN`, `AU_ACN`, `AU_TFN`, `AU_MEDICARE`.
- **India** — `IN_PAN`, `IN_AADHAAR`, `IN_VEHICLE_REGISTRATION`, `IN_VOTER`, `IN_PASSPORT`.
- **Finland** — `FI_PERSONAL_IDENTITY_CODE`.
- **Korea** — `KR_RRN`.

Three things worth internalising about this catalogue:

- **It is jurisdiction-explicit.** `US_SSN`, `UK_NINO`, `AU_TFN`, `PL_PESEL` are separate recognisers, not one generic "national ID". The configuration per language / per deployment names which jurisdictions are in scope.
- **It is checksum-aware where the format has a checksum.** `US_SSN` rejects known-invalid area / group numbers; `IBAN_CODE` validates the mod-97 checksum; `IN_AADHAAR` validates the Verhoeff checksum. Checksum-valid matches score higher than pattern-only matches.
- **It is not comprehensive.** PHI identifiers (medical record numbers; patient account numbers), PCI identifiers beyond a bare `CREDIT_CARD` (track-2 data; expiry + CCV), domain-specific IDs (your product's case IDs; your customer IDs) are *not* here. Chapter 03 is where you add them.

Run the canonical first call to inspect the pipeline:

```python
from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine

analyzer = AnalyzerEngine()
anonymizer = AnonymizerEngine()

text = "Hi, I'm Jane Smith. My SSN is 123-45-6789 and my credit card is 4111-1111-1111-1111."
results = analyzer.analyze(text=text, language="en")
for r in results:
    print(r.entity_type, text[r.start:r.end], round(r.score, 2))

anonymised = anonymizer.anonymize(text=text, analyzer_results=results)
print(anonymised.text)
```

Expect output along the lines of:

```text
PERSON Jane Smith 0.85
US_SSN 123-45-6789 0.85
CREDIT_CARD 4111-1111-1111-1111 1.00
Hi, I'm <PERSON>. My SSN is <US_SSN> and my credit card is <CREDIT_CARD>.
```

Numbers will vary slightly release-to-release; the shape is stable.

## The NLP engine — spaCy, Stanza, transformers

The NLP engine decides three things:

- **Which named entities the `NER`-derived recognisers see.** Out-of-the-box, `PERSON`, `LOCATION`, `NRP`, `DATE_TIME`, `ORGANIZATION` are derived from the NLP engine's NER. If the engine does not emit `PERSON`, Presidio does not find person names. Period.
- **The language coverage.** spaCy has small / medium / large / transformer pipelines for roughly two dozen languages; Stanza has broader language coverage; Hugging Face transformer models have the best per-language recall where a fine-tuned NER model exists.
- **The latency / cost.** spaCy `en_core_web_lg` is CPU-fast (single-digit ms for a short text); a transformer NER model is GPU-preferred and adds tens of milliseconds even then. You will pick an engine per choke-point.

The engine is configured in `conf/default.yaml` or in a `NlpEngineProvider` call:

```python
from presidio_analyzer import AnalyzerEngine
from presidio_analyzer.nlp_engine import NlpEngineProvider

configuration = {
    "nlp_engine_name": "spacy",
    "models": [
        {"lang_code": "en", "model_name": "en_core_web_lg"},
    ],
}
provider = NlpEngineProvider(nlp_configuration=configuration)
nlp_engine = provider.create_engine()

analyzer = AnalyzerEngine(nlp_engine=nlp_engine, supported_languages=["en"])
```

For a transformer back-end:

```python
configuration = {
    "nlp_engine_name": "transformers",
    "models": [
        {
            "lang_code": "en",
            "model_name": {
                "spacy": "en_core_web_sm",
                "transformers": "dslim/bert-base-NER",
            },
        }
    ],
}
```

(The `spacy` entry is used for tokenisation; the transformers model does NER.)

A defensible pipeline for a product that processes medical or legal text will often end up with:

- spaCy `*_lg` for CPU, high-throughput ingest paths.
- Transformer NER (e.g. a HIPAA-fine-tuned BERT model, or a domain-specific model from Hugging Face) for recall-critical surfaces (training-corpus ingest where FN is expensive).
- A fall-through for languages the primary model does not cover.

The eval harness (chapter 04) measures the trade.

## The recogniser classes

Three base classes cover almost every recogniser you will build:

- **`PatternRecognizer`** — one or more `Pattern` objects (regex + score), optional `context` words, optional `deny_list`. The workhorse for format-defined entities (SSN, PAN, custom IDs).
- **`EntityRecognizer`** (subclassed as `NERRecognizer` / `SpacyRecognizer` / etc.) — wraps the NLP engine's NER output. The way `PERSON`, `LOCATION`, `NRP`, `DATE_TIME`, `ORGANIZATION` land in the result list.
- **`RemoteRecognizer`** — calls out to an external service and wraps the response as recogniser results. Useful for a service-boundary (e.g. you own a specialised MRN classifier as a separate micro-service).

Chapter 03 is the hands-on reference for subclassing these.

## `RecognizerRegistry` — the module-level configuration

Which recognisers are loaded, in which language, with what context-helper configuration, is controlled by the `RecognizerRegistry`:

```python
from presidio_analyzer import RecognizerRegistry, PatternRecognizer, Pattern

registry = RecognizerRegistry()
registry.load_predefined_recognizers()        # loads all out-of-the-box recognisers
registry.remove_recognizer("UsSsnRecognizer") # remove one by name

mrn_pattern = Pattern(name="mrn_strong", regex=r"\bMRN-\d{7}\b", score=0.85)
mrn_recognizer = PatternRecognizer(
    supported_entity="MRN",
    patterns=[mrn_pattern],
    context=["MRN", "medical record", "record number"],
)
registry.add_recognizer(mrn_recognizer)

analyzer = AnalyzerEngine(registry=registry)
```

Three operational notes:

- **Load predefined recognisers before adding custom ones.** The order determines which recogniser's result wins on overlap; a custom high-score recogniser should be added after the predefined set it is meant to supplement or override.
- **Remove recognisers you do not use.** Each recogniser adds latency per text. A product that does not process Polish identity numbers has no reason to load `PL_PESEL` recogniser — it adds latency without adding recall on your traffic.
- **Keep the registry configuration under version control.** The registry is the detector's *contract*; your DPIA cites it by version.

## `score`, `score_threshold`, and calibrated confidence

Each recogniser emits a `score` between 0.0 and 1.0. The scores are *not* calibrated probabilities across recognisers — a pattern-only US_SSN match emits 0.5, a checksum-valid match emits 0.85, a context-enhanced match may bump another 0.15. Different recognisers use different internal scales.

Three implications:

- **A single `score_threshold` is a coarse knob.** Setting `score_threshold=0.5` passes all pattern-matched US_SSNs (good recall) but may also pass other noisy patterns.
- **Tune per-entity if you need calibrated behaviour.** Analyzer's `analyze(entities=...)` call does not take per-entity thresholds; you handle this by post-filtering the result list by entity type + score, or by setting `default_score_threshold` per recogniser.
- **Scores are for ordering + overlap resolution, not for release-gates.** A release-gate ("fail if PII found") is a binary decision; the score is the input to it, not the output.

Chapter 04's eval harness is what measures the right threshold for your traffic.

## `decision_process` — explanation output

Presidio can return a decision explanation for every match — which recogniser fired, which pattern matched, what the context-aware boost did, what overlap resolution did. Turn it on with `return_decision_process=True`:

```python
results = analyzer.analyze(
    text=text, language="en", return_decision_process=True
)
for r in results:
    print(r.entity_type, r.analysis_explanation.to_dict())
```

The explanation is the audit artifact — the thing the DPIA quotes when the regulator asks *why* a span was flagged (or not). It is also the debugging surface when you tune recognisers. Turn it off in production for the throughput win; turn it on in the labelled-eval harness.

## Ad-hoc recognisers — passed per-call

For call-time customisation (a per-request recogniser), the analyzer accepts an `ad_hoc_recognizers` list:

```python
analyzer.analyze(
    text=text,
    language="en",
    ad_hoc_recognizers=[case_id_recognizer],
)
```

Useful for request-scoped recognisers (per-tenant deny-list; per-document custom ID formats). The standing recognisers live in the registry; the ad-hoc ones ride in on the request.

## Where Presidio ends and your code begins

Three surfaces Presidio does *not* own, that your pipeline needs:

- **The labelled eval set and the evaluation loop.** Presidio ships a `presidio-evaluator` repository ([github.com/microsoft/presidio-research](https://github.com/microsoft/presidio-research)) with harnesses, but you own the labelled eval data and the per-entity precision / recall budget. Chapter 04.
- **The integration into the AI data path.** The analyzer is a library; placing it at ingest / retrieval / prompt / completion / output is the pipeline-engineer's job. Chapter 09.
- **Multimodal.** Faces, voices, screen captures are outside Presidio's text scope. Chapter 06.

## Summary — what you can do after this chapter

- Describe the Presidio analyzer pipeline end-to-end: NLP → recognisers → context enhancement → overlap resolution → threshold.
- List the three recogniser classes (`PatternRecognizer`, `EntityRecognizer`, `RemoteRecognizer`) and when to reach for each.
- Choose an NLP engine (spaCy / Stanza / transformers) per choke-point based on latency / recall trade-offs.
- Configure a `RecognizerRegistry` with only the recognisers relevant to the deployment and understand why trimming matters.
- Interpret `score`, `score_threshold`, and the decision-process explanation; know why a single global threshold is a coarse knob.
- Place Presidio in the pipeline: what it owns (detection, operators) vs. what your code owns (eval set, integration, multimodal).

The next chapter is the hands-on reference for the custom recognisers you will need — your product's case IDs, MRNs, account numbers, internal tokens — because the out-of-the-box catalogue will not cover them.
