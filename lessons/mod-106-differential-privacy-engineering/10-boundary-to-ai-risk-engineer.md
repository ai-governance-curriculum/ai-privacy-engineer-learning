# Boundary to `ai-risk-engineer` — DP at Working Depth vs. Production Depth

Two tracks in this curriculum cover differential privacy. `ai-risk-engineer` (peer, level 25) mod-107 teaches DP at *working* depth; this track's mod-106 teaches DP at *production* depth. The chapters are written on the assumption that the reader has mod-107's coverage already. This chapter draws the line between the two tracks explicitly, so that a privacy engineer stepping onto a project with risk-engineer counterparts can hand off cleanly rather than redo their work.

The boundary pattern mirrors the same split in mod-105 (chapter 10 there). The two roles are peers. Neither is "senior" to the other; they are specialised.

## What `ai-risk-engineer` mod-107 covers

The risk-engineer track's DP module is designed to be the privacy section of a *generalist* AI risk engineer's toolkit. Its scope:

- **Read Dwork & Roth's monograph at working depth.** Enough to recognise the (ε, δ)-DP definition, the Gaussian mechanism, and the privacy amplification by subsampling. Not the smoothed-sensitivity machinery or the PRV accountant internals.
- **Train a toy model with Opacus DP-SGD.** Follow a tutorial; see the accountant work; report a final ε. Enough to answer "did we DP-train this?" with evidence rather than a guess.
- **Recognise the common engineering failure modes.** BN incompatibility, Poisson vs. shuffled sampling. Enough to flag them in review; not necessarily enough to resolve subtle cases alone.
- **Quote the (ε, δ) number in a risk register.** The RMF-aligned risk entry says "DP-SGD at ε = {value}" and links to the engineering team's artefacts.
- **Participate in a DP-SGD code review.** Enough background to ask the right questions; the final engineering sign-off rests with the privacy engineer.

What the risk engineer does **not** own:

- **The choice of accountant.** The risk engineer may know that RDP and PRV exist; they do not own the production trade-off.
- **The (ε, δ) policy decision.** The risk engineer flags that an ε needs to be picked; the policy conversation is the privacy engineer's.
- **PATE engineering.** Beyond awareness that it exists as an alternative to DP-SGD.
- **DP synthetic data generator engineering.** Beyond awareness that SmartNoise Synth exists.
- **DP analytics deployment via PipelineDP / SmartNoise.** Beyond awareness that the pattern exists.
- **The release package.** The risk engineer references the privacy engineer's release package in the risk register; they do not write it.
- **The regulator-defence narrative on ε.** Chapter 05 is privacy-engineer scope. The risk engineer participates in the review, but the defence is the privacy engineer's.
- **The NIST SP 800-226 section mapping.** The risk engineer knows the document exists; the privacy engineer owns the per-section compliance claim.

## What this module (mod-106) owns beyond that

Each chapter adds something the risk-engineer track does not cover.

- **Chapter 01 (DP foundations).** The accountant family tree — RDP, zCDP, GDP, f-DP — and when each is tight. The risk engineer knows (ε, δ)-DP; the privacy engineer knows the four refinements and can pick one.
- **Chapter 02 (accountants and library mapping).** The Opacus / TF-Privacy / Google-DP / OpenDP accountant map. Cross-check methodology. The audit log as a required artefact.
- **Chapter 03 (DP-SGD with Opacus).** Production-grade concerns: Poisson sampling, virtual batching, the seven engineering failure modes, state-of-the-art-recipe knowledge (large batches, pre-train + fine-tune, GroupNorm).
- **Chapter 04 (DP-SGD with TensorFlow Privacy).** The second framework; the Keras-specific trap set.
- **Chapter 05 (defending ε under audit).** The four regulator questions, the anchor table (Apple / Google / Census), the threat-model note, lifetime-vs-per-release composition.
- **Chapter 06 (PATE engineering).** Teacher ensembles, Confident-GNMax, data-dependent vs data-independent analysis. The whole mechanism family.
- **Chapter 07 (DP synthetic data).** The generator families (DP-GAN, PATE-GAN, DP-CTGAN, DP-WGAN, Private-PGM, AIM), SmartNoise Synth, the three-tier utility audit, the Groundhog Day warning.
- **Chapter 08 (DP analytics).** PipelineDP and SmartNoise Analytics, contribution bounding, partition selection, the standing-dashboard composition problem.
- **Chapter 09 (release package and NIST SP 800-226).** The six-file release package, the section mapping, the linter, the sign-off workflow.

