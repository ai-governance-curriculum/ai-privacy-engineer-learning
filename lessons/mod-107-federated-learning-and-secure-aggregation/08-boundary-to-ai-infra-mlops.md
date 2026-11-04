# Boundary to `ai-infra-mlops` — The Privacy-Engineering Slice of the FL Platform

Federated learning is a distributed system with a privacy obligation on top. The distributed-system engineering — client-side updater binary, over-the-air update channel, model-store, round scheduler, telemetry pipeline, admin console, provisioning kit — is real, large, and *not* this role's primary scope. It belongs to the peer `ai-infra-mlops` role (level 25 in the curriculum). This module owns the privacy-engineering slice of the design; this chapter is where the two slices meet.

The split parallels mod-106 chapter 10's boundary to `ai-risk-engineer` and mod-105 chapter 10's boundary to the same role: two tracks collaborate on one deployment, and each owns a specific slice. Without the boundary written down, both engineers assume the other owns the audit trail, or the DP accountant persistence, or the TEE attestation — and neither does.

## What `ai-infra-mlops` owns on an FL platform

The `ai-infra-mlops` track (level 25) is the generalist MLOps / platform role in the curriculum. On an FL deployment, its scope:

- **Client-side updater.** The binary / app / container that runs on each participant. Build pipeline, signed-release artefacts, over-the-air update channel, version pinning, roll-back, A/B of client versions. On cross-device this is a mobile-OS-level binary; on cross-silo it is a containerised FL executor.
- **Round scheduler and job orchestration.** Which rounds run when, which clients are eligible, how a round is retried on failure, how the training run is paused / resumed / cancelled. The Flower server runtime, the TFF runtime, the FLARE admin server — these are its operational substrate.
- **Model store.** Where the global model lives between rounds. Versioning, checkpointing, access control, encryption at rest. The attestation that a server restart resumes from the right checkpoint.
- **Telemetry pipeline.** The metrics / logs / traces that flow from each participant to the central observability stack. Round duration, aggregation latency, dropout rates, utility metrics.
- **Admin console and provisioning.** For cross-silo: FLARE-style provisioning kits, admin-server UX, the operational dashboards that a site admin uses. For cross-device: the product-team dashboards that show rollout state.
- **Reliability and capacity.** Round throughput, aggregator capacity, backpressure, overload handling, SLO definitions.
- **General MLOps controls.** Model registry, experiment tracking, lineage, A/B serving, the usual MLOps substrate the FL stack sits on.

Treat `ai-infra-mlops` as the platform owner. They decide the deployment topology, the service meshes, the CI/CD pipelines, the on-call rotations. They are not the primary owner of the privacy claim.

## What this module (the `ai-privacy-engineer` mod-107) owns

Everything that makes the FL deployment a *privacy-preserving* system in a way defensible to a DPIA, a regulator, an auditor, or an acquirer.

- **The FL shape decision.** Cross-device vs. cross-silo (chapter 01). The privacy engineer validates the shape against the threat model; mlops implements the picked shape.
- **The algorithm's privacy properties.** FedAvg / FedProx (chapter 02) is the algorithm; privacy engineer specifies the client-selection discipline (uniform, participation-cap-enforced), the per-round metadata that must and must not be logged, the aggregation strategy's effect on DP accounting.
- **The library choice's privacy implications.** TFF vs. FLARE vs. PySyft vs. Flower (chapter 03). Mlops can run any of them; the privacy engineer specifies which privacy primitives must be enabled and configured.
- **Secure aggregation.** The SA protocol (chapter 04), its threat model, its parameter tuning (Shamir threshold `t`, modulus `M`, mask-seed rotation), the attestations that cover its correct deployment.
- **DP-FL.** The granularity choice (chapter 05), the noise parameters (`σ`, `C`), the accountant, the per-round and lifetime `ε` tracking, the accountant-state persistence requirement.
- **FL-specific attack measurement.** Running DLG / iDLG / cross-round MIA / property-inference against the real deployment (chapter 06) and reporting in the release package.
- **Governance documentation.** The data-flow diagram, the DPIA input, the transfer-impact-assessment supplementary-measures narrative (chapter 07).
- **The release package.** The FL-flavoured extension of the mod-106 release package — SA stanza, DP-FL stanza, attack-posture stanza, governance stanza, pointing at mlops's operational audit trail.

