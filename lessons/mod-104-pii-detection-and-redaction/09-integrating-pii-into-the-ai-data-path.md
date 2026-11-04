# Integrating PII Controls into the AI Data Path

Chapters 02-08 built the detector, the operators, the eval harness, the managed-service comparison, the multimodal extensions, the tabular de-identification, and the HIPAA framing. This chapter wires the whole thing into a real AI pipeline at the five choke-points from chapter 01, with concrete call-sites, latency budgets, FP / FN accounting, and the integration contract the DPIA references.

## The choke-point contract — one row per integration

Produce one of these per choke-point, per service. The DPIA quotes this verbatim.

```yaml
# integration-contract.yaml (choke-point row)
choke_point: prompt_side
service: inference-orchestrator
location: before llm.invoke(prompt)
detector:
  engine: presidio-2.2.354
  nlp: spacy:en_core_web_lg==3.7.0
  recognisers_catalogue: product-recognisers-v8.2
  languages: [en, fr]
  ad_hoc_recognisers: [per-request customer-id deny-list]
  entities_in_scope:
    - PERSON
    - EMAIL_ADDRESS
    - PHONE_NUMBER
    - US_SSN
    - CASE_ID  # custom
    - MRN      # custom
  score_threshold: 0.4
operator:
  default: vault_tokenise (deterministic)
  US_SSN: replace with "<SSN>"       # not vault-stored — destructive
  CASE_ID: vault_tokenise
  PERSON: vault_tokenise
  EMAIL_ADDRESS: vault_tokenise
latency_budget:
  p50_ms: 20
  p95_ms: 50
  p99_ms: 100
  action_on_breach: alert; shadow-mode fallthrough after 3 consecutive breaches
measurement:
  eval_set: labelled-prompt-2025-11-04
  per_entity_precision_floor: 0.80
  per_entity_recall_floor: 0.95
  red_team_regression_gate: no new FN vs. prev release
owner: inference-platform (code); privacy-engineer (config)
dpia_section: 4.2.1
```

The contract names everything a reviewer needs: *what fires, where, with what budget, to what effect, and how is it measured*. If a choke-point does not have one, it is an uncontrolled integration.

## Choke-point 1 — Ingest (training-corpus write)

### Shape

An ETL job (Spark / Beam / Dagster / your workflow of choice) reads source documents, normalises them, and writes the training snapshot. Insert the PII pass between parse and write:

```python
for doc in source_stream:
    text = doc["text"]
    pii_results = analyzer.analyze(text=text, language=doc["lang"])
    redacted = anonymizer.anonymize(
        text=text, analyzer_results=pii_results, operators=INGEST_OPS,
    ).text
    writer.write({
        **doc,
        "text": redacted,
        "pii_count": len(pii_results),
        "pii_entities": sorted({r.entity_type for r in pii_results}),
        "pii_detector_version": detector_version_string,
    })
```

### Budget and configuration

- **Latency per doc:** seconds OK; not interactive.
- **Throughput:** 100k+ docs/hour typical ingest. Scale horizontally.
- **Engine choice:** CPU-affordable (spaCy `en_core_web_lg`) unless the FN budget demands transformers.
- **Operator:** `vault_tokenise` (deterministic) by default; `replace` for destructively-redacted entities (SSN into a training set that never needs reversal).
- **Threshold:** recall-biased. Set `score_threshold` low enough to err on redaction.

### What to measure

- **Per-entity FN on a labelled slice** of the training corpus. The regulator asks "what fraction of training-set SSNs did you miss?".
- **Pipeline throughput and cost-per-million-chars.** The build-vs-buy numbers from chapter 05.
- **Downstream join integrity.** Deterministic tokens enable joins; validate the tokeniser is deterministic (CI test).

### Failure modes

- **Drift on new source.** A new ingest source has a format the recognisers were not tuned on. CI gate: fail if the per-source PII-detection rate deviates > 3σ from its historical baseline.
- **Reprocessing races.** A reprocess of the ingest with a new detector version writes the same doc with different tokens. Version the detector and the tokens; pin one version per training-snapshot lineage.
- **Vault outage.** Vault-tokenise calls fail; options: fail the job (recommended — the training-set write is non-urgent), buffer-and-retry, or fallback to destructive redaction (recorded in the datasheet).

## Choke-point 2 — Retrieval index write

### Shape

A chunking + embedding worker reads documents, chunks them, embeds each chunk, and writes chunks + embeddings + metadata to the vector index. Insert the PII pass between chunk and embed:

