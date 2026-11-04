# Why Memorisation, Extraction, and Membership Inference Is the Residual-Risk Measurement

mod-104 taught the *preventive* control: carve PII out of the training corpus, the retrieval chunk, the prompt, the log, the egress. mod-106 will teach the *mathematical* control: DP-SGD bounds how much a single training example moves the weights. This module is the *measurement* control. Given a target model you have already trained, what fraction of the training set is a determined attacker going to be able to recover — as membership, as attributes, as verbatim strings — and what is the confidence interval on that number?

That is the question a regulator, a procurement reviewer, a security engineer with an open bug, or an acquirer's due-diligence lead will put to you by name. "You redacted?" is a predicate they do not trust by itself. "You trained with DP-SGD at ε=4?" is a predicate they ask you to prove. The proof is a *measured* attack: you ran the attacker's playbook against your own model, recorded the AUC and the TPR at low FPR, documented the methodology, and bound the residual risk against a defensible threshold. This module is where you build that muscle to production quality.

## Why this matters

Three concrete things stop happening if you do not own the measurement side.

- **DP-SGD becomes a spec you can quote but not defend.** Training with (ε, δ)-DP at ε=4 gives a worst-case upper bound on membership-inference advantage. Your attacker lives in the *empirical* regime, not the worst case. The empirical attack has an AUC, a TPR@0.01 FPR, a per-example score distribution. If you can quote the ε but you cannot produce the attack curve, the DPO / regulator / review board hears "they bought a privacy guarantee they cannot operate".
- **PII redaction's residual is invisible.** mod-104's labelled eval measures per-entity recall against your labelled set — that is the FN budget you accepted at ingest. But the *long tail* — PII the recogniser missed because the entity type is novel, the format is unusual, the spelling is wrong — does not show up in a recall number. It shows up as *memorised text* the model will emit under extraction prompts. Canary + extraction evals are the way you learn what the labelled eval set did not know to look for.
- **The extraction-shaped incident report writes itself.** The archetype — "a language model emitted an email + phone + address that match a specific customer in the training set" — is the kind of incident that is read in a Senate hearing. "We ran the Nasr / Carlini extraction playbook on each release candidate, measured extraction rate against our canary set at (confidence interval), and gated on it" is the kind of sentence that keeps that incident from being a surprise.

In each case, the measurement is the thing the other controls hang their credibility off. The preventive control is a claim; the measurement is the evidence for the claim.

## Where measurement sits in the stack

Place the module in the stack you already have:

- **Downstream of mod-104's detector.** The detector decides which PII enters the corpus. This module measures the memorisation residue of whatever the detector let through. If this module shipped a high extraction rate, the first fix is the detector (mod-104 recogniser or threshold change), not the model.
- **Alongside mod-106's DP engineering.** mod-106 ships the privacy guarantee; this module measures the empirical attack success against it. The two are the two halves of a defensible DP claim — the mathematical bound + the empirical corroboration. A DP claim without a measured attack is a claim a reviewer cannot cross-check.
- **Feeding mod-110's unlearning.** When machine unlearning is invoked against a specific subject, this module is the acceptance test: has the unlearning *actually* removed the subject's memorised presence? Pre-unlearning extraction rate on the subject's canary vs. post-unlearning extraction rate is the regression test.
- **Feeding mod-111's DPIA.** The DPIA quotes this module's artifacts by name: the attack harness configuration, the ML Privacy Meter version, the AUC + TPR@0.01, the extraction-rate confidence interval, the canary set version, the defender-triage decision log. "We bounded re-identification risk" is not a DPIA answer; "ML Privacy Meter v2.x, LiRA + Shokri + attribute inference + Carlini extraction, AUC 0.52 ± 0.01 at 95 % CI on the labelled holdout, canary extraction rate 2/128 at 10k generations, DP-SGD at ε=4, δ=1e-6 — residual-risk section 4.3" is.

