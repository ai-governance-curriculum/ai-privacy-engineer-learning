# Homomorphic Encryption for Inference — TenSEAL, Microsoft SEAL, CKKS vs. BFV, and the Performance Ceiling

A TEE eliminates the need to trust the operator; homomorphic encryption (HE) eliminates the need to trust the server to even execute the correct program on your data. The server, in an HE deployment, performs the inference computation directly on *ciphertext* without ever decrypting; the client holds the only key that can decrypt the result. There is no attestation chain, no root CA to trust, no side-channel to worry about on the server side — the server is cryptographically prevented from learning the plaintext. That is the most ambitious privacy guarantee this module names.

The price of that guarantee is performance. HE is orders of magnitude slower than cleartext computation; the operation set it supports is restricted; and the model size it can handle is a tiny fraction of modern LLM scale. This is not a tool for your production GPT-class LLM. It is a tool for the narrow, high-stakes use cases where the server must not be trusted at all — biomedical third-party scoring, financial compliance scoring against an outsourced model, privacy-preserving genomics, certain cross-border transfers where the receiving party is specifically prohibited from seeing the plaintext.

This chapter builds the mental model of HE for an inference engineer. It explains the two schemes you will actually touch (CKKS for approximate arithmetic on real-valued vectors, BFV for exact arithmetic on integers), walks what a HE inference deployment looks like with TenSEAL and Microsoft SEAL, and — critically — tells you where the performance ceiling is so you can set product expectations before you spend three months building something that cannot work.

Primary references:

