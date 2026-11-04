# Anonymisation Operators and the PII Evaluation Harness

Detection finds the span. Anonymisation decides what the output looks like after the span is replaced. Evaluation measures whether the detector you built does what you say. This chapter covers both — because the operator choice depends on how precision / recall measured, and the eval harness is what makes any of the per-choke-point decisions defensible.

Primary references:

- Presidio Anonymizer docs — [microsoft.github.io/presidio/anonymizer](https://microsoft.github.io/presidio/anonymizer/).
- Presidio Anonymizer operators — [microsoft.github.io/presidio/anonymizer/#built-in-operators](https://microsoft.github.io/presidio/anonymizer/).
- `presidio-evaluator` — [github.com/microsoft/presidio-research](https://github.com/microsoft/presidio-research).
- NIST SP 800-38G — recommendation for format-preserving encryption. [doi.org/10.6028/NIST.SP.800-38G](https://doi.org/10.6028/NIST.SP.800-38G).
- NIST SP 800-188 — de-identifying government datasets (useful framing for FP / FN costs). [doi.org/10.6028/NIST.SP.800-188](https://doi.org/10.6028/NIST.SP.800-188).

## The anonymisation operators

Presidio's `AnonymizerEngine` ships five built-in operators plus `custom`. Each has a reversibility class that governs its use:

| Operator | Reversibility | Deterministic? | Typical use |
|----------|---------------|----------------|-------------|
| `replace` | Irreversible | If the replacement is a constant yes; if a counter, no | Training-corpus ingest; log sanitisation. Replaces the span with a `new_value` or an entity-type token like `<PERSON>`. |
| `redact` | Irreversible | Yes (always `""`) | Public document release. Removes the span entirely. |
| `mask` | Irreversible | Yes | UI rendering; "last 4" display. `mask(masking_char="*", chars_to_mask=10, from_end=False)` → `***********@example.com`. |
| `hash` | Irreversible, stable | Yes (deterministic over the input + algorithm) | Join keys; log dedup; cross-corpus linkage without plaintext. SHA-256 by default. |
| `encrypt` | Reversible under key | Deterministic (AES-ECB by default) | Vault-backed tokenisation; reversible by the key-holder only. |
| `custom` | Depends | Depends | Your own operator — e.g. vault-tokenise via a Presidio-wrapped vault call. |

Three reversibility classes you must be able to articulate:

- **Irreversible** — the raw value is gone. `redact`, `mask`, and `replace` with a non-unique replacement.
- **Irreversible-but-stable** — the raw value is gone, but equal inputs produce equal outputs. `hash` with a fixed salt; `replace` with a per-value counter persisted to the vault. Useful for joins without revealing values.
- **Reversible under key** — `encrypt`, `custom` vault-tokenise. The raw value is recoverable by a key-holder; the recoverability is the whole point.

Every operator configuration in a pipeline must have its reversibility class documented. A log sink labelled "redacted" that is actually "encrypted" with a key the vendor holds is a known failure mode.

### `replace`

```python
from presidio_anonymizer import AnonymizerEngine
from presidio_anonymizer.entities import OperatorConfig

anonymizer = AnonymizerEngine()
result = anonymizer.anonymize(
    text=text,
    analyzer_results=results,
    operators={
        "DEFAULT": OperatorConfig("replace", {"new_value": "<REDACTED>"}),
        "US_SSN": OperatorConfig("replace", {"new_value": "<SSN>"}),
        "PERSON": OperatorConfig("replace", {"new_value": "<PERSON>"}),
    },
)
```

Use for training-corpus ingest where the model should learn *that there was a token of this type here* without learning the value. Downstream retrieval / prompt-side can use the same operator with per-entity tokens so the prompt remains grammatical.

### `mask`

```python
operators={
    "EMAIL_ADDRESS": OperatorConfig(
        "mask",
        {"masking_char": "*", "chars_to_mask": 10, "from_end": False},
    ),
}
```

Useful at the output side (user sees `**********@example.com`) or at the log side (`***-**-6789`). Length-leakage is real; a masked 10-digit field tells the attacker it was a 10-digit field.

### `hash`

```python
operators={
    "EMAIL_ADDRESS": OperatorConfig("hash", {"hash_type": "sha256"}),
}
```

Deterministic over the input — the same email hashes to the same token every time. Equality-join works; reverse-engineering the value does not (within the input entropy — a hashed SSN is still only ~30 bits of entropy, feasible to brute-force by rainbow table). Use with a *secret salt* if the search space is small enough for brute-forcing; Presidio does not include a salt natively, so wrap with a `custom` operator for salted hashing.

### `encrypt`

```python
operators={
    "EMAIL_ADDRESS": OperatorConfig("encrypt", {"key": AES_KEY_HEX}),
}
```

AES by default. Reversible with the key via the companion `DeanonymizeEngine`. Use when the downstream consumer is authorised to recover the value — rare at the model boundary; common at the log-review boundary for authorised investigators. Record the KMS key in the vault policy.

### `custom` — the vault-tokenise operator

The one you will write most often. Signature:

```python
from presidio_anonymizer.operators import Operator, OperatorType

class VaultTokeniseOperator(Operator):
    def operate(self, text: str, params: dict | None = None) -> str:
        entity_type = params.get("entity_type")
        scheme = params.get("scheme", "opaque_deterministic_v3")
        return vault_client.tokenise(entity_type, text, scheme)

    def validate(self, params: dict | None = None) -> None:
        if not params or "entity_type" not in params:
            raise ValueError("entity_type required")

    def operator_name(self) -> str:
        return "vault_tokenise"

    def operator_type(self) -> OperatorType:
        return OperatorType.Anonymize
```

Register and use:

```python
from presidio_anonymizer.operators.operators_factory import OperatorsFactory
OperatorsFactory.operators.append(VaultTokeniseOperator)

operators = {
    "DEFAULT": OperatorConfig("vault_tokenise", {"entity_type": "DEFAULT"}),
    "US_SSN":  OperatorConfig("vault_tokenise", {"entity_type": "US_SSN"}),
    "EMAIL_ADDRESS": OperatorConfig("vault_tokenise", {"entity_type": "EMAIL_ADDRESS"}),
}
```

This is the bridge to mod-103 chapter 05 — the detector finds the span, the vault-tokenise operator calls the vault's `tokenise` API, and the text now carries a vault-resolvable token in place of raw PII.

## Picking the operator per choke-point

The table restates chapter 01's choke-points with recommended default operators:

| Choke-point | FN cost | FP cost | Recommended default | Reversibility class |
|-------------|---------|---------|---------------------|---------------------|
| Ingest (training corpus) | Very high | Moderate | `vault_tokenise` (custom) with deterministic scheme | Reversible under key |
| Retrieval index | High | Moderate | `vault_tokenise` (custom) with deterministic scheme | Reversible under key |
| Prompt side | High | High | `vault_tokenise` (custom) for user-owned data; `replace` with entity token for third-party | Mixed |
| Completion side (log) | Very high | Low | `hash` (irreversible-stable) for investigation-join; `replace` for pure redaction | Irreversible |
| Output side (to user) | Varies | High | Detokenise user-own; `mask` third-party | Mixed |

Three guiding rules behind the table:

- **Irreversible beats reversible at the log boundary.** A reversible operator on the log side means the log holds the key or the key-holder; both are breach surfaces. Prefer `hash` or `replace`.
- **Deterministic vs. nondeterministic depends on downstream joins.** If any job downstream of the choke-point joins on the token, the operator must be deterministic (equal-input → equal-output). If no join exists, nondeterministic is slightly more privacy-protective (per-call salt makes brute-force harder).
- **Vault-tokenise is the default at ingest + retrieval.** The vault owns the raw; the pipeline sees tokens. This is the chapter-01 belt + chapter-05-vault braces.

## The evaluation harness

A detector without measured precision / recall per entity type is an uncontrolled control. This section covers the labelled-eval harness end-to-end.

### The labelled eval set

Three sources, in rough priority order:

- **Real traffic samples, labelled.** The gold standard. Draw from the actual ingest / prompt / completion streams you are trying to defend. Label with a human-in-the-loop pass through Label Studio / Doccano / Prodigy. 500 – 2000 labelled samples is typical for a first version.
- **Public labelled sets.** Useful as supplementary / evaluation-of-recognisers-against-a-shared-baseline. Candidates:
  - **i2b2 / n2c2** clinical de-identification challenges (de-identified clinical notes with PHI spans) — [n2c2.dbmi.hms.harvard.edu](https://n2c2.dbmi.hms.harvard.edu/data-sets).
  - **Enron email corpus** — [cs.cmu.edu/~enron](https://www.cs.cmu.edu/~enron/), plus the `enron-aeslc` variant.
  - **HIPAA-labelled** synthetic clinical sets published by Presidio's team.
  - **`presidio-research`** generated synthetic datasets (per-entity templates with injected ground-truth spans).
- **Synthetic generation.** `presidio-research` ships a `data_generator` that produces labelled synthetic text per entity template. Use to widen entity coverage (your real traffic may have five `US_SSN` samples; synthetic can produce a thousand) and to probe edge cases (formats with / without dashes; prefixed / unprefixed; adjacent-to-other-PII combinations).

Blend the three. A labelled set that is 70% real-sampled and 30% synthetic (to cover rare entities) is typical.

### Ground-truth label schema

Each eval sample is a tuple:

```json
{
  "id": "eval-2025-11-04-0001",
  "text": "Hi, I'm Jane Smith. My SSN is 123-45-6789.",
  "spans": [
    {"start": 8, "end": 18, "entity_type": "PERSON"},
    {"start": 30, "end": 41, "entity_type": "US_SSN"}
  ],
  "source": "sampled_support_ticket",
  "label_pass": "2025-11-02",
  "labeller": "alice@example.com",
  "label_notes": "SSN is a decoy — sampled with consent for this corpus."
}
```

The span notation (`start`, `end`, `entity_type`) matches Presidio's `RecognizerResult`. Keep the format; the eval loop is a diff over the two span lists.

### The core evaluation loop

Primitives:

```python
from presidio_evaluator import InputSample, Span
from presidio_evaluator.evaluation import Evaluator

samples = load_labelled_eval("eval-2025-11-04.jsonl")
evaluator = Evaluator(model=analyzer)   # presidio_evaluator wraps the analyzer

results = evaluator.evaluate_all(samples)
summary = evaluator.calculate_score(results)

print(summary.pii_recall, summary.pii_precision, summary.pii_f)
for entity, score in summary.entity_recall_dict.items():
    print(entity, round(score, 3))
```

Three per-entity metrics you want:

- **Precision per entity** — of the spans the analyzer *returned* as `US_SSN`, what fraction were actually `US_SSN`?
- **Recall per entity** — of the spans *labelled* as `US_SSN`, what fraction did the analyzer return?
- **F1 per entity** — harmonic mean, for one-number rollup.

A single global F1 hides the per-entity story; the DPIA cites the per-entity table.

### The confusion matrix beyond simple present/absent

Three failure modes a precision / recall table at the span level misses:

- **Mislabelling.** The analyzer returned a span, but as `PHONE_NUMBER` when the label is `US_SSN`. Is that a FP for `PHONE_NUMBER` + FN for `US_SSN`, or a partial success? `presidio_evaluator` records both as "wrong_entity"; choose a policy and document it.
- **Partial-span match.** The label is `123-45-6789`; the analyzer returned `-45-6789`. Scoring policies: "exact" (counts as FN), "overlap" (counts as TP with a quality note), or "subset". Document the policy; the DPIA cites it.
- **Context-dependent entity type.** "John" labelled as `PERSON` in one sentence; "John" (the city) labelled as `LOCATION` elsewhere. The analyzer may return `PERSON` in both — the labelled eval must reflect the ambiguity, not only the common case.

### Thresholds and the ROC-ish picture

Score threshold is a knob between precision and recall. Produce the trade curve:

```python
thresholds = [0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9]
rows = []
for t in thresholds:
    analyzer.default_score_threshold = t
    results = evaluator.evaluate_all(samples)
    summary = evaluator.calculate_score(results)
    rows.append({
        "threshold": t,
        "precision": summary.pii_precision,
        "recall": summary.pii_recall,
        "f1": summary.pii_f,
    })
```

Plot per-entity. For a choke-point with high FN cost (ingest; log), optimise on recall subject to a precision floor (e.g. "recall >= 0.95 at precision >= 0.80"). For a choke-point with high FP cost (output side), the floor flips.

The threshold is a release-time decision, not a run-time one. Freeze it in the configuration; version it alongside the recogniser catalogue.

### Red-team samples

Separate from the labelled eval, keep a **red-team sample set** — adversarial examples designed to elicit FN:

- PII with non-standard formatting (`SSN 123 45 6789`; `s s n 123456789`; `my social is one two three …`).
- PII adjacent to lookalikes (`CASE_20250615_123456789` — looks like `CASE_ID + US_SSN` concatenation).
- PII in context (an email inside a markdown link `[contact](mailto:alice@example.com)`; a phone number in a footer).
- Multi-language PII (`Numero di sicurezza sociale 123-45-6789`).
- PII inside base64 / URL-encoded strings (debatable — your product may or may not deserialise; document the policy).
- PII right at the recogniser's score threshold.

The red-team set is not scored into the headline metric — it is a separate "known adversarial FN" dashboard. New FNs in the red-team set are treated as regressions.

### Regression gate in CI

The eval is a CI gate:

```yaml
# ci.yaml (shape)
- name: pii-regression
  run: |
    python scripts/run_eval.py \
      --recognisers config/recognisers.yaml \
      --eval-set data/eval-2025-11-04.jsonl \
      --red-team data/red-team-2025-11-04.jsonl \
      --fail-if-recall-below-per-entity 0.90 \
      --fail-if-precision-below-per-entity 0.80
```

Gate the merge on the per-entity thresholds. Treat a drop below the floor as a merge-blocker; treat a drop on the red-team as a merge-warning + an owning-team ticket.

## The latency and cost sides of the evaluation

A detector evaluated only on precision / recall is half-evaluated. Measure also:

- **Per-call latency** — p50 / p95 / p99 over a traffic sample, per entity scope and per engine configuration. `pytest-benchmark` or a custom harness; see chapter 09 for the integration-side measurement.
- **Throughput** — characters / second at a saturating load. Per engine (spaCy / Stanza / transformers).
- **Cost per million characters** — infrastructure cost (CPU / GPU hours, memory) attributed to the detector service. Compare against the managed-service per-million-chars price (chapter 05).
- **Model-load / warm-start cost** — the first call after process start. Matters for autoscaling patterns (cold starts are frequent).

Record in the same artifact as the precision / recall table. The DPIA quotes both.

## The eval artifact

The output of the harness is a single machine-readable artifact plus a Markdown report:

```yaml
# eval-2025-11-04.yaml
version: 2025-11-04
analyzer:
  presidio: 2.2.354
  nlp_engine: spacy:en_core_web_lg==3.7.0
  recognisers_catalogue_version: 2025-11-03
  score_threshold: 0.4
eval_set:
  id: labelled-eval-2025-11-04
  n_samples: 1200
  entity_distribution: {US_SSN: 240, EMAIL_ADDRESS: 360, PERSON: 480, ...}
per_entity:
  US_SSN:        {precision: 0.98, recall: 0.94, f1: 0.96, n_support: 240}
  EMAIL_ADDRESS: {precision: 0.96, recall: 0.97, f1: 0.96, n_support: 360}
  PERSON:        {precision: 0.83, recall: 0.88, f1: 0.85, n_support: 480}
red_team:
  n_samples: 150
  n_fn: 7
  n_fn_delta_vs_prev: +2
latency:
  p50_ms: 8
  p95_ms: 24
  p99_ms: 51
throughput:
  chars_per_second: 820000
cost:
  cpu_hours_per_mchar: 0.14
  usd_per_mchar: 0.004     # self-hosted at current cluster rates
owner: privacy-platform
```

This YAML is the output of CI. The DPIA cites it by version.

## Common failure modes in the eval

- **Label leakage.** The eval set is drawn from the same corpus the detector was tuned on. Measurements are optimistic. Keep a held-out eval that the tuning loop never sees.
- **Ground-truth error.** The labellers missed real PII or over-labelled lookalikes. Rotate a 10% sample through a second labeller; expect disagreement.
- **Entity-definition drift.** What counts as `PERSON`? Fictional characters? Public figures? Decide upfront; the labelling guide is the authority.
- **Distribution drift.** The eval set reflects traffic from six months ago; your product's traffic has shifted. Schedule a resample every quarter; alert when the distribution moves.
- **Test-set contamination via fine-tune.** If you fine-tuned a NER on anything that touches the eval set, the eval is contaminated. Keep fine-tune and eval disjoint; prefer time-based splits (train on pre-cutoff, eval on post-cutoff).
- **Operator confusion.** The analyzer precision / recall is measured; the anonymiser's behaviour on the matched spans is assumed correct. Separate test: assert the anonymised text no longer contains the raw PII for each sample.

## Summary — what you can do after this chapter

- Pick an anonymisation operator (redact / mask / replace / hash / encrypt / custom vault-tokenise) per choke-point by its reversibility class.
- Build a labelled PII eval set from real traffic + public corpora + synthetic generation.
- Compute per-entity precision / recall / F1 and set per-entity thresholds that trade off FP / FN per choke-point.
- Maintain a red-team sample set separate from the headline metric; gate CI on both.
- Measure latency / throughput / cost alongside precision / recall; produce one artifact the DPIA cites.
- Name six common eval failure modes and the discipline that prevents each.

The next chapter takes the harness you just built and benchmarks Presidio against the managed services — Google DLP, AWS Comprehend PII, Azure AI Content Safety, Private AI — for the build-vs-buy conversation.
