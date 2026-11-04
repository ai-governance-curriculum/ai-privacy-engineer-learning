# Bonawitz Secure Aggregation — Pairwise Masks, Secret Sharing, Dropout Tolerance

FedAvg's aggregation step is the privacy vulnerability of the base protocol: the server sees every individual client update `Δ_i` before computing the average. Any attacker who sees `Δ_i` — including an honest-but-curious server — can mount the gradient-reconstruction attacks that chapter 06 catalogues. Secure Aggregation (SA) is the cryptographic wrapper that lets the server compute `Σ Δ_i` *without* learning any individual `Δ_i`. The canonical protocol is Bonawitz et al. 2017, "Practical Secure Aggregation for Privacy-Preserving Machine Learning" ([acm.org/doi/10.1145/3133956.3133982](https://dl.acm.org/doi/10.1145/3133956.3133982), [eprint.iacr.org/2017/281](https://eprint.iacr.org/2017/281.pdf)). It is the protocol under Google's Gboard, under TFF's `secure_aggregator`, under NVIDIA FLARE's SA workflow, and conceptually under Flower's SA strategy.

Think of this chapter as teaching you *enough to explain the protocol to a reviewer, pick its parameters, and defend the threat model it defends against*. You will not implement production SA from scratch — the exercise walks you through a didactic implementation — but you need the mechanics to debate design choices and to know when SA is overkill or under-kill.

Primary references:

- Bonawitz, Ivanov, Kreuter, Marcedone, McMahan, Patel, Ramage, Segal, Seth — *Practical Secure Aggregation for Privacy-Preserving Machine Learning*. CCS 2017. [eprint.iacr.org/2017/281](https://eprint.iacr.org/2017/281.pdf) — the foundational paper. Read sections 3 (protocol), 6 (security), and 8 (dropout).
- Shamir — *How to Share a Secret*. Communications of the ACM, 1979. The threshold-secret-sharing scheme that lets SA tolerate dropouts.
- Diffie & Hellman — *New Directions in Cryptography*. IEEE TIT, 1976. The Diffie-Hellman key agreement SA uses for pairwise shared keys.
- Bell, Bonawitz, Gascón, Lepoint, Raykova — *Secure Single-Server Aggregation with (Poly)Logarithmic Overhead*. CCS 2020. [eprint.iacr.org/2020/704](https://eprint.iacr.org/2020/704.pdf) — "SA+", the second-generation protocol with better asymptotic cost; Google's production stack uses the ideas from it.
- Kairouz et al. 2021 §4 — the SA survey section.

## What SA is for — and what it is not for

SA has one job and one threat model. It solves exactly this problem:

> *Honest-but-curious server, untrusted clients, high dropout rate, and we want the server to learn only `Σ_i Δ_i`, not any individual `Δ_i`.*

Internalise every word of that sentence. If the threat model does not match, SA is not the control you need.

- **Honest-but-curious server.** The server follows the protocol correctly (does not inject fake clients, does not run a different protocol, does not tamper with cryptographic outputs). It is nevertheless *curious* — it would happily learn an individual client's update if the protocol allowed it. SA defeats this. SA does **not** defend against an actively malicious server that runs a different protocol — that threat requires zero-knowledge proofs, or TEEs (mod-108), or multi-party server architectures.
- **Untrusted clients.** Clients may drop, misbehave, or collude. SA tolerates up to a configurable fraction of dropped or malicious clients.
- **Does not defend against malicious clients poisoning the model.** SA ensures the server sees only the sum. If a client uploads a gradient designed to embed a backdoor, SA's output is the sum of all gradients including the malicious one — Byzantine-robust aggregation is an orthogonal defence class (Krum, trimmed mean, median; Flower's `fl.server.strategy.Krum`, FLARE's Byzantine-robust filters).
- **Does not defend against the model's leaking training data through its weights.** That is a DP-FL concern (chapter 05) and an extraction-attack concern (chapter 06). SA keeps individual *updates* private; it does not noise them. The eventually-trained model is as leaky as FedAvg without SA.

"SA protects the server from being able to see individual client updates" is the sentence. "SA protects the model's privacy" is not; it is the sentence that gets the control framed too generously in a release package and backfires in a review.

## The protocol, rung by rung

The simplest version of the protocol teaches the idea. Then we add dropout tolerance. Then we add server-collusion resilience. Each rung is the additional machinery you need to defeat one failure mode.

### Rung 0 — Why not just add noise locally and sum?

You could tell each client to add random noise `r_i` to its update, upload `Δ_i + r_i`, and then have the clients collectively broadcast `-Σ r_i` so the server can subtract it. This fails three ways:

- The clients need to coordinate the noise — if they generate independently random `r_i`, the sum is not zero.
- If one client drops after noising but before un-noising, the aggregate is corrupt.
- The server can get at `Δ_i` by comparing two rounds' uploads.

SA fixes all three. The trick is *pairwise cancelling masks*.

### Rung 1 — Pairwise masks that cancel on summation

Assume we have `N` clients labelled `1 … N`. Every pair of clients `(i, j)` agrees (via Diffie-Hellman) on a shared secret `s_{i,j}`. Each client expands `s_{i,j}` into a mask `PRG(s_{i,j})` the same size as `Δ_i`. Client `i` then computes:

```
y_i = Δ_i  +  Σ_{j > i}  PRG(s_{i,j})  -  Σ_{j < i}  PRG(s_{i,j})         (mod M)
```

The server receives `y_i` from every client and computes:

```
Σ_i y_i  =  Σ_i Δ_i  +  Σ_i Σ_{j > i} PRG(s_{i,j})  -  Σ_i Σ_{j < i} PRG(s_{i,j})
         =  Σ_i Δ_i                   (the double-sums cancel pairwise)
```

Each client's `y_i` individually is indistinguishable from random noise (because the mask it adds is PRG output of a key the server does not know). Only when all `y_i` are summed do the masks annihilate. The server learns `Σ Δ_i` and nothing else.

One elegance point: the arithmetic is modular — a large prime or `2^k` modulus — because PRG output has to live in the same field as the gradient coordinates for the cancellation to work exactly. In practice, gradient vectors are quantised to fixed-point before masking.

### Rung 2 — Dropout tolerance via Shamir secret sharing

The pairwise-mask scheme has a fatal weakness: if client `i` drops after everyone has included its mask, the masks `PRG(s_{i,j})` for `j ≠ i` no longer cancel, and the aggregate is garbage. In cross-device FL, dropout is the default — 30–70% per round. The scheme as described cannot survive.

Fix: Shamir-share each client's `s_{i,j}` keys among the other clients, with a threshold `t`. If client `i` drops, as long as at least `t` other clients survive, they can reconstruct the keys `i` would have used, so the server can reconstruct — and subtract — the masks that are now "stuck" on the aggregate.

Shamir (`t, N`)-secret-sharing: a secret `s` is encoded as the constant term of a random degree-`t−1` polynomial over a finite field. Each of `N` participants receives one point `(x_k, p(x_k))`. Any `t` points determine `p`, hence `s`; any fewer than `t` reveal nothing about `s`.

In SA, each client `i` generates two secrets: its own private PRG seed `b_i` (used to mask itself against the server reading a lone surviving client's update) and the collection of pairwise keys `s_{i,j}`. Client `i` Shamir-shares both `b_i` and (the seed to regenerate) `s_{i,j}` among all `N` clients with threshold `t`.

At aggregation time, the server knows which clients dropped (call the set `D`) and which survived (`S`). For each surviving client `i ∈ S`, the server does one of two things, never both:

- If `i ∈ S`: the server asks the other surviving clients to reveal their shares of `i`'s **mask seed** `b_i`. With `t` shares, the server reconstructs `b_i`, computes the client's self-mask `PRG(b_i)`, and knows that `y_i − PRG(b_i)` is `Δ_i + (pairwise masks only)`.
- If `i ∈ D` (dropped): the server asks the surviving clients to reveal their shares of `i`'s **pairwise-mask keys** `s_{i,j}` for `j ∈ S`. With `t` shares, the server reconstructs the keys, regenerates the pairwise masks client `i` would have contributed, and subtracts them.

Why the "one or the other, never both" rule? Because if the server reconstructed *both* `b_i` and the pairwise keys of a surviving client, the server could compute that client's update `Δ_i` in the clear. The protocol forces a disjoint choice per client, and the Shamir threshold `t` is set high enough that the server cannot adaptively flip its choice mid-protocol.

### Rung 3 — Server-collusion resilience

If the server colludes with some clients, those colluding clients can reveal their shares to the server directly, effectively reducing the threshold. The protocol's security assumes:

- **Honest clients ≥ threshold `t`.** If fewer than `t` honest clients remain, security is lost.
- **`t` is set to defeat a specified collusion fraction.** The paper's analysis: `t` should be ⌈(2N+1)/3⌉ to tolerate up to `(N-t)/N ≈ 1/3` adversarial clients while surviving up to `N - t ≈ 1/3` dropouts. These one-thirds are not simultaneous — the analysis handles joint-worst-case.

Pick `t` around `N/2` for research / low-stakes settings; around `2N/3` for production cross-silo; closer to `N` when adversarial-client risk dominates (and dropouts are low).

### Rung 4 — The four-round protocol

In practice, the end-to-end SA protocol takes four communication rounds per training round.

1. **Advertise keys.** Each client generates two DH key-pairs (one for key agreement, one for signatures) and publishes the public halves to the server, which gossips them to the others.
2. **Share keys.** Each client computes its `s_{i,j}` with every other client via DH, generates its mask seed `b_i`, Shamir-shares `b_i` and `s_{i,j}` seeds among the others (encrypted under per-pair keys), and uploads the shares to the server to route.
3. **Masked input collection.** Each client computes `y_i` and uploads it to the server.
4. **Consistency and unmasking.** The server tells each surviving client which other clients survived / dropped. Clients sign the surviving-set, re-upload the relevant shares, server reconstructs the stuck masks, and computes the final aggregate.

Each round has its own failure mode. A client that is alive in round 1 but fails before round 4 is treated as a drop. The total number of DH keys, Shamir shares, and verified signatures per client is `O(N)` — which is the cross-device scaling problem that motivated the Bell-Bonawitz 2020 "SA+" follow-up with `O(log N)` cost.

## Dropout-tolerance tuning

The practical tune on `t`:

- Cross-device (per-round dropout 30-70%): `t ≈ 0.5N` to `0.6N`. Too high and you cannot recover from the realistic dropout; too low and server collusion becomes dangerous.
- Cross-silo (per-round dropout 0-10%): `t ≈ 0.6N` to `0.8N`. The dropout constraint is slack; raise `t` to tighten adversary resilience.
- Enterprise-fleet or sensor-network: tune to your observed dropout histogram. Log the empirical dropout distribution for ten rounds and set `t` at or above the 95th percentile of survivors.

**The failure mode you most want to avoid:** setting `t` so tight that the realistic dropout rate occasionally leaves you with fewer than `t` surviving clients. When that happens, the round *fails entirely* — no aggregate, no update — and from the operational side it looks like an SA bug. Over-provision `t` to tolerate the realistic dropout tail.

## Communication cost — and why it drove the second generation

Per client per round, the base protocol has cost roughly:

- `O(N)` DH public keys exchanged (one with each other client).
- `O(N)` Shamir shares uploaded.
- `O(d)` coordinates of masked update (same as gradient size).
- `O(N)` reconstruction messages at the end.

At `N = 1000` clients per round and `d = 10^6` parameters, that is manageable. At `N = 50 000` per round (a target for Google's internal deployments), the `O(N)` terms dominate the upload budget on a phone. SA+ (Bell et al. 2020) uses secret-shared randomness committees to drop the `O(N)` factor to `O(log N)` and is what Google's current Gboard stack runs. Flower's experimental `SecAgg+` strategy is an implementation of this.

For cross-silo with `N = 10–30`, the base Bonawitz protocol is fine. For cross-device at scale, consider SA+ or an equivalent.

## When *not* to ship SA

Three concrete anti-patterns:

- **Cross-silo with a trusted neutral aggregator in a TEE.** If the aggregator runs in a hardware-attested enclave (mod-108), the "honest-but-curious server" threat is already mitigated by the enclave's isolation guarantee. Stacking SA on top of a TEE buys little and adds operational complexity. Stick to one defence, and document which.
- **When the number of clients per round is tiny (≤ 2).** SA requires `N ≥ t + 1 + {expected dropouts}`. With two clients, SA degenerates — the sum of two values with their difference reveals both. SA assumes a crowd. If you cannot muster a crowd, DP + secure channels is the right stack.
- **When the aggregation is non-linear.** SA sums gradients; it does not natively sum medians, trimmed-means, top-k operations. Byzantine-robust aggregators (Krum, trimmed mean) require per-client visibility that SA by design denies. Composing them requires either weakening SA (so the server can rank clients) or weakening Byzantine-robustness (so the server operates on the summed output). The published compositions (`Flame`, `Siren`) are research-grade.

## SA and DP — do they stack?

Yes, and the composition matters. SA keeps individual updates confidential from the server; DP noises the aggregate so the eventually-trained model is private. In a production design:

- **Server-side DP (DP added post-aggregation).** The server adds Gaussian noise to `Σ Δ_i` before updating the global model. Trust assumption: the server is honest about adding the noise (no cryptographic enforcement). Works well with SA because the DP calculation happens on the aggregate the server has anyway.
- **Local DP (each client noises its own update before SA).** Each client adds its share of the Gaussian noise to `Δ_i` before masking. Trust assumption: each client is honest about noising. Composes cleanly with SA — the sum of per-client Gaussian noise is also Gaussian; the scale adjusts analytically — and does not require trusting the server to add noise at all.

Chapter 05 goes deeper into the composition and which DP granularity (record / user / client) you pair with SA.

## The release-package SA stanza

When the mod-107 training run uses SA, the release package (chapter 09 of mod-106's release-package pattern, chapter 08's boundary note, and the FL-specific adaptations) carries an SA stanza. Minimum fields:

- **Protocol name and version.** "Bonawitz et al. 2017 pairwise-masked SA, as implemented by TFF `secure_aggregator` v{X}" or "SA+ (Bell et al. 2020), as implemented by Flower `SecAgg+` strategy v{Y}".
- **Threat model.** Honest-but-curious server, named collusion fraction, named dropout tolerance.
- **Parameters.** `N` per round, Shamir threshold `t`, modulus `M`, PRG family, DH group.
- **Attestations.** "No client-side logging of raw `Δ_i`. Server-side logging limited to aggregate." "Shamir shares are destroyed after each round." "Mask seeds are regenerated per round; no cross-round key reuse."
- **Verification artefacts.** The per-round server log excerpt, the Flower / TFF / FLARE configuration file, the SA primitive's version hash.

The stanza mirrors the mod-106 DP stanza structurally — a mechanism claim, a threat-model note, a parameter block, a set of attestations, and a pointer to the audit log.

## The common footguns

Four things that go wrong and look right in a code review.

- **Reusing DH keys across rounds.** If `s_{i,j}` is not refreshed each round, a server that compromises the keys once can read updates from past *and future* rounds. Refresh per round. Flower's Secure Aggregation+ strategy does this; TFF's `secure_aggregator` does this; home-rolled implementations frequently forget.
- **Insufficient mask resolution.** The mask lives in `Z_M`. If `M` is smaller than the gradient quantisation range, overflow occurs and the aggregate is wrong. Pick `M` large enough to absorb `N × ||Δ_i||_∞` at the worst-case client.
- **Treating dropouts as "same as failed".** A dropped client is a client whose share can still be reconstructed if enough others cooperate. A client that *exits uncleanly* (crashes, loses network mid-round-4) is counted as a drop, but a client that *quits with intent* (returns a partial, inconsistent view) may be an attacker. The protocol specifies how to handle each; the implementation should not collapse them.
- **Logging individual `Δ_i` server-side for "debugging".** Any logging of individual updates defeats SA's whole point. Remove the debug path before shipping. The server-side audit log should contain the aggregated sum and the per-client participation flag — nothing else.

## Summary — what you can do after this chapter

- State the SA threat model from memory: honest-but-curious server, untrusted clients, high dropout; defends individual-update confidentiality from the server.
- Explain the four-rung construction: pairwise masks → Shamir secret-shared keys → dropout tolerance → four-round protocol.
- Set the Shamir threshold `t` as a function of expected dropouts and adversarial-client tolerance.
- Explain why the server's one-or-the-other rule (mask seed or pairwise keys, never both) is the security invariant.
- Articulate the `O(N)` cost of base Bonawitz SA and the motivation for SA+.
- Identify three settings where SA is overkill or ill-suited (TEE-backed aggregator, tiny `N`, non-linear aggregation).
- Reason about SA + DP composition at both the server-side and local-DP points.
- Populate the release-package SA stanza with protocol, threat model, parameters, attestations, and verification artefacts.
- Spot the four recurring footguns in a code review.

Chapter 05 is the DP-FL layer that composes with this SA foundation.
