# Why PII Detection and Redaction Is the AI Data-Path Tax

mod-103 taught the privacy engineer to *route* data — which fields live in the vault, which carry a `lawful_basis`, which get tokenised before the model ever sees them. All of that assumes a prior question has an answer: *given a byte string, is there PII in it, and where*. This module is the answer to that question.

The AI data path has five choke-points where the "is this PII?" question is asked, by different callers, under different latency budgets, with different consequences if it gets the wrong answer. The module is organised around delivering a detector, an anonymiser, and an evaluation harness good enough to defend at each of those choke-points.

## Why this matters

Three concrete things stop happening if you do not own the PII pipeline.

- **Training-set memorisation goes from "possible" to "measured".** Extraction attacks (Carlini et al.) find verbatim PII in large-model outputs because the training set contained the PII verbatim. A PII-redaction step at ingest shrinks the memorisable surface by the fraction of the corpus that is PII. mod-105 depth follows the attack; this module delivers the preventive control that mod-105 then measures the residual of.
- **Retrieval leaks become cross-tenant leaks.** A vector DB indexed over un-redacted documents lets tenant B's query surface tenant A's PII as a retrieved chunk. mod-109 depth covers the ACL; the ACL depends on the PII first being *locatable*.
- **Logs become the breach.** Prompt / completion logs are the single most reliably leaky sink in an AI system — written by SREs for debuggability, retained because retention isn't anyone's full-time job, and quietly containing SSNs because someone tested with their own. mod-103's vault pattern says *write tokens, not raw PII*. The vault only gets to replace what the detector has found.

In each case, the detector is the hinge control — the thing the other controls hang off.

## Where PII detection sits in the stack

Place the module in the stack you already have:

- **Under mod-103 vault + tokenisation.** The vault substitutes a token for a raw value. The detector decides which spans are raw values in the first place. Free-text ingest pipelines are the classic example: the vault cannot tokenise `"My SSN is 123-45-6789"` — the detector has to carve out the span `123-45-6789` so the vault can replace it with `⟨ssn_tok_8f3c⟩`. Without the detector, mod-103's control applies only to columns whose schema-carrier already names them as sensitive.
- **Over mod-106 differential privacy.** DP-SGD bounds the *memorisation* of a given training example; redaction prevents the example from containing the PII in the first place. Belt and braces. DP without redaction means every training example still carries PII — and DP's privacy guarantee degrades with `epsilon` growth across many epochs on sensitive content. Redaction + DP gives you a much smaller surface for DP to protect.
- **Next to mod-105 memorisation / MIA engineering.** The two modules are the halves of the same problem: this one removes PII from the training set *a priori*; mod-105 measures residual memorisation *a posteriori*. If this module shipped FN >> 0, mod-105's canary tests will find it.
- **Feeding mod-111 DPIA.** The DPIA quotes this module's artifacts by name: the recogniser catalogue, the per-entity precision / recall on the labelled eval set, the latency / cost benchmark against managed services, the FP / FN rate on the integration eval. "We redact PII before training" is not a DPIA answer; "we run Presidio-v2.4 with our custom `RX_MRN` recogniser, measured recall 0.93 on the MRN subset of our labelled eval, FN budget documented in residual-risk section 4.2" is.

The boundary to the peer `ai-risk-engineer` (chapter 10) is about *depth*. That role's mod-107 covers Presidio integration at working depth — enough to configure an analyzer and redact a prompt. This module deepens into multimodal detection, de-identification mathematics (k / l / t, Expert Determination), and the managed-service comparison a product team actually has to defend to procurement.

## The five choke-points

A production AI data path has five places where the "is this PII?" question is asked. The module builds detectors and integration patterns for all five.

### Choke-point 1 — Ingest (before training-corpus write)