Articulated more plainly: mlops ships *the FL system*, you ship *the privacy claim about the FL system*.

## The handshake — four recurring joint artefacts

Four artefacts are produced jointly, each with a dominant owner but requiring the other's sign-off.

### Handshake 1 — the round-level audit log

The round log is the forensic record. It lives in mlops's observability stack (their choice of ELK / Grafana Loki / OpenTelemetry / whatever); the privacy engineer specifies the schema.

- **Mlops owns:** the pipeline, the retention, the access control, the storage, the query interface.
- **Privacy engineer owns:** the schema contract — what fields *must* be present (round number, strategy name, strategy hyperparameters, aggregation algorithm, DP accountant state after the round, participation metadata, aggregation duration, SA protocol version, SA parameters, library versions), what fields *must not* be present (individual client updates, per-client raw metrics, individual pre-aggregation gradients). The attestation that no debug path writes forbidden fields.
- **Joint sign-off:** every schema change goes through both owners.

The log is the thing the DPIA will quote and the regulator will subpoena. It is also what unblocks incident response — chapter 09 of mod-106 structured it as the DP release package's audit log; here the same discipline applies with FL extensions.

### Handshake 2 — the accountant state at checkpoint

If DP-FL is enabled, the server must persist the DP accountant state on every checkpoint, and restore it on every resume. Without this, the ε claim breaks on any server restart.

- **Mlops owns:** the checkpointing mechanism, the storage, the restart orchestration.
- **Privacy engineer owns:** the attestation that the accountant state is in every checkpoint, the test that verifies a resumed run continues the accountant correctly, the escalation path if the test fails.
- **Joint sign-off:** the restart runbook. A restart that does not verify the accountant state is a privacy incident.

### Handshake 3 — the SA protocol configuration

SA has parameters (threshold `t`, modulus `M`, PRG family, DH group, dropout tolerance). Each parameter has a mlops consequence (round latency, bandwidth) and a privacy consequence (threat tolerance).

- **Mlops owns:** the implementation, the operational metrics (round latency, SA-overhead contribution), the dropout observation, the SA library version.
- **Privacy engineer owns:** the parameter choice (justified against the chapter-04 analysis), the threat model statement, the attestation that the SA primitive has not been downgraded or bypassed in a shadowed configuration.
- **Joint sign-off:** the SA configuration file, pinned per training run and logged in the release package.

### Handshake 4 — the incident response

An FL system can have incidents in both operational and privacy dimensions.

- **Mlops-dominant incident.** Round backlog, aggregator overload, client-updater regression — operational. Mlops leads; privacy engineer observes for privacy implications.
- **Privacy-dominant incident.** DP accountant corruption (budget mis-spent), SA parameter regression (wrong threshold), a report of a practical DLG attack succeeding — privacy. Privacy engineer leads; mlops provides the forensic pull.
- **Cross-cutting.** A subject-rights request the system cannot honour technically; a breach of the accountant audit log; a disclosed vulnerability in the SA library. Joint; privacy engineer owns the claim-side and the diagnosis of whether the mechanism's guarantee still holds; mlops owns the operational mitigation and the fix deployment.

Agree on the escalation path during deployment design, not during the incident.

## The kick-off paragraph

At the start of a new FL project where both roles are staffed, the privacy engineer should say this at the design kickoff:

> *I'm the privacy engineer; I own the FL-shape decision, the SA protocol parameters, the DP-FL granularity and noise, the attack-measurement gate, the governance documentation, and the release package. {Mlops lead} is the mlops lead; they own the client-side updater, the round scheduler, the model store, the telemetry pipeline, the admin console, and the general platform SLOs. The four joint artefacts are the round-level audit log schema, the accountant checkpoint persistence, the SA configuration file, and the incident response runbook. I will attest these work; they will operate them.*

