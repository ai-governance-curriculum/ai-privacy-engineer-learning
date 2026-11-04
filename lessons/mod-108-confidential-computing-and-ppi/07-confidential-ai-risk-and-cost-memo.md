# The Confidential-AI Risk and Cost Memo — Threat Defeated, Residual Trust, Latency and Dollar Delta

A confidential-AI deployment costs materially more than the non-confidential baseline. The CPU TEE SKU is more expensive than the regular SKU; the GPU CC-mode has throughput overhead; the attestation infrastructure has operational cost; the policy discipline has a people cost. In exchange, the deployment defeats a specific set of threats and leaves a specific residual trust assumption. The decision to pay that cost is a *business* decision, informed by the privacy engineer and the mlops engineer but ultimately owned by the product, legal, and finance leads. Their input artefact is a short, structured memo.

This chapter is the authoring guide for that memo. Every previous chapter in this module exists so you can fill the memo in defensibly; every claim in it is backed by the mechanisms chapters 01–06 built.

The template below is prescriptive. Copy it, fill it in, iterate with the stakeholders. Keep the memo short — one to three pages. Reviewers who have to read a 20-page confidential-AI memo will skip to the last paragraph; you want them to read the whole thing.

## The purpose and audience of the memo

Who the memo is for, and what each reader will do with it:

- **Product / business owner.** Decides whether the extra cost buys enough privacy to justify the price delta or the slower feature. Reads the "What threat is defeated" and "What is the cost" sections.
- **Legal / DPO / privacy officer.** Needs the memo as the input to the DPIA and to the contractual claim made to customers. Reads the "Residual trust", the "What this does NOT defeat", and the "Boundary to other controls" sections.
- **CISO / security officer.** Validates that the threat model, the attestation policy, and the residual risks are what they signed up for. Reads the whole thing.
- **Finance / procurement.** Approves the recurring spend. Reads the "Cost delta" section.
- **Auditor / external regulator (eventually).** May receive the memo as evidence. Does not need to know it exists during authoring; writing it well means it will survive that future reader.

The memo is therefore *structured for multiple readers*. The threat / residual / cost sections are short enough that each reader can find theirs in 60 seconds.

## The memo structure

Six sections. Keep them in this order.

```
1. Deployment summary
2. What threat is defeated
3. What residual trust remains
4. What this specifically does NOT defeat
5. Cost delta (latency, throughput, dollars, people)
6. Boundary to the other privacy / security controls
```

A seventh "Decision" section at the top is optional; most organisations prefer the decision come at the end so the reader earns it by reading the memo.

### Section 1 — Deployment summary

One paragraph. Names the system, the shape of the confidentiality (CPU TEE + GPU TEE, which model, which protocol), the hyperscaler, and the business purpose.

*Shape:*

> *The {system name} serves {LLM / small-model / cross-organisation} inference to {consumer / enterprise / regulated-vertical} users. It runs on {Azure NCC H100 v5 / GCP A3 Confidential VM / AWS Nitro Enclaves + ...} with {SEV-SNP / TDX} CPU TEEs and {NVIDIA H100 CC} GPU confidential computing. User prompts are {end-to-end encrypted from the client} to the enclave using a session key derived via ECDH against an attested public key. The attestation chain terminates at {AMD / Intel / NVIDIA / AWS / Apple}'s root CA; our relying-party policy at {KMS name} releases decryption keys only when the measurement, TCB version, and {GPU CC-mode} claims pass.*

If there is an alternative cryptographic layer (HE or MPC), it is named here too.

### Section 2 — What threat is defeated

The clearest section. State the three TEE-defeated threats explicitly and concretely.

*Shape:*

> *The deployment defeats:*
>
> *(a) **Curious cloud operator.** A privileged employee or automated system at {Azure / GCP / AWS} with access to the hypervisor, the live-migration path, or the physical host cannot read the plaintext of user prompts, model weights, or activations. They read only ciphertext at the host-memory and PCIe boundaries. This is enforced by {SEV-SNP / TDX} CPU memory encryption and {H100 CC} GPU memory encryption.*
>
> *(b) **Curious co-tenant.** A tenant sharing the same host machine — via hypervisor-side vulnerabilities, Spectre-family side channels, or direct hardware access — reads only ciphertext of the enclave's memory. The hardware-enforced isolation is independent of the hypervisor's correctness.*
>
> *(c) **Curious privileged insider.** A datacentre technician with physical access to the server, a BMC debug operator, or a BIOS-level engineer cannot read the plaintext of the enclave's working state from the hardware they can physically reach. Cold-boot, DMA, and BMC debug reads return ciphertext.*

