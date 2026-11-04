# Misuse Personas Against Privacy and Routing Threats to Controls

LINDDUN and PRAM describe *threats* and *risks*. For each surviving threat, you need a named *actor* and a *control* — otherwise the register is unactionable. This chapter does both: a canonical set of privacy-specific misuse personas for AI systems, and the routing from each threat class to a concrete control family in the curriculum.

The broader *harm model* (misuse personas beyond privacy, capability-surface harms, malicious operator / data / integration-caused harms) is owned by `ai-risk-engineer` (level 25, peer). This chapter covers the *privacy-specific* subset. Chapter 09 sets out the boundary.

## Why personas earn their place

An abstract threat ("the semantic cache enables cross-tenant disclosure") is easier to evaluate — and easier to score likelihood on — when you can name the plausible actor:

- An authenticated user of another tenant stumbling onto a cached response (low effort, present everywhere).
- A downstream integrator systematically probing for cross-tenant hits (moderate effort, requires business access).
- An insider abusing the cache-admin role (high privilege, specific access).

The three personas score differently on PRAM likelihood (chapter 06) and route to different controls (tenant partitioning vs. ACL vs. break-glass logging). The persona frame is what lets the register be *actionable* rather than just correct.

## Four canonical privacy misuse personas for AI systems

These are the *privacy-specific* personas. The AI-risk track (`ai-risk-engineer` mod-107 / mod-108 slice) adds many more (prompt-injection abusers, capability-uplift seekers, misinformation operators); route those to the peer.

### Persona 1 — Adversarial user attempting re-identification

- **Who.** An authenticated or lightly-authenticated user of the system, interacting through supported interfaces (chat, API, retrieval queries, agent tool calls).
- **Capabilities.** Query the inference API. Craft prompts. Observe outputs, confidence scores, timing, retrieval citations. May have side information about a target subject (a name, a location, a partial identifier).
- **Goal.** Determine whether subject X is in the training corpus (membership inference), in the retrieval corpus (presence inference), or recover identifying content about subject X (extraction / inversion).
- **Attaches to.** Model weights (D1, I), retrieval corpus (D1, D2), generated outputs (I), confidence scores (D1), caches (L, D2).
- **Cites.** mod-105 (MIA, extraction, canary methodology), mod-106 (DP-SGD defence).

### Persona 2 — Insider attempting training-data extraction or log mining

- **Who.** A privileged role inside the operator organisation — ML engineer with weight access, SRE with log access, abuse-reviewer, trust-and-safety analyst, customer-support engineer with support-tool console.
- **Capabilities.** Direct access to one or more sinks (weights, logs, caches, retrieval index, prompt corpus, evaluation set). Possibly the ability to run arbitrary code against the model offline.
- **Goal.** Extract specific individuals' data from the corpora or weights, or mine logs for named individuals. Motivation may be curiosity, harassment, commercial resale, or coerced access.
- **Attaches to.** Any sink with a human reader role. Especially prompt / completion logs, abuse-review queues, model weights (offline), retrieval corpora, evaluation samples containing real customer data.
- **Cites.** mod-104 (redaction at write), mod-105 (weight-side risk), mod-108 (confidential compute / attestation-bound inference), mod-110 (DSR surface).

### Persona 3 — Upstream data supplier violating consent / lawful basis

- **Who.** A party *upstream* of your ingestion pipeline — a crawler vendor, a dataset licensor, a user-contribution channel, a partner feed, a scraping operator.
- **Capabilities.** Supplies data into your training or retrieval corpora. May or may not represent correctly whether the data was collected under a lawful basis, whether consent was obtained, whether special-category data is present, whether data subjects were informed.
- **Goal.** Not necessarily adversarial — often negligent. Supplies data that is *downstream-unusable* without triggering non-compliance threats in your system.
- **Attaches to.** Training corpora, retrieval corpora, labelling pipelines, data-augmentation feeds.
- **Cites.** mod-103 (data governance, consent propagation, dataset cards, ML-BOM), mod-101 chapter 02 (GDPR Art. 6 / 9), mod-101 chapter 03 (EU AI Act Art. 10 provenance).

### Persona 4 — Downstream integrator accumulating linkable traces

- **Who.** A party *downstream* of your inference API — a customer integrating your model into their product, a reseller, an aggregator, an analytics vendor. Also includes your own internal analytics team if they are an organisationally separate consumer.
- **Capabilities.** Observes inputs and outputs to your system over time for subjects *they* have identifiers for. May accumulate per-subject profiles from otherwise-pseudonymous traces (completions, confidence scores, tool-call patterns, latency signatures).
- **Goal.** Build a subject-level profile that your system does not itself retain but their position lets them assemble. Linkability + identifiability compounded.
- **Attaches to.** Any API edge your system produces output on. Especially agent telemetry, structured outputs that carry correlatable identifiers, confidence scores, retrieval citations.
- **Cites.** mod-109 (agent-tool scoping, retrieval ACL), mod-101 chapter 02 (GDPR Art. 28 processor contracts; Art. 22 meaningful information shared with downstream parties).

## Routing threats to control families

Every surviving LINDDUN threat (chapter 03/04/05), scored in PRAM (chapter 06), should route to *one* control family from the taxonomy below. Multiple controls may be needed; one *routing* decision is required.