- **Callers:** ETL jobs, data-ingest services, batch snapshots.
- **Input:** documents, rows, transcripts about to land in the training / fine-tune corpus.
- **Latency budget:** seconds-to-minutes (batch); not interactive.
- **FP cost:** moderate — a training sentence becomes "⟨REDACTED⟩ went to the store". The model learns less from that sentence.
- **FN cost:** *high* — PII enters the training set, model memorises it, regurgitates it on prompts six months later.
- **Deliverable this module:** Presidio pipeline with anonymisation operators replace / hash / encrypt; tuned to prioritise recall over precision; integrated into the training-pipeline peer's ingest job (mod-103 chapter 03 propagation surface).

### Choke-point 2 — Retrieval index (before the chunk is written to the vector DB)

- **Callers:** RAG ingest job; document indexing; chunking + embedding worker.
- **Input:** chunk text about to be embedded and stored with metadata in a vector index.
- **Latency budget:** minutes (batch); seconds (incremental).
- **FP cost:** moderate — a retrieved chunk the user needs may now say "⟨REDACTED⟩ is the customer's VP of engineering". Downstream synthesis loses specifics.
- **FN cost:** high — a cross-tenant retrieval surfaces tenant A's PII to tenant B even when the ACL is intact (the index was indexed over un-redacted text; the ACL only enforces on *which rows* return, not on *what text* is in those rows).
- **Deliverable this module:** chunk-level PII pass before embed-and-write; the chunk written to the index carries tokens where sensitive values were detected; mod-109 ACL + this module is the belt + braces.

### Choke-point 3 — Prompt side (before send to model provider)

- **Callers:** inference service, orchestrator, agent runtime.
- **Input:** the composed prompt — system prompt + retrieved chunks + user input — about to be sent to the LLM.
- **Latency budget:** tens of milliseconds (synchronous).
- **FP cost:** moderate-to-high — a redacted prompt gives the model less to work with. "Email ⟨REDACTED⟩ asking for a refund" can be enough; "Call ⟨REDACTED⟩ at ⟨REDACTED⟩" is not.
- **FN cost:** high — raw PII crosses the trust boundary into the model provider. Provider's logs, caches, fine-tune queues all now contain the raw value.
- **Deliverable this module:** a latency-budget-aware detector. The naive Presidio-with-transformers pipeline is probably too slow at p99; chapter 02 covers the engine choice.

### Choke-point 4 — Completion side (before log write)

- **Callers:** inference service, logger, observability sink.
- **Input:** the completion text the model emitted; may contain PII the model generated *de novo* or regurgitated from training.
- **Latency budget:** tens of milliseconds (synchronous — the log write typically blocks response return).
- **FP cost:** low — a redacted log is still a usable log, and over-redaction of a log is cheap.
- **FN cost:** *very high* — this is the choke-point the breach report blames. Log writes are high-volume, long-retention, often multi-system (file logs + metrics + traces + eval replay + ML training feedback). A missed SSN here is the archetypal AI-product PII incident.
- **Deliverable this module:** detector configured for recall-first; destructive operators (replace / hash) rather than reversible ones, because the log itself is often the thing being retained.

### Choke-point 5 — Output side (before return to user)

- **Callers:** inference service / agent runtime / egress gateway.
- **Input:** the completion that is about to be *returned to the user*.
- **Latency budget:** tens of milliseconds (synchronous) or streaming (chunked).
- **FP cost:** high — the user sees "⟨REDACTED⟩" in their own name field. The product is unusable.
- **FN cost:** varies — depends on whether the user is the owner of the data. Returning the end-user their *own* email is fine; returning them *another* customer's email is a cross-tenant breach.
- **Deliverable this module:** context-aware redaction that distinguishes the *user's own* PII (pass-through — detokenise from vault if needed) from PII that belongs to *another* subject (redact / substitute). Hooks into mod-109's tenant model.

The five choke-points are different problems. They share a detector but not an operator, not a latency budget, not a FP / FN target, and not a failure mode. Chapter 09 lands the integration contract; chapters 02-08 build the detector and the operators you integrate.

## Invariants the module enforces

Four invariants that should be true of any PII pipeline you ship. They are the acceptance criteria of mod-104, not aspirations.

