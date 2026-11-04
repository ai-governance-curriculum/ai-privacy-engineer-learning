# Attribute Inference and Fredrikson-Style Model Inversion

Membership inference answers "was x in the training set?". Attribute inference and model inversion answer the complementary questions: "given I partially know x, what is the missing part?" and "give me back something that looks like a training-set input". They are the attacks the public most intuitively recognises as a privacy incident — "the model told me my neighbour's medical condition", "the model gave me back a face that looks like my ex". This chapter is the engineer's reference to building and measuring the two.

Primary references:

- Fredrikson, Jha, Ristenpart — *Model Inversion Attacks that Exploit Confidence Information and Basic Countermeasures*. CCS 2015. [dl.acm.org/doi/10.1145/2810103.2813677](https://dl.acm.org/doi/10.1145/2810103.2813677) — the original Fredrikson paper (also Fredrikson et al. USENIX 2014 on pharmacogenetics).
- Jayaraman, Wang, Gong, Evans — *Revisiting Membership Inference Under Realistic Assumptions*. PETS 2021. [arxiv.org/abs/2005.10881](https://arxiv.org/abs/2005.10881) — attribute inference as a membership-proxy and its limits.
- Zhang, Jia, Chen, Fan, Gong — *The Secret Revealer: Generative Model-Inversion Attacks against Deep Neural Networks*. CVPR 2020. [arxiv.org/abs/1911.07135](https://arxiv.org/abs/1911.07135) — GAN-prior inversion against face classifiers.
- Yeom, Giacomelli, Fredrikson, Jha — *Privacy Risk in Machine Learning*. CSF 2018. [arxiv.org/abs/1709.01604](https://arxiv.org/abs/1709.01604) — the formal relationship between overfitting, membership inference, and attribute inference.

## The two attacks are not the same attack

A common confusion when scoping the work: attribute inference and model inversion share the "reconstruction" flavour but have different attacker models, different evaluation metrics, and different threat scenarios. Keep them separate when you plan the measurement.

- **Attribute inference.** Attacker already has a *partial record* x̃. They want to infer the *specific value* of a missing attribute z. Measurement is per-attribute accuracy / balanced accuracy / lift over the marginal baseline. Low information leak; high policy consequence when the missing attribute is a sensitive category (disease, race, income band).
- **Model inversion.** Attacker has *no specific record* in hand; they have a target *class* or *label*. They want to synthesise a *representative input* that the model assigns to that class — a face, a column vector, a short text. Measurement is fidelity of the reconstruction against ground-truth samples from the class. Headline-generating when the reconstructed face looks like a specific training-set individual.

The two attacks share tooling (model-confidence as an optimisation signal) but are reported separately and routed differently in the DPIA.

## Attribute inference — the attacker model

The clean formulation is the Yeom et al. framework. Fix:

- A set of known attributes K the attacker has for a target record (name, age, zip, medication list).
- A sensitive attribute Z the attacker does not have (e.g. HIV status, income band, political affiliation).
- A model M trained on records (K, Z) — Z is one of the model's features (or correlated with one) but the attacker does not have Z for the specific target.

The attack: query M on all candidate completions (K, z') for z' ∈ Z and use M's confidence / loss to identify the most likely z'. Equivalent to asking the model to serve as a maximum-likelihood classifier for the sensitive attribute given the known ones — which it will, to the extent that the training data taught it that joint.

The catch that Jayaraman et al. formalise: a non-overfit model mostly learns the *population distribution* of (K, Z). An attacker with access to the same population statistics (Census marginals, published epidemiology, a public dataset) can often match the model's prediction without the model. The *privacy-specific* attribute-inference leak is the *lift over the attacker's baseline*. Report both.

### What you actually measure

For a chosen sensitive attribute Z and a chosen eval set of target records (known K, known Z for ground-truth):

- **Attacker-baseline accuracy** — predict Z from K using only the public / marginal statistics you assume the attacker has. This is the "no-model" baseline.
- **Model-aided accuracy** — predict Z from K by querying M, as above.
- **Lift = model-aided − baseline.** The privacy-attributable lift. This is the number the DPIA quotes.
- **Lift conditioned on membership.** The attack typically has *more* lift for records that were in training. Jayaraman et al.'s central result. Report the lift conditioned on membership-label (which you have — you trained M) as a diagnostic.

A threshold for "defensible" lift is domain-specific and is a conversation with the DPO / legal. Zero lift is unrealistic; a 1 % lift on a demographic census attribute is probably ignorable; a 15 % lift on HIV status is a release-blocker. Document the threshold your organisation picks and the rationale.

### Attribute inference against LLMs — the "probing" variant

The extension to language models: construct a prompt that elicits the sensitive attribute given the known ones. "John Smith is a 58-year-old dentist from Columbus, Ohio. John Smith's email is" — ask the model to complete. Measure: fraction of completions that match the ground-truth email on the subject. This is the "extraction-shaped" cousin to tabular attribute inference and overlaps chapter 05's methodology. Keep the two scoped separately: the LLM extraction eval uses *canaries* (chapter 06) you planted; the LLM attribute eval uses *real subjects* in your training set whose attributes you can look up.

Red-team ethics: do not actually run this against third-party production LLMs with real subjects' data as the prompts without legal and IRB review. For your own model, run in a sealed eval environment.

## Model inversion — the attacker model

Fredrikson et al.'s original attack: for a target class y (e.g. a specific person's identity in a face-recognition model), find an input x̂ that maximises the model's confidence for class y, subject to regularisation. Treat x̂ as an estimate of what training inputs for class y looked like.

In the original 2015 paper, the targets were:

- A lifestyle-survey classifier: invert class = sensitive lifestyle attribute → recover the input features correlated with it.
- A face-recognition classifier: invert class = specific individual → recover a face.

The 2015 pixel-space optimisation produces visibly noisy reconstructions. Zhang et al. (CVPR 2020) and subsequent work introduced *generative priors* (StyleGAN, etc.) that produce dramatically higher-fidelity reconstructions. The 2024–2025 literature has further refinements using diffusion priors. The bar for "what a modern attacker can do" is high; report against a modern baseline, not Fredrikson-2015.

### What you actually measure

The question "how faithful is the reconstruction" is domain-specific. Choose a metric appropriate to the modality:

- **Vision (face, X-ray, satellite image, etc.).**
  - SSIM / PSNR vs. a ground-truth training sample of the target class.
  - LPIPS / DreamSim for perceptual similarity.
  - Face-recognition cross-verification: run the reconstructed face through an *independent* face-recognition model; measure the probability it matches the actual training subject.
  - Human rater study for a defensible headline number (when the target is faces or similarly recognisable artefacts).
- **Tabular.**
  - L1 / L2 distance on continuous columns.
  - Exact-match accuracy on categorical columns.
  - Record-linkage score against the target's actual record.
- **Small LMs / short text.**
  - BLEU / ROUGE against ground-truth training strings.
  - Exact-match prefix length — longest prefix of ground-truth the reconstruction recovers verbatim.

For any metric, define the *defensible threshold* up front. The report format: "Model-inversion attack reconstructed subject-level faces at LPIPS = 0.31 (lower is more faithful); our chosen threshold of 0.25 was not breached; we include human-rater similarity scores on a 5-point scale (median 2.1 / 5) as a cross-check."

### Known-limits caveats to call out

- **Representativeness vs. subject-level recovery.** The paper-level distinction most reviewers miss. A model-inversion that produces an "average face" for the class is a representativeness recovery, not a subject-level leak. A model-inversion that produces a face recognisable as a *specific* training subject is a subject-level leak. The two have different policy consequences. The face-recognition cross-verification above is the standard way to distinguish them.
- **Generative-prior contamination.** If the GAN / diffusion prior was trained on data that overlaps the target model's training set (common when using an off-the-shelf face GAN), the reconstruction is partly coming from the prior and not from the target. Report the prior's training data, and ideally cross-reference priors.
- **Fidelity vs. identifiability.** A reconstruction can be low-fidelity (noisy, blurry) and still uniquely identifiable to a human subject (Mona-Lisa-level recognition). Use multiple metrics.

## Composing the two into a module artifact

Exercise 03 builds a three-target suite:

- **Tabular.** A model trained on a tabular dataset (adult-income, Texas hospital-stay, or a representative internal). Attribute inference with lift-over-baseline reporting.
- **Vision.** A small face-classification model (e.g. a reduced FaceScrub / LFW-identity classifier). Fredrikson-style inversion with pixel-space + GAN-prior variants, SSIM + cross-verify metrics.
- **Small LM.** A GPT-2-small or a 125M-param open model fine-tuned on a small corpus. "Probing" attribute inference on named subjects in the fine-tune set. Bridge to chapter 05.

The deliverable is a report that, for each target:

- Names the attacker model (what they know, what they get).
- Names the chosen metric and the defensible threshold.
- Reports the measured value with a confidence interval.
- Documents one mitigation evaluated (reduced capacity, DP-SGD, training-set deduplication, output-side generalisation) and the attack delta after mitigation.

## A pseudocode outline — model inversion with a GAN prior

Not production code — reading guide. The canonical open implementation is the paired-code releases of Zhang et al. (2020) and subsequent work.

```python
# Given:
#   target_model     — trained classifier, predicts logits for num_classes classes
#   generator G      — pretrained GAN that maps z ∈ R^d to an image G(z)
#   target_class y   — the class to invert
#
# Optimise z to maximise target_model's confidence for class y under G.

z = torch.randn(1, D, device=device, requires_grad=True)
opt = torch.optim.Adam([z], lr=1e-2)
for step in range(STEPS):
    x = G(z)                                 # synthesise an image
    logits = target_model(x)
    ce = F.cross_entropy(logits, torch.tensor([y], device=device))
    prior = alpha * (z ** 2).sum()           # keep z plausible under the GAN prior
    loss = ce + prior
    opt.zero_grad(); loss.backward(); opt.step()

x_hat = G(z).detach()
# Evaluate: SSIM(x_hat, ground_truth_training_sample_for_y),
#           cross-verification score from an independent face-rec model,
#           human rating sample.
```

Several variants worth knowing:

- **Average over multiple GAN seeds** to reduce reconstruction noise.
- **Add diversity regularisation** so repeated inversions do not collapse to the same mode.
- **Stop criterion:** target-class confidence above a chosen threshold, or fixed step count.

The exercise 03 build ships this with a proper eval harness, metrics, and mitigation-sweep.

## Where this routes to defences

The mitigations that measurably move attribute-inference and model-inversion numbers:

- **DP-SGD.** Caps per-example influence; reduces the sharpness of the per-class confidence surface the inversion exploits. See chapter 07 for the triage framing and mod-106 for the engineering of DP-SGD itself.
- **Confidence masking / output generalisation.** Return only the top-1 class, or only coarse probability bins (Fredrikson's own "basic countermeasure"). Cheap, limited; a sophisticated attacker runs black-box attacks that tolerate quantised outputs.
- **Reduced capacity / regularisation.** Less-over-fit models have weaker inversion. But privacy-risk-engineered regularisation is not a privacy guarantee — mod-106's DP is the mathematical lever; capacity reduction is the operational one.
- **Deduplication.** For attribute inference against LLMs, duplicate presence in training is the dominant predictor of extractability (Kandpal et al. 2022 — see chapter 06). Dedup the training corpus; the memorised subset shrinks.

Each of these appears in the chapter 07 triage table against the measured attack metrics.

## Summary — what you can do after this chapter

- Distinguish attribute inference from model inversion by attacker model, metric, and threat scenario.
- Build the attribute-inference measurement for a tabular target: baseline-aware lift, conditioning on membership.
- Build the attribute-inference measurement for an LLM target via subject-probing; scope ethics carefully.
- Build a Fredrikson-style model inversion with a GAN / diffusion prior; measure with an appropriate fidelity metric and a defensible threshold.
- Report the "representativeness vs. subject-level recovery" distinction and the identifying cross-verification score.
- Route mitigations (DP-SGD, confidence masking, capacity, dedup) to the measured deltas — chapter 07 is the triage view.

Exercise 03 is the three-target build. Chapter 05 shifts to the LLM-specific memorisation / extraction measurement, which has its own methodology for the same underlying question.
