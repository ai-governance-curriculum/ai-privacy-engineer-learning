# Why Privacy Threat Modelling for AI Needs Its Own Shape

Security threat modelling (STRIDE / attack trees / MITRE ATT&CK) is well-rehearsed in the industry. Privacy threat modelling is a *different* discipline — and when the system is an AI/ML pipeline, the shape of the data flow is new enough that even privacy-literate teams have to rebuild their reflexes.

This chapter sets up the rest of the module: what a privacy threat model *is*, why STRIDE won't do, and why the AI data flow forces the methodology to adapt.

## Why this matters

As an AI Privacy Engineer (level 30), you are the person a product team comes to with "we're about to ship a RAG assistant / a fine-tune / an agent / a tabular classifier on customer data — can you threat-model this?" The output of that session is **not**:

- a security architecture review (that's the Security peer),
- a broader harm analysis (that's `ai-risk-engineer`, level 25 — see chapter 09),
- a legal opinion (that's the DPO / counsel — chapter 08 of mod-101).

The output *is*:

- a privacy-specific data-flow diagram of the AI system with the privacy-relevant sinks and transitions labelled,
- an enumerated list of privacy threats against that diagram in a vocabulary a regulator will recognise (LINDDUN categories map cleanly onto GDPR principles),
- a risk assessment overlaying likelihood × impact (NIST PRAM shape) so engineering can prioritise,
- a routing table from each threat to a specific control in your stack (DP, redaction, tokenisation, minimisation, consent propagation, retention, isolation, attestation),
- the evidence a DPIA (mod-111) will consume downstream.

Everything in this module builds up to that artifact.

## Security threat modelling vs. privacy threat modelling

Security threat modelling asks: *what could a malicious actor do to the system?* STRIDE's six categories — Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege — target CIA (confidentiality / integrity / availability) and authority.

Privacy threat modelling asks: *what could the system do to the data subject?* The subject may not be a party to the session at all — they may be a person whose data was in a training corpus crawled years ago. The threat model has to account for harms the subject experiences even when the system is working **exactly as designed**.

Concretely, STRIDE under-covers:

- **Linkability** — combining two records (each innocuous alone) into a profile. STRIDE does not have a category for this because it is not a confidentiality failure in the CIA sense; nothing is "disclosed" beyond what the operator already held.
- **Identifiability** — turning a pseudonymous record back into a person via side information. Again, no CIA breach.
- **Unawareness** — the subject cannot form a correct mental model of what is being done with their data. Not a security failing; a transparency failing.
- **Non-compliance** — the processing violates a statute. Not an attack; a design gap.
- **Detectability** — the fact that a record exists at all leaks sensitive information (e.g. the existence of a record in a mental-health dataset).
- **Non-repudiation** — in privacy, the *inverse* sign from security: the subject may want to be *able* to repudiate an action, which an over-logged system prevents.

LINDDUN (KU Leuven, DistriNet) was designed to fill that gap. Its seven categories — Linkability, Identifiability, Non-repudiation, Detectability, Disclosure of information, Unawareness and unintervenability, Non-compliance — are the vocabulary for the rest of this module.

Security and privacy threat models often complement each other. The security peer will run STRIDE on authentication, session, and network surface. You will run LINDDUN on the data surface. Where they overlap — "the prompt log contains PII; it is behind weak IAM; an insider can read it" — both methodologies produce the same finding from different sides, and that is a *strength*, not duplication.

## Why the AI data flow forces the methodology to adapt

A pre-AI web application had a small set of privacy-relevant sinks: the application database, an analytics pipeline, logs, maybe a data warehouse. Erasure meant SQL DELETE. Access meant SELECT. Minimisation meant `NOT NULL`-ing a column.

An AI/ML system has many more sinks, and most of them are *stateful* in ways that resist conventional rights-handling:

- **Training corpora** — snapshotted, may be versioned, may be shared with third parties for fine-tuning.
- **Retrieval corpora** — vector-indexed; the index is a derivative of the raw data that still leaks membership.
- **Inference prompts** — captured in session, often logged for quality and safety review.
- **Generated outputs** — sometimes stored; often the thing the subject is most sensitive about.
- **Prompt / completion logs** — frequently a dumping ground for regulated data.
- **Model weights** — the subject's data may be *in the weights*. Erasure here is mod-110-hard.
- **Embeddings** — derivative representations, still often invertible enough to re-identify.
- **Caches** — KV caches, prefix caches, semantic caches; each a distinct sink.
- **Telemetry** — tool calls, latency, user-session metadata; linkable across users.

LINDDUN was designed against classic DFDs (processes, data stores, data flows, external entities, trust boundaries). The categories survive intact when the DFD's elements are an embedding index or a KV cache — but *you* have to enumerate the new elements correctly, or the analysis is hollow. Chapter 02 does that decomposition. Chapters 03–05 run LINDDUN on it.

## The three-layer artifact this module produces

By the end of the module, for any AI system you touch, you can produce a layered threat-model artifact:

1. **Decomposition** (chapter 02) — the privacy-relevant DFD for the system. Nodes labelled by the sink type above; edges labelled by trust-boundary crossings and personal-data classes.
2. **Enumeration** (chapters 03–05) — LINDDUN GO for the workshop pass, LINDDUN MAESTRO for the parts that warrant depth. One threat per (element, category) cell at minimum.
3. **Prioritisation + routing** (chapters 06–07) — NIST PRAM-shape risk assessment (data actions → problematic data actions → likelihood × impact); misuse-persona overlay; each surviving threat routed to a control. 

This layered output is what you hand to the DPIA (mod-111) and to engineering backlog grooming. It is also the artifact the regulator will read first when a question is asked.

## The engineering checkpoints layer

Chapter 08 then converts the model into **engineering checkpoints** based on Cavoukian's seven Privacy-by-Design principles (which GDPR Art. 25 codifies as "data protection by design and by default"). A threat model that lives in a document and a sprint board that never references it fails. The checkpoint set is how the model gets enforced at design-review, CI, and release gates.

## What this is *not*

- **Not a harm model.** The broader harm surface (misuse personas beyond re-identification, capability-surface harms, downstream integrator abuse, societal-level harms) is owned by `ai-risk-engineer` (level 25). Chapter 09 is the deferral contract. You cite into it; you do not re-deliver it.
- **Not a DPIA.** The DPIA is a different artifact with a different audience (regulator-facing, GDPR Art. 35 shape). It *consumes* this threat model. mod-111 teaches the DPIA.
- **Not a security review.** When the threat is "attacker exfiltrates the prompt log", that's a security control first and a privacy control second. The security peer owns the attack surface; you own the data-surface design that limited the blast radius.

## Summary — what you can do after this chapter

- Explain to a product team why STRIDE alone is not enough and what LINDDUN adds.
- Describe the three-layer artifact (decomposition → enumeration → prioritisation + routing) and which chapter delivers which layer.
- Say where this module stops and `ai-risk-engineer` / DPIA / security peer picks up.
- Identify the AI-specific sinks that make the data flow harder than a pre-AI web app.

The next chapter turns the data-flow sinks above into a reusable decomposition template.