The production depth is in the engineering details and in the audit story. The risk engineer can point at the final artefact; the privacy engineer builds it and defends it.

## When the two roles collaborate

Three recurring handoffs on a real project.

### Handoff 1 — risk register entry needs an ε

The risk engineer is working through an AI RMF entry for a new model deployment. The entry has a "privacy risk" section that requires a DP claim.

- Risk engineer: "We need an (ε, δ) for the training pipeline. What's your target?"
- Privacy engineer: "ε = 4 is the current plan. Utility sweep in {utility-eval.md}; defence narrative in {claim.md}; attack corroboration in mod-105 LiRA report. Please link to all three from the risk register."
- Risk engineer: pastes the link, moves on. The RMF entry is complete and auditable.

### Handoff 2 — pre-release code review of a DP-SGD training pipeline

The risk engineer is a required reviewer on the training-pipeline PR. They read engineering-attestations.md.

- Risk engineer notices `[ ]` next to "accountant state saved on checkpoint". Blocks the PR.
- Privacy engineer: "Right, added in follow-up commit. CI test X verifies."
- Risk engineer: re-reviews, approves.

The risk engineer's role is a *sanity filter*, not the primary defender. The attestation list is theirs to scan against.

### Handoff 3 — an incident involving the model

A security team reports a potential extraction incident from the model.

- Risk engineer coordinates the response, pulls the release package.
- Privacy engineer: evaluates whether the incident is within the DP claim's scope (yes: the empirical attack succeeded worse than the ε bound predicted; this is a correctness incident) or outside (no: the incident is a prompt-injection, an orthogonal threat class).
- Jointly: update the threat-model.md and claim.md if needed; add to the DPIA (mod-111) and the risk register.

The incident response is joint; the diagnosis of "is this a DP-mechanism bug or a different class of problem?" belongs to the privacy engineer.

## The one-paragraph version for a project kickoff

When you join a project where both roles are staffed, say this at the kickoff:

> "I'm the privacy engineer; I own the DP mechanism design, the accountant choice, the (ε, δ) policy decision, the release package, and the regulator-defence narrative. {Risk-engineer name} is the risk engineer; they own the AI RMF risk register entry that references my DP claim, and they sit on the code review as a sanity filter. We expect three handoffs: the RMF entry needs my ε, pre-release reviews need their sign-off on my attestations, and incidents go to both of us with me owning the mechanism-bug diagnosis."

That paragraph is what prevents two engineers from both thinking the other owns the final ε decision.

## Reading list a risk-engineer-only reader should pick up to close the gap

If a risk engineer wants to move toward privacy-engineer depth for a particular project, the reading order:

1. **Chapter 01** of this module — accountant family vocabulary.
2. **Chapter 05** of this module — regulator defence.
3. **Chapter 09** of this module — the release package specification.
4. **NIST SP 800-226** end-to-end.
5. **Dwork-Kohli-Mulligan** "Expose Your Epsilons!" in full.

That is roughly the minimum before the risk engineer can stand in as a backup for the privacy engineer on a specific release. The full module is the production-depth version.

## Summary — what you can do after this chapter

- Name the four things `ai-risk-engineer` mod-107 covers (working-depth Opacus, risk-register integration, failure-mode awareness, review participation).
- Name the eight areas mod-106 adds beyond mod-107 (accountant choice, library mapping, PATE, DP synth, DP analytics, ε defence, release package, NIST SP 800-226 mapping).
- Articulate the three recurring handoffs (RMF entry, pre-release review, incident).
- Deliver the kickoff paragraph that prevents the "who owns ε?" collision.
- Hand a risk engineer the five-item reading list that closes the gap for a specific project.

The module is complete. The exercises operationalise each chapter; the resources.md collects the module's primary references.
