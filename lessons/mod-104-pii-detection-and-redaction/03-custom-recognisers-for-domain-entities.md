# Custom Recognisers for Domain Entities

Presidio's out-of-the-box catalogue covers global and jurisdiction-defined identifiers — SSN, IBAN, NHS, Aadhaar, and so on. Every real product also has *its own* sensitive entities: the customer account number format your billing system mints, the case ID a support ticket carries, the medical-record number your hospital system uses, the internal auth token that occasionally shows up in logs. Presidio ships no recogniser for any of those. This chapter covers how to build them.

The patterns in this chapter are the single most-used code in the module. Keep them under version control; keep them unit-tested; keep each one tagged to its own entry in the DPIA.

## The four shapes of a custom recogniser

- **Pattern-based** — regex + optional context. For entities with a stable, parseable format (an MRN like `MRN-1234567`, a case ID like `CASE_20250615_8f3c`).
- **Deny-list-based** — a fixed set of values. For small, known-finite lists (internal project names; VIP employee names; executive family members whose presence in a chunk is a redaction-worthy signal).
- **NER-fine-tuned** — a wrapper over an NLP model (spaCy or transformers) that you have fine-tuned on your domain. For entities that are not pattern-identifiable (drug names in free-text; indicator phrases for religion / health / sexuality in NRP detection).
- **RemoteRecognizer** — a wrapper that delegates to a service. For entities whose detector is expensive, independently owned, or shared across platforms (a dedicated biomarker-ID micro-service; a shared `case_id` classifier maintained by the platform team).

Use pattern-based wherever the entity is parseable; fine-tune NER where it is not; remote where organisational boundaries make it right.

## Building a `PatternRecognizer`

The reference shape — one or more patterns, optional context words, optional deny-list:

```python
from presidio_analyzer import PatternRecognizer, Pattern

mrn_patterns = [
    Pattern(name="mrn_strict", regex=r"\bMRN-\d{7}\b", score=0.85),
    Pattern(name="mrn_loose",  regex=r"\bMRN[\s:-]?\d{6,8}\b", score=0.55),
]

mrn_recognizer = PatternRecognizer(
    supported_entity="MRN",
    patterns=mrn_patterns,
    context=["MRN", "medical record", "record number", "patient"],
)
```

Register it:

```python
analyzer.registry.add_recognizer(mrn_recognizer)
```

Five rules that make pattern recognisers reliable in practice:

### Rule 1 — Anchor on word boundaries and context

Unanchored patterns match substrings. `\d{9}` matches part of a phone number, part of an order ID, part of a timestamp. Anchor on `\b` and on neighbouring punctuation that is actually part of the entity's format. The context list ("MRN", "patient") lets the `ContextAwareEnhancer` boost confidence when the word is present *without* making the match depend on the word (which would cost recall when the surrounding sentence says "The record is 1234567").

### Rule 2 — Separate high-confidence and low-confidence patterns

A single pattern with one score is a coarse instrument. Two patterns — one strict (requires the prefix / dashes / fixed length) with a high score, one loose (prefix-optional / variable length) with a lower score — gives the overlap resolver and the threshold filter a usable scale. The strict pattern matches the format-correct case at 0.85; the loose pattern catches the human-typed variant at 0.55 and the context enhancer can boost it to 0.70 when "MRN" is nearby.

### Rule 3 — Validate the match with a checksum where one exists

