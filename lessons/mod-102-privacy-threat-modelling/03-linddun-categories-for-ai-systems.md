# The Seven LINDDUN Categories, Read for AI Systems

LINDDUN (KU Leuven DistriNet) organises privacy threats into seven categories. This chapter unpacks each category *specifically* for the AI data flow you decomposed in chapter 02, with concrete example threats per sink.

Primary source: the LINDDUN project at [linddun.org](https://linddun.org/). Read the project's current method documents before you lean on this chapter — the canonical vocabulary wins when you escalate an argument.

## The seven categories at a glance

| Letter | Category | One-sentence read for AI |
|---|---|---|
| L | **Linkability** | Two records (prompts, embeddings, log lines, outputs) that should remain unlinkable can be tied to the same subject or session. |
| I | **Identifiability** | A record believed to be pseudonymous is reducible to a person — via memorisation, inversion, or side information. |
| N | **Non-repudiation** | The subject *cannot deny* having said or done something recorded by the system when privacy would ordinarily allow them to. |
| D | **Detectability** | The *existence* of a record is itself sensitive (membership inference, presence-in-corpus leak). |
| D | **Disclosure of information** | Personal data is seen by a party that should not see it (classical confidentiality, but viewed from the subject's side). |
| U | **Unawareness (and unintervenability)** | The subject cannot form a correct mental model of the processing or cannot exercise rights (access, erasure, objection, opt-out). |
| N | **Non-compliance** | The design violates a statute, standard, or published policy irrespective of whether an attack occurs. |

Two LINDDUN conventions are worth internalising before you use the vocabulary with others:

- **Threat ≠ attack.** A LINDDUN threat is a *property of the design* that *enables* harm to the subject. Many threats never require an attacker.
- **Threats attach to DFD elements and transitions, not to abstract systems.** "The system has a linkability problem" is not a LINDDUN threat; "the semantic cache stores (session_id, embedding, response) tuples with no retention, enabling cross-session linkability for a given subject" *is*.

## Linkability (L)

**Reading.** Two or more items of interest (records, sessions, requests, embeddings, outputs) can be correlated to the same subject when the subject should remain unlinkable.

**Where it attaches in the AI DFD.**

- *Prompt / completion logs* — identifier reuse across sessions means multiple prompts can be joined to a single subject; the resulting profile exceeds what any single prompt would have revealed.
- *Semantic cache* — a cache keyed on prompt-embedding similarity can tie a new session's query to a prior session's response, revealing the earlier subject's content to the new one when the cache is cross-user.
- *Telemetry / tool-call traces* — an agent's span IDs, tool arguments, and timing frequently carry re-identifying signal across sessions.
- *Embeddings* — the embedding space is a natural linkage substrate (near-neighbour queries across sessions / tenants).
- *Retrieval corpus* — citing a specific internal document in an answer reveals that the question was about *that* document.

**Example threat.** *"Session-scoped prompt log retains `user_id` and `conversation_id` with no truncation or hashing across logical sessions, enabling an insider or downstream integrator to construct a per-user prompt profile that no single prompt would have revealed."*

**Family of controls.** Session-scoped pseudonymisation that rotates, prompt-log minimisation, cache keying that does not include stable identifiers, per-tenant cache partitioning, retention + deletion.

## Identifiability (I)

**Reading.** A record believed to be pseudonymous or de-identified can be traced back to a specific natural person.

**Where it attaches.**

- *Model weights* — the training corpus includes PII and the model memorises enough that an extraction attack recovers it (mod-105). The record is "in the weights"; the attack returns a subject.
- *Embeddings* — inversion attacks (e.g. vec2text-family research) recover enough of the source text to re-identify.
- *Prompt / completion logs* — a "redacted" log that still carries uncommon locations, dates, or free-text leaks identity.
- *Retrieval corpus* — pseudonymised documents paired with an unredacted author field, timestamp, or structural cue allow re-identification.
- *Generated outputs* — the model names a real individual because it memorised them from training, or because retrieval returned a document naming them.

**Example threat.** *"Fine-tune corpus `support_tickets_v3` was pseudonymised with hashed customer IDs; however, free-text fields contain street addresses and uncommon product-serial numbers, allowing a reader (or an extraction attack on the resulting model) to re-identify specific customers."*

**Family of controls.** k-anonymity / l-diversity on tabular release (mod-104), DP noise at train time (mod-106), redaction on free-text (mod-104), canary-based extraction testing (mod-105).

## Non-repudiation (N)

**Reading.** The system logs enough about the subject's actions that they cannot plausibly deny those actions when they ordinarily could.

**Note on direction.** In security, non-repudiation is desirable (audit trails). In privacy, it is a *threat*: the subject may have had an expectation that their action was unobservable or deniable.

**Where it attaches.**

- *Prompt logs* — the system captures and retains the exact content the subject typed (including drafts cleared before submission) in a way that binds them to it.
- *Agent telemetry* — tool-call traces document each subject action with a timestamp, identifier, and outcome.
- *Chat history replayed into the prompt* — the subject's prior utterances are preserved indefinitely and can be surfaced back to third parties.

**Example threat.** *"The abuse-review queue retains every flagged prompt with the user's authenticated account identity, timestamp, and the full completion for 2 years, leaving the user unable to deny having asked a sensitive question that they later withdrew."*

**Family of controls.** Retention minimisation, pseudonymisation at log-write time, subject-initiated redaction / deletion, policy-driven aging, consider whether anonymous / unauthenticated modes exist for sensitive use-cases.

## Detectability (D1)

**Reading.** The *existence* of a record about the subject is itself sensitive, even without seeing the content. Membership-inference attacks (mod-105) are the AI-native instance.

**Where it attaches.**

- *Model weights* — a successful MIA shows that the subject was in the training set. That fact alone can be a privacy violation (e.g. training set = "patients of hospital X").
- *Retrieval corpus* — the fact that your RAG returns a result for "John Doe" reveals that a document about John Doe exists in the corpus.
- *Prompt / completion logs* — the presence of a flagged event about a user can be sensitive even if the content is encrypted.
- *Vector index* — a near-neighbour query that returns high-similarity hits reveals existence of a close record.

**Example threat.** *"A membership-inference attack on the released `health-support-v2` fine-tune can determine with >70% accuracy whether a given individual's record was in the training set, revealing that the individual was a patient of the partner hospital."*

**Family of controls.** DP-SGD training (mod-106), confidence-score capping on API responses, canary / shadow-model evaluation (mod-105), access-controlled retrieval (never confirm / deny existence for unauthorised queries).

## Disclosure of information (D2)

**Reading.** Personal data is seen by a party that should not see it. This is the category closest to classical confidentiality, viewed from the subject's side.

**Where it attaches.**

- *Prompt / completion logs* — raw PII in logs behind weaker IAM than the application.
- *Retrieval corpus* — cross-tenant leak (RAG returns tenant B's document to tenant A).
- *Model weights* — memorised training content regurgitated in a completion.
- *Caches* — one user's cached response surfaces to another when cache keying is wrong.
- *Telemetry* — tool-call arguments logged to a vendor the data was never contracted to.
- *Backups / snapshots* — the forgotten sink; most GDPR Art. 32 incidents live here.

**Example threat.** *"Semantic-cache lookup keyed on normalised prompt-embedding similarity returns cached completions from other users whose prompts were similar enough, leaking the earlier users' response content."*

**Family of controls.** Per-tenant / per-user cache partitioning, retrieval ACL + negative tests, DP-SGD on training, redaction on log write, encryption at rest with KMS-managed keys, TEE / confidential-compute for high-value sinks (mod-108).

## Unawareness and unintervenability (U)

**Reading.** The subject cannot form a correct mental model of what is being done with their data, or cannot act on it (access, erase, rectify, object, opt-out, opt out of ADMT, withdraw consent).

**Where it attaches.**

- *Training corpora* — the subject was not meaningfully informed that their data would be used to train a model.
- *Retrieval corpora* — the subject does not know that their document is retrievable, or by whom.
- *Prompt logs* — the subject does not know prompts are retained.
- *Fine-tune pipelines* — the subject does not know that feedback / correction they provided feeds back into weights.
- *DSR / rights-handling paths* — the subject has no UI path to request erasure, access, portability, or ADMT opt-out.
- *Meaningful-information artifact* — the subject has no plain-language description of the logic involved in a decision (GDPR Art. 13/14/15, mod-101 chapter 02).

**Example threat.** *"End users of the assistant are not told that their prompts are used to fine-tune the model on a 90-day rolling window; there is no opt-out; privacy notice is silent on this processing purpose."*

**Family of controls.** Layered privacy notice (just-in-time + full), explicit purpose disclosure, DSR intake routed to every sink in the DFD (mod-110), human-intervention path for Art. 22 decisions, model-card + dataset-card transparency (mod-112).

## Non-compliance (N)

**Reading.** The design violates a statute, framework, standard, contract, or published policy irrespective of whether an attack occurs.

**Where it attaches.**

- *Any sink* carrying personal data without a documented lawful basis (GDPR Art. 6).
- *Training corpora* with special-category data without an Art. 9(2) exception (mod-101 chapter 02).
- *Cross-border transfer* edges without a GDPR Chapter V mechanism.
- *Processor / sub-processor* relationships without an Art. 28 contract.
- *Automated decisions* (Art. 22) with no meaningful-information artifact or no contestation path.
- *Retention* longer than the documented purpose justifies (Art. 5(1)(e)).
- *High-risk processing* with no DPIA on file (Art. 35) or no Annex IV technical documentation (EU AI Act, mod-101 chapter 03).

**Example threat.** *"Fine-tune training on user prompts is relied upon legitimate-interests basis (Art. 6(1)(f)) without a documented LIA, and EDPB Opinion 28/2024 criteria for AI-training on legitimate interests are not demonstrably met."*

**Family of controls.** Lawful-basis register, LIA documents, Art. 28 contract checklist, cross-border mechanism (SCC / adequacy / BCR), DPIA (mod-111), Annex IV technical documentation, consent register + revocation propagation, ADMT registry.

## A completeness matrix to run against your DFD

A practical pass: build a table with your DFD elements on one axis and the seven LINDDUN categories on the other. Fill each cell with at least one candidate threat or an explicit "N/A — because …". The matrix is the artifact chapter 04 (LINDDUN GO) will formalise, and chapter 05 (MAESTRO) will deepen. A cell that is empty without explanation is a gap.

Rough coverage expectations to sanity-check completeness:

- **Model weights** — strongly implicated by I, D1, D2 (memorisation / extraction), U (training-corpus consent); less so by L / N-rep.
- **Embeddings / retrieval index** — strongly implicated by L, I, D1, D2.
- **Prompt / completion logs** — implicated across the board — L, I, N-rep, D2, U.
- **Caches** — strongly L, D2; sometimes I.
- **Telemetry** — strongly L, N-rep, U; sometimes D2.
- **Training corpora** — strongly I, U, Non-compliance.
- **Retrieval corpora** — strongly D2, U, Non-compliance (consent-to-be-retrievable).
- **Generated outputs** — strongly I, D2, U (Art. 22 meaningful information).

You should finish the chapter feeling that the vocabulary is fluent enough to label a threat without looking up the category.

## Summary — what you can do after this chapter

- Translate each of the seven LINDDUN categories into an engineer-facing reading.
- Attach each category to the DFD elements where it most naturally arises in an AI system.
- Produce a one-sentence example threat per (element, category) cell.
- Recognise that LINDDUN threats are design properties, not attacks.

The next chapter gives you the workshop-speed workflow (LINDDUN GO) to run this against a system you have one working session to assess.
