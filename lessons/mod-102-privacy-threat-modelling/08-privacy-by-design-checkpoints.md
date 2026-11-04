# Privacy by Design as Engineering Checkpoints

Ann Cavoukian's seven Privacy-by-Design (PbD) principles are the foundational text that GDPR Art. 25 codifies as *data protection by design and by default*. For an AI Privacy Engineer, PbD is **not** a slide at the start of a kickoff — it is a set of **engineering checkpoints** embedded in design review, CI, release gating, and operations. This chapter converts each principle into a checkpoint spec you can enforce.

Primary sources:

- Cavoukian, A., *Privacy by Design — The 7 Foundational Principles*. [iapp.org/resources/article/privacy-by-design-the-7-foundational-principles](https://iapp.org/resources/article/privacy-by-design-the-7-foundational-principles/).
- GDPR Art. 25 at [eur-lex.europa.eu/eli/reg/2016/679/oj](https://eur-lex.europa.eu/eli/reg/2016/679/oj) (and EDPB Guidelines 4/2019 on Art. 25).
- NIST Privacy Framework Core (mod-101 chapter 07) — the analogous engineering-management shape.

Read the Art. 25 text and the EDPB guidelines before a DPIA review; the guidelines are what examiners will quote.

## The seven PbD principles in Cavoukian's words

1. **Proactive not reactive; preventive not remedial.**
2. **Privacy as the default setting.**
3. **Privacy embedded into design.**
4. **Full functionality — positive-sum, not zero-sum.**
5. **End-to-end security — full lifecycle protection.**
6. **Visibility and transparency — keep it open.**
7. **Respect for user privacy — keep it user-centric.**

Read as sentences they are ambitious and non-operational. The job of this chapter is to make them operational.

## Principle 1 — Proactive not reactive: checkpoints at design review

**Checkpoint P1-A — Privacy threat model exists before implementation.** A LINDDUN GO pass (chapter 04) runs at the design-review stage. The PR / RFC proposing the system cannot merge until the threat-model document is linked. The threat-model document must name owner, date, DFD version, scope decisions.

**Checkpoint P1-B — High-risk processing has a DPIA plan.** If any Art. 35 trigger fires (large-scale processing of personal data, systematic monitoring, special-category data, Art. 22 ADMT, EU AI Act high-risk category), the design doc references the planned DPIA schedule and names the DPO delegate. mod-111 delivers the DPIA itself.

**Checkpoint P1-C — "No surprise sinks" review.** Before merge, the DFD is checked against the nine-sink template (chapter 02). Any sink that was *added* post-design-review triggers a re-opening of the threat model.

**Review pattern.** Add a `privacy-threat-model-link` field to the RFC template. Gate the RFC approval on its presence.

## Principle 2 — Privacy as the default setting

**Checkpoint P2-A — Default-off logging of raw prompts / completions.** The system *must* ship with prompt / completion logging off by default. Turning it on requires a documented purpose, a retention, and ACL on the log sink. Enforce as config lint: a merged config with `log_raw_prompts: true` and no documented purpose fails CI.

**Checkpoint P2-B — Default-on redaction at ingest.** Any log, cache, or evaluation-sample sink that receives raw prompts / completions runs redaction (mod-104) as the default path. Bypassing redaction requires an opt-out annotation with an owner.

**Checkpoint P2-C — Default-on retrieval scope.** A multi-tenant retrieval index requires tenant-scoped queries by default. "Query all tenants" must be an explicit, audited admin path with its own ACL.

**Checkpoint P2-D — Default-off cross-session memory for agents.** Agent memory that spans sessions is off by default. Turning it on requires explicit consent capture per the lawful-basis contract (mod-101 chapter 02).

**Checkpoint P2-E — Default-short retention.** Each sink has a retention policy. The default is *the shortest horizon that still serves the documented purpose*, not the longest the storage allows. CI job tests that retention jobs actually delete.

**Review pattern.** Each of these defaults is encoded as *code* (a config lint, a CI test, a schema constraint). A principle that only exists in a style guide is not enforced.

## Principle 3 — Privacy embedded into design

**Checkpoint P3-A — Each new data path has a lawful basis before code review.** PR template includes "Lawful basis for every processing purpose in this PR". Legal basis is not retrofitted after launch.

**Checkpoint P3-B — Each new field has a data-classification tag.** Training features, retrieval columns, logging fields each carry a tag: `public / internal / personal / special-category / regulated-sector`. The tag drives downstream retention, ACL, and redaction choices.

**Checkpoint P3-C — Each new model version has a dataset / model card.** Dataset card (`intended_use`, `in_scope / out_of_scope`, `source + consent`, `special_category`, `retention`); model card (`training_data_summary`, `evaluations`, `known_limitations`, `privacy_controls`). Blocks release if missing. mod-112 teaches card content.

**Checkpoint P3-D — Each new API edge documents the data classes crossing it.** API spec carries the data-class contract for every field. Downstream integrators consume this.

**Review pattern.** Lint these into the RFC and the API spec. The data-classification tag is the single highest-value piece of design-time metadata; miss it and every downstream enforcement is weaker.

## Principle 4 — Full functionality — positive-sum, not zero-sum

**Checkpoint P4-A — Privacy / utility trade-off is quantified, not asserted.** When DP noise is added, the DP ε and the downstream task metric (accuracy / F1 / BLEU / win-rate) are both reported in the same table. "Privacy cost us too much" is not acceptable without numbers; "DP(ε=3) cost 1.4 pts F1 and shipped" is.

**Checkpoint P4-B — Privacy requirements are listed as functional requirements.** A requirement to support erasure (Art. 17) is written in the same requirements doc as a latency requirement. Both are tested.

**Checkpoint P4-C — No implicit "privacy vs. features" bargaining in a sprint.** When a privacy control and a feature compete for scope, the trade-off is escalated to the DPO + the product lead for an explicit decision recorded in the design doc. No silent de-scoping.

**Review pattern.** DP accountant log (mod-106) in the release doc; privacy acceptance criteria in the feature's test plan; escalation path in the RFC template.

## Principle 5 — End-to-end security

**Checkpoint P5-A — Encryption in transit and at rest for every sink.** TLS / mTLS on every edge. KMS-managed keys at rest. Enforce via cloud-config lint / Terraform policy.

**Checkpoint P5-B — Access controls applied, reviewed, audited.** Each sink has an ACL. Each ACL is reviewed on a fixed cadence (quarterly minimum). Role assignments log to an immutable audit sink.

**Checkpoint P5-C — Confidential compute where threat model requires it.** When the model-provider boundary carries data classified as regulated-sector, confidential compute (TEE attestation, mod-108) is applied. Non-attested inference on regulated-sector data fails deployment check.

**Checkpoint P5-D — End-to-end lifecycle protection.** Every sink has a *decommissioning* plan — tombstoning on delete, key destruction on retention end, residual-check procedure. "We'll figure out deletion later" is a Checkpoint failure.

**Review pattern.** Adopt Art. 32 wording directly — appropriate technical and organisational measures. Security peer owns the depth; you own that the privacy-specific sinks are not forgotten.

## Principle 6 — Visibility and transparency

**Checkpoint P6-A — Published privacy notice covers every processing purpose in the DFD.** Every data action in the PRAM overlay (chapter 06) maps to a notice clause. If a processing purpose is not in the notice, it is not deployed. The notice itself is version-controlled (owned by DPO; engineering provides the technical reality).

**Checkpoint P6-B — Model and dataset cards are published where appropriate.** For released / shared models, cards are public; for internal models, cards are available to the DPO and auditor. mod-112.

**Checkpoint P6-C — Art. 22 meaningful-information artifact where applicable.** For automated decision-making under Art. 22, a plain-language description of the logic involved is published and surfaced to affected subjects. mod-101 chapter 02.

**Checkpoint P6-D — Audit trail visible to the subject on request.** DSR response includes the subject-facing audit (what we know about you, what we've done with it). mod-110.

**Review pattern.** Keep a mapping table: each processing purpose → notice clause → dataset card section → DSR-response field. Regulator queries are answered by walking this table.

## Principle 7 — Respect for user privacy

**Checkpoint P7-A — Subject-initiated rights work end-to-end.** DSR flows for access, erasure, rectification, portability, objection, opt-out, ADMT-opt-out are implemented, tested end-to-end, and reach every sink in the DFD. mod-110.

**Checkpoint P7-B — Human intervention path exists for Art. 22 decisions.** A UI path that routes a contestation to a human with authority to overturn, within a documented SLA.

**Checkpoint P7-C — Consent is granular and revocable.** Where consent is the lawful basis, revocation propagates to every sink that holds the data — including fine-tune corpora and retrieval indexes.

**Checkpoint P7-D — Unintervenability tests.** For any processing where the subject cannot practicably intervene (e.g. data memorised into weights), the compensating control is documented (DP-SGD, unlearning plan, synthetic-substitute, re-training cadence) and the limitation is disclosed to the subject in the notice.

**Review pattern.** Keep DSR SLAs instrumented. A DSR that misses the Art. 12(3) one-month clock is a reportable control failure, not a backlog item.

## The engineering-checkpoint register

A checkpoint is only real if it is enforced. Record the full set in a register with these columns:

| Checkpoint ID | PbD principle | Spec | Enforcement (CI / lint / review gate / runtime) | Owner | Current status | Evidence |
|---|---|---|---|---|---|---|

Statuses:

- **Enforced** — gate exists and has blocked at least one bad merge / deploy.
- **Instrumented** — gate exists but has not fired (treat with mild suspicion; synthesise a test case).
- **Documented** — written-down expectation with no gate yet; a known gap.
- **Not applicable** — document the reason.

The register is a durable artifact. It outlives any single system. mod-112 covers how it sits in the program.

## Wiring the checkpoints to CI, review, and runtime

A short pattern catalogue:

- **CI config lint.** Reject configs that violate P2-A / P2-B / P2-C / P5-A / P5-B.
- **CI schema lint.** Reject schemas that lack P3-B data-classification tags.
- **CI retention test.** Assert retention jobs delete what they should — against a test record aged past retention.
- **Release-gate.** Block release of a model version lacking a dataset / model card (P3-C).
- **Release-gate.** Block deployment of regulated-sector-data pipelines without attestation (P5-C).
- **Runtime monitor.** DSR SLA dashboard; redaction false-negative sampling (mod-104); DP accountant drift (mod-106).
- **Review-gate.** RFC template fields for lawful basis, DFD link, threat-model link, DPIA plan.

## Checkpoints and the DPIA

A DPIA (mod-111) under GDPR Art. 35(7)(d) requires "the measures envisaged to address the risks … including safeguards, security measures and mechanisms". The checkpoint register — with status, enforcement, and evidence — is the direct source. Keep it synchronised with the LINDDUN / PRAM registers from chapters 03–06.

## A short worked example

Suppose the LINDDUN register (chapter 06) has threat T-014 — semantic-cache cross-tenant return. Routing (chapter 07) assigns tenant-scoped cache key + negative test. The PbD checkpoints that *enforce* this control:

- **P2-C** — retrieval / cache scope defaults to per-tenant. Cache config without `tenant_scope: true` fails CI.
- **P3-D** — API spec documents cache behaviour and cache-key composition.
- **P5-B** — cache-admin role is a separate ACL; access audited.
- **P7-A** — DSR erasure reaches the cache.
- **Evidence** — cache-config CI run, integration test `test_cache_cross_tenant_negative`, access log sample.

The LINDDUN model describes the threat; the routing names the control; the PbD checkpoint gates the delivery. The three together are what prevents the threat from re-appearing.

## Summary — what you can do after this chapter

- Convert each of Cavoukian's seven PbD principles into one or more enforceable engineering checkpoints.
- Encode the checkpoints as CI lints, config schema, release gates, runtime monitors, or review-gate fields.
- Produce a checkpoint register with enforcement and evidence columns.
- Link a LINDDUN threat → routed control → PbD checkpoint → test / evidence as a single chain.
- Feed the checkpoint register into a DPIA without rework.

The next and final chapter draws the sharp boundary between this role's privacy threat-modelling scope and `ai-risk-engineer`'s broader harm-model scope.
