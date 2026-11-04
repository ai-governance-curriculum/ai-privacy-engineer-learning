# Defending the (ε, δ) Choice Under Audit

Chapters 03 and 04 produced the number. This chapter is about the *defence* of the number. Picking ε is not a tuning decision; it is a policy decision with legal, contractual, and reputational consequences. A regulator, a procurement reviewer, a privacy counsel, or an acquirer's due-diligence lead will ask "why ε = 4 and not ε = 1?" and "how does this compare to Apple's deployment?" A DP engineer who cannot answer in one sentence is a liability. This chapter builds the muscle.

Primary references:

- Dwork, Kohli, Mulligan — *Differential Privacy in Practice: Expose Your Epsilons!* Journal of Privacy and Confidentiality, 2019. [journalprivacyconfidentiality.org/index.php/jpc/article/view/689](https://journalprivacyconfidentiality.org/index.php/jpc/article/view/689) — the canonical argument that ε values must be reported, not hidden.
- Apple Differential Privacy Team — *Learning with Privacy at Scale*. Apple ML Journal, Dec 2017. [machinelearning.apple.com/research/learning-with-privacy-at-scale](https://machinelearning.apple.com/research/learning-with-privacy-at-scale) — Apple's published ε for keyboard / emoji telemetry.
- Erlingsson, Pihur, Korolova — *RAPPOR: Randomized Aggregatable Privacy-Preserving Ordinal Response*. CCS 2014. [arxiv.org/abs/1407.6981](https://arxiv.org/abs/1407.6981) — Google Chrome's local-DP deployment.
- US Census Bureau — *Disclosure Avoidance for the 2020 Census: An Introduction*. [www.census.gov/library/publications/2021/decennial/2020-census-disclosure-avoidance-handbook.html](https://www.census.gov/library/publications/2021/decennial/2020-census-disclosure-avoidance-handbook.html) — the 2020 Decennial DP deployment, the largest public DP release by far.
- NIST SP 800-226 — *Guidelines for Evaluating Differential Privacy Guarantees*. Dec 2023. [nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-226.pdf](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-226.pdf) — the standards reference your release package cites.
- Garfinkel, Abowd, Powazek — *Issues Encountered Deploying Differential Privacy*. WPES 2018. [arxiv.org/abs/1809.02201](https://arxiv.org/abs/1809.02201) — the engineering retrospective on Census deployment.
- Cummings, Kaptchuk, Redmiles — *"I need a better description": An Investigation Into User Expectations For Differential Privacy*. CCS 2021. [arxiv.org/abs/2110.06452](https://arxiv.org/abs/2110.06452) — what non-experts actually hear when you say "ε".

## The regulator's four questions

If you can answer these four questions in a page, you pass the audit. If you cannot, the DP claim is a liability.

1. **What is ε, in English, for this release?** Not the formal definition. The operational one, in terms a non-mathematician's privacy counsel can quote.
2. **Why is this ε, and not a smaller one?** The utility-privacy trade-off, honestly stated.
3. **How does it compare to published deployments?** The Apple / Google / Census anchor.
4. **What is the threat model — what does the ε protect against, and what does it not?** The NIST SP 800-226 evaluation section.

The rest of this chapter is the four answers, filled in.

## Question 1 — the English meaning of ε

The formal meaning: `Pr[M(D) ∈ S] ≤ exp(ε) · Pr[M(D') ∈ S] + δ`. Nobody in the audit room except you reads that and feels anything.

The operational meaning is a statement about *what an adversary can learn about a single record*. Three framings that regulators actually understand:

- **The Bayesian framing.** An adversary with a prior of belief `p₀` that a specific record was in the training set can, after seeing the model's output, update to a posterior at most `e^ε · p₀ / (e^ε · p₀ + (1 - p₀))` in the worst case. At ε = 1, a 50% prior becomes at most ~73%. At ε = 4, at most ~98%. At ε = 8, at most ~99.97%. The posterior bound decays fast as ε grows; the "meaningful privacy" region is sometimes articulated as ε ≤ 1 for that reason.
- **The hypothesis-test framing.** The best adversary trying to decide IN vs. OUT has (TPR + FNR) + e^ε(FNR + TNR) worst-case trade-offs bounded by the ε. For ε = 1, TPR at 1% FPR is bounded above by roughly 3% in the worst case (the mathematical bound, not the empirical attack). Empirically (mod-105 LiRA), DP-SGD at ε = 1 drives LiRA TPR@0.001 to roughly chance; ε = 8 is still meaningfully above chance.
- **The counterfactual framing.** If any one person's record is added, removed, or swapped in the training set, the distribution over model outputs changes by a multiplicative factor of at most `e^ε`. "Nothing you see about the model can distinguish *you* from a counterfactual world where your record was out, beyond this factor."

The release package (chapter 09) picks one framing and sticks with it. Mixing framings across sections of the same document is where reviewers lose trust.

## Question 2 — why *this* ε

There is no universally agreed threshold for "acceptable ε". The literature has converged on three rough bands, with a lot of qualification:

- **ε ≤ 1 — "strong privacy".** The regime where the adversary's advantage is small in the Bayesian framing above. The NIST SP 800-226 reporting checklist calls this "a strong guarantee that individuals' information is well-protected". Apple's iOS 10 keyboard telemetry (Learning with Privacy at Scale, 2017) reports a *per-donation* ε in the single digits and a *daily* budget on the order of 4–6 for some datatypes; the published per-user-per-day privacy budget is the composed number. Google's RAPPOR deployments use similar order-of-magnitude budgets.
- **1 < ε ≤ 10 — "moderate privacy, practical trade-off".** The regime where DP meaningfully bounds attacker advantage but does not vanish utility. Much of the production ML literature operates here. Note the posterior-bound numbers above: at ε = 8 the adversary posterior can rise to ~99.97%; the mathematical worst-case bound is weak at this ε. The empirical attack bounds (from mod-105) are often much better, which is what chapter 09's threat-model note explicitly says.
- **ε > 10 — "mostly symbolic; utility-dominated".** The DP claim is still mathematically meaningful (it rules out the pathological adversary with high prior who would win near-deterministically), but the practical privacy gain over no DP is small. The US Census 2020 DAS person-level ε (about 19.6 for the overall product, broken down into much smaller per-query ε values via the hierarchical mechanism) sits near the top of this range; the Census argued for it as a trade-off against statistical utility for small geographies. (See the next section.)

Note two things:

- **The band is a vocabulary, not a verdict.** Picking ε = 2 is not "right" because 2 < 10; picking ε = 12 is not "wrong" because 12 > 10. The trade-off is contextual: dataset sensitivity, population size, release frequency, utility requirement, and what the empirical attack numbers say.
- **The empirical attack is the corroborator, not the replacement.** DP's ε is a worst-case bound. Your mod-105 LiRA TPR@0.001 is the empirical attack success. Both go in the release package: ε = 4 is the mathematical claim; LiRA TPR@0.001 = 0.02 ± 0.01 (95% CI) is the empirical claim. A release that quotes only one of the two is incomplete.

The engineering answer to question 2, in practice, is a short story: "We piloted ε = 1, ε = 4, and ε = 8. At ε = 1 utility on {downstream task} dropped from {baseline}% to {ε=1 value}%. At ε = 4 utility was {ε=4 value}%, within {gap}% of baseline. At ε = 8 utility was indistinguishable from baseline. We chose ε = 4 because it is the smallest ε in our utility-acceptable range and matches published deployments in {band}. The LiRA empirical attack at ε = 4 is {TPR@0.001}, which is below our acceptance gate of {threshold}."

That is a defensible answer. It names the alternatives, the utility consequence of each, the published anchor, and the empirical corroborator. The DP sweep that produces those three utility numbers is exercise 04.

## Question 3 — the anchor table

Anchoring the organisation's ε against public deployments is the single most effective move in a review. A reviewer who has never seen DP before will be unsure what "ε = 4" means in a vacuum; they are not unsure what "the same order of magnitude as Apple's keyboard telemetry per-donation" means.

The anchor table, with the primary citation for each row. Keep this table in the model card; a reviewer will read it, nod, and move on.

| Deployment | Per-release / per-donation ε | Composition / budget notes | Source |
|---|---|---|---|
| Apple iOS keyboard telemetry (2017+) | Published per-event ε in single digits; daily budget on the order of 4–8 for some datatypes | Local DP; per-event budget composed daily | [Apple ML Journal, 2017](https://machinelearning.apple.com/research/learning-with-privacy-at-scale) |
| Google RAPPOR (Chrome, 2014+) | Per-report ε values published in the paper | Local DP; longitudinal composition bounded by permanent + instantaneous randomised response | Erlingsson et al., CCS 2014 |
| Google Community Mobility Reports (COVID-19) | ε = 0.44 to 2.19 depending on region / query | Central DP; Laplace mechanism | Google Blog & [2020 report](https://blog.google/technology/health/covid-19-community-mobility-reports/) |
| US Census 2020 Disclosure Avoidance System | Person-level global ε ≈ 19.6; per-query ε values much smaller | zCDP accounting; hierarchical mechanism; the ε is broken into geographic-level allocations | US Census PL 94-171 Redistricting Data documentation; Abowd et al., 2022 |
| LinkedIn audience engagement reports | Per-report ε small; Laplace mechanism | Central DP; monthly release | LinkedIn Engineering Blog |
| Meta Opportunities for Prosperity (COVID Symptom Survey w/ CMU) | Published Laplace ε | Central DP | Meta Research posts |
| NIH ALL of Us Research Program | DP used for public-facing aggregate counts | Central DP; conservative ε | NIH documentation |

Several things to notice about this table:

- **Deployments vary ε by *orders of magnitude* even within the "same band".** Census's ε = 19.6 (total) is three orders of magnitude different from Apple's per-event ε. The deployments are *not* directly comparable because the mechanisms, the composition, and the granularity differ.
- **Local DP (Apple keyboard, RAPPOR) uses ε per contribution per user; central DP (Census, Google Community Mobility) uses ε per release.** You cannot compare an Apple-per-event number to a Census-per-release number without translating them to a common granularity.
- **Composition matters.** The number a regulator will care about is the *total* ε spent on the release they are reviewing, under the neighbouring relation that matters to the person they represent. "User-level ε over the lifetime of the deployment" is often that number. If your release is ε = 4 per training run and you train once per month, the annual user-level ε is ε = 48 — which is a very different story.

The engineering discipline: in your anchor table, state each row's *granularity, mechanism, and composition unit* explicitly. A reviewer who cannot read those columns will not trust the comparison.

## Question 4 — the threat model (what DP does and does not protect)

This is where NIST SP 800-226 earns its keep. The document has a threat-model section that enumerates what a DP claim does and does not protect against; the release package (chapter 09) is structured to answer each.

What DP (at a meaningful ε) *does* protect against:

- **Membership inference about any single record.** The ε bound is the mathematical guarantee. Empirical attacks from mod-105 corroborate it.
- **Attribute inference *of information specific to one record*.** The adversary's advantage in predicting a specific person's attribute from the model, beyond what population statistics would give them, is bounded by the ε.
- **Reconstruction of a single record from model outputs** — Fredrikson-style model inversion. Same bound.
- **Verbatim extraction of a single record from an LLM** — at a meaningful ε, extraction should empirically drop to near-zero on canaries inserted once. (mod-105 chapter 06 is the measurement discipline.)

What DP does *not* protect against:

- **Aggregate / population statistics.** DP does not hide that your training set contained many examples of category X. If "many of our users are in region R" is itself sensitive, DP is not the mechanism.
- **Group-level privacy at a granularity different from your mechanism.** Record-level DP does little for a user with many records. User-level DP does little for a group that spans users (a company, a household, a neighborhood).
- **Attacks on the training infrastructure.** DP protects the model's *output*; it does not protect model weights at rest (mod-108), training logs that recorded non-DP information (mod-103), or RAG retrieval traffic (mod-109).
- **Attacks after multiple releases if you did not compose them.** Each release's DP claim is independent; the user's cumulative exposure is the composition. If you train and release many models from the same data, the user-level ε is the composition of all of them, not the maximum.
- **Prompt-injection / jailbreak / adversarial perturbation.** Different threat classes entirely; `ai-security-engineer` territory.
- **DP's failure modes from engineering bugs.** BN not removed, shuffled sampling under Poisson accountant, logs leaking per-example gradients. The ε you quote is only as true as the engineering attestations in the release package.

The release package's threat-model note is structured as two columns: "DP protects" | "DP does not protect; the following controls do", with each "DP does not protect" row linked to the module that owns the control (mod-108 for confidential inference, mod-103 for ingest-side ingest, mod-110 for unlearning after a deletion request, mod-109 for retrieval, etc.).

## The "per-release" vs "lifetime" trap

The single most common place DP claims collapse under review is the distinction between per-release and lifetime ε.

A production ML service retrains its model every N weeks. Each training run spends ε_per_run from the privacy budget. If the training set is the same records each time (the user does not opt out), the *lifetime* user-level ε is the *composition* of all the per-run ε values. For a service that trains weekly for a year, that is 52 compositions.

Three mitigations:

- **Fresh randomness per training run.** Advanced composition over k independent (ε, δ)-DP mechanisms gives roughly `(ε√(2k log(1/δ')), kδ + δ')` with probability `1 - δ'`. Better than linear in k, but still grows with k. The "composed" claim is what the model card quotes.
- **User-level DP from the start.** Train with user-level DP (user-grouped minibatches) so that per-run user-level ε is the quantity that composes. mod-107's federated learning chapter is where this pattern lives.
- **Subsample users per training run.** If only a random 10% of users contribute to each training run, their expected per-year ε scales down by the sampling rate.

The engineering answer in the release package: "Lifetime user-level ε, assuming M retraining cycles per year under our current training schedule, is composed to roughly {value} under the PRV accountant. Mitigations in place: {list}."

## The ε elicitation anti-patterns

A short list of moves that will fail a review. Do not do these.

- **"We used DP"** with no ε, no δ, no mechanism, no accountant, no unit. Every variable is omitted. Reviewers assume you do not have the numbers.
- **ε = 0.001.** If you are reporting a sub-one hundredth-scale ε, either you are at a tiny training-set size or your accountant is wrong. In practice, sub-0.1 ε values are almost always a math or integration bug in the accountant; verify with a cross-check accountant before quoting.
- **ε = 10,000.** An ε this large does not bound the adversary's advantage over the prior in any meaningful way. If your accountant reports ε ~ 10^4, either you have a bug or you have no privacy; do not quote it as "DP" to a regulator.
- **Quoting Apple's ε and not your own.** Reviewers recognise the move. The anchor table is a reference point; your ε is your number.
- **Over-claim on δ.** δ = 1e-5 is usually chosen to be "small enough"; it should be ≪ 1/N. If N = 1B, δ = 1e-5 is 10,000 times larger than the usual threshold. Your δ matters.
- **Ignoring user-level vs record-level.** Reviewers increasingly ask. Whichever you use, name it.

## The model-card entry — the one-page template

The DP section of a model card, in the format reviewers find skimmable:

```
## Differential Privacy

Claim: (ε, δ) = (4.0, 1e-6) under user-level unbounded DP.
Mechanism: DP-SGD (subsampled Gaussian).
Accountant: Opacus PRVAccountant v1.5.2 (primary); cross-checked
            against Microsoft prv_accountant v0.2.0 (ε = 4.05 at same δ).
Mechanism parameters:
  - sampling rate q = 1.25e-3
  - noise multiplier σ = 1.1
  - clipping norm C = 1.0
  - steps T = 40,000 (50 epochs × 800 steps)
Composition over lifetime: anticipated 12 retrainings per year;
                           composed user-level ε per year ≈ 14.2 under PRV.
Engineering attestations:
  - BN removed via ModuleValidator.fix (commit abc)
  - Poisson sampling enabled; UniformWithReplacementSampler
  - Virtual batching via BatchMemoryManager; per-sample clipping
  - Accountant state checkpointed on resume; verified in test X
Empirical corroboration:
  - LiRA TPR @ FPR=1e-3: 0.019 ± 0.008 (95% CI, bootstrap n=10k)
  - LiRA AUC: 0.512 ± 0.004
  - Carlini canary extraction rate at 128 canaries: 0/128
Threat model (NIST SP 800-226 Section 5): see Appendix A.
Lifetime audit log: see appendix B (per-epoch ε at δ = 1e-6).
```

The release package (chapter 09) packages this block plus the attestations and audit log.

## A worked example — defending ε = 4 in a review

Setting: an internal review board for a healthcare NLP model trained on de-identified clinical notes. Reviewers include a privacy counsel, a product lead, and a clinical informaticist.

Prep:
- ε = 4, δ = 1e-7 (dataset N ≈ 10M).
- Opacus PRVAccountant; audit log attached.
- Sweep: ε = 1 (utility: 62% F1), ε = 4 (utility: 78% F1), ε = 8 (utility: 81% F1), no DP (utility: 82% F1).
- Empirical: LiRA TPR@0.001 = 0.03 (CI ±0.01), canary extraction 0/256.

Review exchange (compressed):

- *Counsel:* "Why ε = 4, not ε = 1?"
- *You:* "ε = 1 drops F1 to 62%, below the clinical-utility gate of 70% set in the DPIA. ε = 4 gives 78%, above gate, and matches the published band of Apple's per-donation telemetry."
- *Counsel:* "At ε = 4 the posterior bound goes to 98%. A regulator will note that."
- *You:* "Agreed. The mathematical worst-case posterior is 98%. The empirical attack at this ε shows LiRA TPR at 0.1% FPR is 3%, essentially chance, and canary extraction is zero at 256 insertions. We report both numbers."
- *Clinical:* "What happens when we retrain?"
- *You:* "We retrain quarterly. Composed user-level ε over a year is ~14.2 under PRV. We refresh the training-set opt-out list before each retrain, which offers an opt-out but not a formal privacy amplification. We will revisit this after the first retraining cycle."
- *Counsel:* "What does DP not protect against?"
- *You:* "Aggregate dataset properties (e.g., 'this corpus contains notes from diabetic patients'). Training-infrastructure attacks. Lifetime exposure beyond the composed ε. Those are covered by mod-103 (ingest-side control), mod-108 (confidential inference), and the retraining policy. See Section 5 of the DPIA."

The review ends with "approved, revisit after first retrain for the lifetime-ε number." That is a passing audit.

## Summary — what you can do after this chapter

- Translate (ε, δ) into the Bayesian, hypothesis-test, or counterfactual framing for a non-technical reviewer.
- Answer the four regulator questions in a page: operational meaning of ε, why this ε, how it compares, what it does not protect.
- Produce the anchor table with Apple / Google / Census as calibration points and the correct citations.
- Write the threat-model note aligned to NIST SP 800-226 Section 5's structure.
- Distinguish per-release ε from lifetime user-level ε, and name the three mitigations (fresh randomness composition, user-level DP, user subsampling) for the lifetime number.
- Flag and avoid the six ε elicitation anti-patterns.
- Fill in the model-card DP section from a real DP-SGD training run.

Chapter 06 shifts from DP-SGD to PATE — the teacher-ensemble alternative for cases where DP-SGD does not fit.
