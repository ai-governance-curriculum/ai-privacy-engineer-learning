# The Boundary to `ai-risk-engineer` on PII Detection

Across the privacy track, each module ends with a short chapter marking the coordination boundary with the peer role at the next level down. mod-102 drew it on the privacy threat model; mod-103 drew it on data governance. This chapter draws it on PII detection, which is the single surface most likely to end up double-owned (or silently un-owned) between the two roles.

mod-101 chapter 08 has the full ladder. The short version: `ai-risk-engineer` is a level-25 engineer in the same AI Governance family, one level below `ai-privacy-engineer` (level 30). Their mod-107 covers the privacy-risk-engineering slice at working depth — enough to configure Presidio, redact a prompt, run a labelled eval on a toy set. Your module (mod-104) deepens on multimodal detection, de-identification mathematics, managed-service comparison, HIPAA Safe Harbor / Expert Determination, and the integration contract across the full AI data path.

## What the peer owns at working depth

The `ai-risk-engineer` role (see that track's mod-107) covers:

- **Baseline Presidio integration.** Setting up an analyzer with the default recogniser catalogue; wiring `anonymizer` into a prompt path or a log sink; registering one or two pattern-based custom recognisers.
- **Working-depth labelled eval.** A small labelled eval set (hundreds of samples), overall F1, per-entity F1 as a secondary metric.
- **Awareness of the managed alternatives.** Knows Google DLP / AWS Comprehend / Azure Language PII exist and roughly what they cost; does not usually author the build-vs-buy matrix.
- **PII-as-one-threat in a broader risk model.** PII leak is one threat among many (jailbreak, prompt-injection, capability uplift, agent misuse, availability, IP leak) that the broader AI-risk model covers.

The depth is calibrated to be sufficient for the risk-engineer's broader coverage. It is not sufficient for the privacy engineer's regulator-facing artifact.

## What this role owns at expert depth

Your mod-104 goes deeper on the same surface in five places:

- **Custom recogniser engineering.** Chapters 02-03's full depth: pattern + deny-list + context + NER-fine-tune + remote-recogniser; checksum validators; per-language catalogues; CI'd regex surfaces. The peer builds a couple; you build the catalogue and version it like an API.
- **Evaluation discipline.** Chapter 04's labelled eval + red-team set + per-entity thresholds + latency / cost / throughput measurement. The peer measures F1; you measure per-entity recall against a floor, you ship the eval set itself as an artifact, and the CI gates block on per-entity regressions.
- **Managed-service build-vs-buy matrix.** Chapter 05's full methodology: fair-benchmark harness, entity-mapping table, per-provider measurements, Shape A / B / C decision with the matrix as defence. The peer knows these services exist; you author the procurement-grade comparison.
- **Multimodal.** Chapter 06 entirely. The peer's mod-107 does not cover face / voice / OCR / UI-screenshot detection; your scope does.
- **De-identification mathematics.** Chapters 07-08 — k-anonymity, l-diversity, t-closeness, ARX, sdcMicro, HIPAA Safe Harbor, Expert Determination, statistical-disclosure control. The peer does not touch this surface.
- **Full integration contract.** Chapter 09's per-choke-point contract, with latency budgets, operator per choke-point, FP / FN measurement across all five. The peer integrates one or two choke-points; you own the contract across all five.

The depth is calibrated to produce a DPIA-citeable, regulator-defensible artifact.

## The shared surface — and how to divide it

Two surfaces where both roles overlap and need an explicit contract:

### Shared surface A — Baseline Presidio integration in a new product

Both roles can set up a Presidio analyzer in a new product. The question is who owns it post-integration.

- **Peer (`ai-risk-engineer`)** — may do the first wiring if they are embedded on the product team and your availability is constrained.
- **You (`ai-privacy-engineer`)** — own the standing configuration: the recogniser catalogue, the per-choke-point operator, the eval set, the release gate.
- **Contract** — the peer may *stand up* the integration; you *inherit* the configuration and the eval ownership from first release. Record the handoff in the integration contract (chapter 09).

### Shared surface B — Incident response for a PII leak

A production PII leak via log / prompt / completion is both a privacy incident and a general-risk incident.

- **Peer** — contributes to the broader incident-response coordination; may own the containment of a prompt-injection-caused leak.
- **You** — own the privacy-side root cause analysis, the DPIA update, the regulator notification content (if applicable; mod-111), the pipeline fix, the recogniser update, the labelled-eval regression.
- **Contract** — one incident commander; two overlapping investigations. The DPIA update is your artifact; the broader incident report is theirs.

## What each role defers *up* and *sideways*

Up:

- **`ai-evaluation-engineer`** (L35) packages the PII pipeline's artifacts into the release-assurance bundle. Your eval YAML (chapter 04) and integration contract (chapter 09) land in their bundle.
- **`senior-ai-governance-architect`** (L50) owns the control taxonomy; a new PII recogniser type joining the catalogue is a taxonomy question at their level.

Sideways:

- **`ai-infra-mlops`** (L25, Infra family) — operates the detector service; owns the SLOs; co-owns the latency budget. The contract is "I write the configuration; you run the pod".
- **`ai-eval-engineer`** (L30, AI Engineering) — application-layer evaluation. PII FP / FN on real traffic is on their dashboard too.
- **`training-pipeline-engineer`** (L25) — integrates your ingest-side pipeline; owns the actual ETL.
- **`rag-engineer`** / **`llm-application-developer`** (L25) — integrate your retrieval-side / prompt-side / output-side; own the orchestrator.

Down:

- **`ai-governance-analyst`** (L15) — keeps the recogniser catalogue versioned, tracks the HIPAA attestation across releases, maintains the processor list for managed services in use.

Out of family:

- **DPO / legal** — Expert Determination engagements are usually sourced by legal, often via an external firm. You brief the expert; legal contracts them; the written report comes back to both.
- **Security engineering** — Vault + detokenise audit, KMS, HSM. The vault is shared ownership.
- **Clinical / domain experts** — for HIPAA Expert Determination (chapter 08), a domain-knowledgeable person is part of the attacker-model definition. Record their involvement.

## A sentence you should be able to say in a review

> "For this system, I'm leading the PII pipeline design — Presidio configuration + custom recognisers + managed-service hybrid (chapter 05 matrix) + multimodal (chapter 06) + de-identification method for the tabular release (ARX; chapter 07) + HIPAA Expert Determination path (chapter 08). The per-choke-point integration contract (chapter 09) is the DPIA citation. `ai-risk-engineer` is covering PII-leak as one scenario in the broader risk register; they run a parallel eval against our shared labelled set. Here is the contract for which of us owns which part of the response to a PII-leak incident."

If you cannot say that in a review, re-read this chapter and chapter 09 until you can.

## What happens when the boundary is wrong

Two team-level failure modes — the same two as mod-102 chapter 09.

### Failure mode A — duplication

Both roles maintain Presidio configurations, labelled eval sets, and build-vs-buy opinions. The product has two sets of recognisers with different versions; two divergent evals; two contradictory procurement recommendations. The product team picks whichever answer arrives first. The regulator sees a documentation chain with inconsistencies.

Prevent with: shared recogniser catalogue + shared eval set + one DPIA citation. The peer's mod-107 work references *your* catalogue and eval, not a parallel one.

### Failure mode B — silent gap

Both roles assume the other is covering the multimodal surface / the tabular de-identification / the HIPAA attestation. Neither one does. The product ships with a face-detector that nobody has evaluated, a tabular release with k implicitly = 1, and a Safe Harbor claim that was never written down.

Prevent with: an explicit ownership column in the integration contract (chapter 09). Every choke-point has one name. Every artifact (eval set, build-vs-buy, de-identification report, HIPAA attestation) has one name.

## Summary — what you can do after this chapter

- Describe `ai-risk-engineer`'s mod-107 PII coverage at working depth, and your mod-104's five depth areas beyond it (custom recogniser engineering, eval discipline, build-vs-buy matrix, multimodal, de-identification mathematics / HIPAA, integration contract).
- Divide the two shared surfaces (baseline Presidio integration, incident response) with an explicit contract.
- Articulate what this role defers up / sideways / down on the PII surface.
- Say the one-sentence review line that names the boundary explicitly.
- Recognise the two failure modes (duplication, silent gap) and the mechanisms that prevent each.

That closes the module. The exercises take the material from here to a reusable PII-pipeline bundle. mod-105 picks up the companion half — memorisation / MIA engineering — where mod-104's detector is the preventive control and mod-105 is the residual-risk measurement.
