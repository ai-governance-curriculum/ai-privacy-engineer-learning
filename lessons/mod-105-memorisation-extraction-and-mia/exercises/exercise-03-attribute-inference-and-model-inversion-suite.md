# Exercise 03: Attribute Inference and Model Inversion Suite

**Estimated effort:** 3 hours · **Output:** `attribute_inversion/` directory with three target-specific reports — tabular, vision, small-LM — each with the attack implementation, measured fidelity, and a defensible-threshold decision · **Primary chapter:** 04

## Objective

Build the attribute-inference and Fredrikson-style model-inversion attacks against three target classes representative of what you will see in practice. The goal is to internalise the attacker-model separation between "infer missing attribute" and "reconstruct training input", to pick the right fidelity metric per modality, and to produce a report that commits to a defensible threshold instead of vibes.

## Prerequisites

- Chapter 04 (attribute inference + model inversion) read.
- Fredrikson, Jha, Ristenpart (CCS 2015) at least skimmed.
- Zhang et al. (CVPR 2020) GAN-prior inversion approach skimmed for the vision target.
- Working environment with:
  - PyTorch or TensorFlow for training small models.
  - A pretrained generative prior for the vision target (StyleGAN2 / StyleGAN3 / an open-weights diffusion model; a face-space prior if your vision target is a face classifier).
  - A small LM (e.g. GPT-2-small or similar ≤500M-parameter open model) for the small-LM target.

## Problem statement

Produce three target-specific reports. Each must name the attacker model (what they know, what they get), the chosen metric, the defensible threshold, and the measured value — with a documented "chosen or not? why?" row in the defender triage framing from chapter 07.

### Target A — Tabular attribute inference

Dataset: a tabular dataset with a known-sensitive attribute. Options:

- UCI Adult (predict income; sensitive attribute: a column like relationship / race / sex).
- Texas-100 Hospital Stay (predict length of stay; sensitive attributes are diagnosis / procedure codes).
- A domain tabular dataset you own with at least one labelled sensitive column.

Build the measurement:

