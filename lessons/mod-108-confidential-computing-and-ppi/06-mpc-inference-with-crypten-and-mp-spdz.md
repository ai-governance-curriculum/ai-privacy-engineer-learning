# Secure Multi-Party Computation for Inference — CrypTen, MP-SPDZ, Arithmetic vs. Boolean Protocols, Round Complexity

HE lets one client offload computation to one server who never sees the plaintext. Secure Multi-Party Computation (MPC) lets multiple parties *jointly* compute a function over inputs none of them want to reveal to each other. In privacy-preserving inference, the archetypal MPC setup is: the model owner holds the weights, the data owner holds the input, neither wants the other to see their private values, and together they compute the model output such that only the data owner learns it. In the cross-organisation shape — a hospital holding patient features, a pharma company holding a risk model, a cloud party providing compute — MPC is the protocol of choice.

MPC and HE occupy overlapping-but-distinct niches. HE is cryptographically simpler to reason about at the surface level (one key, one ciphertext type) but runs into performance ceilings fast. MPC is more involved (multiple parties with cryptographic interaction, careful round-complexity accounting) but can run practical neural-network inference at latencies HE struggles to approach — at the cost of a two-party-non-collusion assumption. Picking between them is a design conversation this chapter will equip you for.

Primary references:

- Yao — *Protocols for Secure Computations*. FOCS 1982. [dl.acm.org/doi/10.1109/SFCS.1982.38](https://ieeexplore.ieee.org/document/4568207) — the original garbled-circuits paper; foundational.
- Goldreich, Micali, Wigderson — *How to Play ANY Mental Game, or A Completeness Theorem for Protocols with Honest Majority*. STOC 1987. [dl.acm.org/doi/10.1145/28395.28420](https://dl.acm.org/doi/10.1145/28395.28420) — the GMW protocol; multi-party arithmetic / Boolean circuits with honest majority.
- Beaver — *Efficient Multiparty Protocols Using Circuit Randomization*. CRYPTO 1991. [link.springer.com/chapter/10.1007/3-540-46766-1_34](https://link.springer.com/chapter/10.1007/3-540-46766-1_34) — the "Beaver triples" technique that makes arithmetic MPC practical; pre-computed correlated randomness.
- Bogdanov, Laur, Willemson — *Sharemind: A Framework for Fast Privacy-Preserving Computations*. ESORICS 2008. [eprint.iacr.org/2008/289](https://eprint.iacr.org/2008/289) — the 3-party secret-sharing MPC framework; the design pattern the research-grade and production MPC tooling descend from.
- Mohassel, Zhang — *SecureML: A System for Scalable Privacy-Preserving Machine Learning*. S&P 2017. [eprint.iacr.org/2017/396](https://eprint.iacr.org/2017/396) — 2-party ML with mixed arithmetic / boolean protocols; foundational for the current generation.
- Juvekar, Vaikuntanathan, Chandrakasan — *GAZELLE: A Low Latency Framework for Secure Neural Network Inference*. USENIX Security 2018. [eprint.iacr.org/2018/073](https://eprint.iacr.org/2018/073) — HE + garbled-circuits hybrid for 2-party neural-net inference.
- Facebook / Meta — *CrypTen: Secure Multi-Party Computation Meets Machine Learning*. NeurIPS Privacy in ML Workshop 2020 / arXiv 2109.00984. [arxiv.org/abs/2109.00984](https://arxiv.org/abs/2109.00984) and [github.com/facebookresearch/CrypTen](https://github.com/facebookresearch/CrypTen) — PyTorch-style API over MPC; research-grade.
- Keller — *MP-SPDZ: A Versatile Framework for Multi-Party Computation*. CCS 2020. [eprint.iacr.org/2020/521](https://eprint.iacr.org/2020/521) and [github.com/data61/MP-SPDZ](https://github.com/data61/MP-SPDZ) — the production-adjacent framework; supports 30+ protocols.
- CrypTFlow / Porthos — Kumar, Rathee, et al., *CrypTFlow: Secure TensorFlow Inference*. S&P 2020. [eprint.iacr.org/2019/1049](https://eprint.iacr.org/2019/1049) — another production-ish line; useful to know by name.
- Secret Network / TF Encrypted / PySyft — community MPC libraries worth naming for completeness; MP-SPDZ and CrypTen are the two we focus on.

## The MPC trust model, in one paragraph

MPC schemes have a *threshold*: a stated number of parties out of the total that the protocol tolerates as corrupted without losing privacy or correctness. The common categories:

- **Honest-majority** (`t < n/2`): protocols like GMW / Sharemind-style 3-party secret sharing. Simpler, faster. Assumes a majority of participants is honest.
- **Dishonest-majority** (`t < n`): protocols like SPDZ. More expensive, harder to deploy, but tolerate a single honest party only.
- **Semi-honest** (passive): the adversary follows the protocol but tries to extract information from the transcript. Most ML-focused protocols target this.
- **Malicious** (active): the adversary can deviate from the protocol. Much more expensive; required where the parties may actively try to cheat.

The most common production shape for ML inference is **2-party semi-honest** (the data owner and the model owner, both honest-but-curious; a MITM-level attacker is handled by the transport layer) or **3-party semi-honest** with a non-colluding third-party compute node. The non-collusion assumption is load-bearing; chapter 07's memo will have you state it explicitly.

## Secret sharing — the core primitive

Most MPC schemes build on *linear secret sharing*: a value `x` is split among `n` parties as random shares `x_1 + x_2 + ... + x_n = x (mod p)`. Each party holds one share; no proper subset of parties can learn anything about `x`; all `n` shares together reconstruct `x`. Linear operations (addition, constant multiplication) are then "free" — each party just adds its shares. Multiplication is the hard operation, and the protocol differences are mostly in how multiplication is realised.

### Addition

If party `P_i` holds share `x_i` of `x` and share `y_i` of `y`, then `z_i = x_i + y_i (mod p)` is a share of `z = x + y`. Each party does one addition locally; no communication. This is why linear layers (dense matrix-vector products with plaintext-weight, and additions) are the "cheap" part of MPC inference.

### Multiplication

Multiplying two secret-shared values is the expensive operation. One common technique uses *Beaver triples*: pre-shared `([a], [b], [c])` where `c = a * b` and all three are secret-shared. To compute `[x] * [y]`, the parties:

1. Compute `[d] = [x] - [a]` and `[e] = [y] - [b]` (local, no communication).
2. Open `d` and `e` (each party sends its share to the others).
3. Compute `[z] = d * e + d * [b] + e * [a] + [c]` (local).

One multiplication takes one round of communication (opening `d` and `e`). Over a deep network with millions of multiplications, the total round cost is dominated by the circuit depth, not the number of multiplications — because many multiplications at the same depth can happen in parallel in one round.

The Beaver triples themselves must be generated somehow. Two common sources:

- **Offline-phase precomputation.** A protocol (e.g., SPDZ's offline phase) generates and distributes triples before the online phase. The offline phase can be expensive (seconds of wall clock per million triples) but amortises well.
- **Trusted dealer.** A trusted third-party ("dealer") generates and distributes triples. Simpler but requires an extra trust assumption.

## Arithmetic vs. Boolean protocols

The protocol family is a function of the operation set.

### Arithmetic protocols

Secret-share values as integers modulo a prime (or a large power of 2). Addition and multiplication are the native operations. Suitable for linear algebra: matrix multiplications, convolutions, ReLU-via-polynomial, polynomial activations. Fast when the circuit is arithmetic-heavy.

### Boolean protocols

Secret-share values as bits. XOR is addition modulo 2 (free); AND is the expensive multiplication. Suitable for bitwise operations: comparisons, maxpool, exact ReLU (which is a comparison + mask), bit-level arithmetic. Slower than arithmetic for heavy linear algebra, essential for operations that need bit-level logic.

### Why both are needed

Neural-network inference uses both. A dense layer is arithmetic. Exact ReLU — `max(0, x)` — requires a comparison, which is bit-level. A fully-arithmetic protocol runs ReLU via polynomial approximation (as in HE); a fully-boolean protocol runs the matmul bit-serial (slow). The production answer is **mixed protocols**: do the linear parts arithmetically, do the non-linear parts in Boolean, and *convert* between representations at each boundary.

### Conversions are expensive

Arithmetic-to-Boolean conversion (sharing re-encoded from one form to the other) is the dominant cost in some protocols. The protocol choice is a trade-off between (a) fast linear algebra (arithmetic) and (b) exact non-linearities (Boolean), with conversion overhead proportional to the number of non-linearity-boundary transitions.

### Representative choices

- **ABY** (Demmler et al. 2015) — pioneer of mixed arithmetic / Boolean / Yao 2-party protocols; the "A", "B", "Y" are the three representations and ABY handles conversions between them.
- **ABY3** (Mohassel, Rindal 2018) — 3-party extension with honest majority.
- **SecureML** — 2-party arithmetic for linear parts, garbled circuits for non-linear parts.
- **DELPHI** / **MUSE** / **CrypTFlow / Porthos** — the post-2019 hybrids optimised for neural-network inference latency.

## Round complexity — why it is the thing to track

In MPC, *round complexity* (how many sequential communication rounds the protocol requires) is the dominant cost metric. Reasons:

- Each round incurs the full network RTT between the parties. For parties in different datacentres, RTT is milliseconds to tens of milliseconds. A protocol with 10,000 rounds takes 10,000 × RTT regardless of the compute speed.
- Bandwidth is usually not the bottleneck on datacentre-to-datacentre links; latency is.
- Depth of the circuit (number of multiplication levels when grouped greedily) ≈ number of rounds. Reducing circuit depth reduces rounds.

The engineering consequences:

- Deploy parties close in network topology. Same AZ, same datacentre, same rack where possible. Cross-continent MPC is a research demo, not a production deployment.
- Batch everything. Independent multiplications at the same depth are done in one round; sequential multiplications are not. Batch-size matters.
- Prefer protocols with low round complexity for the circuit you have. A "fewer rounds, more bandwidth" protocol wins on a slow network.

## CrypTen — PyTorch-style API over 2-party MPC

CrypTen ([github.com/facebookresearch/CrypTen](https://github.com/facebookresearch/CrypTen), [arxiv.org/abs/2109.00984](https://arxiv.org/abs/2109.00984)) is Meta's research-grade library that wraps secret sharing and MPC in a PyTorch-style tensor API. The design goal: an ML engineer with PyTorch skills can read and write MPC code without a cryptography PhD.

Key properties:

- **Two-party semi-honest.** Not malicious-secure out of the box; not three-party.
- **Arithmetic secret sharing** over `mod 2^64` with a Beaver-triple-based multiplication protocol.
- **Boolean secret sharing** for comparisons and bit operations.
- **Automatic arithmetic↔Boolean conversion.**
- **PyTorch-like API:** `crypten.cryptensor(x)` wraps a tensor as a secret-shared ciphertext; `+`, `*`, `matmul`, `relu`, `conv2d`, `max` etc. are supported; models can be loaded from a `torch.nn.Module` and converted with `crypten.nn.from_pytorch(...)`.

Minimal example:

```python
import crypten
import torch

crypten.init()
# Party 0 provides the input x (the data owner).
# Party 1 provides the model weights (the model owner).

x = crypten.cryptensor([1.0, 2.0, 3.0], src=0)   # encrypted by party 0
w = crypten.cryptensor([0.1, 0.2, 0.3], src=1)   # encrypted by party 1

y = (x * w).sum()   # dot product, result is secret-shared
print(y.get_plain_text())   # both parties cooperate to reveal
```

CrypTen handles the secret-sharing bookkeeping, the Beaver-triple consumption, and the arithmetic↔Boolean conversions. It is suitable for research, for small-scale production demos, and for teaching. It is not the library you deploy to serve a million inferences per day at low latency — the overhead and the semi-honest security model are limits.

### CrypTen gotchas

- **Dummy Beaver triples by default.** CrypTen's Beaver triples are, by default, *not* cryptographically generated — they are generated by a trusted third party for simplicity. Switch to a real triple generator (`crypten.mpc.provider.TFPProvider`, or an external SPDZ offline phase) for a real deployment.
- **Semi-honest only.** No malicious-security guarantee. A misbehaving party can deviate from the protocol undetected. For adversarial cross-organisation settings, this is not enough; you need MP-SPDZ or similar.
- **Non-linearities use polynomial / Yao-style approximations.** ReLU is implemented as a secret-shared comparison + mask; accurate but costs a round of communication per layer.
- **Model-weight privacy is bounded.** CrypTen's default pattern assumes the party providing the model knows the weights in plaintext. That is often correct; if you need both parties to not know the weights (e.g., a model held by a third-party escrow), that is a different protocol shape.

## MP-SPDZ — the production-adjacent framework

MP-SPDZ ([github.com/data61/MP-SPDZ](https://github.com/data61/MP-SPDZ)) is CSIRO Data61's descendant of the original SPDZ protocol (Damgård, Pastro, Smart, Zakarias 2012) and SCALE-MAMBA. It supports ~30 MPC protocols: honest-majority with 3 parties, dishonest-majority with any number, semi-honest, malicious, and combinations. Written in C++ with a Python-based compiler front-end that lets you write the computation as a Python-like program and compile it to a bytecode run by the MP-SPDZ runtime.

Key capabilities:

- **Choose your protocol.** Same computation can compile to many protocols; benchmark-driven protocol selection becomes feasible.
- **Honest and malicious variants.** Both available.
- **Full Beaver-triple offline phase.** No trusted dealer; the offline phase generates cryptographic correlated randomness.
- **Deployment story.** More operationally involved than CrypTen but actually ships: the Bristol MPC group's deployments and several industrial consortiums use it.

A CrypTen-equivalent example in MP-SPDZ is longer — you write the computation in `.mpc` Python-like code, run `./compile.py your-circuit`, and launch N parties with the compiled program. The learning curve is steeper; the production story is stronger.

### When to pick MP-SPDZ over CrypTen

- Cross-organisation deployments where the parties are adversarial, not merely non-colluding. Malicious security is required.
- Multi-party (`n ≥ 3`) protocols. CrypTen is 2-party; MP-SPDZ is `n`-party.
- Strong auditability — MP-SPDZ's cryptographic rigour is better-studied.
- Willingness to pay the engineering tax for operational correctness.

### When to stay with CrypTen

- Prototype, research, or demo stage.
- 2-party semi-honest is the correct threat model.
- The team's ML engineers need a familiar API.

## A typical MPC inference deployment shape

For a 2-party cross-organisation inference (hospital + model vendor):

- **Party A — Data owner (hospital).** Holds the patient feature vector `x`. Wants to learn the prediction without revealing `x` to the vendor. Does not want to see the vendor's model weights `W`.
- **Party B — Model owner (vendor).** Holds `W`. Wants to serve the prediction without revealing `W` to the hospital. Does not want to see `x`.

The protocol:

1. The parties agree on the model architecture (public) and the protocol parameters (ring size, triple generation source, security level). The architecture must be public because both parties execute it.
2. The offline phase runs (once, or periodically). Beaver triples are generated; each party holds its shares of the pre-computed `([a], [b], [c])` tuples.
3. Per inference request: Party A secret-shares `x` with Party B. Party B secret-shares `W` with Party A. The two parties execute the MPC protocol step-by-step. At the end, Party A holds all shares of the output `y` and reconstructs it; Party B is told the protocol finished successfully.
4. Each operation (matmul, ReLU, maxpool) is a composed sub-protocol; the total round count is a function of the model's depth.

Three-party variants add a non-colluding compute party (typically a cloud operator contracted to not collude with either data or model owner). The protocol is faster (honest-majority) at the cost of an additional non-collusion assumption. This is the setup Sharemind pioneered and that GCP's Confidential Space partially emulates (a non-colluding attested TEE as the "third party").

## Round-complexity reality check for LLM inference

A production LLM (billions of parameters, tens of layers, long contexts) is not within the practical MPC regime today. The published results for MPC inference on ResNet-50-scale models are measured in seconds to tens of seconds per image under optimal conditions; LLM-scale extrapolations are minutes to hours per generated token. MPC is practical for **small-to-medium networks** (logistic regression, decision trees, simple MLPs, small CNNs — like HE, roughly the CryptoNets scale, extending somewhat further with the recent protocol advances). For LLM inference at production latencies, a TEE-based deployment (chapters 01–04) is the pragmatic answer; MPC fits when you need two-party-non-collusion rigour on a small-to-medium model.

<!-- needs-research: cite specific up-to-date benchmark numbers from a recent paper (e.g., Cheetah, Delphi, SIRNN). Numbers in this space age quickly; prefer a paper with its own citation to a fixed benchmark. -->

## When MPC is the right tool

Narrower than HE, but important.

### Scenario 1 — Two institutions, one wants to query the other's private model

Hospital wants to score patient features against a pharmaceutical company's proprietary risk model. Both want privacy on their own side. HE can solve this if the linear-only approximation of the model is acceptable; MPC handles the exact non-linearities at the cost of multiple rounds.

### Scenario 2 — Cross-silo joint analytics

Multiple banks want to run joint fraud detection on their aggregated transaction data without pooling the raw data. Horizontal partition of features across parties; MPC computes the joint statistic. This is mod-107's cross-silo FL territory but with *inference*-style one-shot computations rather than training.

### Scenario 3 — Secure inference against a confidential-model-as-a-service

Model vendor sells "inference as a service" and must not learn the client's prompts. HE handles this for small models; MPC handles it for medium models with exact non-linearities. If the service is a frontier LLM, neither does; the TEE is the pragmatic choice.

### Scenario 4 — Regulated analytics under "cryptographically prevent cross-party data flow"

A regulator (e.g., a financial-sector DPA) specifies that certain joint analyses must not expose any party's raw data to any other party, with cryptographic enforcement. MPC is the tool; the exact protocol (semi-honest vs. malicious; 2-party vs. 3-party) is a function of the regulator's acceptance.

### When MPC is explicitly *not* the right tool

- Single-tenant inference where there is only one data owner and one model owner / server — HE or TEE is simpler.
- Any use case where the parties can tolerate a trusted hardware TEE — a TEE is dramatically cheaper operationally.
- LLM-scale inference under production latency SLOs.
- Settings where the parties are in different regions/countries and network RTT is high — round complexity hurts badly.

## Common pitfalls

### Pitfall 1 — Ignoring the non-collusion assumption

MPC with "2-party semi-honest" assumes the two parties do not collude. "Do not collude with themselves" is trivially true; the assumption is that they do not collude with *each other*. If party A is the hospital and party B is the vendor, the assumption is that the hospital does not help the vendor learn `x` and the vendor does not help the hospital learn `W`. The DPIA has to state this assumption explicitly and the contract between the parties has to codify it.

For 3-party honest-majority with a cloud compute party, the non-collusion assumption is that the cloud party does not collude with *either* data or model owner. In practice, pick a cloud party under a contract that makes collusion a breach.

### Pitfall 2 — Treating semi-honest as adequate for adversarial parties

Semi-honest security assumes the parties follow the protocol. A party that *deviates* — injects malicious shares, drops messages, replays — can break the semi-honest guarantee. If the parties are genuinely adversarial (e.g., they have a commercial incentive to cheat), you need malicious security. Malicious MP-SPDZ, SPDZ-2k, MASCOT. More expensive — orders of magnitude on some operations — but necessary.

### Pitfall 3 — Not planning the offline phase

The offline phase generates correlated randomness (triples) and is often the compute-heavy part of the overall protocol. A deployment that forgets to size the offline phase's throughput and storage will grind to a halt in production. Plan for `K` triples per inference and `N` inferences per day; the offline phase must sustain `K * N / 86400` triples per second.

### Pitfall 4 — Logging too much

Just like HE and TEE, the *protocol transcript* (intermediate shares, openings) can leak information if logged in the clear. MPC deployments should log only aggregate metrics and protocol-health indicators; never log a party's shares.

### Pitfall 5 — Deploying across continents

Round complexity is latency-sensitive. Deploying the parties in different continents turns a 10-millisecond-per-round protocol into a 150-millisecond-per-round one and scales accordingly. Keep the parties close.

### Pitfall 6 — Ignoring the model-architecture leakage

The architecture of the computed circuit is public to all parties. If the architecture *itself* is a trade secret, MPC does not hide it. "The model is a 12-layer transformer with GELU activations and a specific attention pattern" is leaked. For most ML deployments this is acceptable; if the architecture is the IP, say so and plan accordingly.

## MPC + DP — composing with chapter-05 (mod-106) DP controls

MPC protects the inputs during computation; it does not protect the *outputs*. If the output of the joint computation itself can leak the input (e.g., membership inference against an inference result), the output needs noise — a DP mechanism. The common pattern: compute `Σ or function` via MPC, add calibrated Gaussian or Laplace noise *inside the protocol* (so no party sees the pre-noise output), release the noisy result. This is the "distributed differential privacy" pattern and is where MPC meets mod-106's budget-tracking discipline.

The pattern avoids both the trust issue of a single party adding the noise (they'd learn the clean value) and the privacy issue of releasing the clean value (DP would be lost). It is the right pattern for most real MPC + privacy deployments.

## The MPC decision paragraph

The deliverable for an MPC-vs-HE-vs-TEE conversation:

> *This privacy-preserving inference uses {MPC / HE / TEE} because: (a) the data and model are held by {two different organisations}, with a {semi-honest / malicious} adversarial model; (b) the model is {logistic regression / small CNN / medium MLP} with {K multiplications and D multiplicative depth}; (c) the latency budget is {P50 X seconds, P99 Y seconds}; (d) the required protocol parties are {2 / 3 / N}, and the non-collusion assumption is {stated here}; (e) the implementing library is {CrypTen / MP-SPDZ / etc.} because of {reason}. HE was considered and rejected because {non-linear activation cost / model size / non-collusion not relevant}. TEE was considered and {picked / rejected} because {trust-in-silicon-vendor OK / regulator rejects TEE as sufficient / etc.}.*

Reading that paragraph in a design review tells the audience you have thought through the choice, not defaulted to the fashionable option.

## Summary — what you can do after this chapter

- State the MPC privacy guarantee: parties jointly compute a function of their inputs without any party seeing the others' inputs; privacy reduces to the non-collusion assumption and the protocol's threshold.
- Distinguish arithmetic protocols (fast linear algebra, polynomial non-linearities) from Boolean protocols (exact non-linearities, slow linear algebra) and understand why production MPC uses both.
- Articulate round complexity as the dominant cost metric and why same-depth-batched operations matter.
- Prototype a 2-party secure dot product or small-MLP inference with CrypTen.
- Know when to escalate to MP-SPDZ (malicious security, `n > 2` parties, auditable rigour) and know the operational tax that costs.
- Pick MPC over HE for medium-size models with exact non-linearities; pick HE over MPC for the one-client-one-server setting with a small model; pick TEE over both for LLM-scale and when the silicon-vendor trust is acceptable.
- Name the six pitfalls (non-collusion assumption, semi-honest vs. malicious, offline-phase planning, over-logging, cross-continent deployment, architecture leakage).
- Compose MPC with DP for distributed differentially-private release when the output itself can leak.

Chapters 05 and 06 have given you the cryptographic alternatives to TEE. Chapter 07 is where it all comes together — the risk-and-cost memo that the three chapters' mechanisms serve as the technical backing for.
