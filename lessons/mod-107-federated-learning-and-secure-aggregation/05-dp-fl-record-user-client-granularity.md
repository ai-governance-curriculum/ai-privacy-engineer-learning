# DP-FL — Record, User, and Client Granularity and the Composition Cost Across Rounds

Secure aggregation (chapter 04) hides individual client updates from the server. DP-FL — differentially private federated learning — noises the updates so the eventually-trained model does not leak training data through its weights. The two are complementary defences, and a production FL system wants both. This chapter is the module's binding to mod-106's DP engineering: you already know what (ε, δ) is, which accountant to pick, and how to defend an ε under audit. Here you decide at **what granularity** the DP mechanism treats a "record" in the federated setting, and how the per-round composition interacts with the federated round cadence.

The granularity choice is the single biggest DP-FL decision. The same mechanism — Gaussian noise on aggregated updates — yields very different ε under different definitions of "neighbouring datasets". Pick the one that matches your threat model and your business problem; defend the choice in the release package.

Primary references:

- McMahan, Ramage, Talwar, Zhang — *Learning Differentially Private Recurrent Language Models*. ICLR 2018. [arxiv.org/abs/1710.06963](https://arxiv.org/abs/1710.06963) — the user-level DP-FL paper; the first scaled demonstration.
- Kairouz et al. 2021 §4.3 — DP-FL granularity and composition survey.
- Geyer, Klein, Nabi — *Differentially Private Federated Learning: A Client-Level Perspective*. NeurIPS 2017 PPML workshop. [arxiv.org/abs/1712.07557](https://arxiv.org/abs/1712.07557) — client-level DP in FL, early formalisation.
- Kairouz, McMahan, Song, Thakurta, Thakkar, Wang — *The Distributed Discrete Gaussian Mechanism for Federated Learning with Secure Aggregation*. ICML 2021. [arxiv.org/abs/2102.06387](https://arxiv.org/abs/2102.06387) — how to add DP noise inside SA cleanly.
- Ramaswamy, Thakkar, Mathews, et al. — *Training Production Language Models Without Memorizing User Data*. 2021. [arxiv.org/abs/2009.10031](https://arxiv.org/abs/2009.10031) — Google's user-level DP training on Gboard in production.
- TensorFlow Federated DP documentation — [tensorflow.org/federated/tutorials/building_your_own_federated_learning_algorithm](https://www.tensorflow.org/federated/tutorials/building_your_own_federated_learning_algorithm) and `tff.aggregators.DifferentiallyPrivateFactory`.
- NIST SP 800-226 — the standards reference from mod-106; its ε-reporting requirements also apply in FL.

## The three granularities, defined by the neighbouring relation

Recall from mod-106 chapter 01 that (ε, δ)-DP is *relative to a neighbouring relation on datasets*. Two datasets are neighbours if they differ in… the unit of protection. The three standard choices in FL:

- **Record-level DP.** Two datasets are neighbours if they differ in a single training *example* (a single sentence, a single image, a single sample). The unit is a record.
- **User-level DP.** Two datasets are neighbours if they differ in *all records belonging to a single user*. The unit is a user. A user who contributed 500 examples has *all 500 examples* protected jointly.
- **Client-level DP.** Two datasets are neighbours if they differ in *all data from a single client (silo)*. The unit is a client / silo.

Same mechanism, three different guarantees. The ε under the same Gaussian noise is roughly:

```
ε_record  ≤  ε_user  ≤  ε_client            (holding δ, mechanism fixed)
```

A record-level guarantee is the weakest of the three; a client-level guarantee the strongest. The strongest-sounding is not necessarily the right one — see below.

## Which granularity fits which threat model

Pick granularity by answering: *what, exactly, does the attacker want to learn, and from what observation?*

### Cross-device: user-level DP is usually correct

In cross-device FL, the user is the privacy unit. A user's keyboard history, their location trace, their health-sensor stream — each is a bundle of correlated records belonging to one person. The reconstruction attack against a user does not stop at one record; it chains the inferences. The attacker's win condition is *learning something about the user*, not *learning a specific tweet*.

Record-level DP in cross-device is almost always wrong. It protects each keystroke separately but leaves a user's aggregate pattern vulnerable to the composition of hundreds of per-record leaks.

User-level DP is the Google-Keyboard / Apple-Intelligence published posture. The McMahan 2018 paper and the Ramaswamy 2021 Gboard report quote user-level ε.

Client-level DP in cross-device would collapse the whole device's contribution under one noise budget — technically a tighter guarantee, but at scale impractical because the noise overwhelms the model.

### Cross-silo medical: client-level DP vs. record-level DP depends on the collaboration

Cross-silo is the harder decision. Two orthogonal concerns:

- **Does hospital A want hospital B (or the aggregator) to learn whether hospital A's dataset contains any specific patient?** That is a *record-level* question under hospital A's dataset-as-neighbouring-relation. Pick record-level DP, where each patient-record is the unit of protection, and the per-hospital training uses something close to mod-106 DP-SGD.
- **Does hospital A want the broader world to not learn hospital-level properties — for instance, which exact hospitals are in the consortium contributing which types of patients?** That is a *client-level* question. Pick client-level DP, where the whole hospital's update is the unit of protection.

In practice, most medical cross-silo consortia pick **record-level** or **user-level** (patient-level) DP, because the business value is per-patient confidentiality and the client-level claim is at best awkward to interpret (hospitals are not trying to hide their participation). The MELLODDY consortium published its DP posture at patient-level.

Client-level DP is the right pick for *small* client counts (3–5 banks, 2 insurers) where any single client's participation is itself sensitive and the joint-controllership agreement requires it. The Geyer paper describes exactly this setting.

### The "mixed" granularity trap

A common design-doc mistake: proposing that *the server* uses user-level DP but *each client* uses record-level DP-SGD during local training. The composition is not additive in the way the author usually intends; the end-to-end guarantee is usually the weaker of the two, not the stronger.

If the goal is user-level, use user-level consistently: clip and noise per-user contributions at the aggregation step (user-level DP aggregator). If the goal is record-level, use DP-SGD on the client. Mixing the two levels is a compounded-complexity trap; the accountant you need to defend both is esoteric.

## Where the noise goes — three mechanical shapes

The DP noise can be injected at three places in the FedAvg round. Each has different trust requirements and different ε implications.

### Shape 1 — Client-side DP-SGD (record-level only)

Each client trains with DP-SGD locally, producing a private update `Δ_i`. The server aggregates as normal. The privacy guarantee is *per client's local training* — that is, each client's own training process is DP with respect to its own records. The aggregated global model inherits a DP guarantee *per client*, but the end-to-end guarantee over the population is not automatically a global DP statement; it depends on how you formalise the overall mechanism.

- **Trust assumption:** each client honestly runs DP-SGD. No server-side guarantees.
- **When to pick:** cross-silo, where each silo wants to prove its own local training is private.
- **Composition cost:** each client's DP-SGD budget composes across local epochs per round and across rounds. A hospital training for 100 rounds burns through its local budget 100 times.

### Shape 2 — Central / server-side DP (clip + noise at aggregation)

The server clips each incoming update to an L2 norm `C`, sums the clipped updates (optionally inside SA), and adds Gaussian noise with scale `σC`. This is the *canonical DP-FedAvg* mechanism, directly analogous to DP-SGD at the server. The granularity is determined by what each `Δ_i` represents:

- If `Δ_i` is one client's update (the FedAvg default), clipping + noising at the server yields **client-level DP**.
- If each `Δ_i` is explicitly aggregated per-user (user-aggregates-then-server-aggregates), you get **user-level DP**.

```python
# Pseudocode for the server-side DP step.
clipped = [clip_to_norm(delta, C) for delta in client_updates]
summed = sum(clipped)
noise = gaussian(scale=sigma * C, shape=summed.shape)
dp_aggregate = (summed + noise) / N
```

- **Trust assumption:** the server honestly clips and noises. This is the honest-but-curious model that chapter 04's SA already assumes; composes cleanly.
- **When to pick:** default for cross-device user-level DP and for cross-silo client-level DP.
- **Composition cost:** the server's privacy accountant accumulates across rounds. A 10 000-round training run with per-round `(ε_round, δ_round)` composes via RDP / PRV accountant (mod-106 chapter 02) to a (potentially much tighter than linear) final (ε, δ).

### Shape 3 — Distributed DP (noise inside SA)

The ideal composition: each client contributes its share of the Gaussian noise, SA sums them, and the aggregate ends up noised *without* either the server or any individual client holding the noise in the clear. Kairouz et al. 2021 (Distributed Discrete Gaussian) is the reference implementation.

- **Trust assumption:** honest clients; the noise is correct if a sufficient fraction of clients contribute their shares.
- **When to pick:** when you want DP-FL without having to trust the server to add the noise — the canonical pattern for the strongest end-to-end story.
- **Composition cost:** same as server-side DP after round-level accounting, but you lean on the discrete-Gaussian machinery because the DH / SA arithmetic is modular and continuous Gaussian noise breaks the modular cancellation.

## The composition cost across rounds — the mechanic that burns budget

DP-FL composes across rounds the same way DP-SGD composes across steps. Each round adds its noise; the per-round privacy loss accumulates. The question is *how fast* the (ε, δ) grows as rounds accumulate.

### Rate of growth under naïve accounting

Basic composition: `T` rounds at per-round `(ε_round, δ_round)` compose to `(T · ε_round, T · δ_round)`. Linear in `T`. This is the accountant-of-last-resort. For `T = 1000` rounds at ε_round = 0.1, you end at ε = 100 — a nonsense number.

### Rate of growth under RDP / PRV accounting

RDP composition is additive in the (α, γ) curve, not in ε. The Sampled Gaussian Mechanism's RDP curve at per-round noise `σ` and sampling rate `q` composes `T` times by adding `T · γ_round(α)` for each α. Converting back at the end gives an ε proportional to `√T` under advanced composition, and tighter still under PRV.

Concrete rule of thumb for DP-FL with the Sampled Gaussian Mechanism:

```
# of rounds | at σ = 1.0, q = 0.01 | at σ = 1.0, q = 0.001
       100 | ε ≈ 0.6              | ε ≈ 0.07
     1,000 | ε ≈ 1.9              | ε ≈ 0.2
    10,000 | ε ≈ 6.1              | ε ≈ 0.7
   100,000 | ε ≈ 19               | ε ≈ 2.2
```

<!-- needs-research: cross-verify these rule-of-thumb ε values against Opacus PRVAccountant and dp_accounting PRVAccountant at δ = 1e-6. The order-of-magnitude pattern (ε grows roughly √T in the Sampled Gaussian regime, with sub-linear amplification by small q) is standard but the exact table should be reproduced from a run before it is cited in a design review. -->

The take-away: **the sampling rate `q` is the dominant lever**. In cross-device FL, picking only 100 of 10⁷ eligible clients per round gives `q = 10⁻⁵`, which lets you run thousands of rounds at modest ε. In cross-silo FL with all 10 silos participating every round, `q = 1`, and *there is no privacy amplification by subsampling at all* — every round spends its full un-amplified ε. The round-count budget is strictly tighter for cross-silo, and this is the central reason client-level DP in cross-silo is operationally hard.

### The round-cadence interaction

The accountant does not care about wall-clock time; it cares about round count. But the **round cadence** (how fast you run rounds) determines how many rounds you spend within any realistic wall-clock budget and how many of those rounds affect the same user.

- **Cross-device, user-level DP.** A user who has their Keyboard-FL client participating once a day for a year is in roughly 365 rounds. If your per-round noise is calibrated against the empirical mean of participation 1-2 rounds/user/week, the long-tail user burns more budget than you planned. The fix is a *participation cap* — each user may contribute to at most `M` rounds over the lifetime of the deployment, enforced on the client. Chapter 02's client-selection section flagged this; here is where it binds the DP accounting.
- **Cross-silo, round cadence.** A silo participating in every one of 100 rounds concentrates its exposure across the full training run. A cadence change (run twice as many rounds in half the compute per round) does not help; the composition cost is round-count-based.

### Where the accountant lives

Each of the main FL libraries provides an accountant hook:

- **TFF:** `tff.aggregators.DifferentiallyPrivateFactory(...)` wraps `tff.aggregators.SumFactory` with clip + noise; composition is tracked via `dp_accounting` / `dp-accounting-RDP` libraries. The `dp_accounting.rdp.RdpAccountant` or `dp_accounting.pld.PLDAccountant` is what you cite in the release package.
- **NVIDIA FLARE:** DP-FedAvg aggregator with a built-in accountant driven by the `dp_accounting` library. The admin console logs per-round ε.
- **Flower:** community-contributed DP strategies (`DPFedAvgFixed`, `DPFedAvgAdaptive`) with Opacus-style accounting; you typically wire an Opacus `RDPAccountant` or `PRVAccountant` externally and log per-round ε in your training script.
- **PySyft:** DP integration via external DP libraries; expect to wire the accountant yourself.

In every case, **persist the accountant state** on every server-side checkpoint. If the server restarts mid-training and resumes without the accountant state, you either over-spend budget or your reported ε is wrong. The persist-and-restore attestation from mod-106 chapter 03 applies here identically.

## Adaptive clipping — the one tuning trick worth knowing

The L2 clipping norm `C` is a critical hyper-parameter: too small and the model learns nothing; too large and each noised coordinate carries less signal relative to the noise. McMahan-Andrew 2019 ("A general approach to adding differential privacy to iterative training procedures", [arxiv.org/abs/1812.06210](https://arxiv.org/abs/1812.06210)) describes *adaptive clipping* — track an estimate of the quantile of client-update norms across rounds, and set `C` to that quantile, adjusted via a tiny-DP side channel so the adjustment itself is private. TFF and FLARE both ship adaptive clipping as a first-class feature.

Pick adaptive clipping as the default in cross-device; the empirical distribution of client-update norms drifts over the training run and a static `C` is almost always sub-optimal.

## The DP-FL stanza in the release package

The mod-107 release package extends the mod-106 DP stanza with FL-specific fields. Minimum content:

- **Granularity.** "Record-level", "user-level", or "client-level". With a one-sentence justification tied to the business threat model.
- **Mechanism.** "Server-side DP (clip + Gaussian noise)" / "distributed DP (discrete Gaussian inside SA)" / "client-side DP-SGD, then plain aggregation". Named.
- **Parameters.** Clip norm `C`, noise multiplier `σ`, sampling rate `q = k / N`, round count `T`, participation cap `M`.
- **Accountant.** Which accountant (RDP / PRV / PLD) implemented by which library at which version produces the quoted ε.
- **Round-level and lifetime ε.** `ε_round` at `δ_round`, lifetime `ε` at `δ`.
- **Composition note.** "The per-round sampling rate is maintained uniform across rounds and the accountant assumes uniform Poisson-with-replacement sampling; non-uniform selection would invalidate the quoted ε."
- **Attestations.** "Accountant state persisted on every server checkpoint." "Participation cap enforced on client side with server-side verification log." "DP noise addition is server-side; no debug path exists that logs pre-noise aggregates."

Mirror the structure of the mod-106 release package. This stanza *supersedes* the single-training-run DP stanza when the model is trained federatedly.

## Where DP-FL does not help

The same list of things DP does not protect from mod-106 chapter 01 still applies, plus two FL-specific additions:

- **DP-FL does not protect the model from gradient-reconstruction attacks that run on an *un-aggregated* update.** If an attacker can observe one client's `Δ_i` in the clear (because SA is not in place), DP noise added to the aggregate does not help — the noise is not applied to the individual update. SA is the complementary defence.
- **DP-FL does not protect against malicious clients poisoning the model.** Byzantine-robust aggregation is the orthogonal control. A DP mechanism that clips updates reduces the impact of outlier updates somewhat, but it is not a poisoning defence by design.

## The decision tree — picking the DP-FL design in one page

1. **Is this cross-device or cross-silo?** (Chapter 01's framework.)
2. **If cross-device:** user-level DP, central or distributed mechanism. Enforce per-user participation cap. Burn budget by round count × per-round-ε.
3. **If cross-silo with record-level threat (per-patient at each hospital):** record-level DP via client-side DP-SGD. Central aggregation is plain FedAvg.
4. **If cross-silo with client-level threat (per-institution):** client-level DP via server-side clip + noise. Only viable with small round counts or very loose ε.
5. **If cross-silo with both record-level and client-level requirements:** stack both. Client-side DP-SGD for record-level (local guarantee), server-side clip-and-noise for client-level (global guarantee). Log two ε values. The release package has both stanzas.
6. **Everywhere:** persist the accountant state on checkpoint; no debug path through the pre-noise aggregate; participation cap enforced in code.

## Summary — what you can do after this chapter

- State the three DP granularities (record, user, client) by their neighbouring relations.
- Pick the right granularity for a cross-device (user-level default) and cross-silo (record-level or client-level by threat) design.
- Articulate why "mixed" granularity is usually a design-review red flag.
- Explain the three places DP noise can be injected (client-side DP-SGD, server-side clip+noise, distributed via SA) and the trust assumption of each.
- Reason about the composition cost across rounds as a function of round count and sampling rate, and name the sampling rate as the dominant lever.
- Enforce a per-user participation cap for cross-device user-level DP.
- Persist and restore the accountant state on checkpoint.
- Populate the DP-FL stanza in the release package.
- Explain to a stakeholder when DP-FL does and does not help, and what complementary defences (SA, Byzantine-robust aggregation, mod-108 TEEs) do.

Chapter 06 is where the attacks that defeat the naive FL stack show up — and what mitigations this module ships.
