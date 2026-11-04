# The Boundary to `ai-risk-engineer` (Level 25)

An AI Privacy Engineer (L30, AI Governance family) and an AI Risk Engineer (L25, same family, peer) share the same floor on the ladder but *different* surfaces. This chapter draws the boundary sharply. If you cannot cite the boundary in a review, you will either duplicate the peer's work or leave gaps that neither of you owns.

mod-101 chapter 08 lays out the full governance ladder. This chapter zooms in on the one boundary you will cite most often from this module: the privacy threat model (you) vs. the broader AI harm model (`ai-risk-engineer`).

## Why the boundary is often contested

`ai-risk-engineer` and `ai-privacy-engineer` sit at adjacent levels in the same family. Both run threat-model-shape exercises. Both touch LINDDUN-adjacent vocabulary (misuse personas, red-team, control routing). Both feed into DPIA / Annex IV bundles for AI systems. A reader who has only seen one of the two will blur them.

The team-level consequence of blurring: either (a) the two role-holders argue over ownership and ship nothing, or (b) one of them silently drops the surface. Both failure modes are common.

The resolution is not "privacy is a subset of risk" or "risk is a subset of privacy". The two are **related but distinct depth areas**, and both can be excellent and complete simultaneously.

## What `ai-risk-engineer` owns — the broader harm surface

The AI Risk Engineer owns the hands-on AI *risk-engineering* craft end-to-end. Their scope includes:

- **Harm modelling beyond privacy** — the full catalogue of AI-produced harms: misinformation, misuse-enabled harms, operator-caused harms, model-caused harms, data-caused harms, integration-caused harms.
- **Misuse personas across the capability surface** — prompt-injection abusers, jailbreak operators, capability-uplift seekers, agent-abuse operators, model-exfiltration-for-IP actors, mass-manipulation campaigns.
- **LLM / agent red-team** — adversarial prompt engineering, tool-use misuse, tool-chain exploitation.
- **Adversarial ML beyond privacy** — evasion (adversarial examples), model extraction as IP / availability attack, poisoning as integrity attack, backdoors.
- **Fairness / explainability engineering** — bias evaluation, demographic parity / equalised-odds metrics, explainability stack.
- **Model risk management (MRM)** — the finance-regulated version of model governance.
- **Supply-chain risk** — the SBOM / ML-BOM surface with an *attack-surface* lens; model-weights provenance; pre-training-data provenance for *integrity*, not just lawful basis.
- The **mod-107 privacy-risk-engineering slice** — working-depth MIA / LiRA / DP-SGD / Presidio coverage that is the baseline this privacy track deepens.

The AI Risk Engineer's threat model has a *broader surface* and *shallower depth per topic*. They model all of the above; the depth per topic is working-level, not expert-level.

## What this role (`ai-privacy-engineer`) owns — the privacy-specific focused deep dive

The AI Privacy Engineer owns the privacy-engineering craft end-to-end as a focused deep dive. From mod-101 chapter 08 and this module:

- **Privacy threat modelling for AI** — LINDDUN GO + MAESTRO + PRAM at *regulator-grade depth* (this module).
- **Data governance and minimisation** — Datasheets, consent / legal-basis propagation, vaulting / tokenisation, synthetic-data substitutes, CycloneDX ML-BOM with a *consent-and-provenance* lens (mod-103).
- **PII / PHI / PCI pipelines at production scale** — Presidio + benchmarking vs. Google DLP / AWS Comprehend PII / Azure AI Content Safety / Private AI; HIPAA Safe Harbor + Expert Determination; multimodal PII; tabular de-identification with ARX / sdcMicro (mod-104).
- **Memorisation / extraction / MIA risk engineering** — shadow-model MIA, Carlini LiRA, extraction + canary methodology, model inversion, ML Privacy Meter (mod-105) — deeper than the peer's mod-107 slice.
- **DP engineering** — DP-SGD, moments / RDP / zCDP / GDP accountants, PATE, DP synthetic data; Opacus, TensorFlow Privacy, OpenDP SmartNoise, Google DP; NIST SP 800-226 (mod-106) — deeper than the peer's slice.
- **Federated + secure aggregation + DP-FL** — Flower / TFF / FLARE / PySyft; cross-device vs. cross-silo; Bonawitz; DLG / iDLG defences (mod-107).
- **Confidential computing + PP inference** — TEE (SGX / SEV-SNP / H100 CC / hyperscaler offerings), HE (TenSEAL / SEAL), MPC (CrypTen / MP-SPDZ), Apple PCC as published blueprint (mod-108).
- **RAG privacy + agent-tool data scoping** — per-tenant retrieval ACL, prompt / completion logging policy, agent-tool data-boundary review (mod-109).
- **Machine unlearning + DSR engineering** — SISA / approximate / LLM-unlearning, TOFU benchmark; rights-handling for AI systems (mod-110).
- **DPIA + audit-grade privacy evidence** — Art. 35, UK ICO shape, PRAM, ISO/IEC 29134, EDPB Opinion 28/2024, CNIL AI sheets; regulator-facing bundle (mod-111).
- **Cross-functional AI privacy program** — DPO / legal / security / product coordination; privacy slice of model + dataset cards; AI-privacy incident response (mod-112).