The boundary to the peer `ai-risk-engineer` (chapter 10) is about *depth*. That role's mod-107 covers Shokri MIA, LiRA, and memorisation at working depth — enough to run a textbook attack on a toy model and report the AUC. This module deepens into production methodology, per-attack configuration, defender triage against (ε, δ), ML Privacy Meter as the standing harness, and the regulator-defensible coverage story.

## The attack surface this module covers

The AI model is the asset; the attacker has four classes of goal. The module builds a measurement discipline for each.

### Membership inference — "was this record in the training set?"

- **Shokri shadow-model MIA.** The 2017 reference attack: train shadow models on known IN / OUT splits, learn an attack classifier that maps (model output on x) → IN/OUT, evaluate on the target model. Chapters 02 and exercise 01.
- **Carlini-style likelihood-ratio attack (LiRA).** The state-of-the-art recalibration — fit per-example IN and OUT score distributions across many shadow models, do a likelihood-ratio test per target example. Reports TPR at low FPR, not just AUC, because the thing the attacker cares about is the *worst-case per-example leak*, not the average. Chapters 03 and exercise 02.
- **Why both.** Shokri is pedagogical scaffolding and still the baseline a reviewer may recognise. LiRA is the production standard for a defensible residual-risk number because the AUC-averaged number hides the per-example tail that the privacy risk actually lives in.

### Attribute and property inference — "what *values* of what I already know about this record are in the training set?"

- **Attribute inference.** Given partial record x̃ (the attacker knows some columns), predict the missing column from the model's output confidence. A membership proxy when the attacker already has partial PII and wants to learn the sensitive attribute.
- **Fredrikson-style model inversion.** The 2015 attack — Reconstruct a plausible training-set input (a face, a medical-record column, a short text) that the model assigns high confidence for a target class or label. Works at different fidelity on tabular / vision / small-LM targets. Chapter 04 and exercise 03.

### Memorisation and verbatim extraction — "give me, verbatim, a string from the training set"

- **Carlini extraction methodology.** The 2021 attack on GPT-2: prompt the model with random prefixes, score completions by (model perplexity / baseline perplexity), manually verify. Measures the *extractable* fraction of training data, not just membership. Chapter 05 and exercise 04.
- **Canary + Quantifying Memorization.** Plant known-random strings ("canaries") at known counts in the training set; after training, measure how many canaries the model will emit under a targeted prompt. This yields a *quantitative* memorisation rate as a function of canary insertion count — the defender's dose-response curve. Chapter 06 and exercises 04 + 05.
- **Nasr et al. divergence / black-box extraction on production models.** The 2023 attack against ChatGPT-class aligned models: an alignment-breaking prompt induces the model to diverge into verbatim training data. The playbook the module adopts for the *API-setting* extraction eval. Chapter 05.

### Property and distribution inference — "what fraction of the training set has property P?"

- Not the module's main surface — mentioned in chapter 09 for completeness, because the NIST AI 100-2 and MITRE ATLAS mappings include it and a reviewer will want to see it acknowledged.

The four classes sum to the *privacy* attack surface on model weights. The module builds the measurement discipline for the first three (the production surface) and names the fourth in the coverage story.

## Invariants the module enforces

Four invariants that should be true of any residual-risk measurement you ship. Treat them as the acceptance criteria of mod-105, not aspirations.

