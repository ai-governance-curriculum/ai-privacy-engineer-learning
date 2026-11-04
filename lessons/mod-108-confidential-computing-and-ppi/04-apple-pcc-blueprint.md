# Apple Private Cloud Compute — The Published Enterprise Blueprint and What It Means for Your Deployment

In June 2024 Apple published *Blog post: Private Cloud Compute — A new frontier for AI privacy in the cloud* and a companion *Security Guide* describing the end-to-end architecture of Private Cloud Compute (PCC) — the cloud component of Apple Intelligence. It is the single most-detailed first-party description of a production, consumer-scale confidential-AI deployment that has ever been published. Whether or not you deploy on Apple hardware, PCC is the design reference the sophisticated enterprise reviewer will measure your architecture against. When a regulator or a chief privacy officer asks "what would a confidential-AI deployment at Apple quality look like?", the PCC design is the answer they have in mind.

This chapter is a *design read* of PCC. We do not need you to clone PCC; we need you to extract its load-bearing properties, translate each one into a general engineering requirement, and audit your own deployment against the list. By the end, you should be able to walk into a design review with "here is where our deployment meets each PCC property, and here is where it does not — and here is why".

Primary references:

- Apple — *Private Cloud Compute: A new frontier for AI privacy in the cloud*. Blog post, 10 June 2024. [security.apple.com/blog/private-cloud-compute/](https://security.apple.com/blog/private-cloud-compute/) — the design paper. Read in full.
- Apple — *Private Cloud Compute Security Guide*. [security.apple.com/documentation/private-cloud-compute/](https://security.apple.com/documentation/private-cloud-compute/) — the longer-form documentation.
- Apple Security Research — *Private Cloud Compute: Security research*. [security.apple.com/documentation/private-cloud-compute/securityguide](https://security.apple.com/documentation/private-cloud-compute/securityguide) and the public research image at [security.apple.com/private-cloud-compute/research-vm/](https://security.apple.com/private-cloud-compute/research-vm/) — the virtual-research-environment image Apple publishes for third-party inspection.
- Apple — *PCC transparency log* publication. [security.apple.com/private-cloud-compute/transparency/](https://security.apple.com/private-cloud-compute/transparency/) — the certificate-transparency-style log of published PCC software images.
- Apple Secure Enclave reference — Apple Platform Security guide, Apple Silicon security chapter. [apple.com/support/platform-security/](https://support.apple.com/guide/security/welcome/web) — background on the Secure Enclave Processor (SEP) and the keychain infrastructure the client side depends on.
- Devan — *How Apple Private Cloud Compute Works* (third-party technical read). <!-- needs-research: find a reputable third-party technical write-up to cite for cross-validation; multiple exist but of variable quality. -->

The chapter walks each of PCC's five published core properties, translates each into a general engineering requirement, and calls out the gap between "PCC quality" and "typical enterprise confidential-AI deployment".

## PCC's five core properties

Apple names five properties PCC's design is built to guarantee. Memorise them; they are the rubric.

1. **Stateless computation on personal data.** No data is persisted; a request is processed and the state torn down.
2. **Enforceable guarantees.** The privacy property is enforced by the platform, not by policy or employee discipline.
3. **No privileged runtime access.** No Apple employee can gain shell/debug access to PCC nodes processing user data.
4. **Non-targetability.** No external or internal actor can single out a specific user's requests to send to a compromised node.
5. **Verifiable transparency.** Each production PCC software image is published to a tamper-evident log; researchers can download the image, reproduce its binary, and verify what is running.

Each of these is more than marketing. Each ties to a specific engineering mechanism. We take them one at a time.

## Property 1 — Stateless computation on personal data

### What PCC does

A PCC node processes a user request and discards all state at the end of the request. The node has no persistent writable storage of user data; the OS image is read-only; the memory is wiped (or freed and overwritten) at request completion; there is no logging path that retains the user's prompt or the model's response on the node. Even if a node were compromised later, there is no historical corpus of user data on it to exfiltrate.

### The engineering requirements this translates to

- **Immutable root filesystem.** The node boots from a signed, read-only image. If the image cannot write to its own root filesystem, there is no path for state to persist to disk.
- **In-memory only prompt handling.** The prompt and the response live in RAM and are discarded before the enclave accepts the next request. No write to tmpfs, no swap (confidential VMs disable swap by policy), no debug core dumps.
- **Explicit memory sanitisation.** When a request completes, the memory pages that held the prompt and activations are explicitly zeroed or securely freed before the next request. On NVIDIA H100 CC the GPU HBM is similarly sanitised; the KV cache is torn down.
- **Log policy of "aggregate telemetry only".** The node emits operational telemetry — request latency, error rate, token counts, model-version-in-use — that *cannot* be used to reconstruct a request. No PII, no prompt text, no response text, no per-user identifier. This is a code review you perform on every logging path.
- **Short-lived session keys with immediate destruction.** The ephemeral session key the enclave used to decrypt the request is destroyed; a new key is generated for each session.
- **No node-local caching across requests.** If a request's intermediate state would otherwise be cached (e.g., a prefix of KV cache that could be reused across a user's requests), the caching is session-scoped and torn down at session end.

### The gap most deployments have

Default telemetry pipelines. A "privacy-preserving" inference server that nonetheless ships the user's prompt to an APM tool for latency analysis has lost the stateless property. Default debug logging. A `debug_level=2` configuration that writes the first 500 tokens of every request to syslog breaks the property. Default caching. An "efficiency optimisation" KV-cache sharing across users for common prefixes breaks the property — one user's cached state becomes available to another user via timing side channels.

**The rule of thumb:** list every write path from the enclave to any persistent store or any external service. Each is a trust boundary the stateless property forbids *unless* the data is cryptographically protected or fully anonymised. If the list has entries you cannot defend, the stateless property is not held.

## Property 2 — Enforceable guarantees

### What PCC does

Apple's design is that the privacy properties are not enforced by a Security Operations team's vigilance, by a policy document, or by employee training. They are enforced by *platform machinery* — code-signing on the OS image, hardware attestation on the node, cryptographic key release gated by attestation, and software constraints that prevent an insider from deviating from the policy even if they wanted to. A PCC node that strayed from the published image would fail attestation and would not be able to decrypt incoming requests; the policy does not need to catch the strayed node because it cannot do its intended work.

### The engineering requirements

- **The attestation chain is the enforcement point.** Chapter 02's KMS-gated session-key release is exactly this property. The node cannot decrypt a prompt unless its attestation passed; the attestation can only pass if the image measurement matches the signed known-good set; therefore an insider who swaps in a different image is physically unable to read user prompts.
- **No human-privileged escape hatch.** The deployment has no "break-glass" admin path that lets a human decrypt a request outside the attestation chain. If the ops team needs to diagnose a request, they do it with the *aggregated* telemetry; the raw data is not available by design.
- **Policy-is-code and policy-is-signed.** The KMS policy itself is version-controlled, reviewed, and signed. Changes to the policy are tracked in a tamper-evident log. An insider who tried to add a new measurement to the allow-list leaves evidence.
- **Trust is reducible to a small, auditable set of signatures.** A reader of the policy can enumerate every trusted certificate: AMD root, NVIDIA root, the deployer's own image-signing root, the deployer's attestation-verifier signing key. Nothing is implicit.

### The gap most deployments have

"The security team will audit accesses." That is a *procedural* control; PCC's property is that there are no accesses to audit because the platform makes them impossible. If your DPIA argues "we trust our ops team not to peek", you have a procedural guarantee, not an enforceable one.

**The rule of thumb:** for every "we don't do X" privacy claim, ask "what platform mechanism prevents an insider from doing X anyway?". If the answer is "they wouldn't" or "they'd be caught", the guarantee is procedural. If the answer is "the KMS wouldn't release the key" or "the attestation would fail", the guarantee is enforceable.

## Property 3 — No privileged runtime access

### What PCC does

The PCC node's runtime has no shell, no remote-access service, no SSH, no debug port. There is no mechanism for an Apple employee — no matter how senior, no matter what tooling they bring — to open an interactive session on a running PCC node that is processing user data. The nodes are "sealed" not just at build time but at runtime. Debugging a production issue is done entirely through the aggregated telemetry, or by rolling a specific node out of the user-serving fleet and only then opening it up (at which point it is handling no live user data).

### The engineering requirements

- **No interactive services on production enclaves.** Remove SSH, remove any remote-admin daemon, remove `kubectl exec`-equivalent paths, disable any backdoor the base image might ship with. The image allow-list enforces this because an image with SSH enabled produces a different measurement than one without.
- **Serial console disabled.** The hypervisor-level serial-console path that a cloud operator could normally use to access a VM is not available on confidential VMs by policy (and on PCC specifically, is not wired up at all).
- **Debugging is "roll-out-and-attach".** When a node needs forensic inspection, it is removed from the serving pool (so it processes no new requests), its in-flight sessions are drained, its memory is wiped, and only then is a different image (one that allows interactive access) booted on the same hardware. The transition requires a new attestation.
- **No "shadow" telemetry endpoint.** The telemetry schema is signed off as part of the image review. There is no path to turn on a debug firehose on a production node.

### The gap most deployments have

A "break-glass" SSH path for incident response. A `kubectl exec` enabled on production confidential pods for "operability". A debug flag that can be flipped by an ops engineer at 3 AM. These all break the property in exactly the way PCC's design is intended to prevent.

**The rule of thumb:** draw the list of ways an operator could gain runtime access to the node. Each entry is a confidentiality hole. PCC's design is that the list is empty.

## Property 4 — Non-targetability

### What PCC does

An attacker who has compromised a single PCC node (or placed a single malicious node in the fleet) cannot force a specific user's request to land on that node. Request routing is designed such that no external party — and no insider — can single out a specific user. The user's device establishes a session with the PCC fleet and the routing within the fleet is designed to be opaque to any observer. If an attacker's goal is "see Alice's prompts", they cannot achieve it by compromising one node; they would have to compromise the entire serving fleet, which is implausible.

### The engineering mechanisms

PCC achieves non-targetability through a combination of:

- **Oblivious HTTP-style routing.** The client's identity is decoupled from the routing decision via an intermediate relay that strips identity before forwarding to the PCC fleet. The PCC node sees an unauthenticated request; it does not know which user the request originated from.
- **No persistent user-to-node affinity.** Requests from the same user go to different nodes across sessions; there is no "session stickiness" that would let an attacker's compromised node accumulate a target user's requests.
- **Request-payload-based routing is suspect.** Any routing decision that looks at the request contents is a potential targeting vector. PCC's design specifically routes based on properties that are uncorrelated with user identity.

### The engineering requirements this translates to for your deployment

- **Client identity is terminated before the enclave.** The enclave sees a request body and a session key; it does not see the user's identity. The user's identity is handled by a separate component (the relay / the authentication service) that does not learn the request contents.
- **Routing is identity-blind.** Load-balancing is round-robin or based on load, not on user ID, device ID, or request contents. Logs of routing decisions carry no user identifier.
- **The fleet is large and homogeneous.** A smaller fleet with distinguishable nodes is more vulnerable to targeting than a larger fleet with cryptographically indistinguishable nodes. "Many nodes, all attested to the same image" is the shape.
- **No "VIP" routing.** Routing a "high-value customer" to a dedicated pool is a targetability vector by construction. If the enterprise contract demands it, that contract has a different confidentiality claim than the PCC property.

### The gap most deployments have

Session stickiness for cache efficiency. Per-customer dedicated pools for "performance". Tenant-ID-based routing for billing. Each is a non-targetability failure.

**The rule of thumb:** can an attacker who compromises one node and knows a target user's identity select for that user's traffic? If yes, non-targetability does not hold. Ask the question with real adversarial intent.

## Property 5 — Verifiable transparency

### What PCC does

Every production PCC software image is published to a tamper-evident transparency log, similar in architecture to Certificate Transparency for TLS certificates (RFC 6962). Independent researchers can:

- Download the exact software image a given PCC node is running.
- Reproduce the image from source (the build is deterministic).
- Inspect the binary against the published claims.
- Verify that the transparency log is append-only and that the image actually logged matches the image the fleet attests to.

Apple also publishes a Virtual Research Environment that lets a researcher boot the image inside a VM and test it. The combination is a research-grade "verifiable claim" — not a marketing claim.

### The engineering requirements

- **Deterministic builds.** The image build pipeline produces bit-identical output from the same source tree. Without this, "reproducibility by a third party" is empty.
- **Public source availability.** Enough of the software is open-source (or at least publicly inspectable) that a third party can audit it. PCC is not fully open-source — Apple published components, source for some — but the attested runtime is sufficiently published that a researcher can audit it.
- **Append-only transparency log.** Every image version is committed to a public log that cannot be rewritten. Compare against the CT precedent: Signed Tree Heads, Merkle inclusion proofs, witness quorums.
- **Attestation-to-log binding.** The attestation claim must reference an image digest that is *in* the transparency log. A deployment that attests to an image not in the log is invalid.
- **Researcher incentive.** A bug-bounty or security-research program for PCC that encourages third-party inspection.

### The gap most deployments have

No published image. No deterministic build. "The security team reviewed the image" with no external verifiability. This is the most ambitious of the five PCC properties and the one that enterprise deployments most commonly fall short of.

**The rule of thumb:** can a motivated external researcher independently verify that the image running in your confidential-AI fleet is what you claim it is, without trusting your word? If no, verifiable transparency does not hold. PCC's property is intended to make the answer "yes".

## Applying the PCC rubric — a design-review template

When you review a confidential-AI deployment (your own or a vendor's), produce a one-page audit against the five properties. The template:

```
Deployment: {name}
Reviewer: {you}
Date: {date}
Trust claim: {one-line claim the deployment makes}

Property 1 — Stateless computation on personal data
  Mechanism: {what enforces it}
  Compliance: {Full / Partial / None}
  Gaps: {list}
  Risk if not held: {description}

Property 2 — Enforceable guarantees
  Mechanism: {what enforces it}
  Compliance: {Full / Partial / None}
  Gaps: {list}

Property 3 — No privileged runtime access
  Mechanism: {what enforces it}
  Compliance: {Full / Partial / None}
  Gaps: {list}

Property 4 — Non-targetability
  Mechanism: {what enforces it}
  Compliance: {Full / Partial / None}
  Gaps: {list}

Property 5 — Verifiable transparency
  Mechanism: {what enforces it}
  Compliance: {Full / Partial / None}
  Gaps: {list}

Residual risk summary:
Recommendations:
```

Walking the five properties out loud at a design review is the fastest way to find the weak joint in a deployment. Teams that have never done this exercise typically discover gaps in properties 1, 3, and 5.

## PCC-adjacent mechanisms worth knowing about

A few supporting details the PCC design mentions that are worth naming so you recognise them in conversation.

### Secure Enclave Processor (SEP) on Apple Silicon

PCC nodes run on Apple Silicon servers (M-series) that have a dedicated Secure Enclave Processor for key operations. The SEP is a separate silicon domain from the main CPU — architecturally analogous to AMD's SP and NVIDIA's device identity hardware. The client side (iPhone / Mac) also has an SEP; this is why the end-to-end PCC flow can bind client-side keys to the device's SEP and server-side keys to the PCC node's SEP, with a hardware-backed chain of custody throughout.

### Code-signing + launch policy on PCC

PCC nodes boot from an image that is signed with Apple's PCC signing key. The boot loader verifies the signature; the measurement of the launched image is published to the transparency log. A node whose launch measurement does not match a published image cannot join the fleet and cannot serve user requests.

### The "research image" mechanism

Apple publishes a *Virtual Research Environment* (VRE) — a downloadable VM image researchers can boot locally to audit the PCC software. The VRE is the *same* image the production fleet runs, published for inspection. A researcher who finds a vulnerability can reproduce it in the VRE and report it through Apple's Security Bounty program. This is the operational realisation of "verifiable transparency".

### How PCC uses the iPhone's end-to-end keypair

The client's device keypair is kept in the SEP. When a client sends a request to PCC, it uses the attestation-bound server public key to encrypt the request; the PCC node decrypts inside the enclave; the response is encrypted back to the client's SEP-held public key. The PCC node has never seen the client's long-term keys and cannot correlate requests across sessions because each session uses a fresh client-side ephemeral key.

### Comparison to our chapter-02 flow

The sealed inference path from chapter 02 is the generic shape; PCC is a worked example of that shape with additional constraints (stateless computation, non-targetability, verifiable transparency). If chapter 02 is "what the cryptography looks like", PCC is "what the cryptography looks like when you take all five properties seriously".

## What PCC does not do

A grounded reading of PCC should also name its limits so you can position your own deployment honestly:

- **PCC does not guarantee anything about the Apple Intelligence models themselves.** If a model memorised training data (mod-105) or leaks via differential-privacy violation (mod-106), PCC does not help. The model-output privacy is a separate discipline. Apple has described training practices around this, but it is distinct from the PCC transport guarantee.
- **PCC is not a response to "I don't want Apple to have my data at all".** PCC is a response to "I want Apple to process my data without seeing it". A user who objects to Apple processing their data at all is answered by on-device processing (mod-107 federated / on-device models), not PCC.
- **PCC depends on Apple's root CA and signing-key management.** Just like SEV-SNP depends on AMD and H100 CC depends on NVIDIA, PCC is only as secure as Apple's custody of the PCC signing root. This is a baseline trust assumption, not a cryptographic proof.
- **PCC's verifiable-transparency scope is bounded.** The published image is the one that runs; researchers cannot verify behaviour at every moment of every request's execution, only that the running code matches the published image. Side channels and in-enclave bugs are still possible.

PCC is an upper bound on what is currently achievable, not an absolute guarantee. Position it as "what a well-designed confidential-AI deployment looks like today", not "a solved problem".

## Translating each PCC property into your deployment's requirements, in table form

| PCC property | Engineering requirement you own |
| --- | --- |
| Stateless computation | Immutable root FS; no persistent user state; aggregated telemetry only; memory sanitisation; no cross-request caching |
| Enforceable guarantees | KMS-gated session keys; measurement allow-list; signed policy; no break-glass path; auditable trust-chain |
| No privileged runtime access | No SSH / shell / debug port; serial console off; drain-and-attach forensics model |
| Non-targetability | Identity terminated before enclave; identity-blind routing; no session stickiness; large homogeneous fleet |
| Verifiable transparency | Deterministic build; publicly inspectable image; transparency log with Merkle proofs; attestation-to-log binding; third-party research program |

A deployment that can tick all five columns has reached PCC quality. A deployment that ticks three has a defensible privacy claim for most enterprise buyers. A deployment that ticks fewer than three has a marketing claim, not an engineering one.

## Summary — what you can do after this chapter

- Name PCC's five core properties from memory: stateless, enforceable, no privileged runtime access, non-targetability, verifiable transparency.
- Translate each property into the specific engineering mechanism it requires.
- Audit your own confidential-AI deployment using the PCC rubric and produce the one-page review document.
- Identify the gaps most deployments have (telemetry breaking statelessness, break-glass paths breaking enforceable guarantees, SSH breaking no-privileged-access, session stickiness breaking non-targetability, no deterministic build breaking verifiable transparency) and name the fix for each.
- Articulate what PCC does *not* do (model privacy, on-device alternatives, root-CA trust assumption, bounded transparency scope) so you can defend your own deployment's limits honestly.
- Position PCC as a published enterprise reference the next time a reviewer asks "what does this look like at best-in-class?".

Chapters 05 and 06 switch to the cryptographic alternatives: homomorphic encryption and multi-party computation. These are the tools you reach for when the TEE's residual trust assumption — however small — is still too generous for the use case.
