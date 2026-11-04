# The Boundary to `ai-risk-engineer` on Memorisation and MIA

Across the privacy track, each module ends with a short chapter marking the coordination boundary with the peer role at the next level down. mod-102 drew it on the privacy threat model; mod-103 drew it on data governance; mod-104 drew it on PII detection. This chapter draws it on membership inference, memorisation, and extraction — the module surface where the depth gap between `ai-privacy-engineer` (L30) and `ai-risk-engineer` (L25) is sharpest and the most likely to produce duplicated or contradictory measurement claims.

mod-101 chapter 08 has the full ladder. The short version: `ai-risk-engineer` is a level-25 engineer in the same AI Governance family, one level below `ai-privacy-engineer` (level 30). Their mod-107 covers the privacy-risk-engineering slice at working depth — enough to run a Shokri or LiRA attack on a toy model, prompt a GPT-2 for memorisation, and report the AUC for the broader risk register. Your module (mod-105) deepens on production methodology, per-attack configuration, defender triage against (ε, δ), ML Privacy Meter as the standing harness, and the regulator-defensible coverage story.

## What the peer owns at working depth

The `ai-risk-engineer` role (see that track's mod-107 — titled similarly, "membership inference, memorisation, model extraction" in that curriculum) covers:

- **Textbook Shokri shadow-model MIA on a toy classifier.** CIFAR-10 / Purchase100 / Texas Hospital Stay — enough to stand up shadow models, build the attack classifier, report the attack AUC.
- **LiRA at paper-tutorial depth.** Enough to explain the Carlini-2022 improvement over Shokri; enough to produce a log-log ROC on the same toy target. Shadow-count is small (k = 10), the attack is the offline variant, the per-example variance is pooled by default.
- **Carlini-2021 extraction on GPT-2 small.** Reproducing the published methodology on a small open model with a published training-set subset. Reports a sample of extractable strings.
- **Awareness of ML Privacy Meter.** Knows the tool exists and that its output is what a privacy engineer consumes; does not usually operate it as a standing harness.
- **MIA-as-one-threat in a broader risk register.** MIA / memorisation is one scenario among many (prompt injection, jailbreak, capability uplift, poisoning, model extraction, availability) that the broader risk model covers.

The depth is calibrated to be *sufficient* for the risk-engineer's broader coverage. It is not sufficient for your regulator-facing artifact.

## What this role owns at expert depth

Your mod-105 goes deeper on the same surface in five places:

- **Production MIA methodology.** Chapter 02's shadow discipline + chapter 03's LiRA at k = 64, online variant, query-with-augmentation, per-example variance with pooled fallback, TPR@FPR triple reporting at (10⁻², 10⁻³, 10⁻⁴), CI handoff to `model-evaluation-engineer`. The peer may run the attack; you ship the methodology the regulator accepts.
- **The low-FPR-regime discipline.** Chapter 03's whole point. The peer may report AUC; you report the Carlini-2022 numbers that are actually privacy-relevant. If the DPIA quotes an AUC-only number from the peer's register, you re-measure with TPR@FPR and update.
- **Canary design and cross-corpus FP defence.** Chapter 06 entirely. The peer's mod-107 does not touch this; it assumes memorisation is measured by direct-extraction on known-sensitive strings. Your canaries are the controlled-experiment version.
- **Defender triage.** Chapter 07's table. The peer may know DP-SGD and dedup exist; you quantify the Δ-per-mitigation across the attack suite with measured numbers, document the rejected rows, and sign the residual-risk claim.
- **The standing harness.** Chapter 08's ML Privacy Meter operation — target-model wrapping, pinned config, provenance record, release-gate integration, CI handoff. The peer runs a one-off; you own the harness.
- **The coverage story.** Chapter 09's NIST AI 100-2 + MITRE ATLAS + OWASP LLM02 mapping. The peer's mod-107 may name the standards informally; you ship the row-level mapping as a versioned artefact.

The depth is calibrated to produce a DPIA-citeable, regulator-defensible artifact.

## The shared surface — and how to divide it

Three surfaces where both roles overlap and need an explicit contract:

### Shared surface A — First MIA on a new target

Both roles can set up a Shokri or LiRA attack on a new target. The question is who owns it post-measurement.

- **Peer (`ai-risk-engineer`)** — may do the first measurement if embedded on the product team and your availability is constrained, or if the measurement is a quick diagnostic in the broader risk register.
- **You (`ai-privacy-engineer`)** — own the standing audit. Your ML Privacy Meter configuration, your pinned shadow count, your TPR@FPR gates, your defender triage, your DPIA citation.
- **Contract** — the peer may *do* the first attack as a diagnostic; you *inherit* the audit configuration and ownership from first release-gate. Record the handoff in the audit-config artefact.

### Shared surface B — Extraction / memorisation on a new LLM

Both roles can prompt a new LLM for memorisation.

- **Peer** — may do the first probing as part of the broader risk-register coverage (memorisation-as-one-threat).
- **You** — own the canary-controlled measurement, the extraction rate reporting, the cross-corpus FP defence, the DPIA citation.
- **Contract** — the peer's probing informs the risk register; your canary + dose-response is the *measurement*. When the two disagree, the measurement wins and the risk register is updated.

### Shared surface C — Response to a memorisation / extraction incident

A production leak via model-emitted sensitive content is both a privacy incident and a general-risk incident.

- **Peer** — contributes to the broader incident-response coordination; may own the containment of a jailbreak-caused leak.
- **You** — own the privacy-side root cause analysis, the DPIA update, the regulator notification content (if applicable; mod-111), the pipeline fix, the re-measurement, the canary-set augmentation.
- **Contract** — one incident commander; two overlapping investigations. The DPIA update is your artifact; the broader incident report is theirs.

## What each role defers *up* and *sideways*

Up:

- **`model-evaluation-engineer`** (peer, L30, AI Governance). Owns the CI methodology on MIA AUC / TPR@FPR / extraction rate. Your harness emits raw per-example scores; their methodology produces the CI for the DPIA. Chapter 08 names the handoff.
- **`ai-evaluation-engineer`** (L35) packages the memorisation/MIA artifacts into the release-assurance bundle. Your audit directory + DPIA inputs land in their bundle.
- **`senior-ai-governance-architect`** (L50) owns the control taxonomy; a new attack class joining the catalogue is a taxonomy question at their level.

Sideways:

- **`ai-infra-mlops`** (L25, Infra family) — operates the audit cluster; owns the SLOs on the shadow-model training infrastructure.
- **`training-pipeline-engineer`** (L25) — integrates the canary insertion into the pre-training / fine-tune pipeline; owns the actual ETL.
- **`ai-security-engineer`** — the model-extraction / weight-stealing attack surface (chapter 09 boundary). Also operates the production egress filter (chapter 07 lever 5).
- **`rag-engineer`** / **`llm-application-developer`** (L25) — integrate the egress filter on completion; own the latency / cost budget around it.

Down:

- **`ai-governance-analyst`** (L15) — keeps the attack-coverage matrix versioned, tracks the peer methodology version in use, maintains the canary set's handling.

Out of family:

- **DPO / legal** — consume the defender triage's residual-risk claim and sign it.
- **Security engineering** — audits the canary-set storage (sensitive artefact — leaked canaries invalidate the measurement).

## A sentence you should be able to say in a review

> "For this model, I'm leading the privacy-residual-risk measurement — Shokri + LiRA at k=64 (chapter 03), attribute inference + Fredrikson inversion (chapter 04), Carlini-family extraction with the canary methodology from chapter 06, running under ML Privacy Meter as the standing harness (chapter 08). The defender triage (chapter 07) gave us DP-SGD at ε=8 + dedup + output filter as the chosen set; the residual LiRA TPR@10⁻³ is 0.02 with 95% CI from `model-evaluation-engineer`'s bootstrap. The coverage mapping (chapter 09) is the NIST AI 100-2 + ATLAS + OWASP LLM02 row-table. `ai-risk-engineer` is covering memorisation as one scenario in the broader risk register; their measurement is the fast-diagnostic version and defers to this one when they disagree. Here is the contract for which of us owns which part of the response to a memorisation-shaped incident."

If you cannot say that in a review, re-read this chapter and chapter 07 until you can.

## What happens when the boundary is wrong

Two team-level failure modes — the same two as mod-102 chapter 09, mod-103 chapter 08, and mod-104 chapter 10.

### Failure mode A — duplication

Both roles maintain separate attack harnesses, run on separate target checkpoints, with separate shadow-model populations. Two divergent AUC numbers; one favours the broader risk register's position, one favours the DPIA's. The product team picks whichever answer arrives first; the regulator sees a documentation chain with inconsistencies.

Prevent with: shared target-model wrapper + shared canary set + one audit configuration. The peer's mod-107 runs with *your* ML Privacy Meter config, not a parallel one.

### Failure mode B — silent gap

Both roles assume the other is covering LiRA / attribute inference / extraction / canary design. Neither does. The product ships with "we measured MIA" that turns out to be an AUC-only Shokri number on a checkpoint from six months ago, no TPR@FPR, no canaries, no triage table.

Prevent with: an explicit ownership column in the coverage matrix (chapter 09). Every attack row has one name. Every artifact (harness, canary set, triage table, coverage matrix) has one name.

## Summary — what you can do after this chapter

- Describe `ai-risk-engineer`'s mod-107 MIA/memorisation coverage at working depth, and your mod-105's six depth areas beyond it.
- Divide the three shared surfaces (first MIA, extraction on new LLM, incident response) with an explicit contract.
- Articulate what this role defers up / sideways / down on the memorisation / MIA surface, including the critical peer handoff to `model-evaluation-engineer` for CI methodology.
- Say the one-sentence review line that names the boundary explicitly.
- Recognise the two failure modes (duplication, silent gap) and the mechanisms that prevent each.

That closes the module. The exercises take the material from here to a reusable measurement bundle. mod-106 picks up the companion half — differential privacy engineering — where the measured residuals from this module are the thing DP is tuned to bound.
