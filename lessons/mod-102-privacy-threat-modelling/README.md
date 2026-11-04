# mod-102-privacy-threat-modelling: Privacy Threat Modelling for AI Systems

**Estimated effort:** 14 hours

This module teaches the privacy threat model as an *engineering artifact* — LINDDUN GO for workshop-speed assessment, LINDDUN MAESTRO for depth, NIST PRAM for prioritisation, misuse-persona routing to controls, and Privacy-by-Design engineering checkpoints. The output is the threat-model + risk-register + checkpoint-register bundle a DPIA (mod-111) will consume.

## Learning objectives

- Author a LINDDUN-shape privacy threat model for an AI system — Linkability, Identifiability, Non-repudiation, Detectability, Disclosure, Unawareness, Non-compliance — using LINDDUN GO for workshop-speed assessment and LINDDUN MAESTRO where the depth is warranted.
- Decompose the AI system into a privacy-relevant data-flow diagram covering training corpora, retrieval corpora, inference prompts, generated outputs, prompt / completion logs, model weights, embeddings, caches, and telemetry — and enumerate the privacy threats at each stage and transition.
- Enumerate misuse personas against privacy — adversarial user attempting re-identification, insider attempting training-data extraction, upstream data supplier violating consent, downstream integrator accumulating linkable traces — and route each to a privacy control.
- Perform a NIST PRAM-shape privacy risk assessment on top of the LINDDUN model — identify data actions, data processing ecosystem, problematic data actions, and privacy risks, then score likelihood × impact on a defensible scheme.
- Operationalise Privacy by Design (the Cavoukian seven principles that GDPR Art. 25 codifies) as engineering checkpoints — proactive not reactive, privacy as default, privacy embedded into design, full functionality (positive-sum, not zero-sum), end-to-end security, visibility and transparency, respect for user privacy.
- Cite the boundary to `ai-risk-engineer` (peer, level 25) — that role owns the broader harm-model surface (misuse personas, capability surfaces, operator / model / data / integration-caused harms); this role owns the privacy-specific threat model as a focused deep dive.

## Chapters

1. [Why privacy threat modelling for AI needs its own shape](./01-why-privacy-threat-modelling-for-ai.md) — motivation, security-vs-privacy framing, the three-layer artifact this module produces.
2. [Decomposing an AI system into a privacy-relevant DFD](./02-privacy-data-flow-decomposition.md) — the nine-sink template (training / retrieval / prompts / outputs / logs / weights / embeddings / caches / telemetry) and how to label edges and trust boundaries.
3. [The seven LINDDUN categories, read for AI systems](./03-linddun-categories-for-ai-systems.md) — L, I, N-rep, D1, D2, U, N-comp with example threats per DFD element.
4. [LINDDUN GO — the workshop-speed privacy threat model](./04-linddun-go-workshop.md) — the time-boxed workshop playbook for AI systems.
5. [LINDDUN MAESTRO — when the depth is warranted](./05-linddun-maestro-deep-dive.md) — per-element completeness matrix, threat-tree refinement, controls linked to code.
6. [NIST PRAM — risk-assessing the LINDDUN model](./06-nist-pram-overlay.md) — data actions, problematic data actions, likelihood × impact anchors, residual-risk register.
7. [Misuse personas and control routing](./07-misuse-personas-and-control-routing.md) — four canonical personas and the routing table to curriculum control families.
8. [Privacy by Design as engineering checkpoints](./08-privacy-by-design-checkpoints.md) — Cavoukian's seven principles as CI lints, release gates, and runtime monitors.
9. [The boundary to `ai-risk-engineer` (level 25)](./09-boundary-to-ai-risk-engineer.md) — the deferral contract on the shared surface.

## Exercises

Each is a hands-on build; outputs stack into a reusable threat-model bundle you will carry through the rest of the track.

1. [Privacy data flow decomposition for one AI system](./exercises/exercise-01-privacy-data-flow-decomposition-for-one-ai-system.md)
2. [LINDDUN GO workshop walkthrough](./exercises/exercise-02-linddun-go-workshop-walkthrough.md)
3. [LINDDUN MAESTRO deep dive on one flow](./exercises/exercise-03-linddun-maestro-deep-dive-on-one-flow.md)
4. [NIST PRAM overlay on the LINDDUN model](./exercises/exercise-04-nist-pram-overlay-on-the-linddun-model.md)
5. [Misuse persona and privacy control routing](./exercises/exercise-05-misuse-persona-and-privacy-control-routing.md)
6. [Privacy by Design checkpoint set](./exercises/exercise-06-privacy-by-design-checkpoint-set.md)

## Structure

- `01-…md` … `09-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