- **Every reported MIA number is TPR@fixed-FPR, not just AUC.** The per-example tail is where the privacy incident actually lives. AUC alone is Carlini et al. 2022's named foot-gun. Chapter 03 lands the mechanics.
- **Every canary design is defended against cross-corpus false positives.** A canary that happens to look like a real string in Common Crawl or in your own corpus — or in the model's base-pretraining data — inflates your extraction rate without a real memorisation leak. The defence is a documented canary-generation procedure with collision checks. Chapter 06 is where the discipline lands.
- **Every attack is bound to a confidence interval.** Point estimates on 50 target examples are not a measurement; they are a vibe. CI methodology for AUC and for extraction rate is a peer handoff — `model-evaluation-engineer` (peer, L30) owns the bootstrap and permutation-test discipline; you consume it. Chapter 08 names the handoff.
- **Every mitigation decision is a documented triage.** DP-SGD at ε=4 is one defender's choice; deduplication is another; a smaller model is a third. The "we did all three" answer is usually wrong — you did the one that moved your measured AUC below the gate, and you documented why the others were not enabled. Chapter 07 is the triage table.

Call these out in the DPIA. If any is missing, the DPIA has a known-weak section.

## What this module is *not*

- **Not a DP engineering module.** mod-106 owns DP-SGD, privacy accountants, (ε, δ) accounting, moment-accountant implementations. This module *measures* the empirical residual that DP's upper bound dominates.
- **Not a PII detection module.** mod-104 owns ingest-side detection. This module measures what the detector missed by watching for its memorised residue.
- **Not an unlearning module.** mod-110 owns deletion mechanisms. This module is the acceptance test on unlearning — pre vs. post attack success on the deleted subject.
- **Not a confidential-computing module.** mod-108 owns attested inference. This module's attack catalogue is about the model's *outputs* under legitimate use, not the model's *weights* being stolen at rest.
- **Not an adversarial-robustness module.** This is a privacy-attack catalogue; model-poisoning, jailbreak, prompt-injection, model-stealing (as in weight recovery) all live in `ai-security-engineer` scope. Chapter 09 draws the boundary explicitly.

## The artifacts this module hands downstream

By the end of the module, for any model release you touch, you can produce:

- A **measured MIA report** — Shokri AUC, LiRA AUC + TPR@fixed-FPR, 95 % CI from the peer-handed bootstrap methodology, methodology appendix (shadow count, split, attack-model family, hyper-parameters).
- A **measured attribute-inference / model-inversion report** — attribute-inference accuracy relative to an attacker-informed baseline, model-inversion fidelity (e.g. SSIM, MSE, or a domain-appropriate reconstruction metric) against a defined threshold.
- A **measured extraction report** — canary-recovery rate at (insertion count, model, prompt budget), plus a sample of real-corpus verbatim completions found (if any) and their provenance.
- A **canary set** — versioned, with the collision-check procedure and the expected FP rate against your base corpus documented.
- A **defender-triage table** — the levers (DP-SGD, deduplication, per-example clipping, epoch cap, reduced-capacity, output filter) with the measured delta each one produced on the attack metrics.
- A **ML Privacy Meter harness** — repeatable, CI-integrated, pinned-version, with a run manifest so the regulator can replay the measurement on a sealed model binary.
- An **attack-to-standard mapping** — NIST AI 100-2 attack category, MITRE ATLAS tactic + technique, OWASP LLM Top 10 entry (LLM02 Sensitive Information Disclosure), one row per attack in your suite.

These are the artifacts a DPIA (mod-111) will quote, an audit (`ai-audit-engineer`) will replay, a release-assurance review (`ai-evaluation-engineer`, L35) will bundle, and a regulator will ask for by name.

## Summary — what you can do after this chapter

- Place the module in the stack: downstream of mod-104's detector, alongside mod-106's DP, feeding mod-110's unlearning and mod-111's DPIA.
- Name the four attack classes and know which chapter owns each: membership (02 + 03), attribute + inversion (04), memorisation + extraction (05), canary design (06).
- State the four invariants: TPR@FPR not just AUC, cross-corpus-false-positive defence for canaries, confidence intervals on every number, documented mitigation triage.
- Say what this module is *not* — DP, PII detection, unlearning, confidential computing, adversarial-robustness.
- List the seven durable artifacts the module hands downstream.

The next chapter is the first concrete attack: Shokri et al.'s shadow-model membership-inference, built end-to-end at production quality.