If the entity has a checksum (Luhn for PAN, Verhoeff for Aadhaar, ISO 7064 Mod-11 for VAT, your product's HMAC prefix on customer IDs), implement it. Override the recogniser's `validate_result(pattern_text)` method:

```python
class LuhnPatternRecognizer(PatternRecognizer):
    def validate_result(self, pattern_text: str) -> bool:
        digits = [int(c) for c in pattern_text if c.isdigit()]
        checksum = 0
        for i, d in enumerate(reversed(digits)):
            if i % 2 == 1:
                d *= 2
                if d > 9:
                    d -= 9
            checksum += d
        return checksum % 10 == 0
```

The analyzer reports the validity and the overlap resolver prefers checksum-valid matches.

### Rule 4 — Record the regex's attack surface

A regex is a parser; parsers have attack surfaces. Catastrophic backtracking (`(a+)+`), unbounded quantifiers (`.*`), and patterns compiled on untrusted input are DoS surfaces. Use non-backtracking patterns, cap quantifiers, and *never* compile a regex from a user-controlled string. The `re2` / `google-re2` Python library can be dropped in where backtracking-safety matters; the Presidio Pattern `regex` field accepts any compiled pattern.

### Rule 5 — Unit-test the recogniser

For each custom recogniser, the test file should contain:

- A set of positive examples (format-correct values the recogniser must find).
- A set of negative examples (adjacent-looking strings that are *not* the entity).
- A set of context tests (the same value with and without surrounding context words; score difference must behave as expected).
- A round-trip anonymise test (the recogniser's match feeds the anonymiser and the anonymised text contains the token).

The tests are the regression bed when you bump Presidio or spaCy versions. They are also the labelled-eval slice's seed.

## Building a deny-list recogniser

Deny-list recognisers match literal values. Trivial, effective for small lists, and easy to maintain:

```python
project_codename_recognizer = PatternRecognizer(
    supported_entity="INTERNAL_PROJECT",
    deny_list=["Phoenix", "Nightjar", "Blackstar"],
    deny_list_score=0.95,
    context=["project", "codename", "internal"],
)
```

The deny-list is case-insensitive by default; you can set `global_regex_flags=re.IGNORECASE | re.MULTILINE` or pass `supported_language="en"` to constrain. Pair with a context list so a literal "Phoenix" in unrelated text does not score 0.95 — but understand that a deny-list value *does* score high even without context, which is often what you want (a VIP employee name present anywhere in training data is a redaction signal).

Three uses in practice:

- **Executive / VIP names.** The list is short, sensitive, and updated monthly.
- **Internal project codenames.** Treating them as PII prevents leakage into training data on fine-tunes that will later be externally hosted.
- **Credentials / tokens in logs.** The list is a signature set of known-sensitive-strings; a match is an incident.

## Adding a context-aware enhancer

The `ContextAwareEnhancer` (default: `LemmaContextAwareEnhancer`) bumps a recogniser's score when context words appear within a window. Two knobs:

- **Window size.** Default is 5 tokens; you can widen for long-form document text.
- **Score boost amount.** Default is +0.35; you can tune per-recogniser.

For a custom recogniser, the context is a feature of the recogniser (see `PatternRecognizer(context=[...])` above). If you need a different window or boost, subclass the enhancer:

```python
from presidio_analyzer.context_aware_enhancers import LemmaContextAwareEnhancer

custom_enhancer = LemmaContextAwareEnhancer(
    context_similarity_factor=0.35,
    min_score_with_context_similarity=0.4,
    context_prefix_count=5,
    context_suffix_count=0,
)

analyzer = AnalyzerEngine(context_aware_enhancer=custom_enhancer)
```

Two gotchas worth knowing:

- **The enhancer lemmatises tokens.** "patient" and "patients" are the same context signal. "MRN" (an acronym) is *not* lemmatised and must appear literally.
- **The window spans across sentence boundaries by default.** A context word two sentences away still boosts. For medical / legal text where each sentence is semantically dense, consider a narrower window.

## Fine-tuning an NER model for domain entities

Where the entity is not pattern-identifiable — a drug brand name, a procedure code description in free prose, a disease term — a fine-tuned NER model is the only usable approach. Two engines:

### spaCy fine-tune

- Start from `en_core_web_lg` (or `*_trf` if you can afford transformers on CPU).
- Annotate 300–1000 text spans using Prodigy / Doccano / Label Studio / label-studio / brat.
- Train a `ner` component via spaCy's `train` CLI, reserving a held-out eval set.
- Package the result as a loadable spaCy model and point Presidio's `NlpEngineProvider` at it.

Presidio's `SpacyRecognizer` picks up custom entity labels automatically; the mapping from spaCy label to Presidio entity type lives in `conf/default.yaml` under `ner_model_configuration.model_to_presidio_entity_mapping` — add your custom labels there.

### Transformer fine-tune (Hugging Face)

- Start from a strong base (`bert-base-cased`, `roberta-base`, or a domain base like `bio_ClinicalBERT` for clinical text).
- Fine-tune with Hugging Face `transformers` + `datasets` on BIO-tagged spans.
- Export as a pipeline; load via Presidio's `transformers` NLP engine configuration.

The transformer pipeline's advantages are higher recall on prose-style entities and better generalisation to unseen formats; the costs are latency (tens of ms minimum) and infrastructure (GPU preferred for throughput). The build-vs-buy conversation (chapter 05) often lands on transformer-NER for ingest and spaCy for prompt-side.

Scope note — Presidio's project offers a `presidio-research` repository ([github.com/microsoft/presidio-research](https://github.com/microsoft/presidio-research)) with data augmentation, synthetic PII generation, and fine-tune scaffolds that save weeks on the first fine-tune.

## Building a `RemoteRecognizer`

A `RemoteRecognizer` calls out to a separately-operated service:

```python
from presidio_analyzer import RemoteRecognizer, RecognizerResult

class BiomarkerIDRemoteRecognizer(RemoteRecognizer):
    def __init__(self, service_url: str, api_key: str):
        super().__init__(
            supported_entities=["BIOMARKER_ID"],
            supported_language="en",
            name="biomarker_id_remote",
            version="1.0",
        )
        self.service_url = service_url
        self.api_key = api_key

    def load(self) -> None:
        # init any clients
        pass

    def analyze(self, text, entities, nlp_artifacts=None):
        if "BIOMARKER_ID" not in entities:
            return []
        resp = call_biomarker_service(self.service_url, self.api_key, text)
        return [
            RecognizerResult(
                entity_type="BIOMARKER_ID",
                start=m["start"], end=m["end"], score=m["score"],
            )
            for m in resp["matches"]
        ]
```

Three design notes:

- **Treat the remote service as a trust boundary.** It receives the text; log accordingly. A remote recogniser whose purpose is to detect PHI must itself be in-scope for the DPIA — it is reading the PHI.
- **Rate-limit at the recogniser.** Analyzer calls can parallelise; a remote-recogniser call that fails open (returns `[]`) under load is a silent FN-inflator. Prefer fail-closed (raise) and surface the backpressure to the caller.
- **Latency budgets stack.** A prompt-side analyzer that holds three remote-recognisers now has three network hops in its budget. Keep remote recognisers off the prompt-side path unless they are strictly necessary.

## Priority and overlap resolution with custom recognisers

When a custom recogniser and a built-in recogniser both match the same span:

- The *higher-scoring* result wins.
- If scores tie, the *first-registered* recogniser wins.

You usually want your custom recogniser to *supplement* (add new entity types) rather than *override* (replace a built-in). When overriding is the intent — for instance, a custom `PERSON` recogniser trained on your product's naming conventions:

- Remove the built-in (`registry.remove_recognizer("SpacyRecognizer")` for spaCy's built-in NER wrapper).
- Register the custom under the same entity name with a sensible default score.
- Unit-test that the removed recogniser is not loaded on `analyzer.get_recognizers()`.

Silent overlap is a known-bad; the eval harness (chapter 04) catches it when the per-entity precision / recall shifts unexpectedly.

## Per-language recognisers

Many entities are language-coded:

- A `PatternRecognizer` is language-tagged via `supported_language` (default `"en"`). An English pattern on French text will not run.
- A multi-language product needs one recogniser per language or a language-agnostic regex plus per-language context words. Which you pick depends on whether the format is language-agnostic (MRN format is the same across locales; date-of-birth parsing is not).

Record in the recogniser catalogue which language each recogniser is in scope for. A deployment that processes French prompts without a French recogniser catalogue has an auditable coverage gap.

## A worked example — `CASE_ID` for a support product

Full shape of a recogniser you might ship:

```python
from presidio_analyzer import PatternRecognizer, Pattern

case_id_patterns = [
    # Format: CASE_20250615_<8-hex-chars>
    Pattern(
        name="case_id_strict",
        regex=r"\bCASE_\d{8}_[0-9a-f]{8}\b",
        score=0.9,
    ),
    # Loose: just an 8-hex-char suffix after "case"
    Pattern(
        name="case_id_loose",
        regex=r"\bcase[\s_-]?[0-9a-f]{8}\b",
        score=0.4,
    ),
]

case_id_recognizer = PatternRecognizer(
    supported_entity="CASE_ID",
    name="support_case_id_recognizer",
    patterns=case_id_patterns,
    context=["case", "ticket", "support", "SUP-", "escalation"],
    supported_language="en",
)

# Deny-list for the sub-system's "sentinel" IDs that should never be in prod:
sentinel_recognizer = PatternRecognizer(
    supported_entity="CASE_ID",
    deny_list=["CASE_SENTINEL_00000001", "CASE_TEST_00000001"],
    deny_list_score=1.0,
    context=["case", "ticket"],
)

registry.add_recognizer(case_id_recognizer)
registry.add_recognizer(sentinel_recognizer)
```

And a test suite (shape; see chapter 04's eval harness for the full expression):

```python
POSITIVE = [
    "see CASE_20250615_8f3c6e2a for details",
    "the escalation CASE_20251101_1a2b3c4d is blocked",
]
NEGATIVE = [
    "case closed",
    "case 12345",     # too short, not hex
    "case abcdefgh00",# too long
]
CONTEXT_BOOSTED = [
    ("ticket SUP-8f3c6e2a logged", 0.55),  # loose match + context
]
```

And the entry in the recogniser catalogue:

```yaml
- name: support_case_id_recognizer
  entity_type: CASE_ID
  class: PatternRecognizer
  owner: support-platform
  patterns:
    - name: case_id_strict
      score: 0.9
    - name: case_id_loose
      score: 0.4
  context: [case, ticket, support, SUP-, escalation]
  eval:
    labelled_set: support-tickets-2025-q4
    precision: 0.98
    recall: 0.91
    last_run: 2025-11-01
  residual_risk: |
    Loose pattern may match arbitrary 8-hex suffixes after "case" in English prose;
    FP acceptable because downstream operator is hash (irreversible), FN the greater risk.
```

The catalogue is the DPIA-visible artifact.

## Common failure modes

- **Pattern drift.** A format evolves (the product adds a hyphen; the vendor appends a region code). The recogniser silently misses the new values and the FN rate climbs over months. Catch with scheduled runs of the labelled-eval + a canary (synthetic-but-current samples) in CI.
- **Case sensitivity surprises.** Regex case flags default varies by Python version and by whether you pass `global_regex_flags`. Prefer explicit `re.IGNORECASE` where you want it.
- **Overlap on the same span.** Two custom recognisers (e.g. `CASE_ID` and `TICKET_ID`) both fire on a span. Resolve by score; be explicit about which one's token the anonymiser emits (this is typically the first-registered-wins fallback).
- **Model-engine upgrade regressions.** Fine-tuned spaCy model → new spaCy major version → silently different tokenisation → previously-matching context words no longer lemmatise identically. Pin NLP engine versions in `requirements.txt` and rerun the eval.
- **Lemma-case leakage.** A context word "Patient" being lemmatised to "patient" and matching "Patients" is usually what you want, but on acronyms and codenames it is wrong. The `LemmaContextAwareEnhancer` does lemmatisation; a cased-exact enhancer is appropriate for acronym / codename recognisers.
- **Deny-list drift.** A deny-list value (a VIP employee name) is spelled slightly differently in a real document ("Jane A. Smith" vs. the deny-list's "Jane Smith"). Deny-list is literal-match; pair with a `PERSON`-NER + context recogniser that catches variant spellings.

## Summary — what you can do after this chapter

- Author a `PatternRecognizer` with multiple patterns, context words, and a checksum validator where one applies.
- Author a deny-list recogniser for a small, auditable set of literal values.
- Fine-tune a spaCy or transformer NER for a domain entity, map it into Presidio's entity catalogue, and swap the NLP engine.
- Build a `RemoteRecognizer` for a service-boundary detector, with latency + rate-limit discipline.
- Resolve overlaps between custom and built-in recognisers predictably.
- Maintain a recogniser catalogue the DPIA cites.

The next chapter is the companion: once the recogniser finds the span, which anonymisation operator is applied, and how do you measure the whole thing on a labelled eval set?
