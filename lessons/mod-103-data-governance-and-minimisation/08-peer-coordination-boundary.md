# The Peer Coordination Boundary

Data governance is a shared surface. The AI Privacy Engineer (this role, level 30) owns the *policy, documentation, and privacy-specific controls* described in this module. Three peers at level 25 own the *platform and application mechanics* that carry those controls:

- **`training-pipeline-engineer`** (ML Engineering family, level 25) — the training pipeline itself. Ingest, feature store, training-job orchestration, snapshotting, dataset-versioning primitives, training-run reproducibility.
- **`rag-engineer`** (AI Engineering family, level 25) — the retrieval surface. Ingest to retrieval corpus, embedding, index, retrieval orchestrator, chunking strategy, ACL plumbing.
- **`llm-application-developer`** (AI Engineering family, level 25) — the inference-time application. Prompt assembly, orchestration, tool-calling, user-session management, UI.

This chapter draws the sharp line — what you deliver, what you consume from the peer, and what you co-author. The line matters because mis-ownership leaks into real incidents: the DPO expects *you* to have engineered propagation into the training snapshot; the ML platform expects *the peer* to have operationalised it; nobody has.

## Why this matters

Every artifact this module produces lives at a boundary between privacy ownership and a peer role's ownership:

- The **datasheet** is authored by you — but the dataset itself is produced by the training-pipeline / rag / app peer.
- The **propagation contract** is designed by you — but the columns and job manifests are implemented by the peer.
- The **minimisation checkpoints** are specified by you — but the CI lints and the schema enforcement live in the peer's repo.
- The **vault + tokenisation integration** is designed by you — but the training / retrieval / inference call-sites are the peer's.
- The **synthetic-data audit** is run by you — but the synthesiser is run by the peer.
- The **ML-BOM** is signed off by you — but the pipeline emits it.

The coordination is baked into every chapter of this module. This chapter names it explicitly so you can write the hand-off cleanly.

## RACI — four roles, six artifacts

| Artifact | AI Privacy Engineer (L30) | training-pipeline-engineer (L25) | rag-engineer (L25) | llm-application-developer (L25) |
|---|---|---|---|---|
| Datasheet per corpus (ch. 02) | **R, A** — authors, signs | **C** — supplies contents | **C** — supplies retrieval-side contents | **I** |
| Propagation contract (ch. 03) | **R, A** — designs, specifies | **R** — implements in training-ETL | **R** — implements in vector-DB metadata | **R** — implements in prompt + log |
| Minimisation checkpoints (ch. 04) | **R, A** — specifies, verifies | **R** — training-side enforcement | **R** — retrieval-side enforcement | **R** — inference / output enforcement |
| Vault + tokenisation integration (ch. 05) | **R, A** — designs, policy | **R** — training-side integration | **R** — retrieval-side integration | **R** — inference-side integration |
| Synthetic-data audit (ch. 06) | **R, A** — audits, signs | **C** — runs synthesiser | **C** — consumes substituted data | **C** |
| ML-BOM (ch. 07) | **R, A** — schema, review | **R** — pipeline emits | **R** — retrieval components emits | **I** |

R = Responsible (does the work); A = Accountable (signs off); C = Consulted; I = Informed.

Three adjacent roles are also consulted on each: **DPO** (statutory privacy oversight), **Security Peer** (security-side controls), **ai-risk-engineer** (level 25, Governance family — broader harm model).

## What this role owns end-to-end

Five deliverables where ownership is *unambiguous* — the regulator-facing line is yours:

- **The datasheet set** — one per corpus, signed. If the datasheet is wrong or missing, this role is accountable.
- **The residual-risk claim** — one paragraph per datasheet, signed by DPO + data owner + this role.
- **The propagation schema and the job-manifest matcher** — the specification; peers implement it to spec.
- **The minimisation-checkpoint register** — the full list, with enforcement status.
- **The synthetic-data audit report** — one per synthetic set; the audit is this role's work product.

Each is a review artifact the DPIA (mod-111) consumes.

## What this role consumes from peers

What this role cannot do in isolation and must receive:

- From the **training-pipeline-engineer**: dataset-versioning primitives, training-run reproducibility (same inputs → same outputs or a documented deviation), snapshot manifest with the content hash, feature-store lineage, training-job orchestration logs. Without these, the ML-BOM is a guess.
- From the **rag-engineer**: embedding-encoder version + hash, index schema, chunking strategy, retrieval telemetry, tenant-ACL implementation (mod-109 depth). Without these, the retrieval-corpus datasheet is approximate.
- From the **llm-application-developer**: prompt-assembly code, tool-call surface, session-management policy, logging configuration, UI attribution. Without these, the output-minimisation checkpoints cannot be verified.

A pattern that helps: each peer publishes a **capabilities declaration** (what the platform guarantees) and a **contract declaration** (what the platform expects from upstream). This role writes against the capabilities and reviews the contract.

## What this role does *not* own

Three surfaces that look like privacy but belong to peer roles:

- **Training-pipeline reliability.** The pipeline's failure modes (job failed; snapshot corrupted; versioning drifted) are the training-pipeline-engineer's concern. Where the failure has a privacy side-effect, you are *consulted*; you do not own the fix.
- **Retrieval-ACL implementation depth.** The specification of tenant-scoped retrieval lives here (chapter 03 propagation + mod-109 ACL contract); the implementation and its negative-test harness live with rag-engineer + mod-109.
- **Prompt engineering and model behaviour.** The system-prompt wording, the output format, the tool-use plan are the llm-application-developer's. Where their decisions affect privacy (output minimisation, cross-session memory), you set the policy; the implementation is theirs.

A common anti-pattern is for the privacy engineer to *rewrite* the peer's work to fit the policy. The correct move is to *specify the policy as a reviewable contract* and let the peer implement — this is how the hand-off survives staff changes.

## Co-authored deliverables

Three deliverables you jointly author:

### Dataset registration

- You write the datasheet (ch. 02). The training-pipeline-engineer registers the dataset in the data catalog and emits the ML-BOM fragment.
- The training-pipeline-engineer commits the dataset's manifest; a CI check verifies the datasheet exists and matches the manifest's structural metadata (row counts, schema, source signature).

### Retrieval-corpus ingest

- You specify the propagation metadata that each chunk must carry (ch. 03).
- The rag-engineer implements the chunk-level metadata plumbing in the ingest job and the vector DB schema.
- A CI test asserts a seeded sensitive row does not survive the encoder without the tokenisation step.

### Prompt / completion log policy

- You specify what is loggable and under what retention (ch. 04 output + retention + mod-109 depth).
- The llm-application-developer implements the logger's redaction / token-mode switches and the retention job's deletion.
- The security peer consults on access-control for the log sink.

In each case, the chapter 02 datasheet, the chapter 03 contract, the chapter 04 checkpoints, and the chapter 07 ML-BOM are *the* review documents.

## Working with the DPO

The DPO is a *statutory* role (GDPR Art. 37–39) and not a peer engineer. The practical shape:

- The DPO signs the residual-risk claim and the datasheet, and signs off on the DPIA.
- The DPO needs *readable*, not implementation-level, evidence. Your artifacts are written to be DPO-consumable.
- A DPO flagging a concern on a datasheet is a release-blocker. Allocate time in the release plan for DPO review (10 business days is a reasonable shape).
- You do not substitute for the DPO's judgement; they do not substitute for your engineering.

## Working with Security

Security is the peer owning the attack-surface half. Where this module's controls overlap with security controls:

- The vault's network boundary, IAM, HSM, KMS rotation — security.
- The vault's *privacy policy* (who gets to detokenise for what purpose) — this role.
- Prompt / completion log access-control — security.
- Prompt / completion log *redaction rules* — this role.
- Encryption in transit + at rest — security.
- The decision that regulated-sector data is in scope and therefore requires confidential-compute inference — this role.

A short joint-RACI with the security peer at the start of a project avoids the dual-ownership tangle at release time.

## Working with `ai-risk-engineer` (level 25, same family)

The peer `ai-risk-engineer` (also level 25, Governance family) owns the broader *harm-model* surface — misuse personas beyond privacy, capability-surface harms, downstream integrator abuse, operator / model / data / integration-caused harms, and the privacy-risk-engineering slice that mod-107 of that curriculum covers at working depth.

The depth boundary to that peer is the same shape as mod-102 chapter 09:

- `ai-risk-engineer` covers privacy risk at *working* depth as one pillar of the harm model.
- This role covers privacy at *engineering-ownership* depth — the datasheet, the propagation, the minimisation, the vault, the synth audit, the ML-BOM.
- Where both appear, cite into the peer's artifact rather than duplicating. On a shared engagement, the harm-model contact is the peer; the engineering contact is you.

## A hand-off checklist for a new AI system

When a new AI system comes on-scope, walk this list with the three peer roles:

- [ ] Datasheet template exists for each corpus the system will touch; peers know the fields they are expected to populate.
- [ ] Propagation schema is adopted by the training-ETL and the vector-DB schema.
- [ ] Vault + tokenisation integration touch-points are identified; vault policy draft reviewed.
- [ ] Minimisation checkpoints are mapped to CI lints / release gates / runtime monitors in each peer's repo.
- [ ] Synthetic-data policy — which corpora are synthesisable, which are not — is agreed and recorded.
- [ ] ML-BOM emission is wired into the pipeline; `compositions.aggregate = complete` is asserted.
- [ ] DSR / unlearning routing (mod-110) is sketched, so provenance carries far enough to be actionable.
- [ ] DPO review window is on the release plan.

A missed item here is a known gap, not a surprise finding.

## Summary — what you can do after this chapter

- State which five artifacts this role owns end-to-end and which three it specifies for peers.
- Name the consumption inputs from training-pipeline-engineer, rag-engineer, and llm-application-developer that this role depends on.
- Co-author the dataset-registration, retrieval-ingest, and log-policy workflows with the correct peer in each case.
- Work with the DPO as the signing authority and with the security peer on the attack-surface overlap.
- Cite into `ai-risk-engineer` (level 25) for the harm-model depth rather than duplicating.

This chapter closes the lecture material. Exercises 01–06 convert the chapters into the hands-on bundle you will carry into mod-104 (PII pipeline), mod-109 (RAG privacy), mod-110 (unlearning), mod-111 (DPIA), and the mod-103-adjacent exercises in projects 101–103.
