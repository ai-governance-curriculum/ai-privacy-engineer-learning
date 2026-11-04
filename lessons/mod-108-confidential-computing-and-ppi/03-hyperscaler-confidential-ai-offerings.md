# Hyperscaler Confidential-AI Offerings — Azure, GCP, AWS, and the Shared-Responsibility Boundary

Confidential computing as a platform offering did not land in all three major clouds at the same time, in the same shape, or with the same deliberate framing. Azure went first and has the longest-standing "Confidential AI" product narrative; Google Cloud converged from its Confidential VMs line into Confidential Space (a managed TEE for multi-party workflows) and then into GPU-enabled confidential VMs; AWS took a different path with Nitro Enclaves, a CPU-TEE-shaped offering that predates its rivals and interacts differently with GPUs. Picking the right offering for a confidential-AI deployment is not a feature-parity exercise — the differences are load-bearing for the privacy claim.

This chapter positions the three hyperscaler offerings against the trust model (chapter 01) and the attestation plumbing (chapter 02), and names the shared-responsibility boundary each defines — the line between what the cloud operator attests to and what you as the AI deployer still own.

Primary references:

- Microsoft — *Azure Confidential Computing* overview. [learn.microsoft.com/en-us/azure/confidential-computing/overview](https://learn.microsoft.com/en-us/azure/confidential-computing/overview) — product catalogue and threat-model doc.
- Microsoft — *Azure Confidential Computing: SKU catalogue* (DCasv5 / DCesv5 series for SEV-SNP and TDX-based VMs; NCC H100 v5 / NDm H100 v5 for GPU-enabled confidential VMs). [learn.microsoft.com/en-us/azure/virtual-machines/dcasv5-dcadsv5-series](https://learn.microsoft.com/en-us/azure/virtual-machines/dcasv5-dcadsv5-series) and [learn.microsoft.com/en-us/azure/virtual-machines/ncc-h100-v5](https://learn.microsoft.com/en-us/azure/virtual-machines/ncc-h100-v5) — the sizes you will actually provision.
- Microsoft — *Azure Attestation Service* product docs. [learn.microsoft.com/en-us/azure/attestation/overview](https://learn.microsoft.com/en-us/azure/attestation/overview).
- Microsoft — *Azure Confidential Clean Rooms*. [learn.microsoft.com/en-us/azure/confidential-computing/confidential-clean-rooms](https://learn.microsoft.com/en-us/azure/confidential-computing/confidential-clean-rooms) — the multi-party confidential-collaboration product.
- Google Cloud — *Confidential VMs overview*. [cloud.google.com/confidential-computing/confidential-vm/docs/overview](https://cloud.google.com/confidential-computing/confidential-vm/docs/overview) — the n2d / c2d / c3 / a3 confidential series.
- Google Cloud — *Confidential Space overview*. [cloud.google.com/confidential-computing/confidential-space/docs/confidential-space-overview](https://cloud.google.com/confidential-computing/confidential-space/docs/confidential-space-overview) — the managed TEE for multi-party collaborations with attested workload identity.
- Google Cloud — *Confidential GKE Nodes*. [cloud.google.com/kubernetes-engine/docs/how-to/confidential-gke-nodes](https://cloud.google.com/kubernetes-engine/docs/how-to/confidential-gke-nodes) — Kubernetes nodes backed by confidential VMs.
- AWS — *AWS Nitro Enclaves User Guide*. [docs.aws.amazon.com/enclaves/latest/user/nitro-enclave.html](https://docs.aws.amazon.com/enclaves/latest/user/nitro-enclave.html) and the companion *Nitro System* whitepaper, [aws.amazon.com/ec2/nitro/](https://aws.amazon.com/ec2/nitro/).
- AWS — *The Security Design of the AWS Nitro System*. [docs.aws.amazon.com/whitepapers/latest/security-design-of-aws-nitro-system/security-design-of-aws-nitro-system.html](https://docs.aws.amazon.com/whitepapers/latest/security-design-of-aws-nitro-system/security-design-of-aws-nitro-system.html) — the primary design document.
- Cloud Security Alliance — *Confidential Computing Mini-Surveys 2023 / 2024*. [cloudsecurityalliance.org/](https://cloudsecurityalliance.org/) — the cross-vendor comparison framework, useful when you need a vendor-neutral artefact for a procurement review. <!-- needs-research: verify the latest CSA confidential computing publication year and exact title. -->

## The three offerings at a glance

| Dimension | Azure Confidential Computing | GCP Confidential Computing | AWS Nitro Enclaves |
| --- | --- | --- | --- |
| CPU TEE technology | SEV-SNP (DCasv5 / ECasv5), TDX (DCesv5 / ECesv5), SGX (DCsv3 — legacy) | SEV-ES (n2d, c2d), SEV-SNP (c3d), TDX (c3) | Nitro Hypervisor + Nitro Enclaves (CPU-TEE equivalent using AWS's own silicon root) |
| GPU TEE technology | NVIDIA H100 CC in NCC H100 v5 / NDm H100 v5 | NVIDIA H100 CC in A3 Confidential VMs | <!-- needs-research: AWS announced "Nitro Enclaves with GPU" / H100 CC roadmap at different events; confirm GA status before claiming availability. --> |
| Attestation verifier | Microsoft Azure Attestation (hosted) + MAA client SDK | Google Cloud Attestation (built into Confidential Space) + open-source `go-tpm-tools` | Nitro Enclaves attestation document (CBOR, COSE-signed by AWS); verified by KMS condition or custom client |
| Native KMS integration | Azure Key Vault Managed HSM with secure key release (SKR) policy | Google Cloud KMS; confidential-space workload identity federates to IAM | AWS KMS with `kms:RecipientAttestation:*` condition keys |
| Confidential orchestration layer | Azure Confidential Containers (ACI, AKS), Confidential Clean Rooms | Confidential GKE Nodes, Confidential Space | Nitro Enclaves (parent-EC2 + enclave model); no native K8s shape |
| Multi-party workflow product | Azure Confidential Clean Rooms | Confidential Space | None native (pattern must be built on top of KMS + enclave) |
| Primary vendor whitepaper | Azure Confidential Computing overview (linked above) | Confidential VM overview + Confidential Space overview | *Security Design of the AWS Nitro System* |
| Threat-model distinctive | "Azure is a service; the TCB excludes Azure itself" | "Workload identity, not operator trust" | "The Nitro Card is a dedicated hardware root; AWS doesn't need to be trusted for the enclave boundary" |
| CPU-TEE deprecations | SGX DCsv2/DCsv3 for lift-and-shift SGX; new workloads should use SEV-SNP or TDX | n2d SEV / SEV-ES being succeeded by c3d SEV-SNP; TDX in c3 | n/a — Nitro is AWS's own stack from the start |

Each offering has a distinctive framing of the shared-responsibility boundary. The cloud operator makes certain claims; the deployer picks up the responsibility at a specific point. The claim the operator *does not* make is where the privacy engineer has to design their own control.

## Azure Confidential Computing

### What Azure attests to

Azure Confidential Computing's threat model is articulated as *"reduce the Trusted Computing Base by removing Azure itself from the trust boundary"*. The product surface:

- **Confidential VMs (SEV-SNP and TDX-backed)** — DCasv5, DCesv5, ECasv5, ECesv5 series on AMD EPYC Milan / Genoa and Intel Sapphire Rapids. The VM boots with SEV-SNP or TDX enabled; the launch measurement is computed by the AMD SP or the Intel TDX module; the VM can request its own attestation via `/dev/sev-guest` or TDX's attestation ABI.
- **Confidential containers on ACI and AKS** — ACI confidential container groups run a container inside a confidential VM; AKS confidential nodes do the same at the node level. The attestation chain extends to the container image if the deployer configures it.
- **Azure Attestation Service (MAA)** — Azure's managed verifier. Accepts SEV-SNP reports, TDX reports, SGX quotes, and VBS evidence; returns a JWT EAT with Azure-defined claims (`x-ms-isolation-tee.x-ms-sevsnpvm-launchmeasurement`, `x-ms-sevsnpvm-tcb-version`, `x-ms-compliance-status`, etc.).
- **Azure Key Vault Managed HSM with Secure Key Release** — a key-vault policy consumes the MAA EAT and releases wrapped key material only on a match.
- **Azure Confidential Clean Rooms** — a product shape for multi-party analytics where two or more parties load data into an attested workload; neither party sees the other's data, only the sanctioned outputs.
- **NCC H100 v5 / NDm H100 v5** — confidential VMs with NVIDIA H100 GPUs in CC mode. Supports the full CPU+GPU confidential-inference shape chapter 02 developed.

### What Azure does not attest to

- **The code *you* ship inside the confidential VM.** The launch measurement covers the firmware and the boot blob; it does not cover your inference server's container image unless you have built an image-measurement chain that extends to it. You own the rest of the measurement chain — the IPA (initial point of attestation), the dm-verity root hash of the root filesystem, the signing of your container registry.
- **The policy on MAA and on Key Vault.** The default MAA configuration will emit a signed EAT for any valid attestation report; the *policy* that decides "only release my key if the measurement is X" lives on the key-vault or on your own relying-party service. You own it.
- **The trust in NVIDIA's device identity CA.** Azure attests the SEV-SNP side; NVIDIA attests the GPU side. Both root certificates are baseline trust assumptions; neither is Azure's responsibility.

### Shared-responsibility boundary, in one paragraph

> *Azure guarantees that the SEV-SNP or TDX hardware functions as specified, that its attestation verifier produces trustworthy claims from valid evidence, and that the Secure Key Release mechanism enforces the policy you configure. Azure makes no guarantee about the application code running inside the confidential VM, about your attestation policy's completeness, or about operational discipline on the key-vault policy (who can edit it, how often it is reviewed). The deployer is accountable for the image-measurement allow-list, the Key Vault SKR policy, the operational rotation of both, and all application-level confidentiality controls.*

### Practical gotchas on Azure

- **MAA endpoint selection.** MAA has region-specific endpoints (`sharedneu.attest.azure.net`, `sharedeus.attest.azure.net`, etc.). Pin the endpoint in your policy; do not accept EATs signed by an arbitrary MAA region. A deployer who accepts any MAA EAT loses the regional residency property.
- **Trusted Launch vs. Confidential VM.** Azure also sells "Trusted Launch" (Secure Boot + vTPM + no memory encryption) under the general umbrella of "security". Trusted Launch is *not* confidential computing — it does not encrypt memory. Design reviewers should check the SKU, not the marketing label.
- **CPU and GPU attestation pinning.** On NCC H100 v5, the attestation chain for the GPU is NVIDIA's; the CPU side is Azure's MAA. Your KMS policy must check both.
- **SGX legacy.** DCsv2/DCsv3 SGX SKUs remain available for existing workloads but the Open Enclave + MAA SGX flow is a different code path from the SEV-SNP / TDX flow. New workloads should pick the VM-level TEE.

## Google Cloud Confidential Computing

### What GCP attests to

GCP's confidential-computing line split historically between "Confidential VMs" (lift-and-shift confidential compute) and "Confidential Space" (managed TEE for multi-party collaborations). The GPU-enabled path joined later.

- **Confidential VMs on n2d, c2d, c3d (AMD SEV / SEV-ES / SEV-SNP) and c3 (Intel TDX).** The n2d family used the original SEV (memory encryption without integrity); c3d uses SEV-SNP (the chapter-01 preferred scheme). The c3 family provides TDX.
- **Confidential GKE Nodes** — standard GKE, with node pools backed by confidential VMs.
- **Confidential Space** — a managed TEE whose primary use case is multi-party data clean rooms. A workload image runs inside an attested VM; the attestation claims are federated into IAM via workload-identity-federation, and parties grant the workload identity access to their data buckets conditional on the attestation matching a known workload image. GCP's threat-model framing: "the workload identity itself is the attested artefact, not just the host".
- **A3 Confidential VMs with NVIDIA H100 CC.** The GPU-confidential path on GCP.
- **Attestation verifier.** Built into Confidential Space; also available via the `go-tpm-tools` library and the Confidential Computing API.

### What GCP does not attest to

Same as Azure, with the same caveats: the application code, the policy on the relying party, the GPU vendor's root CA, the operational hygiene around the policy allow-list.

### What is distinctive about Confidential Space

Confidential Space's design shifts the attestation audience from "the KMS releasing a key" to "the IAM subsystem granting data access". The data partner grants `objectViewer` on their cloud-storage bucket to a principal of the form:

```
principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL/attribute.workload_image_reference/WORKLOAD_IMAGE_REFERENCE
```

The `attribute.workload_image_reference` is populated by the Confidential Space attestation verifier only when the workload inside the TEE matches a known image reference (a signed container image digest). This means the data partner never hands a key to a workload; the workload's *identity* is the attestation artefact, and IAM itself gates access. Reduces the number of moving parts for multi-party collaborations materially.

### Shared-responsibility boundary, in one paragraph

> *Google Cloud guarantees that the SEV-SNP or TDX hardware functions as specified, that Confidential Space's attestation verifier produces trustworthy workload-identity claims from valid evidence, and that IAM enforces the conditions on principals derived from those claims. Google makes no guarantee about the application code inside the workload image, about the correctness of the data-partner's IAM conditions (did they bind to the right image digest? did they review the workload's manifest?), or about the lifecycle discipline around rotating the approved image reference. The data partner and the workload operator jointly own the policy design.*

### Practical gotchas on GCP

- **n2d SEV-ES is not SEV-SNP.** The older n2d family offered SEV without the SNP integrity protection. Confidential-AI deployments that make a strong claim should run on c3d (SEV-SNP) or c3 (TDX), not on n2d.
- **Container image digest vs. image reference.** Confidential Space bindings should use the *immutable content digest* (`sha256:...`), not a mutable tag. A deployer who binds to a tag has the policy silently migrate when a new image is pushed with the same tag.
- **GPU-enabled Confidential VMs are a narrower availability footprint than CPU-only confidential VMs.** A multi-region deployment plan has to verify every intended region.
- **Workload identity federation vs. KMS release.** The two patterns can be combined — "the workload identity grants access to a KMS key which then decrypts the data" — but they can also replace each other, and picking one cleanly matters for the shared-responsibility claim.

## AWS Nitro Enclaves

### What AWS attests to

AWS Nitro Enclaves is older than the SEV-SNP / TDX ecosystem and is architected differently. The key architectural facts:

- **The Nitro Hypervisor and Nitro Cards.** EC2 instances on Nitro run on a minimalist hypervisor; the "hardware" peripherals the guest sees (networking, storage, memory encryption) are implemented in dedicated silicon on the Nitro Card, which is a self-contained SoC AWS designed. The Nitro Card has its own signing keys, rooted in AWS's manufacturing CA.
- **Nitro Enclaves** are a *virtual* TEE: they are a separate VM launched from a parent EC2 instance, with no persistent storage, no interactive access, no external networking (only a `vsock` channel to the parent). The parent cannot read the enclave's memory because the Nitro hypervisor physically denies it. The enclave's memory is encrypted; the root of trust is the Nitro Card's attestation signing key, not an AMD or Intel silicon root.
- **Attestation document.** The enclave calls `nsm-api` to get an attestation document — a CBOR-encoded, COSE-signed structure containing the enclave's PCR (Platform Configuration Register) values (PCR0: enclave image hash, PCR1: kernel+bootstrap, PCR2: application, PCR3: parent instance ID, PCR4: parent instance IAM role, PCR8: enclave signing certificate). The signature is by the Nitro Card attestation PKI.
- **AWS KMS integration.** KMS condition keys (`kms:RecipientAttestation:ImageSha384`, `kms:RecipientAttestation:PCR8`) let you write a key policy that only releases a decrypted secret when the attestation document matches an allow-list.
- **AWS Nitro Security Key.** The hardware root is AWS; the root certificates are published at a stable URL (`aws-nitro-enclaves.amazonaws.com/.well-known/nitro-public-key`).

### What AWS does not attest to

Same shape as the other two — application code, policy on KMS, lifecycle discipline. One additional nuance: AWS does *not* use AMD SEV-SNP or Intel TDX for Nitro Enclaves; they built their own. This has an upside (AWS has end-to-end control of the TCB; the Nitro whitepaper argues this is tighter than depending on AMD/Intel firmware supply chains) and a downside (the attestation ecosystem is AWS-specific; the deployer who wants to compare evidence across clouds has three different formats to normalise).

### GPU-side

Nitro Enclaves historically did not have GPU access — the enclave is isolated, including from the GPU hardware on the parent instance. AWS has described plans to combine Nitro Enclaves with H100 CC-mode GPUs; <!-- needs-research: confirm exact GA status of "Nitro Enclaves with GPU access" / H100 CC on AWS, which has been iteratively announced; check latest AWS re:Invent announcements. --> For deployments that need full confidential-LLM inference today, Azure NCC H100 v5 or GCP A3 Confidential VMs are the more established choices.

### Shared-responsibility boundary, in one paragraph

> *AWS guarantees that the Nitro Hypervisor enforces the enclave's memory isolation from the parent EC2 instance, that the Nitro Card's attestation signing root is correctly managed, and that KMS enforces the attestation conditions you configure in the key policy. AWS makes no guarantee about the enclave image's contents, about the parent instance's IAM role scope, or about the operational discipline on the KMS key policy. The deployer is accountable for the enclave image allow-list (PCR0 values), the parent instance role, and all application-level controls.*

### Practical gotchas on AWS

- **Nitro Enclaves have no external network.** All I/O goes through the parent EC2 instance via `vsock`. If your inference server inside the enclave needs to call OpenAI, download a model from S3, or hit a third-party API, the parent proxies the traffic. The parent is a trust boundary for that proxied traffic. Plan accordingly — session-key material shouldn't be proxied through a non-confidential parent.
- **No GPU today for LLM inference.** Until AWS ships a mainline GPU-capable enclave shape, large-model confidential inference on AWS is the odd fit; use Nitro Enclaves for CPU-only workloads (key-signing, tokenised data processing, small-model inference) and Azure/GCP for GPU-backed confidential LLM inference.
- **Enclave images are built from EIF files.** The EIF (Enclave Image File) format is AWS-specific; the build process produces a predictable PCR0 hash for a given input. Reproducible builds are the discipline.
- **Nitro's "AWS cannot access your data" claim is strong.** The Nitro whitepaper argues that even AWS employees with admin access to the host cannot read enclave memory, because the hypervisor physically does not expose that capability. This is a legitimate privacy-claim lever that AWS Nitro has had longer than any competitor.

## The decision matrix — which hyperscaler for which shape

Use this to collapse the "which cloud" conversation to the actual decision. Do not pick on vendor loyalty.

| Business need | Pick |
| --- | --- |
| Confidential LLM inference with H100-class GPUs on an SEV-SNP-backed VM | Azure NCC H100 v5 / NDm H100 v5, or GCP A3 Confidential VM |
| Confidential LLM inference on your preferred cloud (AWS) today | Partial fit; CPU-only enclave works, GPU shape is <!-- needs-research: AWS Nitro + H100 CC status --> |
| Multi-party clean room (e.g., join advertiser data with publisher data) | Azure Confidential Clean Rooms *or* GCP Confidential Space |
| CPU-only confidential signing / key operations | AWS Nitro Enclaves is arguably the best-engineered option |
| Kubernetes-native confidential workloads | Azure AKS confidential nodes, GCP Confidential GKE Nodes |
| Lift-and-shift SGX legacy workload | Azure DCsv3 (SGX-backed) — but plan for migration to SEV-SNP or TDX |
| Avoid vendor-specific attestation format | Use IETF EAT (RFC 9711) as the normalised format; wrap the vendor evidence behind a verifier that emits EAT |
| Regulatory requirement for a specific residency + confidential property | Check the region availability table of each SKU before committing; TDX / SEV-SNP / H100 CC are not in every region yet |

## The shared-responsibility matrix — what the cloud always owns, what you always own

Across all three hyperscalers, the split is the same.

### What the cloud always owns

- The *correctness* of the TEE hardware and firmware on the host side — SEV-SNP / TDX / Nitro Hypervisor / Nitro Card.
- The attestation verifier's signing-key management (if you use the hosted verifier).
- The KMS's correct enforcement of the attestation condition (if you use the hosted KMS).
- Patching the host OS, hypervisor, microcode within the operational timeline the cloud commits to.
- Physical-security of the datacentre.
- Prompt disclosure of discovered vulnerabilities in their own stack.

### What you always own

- The code inside the enclave — your inference server, your tokeniser, your sampling logic, your request-handling middleware.
- The attestation policy — the measurement allow-list, the TCB-version floor, the `report_data` binding check.
- The KMS policy — the condition that only this measurement passes, the IAM scope on who can edit the policy.
- The operational discipline — rotating the allow-list, auditing policy changes, responding to a CVE-disclosed microcode update.
- The vendor-independent trust decisions — "do I trust AMD's root CA? NVIDIA's device identity CA? AWS's Nitro root CA?" These are terms you cite in the DPIA; the cloud does not relieve you of making them explicit.
- The application-level controls — logging policy, output filtering, moderation, memorisation controls (mod-105), DP training (mod-106).
- The customer trust and legal-contractual story — what you tell your users about the privacy guarantee, how you document it in a Privacy Policy, how it maps to your DPIA.

### The boundary one-liner

> *The cloud provides the hardware TEE and the attestation verifier; you own the policy that decides what the attestation means, and all code and data inside the enclave.*

If you cannot resolve a question using that one-liner, re-scope the question until you can.

## Writing the hyperscaler-selection paragraph

The deliverable at the end of the "which cloud" conversation is a paragraph in the confidential-AI design doc:

> *This deployment runs on {Azure NCC H100 v5 / GCP A3 Confidential VM / AWS Nitro Enclaves} because (a) it needs {GPU confidential computing / CPU-only confidential signing / multi-party clean-room collaboration}, which the chosen hyperscaler delivers via {product name}; (b) the data-residency requirement is met by region {region-name}, which has the SKU generally available as of {date}; (c) the attestation verifier is {MAA / Confidential Space / AWS KMS condition keys}, which is acceptable because {reason tied to the DPIA}; (d) the shared-responsibility boundary positions us as owner of {image allow-list, KMS policy, application controls}, which we document in {release-package section N}; (e) the residual trust assumption on the silicon vendors {AMD / Intel / NVIDIA} and on {Azure / GCP / AWS} is listed in the trust statement (chapter 01 / chapter 07 memo).*

That paragraph is the thing you read out in the architecture-review board meeting. The rest of this chapter is what defends every clause of it.

## Summary — what you can do after this chapter

- Enumerate the Azure, GCP, and AWS confidential-AI offerings by product name and know which CPU-TEE and GPU-TEE technology each uses.
- Position each hyperscaler against the trust model's threat rows from chapter 01 and know what each claims to defend vs. what it leaves to the deployer.
- Articulate the shared-responsibility boundary for each cloud in one paragraph — using the shape "cloud owns X, you own Y".
- Diagnose the three hyperscaler-specific gotchas you cannot afford to miss: Azure's Trusted-Launch vs. Confidential-VM confusion, GCP's n2d SEV vs. c3d SEV-SNP confusion, AWS Nitro Enclaves' lack of mainline GPU access.
- Pick a hyperscaler for a given business need using the decision matrix.
- Write the hyperscaler-selection paragraph for the deployment's design doc.

Chapter 04 widens the lens one more time — to Apple's Private Cloud Compute, which is the only published enterprise-blueprint statement of what a confidential-AI deployment at real scale is supposed to look like.