### Control families in this curriculum

| Family | Concept | Primary module |
|---|---|---|
| **Data minimisation** | Collect / retain / log only what the purpose requires. | mod-103 |
| **Consent + lawful-basis propagation** | Record, propagate, honour the basis / consent per processing purpose. | mod-103 |
| **Redaction / tokenisation / vaults** | Replace PII with tokens / vault references before it enters a persistent sink. | mod-104 |
| **De-identification (statistical)** | k-anonymity / l-diversity on tabular release; expert-determination / Safe-Harbor for health data. | mod-104 |
| **Differential privacy** | Formal DP at training (DP-SGD), at analysis (DP queries), at release (DP synthetic data). | mod-106 |
| **Federated learning + secure aggregation** | Train without centralising raw data; aggregate with crypto guarantees. | mod-107 |
| **Confidential computing + PP inference** | Attestation-bound inference / training in TEEs; HE; MPC. | mod-108 |
| **RAG privacy + agent scoping** | Per-tenant retrieval ACL; agent-tool data boundary; prompt / completion log policy. | mod-109 |
| **Machine unlearning + DSR** | Weight-level deletion decisions; per-sink erasure paths; DSR intake + routing. | mod-110 |
| **DPIA + evidence** | Documented pre-processing risk assessment; regulator-ready evidence bundle. | mod-111 |
| **Program + communication** | Privacy-notice content, model/dataset-card transparency, incident response. | mod-112 |

### Routing table for the four personas

The persona-to-control routing is a defensible default; adapt per system.

| Persona | Primary routed controls | Secondary |
|---|---|---|
| Adversarial user (re-identification) | DP (mod-106), API confidence capping, extraction testing (mod-105), retrieval ACL (mod-109) | Rate-limiting + anomaly detection (security peer) |
| Insider (extraction / log mining) | Redaction at write (mod-104), ACL + just-in-time access, break-glass + audit, confidential compute (mod-108), log retention (mod-103) | Background-check + role review (People) |
| Upstream supplier (consent / lawful basis) | Dataset-card contract, consent-propagation metadata, special-category screening (mod-103, mod-104), ML-BOM | Procurement contracts (Legal) |
| Downstream integrator (linkage) | Output minimisation, telemetry scoping (mod-109), Art. 28 processor contract, output-side pseudonymisation | Rate / purpose limits (Product) |

### Routing by threat *category*, when persona is unclear

When you have a threat but the actor is ambiguous, route by LINDDUN category:

- **L (Linkability)** — identifier rotation, session pseudonymisation, cache + telemetry scoping (mod-109).
- **I (Identifiability)** — redaction / tokenisation (mod-104), de-identification (mod-104), DP training (mod-106), extraction testing (mod-105).
- **N (Non-repudiation, undesired)** — retention minimisation, pseudonymisation at write, subject-initiated redaction (mod-110).
- **D1 (Detectability)** — DP training (mod-106), confidence capping, ACL on retrieval (mod-109).
- **D2 (Disclosure)** — tenant partitioning, encryption + KMS, TEE / confidential compute (mod-108), break-glass logging.
- **U (Unawareness / unintervenability)** — layered privacy notice, model + dataset cards (mod-103, mod-112), DSR surface (mod-110), Art. 22 meaningful-information artifact.
- **Non-compliance** — lawful-basis register, LIA documents, Art. 28 contracts, cross-border mechanism, DPIA (mod-111), Annex IV documentation.

## The routing register

The deliverable from this chapter integrates into the register from chapter 06. Add columns:

- **Persona(s)** — the misuse persona(s) the threat belongs to.
- **Primary control family** — one choice from the taxonomy above.
- **Secondary control families** — optional additional defence-in-depth selections.
- **Owning module** — which curriculum module delivers the control depth (lets you cite "mod-106 DP-SGD" instead of re-authoring the control).
- **Backlog ticket** — where the engineering work tracks.

With this in place, the register is a direct bridge to the engineering backlog. Chapter 08 turns the bridge into *gates* that block delivery when the control is missing.

## Two routing anti-patterns

- **Routing everything to "DP".** DP is the right control for a specific class of threats (training-time identifiability / detectability). Routing a prompt-log IAM problem to "add DP to training" is a category error. Match the control family to the category and the sink.
- **Routing everything to "redaction".** Redaction is powerful but partial. It cannot fix a non-compliance threat on lawful basis; it cannot fix a downstream-integrator linkability threat after the API emits a traceable identifier; it cannot fix a tenant-isolation bug. Apply redaction at the sinks where it earns its keep (logs, caches, evaluation sets) and route other threats elsewhere.

## Summary — what you can do after this chapter

- Name the four privacy misuse personas and the capabilities / goals / sink attachments of each.
- Route any LINDDUN threat to a control family in the curriculum.
- Produce a routing register that bridges the PRAM register to the engineering backlog.
- Avoid the "everything is DP" / "everything is redaction" anti-patterns.
- Cite the role boundary to `ai-risk-engineer` for personas outside the privacy scope.

The next chapter converts the whole model into Privacy-by-Design engineering checkpoints — the enforcement surface that keeps the threat model from drifting.