- Gentry — *Fully Homomorphic Encryption Using Ideal Lattices*. STOC 2009. [dl.acm.org/doi/10.1145/1536414.1536440](https://dl.acm.org/doi/10.1145/1536414.1536440) — the original FHE construction; the paper that started the field. Read for the historical anchor even if the specific construction is not what you ship today.
- Fan, Vercauteren — *Somewhat Practical Fully Homomorphic Encryption*. IACR ePrint 2012/144. [eprint.iacr.org/2012/144](https://eprint.iacr.org/2012/144) — the BFV scheme; exact arithmetic on integers mod t. Shipped in SEAL.
- Cheon, Kim, Kim, Song — *Homomorphic Encryption for Arithmetic of Approximate Numbers* (CKKS). ASIACRYPT 2017. [eprint.iacr.org/2016/421](https://eprint.iacr.org/2016/421) — the CKKS scheme; approximate arithmetic on real / complex numbers; the one you reach for first for ML workloads.
- Brakerski, Gentry, Vaikuntanathan — *(Leveled) Fully Homomorphic Encryption without Bootstrapping*. ITCS 2012. [eprint.iacr.org/2011/277](https://eprint.iacr.org/2011/277) — the BGV scheme; also in SEAL (deprecated for new workloads but still cited).
- Microsoft — *Microsoft SEAL* (GitHub). [github.com/microsoft/SEAL](https://github.com/microsoft/SEAL) — the reference C++ implementation of BFV, BGV, and CKKS. The documentation is the de-facto HE API reference.
- OpenMined — *TenSEAL*. [github.com/OpenMined/TenSEAL](https://github.com/OpenMined/TenSEAL) — Python bindings over SEAL with tensor support; this is the library you will prototype against.
- Albrecht, Chase, Chen, et al. — *Homomorphic Encryption Standard*. HomomorphicEncryption.org, November 2018. [homomorphicencryption.org/standard/](https://homomorphicencryption.org/standard/) — the community standardisation effort; the reference for parameter choices and security levels.
- Gilad-Bachrach, Dowlin, Laine, Lauter, Naehrig, Wernsing — *CryptoNets: Applying Neural Networks to Encrypted Data with High Throughput and Accuracy*. ICML 2016. [proceedings.mlr.press/v48/gilad-bachrach16.html](https://proceedings.mlr.press/v48/gilad-bachrach16.html) — the seminal paper on HE inference over neural networks; the archetype of what you will build.

## The problem HE actually solves

Formally, a homomorphic encryption scheme provides algorithms `Encrypt`, `Decrypt`, and (at minimum) two operations `Add` and `Mult` that work on ciphertexts:

```
Decrypt(sk, Add(Encrypt(pk, a), Encrypt(pk, b)))  =  a + b
Decrypt(sk, Mult(Encrypt(pk, a), Encrypt(pk, b))) =  a * b
```

If the scheme supports an *unbounded* number of adds and mults, it is "fully" homomorphic (FHE). If the depth of the operations is bounded, it is "leveled" HE (the parameters are picked at setup time to tolerate a specific circuit depth). For practical ML, leveled HE is what you deploy — the parameters are chosen to support the specific circuit of your model, no more.

Two consequences for engineering:

- You design the model as a *circuit* of adds and mults. Non-linear activations (ReLU, softmax, GELU) cannot be done directly; they must be either polynomial-approximated, swapped for a HE-friendly activation (square, LeakyReLU-ish polynomial), or run in a separate round-trip to the client (expensive).
- The ciphertext grows in size (and in "noise budget") with each operation. There is a maximum number of operations you can perform before decryption fails. Bootstrapping (Gentry's original contribution) refreshes a ciphertext at substantial compute cost; most practical deployments avoid bootstrapping by picking parameters that accommodate the model's full circuit depth directly.

## CKKS vs. BFV — which scheme for which problem

The scheme choice is the first engineering decision. The two schemes you will actually touch have different numeric characters.

### CKKS (Cheon–Kim–Kim–Song)

- **Plaintext space:** vectors of complex numbers (in practice, vectors of real numbers).
- **Arithmetic:** approximate — small, controlled noise accumulates with each operation. This is a feature for ML, where everything is already approximate.
- **Why pick it for inference:** matmuls and additions over real-valued tensors (weights, activations) map naturally; one ciphertext holds a batch of values (SIMD over the "plaintext slots"), so a dense layer is a few ciphertext multiplications and rotations.
- **Weakness:** the noise is bounded but grows; the *precision* of the result degrades with the circuit depth. For deep networks the precision drop matters.

### BFV (Fan–Vercauteren)

- **Plaintext space:** vectors of integers modulo a prime `t`.
- **Arithmetic:** exact — no noise in the plaintext, only in the ciphertext.
- **Why pick it:** when you need exact-integer semantics — a counter, a bitmask-based privacy-preserving comparison, exact tabular lookups, quantised integer models.
- **Weakness:** the modulus `t` bounds the dynamic range of your computation. A naive deep network will overflow quickly unless you normalise aggressively.

### The rule of thumb

- ML inference on real-valued tensors (the typical case): **CKKS**.
- Exact-integer operations (quantised models, counting, exact comparisons): **BFV**.
- Both can live in the same TenSEAL program for different sub-operations.

## Parameter choice — what the numbers actually mean

A CKKS or BFV instantiation is a triple of parameters: the polynomial modulus degree `N`, the coefficient modulus `q` (a chain of primes), and the scale (CKKS only) or plaintext modulus `t` (BFV). The Homomorphic Encryption Standard (Albrecht et al. 2018) prescribes safe choices:

- `N = 2^{12} = 4096`: cheap, suitable for very shallow circuits (one or two multiplications before noise ruins the ciphertext). Not for ML in practice.
- `N = 2^{13} = 8192`: usable for small dense layers; supports ~4-5 multiplications. Logistic regression fits.
- `N = 2^{14} = 16384`: usable for a small CNN like CryptoNets (3 conv layers, polynomial activation).
- `N = 2^{15} = 32768`: usable for medium-depth networks. Ciphertext size balloons to ~1 MB each; operations are slow.

A larger `N` gives more "noise budget" (more operations before decryption fails) and more plaintext slots (SIMD width), at the cost of larger ciphertexts and slower operations. The parameter choice is a function of the model circuit depth and the required security level (typically 128-bit security is the modern minimum; the standard prescribes the `(N, log q)` pairs that achieve each security level).

**Picking parameters is the hardest part of the first HE project.** TenSEAL's `ts.context(ts.SCHEME_TYPE.CKKS, poly_modulus_degree=8192, coeff_mod_bit_sizes=[60, 40, 40, 60])` style API makes it concrete; the parameters there mean "4 levels of multiplication supported, with the primes in the coefficient-modulus chain sized to leave a reasonable noise budget". Each level consumed is one `multiply` operation; a `rescale` restores precision at the cost of one level.

## Microsoft SEAL — the reference C++ implementation

SEAL is the library, written in C++, that most HE-for-ML tooling binds to. Core concepts in the SEAL API, which you will meet whether you code SEAL directly or use TenSEAL:

- `EncryptionParameters` — the `(scheme, poly_modulus_degree, coeff_modulus, plain_modulus)` tuple.
- `SEALContext` — the derived state (small-primes decomposition, Galois keys, relin keys that the context will need).
- `KeyGenerator` → `PublicKey`, `SecretKey`, `RelinKeys`, `GaloisKeys` — the key material. Relin keys reduce ciphertext size after a multiplication; Galois keys enable slot rotations within a ciphertext.
- `Encoder` (BatchEncoder for BFV, CKKSEncoder for CKKS) — maps a plaintext vector into the "slots" of a plaintext polynomial.
- `Encryptor`, `Evaluator`, `Decryptor` — the three objects that do the work.
- `Plaintext` and `Ciphertext` — the data types.
- `Evaluator.add(a, b, dest)`, `.multiply(a, b, dest)`, `.rotate_vector(a, k, galois_keys, dest)`, `.relinearize_inplace(a, relin_keys)`, `.rescale_to_next_inplace(a)` — the operation set.

SEAL is a well-engineered library but it is *not* an ML library. You build matmuls and convolutions out of its primitives. This is fine for a logistic regression; it gets tedious very fast for anything bigger. TenSEAL exists to raise the abstraction level.

## TenSEAL — the Python tensor layer

TenSEAL ([github.com/OpenMined/TenSEAL](https://github.com/OpenMined/TenSEAL)) wraps SEAL in a Python / PyTorch-adjacent API. It adds:

- `ts.Context` for the SEAL context.
- `ts.PlainTensor` / `ts.CKKSTensor` / `ts.BFVTensor` — tensor-of-ciphertexts types with a shape and an API that supports `+`, `*`, `matmul`, convolutional operations, `polyval` (polynomial evaluation, used for activation approximations), and tensor rotations.
- Serialisation helpers to pass ciphertexts between client and server.

The TenSEAL API is intentionally close enough to NumPy / PyTorch that a logistic regression or a small CNN can be ported in a day. The code to encrypt a feature vector and run it through a logistic-regression layer:

```python
import tenseal as ts
import torch

# Server has these in cleartext — the model weights.
W = torch.tensor([[0.3, -0.5, 1.2, ...], ...])  # shape [n_features, n_classes]
b = torch.tensor([0.1, -0.2, ...])              # shape [n_classes]

# Client sets up HE context.
ctx = ts.context(ts.SCHEME_TYPE.CKKS,
                 poly_modulus_degree=8192,
                 coeff_mod_bit_sizes=[60, 40, 40, 60])
ctx.generate_galois_keys()
ctx.generate_relin_keys()
ctx.global_scale = 2**40

# Client encrypts the feature vector.
x = torch.tensor([...features...])
enc_x = ts.ckks_vector(ctx, x.tolist())

# --- Serialise enc_x and ship to server, along with the public parts of ctx ---

# Server runs the dense layer on ciphertext.
enc_logits = enc_x.matmul(W.T.tolist()) + b.tolist()
# Note: on the server, W and b are plaintext; the matmul is ciphertext × plaintext,
# the result is ciphertext. The server never saw x, only enc_x.

# --- Serialise enc_logits and return to client ---

# Client decrypts.
logits = enc_logits.decrypt()  # a Python list of floats ~ the true logits
```

That is the whole shape. The server did the inference without ever seeing the plaintext input; the client got the result. The property is held cryptographically — not by policy, not by attestation, not by the server's goodwill. A server operator with full root access on the inference host sees only ciphertexts in memory.

## Non-linear activations — the real challenge

The toy above has a linear classifier. Any neural network with non-linear activations (ReLU, GELU, sigmoid, softmax) does not fit directly in HE's add-and-multiply operation set. Four engineering choices:

### Option 1 — Polynomial approximation

Replace the activation with a low-degree polynomial on the operating range. `sigmoid(x) ≈ 0.5 + 0.197 x - 0.004 x^3` is a classical 3-term approximation on `[-5, 5]`. The CryptoNets paper used `square(x)` as an activation (literally `x^2`) because it fits HE perfectly — one multiplication — and showed that MNIST-level accuracy survives the substitution for small networks.

Consequences: you re-train the model (or at least fine-tune) with the chosen polynomial activation so the weights adapt to the different non-linearity. Accuracy will drop; validate on your task.

### Option 2 — Interactive protocol — client does the non-linearity

The server sends the pre-activation ciphertext back to the client, the client decrypts, applies ReLU, re-encrypts, sends back. This is correct but slow — each non-linearity is a network round-trip. For a 10-layer network, 10 round-trips per inference. Only viable for very small models with strict privacy requirements and tolerant latency.

### Option 3 — Garbled-circuit non-linearity (hybrid)

Combine HE for the linear parts and a garbled-circuit / Yao's-garbled-circuit protocol for the non-linearity. This is the "nGraph-HE", "Gazelle", "Chameleon", and (more recently) "Delphi" / "Cheetah" line of research. Enterprise-grade libraries are sparse; this is a research-depth direction not yet a one-line production library.

### Option 4 — Don't use HE; use MPC or TEE instead

The limitations above are often what pushes a privacy-preserving-inference project to MPC (chapter 06) or TEE (chapters 01-04). HE is correct when the non-linearity count is small, the model is small, and the latency budget is generous.

## Performance ceiling — the numbers you need to anchor product expectations

The order-of-magnitude facts your product partner needs to hear, early.

- **Encryption / decryption cost.** Encrypting a vector into a single CKKS ciphertext at `N=8192` takes milliseconds; at `N=32768`, tens of milliseconds. Not the dominant cost.
- **Ciphertext size.** A CKKS ciphertext at `N=16384, log q = 240` is ~400 KB. Transferring hundreds of them per inference is non-trivial bandwidth.
- **Multiplication cost.** A single ciphertext-ciphertext multiplication at `N=16384` takes on the order of tens of milliseconds on modern CPUs. At `N=32768`, hundreds. These numbers vary with hardware and SEAL version; <!-- needs-research: cite SEAL's published benchmarks or a recent survey (e.g., Benaissa et al., "TenSEAL: A Library for Encrypted Tensor Operations", 2021) for a specific ballpark. -->
- **Ciphertext-plaintext multiplication is faster.** When one operand is plaintext (the model weight), the cost is order-of-magnitude lower. This is why "server holds weights in plaintext, client encrypts features" is the common deployment.
- **Model-size ceiling.** CryptoNets-era results were for networks with tens of thousands of parameters. Deeper CNNs (ResNet-20, ResNet-32) have been published in the HE literature with inference times measured in minutes to tens of minutes per image. LLM-scale models (billions of parameters, non-linearities at every layer, long contexts) are outside the practical range as of this writing. Published research on LLM inference under HE exists (e.g., "LLMs go FHE") but at latencies that are not production-ready. <!-- needs-research: name and cite the specific recent results. -->
- **Bootstrapping is still slow.** Even the modern CKKS bootstrap (Han, Kim, et al. "Better bootstrapping for approximate homomorphic encryption", CT-RSA 2020, eprint 2018/1043) takes seconds per operation. If your circuit requires bootstrapping, you have multi-second latency per bootstrap call.

### The product conversation

Say this out loud when the PM asks "can we HE our LLM inference?":

> *HE is a real privacy guarantee, and it is a real tool. It is currently practical for small models — logistic regressions, decision trees, small CNNs, simple MLP tabular classifiers — with inference latency in the hundreds of milliseconds to a few seconds per request. It is not practical today for LLMs or any large vision model; the latency is minutes or longer, the ciphertexts are large, and the model's non-linearities (attention, softmax, LayerNorm) are expensive to approximate. If the use case is truly "server must not see the plaintext at all" and the model is small, HE is the right tool. If the use case is an LLM, HE is the wrong tool — a TEE deployment (chapters 01-04) is the production-practical path, with MPC (chapter 06) as the alternative for cross-organisation settings.*

That paragraph is the thing that prevents an HE prototype from being built for the wrong use case.

## When HE is the right tool — the three use cases

HE's performance ceiling is harsh enough that the use cases where it is the right tool are narrow but real.

### Use case 1 — Outsourced small-model scoring with "no visibility" contract

A hospital wants to score patient records against a third-party-held risk model. The third party is a commercial vendor; the hospital's legal position is "we cannot send plaintext PHI to a third party, even under BAA". The model is small (a few thousand parameters, a logistic regression or a small gradient-boosted tree approximated in a HE-friendly way). The throughput requirement is low (hundreds of scores per day, not per second). HE fits: the hospital encrypts, sends, decrypts the score. The vendor sees only ciphertexts.

### Use case 2 — Privacy-preserving genomics

Genomic data is extremely sensitive; the re-identification risk is high; the "even an insider of the service provider should not see this data" bar is argued hard by privacy advocates. HE implementations of GWAS (genome-wide association studies), ancestry inference, and small polygenic-risk-score models exist and have been deployed in research contexts. The model sizes are small enough to fit; the latency tolerance (hours, not seconds) is adequate.

### Use case 3 — Cross-border scoring where even a TEE's residual trust is contractually forbidden

Some data-protection authorities or specific contracts have taken the position that a TEE (whose trust ultimately reduces to AMD / Intel / NVIDIA / the cloud operator) is still insufficient for certain data flows — typically national-security-adjacent, or involving particular special-category data. HE removes the "we must trust the silicon vendor" term entirely; the trust is only in the mathematics of the lattice-problem hardness the scheme is based on. For small models where the trade-off is acceptable, HE wins.

### When HE is explicitly *not* the right tool

- LLM inference at production latencies.
- Any use case where the server also needs to correlate the input across multiple clients (HE is per-client; cross-client aggregation is a different problem — usually MPC).
- Any use case where the model itself is highly non-linear and the polynomial approximations hurt accuracy badly.
- Any use case where the attack surface the deployment actually faces is "curious cloud operator" rather than "server must not see the data mathematically" — the TEE is the engineering-pragmatic answer there.

## Deployment shape — client / server with HE

The end-to-end shape of an HE-based inference deployment:

- **Client holds the private key `sk` and the public key `pk`.**
- **Server holds the model weights (plaintext) and the public key (`pk`).**
- **Per request:**
  - Client encrypts the input `x` to `enc_x` with `pk`.
  - Client ships `enc_x` to the server over a secure channel (TLS).
  - Server computes `enc_y = model(enc_x)` using the HE operations against plaintext weights.
  - Server ships `enc_y` back to the client.
  - Client decrypts `enc_y` with `sk` to get the result `y`.
- **The server never has the capability to decrypt anything.** `sk` never leaves the client.

If the model weights are *also* the server's private secret, you have an added constraint — the client should learn `y` but not `W`. Linear models in HE naturally satisfy this: the client sees `enc(W x + b)` as a ciphertext dependent on `W` and the input, and decryption reveals only the result, not `W`. For deeper models, the "both private" setting is a two-party MPC problem (chapter 06).

### Transport + HE — belt-and-suspenders

TLS between client and server is still required; HE does not replace it. Reasons:

- Replay protection, integrity on the ciphertexts' transport, server authentication — TLS gives you all of these.
- The HE ciphertext is confidential for its *contents* but a replay attacker could still mount denial-of-service or billing-abuse attacks without TLS.

The confidentiality guarantee of HE is *in addition to* standard transport security, not instead of it.

## Pitfalls to avoid

Six mistakes recur in HE prototypes; any of them loses the guarantee.

### Pitfall 1 — Picking parameters that don't support the model's depth

The ciphertext's noise budget is finite. A deep model that consumes more levels than the parameters allocate will decrypt to garbage. Compute the circuit's multiplicative depth off-line before picking `N` and the coefficient-modulus chain.

### Pitfall 2 — Reusing the same secret key across sessions without rotation

The HE schemes are IND-CPA secure (secure against chosen-plaintext attack). They are not IND-CCA secure. If the attacker can observe the client's decryptions of chosen ciphertexts, they can mount an attack. Rotate client keys per session where possible; never allow the server to submit chosen ciphertexts for the client to decrypt.

### Pitfall 3 — Leaking the plaintext via the circuit shape

The circuit the server runs is itself a (public) function. If the circuit is data-dependent — e.g., "run this branch if the first feature is positive" — the branch selection leaks the plaintext. Keep all circuits data-independent; evaluate both branches and select the result with a multiplexer built out of HE multiplications.

### Pitfall 4 — Misinterpreting approximate decryption as a leak

In CKKS, the decrypted result is approximate — within a small noise range. This is normal and expected; it is not a bug and not a leak. Document it so the downstream consumer handles approximate results correctly.

### Pitfall 5 — Shipping Galois keys that enable too many rotations

TenSEAL's `generate_galois_keys()` without arguments generates a full suite. If the model only needs specific rotations, generate only those — Galois keys are *large* and bandwidth is finite.

### Pitfall 6 — Treating HE as a drop-in replacement for the model's cleartext behaviour

The polynomial-activation substitution, the quantisation to fit the scheme's precision, the pre-encryption normalisation of the input — all of these change the model's effective decision function. Re-validate accuracy, re-run the fairness audit, re-measure the model's behaviour on the task. HE is not transparent to ML quality.

## Summary — what you can do after this chapter

- State the HE guarantee precisely: "the server performs the inference on ciphertext and never has access to the plaintext; the privacy is cryptographic, not operational".
- Choose between CKKS (approximate arithmetic on real vectors) and BFV (exact arithmetic on integers) for a given problem.
- Read a TenSEAL / SEAL parameter triple and know what each knob does (poly modulus degree `N`, coefficient modulus chain, plaintext scale for CKKS, plaintext modulus `t` for BFV).
- Prototype a logistic-regression or small-MLP HE inference deployment using TenSEAL.
- Handle non-linear activations via polynomial approximation (and know the accuracy cost), interactive client-side evaluation, or the GC-hybrid path at a research level.
- Articulate the performance ceiling: HE is practical for small models with modest throughput; it is not practical for LLM-scale inference as of this writing.
- Name the three use cases where HE is the right tool (outsourced small-model scoring, privacy-preserving genomics, cross-border cases where TEE residual trust is forbidden).
- Avoid the six common pitfalls: insufficient circuit depth, key reuse without rotation, data-dependent circuit shape, approximate-decryption confusion, over-broad Galois keys, HE as cleartext drop-in.

Chapter 06 covers the other cryptographic path — Secure Multi-Party Computation — which is the right tool when the data is split across multiple non-colluding parties (hospitals, banks) rather than a single client-plus-server shape.