For an HE deployment the paragraph changes: "the server is cryptographically prevented from learning the plaintext at all; the privacy guarantee is mathematical, not operational". For an MPC deployment: "no party learns the other's inputs under the non-collusion assumption stated in section 3".

### Section 3 — What residual trust remains

The hard section. This is where the memo earns the reviewer's confidence by being honest about what still has to be trusted.

For a TEE deployment:

> *The confidentiality property reduces to the following trust assumptions, each of which is a baseline we inherit from the provider:*
>
> *(i) **Silicon vendor key custody.** {AMD / Intel} correctly manages the {VCEK / PCK} root certificates and has not had those keys exfiltrated. {NVIDIA} correctly manages the H100 CC device-identity CA. These are organisation-level trust assumptions; our deployment cannot verify them cryptographically beyond pinning the published roots.*
>
> *(ii) **Side-channel mitigation timeliness.** Side-channel attacks against SEV-SNP / TDX / H100 CC are an ongoing research area. Known-variant mitigations are shipped as microcode / firmware updates by the vendor. Our attestation policy enforces a {TCB version floor of X}, reviewed on each CVE disclosure, which bounds the residual window. The DPIA notes {N days} as the target mitigation window.*
>
> *(iii) **Hyperscaler platform integrity.** {Azure / GCP / AWS} correctly implements the TEE host-support firmware, the attestation verifier, and the KMS attestation condition enforcement. {The provider's SOC 2 / ISO 27001 / FedRAMP certification} is the evidence base.*
>
> *(iv) **Our own policy correctness.** The measurement allow-list, the TCB floor, the report-data binding, and the KMS release policy are correctly configured and reviewed by {role}. A review cadence of {quarterly / on each firmware update} is operated. See release-package section {N}.*
>
> *(v) **Our enclave software correctness.** The inference server binary inside the enclave has been reviewed for application-level vulnerabilities, follows supply-chain scanning on {interval}, and does not emit the plaintext prompt in telemetry or logs. Application-layer bugs can defeat the TEE guarantee; this is the attack surface we own.*

For an HE deployment the paragraph collapses to lattice-problem hardness + key-management discipline. For an MPC deployment it expands to add the non-collusion assumption explicitly:

> *(vi) **Non-collusion.** {Party A} and {Party B} do not collude. The commercial contract between the parties, dated {date}, treats collusion as a material breach. The enforcement is contractual, not cryptographic.*

The DPO will quote this section verbatim. Every clause has to map to a defensible position.

### Section 4 — What this specifically does NOT defeat

The reviewer-trust-building section. By explicitly naming what the control does not defeat, you prevent the control from being sold past its design.

> *The deployment does **not** defend against:*
>
> *- **Compromise of {AMD / Intel / NVIDIA / AWS / Apple}'s silicon-signing root.** Mitigation: baseline trust assumption; monitoring of vendor security disclosures.*
>
> *- **Zero-day side-channel attacks.** Mitigation: the TCB-version floor and the vendor's microcode-patch pipeline; sensitive workloads additionally use {HE / MPC / further controls} as specified.*
>
> *- **Replay / rollback attacks against persisted enclave state.** Mitigation: the enclave is {stateless / uses a freshness protocol with monotonic counters}; chapter 02 / chapter 04 reference.*
>
> *- **Model-output leakage of training data (memorisation, inversion).** Mitigation: mod-105's extraction testing + mod-106's DP training controls. The TEE protects the prompt in flight; it does not make the model's outputs non-leaky.*
>
> *- **Bugs in our own enclave software.** Mitigation: supply-chain scanning, memory-safe language where possible, pre-production fuzzing, logging-policy review.*
>
> *- **Weakness in our relying-party policy.** Mitigation: the policy is version-controlled, peer-reviewed, and audited on {interval}. Changes require {approval-chain}.*
>
> *- **User's own device / client-side compromise.** Mitigation: client-side attestation verification (chapter 02 Step 9); users on compromised devices cannot be protected by server-side TEE alone.*

A reviewer who sees this section drawn tight will trust the sections on either side of it. A reviewer who sees only the "defeats" section will not.

### Section 5 — Cost delta

The finance-and-ops section. Quantify the cost of confidentiality against the non-confidential baseline. Three sub-costs: compute, operations, people.

#### Compute cost

```
Non-confidential baseline:
  Instance SKU: {e.g., Standard_ND96amsr_A100_v4 (Azure) or a3-highgpu-8g (GCP)}
  On-demand price: ${X}/hour
  Throughput: {Y} inferences/second

Confidential-AI SKU:
  Instance SKU: {e.g., Standard_NCC40ads_H100_v5 (Azure) or a3-megagpu-8g confidential (GCP)}
  On-demand price: ${X'}/hour     [delta: +{Z}%]
  Throughput: {Y'} inferences/second     [delta: -{W}%]

  Effective ${per-inference}: {ratio}×  non-confidential baseline
```

The fields come from your cloud bill and from your benchmarking harness. The throughput overhead of GPU CC-mode is published by NVIDIA and measured by third parties; <!-- needs-research: cite a specific published benchmark number from NVIDIA's CC documentation or a reputable third-party study; numbers reported include single-digit to ~10% ranges for various workloads; verify for the specific model / sequence length / batch size. -->

#### Latency delta

```
Baseline P50 / P95: {M} ms / {N} ms
Confidential P50 / P95: {M'} ms / {N'} ms

  Added latency from:
    - CPU TEE memory-encryption overhead: ~{K}%
    - GPU CC-mode PCIe encryption: ~{K'}%
    - Attestation handshake (one-time per session): ~{A} ms
    - KMS key release (one-time per session): ~{B} ms
```

Attestation + KMS handshake are once-per-session, not per-request, so for interactive sessions they amortise; for one-shot requests they dominate.

#### Operational cost

```
Attestation verifier:
  Hosted ({MAA / Confidential Space / in-house}): ${X}/month at our volume
KMS key operations:
  Attestation-conditional key releases: ${Y} per million operations
Image build + measurement computation pipeline:
  Additional CI minutes per release: {Z} minutes × {N} releases/year
TCB-floor review (recurring):
  Analyst time per CVE disclosure: ~{H} hours
Attestation-policy review (recurring):
  Legal / privacy-engineer time per policy change: ~{H'} hours
Fleet patching for microcode CVEs:
  Operational cost owned by `ai-infra-security` (chapter 08); flagged for cross-charge visibility
```

#### People cost

```
Specialised roles:
  Privacy engineer (this role) — {headcount × salary} portion
  AI infra security (sibling role, chapter 08) — portion
  Dedicated attestation / KMS operator — {role / fraction}
Training / certification:
  Cross-team familiarity with the attestation model: ~{H} hours × {N} engineers
Vendor relations:
  Direct contact with {AMD / NVIDIA} for CVE disclosures and TCB alerts
```

The numbers here are realistic: a confidential-AI deployment is roughly 1.5–3× the compute cost of the non-confidential baseline in the published literature, with more variance by SKU and provider, plus a meaningful operational overhead. <!-- needs-research: validate the "1.5-3x" range against a reputable published comparison; this varies substantially and should be checked against the specific SKU / workload before quoting. -->

### Section 6 — Boundary to the other privacy / security controls

The confidential-AI deployment is not a one-stop control. It complements but does not replace:

> *- **Model memorisation / extraction defences (mod-105).** The TEE protects the prompt in flight and at rest in the enclave. It does not defend against the model emitting memorised training data in response to legitimate prompts. Our deployment additionally applies {canary-based extraction testing, output-side moderation, DP-trained model weights}.*
>
> *- **DP training (mod-106).** Where our model was trained on user data, DP-SGD controls the training-time privacy. The TEE is a serving-time control; DP is a training-time control. They compose as independent axes.*
>
> *- **Federated learning (mod-107).** For deployments where even the training data cannot leave the user's device, FL is the complement. A TEE-based inference deployment can be paired with FL-based training.*
>
> *- **RAG privacy (mod-109).** The prompt entering the enclave is only as confidential as the retrieval layer in front of it. If the retrieval surface leaks the query to a non-confidential vector store, the TEE's protection of the prompt-in-enclave is undermined.*
>
> *- **Unlearning / DSR (mod-110).** The TEE does not relieve us of data-subject-rights obligations. Subject requests for deletion or extraction still apply; the TEE affects *how* the request is processed, not *whether*.*
>
> *- **DPIA (mod-111).** This memo is an input to the DPIA, not a replacement for it. The DPIA covers the full lifecycle; this memo covers the deployment-architecture slice.*
>
> *- **Platform-scale attestation + KMS + fleet key rotation (ai-infra-security, chapter 08).** The sibling role operates the fleet-scale attestation verifier, the KMS at platform scale, and the microcode patching. This memo attests to the privacy-engineering design; the platform-security role attests to the operational discipline.*

### Section 7 (optional) — Decision and sign-off

```
Decision: Deploy {TEE / HE / MPC} based on this memo.

Signed off by:
  Product owner: {name}, {date}
  Privacy engineer (this role): {name}, {date}
  CISO / security: {name}, {date}
  DPO / legal: {name}, {date}
  AI infra security sibling: {name}, {date}
  Finance: {name}, {date}
```

The sign-off list is the forcing function that makes the memo a commitment artefact, not a narrative document.

## The failure modes of a weak memo

In reviewing other teams' memos (and your own earlier drafts), watch for these.

### Failure 1 — Threats stated in abstract

*Bad*: "Protects against cloud attacks."
*Better*: "Defeats a privileged employee of {Azure / GCP / AWS} who has hypervisor-level access from reading the plaintext of user prompts in enclave memory."

The reviewer should know *who* the attacker is and *what* they are kept from doing.

### Failure 2 — Residual trust elided

*Bad*: "The deployment is confidential."
*Better*: "The confidentiality reduces to trust in {AMD} managing the SEV root signing key correctly and in {Azure} correctly implementing the host-side TEE firmware. We review microcode floors on each CVE."

A memo that does not name the residual trust chain is not a trustworthy memo.

### Failure 3 — Cost presented without the alternative

*Bad*: "The confidential SKU costs $X/hour."
*Better*: "The confidential SKU costs $X/hour vs. $Y/hour for the non-confidential baseline; +Z% uplift. Finance has approved the uplift based on the contractual commitment to customer {customer-name}."

The reviewer should see the delta, not the absolute number.

### Failure 4 — "This covers everything"

*Bad*: "This deployment is privacy-compliant end to end."
*Better*: "This deployment covers the transport-and-execution privacy of user prompts to the inference node. Model-output privacy, DPIA-level governance, and subject-rights handling are documented in {other-sections}."

Over-claiming breaks the memo on first scrutiny.

### Failure 5 — Attestation policy implied but not specified

*Bad*: "The enclave is attested."
*Better*: "The enclave attestation claim must include measurement in {known-good set}, TCB version ≥ {date}, policy.debug=0, and a report-data field equal to SHA-256(ephemeral-pk || deployment-nonce). The KMS releases the session key only on an attestation token issued by {MAA endpoint} within the last 5 minutes."

Specificity is the gate between "we did it" and "we can defend it".

### Failure 6 — No GPU-side binding when the deployment has GPUs

*Bad*: The memo talks about SEV-SNP and never mentions the GPU side.
*Better*: The memo pairs CPU and GPU attestation, names the H100 CC binding, and notes that the KMS policy requires both to pass.

GPU-side attestation is the most-skipped step; make sure your memo and your implementation match on it.

## Worked-example micro-memo (fictional)

For orientation, here is a compact-rendering memo for a hypothetical deployment. Numbers are illustrative; your real memo uses measured numbers.

```
MEMO — Confidential AI Deployment Risk and Cost
System: Clinical Triage Assistant (internal)
Date: 2026-11-01
Owner: Privacy Engineering

1. Deployment summary
The Clinical Triage Assistant serves LLM inference to clinicians inside our hospital
network. It runs on Azure NDm H100 v5 with AMD SEV-SNP CPU TEEs and NVIDIA H100 CC
GPU confidential computing. Prompts are end-to-end encrypted from the clinician's
client to the enclave via ECDH against the attested server public key. The
attestation chain terminates at AMD, NVIDIA, and Azure (MAA); our relying-party
policy at Azure Key Vault releases the model-weight decryption key only when the
measurement, TCB version, and GPU CC-mode claims pass.

2. What threat is defeated
(a) Curious cloud operator: Azure employees with hypervisor access cannot read
prompts, model weights, or KV cache.
(b) Curious co-tenant: Spectre-family or hypervisor-escape attacks by co-tenants
return ciphertext at the host-memory boundary; GPU memory is similarly isolated.
(c) Curious privileged insider: Datacentre technicians with physical access to the
host hardware read only encrypted memory.

3. What residual trust remains
(i) AMD's VCEK and SEV root key custody.
(ii) NVIDIA's H100 device-identity CA custody.
(iii) Azure's correct implementation of SEV-SNP host firmware and MAA.
(iv) Our attestation policy's correctness (reviewed quarterly, owner: privacy eng).
(v) Our enclave software's correctness (reviewed per release, owner: AI platform).
(vi) Microcode / firmware patching within 30 days of CVE (owner: ai-infra-security).

4. What this does NOT defeat
- Compromise of AMD/NVIDIA silicon signing roots — baseline trust assumption.
- Zero-day side-channel attacks — residual risk; mitigated by TCB-version floor.
- Model memorisation / extraction — mod-105 scope.
- Bugs in our own enclave software — supply-chain scanning + code review.
- Clinician's own device compromise — out of deployment scope.

5. Cost delta
Compute: ~2.2x the non-confidential baseline per-hour cost on the equivalent H100 SKU.
Throughput overhead: ~5% reduction in tokens/second (vendor-published range).
Latency overhead: +~15 ms per request (steady-state), plus ~200 ms once per session
for the attestation handshake.
Operational overhead: ~0.3 FTE for attestation-policy + TCB-floor maintenance.

6. Boundary
Model-output privacy — mod-105 unlearning review + mod-106 DP training weights.
RAG privacy — mod-109 (retrieval layer is end-to-end encrypted to the enclave).
Subject rights — mod-110 unlearning + mod-111 DPIA.
Platform-scale attestation + KMS fleet — ai-infra-security, chapter 08.

Decision: Proceed with the deployment. Privacy engineering signs off on the
architectural design; dependencies on ai-infra-security are tracked in the handoff
runbook (internal doc ref XXX).
```

## Pitfalls specific to the memo-authoring exercise

### Pitfall 1 — Writing the memo *after* the deployment is live

The memo is a design artefact; it should exist before the first production traffic. Writing it after-the-fact means the reviewer has no leverage to change the design, and the memo reads as justification rather than analysis.

### Pitfall 2 — Writing the memo once and never revising

The memo has to be revised when the architecture changes (new SKU, new attestation verifier, new KMS integration), when a relevant CVE lands (updates the TCB floor and the residual-trust section), and when the business use case shifts (updates the "what threat is defeated" to match the new user population).

### Pitfall 3 — Hiding the HE / MPC alternative that was rejected

If the team considered HE or MPC and chose TEE, the memo should say *why*. "We considered HE; model size and non-linearity cost rule it out at this deployment's latency SLO. We considered MPC; the cross-organisation shape does not apply (single-tenant deployment). TEE is the right choice."

### Pitfall 4 — Hand-waving "cost is acceptable"

Finance will ask. Having the number and the approval chain in the memo prevents a re-litigation three months later.

### Pitfall 5 — Treating the memo as an internal-only artefact

Certain customers and regulators will ask to see the deployment's confidentiality claim. The memo, with minor redactions, is what you give them. Write it in a shape that survives being shared.

## Summary — what you can do after this chapter

- Author a confidential-AI risk-and-cost memo in the six-section template.
- Fill in the "threat defeated", "residual trust", "does NOT defeat", "cost delta", and "boundary" sections with specific, defensible content for your deployment.
- Avoid the six common memo failures (abstract threats, elided residual trust, cost-without-alternative, overclaiming, under-specified attestation policy, missing GPU-side binding).
- Pair the memo with the attestation policy (chapter 02), the hyperscaler shared-responsibility boundary (chapter 03), the PCC rubric (chapter 04), and the HE / MPC alternatives (chapters 05-06) to defend each decision.
- Iterate the memo at the right lifecycle moments (design, CVE, architecture change, use-case shift).
- Use the memo as the conversation artefact with product, finance, legal, and the platform security sibling role.

Chapter 08 is the one-page boundary to `ai-infra-security` — the sibling role that owns the platform-scale attestation operations this module has treated as background. If this module is the privacy-engineering design slice, chapter 08 is where it hands off to the operations slice.
