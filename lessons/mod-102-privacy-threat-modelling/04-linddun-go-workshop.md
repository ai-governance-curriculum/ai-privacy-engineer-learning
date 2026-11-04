# LINDDUN GO — The Workshop-Speed Privacy Threat Model

LINDDUN GO is the lightweight, card-based variant of LINDDUN designed to run as a time-boxed workshop with a product team rather than as a solo analyst exercise. It is the right method when you need a useful privacy threat model in *one working session* and you have engineers, PM, and (ideally) a DPO delegate in the room. This chapter is the engineer's playbook for running one on an AI system.

Primary source: the LINDDUN GO materials and card deck at [linddun.org](https://linddun.org/). Download the current deck before you facilitate — the canonical cards are the artifact the team should be looking at.

## When to pick GO (and when not to)

**Use LINDDUN GO when:**

- The system is at *design review* or *early prototype* and the team is still open to cheap changes.
- You can get the right room for 2–4 hours.
- The system is small enough (one or two inference paths, one training path, a handful of sinks) that a card-by-card pass can cover it.
- You need a prioritised list of threats *fast* to unblock a DPIA or a release go/no-go.

**Pick LINDDUN MAESTRO (chapter 05) instead when:**

- The system is already in production and the threat model will drive durable control investment.
- The surface is wide enough that card-by-card is too shallow (e.g. multi-tenant platform with 20+ retrieval corpora, multiple fine-tune paths, agent-tool integrations).
- You need regulator-grade rigour — exhaustive per-element enumeration, threat trees, documented refinement reasoning — because the DPIA is high-stakes.

A common pattern is *GO first, MAESTRO on the surviving hot spots*. The GO pass triages; the MAESTRO pass deepens. This is the recommended default for a new AI system.

## Inputs you must bring

Do not walk into a GO workshop without:

1. **The privacy DFD** (chapter 02). Drawn, printed or projected, with sinks and trust boundaries labelled. Without a DFD the workshop devolves into generic hand-waving.
2. **The LINDDUN GO card deck**, physical or on a shared whiteboard. The cards contain prompt questions per LINDDUN category — e.g. for Linkability, "Can an attacker *link* two or more actions / data items of the subject together?" Use the published deck; do not paraphrase from memory.
3. **A scribe** separate from the facilitator.
4. **The plan from chapter 02 to iterate element-by-element** — not category-by-category. Doing it category-first produces a shallow cross-system scan; doing it element-first ensures every sink gets attention.
5. **A time-boxed agenda**. 15 minutes per element is a reasonable starting block; some elements (weights, logs) will overrun; move short on others.

## A four-stage GO agenda for an AI system

### Stage 0 — Pre-brief (owner: you, 30 minutes, async)

- Send the DFD, the LINDDUN card deck, this chapter, and the running example to participants 24+ hours ahead.
- Have the team pre-fill the *DFD elements* list and flag any they disagree with.
- Share a short list of "assumed out of scope" items (e.g. authentication surface — security peer, chapter 09's boundary).

### Stage 1 — DFD walkthrough (facilitator + system lead, 20 minutes)

- The system lead walks the DFD aloud.
- Everyone raises hands for *missing sinks* using the nine-sink template from chapter 02 as a checklist.
- Scribe adds any missed sinks live. The DFD for the rest of the workshop is now authoritative.

### Stage 2 — Card pass per element (90 minutes for a small system)

For each DFD element (process, data store, data flow, external entity, trust boundary):

1. Facilitator lays out the seven cards (one per LINDDUN category) next to the element.
2. For each card, the facilitator reads the prompt, and the room produces *one or more candidate threats* or an explicit "N/A — because …".
3. Scribe captures each threat in a one-sentence form: *"[Element] → [category]: [threat statement]."*
4. If the room is split on whether a candidate is a threat, write it down anyway. Triage later.

Timings for an AI system typically land:

- Model weights — 20+ minutes (I, D1, D2, U all surface).
- Prompt / completion logs — 15+ minutes (L, I, N-rep, D2, U).
- Retrieval corpus + vector index — 15+ minutes per tenant model.
- Caches — 10 minutes.
- Training corpus — 10 minutes (much of the depth is deferred to mod-103).
- Everything else — 5–10 minutes each.

### Stage 3 — Triage + routing (30 minutes)

For every captured threat:

1. Mark **in scope** or **out of scope**. Out-of-scope categories:
   - Covered by the security peer (STRIDE will catch it) — tag and move on.
   - Owned by a different role per chapter 09 — tag with the owning role and move on.
   - "Not a real threat on this system because [reason]" — write the reason, keep for the record (so the next GO does not re-ask it).
2. Assign a **rough likelihood × impact** on a 3-point scale (High / Medium / Low) each. This is intentionally *coarse*; chapter 06 formalises it with NIST PRAM.
3. Route each surviving threat to a **control family** — DP, redaction, tokenisation, retention, ACL, consent, isolation, confidential compute, DSR path, meaningful-information artifact. Do not over-specify the control in the workshop — just the family.

### Stage 4 — Wrap (10 minutes)

- The facilitator reads the top five threats by (likelihood × impact) aloud.
- Each threat is owned by *a named person* in the room, with a next-action commitment.
- The scribe exports the threat list; the artifact lands in the system's threat-model document (versioned with the code, not in a shared drive).

## The artifact to produce

The deliverable from a GO workshop is a single Markdown document with:

- The DFD (image / Mermaid inline).
- A table with columns: **ID**, **element**, **category**, **threat statement**, **in-scope?**, **likelihood**, **impact**, **control family**, **owner**, **next action**, **status**.
- An "out of scope" appendix with the reason for each exclusion.
- A changelog.

This becomes the input to the DPIA (mod-111), to the engineering backlog, and to the next LINDDUN MAESTRO pass (chapter 05).

## Facilitation notes specific to AI systems

Running GO on AI systems surfaces some facilitation pitfalls worth naming:

- **Model weights almost always get under-threat-modelled.** Teams who have not internalised mod-105 tend to treat weights as inert. Pre-brief the team on the memorisation / extraction threat class so the room is prepared.
- **Caches are forgotten.** Prefix cache, KV cache, semantic cache, provider-side cache, retrieval-router cache — enumerate explicitly on the DFD or GO will miss them.
- **Telemetry is treated as "metadata, not personal data".** Tool-call arguments, trace payloads, and span IDs often contain personal data and need their own Linkability pass.
- **The tenant boundary is *the* highest-impact boundary in multi-tenant RAG.** Spend disproportionate time on it.
- **"Legitimate interests for training" needs the DPO.** When the room reaches a Non-compliance card on training-data lawful basis, you cannot resolve it in the workshop — route it to the DPO per chapter 09 and move on. Do not wait.
- **End with named owners.** A threat without an owner is a threat that will be present in the next GO workshop.

## A tiny worked slice

Suppose the DFD element is **semantic cache** and the system is the tenant-scoped RAG customer-support assistant from chapter 02. A plausible card pass:

| Card | Threat captured | In-scope | L | I | Control family | Owner |
|---|---|---|---|---|---|---|
| L (Linkability) | Cache keyed on normalised prompt-embedding similarity; cross-user hits possible when keys collapse near-duplicates. | yes | H | H | isolation (per-tenant partition) | retrieval-lead |
| I (Identifiability) | Cached prompts contain uncommon strings (customer names, order IDs) that re-identify. | yes | M | M | redaction at cache-write | logs-lead |
| N-rep | Low; no subject-attributable action recorded beyond request-time. | no | — | — | — | — |
| D1 (Detectability) | Cache size / latency side-channel reveals whether a given prompt has been asked before. | yes | L | M | cache size capping, latency jitter | infra-lead |
| D2 (Disclosure) | Cross-user hit returns another subject's completion verbatim. | yes | H | H | tenant partitioning + negative tests | retrieval-lead |
| U (Unawareness) | Privacy notice does not disclose caching. | yes | M | M | notice update | privacy-notice-lead (routes to DPO) |
| N-comp | Caching across jurisdictions without a cross-border mechanism. | yes | M | H | geo-aware partitioning | infra-lead |

Six of the seven categories produced a real in-scope threat on one element, from a single card pass. That is typical for AI systems.

## Summary — what you can do after this chapter

- Facilitate a LINDDUN GO workshop on an AI system end-to-end in a working session.
- Pre-brief the room so they arrive with the DFD and the right scope.
- Run the per-element card pass and avoid the AI-specific pitfalls (weights, caches, telemetry, tenant boundary, legitimate interests).
- Produce a Markdown threat-model artifact routable to a DPIA and a backlog.
- Know when to escalate to LINDDUN MAESTRO (chapter 05).

The next chapter is LINDDUN MAESTRO — what to do when the GO pass surfaces hot spots that need more depth.
