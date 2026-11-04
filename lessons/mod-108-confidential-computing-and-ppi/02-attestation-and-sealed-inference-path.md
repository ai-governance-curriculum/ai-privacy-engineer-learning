# Attestation and the Sealed Inference Path — SEV-SNP, H100 CC, Image Measurement, KMS-Gated Session Keys

A TEE without attestation is a locked box with no proof of its own identity. Anyone can stand up a VM, call it "confidential", and tell a user "trust us, your prompt runs in a TEE". *Remote attestation* is the cryptographic mechanism by which the TEE proves to a *remote verifier* — a KMS, a key-release service, an auditor — that it is (a) running on genuine confidential-computing hardware, (b) booted with a specific software image whose measurement matches a known-good value, and (c) patched to at least a specified TCB version. Only after the verifier checks those claims does it release the session key the enclave needs to decrypt an incoming prompt.

This chapter builds the full sealed-inference path, AMD SEV-SNP guest + NVIDIA H100 CC. It is the piece of engineering that turns the trust model from chapter 01 into a running deployment.

Primary references:

- AMD — *SEV Secure Nested Paging Firmware ABI Specification*, Publication 56860. [amd.com/content/dam/amd/en/documents/epyc-technical-docs/specifications/56860.pdf](https://www.amd.com/content/dam/amd/en/documents/epyc-technical-docs/specifications/56860.pdf) — the primary specification. Read the sections on `SNP_GUEST_REQUEST / MSG_REPORT_REQ` and the attestation-report format.
- AMD — *SEV Certificate Specification*. [amd.com/content/dam/amd/en/documents/epyc-technical-docs/specifications/57230.pdf](https://www.amd.com/content/dam/amd/en/documents/epyc-technical-docs/specifications/57230.pdf) — describes the VCEK / ASK / ARK certificate chain the verifier walks.
- NVIDIA — *Confidential Compute with NVIDIA H100 Tensor Core GPU* (developer deployment guide). [docs.nvidia.com/confidential-computing-deployment-guide.pdf](https://docs.nvidia.com/confidential-computing-deployment-guide.pdf) and *NVIDIA Attestation SDK*. [github.com/NVIDIA/nvtrust](https://github.com/NVIDIA/nvtrust) — the operational documentation and the open-source attestation SDK for H100 CC.
- DMTF — *Security Protocol and Data Model Specification* (DSP0274). [dmtf.org/sites/default/files/standards/documents/DSP0274_1.3.0.pdf](https://www.dmtf.org/sites/default/files/standards/documents/DSP0274_1.3.0.pdf) — the device-authentication standard NVIDIA's accelerator attestation builds on.
- IETF RATS — *Remote ATtestation procedureS (RATS) Architecture*, RFC 9334. [datatracker.ietf.org/doc/html/rfc9334](https://datatracker.ietf.org/doc/html/rfc9334) — the standardised vocabulary (Attester / Verifier / Relying Party / Evidence / Attestation Result). Use these names in your design docs; they are what a careful reviewer will match against.
- IETF — *Entity Attestation Token (EAT)*, RFC 9711 (formerly draft-ietf-rats-eat). [datatracker.ietf.org/doc/html/rfc9711](https://datatracker.ietf.org/doc/html/rfc9711) — the CWT/JWT-based claim format the verifier produces; useful whenever a KMS policy wants to consume a standard token rather than a vendor-specific report.
- Microsoft — *Azure Attestation Service* documentation. [learn.microsoft.com/en-us/azure/attestation/overview](https://learn.microsoft.com/en-us/azure/attestation/overview) — Azure's hosted verifier that produces EAT-style tokens from SEV-SNP, TDX, SGX, and VBS evidence.

## The RATS vocabulary — four roles your design doc must name

RFC 9334 defines the roles precisely. Use these words consistently.

- **Attester.** The entity producing evidence of its own state. In our case, the confidential VM (SEV-SNP guest) and its attached GPU (H100 CC).
- **Verifier.** The entity that ingests raw evidence, walks the certificate chain, and decides whether the evidence is valid. Can be a hyperscaler's hosted service (Azure Attestation, GCP's attestation verification in Confidential Space), an open-source verifier (`snpguest` CLI + custom policy, NVIDIA's `nvtrust` verifier), or a bespoke in-house component.
- **Relying party.** The entity that acts on the verifier's conclusion. In a confidential-AI deployment, this is the KMS that releases the session key (AWS KMS with a condition policy, Azure Key Vault with attestation-based access policies, HashiCorp Vault with the `vault-attestation` plugin or similar) or your custom key-release service.
- **Supply chain / endorser.** The entity whose signatures underpin the trust in the hardware. AMD for SEV-SNP, Intel for TDX, NVIDIA for H100 CC. Their root certificates are the terminal trust anchor.

The diagram your design doc should contain:

```
+------------+   evidence   +----------+   attestation   +------------+
|  Attester  | -----------> | Verifier | --------------> |  Relying   |
|  (VM+GPU)  |              |          |  result (EAT)   |   party    |
+------------+              +----------+                 |   (KMS)    |
       |                         |                       +------------+
       |                         | checks chain against         |
       |                         | AMD/Intel/NVIDIA roots       | releases
       |                         |                              | session key
       |                                                        |
       +<------------------- session key ------------------------+
```

Nothing clever — just these four boxes, with your specific tool-chain named in each. If your design doc has an "attestation service" with no indication of which role it plays, that is the first question to ask in review.

## AMD SEV-SNP guest attestation, step by step

The following is the SEV-SNP guest's path from "VM just booted" to "has a session key that only an attested instance of this specific image could get".

### Step 1 — The VM boots; the Secure Processor records the launch measurement

When the guest VM is launched, the AMD Secure Processor (SP, the on-die ARM core whose firmware mediates SEV-SNP) computes a *launch measurement*: a cryptographic hash over the initial memory layout, the OVMF firmware blob the guest will execute from, and the VMSA (VM Save Area) for each vCPU. This measurement becomes part of the guest's "identity" for the rest of its lifetime — any later attestation report will include it in the `MEASUREMENT` field.

Critically, the launch measurement is deterministic: if two deployers use the same OVMF binary and the same boot parameters, they will get the same measurement. This is what makes "known-good measurement" a thing — your relying-party policy says "accept only measurements in the set `{M_approved_1, M_approved_2, ...}`" and the deployer can compute the expected measurement off-line (the AMD `sev-snp-measure` tool does this: [github.com/virtee/sev-snp-measure](https://github.com/virtee/sev-snp-measure)) and configure the policy with it.

### Step 2 — The guest requests an attestation report

Inside the guest, the SEV-SNP driver exposes a device (`/dev/sev-guest` on Linux) that accepts an `SNP_GET_REPORT` ioctl. The ioctl takes a 64-byte *report data* field — the critical slot where the guest places a nonce or a hash it wants to bind to the attestation. For our confidential-AI case, this is where the guest puts the public key of an ephemeral ECDH key-pair it just generated. The attestation report, once signed, binds that public key to this specific attested instance.

The SP returns a signed report. The report structure (abridged — see the AMD ABI spec for the full layout):

```
AttestationReport {
    version                  // report format version
    guest_svn                // security version number the guest rolled forward to
    policy                   // guest policy (debug off, migration disabled, SMT settings, etc.)
    family_id, image_id      // optional guest-declared IDs
    vmpl                     // VM Privilege Level of the caller
    signature_algo
    current_tcb              // current TCB version (microcode, snp-fw, etc.)
    platform_info            // SMT enabled, TSME, etc.
    report_data[64]          // the guest's nonce / public key / binding data
    measurement[48]          // the launch measurement — SHA-384 of initial memory
    host_data[32]            // 32 bytes the hypervisor can inject at launch (not secret; used to tag)
    id_key_digest, author_key_digest   // optional key-chain for VM identity
    report_id, report_id_ma
    reported_tcb             // the TCB the report is signed under
    chip_id[64]              // unique per-chip identifier
    signature                // VCEK signature over all preceding fields
}
```

### Step 3 — The verifier walks the certificate chain

The report is signed by a VCEK (Versioned Chip Endorsement Key), a per-chip key that is *itself* signed by AMD's SEV signing CA (ASK), which is signed by AMD's SEV root CA (ARK). The verifier walks this chain:

```
report.signature  --->  verified by VCEK
VCEK certificate  --->  verified by ASK
ASK certificate   --->  verified by ARK
ARK certificate   --->  pinned root (AMD's published key)
```

AMD publishes the ARK on its Key Distribution Server (`kdsintf.amd.com`); the VCEK for a specific chip is fetched by the verifier from AMD's KDS using the `chip_id` and the `reported_tcb` as query parameters. A verifier that fetches the certificate chain at attestation time has a weak dependency on AMD's KDS being online; mature deployments cache the ARK and ASK (the two long-lived roots) and only hit KDS for per-chip VCEKs.

### Step 4 — The verifier applies policy

A valid signature on a well-formed report is a necessary but not sufficient condition. The verifier's *policy* decides whether the report represents an acceptable state:

- `measurement` must be in the approved set — this is where "image attestation" enforces "the VM is running the code we approved".
- `reported_tcb` must be at or above the current floor — the side-channel mitigation lever from chapter 01.
- `policy.debug` must be `0` — a debug-enabled guest can be single-stepped by the hypervisor and is not a real TEE.
- `policy.smt` and `policy.migrate_ma` decide whether multi-threading and migration are allowed; most confidential workloads disable both for a tighter threat model.
- `vmpl` must be the expected privilege level — guards against a report issued from a less-privileged VMPL being accepted for a higher-trust context.
- `report_data` must match the binding the guest claimed — this is where the ephemeral public key from Step 2 gets locked into the policy.

Encoding this policy as JSON / Rego / CEL is a design choice; the salient point is that *every field above has to be checked*, and if a field is missing from the policy, you have a silent degradation of the guarantee. The acceptance test for the policy is: *given a correctly-formed attestation report with one tampered field, the verifier rejects it*. If the verifier accepts the tampered report, the policy has a gap.

### Step 5 — The verifier produces an attestation result

The verifier emits an EAT — an Entity Attestation Token, usually a JSON Web Token (JWT) or a CBOR Web Token (CWT) — summarising its conclusion. The token includes the claims the verifier has validated (the measurement, the TCB version, the chip ID), its own signature, and a short validity period. Downstream components (the KMS) consume the EAT instead of the raw SEV-SNP report; this decouples the KMS from AMD-specific cryptography.

The relying-party policy then becomes: *release session key X if and only if the token is (a) signed by the verifier I trust, (b) claims a measurement in the approved set, (c) claims a TCB version at or above the floor, (d) is less than N seconds old.*

Azure Attestation and Google Cloud Confidential Space both ship this exact flow out of the box; the private-cloud shape (e.g., bare-metal with a self-hosted verifier) requires standing up the verifier yourself. In both cases the *policy* is the engineering artefact that owns the privacy-claim side.

## NVIDIA H100 Confidential Computing attestation

The GPU-side attestation is conceptually parallel to the CPU-side but has its own primitives. Its steps:

### Step 1 — Device identity

Each H100 GPU has a per-device identity certificate rooted in NVIDIA's device identity CA. The certificate is provisioned at manufacturing and includes the GPU's serial number and its public ECC key. The CPU TEE establishes a session with the GPU using the DMTF SPDM protocol, at which point the GPU presents its device certificate.

### Step 2 — Device evidence

The CPU TEE issues an SPDM `GET_MEASUREMENTS` request to the GPU. The GPU responds with signed measurements of its firmware image, the current configuration state (confidential computing mode on/off, GPU memory encryption key status), and a nonce the CPU provided. This is the GPU-side analogue of the SEV-SNP launch measurement.

### Step 3 — Verifier

NVIDIA ships two options.

- **NVIDIA Remote Attestation Service (NRAS)** — a cloud-hosted verifier at `nras.attestation.nvidia.com` that accepts GPU evidence, walks the certificate chain back to NVIDIA's root, applies policy (firmware version must be at or above floor, CC mode must be enabled, debug mode must be off), and returns an EAT.
- **Local verifier** — the `nvtrust` open-source library ([github.com/NVIDIA/nvtrust](https://github.com/NVIDIA/nvtrust)) can be run inside the CPU TEE itself, using NVIDIA's published root certificates, so the deployment has no runtime dependency on NVIDIA's cloud service.

### Step 4 — The GPU session key

Once the attestation is accepted, the CPU TEE and the GPU complete the SPDM session key agreement. All subsequent PCIe traffic between the CPU VM and the GPU — commands, weight loads, activations — is encrypted with the session key. The GPU's HBM memory is also encrypted with keys the GPU hardware holds, so even a peer tenant with GPU-side access to the host cannot read the HBM contents.

## Composing CPU and GPU attestation into one policy

A confidential-AI deployment needs *both* attestations to pass before it releases the session key:

```
KMS policy:
  release session key K if:
    CPU attestation EAT:
      iss == "https://sharedneu.attest.azure.net" (or your verifier)
      measurement in approved_cpu_measurements
      tcb_version >= floor_cpu_tcb
      policy.debug == 0
      aud == this deployment
      nbf <= now <= nbf + 5 minutes
    AND
    GPU attestation EAT:
      iss == "https://nras.attestation.nvidia.com" (or local verifier)
      cc_mode == enabled
      firmware_version >= floor_gpu_fw
      debug == 0
      device_id matches a GPU in the approved allow-list
      nbf <= now <= nbf + 5 minutes
    AND
    CPU EAT.report_data == hash(GPU session public key || CPU ephemeral public key || deployment nonce)
```

The last condition — the binding between the CPU report and the GPU session — is the piece that prevents an attacker from pairing a valid CPU attestation with a GPU in a different (unattested) machine and racing the KMS. It is often the most-missed check in a hastily-built policy.

## The sealed inference path, end to end

Now put the attestation together with the actual prompt-and-response data flow. The sequence a confidential-AI inference node runs for each user's request:

### At boot (one-time per VM lifetime)

1. The confidential VM boots. SEV-SNP firmware records the launch measurement.
2. The inference server inside the enclave starts. It generates an ephemeral ECDH key-pair `(sk_e, pk_e)`.
3. The server requests an SNP attestation report with `report_data = SHA-256(pk_e || deployment_nonce)`.
4. The server initiates an SPDM session with the attached H100 GPU and obtains the GPU attestation evidence; it verifies the GPU evidence locally (if using `nvtrust`) or submits to NRAS.
5. The server submits both pieces of evidence to the verifier, which emits an EAT containing both sets of claims.
6. The server presents the EAT + `pk_e` to the KMS. The KMS applies policy and, if the policy passes, releases the long-term secrets the enclave needs (model-weight decryption keys, if the weights are stored encrypted; back-channel API keys; etc.), optionally encrypted to `pk_e`.
7. The server decrypts the model weights (from encrypted-at-rest storage) and loads them into GPU memory via the attested SPDM session. The weights never exist in plaintext outside the enclave boundary.
8. The server publishes `pk_e` (the one the attestation bound) on a short-lived rotation so clients can discover it.

### Per request (one per prompt)

9. The user's client fetches the enclave's current `pk_e` and the verifier's EAT. The client *verifies the EAT itself* — against the verifier's public key and the same policy the KMS enforces. This is the non-negotiable client-side step: it is what prevents a man-in-the-middle from substituting an unattested endpoint.
10. The client generates an ephemeral `(sk_c, pk_c)`. It computes the shared secret via ECDH with `pk_e`, derives a symmetric session key with HKDF.
11. The client encrypts the prompt with the session key (AES-GCM or similar AEAD) and sends `(pk_c, ciphertext)` to the enclave.
12. The enclave derives the same session key via ECDH `(sk_e, pk_c)`, decrypts the prompt. The plaintext exists only inside the enclave's memory.
13. The enclave processes the prompt (tokenise, forward-pass on the H100, decode). The intermediate activations live on the GPU, encrypted from the outside; the KV cache lives in GPU HBM, protected.
14. The enclave encrypts the response with the same session key and returns it to the client.
15. The enclave discards the session key (and all request-local state) promptly. If the deployment is *stateless* (Apple PCC's preferred property, chapter 04), no residue remains.

### The invariant

At every moment, the plaintext prompt and the plaintext response exist *only* inside the enclave's memory. The network path is end-to-end encrypted with a session key derived from an attested ephemeral public key. The KMS has released no secret until the attestation checked. The GPU-side session key is established only after the GPU is attested. The weight decryption key is released only after the full policy chain passes. The user's trust in the deployment reduces to trust in the attestation policy — which is auditable and documentable.

## Image measurement verification — the known-good set

The `measurement` field is "SHA-384 of the initial memory of the VM, including the firmware and boot parameters". Two engineering realities follow.

**Realised measurement is deterministic only to the extent the build is deterministic.** Non-reproducible builds, non-deterministic OVMF, randomised boot parameters all produce different measurements across runs of the "same" image. The fix is a reproducible build pipeline and a locked OVMF version. AMD's `sev-snp-measure` tool computes the expected measurement given the OVMF image and the boot parameters, so your CI can produce the expected measurement at build time and publish it alongside the image.

**The approved set must be explicit.** "Accept any measurement signed by the right chip" is not enough — any measurement would include images the attacker could themselves build and boot. The policy must specify a *finite allow-list* of measurements, each of which corresponds to an audited image version. The lifecycle is (a) build a new image version, (b) compute the expected measurement, (c) run the privacy-engineering review on what the image contains, (d) sign off and append the measurement to the policy, (e) rotate the fleet onto the new version, (f) remove the old measurement from the policy after all instances are upgraded. Chapter 08 names the fleet-scale operational discipline this implies; this chapter owns the discipline of "the allow-list is the artefact you sign off".

**For the GPU side:** the "measurement" equivalent is the GPU firmware version and the CC-mode state. The allow-list is a set of `(firmware_version, cc_mode == enabled)` pairs, updated on each NVIDIA firmware release after a corresponding review.

## KMS-held session keys — the policy is the control

The KMS is the enforcement point. Three deployment shapes are common; each has its own policy syntax but the same semantic requirements.

### Shape A — AWS KMS with attestation condition (Nitro-Enclaves flavour)

AWS Nitro Enclaves ship with a KMS integration where the KMS key policy can include a `kms:RecipientAttestation:ImageSha384` condition. The enclave submits its attestation document with the `Decrypt` call; KMS checks the condition before releasing the plaintext. The policy is JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "EnclaveOnly",
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::ACCOUNT:role/enclave-role" },
    "Action": "kms:Decrypt",
    "Resource": "*",
    "Condition": {
      "StringEqualsIgnoreCase": {
        "kms:RecipientAttestation:ImageSha384": "<the known-good measurement>",
        "kms:RecipientAttestation:PCR8": "<signing cert measurement>"
      }
    }
  }]
}
```

Nitro Enclaves are not SEV-SNP; AWS uses its own Nitro hardware root. But the policy shape — "KMS releases key only when the attestation claim matches a known measurement" — is identical and transferrable.

### Shape B — Azure Key Vault + Managed HSM with attestation policy

Azure Key Vault's "secure key release" (SKR) supports a policy that the key can be released only to a workload whose attestation claims match. The policy is a JSON document referencing the Azure Attestation Service's claims:

```json
{
  "version": "1.0.0",
  "anyOf": [{
    "authority": "https://sharedneu.attest.azure.net",
    "allOf": [
      { "claim": "x-ms-isolation-tee.x-ms-attestation-type",
        "equals": "sevsnpvm" },
      { "claim": "x-ms-isolation-tee.x-ms-compliance-status",
        "equals": "azure-compliant-cvm" },
      { "claim": "x-ms-isolation-tee.x-ms-sevsnpvm-launchmeasurement",
        "equals": "<the known-good measurement>" },
      { "claim": "x-ms-isolation-tee.x-ms-sevsnpvm-tcb-version",
        "atLeast": "<minimum TCB version>" }
    ]
  }]
}
```

Google Cloud offers a parallel mechanism via Confidential Space's workload identity federation and the attestation verifier claims.

### Shape C — Self-hosted KMS (HashiCorp Vault, in-house service)

A self-hosted KMS follows the same shape with its own policy language. The point is not the language; it is the *invariant*: *the session key leaves the KMS only if the EAT passes the policy*. Any deployment that releases keys before verifying attestation has the TEE as theatre.

## Three common policy failure modes

In design review, look for each of these.

### Failure 1 — Policy that accepts without checking the measurement

Symptom: the policy checks the attestation signature but not the measurement allow-list. Result: any correctly-booted VM on the right hardware passes, including one running the attacker's substituted binary. Fix: the measurement allow-list is non-optional; the policy must fail-closed when the policy file is empty.

### Failure 2 — Policy that accepts without a TCB floor

Symptom: the policy does not check `reported_tcb`. Result: a VM on a CPU with a known, unpatched side-channel vulnerability passes attestation. Fix: the TCB floor is maintained as an operational artefact and raised promptly after each CVE; the policy references the current floor.

### Failure 3 — Policy that does not verify the `report_data` binding

Symptom: the policy accepts any valid report regardless of what the guest put in `report_data`. Result: an attacker can race a legitimate attestation report against a stolen session-key request bound to a different ephemeral public key — a replay-pairing attack. Fix: `report_data` must equal `H(ephemeral_pk || deployment_nonce || GPU_binding)` and the policy must verify it exactly.

## What this chapter does not cover

- **Fleet-scale attestation operations** — rotating allow-lists across thousands of hosts, replacing VCEK certificates in bulk, running the verifier at scale. That is `ai-infra-security` scope; see chapter 08.
- **HSM hardware for the KMS itself** — the choice between cloud-KMS (software-backed), cloud-KMS (HSM-backed), and bring-your-own HSM. `ai-infra-security` discipline; chapter 08 names the boundary.
- **TDX-specific syntax** — Intel TDX has its own report format (`TDREPORT`) with similar semantics but different field names. The policy shape is identical; the field mappings live in Intel's TDX ABI spec and in Azure/GCP's attestation claim documentation.
- **AWS Nitro Enclaves end-to-end** — covered in chapter 03, which positions the three hyperscaler shapes.

## Summary — what you can do after this chapter

- Name the four RATS roles (attester, verifier, relying party, endorser) and draw the attestation flow with your deployment's specific components in each.
- Walk the SEV-SNP attestation report structure field-by-field and explain the role of `measurement`, `report_data`, `reported_tcb`, `vmpl`, and `policy.debug` in a release policy.
- Walk the H100 CC attestation flow (SPDM session, device certificate, measurements, EAT) and know when to use NRAS vs. a local `nvtrust` verifier.
- Compose a KMS policy that releases session keys only when *both* CPU and GPU attestations pass, and the two are bound via `report_data`.
- Articulate the sealed inference path from "VM boots" to "response returned", naming every cryptographic artefact and the point at which each is released.
- Diagnose the three common policy failure modes (missing measurement check, missing TCB floor, missing report-data binding).
- Operate the known-good-measurement allow-list: build-time computation, review, append, rotate, remove.

Chapter 03 widens the lens to the hyperscalers — Azure, GCP, AWS — and the shared-responsibility boundary each defines. Chapter 04 reads Apple's PCC as the published enterprise reference for everything we have just built.