```python
for chunk in chunker(doc):
    pii_results = analyzer.analyze(text=chunk.text, language=doc["lang"])
    redacted = anonymizer.anonymize(
        text=chunk.text, analyzer_results=pii_results, operators=RETRIEVAL_OPS,
    ).text
    vector = embedder.embed(redacted)
    index.upsert({
        "id": chunk.id,
        "text": redacted,
        "vector": vector,
        "metadata": {
            **chunk.metadata,
            "pii_entities": sorted({r.entity_type for r in pii_results}),
            "detector_version": detector_version_string,
        },
    })
```

### Budget and configuration

- **Latency per chunk:** 50-200ms. Not interactive but higher throughput than ingest.
- **Engine choice:** spaCy; transformer for the ingest-side case where recall matters.
- **Operator:** `vault_tokenise` by default; the retrieved chunk at query time will carry tokens. Detokenisation (chapter 05) happens selectively at render time per mod-109's tenant policy.
- **Threshold:** balanced; FP in a retrieved chunk is annoying, FN is a cross-tenant leak risk.

### What to measure

- **Per-entity recall on the chunk eval.** Chunks are shorter than documents; recognisers behave differently. A chunk-specific labelled eval is a must.
- **Chunk-to-chunk consistency.** Two near-duplicate chunks produce the same token (deterministic). Test.
- **Retrieval-quality side-effect.** Does the redaction degrade the retrieved chunks' task-usefulness? A sampled A/B on retrieval recall @ k with / without redaction on an eval of queries.

### Failure modes

- **Semantic loss.** Over-redaction erodes retrieval quality; the user asks "what's Jane's case status?" and nothing retrieves because every Jane became `<PERSON>`. Mitigation: deterministic tokens preserve equality; retrieval on the token works.
- **Index drift on detector upgrade.** Re-indexing after a detector-version bump is a big job. Maintain a `detector_version` in the metadata; backfill in batches; freeze old entries if backfill is impractical and record in the datasheet.
- **Cross-tenant leak.** mod-109's ACL layer is the primary tenant-scoping control; this choke-point is the belt. Residual: a tenant's own data in a retrieved chunk should not be a token — chapter 05 covers the detokenise-at-render policy.

## Choke-point 3 — Prompt side

### Shape

The orchestrator composes a prompt from system prompt + retrieved chunks + user input, then invokes the model. Insert the PII pass just before the invocation:

```python
def invoke_model(system_prompt, retrieved_chunks, user_input):
    prompt = compose(system_prompt, retrieved_chunks, user_input)
    results = analyzer.analyze(text=prompt, language=request.lang)
    prompt_safe = anonymizer.anonymize(
        text=prompt, analyzer_results=results, operators=PROMPT_OPS,
    ).text
    return llm.invoke(prompt_safe)
```

Three composition decisions to make up-front:

- **User's own vs. other data.** The user asks "forward my email alice@example.com to finance". Redacting `alice@example.com` makes the model unable to perform. The detector cannot distinguish by itself; the orchestrator provides context — the user's own field values are exempt from tokenisation (or the tokens are detokenised in the composed prompt for this request).
- **Retrieved chunks.** Already tokenised at choke-point 2. Sanity-check they still contain only tokens, not raw PII from a tier that bypassed the index pipeline.
- **System prompt.** Usually fixed and non-PII; occasionally contains per-tenant configuration that is PII-adjacent. Audit separately.

### Budget and configuration

- **Latency:** tens of milliseconds. The detector is part of the p95 of the invocation latency.
- **Engine choice:** spaCy, strictly. Transformer NER at prompt-side is a p99 cliff.
- **Operator:** mix — `vault_tokenise` for third-party data; pass-through / detokenise for user-own; `replace` for destructive entities.
- **Threshold:** biased toward precision to avoid harming legitimate prompts.

### What to measure

- **p50 / p95 / p99 overhead** added to invocation latency. Report as a histogram; the product team will care.
- **Precision floor on prompt FP** — the "user's own name became `<PERSON>`" rate. Low-FP is a product-quality requirement here.
- **Cross-tenant PII in prompts** — a counter; non-zero is an incident.

### Failure modes

- **Streaming / partial inputs.** Streaming user input to the model (OpenAI stream endpoint) does not have the full prompt until the stream closes. Detector must either buffer-to-boundary or operate on tokens as they arrive.
- **Multi-turn context accumulation.** Conversation turns N-1 and N-2 are in the current prompt. PII redacted in the current turn may be visible in prior turns. Decide: redact-on-each-turn-rewrite, or only on new content. Record the choice.
- **Model-provider prompt caching.** OpenAI / Anthropic prompt caching retains prefix hashes; your tokenised prompt prefix becomes a cache key held by the vendor. Not raw PII, but a sinkable trace — document.

