# Exercise 01: Privacy Data Flow Decomposition for One AI System

**Estimated effort:** 2 hours · **Output:** one Markdown file with an inline DFD (Mermaid) + element / edge tables · **Primary chapter:** 02

## Objective

Produce a **privacy-relevant data-flow diagram** of one real AI system, labelled to the depth LINDDUN GO (chapter 04) and PRAM (chapter 06) can run against. This is the artifact every subsequent exercise in this module reuses, so build it well.

## Prerequisites

- Chapter 02 (privacy data flow decomposition) read.
- Chapter 01 (motivation) read so you know what the DFD is *for*.
- An AI system you can describe at sink / edge / trust-boundary depth. Candidates:
  - A system you've built or shipped (ideal).
  - A system at your current employer you can describe in detail.
  - A documented public system — pick one with sufficient public material (an Apple PCC blueprint, an OpenAI system card, a published RAG reference architecture).

If you have no system in mind, use **the default scenario** from mod-101 exercise 01 (the tenant-scoped RAG customer-support assistant) so you can finish.

## Problem statement

Produce a single Markdown document containing three sections:

### Section A — the DFD (Mermaid, inline)

A Mermaid diagram embedded in the Markdown. Nodes should include at least:

- **External entities** — data subject, end user, upstream data supplier, model provider (if any), retrieval-document author (if any), downstream integrator, regulator.
- **Processes** — intake / normalisation, trainer, retrieval orchestrator, inference server, log-write pipeline, abuse-review pipeline, evaluation pipeline, DSR processor.
- **Data stores** — raw corpus, training snapshot, retrieval corpus (per-tenant if applicable), vector index, prompt / completion log (break into sub-stores if they exist), model weights (base + fine-tune + any LoRA / adapters), embedding cache, semantic cache, telemetry trace store, backups / snapshots.
- **Data flows** — every arrow you would draw.
- **Trust boundaries** — tenant, organisational (to model provider / sub-processor), jurisdictional (cross-border), model boundary (in / out of weights), confidential-compute enclave (if applicable).

Use subgraph boxes to show the trust boundaries. The graph does not need to look like production-quality architecture art — it needs to be *correct* and *review-able*.

### Section B — the element table

A Markdown table listing every element (external entities, processes, data stores) with:

| ID | Name | Element type | Short description | Personal data classes present | Lawful basis (if personal data) | Retention | ACL / owner role |

Rules:

- Use the nine-sink template from chapter 02 as a completeness checklist. If a sink is absent, state so explicitly in a "Not in scope" bullet with a reason.
- *Model weights* must appear as a data store even when they live at a third-party provider — mark the ownership accordingly.
- *Embeddings* must appear separately from the retrieval corpus.
- *Caches* — enumerate each distinct cache (prefix / KV / semantic / retrieval / provider-side) that exists in your system. Do not collapse.

### Section C — the edge table

A Markdown table of every data flow in the DFD, with:

| ID | From | To | Data classes crossing | Trust boundary crossed (if any) | Transport control | Destination retention | Notes |

Rules:

- Every edge that *crosses a trust boundary* must be marked as such.
- Every edge carrying personal data must name the data class (personal / special-category / PHI / PCI / children's / inferred).
- Transport control: TLS, mTLS, in-memory-only, confidential-VM / TEE, or "none" (if it is a within-process call).
- If an edge crosses a jurisdictional boundary, name the cross-border mechanism (SCC / adequacy / BCR / not-applicable).

## Starter guidance

- **Draft the Mermaid DFD first; iterate on the tables second.** The diagram will surface missing elements that the tables otherwise would not.
- **Name sinks concretely.** `prompt-log-raw` (S3 bucket, 90-day retention, `prompt-log-reader` IAM role) is a sink; "prompt logs" is not.
- **Include the backup / snapshot sink explicitly.** Most GDPR Art. 32 incidents live here and most DFDs forget it.
- **If your system has no retrieval / fine-tune / agent tooling, mark those elements "Not in scope — because …" rather than deleting from the template.** The reader needs to see you considered them.
- **Separate fine-tune / LoRA adapters from the base model.** Erasure (mod-110) reaches them differently.
- **Draw the "legitimate interests for training" edge explicitly if that is your basis.** This is one of the most-questioned decisions in regulator reviews.

## Acceptance criteria

- [ ] Mermaid DFD renders and visually distinguishes trust boundaries.
- [ ] Every one of the nine sinks from chapter 02 is either on the diagram or explicitly marked "Not in scope — because …".
- [ ] Every data flow carrying personal data has a labelled data class and transport control.
- [ ] Every trust-boundary crossing is marked with the mechanism that governs it (ACL, Art. 28 contract, cross-border mechanism, enclave attestation).
- [ ] Caches, telemetry, embeddings, and model weights are *individually* enumerated — not collapsed into "logs".
- [ ] The retention column is populated for every data store (even if the value is "indefinite" — call it out).
- [ ] At least one edge is labelled with a cross-border mechanism *or* an explicit "no cross-border processing" statement.
- [ ] A reader who has not seen your system can walk the DFD and the tables and produce a candidate LINDDUN threat per element without asking you a clarifying question.

## Stretch goals

- Add a fourth table listing *external entities* with a one-line "why we care about this party in a privacy sense" statement. This becomes the skeleton of the misuse-persona exercise (exercise 05).
- For any sink labelled "indefinite" retention, write a short justification (one paragraph) referencing the processing purpose and the GDPR Art. 5(1)(e) storage-limitation principle.
- If your system has no DSR intake / routing process in the DFD, add one — even a stub — and state what sinks it would need to reach. This sets up the mod-110 unlearning / DSR work.
- Produce a second version of the DFD showing *only the edges that carry special-category data*. The resulting sub-graph is a useful artifact for Art. 9 / EU AI Act Art. 10(5) review.

## What this lands

The DFD and its tables are the *input* to every other exercise in this module:

- Exercise 02 (LINDDUN GO workshop) needs the DFD in the room.
- Exercise 03 (LINDDUN MAESTRO) refines the DFD at sub-store depth.
- Exercise 04 (PRAM overlay) enumerates data actions from the edges.
- Exercise 05 (misuse personas) uses the external-entities list.
- Exercise 06 (PbD checkpoints) gates work the DFD enforces.

Keep the DFD and tables in a repository (same repo as the system if possible). Version the file. Date it. You will update it.