- **Every detector is evaluated on a labelled eval set, per entity type, with reported precision / recall.** A detector without a measured recall for `US_SSN` is not a controlled detector; it is a prayer. Chapter 04 is where the eval-set discipline lands. The regulator's question is not "do you run Presidio?" but "what is Presidio's recall for SSN on your traffic, and how do you know?".
- **Every operator is reversibility-tagged.** `redact` and `mask` are irreversible; `replace` and `hash` are irreversible-but-stable; `encrypt` is reversible under a key. Confusion between reversibility classes is how "we redacted it" quietly means "we tokenised it and the vendor can reverse it". Chapter 04 catalogues the operators and their reversibility.
- **Every integration is latency-budgeted and cost-accounted.** A prompt-side detector running transformer NER adds 150ms + a GPU. Either you meet the budget or you escalate to the product team. Chapter 09 is the measurement discipline.
- **Every tabular release has a defended k / l / t.** A tabular dataset released or shared without a defended quasi-identifier model is not de-identified — it is unmeasured. Chapter 07 covers how to defend the k you chose against the attacker model you assumed.

Call these out in the DPIA. If any is missing, the DPIA has a known-weak section.

## What this module is *not*

- **Not a vault module.** mod-103 chapter 05 owns the vault and tokenisation architecture. This module *feeds* the vault — the detector produces the spans the vault's `tokenise_text` call replaces.
- **Not a memorisation-attack module.** mod-105 owns MIA, LiRA, extraction, canaries. This module is the preventive control; mod-105 measures what slipped through.
- **Not a differential-privacy module.** mod-106 owns DP-SGD, accountants, DP synthetic data. Redaction and DP compose but are distinct controls.
- **Not an RAG ACL module.** mod-109 owns per-tenant retrieval ACL. This module carves the PII out of the chunk; the ACL decides which chunks return.
- **Not a machine-unlearning module.** mod-110 owns unlearning. This module prevents the PII from needing to be unlearned by not writing it to the training set in the first place.

## The artifacts this module hands downstream

By the end of the module, for any AI system you touch, you can produce:

- A **recogniser catalogue** — Presidio out-of-the-box recognisers in scope, custom recognisers you built for domain entities, each with a labelled-eval precision / recall.
- A **PII eval-set** — labelled by entity type, drawn from a sample of your real traffic + synthetic edge cases, versioned.
- A **build-vs-buy benchmark** — Presidio vs. Google DLP vs. AWS Comprehend PII vs. Azure AI Content Safety vs. Private AI on your eval set, with precision / recall, latency, cost per million characters, and vendor-lock-in notes.
- A **multimodal pipeline** — face / voice / OCR / screen-capture handlers wired into the ingest flow for whichever modalities your product handles.
- A **tabular de-identification config** — ARX project with generalisation hierarchies, chosen k / l / t, attacker model, information-loss metrics, and residual-risk statement.
- A **HIPAA de-identification decision** — Safe Harbor or Expert Determination, with the identifier-removal list or the statistical report that justifies it.
- An **integration contract** — the five choke-points named, each with the detector configuration, operator, latency budget, cost budget, FP / FN rate, and failure mode.

These are the artifacts a DPIA (mod-111) will quote, a release-assurance review (`ai-evaluation-engineer`, L35) will assemble, and a regulator will ask for by name.

## Summary — what you can do after this chapter

- State the five choke-points (ingest / retrieval / prompt / completion / output) and the different FP / FN / latency profile of each.
- Place this module in the stack: under mod-103's vault, over mod-106's DP, next to mod-105's MIA, feeding mod-111's DPIA.
- Name the four invariants: per-entity eval, operator reversibility tagging, latency / cost accounting, defended k / l / t.
- Say what this module is *not* — vault, MIA, DP, RAG ACL, unlearning.
- List the seven durable artifacts the module hands downstream.

The next chapter is the first concrete control: Presidio's analyzer / anonymizer architecture and the out-of-the-box recognisers it ships.