## Choke-point 4 — Completion side (log sink)

### Shape

The model returns a completion. Before any log / trace / metric sink writes the completion, run the PII pass:

```python
completion = llm.invoke(prompt_safe)
results = analyzer.analyze(text=completion, language=request.lang)
completion_safe = anonymizer.anonymize(
    text=completion, analyzer_results=results, operators=LOG_OPS,
).text

log.info("completion", completion=completion_safe, request_id=req.id, ...)
tracer.span("llm_completion", attributes={"completion.text": completion_safe})
```

The completion may be reused — returned to the user, stored in a trace, fed back into context, added to the eval replay queue. The log pass is the strictest; other reuses may detokenise selectively.

### Budget and configuration

- **Latency:** tens of ms; synchronous (the log write typically blocks response return).
- **Engine choice:** spaCy; transformer only if the log budget is lenient.
- **Operator:** *irreversible* preferred — `replace`, `hash`. The log is retained; a reversible token on the log side means the vault holds the raw for the life of the log.
- **Threshold:** recall-biased — a missed PII in a log is a durable leak.

### What to measure

- **FN rate on a sampled log-stream labelled eval.** A sampled 1% of log writes labelled by a human-in-the-loop; alert on FN.
- **Log-side PII counter.** Count of entities per entity type per hour; alert on step-changes.

### Failure modes

- **Hallucinated PII.** The model emits a plausible-looking but not-real SSN. The detector flags it; the log replaces it. Correct behaviour; count it as a TP even though no real person was harmed.
- **Regurgitated PII from training.** The model emits a real training-set SSN. The detector flags it; log is safe. The *fact* that the model regurgitated is a mod-105 memorisation incident; alert separately.
- **Log sinks that bypass the pass.** Debug logs; slow-query logs; trace samplers. Every log sink is in scope; CI test: grep the repo for direct `completion` log writes that skip the pass.

## Choke-point 5 — Output side (to user)

### Shape

The completion is about to be returned to the user. For a well-engineered pipeline, most PII in the completion was already tokenised in the prompt and the model produced tokens. The output-side pass is:

1. Detokenise user-owned tokens (the user is authorised for their own data).
2. Redact / substitute third-party tokens.
3. Run a final PII pass to catch any PII the model generated *de novo* that was not tokenised in the prompt.

```python
# After choke-point 4's log pass has already fired on completion_safe:
def finalise_for_user(completion_safe, user_id):
    detokenised = vault.detokenise_authorised(
        completion_safe, authorised_subject=user_id
    )
    results = analyzer.analyze(text=detokenised, language=request.lang)
    final = anonymizer.anonymize(
        text=detokenised, analyzer_results=results, operators=OUTPUT_OPS,
    ).text
    return final
```

### Budget and configuration

- **Latency:** tens of ms, streaming-sensitive.
- **Engine choice:** spaCy.
- **Operator:** mix — detokenise user-own; `mask` third-party for display; preserve legitimate content.
- **Threshold:** biased toward *precision* — user sees redaction, must be legitimate.

### What to measure

- **User-complaint rate on over-redaction.** A rare but telling product-quality metric.
- **Third-party PII leak rate.** FN here is a cross-tenant breach. Sampled QA.

### Failure modes

- **Streaming vs. finalise race.** A streamed response is emitted token-by-token; the full completion is not available until the stream closes. Redact as the stream closes, or buffer-to-sentence for mid-stream redaction (with latency penalty), or stream un-redacted and depend on upstream choke-points.
- **Markdown / HTML rendering.** The user's client renders the response. A redacted span inside a link's `href` is a known failure (attributes are not always analyzed). Analyze the full output including attribute values.

## Measuring FP / FN across the five choke-points

Report the full table per release:

```yaml
# pii-pipeline-eval-2025-11-04.yaml
release: v2025.11
choke_points:
  ingest:
    per_entity_recall: {US_SSN: 0.97, EMAIL: 0.98, MRN: 0.94, ...}
    per_entity_precision: {US_SSN: 0.95, EMAIL: 0.99, MRN: 0.91, ...}
    throughput_chars_per_sec: 820000
    cost_usd_per_mchar: 0.004
  retrieval:
    per_entity_recall: {US_SSN: 0.97, EMAIL: 0.98, ...}
    per_entity_precision: {US_SSN: 0.96, EMAIL: 0.99, ...}
    retrieval_quality_impact: -0.01 recall@k on eval
  prompt:
    per_entity_recall: {US_SSN: 0.95, EMAIL: 0.97, ...}
    per_entity_precision: {US_SSN: 0.92, EMAIL: 0.98, ...}
    p95_overhead_ms: 42
  completion_log:
    per_entity_recall: {US_SSN: 0.98, EMAIL: 0.98, ...}
    per_entity_precision: {US_SSN: 0.93, EMAIL: 0.98, ...}
    p95_overhead_ms: 38
    hallucinated_pii_rate: 0.002
  output:
    per_entity_recall: {US_SSN: 0.94, EMAIL: 0.97, ...}
    user_complaint_over_redact_rate: 0.0003
red_team:
  prior_release_fn: 7
  current_release_fn: 5     # improvement
```

This is the single artifact that answers "how well does your PII pipeline work?". The DPIA cites it.

## Cost accounting

Three cost categories to attribute:

- **Detector infrastructure.** CPUs / GPUs running the detector service. Attribute to the "privacy pipeline" cost centre; break down per choke-point.
- **Managed-service spend.** Dollars / month at a managed service in the hybrid pattern.
- **Vault + tokenisation cost.** Vault compute + KMS calls per tokenise / detokenise + audit storage.

Pipeline-wide cost per million chars processed, per choke-point, is the headline number for the product-finance conversation. "PII compliance costs $X per 1M user prompts" is the sentence the CFO is asking for.

## CI / runtime enforcement

The contract is enforced at three layers:

- **CI lint.** Grep the orchestrator / ingest / logger code for direct writes that skip the pass. Fail the build on a bypass. Hook into the mod-103 chapter 04 lint set.
- **CI eval gate.** Per-entity recall / precision floors; fail if below. Red-team FN delta gate.
- **Runtime monitors.** Counters per choke-point (entities detected per minute; FN rate on sampled traffic; latency p95). Alerts on step-change.

## Incident playbook

For each failure mode, a runbook entry:

- **"Detector service is down."** Prompt-side: fail-closed (reject requests); fail-open is a leak — choose by product. Ingest / retrieval: pause the job. Log side: refuse to log.
- **"FN on a rare entity found in production log."** Confirm; file ticket; patch recogniser + add red-team sample; retroactive scrub of logs that match the pattern.
- **"Model regurgitated a training-set PII."** This is a mod-105 incident. Pull the log; identify the training-set origin; consult unlearning (mod-110); DPIA residual-risk update.
- **"Vault detokenise rate anomaly."** A legitimate detokenise role is pulling more than baseline; investigate.
- **"User complaint on over-redaction."** Confirm with a repro; evaluate per-entity precision drop; consider a per-tenant-override deny-list for this entity (and record the override in the mod-103 datasheet).

## Architecture — one deployment shape

A reference deployment:

```text
                 ingest worker  ─┐
                 chunker worker ─┤
 inference orch. ──prompt──────┐ │
                 ──completion─┐│ │
                              ▼▼▼▼
                      PII detector service
                   (spaCy + Presidio library,
                    autoscaled; stateless pods)
                              │
            ┌─────────────────┼────────────────┐
            ▼                 ▼                ▼
       vault API         managed DLP      log / metrics
       (chapter 05)      (hybrid path)    (sanitised sink)
```

Three deployment properties worth calling out:

- **The detector is a stateless pod.** Horizontal scaling; pin the registry + NLP version per deployment.
- **The detector is *owned* by the privacy platform team.** Peer `ai-infra-mlops` operates the deployment; the privacy engineer owns the configuration and the eval.
- **The library form vs. service form choice is per-callsite.** Ingest can import the library; prompt-side benefits from the service to avoid library pin coupling with the orchestrator.

## Summary — what you can do after this chapter

- Author a per-choke-point integration contract with detector / operator / latency budget / measurement discipline.
- Place the detector at ingest / retrieval / prompt / completion / output with the right engine, operator, and threshold per choke-point.
- Measure per-entity precision / recall per choke-point and produce the release artifact the DPIA cites.
- Attribute cost per choke-point and defend the privacy pipeline's cost to the product-finance conversation.
- Enforce the integration contract via CI lint, CI eval gate, and runtime monitors.
- Operate a runbook for the five common failure modes.

The last chapter closes the module: the coordination boundary with the peer `ai-risk-engineer` (level 25), whose mod-107 covers this surface at working depth.
