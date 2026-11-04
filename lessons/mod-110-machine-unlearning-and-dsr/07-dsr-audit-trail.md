# The DSR Audit Trail — Packaging Evidence That Survives a Supervisory-Authority Inspection

A DSR response that the engineer and the `ai-governance-analyst` consider complete is only half-complete. The other half — the half the supervisory authority, the DPO, and the internal audit will look for — is the *audit trail*: a bundle of evidence that lets an outside reviewer verify the DSR was received, interpreted, actioned, verified, and communicated back to the subject, within the clock, with the correct engineering actions. Without the audit trail the engineering work is unauditable; with it, the regulator's cross-check becomes a desk review rather than an on-site investigation.

This chapter is the artefact-authoring discipline. It names the fields every DSR record must carry, the evidence each engineering action must capture, the retention and access policies that apply to the audit trail itself, and the format in which the bundle is produced for a supervisory-authority inspection. It ties the chapters of the module together — chapter 01's runbooks produce evidence; chapters 02–04's unlearning techniques produce verification artefacts; chapter 05's decision tree produces the leaf choice and its reasoning; chapter 06's cascade produces the deletion manifests. Chapter 07 is where they are packaged.

Primary references:

- Regulation (EU) 2016/679, *GDPR*, Articles 12 (transparent communication), 15 (access), 16 (rectification), 17 (erasure), 18 (restriction), 20 (portability), 21 (object), 22 (automated decisions), 30 (records of processing), 33 (breach notification), 77 (right to lodge a complaint). [eur-lex.europa.eu/eli/reg/2016/679/oj](https://eur-lex.europa.eu/eli/reg/2016/679/oj).
- European Data Protection Board — *Guidelines 01/2022 on data subject rights – Right of access*, v2.0 (2023). [edpb.europa.eu/system/files/2023-04/edpb_guidelines_202201_data_subject_rights_access_v2_en.pdf](https://www.edpb.europa.eu/system/files/2023-04/edpb_guidelines_202201_data_subject_rights_access_v2_en.pdf).
- UK ICO — *Accountability and governance* (Art. 24 / Art. 30 guidance). [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/accountability-and-governance/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/accountability-and-governance/).
- UK ICO — *Guide to the UK GDPR — Record keeping*. [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/documentation/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/documentation/).
- NIST — *SP 800-53 Rev. 5*, AU (Audit and Accountability) and PT (PII Processing and Transparency) families. [doi.org/10.6028/NIST.SP.800-53r5](https://doi.org/10.6028/NIST.SP.800-53r5).
- NIST — *Privacy Framework v1.0*, "Communicate-P" category. [nist.gov/privacy-framework](https://www.nist.gov/privacy-framework).
- ISO/IEC 27701 — *Security techniques — Extension to ISO/IEC 27001 and 27002 for privacy information management*. [iso.org/standard/71670.html](https://www.iso.org/standard/71670.html).
- ISO/IEC 29184 — *Online privacy notices and consent*. [iso.org/standard/70331.html](https://www.iso.org/standard/70331.html).

## What a supervisory authority will ask for

The supervisory authority (an EU DPA, the UK ICO, the CNIL, the Data Protection Commissioner, the equivalent in other jurisdictions) does not care about the elegance of your unlearning algorithm. They care about four things, in this order:

1. **Was the request actioned within the clock?** Date received, date completed, extension notice if applicable.
2. **Did the action match the request?** The subject asked for erasure of X; the engineering action erased X and not merely Y.
3. **Is there evidence that the action happened?** Not "we say it did"; "here is the log / manifest / hash that shows it did".
4. **Was the subject informed, and in plain language?** The response to the subject must be in their language (literal and figurative) and must describe what was done.

The audit trail is the artefact that answers these questions without requiring the auditor to interview an engineer or re-run a pipeline. Every field in this chapter maps to one of the four.

## The DSR record — the top-level schema

Each DSR produces one top-level record. In a real system this is a row in a dedicated DSR registry — a Postgres table, an enterprise system (OneTrust, TrustArc, a custom in-house platform), or a flat-file journal. The required fields:

| Field                       | Content                                                                                   |
|-----------------------------|-------------------------------------------------------------------------------------------|
| `request_id`                | Immutable identifier (UUID v4) assigned at intake.                                         |
| `received_at`               | Timestamp of receipt (ISO 8601, UTC).                                                      |
| `subject_identity`          | Subject identifiers, including verification method (see "Subject verification" below).      |
| `request_type`              | One of: access, rectification, erasure, restriction, portability, object, Art. 22 contest. |
| `request_scope`             | Subject-stated scope (verbatim) plus the analyst-interpreted scope (structured).            |
| `legal_basis`               | The Article (and the operative legal framework — GDPR, UK GDPR, CCPA/CPRA, LGPD, state).    |
| `clock_due_at`              | Date the response is due (`received_at` + 30 days under GDPR default).                      |
| `extension`                 | If taken: date requested, date notified, new due date.                                      |
| `analyst_owner`             | `ai-governance-analyst` responsible for the request.                                        |
| `engineer_owners`           | Engineers involved per system / corpus.                                                     |
| `decision_tree_leaf`        | Per-model: retrain, unlearn, no-action, wait-for-refresh. With reasoning reference.          |
| `engineering_actions`       | Structured list — per action, see "Per-action record" below.                                |
| `verification_artefacts`    | Hashes / references to probe outputs, MIA / TOFU reports, retrieval probes.                 |
| `response_sent_at`          | Timestamp the response was sent to the subject.                                             |
| `response_bundle_hash`      | Cryptographic hash of the response bundle sent.                                             |
| `status`                    | One of: received, verifying-identity, in-progress, completed, extended, denied, escalated.  |
| `closure_notes`             | Final notes from the analyst.                                                               |

The record is append-only at the audit level — changes are new rows referencing the `request_id`, not edits to the original. This gives the reviewer a clean history.

## Subject verification

GDPR Art. 12(6) permits (and in practice requires) identity verification before actioning a request, with the proportionality caveat that the verification cannot ask for excessive information. The audit trail records:

- Verification method (email challenge, logged-in-session, government-ID attestation, step-up auth).
- Verification artefact — the challenge-response token, the session ID, the attestation record.
- Verification timestamp.

A request whose identity could not be verified is closed with the specific reason and an audit entry; the subject is informed of the denial and the appeal path.

## Per-action record

Each engineering action — a deletion, a retrain, an unlearn run, a cache flush, a log purge — produces a structured sub-record attached to the DSR. The fields:

| Field                   | Content                                                                             |
|-------------------------|-------------------------------------------------------------------------------------|
| `action_id`             | UUID.                                                                               |
| `request_id`            | The parent DSR.                                                                     |
| `action_type`           | One of: db-delete, chunk-delete, embedding-delete, index-rebuild, cache-flush, log-purge, agent-memory-purge, model-unlearn, model-retrain, model-no-action, source-notification, verification-probe. |
| `target_system`         | The specific store / model (`postgresql://prod-main/users`, `model://product-v2`).   |
| `scope_predicate`       | The predicate applied (SQL where-clause, filter expression, forget-set hash).        |
| `records_affected`      | Count. For model actions: parameter-count-scope or ensemble-scope.                   |
| `started_at`, `finished_at` | Timestamps.                                                                     |
| `executor`              | Human user / service account / scheduled job that ran the action.                    |
| `config_hash`           | For unlearn / retrain: hash of the training / unlearning config.                     |
| `pre_state_hash`        | Hash of the pre-action state (weight hash, chunk-count manifest hash).               |
| `post_state_hash`       | Hash of the post-action state.                                                      |
| `verification_result`   | Reference to the verification artefact (probe transcript, MIA / TOFU numbers).       |
| `evidence_uri`          | Blob storage URI of the detailed evidence (SQL execution plan, index-rebuild log).   |

The hashes are the regulator-facing evidence: "the weights after the deletion are hash H2; the weights before were H1; H1 ≠ H2; the training manifest changed from M1 to M2 in a defined way". Without hashes, the audit trail is testimony; with them, it is evidence.

## The decision-tree reasoning trace

Chapter 05's decision tree is walked for each affected model. The audit trail records:

- Per-question evidence (Q1: was the record in training — reference to corpus-search output; Q2: did the model memorise — reference to probe output; Q3: refresh schedule — reference to roadmap commitment; Q4: method-feasibility — reference to system-card entry; Q5: clock — reference to wall-clock estimate; Q6: retrain-required — DPO sign-off or auto-rule).
- Per-question answer with timestamp.
- Leaf reached.
- Any override (a case where the engineer or DPO chose a leaf other than the one the tree indicated, with reasoning).

This is the artefact that answers a regulator's "why no action on the weights?" — the probe was run, it produced baseline-like output, and the reasoning is traceable step by step.

## Verification artefacts — by action class

Each engineering action class has a specific verification artefact that must land in the trail.

### Chunk / embedding deletion

- The verification probe query, as text.
- The post-deletion retrieval result on the probe (confirming zero subject-content).
- The retrieval caller used (tenant, role).
- Timestamp.

### Index rebuild

- Pre-rebuild tombstone count.
- Post-rebuild live-vector count.
- Rebuild wall-clock.
- Rebuild job ID.

### Cache invalidation

- Cache keys invalidated (count and sample).
- Invalidation primitive used.
- Post-invalidation read-through verification.

### Log purge

- Pre-purge query count matching the subject predicate.
- Post-purge query count (zero or the retention-naturally-purged count).
- Purge job ID.

### Agent-memory purge

- Memory-store identifiers covered.
- Per-store deletion count.
- Verification queries, as text + result.

### Model retrain

- Pre-retrain corpus manifest hash.
- Post-retrain corpus manifest hash.
- Pre-retrain model weight hash.
- Post-retrain model weight hash.
- Eval-suite results (accuracy, MIA on the subject's records, extraction probe on known content).
- Training wall-clock, GPU-hours.

### Model unlearn (SISA)

- Shard and slice identified.
- Checkpoint retrained from.
- Pre-unlearn shard-model hash.
- Post-unlearn shard-model hash.
- Ensemble-level eval delta.
- Determinism-profile lock (chapter 02).

### Model unlearn (approximate / certified-removal)

- Pre-unlearn weight hash.
- Post-unlearn weight hash.
- Influence-function config (LiSSA parameters, HVP step count).
- (ε, δ) consumed if certified-removal.
- MIA verification: AUC + 95% CI, pre vs. post.
- Canary extraction rate if applicable.

### Model unlearn (LLM)

- Method (gradient ascent / task vector / Who's-Harry-Potter).
- Config hash (LR, steps, KL beta, α).
- Forget-set hash.
- Retain-set hash.
- TOFU result pre vs. post (Forget Quality, Model Utility, MIA, Extraction).
- Content-specific extraction probe transcript.
- Utility delta on general benchmark suite.

### Model no-action

- Not-in-training evidence (corpus-search output).
- Not-memorised evidence (probe output + baseline comparison).
- De-duplication / aggregation evidence (if applicable).
- Analyst / DPO sign-off on the no-action determination.

### Source notification (Art. 17(2))

- List of downstream controllers / processors notified.
- Notification channel and timestamp per recipient.
- Acknowledgement artefact (where available).

## Response to the subject

The response the subject receives is itself an audit-trail artefact. The fields:

- *Request summary.* What the subject asked, verbatim and in the system's words.
- *Scope actioned.* What was done, in plain language.
- *What was not done and why.* If a right was partially or fully denied — Art. 17(3) exemption, legitimate-interests outweighing, legal-obligation retention — the reasoning and the Article reference.
- *The data (for access and portability responses).* Exported in a structured format.
- *The appeal path.* The subject's right to lodge a complaint with the supervisory authority (GDPR Art. 77), with the authority's contact details for the subject's jurisdiction.
- *Who signed off.* The DPO contact.

The response is sent through the channel the subject used or an equivalent secure channel. A hash of the response bundle is retained in the audit trail; the bundle itself is retained for the challenge window (typically the longest of: supervisory authority's statute-of-limitations for the right in question, any contractual retention, any legal-hold).

## Retention and access to the audit trail itself

The audit trail is personal data in its own right (it references the subject). It must be retained long enough to serve its purpose — the GDPR challenge window — and no longer. Typical retention:

- *DSR record and action logs*: retained for the duration of any statute-of-limitations for a claim by the subject (often 3–6 years, per jurisdiction) or any regulatory-reporting lifecycle, then purged.
- *Evidence blobs (probe transcripts, eval reports)*: retained alongside the DSR record.
- *Model weight hashes*: retained with the model's own lifecycle record (even after the model is decommissioned, the audit-trail record is kept with the hash).

Access to the audit trail is scoped: the DPO, the `ai-governance-analyst` team, the privacy-engineering leads, the internal audit function. A subject's own right of access covers their own audit-trail record *about themselves* but not the full-system trail (personal data about other subjects is not included).

## The supervisory-authority inspection bundle

When a supervisory authority opens an investigation (routine audit, complaint-driven, breach-driven), they typically request a bundle of artefacts. Prepare the bundle format in advance so the response is minutes, not weeks:

- *RoPA entries* (Art. 30 records of processing) for the AI systems in scope.
- *DSR register* with all requests in the inspection window, with status and outcome.
- *Sample DSR audit trails* — full detail per DSR for a selection the authority chooses (often random, sometimes complaint-driven).
- *Model system cards* (incorporating chapter 05's Erasure Posture).
- *DPIA artefacts* (mod-111).
- *Policies* — the DSR policy, the data-retention policy, the AI-system-specific privacy policy.

The bundle format: zip of folders, one folder per artefact, with a top-level index and cryptographic signatures. Chapter 09 of mod-106 (DP Release Package) uses the same pattern for a different artefact; this chapter borrows the pattern.

## Operational notes — the engineer-facing contribution

The engineer does not typically own the DSR register — that is the DPO / `ai-governance-analyst` / compliance-tooling domain. The engineer owns the *inputs* to the register from the engineering systems:

- A *DSR intake webhook* that the analyst system calls when a new request arrives, triggering the engineering workflow (ticket creation, decision-tree walk, action execution, verification).
- A *DSR evidence API* that the engineering system writes to as each action completes, pushing hashes, timestamps, and evidence blob references into the register.
- A *verification runner* that executes the probe for each completed DSR and writes the result.

A well-designed DSR plumbing has the register as the system of record, with the engineering systems as producers; a badly-designed one has a human engineer copy-pasting hashes into a spreadsheet. Build the plumbing early; the first supervisory inspection is a bad time to discover you lack it.

## The DSR SLA dashboard

Operationalising "within the clock" requires visibility. The DSR register's companion is a dashboard that shows:

- Open DSRs by age and request type.
- SLA at-risk (approaching the 30-day window).
- Completion rate and average wall-clock by request type.
- Verification pass / fail rate.
- Extension rate.

The dashboard is a management-level artefact; the DPO and `ai-governance-analyst` consume it. The engineering lead uses it to size the team's capacity — a sustained month-on-month increase in DSRs is a signal to invest in SISA or approximate-unlearning plumbing before the SLA breaks.

## Common failure modes — the review checklist

Walk this list before a supervisory inspection or an internal DPO audit:

- A DSR record with no `response_sent_at` and no explicit status — a lost request.
- An erasure DSR with chunk-deletion actions but no embedding-deletion actions — incomplete cascade.
- A no-action decision with no probe evidence — undefended.
- A retrain action with no pre/post weight hash — unverifiable.
- An unlearn action with no verification artefact — unverifiable.
- A response sent to the subject in jargon ("we ran the certified-removal procedure") rather than plain language.
- An audit-trail retention that outlives its purpose — a privacy violation on the trail itself.
- An action executed by a service account with no clear human accountability — ownership gap.
- A per-DSR extension taken without the pre-notification to the subject required by Art. 12(3).
- Backup retention window not disclosed to the subject in the response.

Each of these is a finding a reviewer will record. Fix them before the review, not during.

## Breach-notification adjacency

Art. 33 requires a breach to be notified within 72 hours. A DSR that cannot be completed on time (the clock expires without a complete response), *if it results in a data-protection risk*, is itself potentially a notifiable incident — the DPO and `ai-governance-analyst` make the determination, not the engineer. The engineer's input is to surface the failure early (via the dashboard above) so the determination can be made inside the 72-hour window.

A DSR that is missed is also a reportable item to the internal privacy committee and to the board where applicable; the audit trail's closure notes capture this.

## The DSR audit-trail template — fields that must appear

For reviewers and auditors, the DSR audit trail is primarily a schema. The fields that must appear, pulled together:

- *Identity*: `request_id`, `received_at`, `subject_identity`, `verification_method`, `verification_artefact`.
- *Scope and interpretation*: `request_type`, `legal_basis`, `request_scope` (verbatim and interpreted), `analyst_owner`.
- *Clock*: `clock_due_at`, `extension`, `response_sent_at`.
- *Decision tree*: `decision_tree_leaf` per affected model, with per-question evidence references.
- *Actions*: list of per-action records with their own hashes and evidence.
- *Verification*: per-action verification artefacts including probe outputs, MIA / TOFU numbers, retrieval-probe transcripts.
- *Response*: `response_bundle_hash`, `response_channel`, `response_language`.
- *Closure*: `closure_notes`, `status`.

Any missing field is a gap a reviewer will find. Build the schema before you ship the first DSR runbook.

## Summary — what you can do after this chapter

- Author the top-level DSR record schema and the per-action sub-record schema, filling all required fields.
- Produce the decision-tree reasoning trace for a specific DSR, with per-question evidence.
- Collect the per-action verification artefacts — probe transcripts, weight hashes, MIA / TOFU results, retrieval probes — for every action class.
- Draft the subject response in plain language, with the scope-actioned / scope-not-actioned / appeal-path fields.
- Build the DSR intake webhook, the evidence API, and the verification runner so the audit trail is populated as a side-effect of the engineering workflow, not a manual spreadsheet.
- Prepare the supervisory-authority inspection bundle: RoPA entries, DSR register, sample full-detail trails, system cards, DPIA artefacts, policies.
- Run the common-failure-mode checklist against your DSR register before an audit.

With chapter 07 the module closes. A complete DSR-ready AI system:

- Knows the six rights and maps each to a per-store runbook (chapter 01).
- Has an unlearning mechanism — SISA, approximate, LLM — in place or decided against (chapters 02–04).
- Walks the retrain-vs-unlearn-vs-no-action decision tree for every Art. 17 request (chapter 05).
- Executes the RAG / agent erasure cascade for every corpus-side request (chapter 06).
- Packages the audit trail for every request such that a supervisory authority can verify the response without a desk visit (chapter 07).

The downstream module (mod-111) makes the DPIA from these artefacts; the upstream module (mod-109) makes sure the RAG system has the subject-ID propagation this module's runbooks depend on. The privacy engineer's seat is in the middle of this chain — delivering the operational control that makes the regulator-facing claim defensible.
