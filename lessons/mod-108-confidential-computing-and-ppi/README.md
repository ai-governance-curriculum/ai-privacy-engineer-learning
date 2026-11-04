# mod-108-confidential-computing-and-ppi: Confidential Computing and Privacy-Preserving Inference (TEE, HE, MPC)

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 14 hours

## Learning objectives

- Build the trust model for a confidential-AI deployment — what the TEE threat model protects against (curious cloud operator, curious co-tenant, curious insider) and what it does not (compromised firmware, side-channel attacks, trusted authority compromise); position Intel SGX, AMD SEV-SNP, and NVIDIA H100 Confidential Computing against that model
- Deploy a TEE-based LLM inference node — remote attestation (AMD SEV-SNP guest / NVIDIA H100 CC attestation), image-measurement verification, encrypted-prompt / encrypted-response path, KMS-held session keys
- Position hyperscaler offerings — Azure Confidential AI, GCP Confidential VMs / Confidential Space, AWS Nitro Enclaves — and understand the shared-responsibility boundary between the cloud provider and the AI deployer
- Read the Apple Private Cloud Compute (PCC) security design as the published enterprise blueprint — stateless computation, attested images, non-targetability, verifiable transparency — and translate each property into engineering requirements for your own deployment
- Engineer homomorphic-encryption inference on simple models with TenSEAL (CKKS / BFV) and Microsoft SEAL — understand the performance ceiling, when HE is the right tool (tiny models, low throughput, maximum trust guarantee) and when it is not
- Engineer secure multi-party computation (MPC) inference on a cross-organisation setting with CrypTen (research-grade) and MP-SPDZ (production-adjacent) — know the arithmetic / boolean protocol choice and the round-complexity cost
- Author a confidential-AI deployment risk-and-cost memo — what threat it defeats, what residual trust assumption remains, what the latency / throughput / dollar cost is vs. the non-confidential baseline
- Cite the boundary to `ai-infra-security` (level 35) on platform-scale enclave attestation + KMS / HSM integration + fleet-scale key rotation — this module owns the privacy-engineering design; platform hardening is linked out

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