1. Train a classifier on the dataset (classification target your choice).
2. Choose the sensitive attribute Z and the known-to-attacker attributes K.
3. Implement the **attacker-baseline predictor** — predict Z from K using only marginal / joint statistics from a *public* / *holdout* reference set the attacker is assumed to have.
4. Implement the **model-aided predictor** — for each record, query the classifier at all candidate completions (K, z') for z' ∈ Z; take the most likely z'.
5. Compute **lift = model-aided − attacker-baseline** on a held-out eval set.
6. **Stratify by membership.** For records in the classifier's training set vs. not. Report both lifts.

Defensible threshold: pick a lift threshold with a documented rationale (e.g. "lift > 0.05 on any demographic attribute is a release-blocker given the DPIA's residual-risk tolerance").

### Target B — Vision model inversion

Model: a small identity classifier. Recommended setup — a face-classification model on a reduced FaceScrub, LFW, or VGGFace subset (reduce to 20–100 identities so training is quick; use only identities with publicly-licensed images).

Build the measurement:

1. Train the identity classifier.
2. Implement **pixel-space Fredrikson inversion** (gradient-descent on input x̂ to maximise target-class logit, with small regulariser).
3. Implement **GAN-prior inversion** (optimise in the latent space z of a pretrained prior; reconstruct x̂ = G(z)).
4. For each target identity, run both and save the reconstructions.
5. Evaluate fidelity with three metrics:
   - SSIM / PSNR against a ground-truth training image of the identity.
   - **Face-recognition cross-verify** — run reconstruction through an *independent* face-recognition model; record the match probability to the target identity.
   - Qualitative visual grid for human inspection (small sample).
6. Document the representativeness-vs-subject-level distinction: do the reconstructions look like a *specific* training subject or like a *class-average* face? The cross-verify score answers this quantitatively.

Defensible threshold: pick a cross-verify threshold (e.g. "cross-verify > 0.7 to the actual training subject is a release-blocker"). Document the rationale.

### Target C — Small LM attribute probing

Model: GPT-2-small or similar (≤500M parameters) fine-tuned on a small corpus that contains named subjects with associated attributes you can look up. Options:

- A synthetic corpus you construct (safer — easy ethics).
- A small Wikipedia-biography slice (public; subject attributes verifiable).
- A domain fine-tune of your own data (production value; requires legal/ethics clearance).

Build the measurement:

1. Fine-tune the LM on the corpus.
2. Construct probing prompts for each subject that ask for a specific attribute (e.g. "Alice Smith's email address is").
3. Measure: fraction of prompts where the completion matches the ground-truth attribute (exact and approximate match).
4. Baseline: ask the *un-fine-tuned* model the same prompt; measure the fraction of coincidental matches.
5. Report lift (fine-tuned − baseline) per attribute.

Defensible threshold: "lift > 0.1 on any PII-shaped attribute is a release-blocker".

Ethical note: the small-LM probing variant must be run *only* on corpora you have legal and ethical clearance to use. Document the clearance. Do not probe production third-party LLMs with real subject names and real attributes without an incident-response plan for handling what you find.

## Starter guidance

- **Keep the three targets at a size your laptop can handle.** This is not about setting a benchmark — it is about building the muscle. 20-identity face classifier > full VGGFace. GPT-2-small > a 7B fine-tune.
- **Pick metrics that are defensible, not maximally impressive.** SSIM alone is weak against a subject-level claim; cross-verification against an independent recognition model is the number that holds up.
- **Document the attacker's baseline for every lift.** Attribute inference without a baseline is not privacy-attributable leakage; it is just inference.
- **Pin generative-prior training data.** If you use a face GAN pretrained on FFHQ, record that; the reconstruction's fidelity is partly owed to the prior and the DPIA report must reflect that.
- **Run one mitigation per target.** Even a quick mitigation sweep — training at a lower epoch count, adding differential privacy to the classifier, deduplicating the fine-tune — gives the chapter 07 triage table data points.

## Acceptance criteria

- [ ] Three target-specific reports (tabular, vision, small-LM) produced.
- [ ] For each target: attacker model named, chosen metric named, defensible threshold named, measured value reported.
- [ ] Tabular: attacker baseline *and* model-aided accuracy *and* lift — all three reported. Member vs. non-member stratification included.
- [ ] Vision: pixel-space and GAN-prior inversions both produced; SSIM + cross-verify + qualitative grid reported for each. Representativeness-vs-subject-level distinction explicitly addressed.
- [ ] Small-LM: probing prompts versioned; fine-tuned and baseline model completions scored; lift reported per attribute with ethics-clearance note.
- [ ] At least one mitigation evaluated per target with the Δ reported.
- [ ] Each report ends with the chapter 07 triage framing: chosen-or-not, rationale.

## Stretch goals

- **Attribute inference under DP-SGD.** Train the tabular classifier with DP-SGD at ε ∈ {∞, 8, 4, 1}; report the lift reduction curve.
- **Vision inversion with a diffusion prior.** Replace the GAN prior with a diffusion prior; compare fidelity metrics.
- **Jayaraman et al. attribute inference as MIA-proxy.** Implement Jayaraman's framework and show the lift is larger on members than on non-members; report the margin.
- **LLM probing with alignment-breaking prompts.** Run the Nasr-2023-style alignment-breaking prompts on the fine-tuned LM; report the delta vs. benign prompts.
- **Human-rater study for the vision target.** Small (n = 10 raters, 5-point similarity Likert) rating of reconstructions; report mean + CI.

## What this lands

- The three measurement reports are three rows in the module's attack-coverage matrix (exercise 06 + chapter 09).
- The defender-triage framing feeds the mitigation sweep in exercise 06.
- The ethics / clearance documentation for the small-LM probe is a template you can reuse for the extraction eval in exercise 04.
- The modality-specific metric decisions are durable — the next time you touch a vision or LM target, the choice of SSIM + cross-verify / exact-match + lift is pre-decided.

Version the artefacts. Record the defensible threshold and the rationale — the whole exercise is worth little without them.
