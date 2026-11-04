# Decomposing an AI System Into a Privacy-Relevant Data-Flow Diagram

LINDDUN, NIST PRAM, and every downstream DPIA assume you have a decomposition of the system to enumerate threats against. This chapter is the engineer's template for producing one — specifically for AI/ML systems, where the sinks and transitions are wider than a classical web application.

The output of this chapter is a reusable **privacy DFD** template: a canonical set of nodes, edges, trust boundaries, and per-element labels you fill in for any AI system you touch. Chapters 03–05 run LINDDUN against this template. mod-111 feeds it into the DPIA.

## Why the AI data flow deserves its own template

Classical DFD guidance (Microsoft SDL, OWASP, the original LINDDUN papers) names four element types: *processes*, *data stores*, *data flows*, *external entities*, plus *trust boundaries*. Those survive. The difficulty is that an AI system introduces **sink types** that a classical DFD does not call out by name:

- model weights are a *data store* (they memorise data),
- embeddings are a *data store* that is a one-way transform of personal data (but far from fully one-way),
- KV / prefix / semantic caches are *data stores* that persist beyond a session in ways most code reviewers miss,
- a vector retrieval index is a *data store* that indexes identifiers of source documents,
- tool-call telemetry from an agent is a *data flow* that may cross a trust boundary the team did not draw.

If you draw a boilerplate "user → app → DB" DFD for an AI system, you lose half the surface. The template below is designed to prevent that.

## The nine-sink AI privacy DFD template

Treat these as the *default list of nodes* in your diagram. For a given system, some will be absent — a classifier that is not retrieval-based has no retrieval corpus — but you must consciously decide which are absent, not accidentally omit them.

### 1. Training corpora

- Raw ingested data that trained the model(s) in use.
- May be versioned, snapshotted, or shared with a third-party fine-tuning service.
- Includes *scraped* data, *purchased* data, *user-contributed* data, *synthetic* data (which is still regulated when it contains inferred personal data).
- **Transitions you must draw**: ingest → clean → annotate → freeze-as-snapshot → feed-to-trainer. Each has a trust boundary.

### 2. Retrieval corpora

- Documents / tables / records the system retrieves from at inference time (RAG, lookup-augmented models).
- The *vector index* is a distinct sink from the raw corpus (chapter 05 of this module and mod-109 go deeper here).
- Often per-tenant-scoped; the ACL design is a privacy-critical control.

### 3. Inference prompts

- The text / image / audio input a user (or upstream service) sends to the model.
- *Includes* system prompts, which may themselves contain privacy-sensitive configuration (names of customers, internal case IDs).
- *Includes* chat history replayed into the prompt — a frequent unexamined sink.

### 4. Generated outputs

- Model completions, images, decisions, scores.
- Often the thing the subject is *most* sensitive about (e.g. an inferred mood, credit outcome, health risk) and a frequent GDPR Art. 22 touchpoint (see mod-101 chapter 02).

### 5. Prompt / completion logs

- Captured for quality review, safety / abuse review, debugging, evaluation, future fine-tuning.
- Frequently the privacy-worst sink in an AI product — everything that transited the inference path lands here, often with weaker retention and ACL than the operational database.
- Sub-sinks: raw log store, sampled fine-tune corpus, evaluation set, red-team set, abuse-review queue.

### 6. Model weights

- Base model, fine-tune deltas, LoRA adapters, PEFT artifacts, distilled student models.
- A privacy-relevant data store because *training-corpus content can be memorised and extracted* (mod-105).
- Right-of-erasure reaches this sink last and hardest (mod-110).

### 7. Embeddings

- Dense vectors produced by an encoder and stored in a retrieval index, a feature store, or a cache.
- Often believed to be privacy-neutral because they "aren't the raw text"; inversion attacks (`Embedding Inversion`, vec2text) have shown this is not reliable.
- Enumerate separately from the retrieval corpus — the embedding is a *derivative* that outlives any deletion of the raw record if you forget to re-index.

### 8. Caches

- KV caches (attention state), prefix / prompt caches, semantic caches (recent request → recent response), inference-router caches, provider-side caches (watch the vendor contract).
- Typically weakly logged, weakly retained, weakly deleted-on-DSR. A high-frequency source of surprise findings.

### 9. Telemetry

- Tool-call traces, span data, latency per span, feature flags evaluated, request IDs, session IDs.
- Linkable across users with surprising ease once you have more than a handful of dimensions.
- Agent systems multiply this surface: each tool invocation is a new span, often with an argument that contains the user's data.

## Edges and trust boundaries

Nodes alone are not enough. The *edges* and the *trust boundaries they cross* are where most LINDDUN threats attach. For each edge in your diagram, label:

- **Data classes crossing** — personal data, special-category (GDPR Art. 9), PHI, PCI, children's data, derived / inferred data, pseudonymised data.
- **Lawful basis** (where personal data is in scope) — consent, contract, legitimate interests, legal obligation (see mod-101 chapter 02).
- **Trust boundary crossed** — tenant boundary, process boundary, VPC boundary, cloud-tenant boundary, geographic / jurisdictional boundary (cross-border transfer), organisational boundary (to a processor / sub-processor).
- **Transport controls** — TLS, mTLS, in-memory only, confidential-VM / TEE (mod-108).
- **Retention controls at the destination** — how long the data stays, under what deletion policy.

A privacy DFD without labelled edges is a security DFD. The labels are what make it *privacy*.

## External entities

Every AI system has at least these external entities. Draw them explicitly:

- **Data subject** — whose data is in the corpora / prompts.
- **End user / operator** — the person interacting with the system (may or may not be the subject).
- **Upstream data supplier** — crawler feed, licenced dataset vendor, user-contributed data upstream.
- **Model provider / fine-tune service** — if you do not own the weights.
- **Retrieval-document author** — the party who created the documents the RAG retrieves from.
- **Downstream integrator** — the party that calls your API and accumulates results (an important LINDDUN actor — see chapter 07).
- **Regulator / supervisory authority** — observer of the whole system.

## Trust boundaries worth drawing explicitly

- **Tenant boundary** (multi-tenant systems) — the single highest-leverage control in RAG privacy (mod-109).
- **Organisational boundary to a processor / sub-processor** — where GDPR Art. 28 lands.
- **Jurisdictional boundary** — where GDPR Chapter V cross-border rules land.
- **Model boundary** — in / out of the weights is a trust boundary for mod-105 memorisation / extraction threats.
- **Confidential-compute boundary** — an enclave attestation (mod-108) defines a trust boundary the DFD should show.

## Worked example — a tenant-scoped RAG customer-support assistant

The default scenario from mod-101 exercise 01 is a useful running example. Its privacy DFD has:

- **External entities**: data subject (= end-customer), end-user (= same, but could be a human support agent), upstream data supplier (= the ticket platform), model provider (hosted LLM), downstream integrator (= internal analytics team querying the log), regulator.
- **Processes**: intake / normalisation, fine-tune trainer, retrieval orchestrator, inference server, abuse-review pipeline, evaluation pipeline, DSR processor.
- **Data stores**: raw ticket archive, fine-tune training snapshot, retrieval corpus (per-tenant), vector index, prompt/completion log, model weights (fine-tuned + base), embedding cache, semantic cache, telemetry trace store.
- **Trust boundaries**: tenant boundary (crosses the retrieval index, the semantic cache, the log), organisational boundary to model provider (crosses inference-time prompt / completion), jurisdictional boundary if any tenant is EU and any store is outside, model boundary (training data is *in* the fine-tune).
- **Edges to label**: every inference call carries prompt + authenticated-customer-identity across the organisational boundary to the model provider; labelling this edge forces questions about retention at the provider, IP for the completion, and whether the prompt can be used for provider-side training.

For your deliverables (exercise 01 of this module), you will produce an equivalent DFD for an AI system you choose.

## Toolchain options

You can draw the DFD in any format you can version-control. Preferred shapes:

- **Mermaid / `graph`** — lives in Markdown next to the threat-model doc, trivially diffable.
- **draw.io / diagrams.net** — exports to SVG + XML; the XML is diff-able.
- **OWASP Threat Dragon** — threat-modelling-aware, STRIDE-oriented; you can still author LINDDUN notes.
- **LINDDUN GO / PRO tooling** — see chapter 04.

Avoid Visio / Lucid for this purpose unless you have no alternative — the artifact needs to be review-able in a code-review workflow.

## Two common failure modes

### Failure mode 1 — missing sinks

A DFD that lacks model weights, embeddings, caches, or telemetry under-represents the privacy surface. The LINDDUN pass against such a DFD will miss the memorisation threats (mod-105), the embedding-inversion threats (mod-109), and the cache / telemetry linkability threats. Chapter 03 shows which categories attach to which sinks; use that as your completeness check.

### Failure mode 2 — missing trust boundaries

A DFD without a tenant boundary for a multi-tenant RAG system will hide the single-highest-impact threat class (cross-tenant retrieval leak). A DFD without a model-provider organisational boundary will hide a cross-border / processor-contract failure. Trust boundaries are what turn flow lines into LINDDUN threats; without them the model is almost decorative.

## Summary — what you can do after this chapter

- Draw a privacy DFD for an AI system using the nine-sink template.
- Label each edge with data class, lawful basis (where applicable), trust boundary crossed, transport control, and destination retention.
- Name the standard external entities (subject, user, upstream supplier, model provider, downstream integrator, regulator).
- Spot a DFD that is missing sinks (weights, embeddings, caches, telemetry) or missing trust boundaries (tenant, organisational, jurisdictional).

The next chapter walks the seven LINDDUN categories and anchors example threats to each DFD element above.
