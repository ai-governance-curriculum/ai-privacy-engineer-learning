# The Trust Model for a Confidential-AI Deployment — What TEEs Defend Against and What They Do Not

Confidential computing, in one sentence, is "run the computation inside a hardware-enforced enclave whose state is cryptographically opaque to everyone outside it, including the machine's operator". It is the control privacy teams reach for when the bare cloud-trust story — "we trust Amazon / Google / Microsoft to not peek at our GPU RAM" — is not good enough for the customer, the regulator, or the risk register. The AI deployment that lives inside a TEE promises to its users: *even if the cloud operator wanted to read your prompt, they could not; even if a co-tenant escaped their VM, they would hit opaque memory; even if a privileged insider at the cloud ran a memory dump, they would see ciphertext.*

That promise is real. It is also narrow. The first thing a privacy engineer owes a design review is a correctly-scoped statement of what the TEE *does* and does *not* defend against. A confidential-AI deployment sold as a universal privacy wrapper, with no mention of side-channel risk, no mention of firmware trust, no mention of the remote-attestation chain's root CA, is a release package that will not survive a serious review.

This chapter builds the trust model. Chapters 02–04 build the deployment and the enterprise blueprint on top of it. Chapters 05–06 cover the cryptographic alternatives (HE and MPC) for the cases where the TEE's residual trust assumption is still too generous.

Primary references — read them in this order:

- Confidential Computing Consortium — *A Technical Analysis of Confidential Computing*, v1.3, 2022. [confidentialcomputing.io/wp-content/uploads/sites/10/2023/03/CCC-A-Technical-Analysis-of-Confidential-Computing-v1.3_Updated_November_2022.pdf](https://confidentialcomputing.io/wp-content/uploads/sites/10/2023/03/CCC-A-Technical-Analysis-of-Confidential-Computing-v1.3_Updated_November_2022.pdf) — the vendor-neutral definition. The CCC's working definition of "data-in-use protection" and the taxonomy of what is and is not in scope is the common reference auditors cite.
- Confidential Computing Consortium — *Common Terminology for Confidential Computing*, 2022. [confidentialcomputing.io/wp-content/uploads/sites/10/2023/03/Common-Terminology-for-Confidential-Computing.pdf](https://confidentialcomputing.io/wp-content/uploads/sites/10/2023/03/Common-Terminology-for-Confidential-Computing.pdf) — the glossary. Use it so your DPIA and the platform team's security review use the same words.
- AMD — *AMD SEV-SNP: Strengthening VM Isolation with Integrity Protection and More*, 2020. [amd.com/content/dam/amd/en/documents/epyc-business-docs/white-papers/SEV-SNP-strengthening-vm-isolation-with-integrity-protection-and-more.pdf](https://www.amd.com/content/dam/amd/en/documents/epyc-business-docs/white-papers/SEV-SNP-strengthening-vm-isolation-with-integrity-protection-and-more.pdf) — the primary design document for SEV-SNP. Read sections 1–3 for the threat model.
- Intel — *Intel® Trust Domain Extensions (Intel® TDX) White Paper*, 2023. [intel.com/content/www/us/en/developer/articles/technical/intel-trust-domain-extensions.html](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-trust-domain-extensions.html) — Intel's VM-level TEE; TDX is the SEV-SNP peer on Intel silicon and the successor framing to SGX for full-VM confidential computing.
- NVIDIA — *NVIDIA Confidential Computing: Solution Brief*, 2023. [images.nvidia.com/aem-dam/Solutions/Data-Center/HGX/nvidia-confidential-computing-solution-brief.pdf](https://images.nvidia.com/aem-dam/Solutions/Data-Center/HGX/nvidia-confidential-computing-solution-brief.pdf) and *H100 Product Brief*, [nvidia.com/content/dam/en-zz/Solutions/gtcs22/data-center/h100/PB-11133-001_v01.pdf](https://www.nvidia.com/content/dam/en-zz/Solutions/gtcs22/data-center/h100/PB-11133-001_v01.pdf) — NVIDIA's documentation of the H100's confidential computing mode.
- Costan & Devadas — *Intel SGX Explained*, IACR ePrint 2016/086. [eprint.iacr.org/2016/086](https://eprint.iacr.org/2016/086) — the standing academic reference for SGX internals; included here because Intel SGX is the historical anchor against which newer TEEs are described, even though SGX itself was deprecated for mainstream server use in 11th-/12th-gen Xeon (the server ecosystem has moved to TDX).

## What a TEE is, in one paragraph

A Trusted Execution Environment is a hardware-enforced region whose memory is encrypted by a key that only the CPU (or GPU) hardware holds; whose code and initial state can be *measured* and *attested* to a remote verifier; and whose state is protected against the privileged software stack on the same host (the hypervisor, the host OS, and in the newer schemes, SMM / microcode). The three defining properties are **confidentiality** (external software cannot read enclave memory), **integrity** (external software cannot modify enclave memory undetected), and **attestation** (a remote party can cryptographically verify *what* is running in the enclave before trusting it with data). Any technology that does not deliver all three is not a TEE under the CCC definition.

The TEE comes in two architectural shapes relevant to AI deployments:

- **Process-level TEE.** Intel SGX is the archetype. A specific *process* is placed in an enclave; the OS and hypervisor cannot read its memory. Deprecated for mainstream server use (removed from client platforms in 11th-gen Core and from most server platforms in Xeon Scalable "Ice Lake" onward) but still cited extensively in the literature. You will encounter SGX in DCAP-based fleet attestation documentation and in older confidential-AI products (Microsoft's Azure Confidential Computing SGX nodes, Edgeless Systems' first generation).
- **VM-level TEE.** AMD SEV-SNP and Intel TDX are the archetypes. An entire *virtual machine* is placed in an enclave; the host kernel and hypervisor cannot read the guest's memory. This is the dominant server-TEE shape for AI deployments in 2024–2026, because lifting-and-shifting a GPU-inference container into a confidential VM is operationally lighter than refactoring the inference binary to fit an SGX enclave. All three hyperscalers (Azure, GCP, AWS Nitro Enclaves is a closely-related cousin) ship on this model.
- **Accelerator TEE.** NVIDIA H100 (and successor) Confidential Computing extends the TEE boundary to the GPU. The CPU VM is confidential (SEV-SNP or TDX); the GPU driver and the GPU hardware cooperate to encrypt the data in transit over PCIe and to isolate the GPU memory from the host. The combination is what makes *confidential LLM inference* on datacentre GPUs possible at all; without accelerator-side confidential computing, a confidential VM that offloads work to a non-confidential GPU leaks the prompt at the PCIe boundary.

This module treats VM-level + accelerator-side as the deployment model because it is what hyperscalers ship, what Apple's PCC uses, and what the production confidential-AI reference architectures look like. We will name SGX where it still matters (older attestation infrastructure, some edge appliances) but the engineering focus is SEV-SNP + TDX + H100.

## What the TEE threat model defends against

The CCC's technical analysis names three threats a TEE is designed to defeat. Read them with examples in mind so the DPIA language is concrete, not platonic.

### 1. The curious cloud operator

Scenario: your model and your users' prompts run on EC2 / GCE / Azure VM. The cloud operator is a vast organisation of employees, contractors, support engineers, and automated agents. Some of them have the technical ability — via a hypervisor-level debug tool, a memory-snapshot API, a live-migration probe, a SmartNIC firmware hook — to read the guest's memory. The TEE's confidentiality guarantee says: those operator-side privileged paths read *ciphertext only*; the key is held by the CPU hardware and is not exported to any host-software surface the operator's employees can reach. AMD SEV-SNP's SEV-ES feature encrypts register state during VMEXITs so even a hypervisor that owns the VCPUs cannot snoop register values without the SNP firmware's consent; SEV-SNP's "Reverse Map Table" (RMP) prevents the hypervisor from remapping guest pages to steal data via aliasing.

What this blocks concretely: a cloud-provider employee who gains access to the hypervisor cannot grep your model's KV cache for prompts, cannot dump the VRAM of your GPU, cannot read the plaintext of your KMS-wrapped weights while they are decrypted inside the enclave.

What this does not block: operator access to anything *outside* the enclave. If your enclave writes a log file to an unencrypted volume, the operator reads the log. If your enclave accepts a plaintext API request on an untrusted socket, the operator reads the request in flight. Every I/O boundary of the enclave is a trust decision you have to design.

### 2. The curious co-tenant

Scenario: you share the host machine with another customer's VM (or a container, or a function). A bug in the hypervisor, a Spectre-family side channel, a timing attack on the shared L3 cache, or a straight-up VM-escape vulnerability lets your co-tenant read your memory. Historically, every hypervisor has had at least one such bug in its lifetime; confidential-VM designs put a hardware-enforced memory-encryption boundary around each tenant so that even if the hypervisor is compromised by a co-tenant, the *data* the co-tenant can read from the other tenant's pages is ciphertext.

What this blocks concretely: a co-tenant who exploits a hypervisor bug to read RAM pages belonging to your VM gets random-looking ciphertext. The attacker would additionally need to break the AES key management inside the CPU — a much higher bar than the hypervisor compromise alone.

What this does not block: *side-channel* attacks that infer secrets from timing, cache behaviour, power draw, or electromagnetic emanations. The CCC document is explicit that side-channel resistance is "out of scope for the baseline definition"; TEE vendors mitigate known classes (and new mitigations ship in microcode / firmware updates) but do not claim all-channel resistance. Covered in detail below.

### 3. The curious privileged insider

Scenario: a datacentre technician with physical access, a BIOS engineer with the ability to push a firmware update, a vendor support engineer with a debug hook into the server's BMC. The TEE's design assumption is that *most* of the physical and administrative access paths that would grant plaintext memory access on a conventional server are blocked by the hardware's memory encryption + the attestation chain.

What this blocks concretely: a technician who installs a cold-boot RAM-freeze device onto an active confidential VM's DIMMs reads ciphertext. A BMC-level debug shell that would normally allow DMA reads of guest memory reads ciphertext. A rogue SMM handler that reads physical memory reads ciphertext.

What this does not block: insiders who compromise the *root of trust* itself. If an insider can sign a malicious firmware with the vendor's CPU-key (Intel's PCK/Platform Certification Key for TDX attestation, AMD's VCEK/Versioned Chip Endorsement Key for SEV-SNP, NVIDIA's device identity certificates for H100 CC), they can fake an attestation and convince your relying party that a tampered VM is a pristine one. The root-of-trust compromise is explicitly out-of-scope; it is the security boundary the attestation chain *terminates* at. Your trust in the TEE is, in the end, trust in AMD's / Intel's / NVIDIA's key-management discipline for those root keys. We will come back to this when we build the attestation chain in chapter 02.

## What the TEE threat model does **not** defend against

Half of the privacy-engineer job in a confidential-AI design review is articulating these exclusions so no one sells the deployment past them.

### Exclusion 1 — Side-channel attacks

A side channel is a leakage path that bypasses the explicit confidentiality boundary by inferring secrets from something the hardware *lets* an attacker observe: timing differences in cache access, power draw patterns, electromagnetic emanations, speculative-execution residue. The published history of TEE-relevant side channels is long and continues:

- **Spectre v1 / v2 / v4** (Kocher et al. 2019, [spectreattack.com](https://spectreattack.com/spectre.pdf)) — speculative-execution leakage of enclave secrets; mitigated in microcode and compiler flags, but new variants continue to appear.
- **Meltdown** (Lipp et al. 2018, [meltdownattack.com](https://meltdownattack.com/meltdown.pdf)) — similar, fixed in hardware for post-2019 CPUs.
- **Foreshadow / L1TF** (Bulck et al. 2018, [foreshadowattack.eu/foreshadow.pdf](https://foreshadowattack.eu/foreshadow.pdf)) — extracted SGX attestation keys from a vulnerable Skylake generation; mitigated in microcode.
- **MDS / RIDL / Fallout / ZombieLoad** (2019) — microarchitectural data sampling; mitigated by VERW-fence microcode.
- **CacheOut / L1DES / TAA** (2020) — further sampling variants.
- **SGAxe** (Bulck et al. 2020, [cacheoutattack.com/files/SGAxe.pdf](https://cacheoutattack.com/files/SGAxe.pdf)) — leveraged CacheOut to extract SGX sealing keys.
- **ÆPIC Leak** (Borrello et al. 2022, [aepicleak.com/aepicleak.pdf](https://aepicleak.com/aepicleak.pdf)) — uninitialised APIC-register leakage affecting Intel SGX.
- **Hertzbleed** (Wang et al. 2022, [hertzbleed.com](https://www.hertzbleed.com/hertzbleed.pdf)) — power-frequency-scaling as a timing side channel affecting constant-time cryptographic implementations.

Two facts matter for the design review.

First, **side channels are a real, recurring, class-of-finding** against TEEs. A deployment whose privacy claim depends on "nothing leaks out of the enclave ever" is over-claiming; the regulator's technical consultant knows these papers exist.

Second, **the mitigation path is real and institutional**: TEE vendors ship microcode / firmware updates for each disclosed variant; attestation policies let a verifier demand a minimum TCB (Trusted Computing Base) version, so a relying party can refuse to use a VM running on a CPU that has not patched a known attack. The engineering requirement that follows is that your attestation *policy* has a TCB-version floor and is updated promptly when a CVE is disclosed. Chapter 02 develops the attestation-policy syntax; chapter 08 names the fleet-patching discipline that `ai-infra-security` owns.

Your DPIA-friendly sentence: *"The TEE defeats direct memory inspection by the operator, the co-tenant, and the privileged insider. Side-channel attacks are an ongoing class of vulnerability which the vendor patches in microcode; our attestation policy enforces a minimum patched TCB version; residual side-channel risk is the main reason extremely sensitive workloads additionally use HE or MPC (chapters 05-06)."*

### Exclusion 2 — Compromised firmware

The attestation chain begins at a root certificate held by the silicon vendor (AMD's SEV / AMD Secure Processor root, Intel's PCK, NVIDIA's device identity CA for H100 CC). Everything above the root is measured and signed; the root itself is trusted by assertion. If the vendor's signing key is leaked, or an insider at the vendor signs malicious firmware, the attestation chain produces valid-looking signatures on tampered VMs. This is "trusted authority compromise" and the CCC document is explicit that it is out of scope.

In practice, vendor key-leakage would be catastrophic for the entire confidential-computing ecosystem simultaneously, and the vendors invest accordingly in root-key custody. But two realistic sub-concerns do fall on *you* as the deployer:

- **Firmware-update race.** A disclosed CPU vulnerability has a window between patch availability and fleet rollout. During that window, an attacker who has a working exploit can mount the attack against unpatched hosts. Your attestation-policy TCB floor is the lever that closes the window at the relying-party side.
- **Non-vendor firmware.** The BMC, the NIC firmware, the SmartNIC SoC, the GPU firmware — each has its own supply chain. On an AWS Nitro Enclave the Nitro Card is a dedicated hardware root; on an Azure Confidential VM the HGS (Host Guardian Service) attests the host's boot state. The attestation chain in your deployment *includes* all of these; a compromised non-CPU firmware can defeat the TEE even if the CPU signing root is intact.

Engineering requirement: enumerate the firmware entities in your attestation chain and record them in the release package. Chapter 02 names the AMD / NVIDIA fields; chapter 03 names the hyperscaler-specific ones.

### Exclusion 3 — Rollback and replay

A TEE protects data *while it is in the enclave*. If you seal data for persistence (the enclave encrypts with a key sealed to its identity, and writes to disk), an attacker with disk access can later deliver a stale version of the sealed blob back to the enclave — and the enclave, assuming the attacker is honest about freshness, might decrypt an older state. This is the "rollback attack" and it is a well-known class of TEE weakness. SEV-SNP's "monotonic counters" and TDX's "service TD" address it partially; applications that depend on state freshness (rate limits, nonces, an accountant's state like the DP accountant in mod-106) must build their own freshness protocol on top of the TEE.

Engineering requirement: if your confidential-AI node persists *any* state across restarts (session keys, usage counters, moderation decision caches), name the freshness mechanism explicitly.

### Exclusion 4 — Compromise of the enclave software itself

The TEE protects the enclave's memory from the outside; it does not protect the enclave from bugs in its own code. If the inference server running inside the enclave has a buffer overflow, an injection vulnerability, a compromised library dependency, or a logic bug that writes the prompt to a log, the TEE's confidentiality guarantee is irrelevant because the *application itself* exfiltrated the data.

This is where the confidential-AI deployment and the ordinary software-security discipline meet. The TEE is not a substitute for the app-security discipline on the inference server — it is a complement. Supply-chain scanning, memory-safe languages where possible, least-privilege IAM on anything the enclave can call out to, and rigorous review of what the enclave logs and where it logs it are all still required. The TEE gives you a smaller, better-defined thing to defend; it does not defend it for you.

### Exclusion 5 — Compromise of the relying party or its policy

The attestation evidence is only as useful as what the *relying party* does with it. If your KMS releases the session key whenever it sees a signed attestation blob — without checking that the measurement matches an approved image, or that the TCB version is above the current floor, or that the enclave's platform-certificate chain terminates at the expected root — the attestation is theatre. The policy on the relying party is a first-class part of the trust model; chapter 02 develops the policy syntax, and chapter 08 covers the platform-scale version.

### Exclusion 6 — "The model"

The TEE protects the prompt in flight and the activations in RAM. It does *not* make the model itself non-leaky. A perfectly-attested enclave running a model that has memorised training-data verbatim (mod-105) will happily emit memorised secrets to a legitimate user's prompt; the TEE is a transport-and-execution privacy control, not a model-output privacy control. This is the standing boundary — the TEE module and the memorisation module (105) and the DP training module (106) are complements, not substitutes.

## Positioning Intel SGX, AMD SEV-SNP, Intel TDX, and NVIDIA H100 CC

The four technologies are not interchangeable; they occupy different points in the TEE design space and each solves a different subset of the confidential-AI problem.

### Intel SGX

- **Architecture:** Process-level enclave; a specific application is launched into an Enclave Page Cache (EPC) region whose memory is encrypted by the CPU's Memory Encryption Engine (MEE).
- **Threat model:** Protects the enclave from the OS and hypervisor. The attestation is per-enclave, anchored in the Intel-signed Quoting Enclave and, in the DCAP (Data Center Attestation Primitives) flow, in a per-platform Provisioning Certification Key.
- **Where it is in 2024–2026:** Deprecated for mainstream client and server use. Removed from 11th-gen Core and most mainstream Xeon Scalable processors; still available on some Xeon E-class server SKUs and some cloud SGX offerings (Azure DCsv2 / DCsv3 generations). Intel's server-TEE roadmap is TDX.
- **AI relevance:** The historical confidential-AI deployments on Azure (DCsv3-series "Confidential Computing") used SGX enclaves; the development model (Open Enclave SDK, Edgeless Systems' EGo, Fortanix EDP) is well-documented but requires an SGX-aware application. Not a greenfield choice for a 2026 confidential-LLM deployment.

### AMD SEV-SNP (Secure Encrypted Virtualization — Secure Nested Paging)

- **Architecture:** VM-level TEE; an entire guest VM's memory is encrypted with a per-VM AES key held by the AMD Secure Processor (SP, a separate ARM core on the CPU die). SEV-SNP adds *integrity* and anti-replay protection on top of the older SEV and SEV-ES schemes, via the Reverse Map Table that prevents the hypervisor from remapping, aliasing, or replaying guest pages.
- **Threat model:** Protects the guest from the host kernel, the hypervisor, SMM, and the cloud operator. Attestation is provided via the "attestation report" generated by the Secure Processor; the report is signed with a per-chip VCEK (Versioned Chip Endorsement Key) that chains to AMD's SEV root CA.
- **Where it is in 2024–2026:** Shipping in AMD EPYC "Milan" (Zen 3), "Genoa" (Zen 4), and "Turin" (Zen 5) server platforms. The dominant server-TEE for confidential-LLM deployments as of this writing, because the hyperscalers (Azure, GCP, AWS Nitro counterpart) all offer SEV-SNP-backed confidential VM SKUs.
- **AI relevance:** The CPU-side of the confidential-LLM deployment. Pair with an H100-CC GPU in the same VM and the full inference path — prompt arrives at the confidential VM's network stack, is decrypted inside the enclave, is transferred encrypted to the GPU, is processed by the GPU in confidential mode, result comes back — is covered.

### Intel TDX (Trust Domain Extensions)

- **Architecture:** VM-level TEE; Intel's peer to SEV-SNP. A "Trust Domain" (TD) is an isolated VM whose memory is encrypted with per-TD keys (via Multi-Key Total Memory Encryption, MKTME); the TDX module (an Intel-signed component, loaded by the host but hardware-isolated) mediates the boundary. The attestation report is signed with a key chained to Intel's PCK/PCE infrastructure, the same root as DCAP-based SGX attestation.
- **Threat model:** Equivalent in scope to SEV-SNP — protects the TD from the host kernel, VMM, SMM, and operator.
- **Where it is in 2024–2026:** Shipping in Intel Xeon "Sapphire Rapids" (4th-gen) and later. Azure's DCe/ECe-series Confidential VMs and GCP's C3 Confidential VM instances offer TDX-backed VMs alongside SEV-SNP.
- **AI relevance:** The Intel-silicon alternative to SEV-SNP for the CPU-side of a confidential-LLM deployment. Pair with H100 CC on the same VM for the full path.

### NVIDIA H100 Confidential Computing (and successors: H200, B100 / Blackwell)

- **Architecture:** Accelerator TEE. In "confidential-computing mode" the H100 (a) isolates GPU memory from the host and the DMA engines so the GPU's HBM contents are inaccessible to anything outside the GPU, (b) encrypts PCIe traffic between the CPU VM and the GPU using keys negotiated at session establishment, and (c) provides an attestation flow where the GPU reports its identity and firmware state to a verifier. The protocol relies on the SPDM (Security Protocol and Data Model) standard for device authentication.
- **Threat model:** Protects the model weights, the KV cache, and the activations resident in GPU memory from the host OS, the hypervisor, and co-tenants on the same machine. The CPU-side of the path is only as confidential as the CPU TEE — H100 CC does not do anything about CPU-side memory; it must be paired with SEV-SNP or TDX for an end-to-end confidential inference.
- **Where it is in 2024–2026:** H100 CC is generally available on Azure's NCC H100 v5, NDm H100 v5 confidential VM SKUs and in Google Cloud's A3 confidential VMs in preview / selected regions. H200 and Blackwell generation inherit the mode. <!-- needs-research: confirm exact GA status of H100 CC on AWS, which has historically been a Nitro-Enclave shop and lagged on host-TEE-plus-accelerator-CC shapes. -->
- **AI relevance:** The piece that makes "confidential LLM inference" possible for frontier-scale models that only run on datacentre GPUs. Without H100 CC, your confidential VM would decrypt the prompt in the CPU enclave and then copy it, in cleartext over PCIe, to a non-confidential GPU — the GPU memory would be snoopable by any tenant on the same host or by the operator's debug tooling. H100 CC closes that gap.

## The three archetypes, in one table

| Threat | Intel SGX | AMD SEV-SNP | Intel TDX | NVIDIA H100 CC |
| --- | --- | --- | --- | --- |
| Curious cloud operator reads CPU RAM | Blocked (per enclave) | Blocked (whole VM) | Blocked (whole VM) | n/a (CPU-side) |
| Curious cloud operator reads GPU RAM | n/a | Not covered by CPU TEE | Not covered by CPU TEE | Blocked |
| Co-tenant via hypervisor compromise | Blocked (ciphertext only) | Blocked (ciphertext + integrity) | Blocked (ciphertext + integrity) | Blocked (GPU isolation) |
| Privileged insider cold-boot / BMC DMA | Blocked for RAM content | Blocked for RAM content | Blocked for RAM content | Blocked for GPU memory |
| Vendor root-key compromise | Out of scope | Out of scope | Out of scope | Out of scope |
| Known side-channel (Spectre, L1TF, ÆPIC, etc.) | Partial — mitigated per-variant in microcode | Partial — same model, VM-level | Partial — same model, VM-level | Partial — GPU microarchitectural research is ongoing |
| Firmware rollback / replay | Partial — monotonic counters per-enclave | Partial — guest-counter mechanism | Partial — service TD mechanism | Device-attestation nonce per session |
| Bugs in the enclave's own software | Out of scope | Out of scope | Out of scope | Out of scope |

Use this table as the opening slide of the confidential-AI design review. "Here is what we defend against; here is what we do not; here is the mitigation path for the ones we do not." Anything added to the deployment in the name of "defence in depth" that does not specifically address a row of this table is noise; anything that addresses a row is an upgrade worth defending.

## The residual trust statement

The trust-model deliverable at the end of a confidential-AI design is a one-paragraph *residual trust statement* that your DPIA will quote verbatim. Its shape:

> *The confidential-{application} deployment runs on {CPU TEE: AMD SEV-SNP / Intel TDX} on {vendor: Azure / GCP / AWS} hardware, paired with {GPU TEE: NVIDIA H100 CC} for accelerator workloads. The TEE defeats direct memory inspection by the cloud operator, by a co-tenant, and by a privileged insider at the datacentre. The TEE does not defend against (a) side-channel attacks on the CPU or GPU microarchitecture, mitigated by enforcing an attestation TCB-version floor of {date}; (b) compromise of the silicon vendor's attestation root signing key, which is a baseline trust assumption inherited from {AMD / Intel / NVIDIA}; (c) compromise of the enclave's own application code, mitigated by {supply-chain and application-security controls}; (d) leakage through the model's outputs themselves, mitigated by {the memorisation / DP controls in modules 105 / 106}. The residual trust assumption is that {AMD / Intel / NVIDIA} correctly manages their attestation root keys, that microcode patches are applied within {N} days of CVE disclosure, and that our relying-party attestation policy is reviewed on each TCB update.*

Every term in that paragraph should resolve to a concrete artefact in the release package. Chapter 07 is where the memo wraps it up; the rest of the module is the engineering that makes each term defensible.

## Summary — what you can do after this chapter

- State the three threats a TEE defeats (curious operator, co-tenant, privileged insider) and the five it does not (side channels, vendor-root compromise, rollback / replay, in-enclave bugs, policy weakness at the relying party, model-output leakage).
- Name Intel SGX, AMD SEV-SNP, Intel TDX, and NVIDIA H100 CC, know which are process-level vs. VM-level vs. accelerator, and position them against each threat in the table.
- Cite the CCC's technical analysis as the auditor-recognised baseline definition.
- Cite at least four specific side-channel results (Spectre, Foreshadow, SGAxe, ÆPIC, Hertzbleed, MDS family) when a reviewer asks "but what about side channels?".
- Articulate the residual trust statement in the shape the DPIA will consume.
- Name the attestation-TCB-version floor as the engineering lever that closes the side-channel-mitigation window.

Chapter 02 builds on this foundation by taking the attestation protocol apart — SEV-SNP guest attestation, NVIDIA H100 CC attestation, image measurement verification, the encrypted prompt-and-response path, and the KMS policy that only releases session keys when the full chain checks.