That paragraph is the thing that prevents four months of ambiguity about who is on the hook when the regulator calls.

## What `ai-infra-mlops`'s FL module (their own track) covers

For reference, when you collaborate with an mlops engineer on their own learning path:

- Their mod-107 (or equivalent) covers FL at *platform* depth — Flower / TFF / FLARE / PySyft deployment patterns, client-updater packaging, round-scheduler operation, model-store integration, observability patterns, chaos testing of the aggregator, capacity planning. It does *not* cover SA protocol internals, DP accountant choice, attack measurement discipline, or governance documentation at production depth.
- You are therefore the person they will come to with the questions "which SA threshold do I pick?", "is this logging schema privacy-safe?", "does enabling DP-FL break the model's utility too much, and what are my levers?". Be ready for those conversations.

<!-- needs-research: confirm that `ai-infra-mlops` track has an FL module and that its number is mod-107 in that track; the role boundary is stable even if the module numbering differs across tracks. -->

## Reading list an mlops-only reader should pick up to close the gap

If an mlops engineer wants to approach privacy-engineer depth for a specific FL deployment, the recommended reading order:

1. **Chapter 01** of this module — the cross-device / cross-silo shape decision.
2. **Chapter 04** of this module — SA protocol mechanics, so configuration choices are informed.
3. **Chapter 05** of this module — DP-FL granularity and composition.
4. **Chapter 06** of this module — attack measurement, so they know what the gate checks.
5. **Chapter 07** of this module — governance, so they know what the DPO wants.
6. **Kairouz et al. 2021** — the full survey.

That is the minimum before the mlops engineer can stand in as a backup reviewer on an FL release. The full module is the production-depth version.

## Common collision points

Three places the two roles routinely collide, and how to resolve each.

### Collision 1 — telemetry granularity

Mlops wants fine-grained per-client telemetry to debug operational issues. Privacy engineer does not want individual client metrics leaving the device. Resolution: *k-anonymised aggregate telemetry by default*, *individual telemetry only under a documented debug-mode flag that is off in production*, and *a log-level attestation in the release package*.

### Collision 2 — client-selection "optimisation"

Mlops (or a sibling ML engineer) wants to pick clients with fast networks, high-quality data, or recent participation. Privacy engineer objects because non-uniform selection breaks DP's sampling-rate accounting (chapter 05). Resolution: *data-dependent selection is a privacy-review-gated decision*; uniform or stratified selection by non-sensitive attributes (region, device class) is fine.

### Collision 3 — model store encryption

Mlops encrypts the model at rest; privacy engineer wants to know *with what key* and *who holds it*. If the key is operational (platform-owned), the aggregator team can decrypt the model at will. For a cross-silo deployment where that violates the joint-controllership arrangement, the key needs to be held by the controllers or in a HSM / TEE (mod-108). Resolution: *document the key-custody chain in the release package*; escalate to mod-108 / `ai-security-engineer` if the control is insufficient.

## The one-sentence boundary

When in doubt about who owns what:

> *Infrastructure engineering on an FL platform belongs to `ai-infra-mlops`; everything that makes the platform's output defensible as "privacy-preserving" to a regulator belongs to the privacy engineer.*

If the task is defensibility, it is yours. If the task is operability, it is theirs. Most tasks need both.

## Summary — what you can do after this chapter

- Enumerate what `ai-infra-mlops` owns on an FL platform (updater, scheduler, store, telemetry, provisioning, SLOs).
- Enumerate what the privacy engineer owns (shape, algorithm choice, SA parameters, DP-FL design, attack measurement, governance, release package).
- Operate the four joint artefacts (audit-log schema, accountant checkpoint persistence, SA configuration, incident runbook).
- Deliver the kickoff paragraph that prevents role-collision.
- Hand an mlops engineer the six-item reading list that closes the privacy gap.
- Resolve the three recurring collision points (telemetry granularity, client-selection optimisation, model-store encryption key custody).
- Articulate the one-sentence boundary.

The module is complete. The exercises operationalise each chapter; `resources.md` collects the module's primary references.
