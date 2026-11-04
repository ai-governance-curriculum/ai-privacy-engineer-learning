# Prompt and Completion Logging — What to Log, What to Redact, How Long to Keep It, and How to Erase

Every LLM-serving system logs prompts and completions, in some form, somewhere. The question is never "should we log?" — the ops team needs it to debug, the safety team needs it to audit, the trust-and-safety team needs it to investigate abuse, the model team needs it to evaluate quality. The question is *what* is logged, *where*, with *what redactions in place before it is written*, *for how long*, *who can read it*, and *how an erasure request against it is honoured*. Getting all five answers wrong at once is how a RAG deployment ends up in a data-protection news cycle.

This chapter is the logging-policy counterpart to chapter 01's retrieval-isolation and chapter 02's test harness. The design artefacts: a *log-schema spec* that names each field and its sensitivity class, a *retention schedule* that each class maps to, an *access matrix* that says who can read each class, and an *erasure path* that can delete specific records on request.

Primary references:

- Microsoft — *Presidio* documentation. [microsoft.github.io/presidio/](https://microsoft.github.io/presidio/) — the recognisers, the anonymizer, the structured outputs that chapter 04 of the previous module (104) covered in depth. We use the hot-path integration here.
- Skyflow — *Data Privacy Vault* reference architecture and open papers. [skyflow.com/post/what-is-a-data-privacy-vault](https://www.skyflow.com/post/what-is-a-data-privacy-vault) — the canonical reference for the vault pattern. Also see Mitchell's seminal design discussion at [skyflow.com/post/the-case-for-a-data-privacy-vault](https://www.skyflow.com/post/the-case-for-a-data-privacy-vault).
- OpenAI / Anthropic / Google — provider-specific *API data-usage* policies. [openai.com/policies/business-terms/](https://openai.com/policies/business-terms/), [anthropic.com/legal/privacy](https://www.anthropic.com/legal/privacy), [cloud.google.com/vertex-ai/docs/generative-ai/data-governance](https://cloud.google.com/vertex-ai/docs/generative-ai/data-governance) — the "what does the provider log and retain" question is a direct input into your own logging claim.
- UK ICO — *Guidance on AI and data protection*. [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/) — concrete UK regulator expectations on logging and retention for AI systems.
- Article 17 of the GDPR (right to erasure). [gdpr-info.eu/art-17-gdpr/](https://gdpr-info.eu/art-17-gdpr/) — the legal anchor for the erasure path.
- NIST — *SP 800-53 Rev. 5*, AU control family (audit and accountability). [doi.org/10.6028/NIST.SP.800-53r5](https://doi.org/10.6028/NIST.SP.800-53r5) — the FedRAMP / government-audit anchor for what "good enough" logging looks like.

## Why "log less, more carefully" rather than "log everything, encrypt it"

The strong temptation is to keep every prompt and every completion encrypted at rest and call that privacy-preserving. It is not. Three reasons:

- **Encryption-at-rest is a key-management discipline, not a privacy guarantee.** Everyone who can read the key can read the log. The scope of that set tends to be larger than the DPIA assumes.
- **An encrypted log is still subject to erasure requests.** GDPR Article 17 does not care whether the data is encrypted — it cares whether you can honour an erasure. If erasure means decrypting the log, finding the record, and re-encrypting without it, you need the plaintext path to exist. If erasure means dropping the key for the window the record is in, you lose every other record in that window too.
- **An encrypted log is still a liability.** A subpoena, a discovery request, a security incident that forces disclosure, a regulator audit — each of these can compel you to produce plaintext from an encrypted log. The best defence is the record never existing.

The operating posture is therefore: *log the minimum necessary to satisfy each operational need, with PII redacted on the hot path before anything is written to a durable store, and with a structured plan for honouring erasure against what does get written*.

## What each stakeholder actually needs

Name the stakeholders explicitly so the log schema can be derived from their needs.

- **SRE / on-call.** Needs enough to debug a failing request: timestamps, latency, status, the model identity, the retriever's index version, the top-k chunk IDs, error classes. Does *not* need the prompt text to resolve "the model is returning 500s".
- **Safety / trust-and-safety.** Needs enough to investigate reported harms: the user ID or session ID (indirect identifier, see below), the time window, the generated content, and the retrieved context that fed it. The prompt text is often required for a full investigation; this is the stakeholder whose needs push hardest on the "don't log prompts" default.
- **Model / evals team.** Needs enough to compute offline evaluations: a representative *sample* of prompts and completions, optionally with labels and ratings. The sample is drawn with user consent (or a lawful-basis equivalent) and is usually a tiny fraction of traffic.
- **Product / analytics.** Needs aggregates: query volume by feature, latency distribution, retrieval top-k distribution. Can almost always be served from pre-aggregated metrics without storing raw prompts.
- **Legal / compliance.** Needs the *existence* of logs sufficient to defend the service in disputes and to prove compliance claims. Usually does not need the prompt text; usually needs retention proof.
- **Billing / rate-limiting.** Needs counts of requests per user, tokens per request. No content.

The sensitivity hierarchy that follows is natural: *counts* and *latencies* at the bottom; *retrieval IDs and model/settings* above; *redacted summaries* above that; *raw prompts and completions* at the top, with the smallest audience and the shortest retention.

## The log schema spec

A single row in the log store, with field classifications:

```yaml
# Example log row — one per LLM invocation
request_id:        <uuid>           # class: identifier, pseudonymous
timestamp:         <iso8601>        # class: identifier
tenant_id:         <uuid>           # class: identifier
user_id:           <hashed>         # class: identifier, pseudonymous
session_id:        <hashed>         # class: identifier, pseudonymous
model:             "claude-opus-4-7"# class: config
retriever_version: "idx-v142"       # class: config
top_k_chunk_ids:   [c1, c2, ...]    # class: config (joinable → sensitive)
latency_ms:        1234             # class: ops
tokens_prompt:     512              # class: ops
tokens_completion: 256              # class: ops
status:            "ok"             # class: ops
error_class:       null             # class: ops
prompt_redacted:   "<redacted>"     # class: content, redacted (see §redaction)
completion_redacted: "<redacted>"   # class: content, redacted
pii_detected:      ["EMAIL", "PHONE"] # class: content-metadata
vault_token_prompt:   vault-tok-...  # class: pointer (see §vault)
vault_token_completion: vault-tok-...# class: pointer
flags:             {abuse_report: false, ...} # class: ops
```

Five field classes, each with its own retention, access, and erasure answer:

- **identifier** — IDs that are either direct or pseudonymous references to people or tenants. Retention: as short as feasible for operations (90 days is a common default for structured operational logs). Access: ops, safety, legal on investigation. Erasure: directly addressable.
- **config** — model name, retriever version, chunk IDs. Retention: long enough to correlate with eval runs (365 days is typical). Access: wider; this is not personal data in isolation. Erasure: not required for this class on its own, but the config row for a user's request *is* joinable, so follows the identifier's erasure.
- **ops** — latencies, token counts, status. Retention: aggregated metrics forever, raw rows 90 days. Access: ops. Erasure: not required at the raw level because the row is not individually identifying once separated from identifier fields.
- **content** — prompt and completion text. Retention: *the shortest you can defend operationally* — often 7 or 14 days in a hot store for abuse investigations; nothing permanent at this tier. Access: minimal — safety team on named incidents, with just-in-time approval. Erasure: must be addressable per-user.
- **content-metadata** — the output of PII detection, the labels of what was present without the text. Retention: longer than content; this is derived, aggregatable. Access: trust-and-safety analytics, model team for eval sampling. Erasure: follows the user's erasure request.
- **pointer** — an opaque token that redeems (via the vault, below) to the original content. The log itself does not contain the content; the vault does. Erasure: the pointer is useless once the vault record is deleted.

The schema does not need to be literal YAML; this is the design vocabulary. The point is: *every field has a class and every class has an answer to the retention / access / erasure questions*.

## Redact before log — the hot-path pipeline

The principle: anything written to persistent storage must have passed through a redaction stage. If the redactor is skipped, the writer fails. Module 104 covers Presidio and the broader PII-pipeline toolkit in depth; we use the hot-path integration here.

A minimal pipeline in code:

```python
from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine
from presidio_anonymizer.entities import OperatorConfig

analyzer = AnalyzerEngine()      # built once, process-level
anonymizer = AnonymizerEngine()

OPERATORS = {
    "EMAIL_ADDRESS":   OperatorConfig("replace", {"new_value": "<EMAIL>"}),
    "PHONE_NUMBER":    OperatorConfig("replace", {"new_value": "<PHONE>"}),
    "PERSON":          OperatorConfig("replace", {"new_value": "<PERSON>"}),
    "US_SSN":          OperatorConfig("replace", {"new_value": "<SSN>"}),
    "CREDIT_CARD":     OperatorConfig("replace", {"new_value": "<CC>"}),
    # add tenant-specific recognisers here
}

def redact_for_log(text: str, language: str = "en") -> tuple[str, list[str]]:
    results = analyzer.analyze(text=text, language=language)
    detected_types = sorted({r.entity_type for r in results})
    anonymised = anonymizer.anonymize(
        text=text, analyzer_results=results, operators=OPERATORS
    )
    return anonymised.text, detected_types
```

This is the function the logging middleware calls before writing `prompt_redacted` and `completion_redacted`. The detected types populate `pii_detected`.

Four design properties of a defensible hot-path redactor:

1. **Deterministic configuration.** The recogniser set, the operators, and the language configuration are code, under version control, with a test suite. A regression in Presidio's PhoneNumberRecogniser is a code-visible change, not a configuration drift.
2. **Latency budget.** Presidio-on-CPU, sensibly configured, handles a few-thousand-character prompt in low tens of milliseconds. Under a strict latency budget (streaming RAG responses in a chat UX), the redaction runs *after* the completion is produced but *before* the log is written — the hot path serves the user first, then redacts in a non-blocking stage before persistence. The prompt redaction can either run before dispatch (safer but adds to time-to-first-token) or run in parallel with the generator call (tolerable because the raw prompt is not yet persistent).
3. **Fail-closed on redactor error.** If the redactor throws, the log row is not written — or is written with the content fields marked as `"redaction_failed"` and the raw text dropped. The alternative (log the raw text when the redactor fails) is how an incident begins.
4. **Tenant-aware recognisers.** Tenants in regulated verticals need extra recognisers (patient identifiers in a medical context, case numbers in a legal context, order IDs in a retail context). The pipeline supports registering tenant-scoped recognisers so the hot path picks up the right set.

### What redaction is *not*

Presidio is a recogniser-and-anonymiser, not an invisibility machine. The text "the patient with the appointment at 3pm yesterday in the cardiac unit, next to Dr. Chen's office" has no SSN, no phone number, and no named person other than Dr. Chen — and may still be identifying in context. Redaction before log reduces *direct* PII; the retention-and-access controls below handle the residual re-identification risk. The system-card claim (chapter 06) phrases this precisely: *"Logs are passed through Presidio with recogniser set {X}, which removes the following direct identifier classes before write. Residual re-identification risk via indirect identifiers is managed by the retention and access-control claims below."*

## Data-privacy vault for post-hoc analytics

The vault pattern (Skyflow is the reference commercial implementation; open-source implementations exist) separates the identifying data from the operational systems. The log row stores a *token* that redeems in the vault to the original content. Code that wants to analyse prompts calls the vault; the vault enforces its own access control.

Why this is useful for prompt / completion logs specifically:

- **Access gating lives in one place.** The vault enforces who can detokenize. The operational log is accessible to many systems; the vault is accessible to a narrow set.
- **Erasure is a single operation.** Deleting the vault record invalidates the token everywhere. There is no "find all the places we wrote this prompt" problem.
- **Residency is a vault property.** The vault can enforce "these records are stored only in the EU" independently of where the log fabric lives.
- **Format-preserving tokenisation is available.** For structured fields (email, phone), a format-preserving token can flow through analytics systems without special handling; for free-form prompts, an opaque UUID token is the norm.

A minimal architecture:

```
request → LLM serving → redactor → log store (holds vault tokens only)
                                 → vault (holds raw prompt + completion)
                                         ← analyst query (authenticated,
                                                          logged, scoped,
                                                          detokenizes k rows)
```

Three design choices the privacy engineer owns:

- **What goes into the vault.** Only content fields, not identifiers or ops. Identifiers live in the operational log with their own retention.
- **Who can detokenize, and how.** A small, named group; MFA + role-based access; every detokenization is itself audit-logged. "View N raw prompts for incident I" is an API call with a case-ID parameter; a bulk detokenisation is a separate privileged operation with a second-party approval.
- **Retention inside the vault.** Can be shorter than in the operational log. A common pattern: operational log at 90 days, vault content at 14 days. Analytics work that needs a longer window uses pre-aggregated or redacted summaries, not raw prompts.

## Retention schedule

An example schedule for a RAG-serving system. Adjust for your regulated-vertical constraints.

| Field class            | Hot store (queryable)   | Cold store (archive)       | Vault (detokenisable)  |
| ---                    | ---                     | ---                        | ---                    |
| identifier             | 90 days                 | —                          | n/a                    |
| config                 | 365 days                | 7 years (aggregated)       | n/a                    |
| ops                    | 90 days (raw)           | 7 years (aggregated)       | n/a                    |
| content-redacted       | 30 days                 | —                          | n/a                    |
| content-metadata       | 180 days                | —                          | n/a                    |
| content-raw (vault)    | n/a                     | n/a                        | 14 days                |

Two operational properties of the schedule:

- **Retention is enforced by the store, not the application.** Hot-store indexes have TTLs; cold-store objects have lifecycle policies that transition and then delete; vault records have expiry. A log row that outlives its class's retention is an operational bug worth paging on.
- **Aggregates survive the raw.** Aggregate metrics (QPS, latency percentiles) that are computed from raw rows persist on their own retention; the raw rows that fed them do not.

## Access matrix

One row per class, columns for role. Granted means "can read the class via a documented, logged path".

| Class                | SRE | Safety | T&S analyst | Model team | Legal | Billing |
| ---                  | --- | ---    | ---         | ---        | ---   | ---     |
| identifier           | Y   | Y      | Y           | Y (sample) | Y     | Y       |
| config               | Y   | Y      | Y           | Y          | Y     | —       |
| ops                  | Y   | —      | —           | Y          | —     | Y       |
| content-redacted     | —   | Y      | Y           | Y (sample) | —     | —       |
| content-metadata     | —   | Y      | Y           | Y          | —     | —       |
| content-raw (vault)  | —   | Y (on incident) | — | Y (sample, consent)| Y (legal process) | — |

Three disciplines around the matrix:

- **Roles are tagged in the IAM system with the exact class name.** No reviewer should have to ask "what counts as content-raw access".
- **Every access to `content-raw` is logged in a separate audit stream** with request_id, caller, case ID, and purpose. The audit stream is itself subject to the retention-and-erasure disciplines on logs.
- **Just-in-time elevation** for `content-raw`. Default is no access; a case-ID and a second-party approval grant access for a bounded time window.

## Honouring an erasure request against logs

The user requests deletion of their data. The logs contain their content. The erasure path has three parts.

### Part 1 — Inventory

For a given user_id, enumerate every log row and every vault record that pertains to them. Because the user_id (or its hash) is a stored field, this is a query against the operational log store and the vault.

### Part 2 — Vault purge

Delete the vault records. Every token in the log that pointed to those records now dangles — the token is still there, but detokenisation returns a "deleted" marker. This is the most important part because the vault is where the plaintext lives.

### Part 3 — Operational log scrub

In the operational log, replace the identifier fields (user_id hash, session_id hash, possibly tenant_id if the user's departure implies the tenant's) with a well-known "deleted-user" sentinel. The redacted content fields remain; they have no PII in them (by the redactor's claim); the row stays for aggregate statistics but is no longer attributable. The dangling vault token is replaced with null.

Some deployments go further and delete the entire row. Both are valid; the choice is a legal one (does your lawful basis for aggregates require per-user attribution?) that the DPO should answer. Document the choice in the system card.

### Deadline

GDPR Article 17 says "without undue delay". Regulators have started naming concrete expectations — typically 30 days for an erasure action on live systems, with longer windows acceptable for backup media so long as the policy ensures deleted data is not restored. A reasonable internal target: *24 hours* for the vault purge, *7 days* for the operational-log scrub, *90 days* for the backup / archive catch-up. The system card names the targets explicitly and includes the current SLO attainment.

### Backups

Backups are the hard case. A naive backup includes the vault records and the operational logs; restoring from such a backup resurrects deleted content. The pattern that scales: include the erasure-list with every backup, and have the restore procedure replay the erasure-list before the restored system is made available. The DPO-facing claim is "our restore procedure guarantees erasure requests that predate the restore are honoured within {N} hours of the restore completing".

## Logs of logs — the audit stream

A separate audit stream records every access to the content classes (especially `content-raw`). It is append-only; it is readable only by a small set of incident-response and legal roles; it is itself retained on a longer schedule than the data it audits (because it is the evidence base for the compliance claim).

The audit row has a minimal, non-controversial schema: who, what, when, why. "Who" is the IAM identity of the caller; "what" is the class accessed and a pointer to the record; "when" is a timestamp; "why" is a case ID or purpose code. The vault's detokenisation API emits one such row per call.

## Provider-logging transparency

The RAG deployment may call out to a model-provider API (OpenAI, Anthropic, Google, a self-hosted model). The provider has its own logging and retention policy. The system card's logging section must *name* the provider's retention commitment (is it 30 days? zero-retention? conditional on a enterprise-tier contract?) and bind the claim to a specific contract or policy URL. A deployment that redacts aggressively before sending to the provider is harder to compromise via the provider's logs; a deployment that assumes the provider does not log is overclaiming.

The specific pattern to look for and document:

- **Zero-retention claim.** Some providers offer zero-retention on enterprise tiers. The contract language matters; link to the exact term.
- **Opt-out of training.** Some providers train on API traffic by default; opt-out is a contract or a dashboard setting. State the opt-out status in the system card.
- **Regional residency.** Provider APIs can be routed to specific regions (EU-only, US-only). The residency of the log, even if zero-retention, affects compliance claims.

## Summary — what you can do after this chapter

- Enumerate the five log-field classes (identifier, config, ops, content-redacted, content-metadata, pointer) and give each a retention, access, and erasure answer.
- Specify a redact-before-log pipeline using Presidio: operator config, deterministic version control, fail-closed on redactor error, tenant-aware recognisers, where it sits in the request flow.
- Design the data-privacy-vault integration: what goes in, who detokenizes, how retention composes with the operational log.
- Build a retention schedule and name the enforcement mechanism (TTL, lifecycle policy, vault expiry) for each row.
- Build an access matrix and bind it to IAM roles, with just-in-time elevation for raw-content access.
- Handle an erasure request end-to-end — inventory, vault purge, operational-log scrub, backup replay — with an internal SLO.
- Maintain an audit stream for content-class accesses with its own retention and access discipline.
- Fold the provider's logging and retention commitments into the system-card claim.

Chapter 04 moves from logging to the next trust surface: agents that call tools on behalf of users, and the identity-propagation story that lets a tool-invocation be scoped to the exact user and the exact read / write it needs.