The AI Privacy Engineer's threat model has a *focused surface* (privacy-specific) and *deeper depth per topic*. They model the privacy-relevant subset of harms; depth per topic is expert-level, cite-able, and defensible to a regulator.

## The shared surface — and how to divide it

Several surfaces are genuinely *shared*. A deferral contract per surface:

### Shared surface 1 — Memorisation / extraction / MIA

- **`ai-risk-engineer`** — working-depth defender of weights as an *IP* and *integrity* asset; runs LiRA / MIA at baseline competence.
- **`ai-privacy-engineer`** (you) — expert-depth owner of memorisation / extraction *as a privacy threat*; owns canary methodology, DP-SGD tuning, release-gated extraction testing, DPIA Section on model-side privacy risks.
- **Contract**: you lead; the peer's baseline is the fallback for coverage when you are not available.

### Shared surface 2 — Agent / RAG safety

- **`ai-risk-engineer`** — agent red-team for capability misuse, tool-chain exploitation, jailbreak / injection, availability.
- **`ai-privacy-engineer`** — agent data boundaries; tool-call telemetry linkability; retrieval ACL; prompt / completion log policy; cross-tenant RAG leak.
- **Contract**: both cover; cite each other's model.

### Shared surface 3 — Supply chain + ML-BOM

- **`ai-risk-engineer`** — supplier integrity, tampered weights, pre-training data poisoning, dependency CVEs.
- **`ai-privacy-engineer`** — supplier consent / lawful basis, special-category screening, dataset-card provenance, ML-BOM as a *consent-and-provenance* instrument.
- **Contract**: one ML-BOM; two overlapping lenses on its contents.

### Shared surface 4 — DPIA / Annex IV

- **`ai-risk-engineer`** — contributes harm-surface analysis, bias evaluation, adversarial-robustness evaluation.
- **`ai-privacy-engineer`** — authors the DPIA (mod-111); contributes privacy threat model, PRAM register, PbD checkpoint register.
- **Contract**: the DPIA is *your* artifact; the risk peer is a named contributor for the sections they own.

### Shared surface 5 — Model + dataset cards

- **`ai-risk-engineer`** — bias / robustness / explainability sections.
- **`ai-privacy-engineer`** — privacy-control, data-provenance, Art. 22 meaningful-information, special-category-handling sections.
- **Contract**: one card; two sign-offs.

## What each role defers *down*, *sideways*, and *up*

(Short version — mod-101 chapter 08 has the full map.)

Down:

- Both roles defer operational analyst work (ROPA, intake-questionnaire, framework crosswalk, jurisdictional tracking) to **`ai-governance-analyst`** (L15).
- Both roles assume the **`ml-engineer`** (L20) ML fundamentals.

Sideways:

- **`ai-infra-mlops`** (L25, Infra family) — gate / platform contract for release-time controls; delivery mechanism for CI lints, release gates, runtime monitors.
- **`ai-eval-engineer`** (L30, AI Engineering) — application-layer eval / trace / judge depth.
- **`model-evaluation-engineer`** (L30, ML Engineering) — statistical calibration on MIA / DP utility deltas.

Up:

- **`ai-evaluation-engineer`** (L35, Governance) — release-assurance; Annex IV *packaging* into regulator-facing bundle.
- **`senior-ai-governance-architect`** (L50) — control-library architecture, policy taxonomy, cross-jurisdiction reconciliation.
- **`head-of-ai-governance`** (L60) — program leadership, board reporting, regulator engagement.

Out of family:

- **DPO / legal counsel** — legal opinion, statutory DPO duties, formal regulator interaction.
- **Security / AI infra security** (L35) — deep ML / AI *security* at platform scale.
- **Security operations / DFIR** — breach handling.

## A sentence you should be able to say in a review

> "For this system, I'm leading the LINDDUN / PRAM privacy threat model, the memorisation / extraction testing, the DP configuration, the DSR surface, the PbD checkpoints, and the DPIA. `ai-risk-engineer` is leading the broader harm model, the agent / jailbreak red-team, the fairness evaluation, and the MRM review. Our registers cross-reference by ID. Here is the shared surface deferral contract we committed to for this release."

If you cannot say that in a review, re-read this chapter until you can.

## Summary — what you can do after this chapter

- Draw the boundary between this role and `ai-risk-engineer` on five concrete shared surfaces.
- State what each role owns independently, what they defer down / sideways / up, and what is genuinely shared.
- Avoid the two team-level failure modes — duplication and silent gaps.
- Cite the mod-101 chapter 08 ladder for any role question beyond the privacy / risk pair.

That closes the module. The exercises take the material from here to a hands-on threat-model artifact of your own; `mod-103` picks up data governance and minimisation.
