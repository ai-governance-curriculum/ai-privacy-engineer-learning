# exercise-03: Prompt and Completion Logging Policy

**Estimated effort:** 2 hours

## Objective

Author a prompt-and-completion logging policy document for a given RAG deployment, mapping each log field to a sensitivity class with retention, access, and erasure answers. The deliverable is a policy a DPO and a security officer can sign off on and an operations team can implement from.

## Scenario

You are the privacy engineer on the same B2B SaaS from exercise 01. The team is adding a RAG-answered support experience and needs a logging story before launch. The operations team wants to debug failing requests; the safety team wants to investigate reported misuse; the model team wants a sample for offline evals; the legal team wants defensible retention; the DPO wants an erasure path that honours GDPR Article 17. You have to produce a single policy document that satisfies all five stakeholders without logging more than is necessary.

The deployment calls out to a hosted LLM provider (pick one: Anthropic, OpenAI, or Google Vertex AI) for generation. Your operational logs live in Elasticsearch / OpenSearch with 90-day indexes; your optional data-privacy vault is Skyflow (or a self-hosted equivalent). The retention and access claims in the policy must map to configurable mechanisms in these systems.

## Requirements

1. **Stakeholder inventory.** For each of the five stakeholders above (plus any additional ones specific to your scenario — billing, trust-and-safety analytics, etc.), name the minimum information they need. Explicitly rule out what they do *not* need.
2. **Log-schema spec.** Write a complete log-row schema in YAML or JSON Schema. Classify each field as `identifier`, `config`, `ops`, `content-redacted`, `content-metadata`, or `pointer`. No field in the log row stores raw prompt or completion text.
3. **Retention schedule.** For each class, specify a retention period, the enforcement mechanism (index TTL, lifecycle policy, vault expiry), and the justification. Produce the retention table from chapter 03 with real numbers that your scenario can defend.
4. **Access matrix.** For each class × role pair, specify granted / not-granted and the elevation protocol where applicable. Include just-in-time elevation for raw content access with a second-party approval.
5. **Erasure path.** Specify the end-to-end procedure for honouring a GDPR Article 17 request: inventory, vault purge, operational-log scrub, backup replay. Include internal SLOs for each phase and the audit trail.
6. **Provider claim.** Write one paragraph that quotes the chosen LLM provider's retention commitment with a URL, names the contract term that pins it, and states whether training on API traffic is on or off for your deployment.
7. **Audit stream.** Specify the shape, retention, and access of the audit log that records every access to `content-raw` and every detokenisation call.
8. **System-card draft claim.** Produce the 5.1–5.6 subsections in the shape chapter 06 described, worded for a customer's privacy officer to read.

## Deliverables

Place artefacts under `exercises/exercise-03-prompt-and-completion-logging-policy-work/`:

- `policy.md` — the authored policy document (requirements 1–7).
- `schema/log_row.yaml` — the log-row schema.
- `schema/retention.yaml` — the retention schedule as data.
- `schema/access_matrix.yaml` — the access matrix as data.
- `runbook/erasure.md` — the step-by-step erasure runbook.
- `system_card_draft.md` — the system-card claim (requirement 8).
- `README.md` — a one-paragraph summary and a pointer to where each decision is made.

## Starter guidance

- The policy document itself should run three to six pages. The schemas, runbook, and system-card-draft live as separate structured artefacts.
- Challenge every "log the prompt" decision. The default is "no"; a "yes" requires a justification tied to a specific stakeholder's need, a retention short enough to respect storage-limitation, and an access gate narrow enough to satisfy the DPO.
- For the vault vs no-vault decision, both are defensible. If you choose no vault, the operational logs must carry only redacted content; abuse investigation uses the redacted content plus the request-trace context. If you choose a vault, the operational logs carry only tokens.
- Draft the erasure SLO against the industry reasonable: 24h for vault, 7d for operational log, 90d for backup replay. If your scenario needs tighter, justify.
- The chosen provider's retention policy changes materially. Quote the current page; don't rely on memory.
- Do not quote metrics or SOC-2 claims about the provider that you cannot verify from a public page.

## Acceptance criteria

- Every log-row field has a class and every class has retention / access / erasure answers.
- Retention periods are concrete numbers (not "appropriate" or "reasonable"), each mapped to an enforcement mechanism that exists in the chosen stores.
- The access matrix lists each role × class cell with granted / not-granted and the elevation protocol where applicable.
- The erasure runbook can be executed as a numbered procedure; a reader could follow it without additional context.
- The provider-logging paragraph quotes a specific URL, policy term, and training-on-API-traffic status.
- The system-card draft claim is written in the voice of chapter 06 (quantitative, boundary-aware, verifiable).
- A reviewer who reads only the system-card draft can form a correct mental model of what is logged and what is not.

## Stretch goals

- Implement the redact-before-log function as Python code and show it passes a small test (overlap with exercise 04 — you may choose to do this here or defer).
- Add a data-flow diagram of the logging path (request → redactor → log store, with optional vault) and name each boundary's trust owner.
- Write the DPIA companion paragraph that summarises the logging story in the shape mod-111 will consume.
- Extend the access matrix with a "customer self-service" column for a DSR portal that lets the customer's users see and delete their own logs without an internal ticket.
