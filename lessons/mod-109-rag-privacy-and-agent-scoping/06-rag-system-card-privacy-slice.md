# Authoring the Privacy Slice of a RAG / Agent System Card

A system card is the document that answers, in writing, "what does this system do and how does it handle data". For a RAG-and-agent deployment, the privacy slice is the half of the card that stands up in front of a customer's privacy officer, a regulator's technical consultant, and your own incident-response team on a bad day. It is the compact, honest, verifiable statement of the retrieval ACL, the context-window-leakage test posture, the logging policy, the agent-tool scoping, and the ingest-side defences you have built across chapters 01–05.

This chapter is the authoring guide. It takes the artefacts produced in the preceding chapters and threads them into a document that can be published (internally to the engineering organisation, externally to customers under NDA, or publicly if your product discloses system cards).

Primary references:

- Mitchell et al. — *Model Cards for Model Reporting*, FAccT 2019. [arxiv.org/abs/1810.03993](https://arxiv.org/abs/1810.03993) — the model-card paper; the system-card pattern is a generalisation.
- Gebru et al. — *Datasheets for Datasets*, Communications of the ACM 2021. [arxiv.org/abs/1803.09010](https://arxiv.org/abs/1803.09010) — the datasheet pattern for corpora, useful for the ingest slice.
- OpenAI — *GPT-4 System Card*. [openai.com/index/gpt-4-system-card/](https://openai.com/index/gpt-4-system-card/) — the publicly-published example to study for structure.
- Anthropic — *Claude 3 Model Card Addendum / Claude 3 Family model card*. [anthropic.com/news/claude-3-family](https://www.anthropic.com/news/claude-3-family) and successor updates — Anthropic's version of the same pattern.
- NIST — *AI RMF 1.0, Appendix A (map / measure / manage / govern functions)*. [nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework) — regulator-recognised framing the system card can map onto.
- EU — *AI Act, Articles 10 (data governance) and 13 (transparency)*. [artificialintelligenceact.eu/the-act/](https://artificialintelligenceact.eu/the-act/) — the legal frame the system card may need to speak to for EU deployments.

## Who the system card is for

Three readers, each of whom opens the document with a different question in mind.

- **The customer's privacy officer.** Preparing to accept your deployment inside their organisation. Reads to find the DPA-relevant claims: data flow, residency, retention, erasure, sub-processors.
- **The regulator's technical consultant.** Verifying an AI-Act / UK-AI / state-AI claim or responding to a complaint. Reads to find the measurable assertions (retention numbers, test-coverage percentages, specific controls) and the evidence that supports them.
- **Your own incident-response team.** Needs to find, in two minutes, "what did we say we do about this?" when a customer reports a potential leak. Reads to find the control they expected to be in place and whose state they can now verify.

The document therefore needs to be: *structured* (so each reader finds their section), *precise* (so the claims are checkable), and *honest* (so future-you does not get caught by a claim that was overstated today).

## Length and format

Internal system cards run 5–15 pages. External system cards tend to the shorter end. The privacy slice is roughly half of a complete system card — the other half covers model capabilities, safety, red-teaming, and performance. If you are writing only the privacy slice (this module's scope), it is 3–6 pages.

Prose and tables, not slide decks. The card is a record; a reader opens it in a PDF viewer or a `README.md` in a repo and skims. Use tables for the retention schedule, the access matrix, and the per-tool scope list; use prose for the threat model, the trust boundaries, and the residual-risk statement.

Put a version at the top and a changelog at the bottom. Readers trust a card with a version history more than one without.

## The structure

A nine-section outline for the privacy slice. Keep the order; readers scan by header.

```
1. Scope and version
2. Data flow diagram (one figure)
3. Retrieval access-control claims
4. Context-window-leakage test posture
5. Logging and retention
6. Agent-tool scoping
7. Corpus ingest and provenance
8. Erasure and takedown
9. Residual privacy risks and the boundary to other controls
```

Each section, in turn.

### Section 1 — Scope and version

One paragraph. Names the system, the deployment environment, and the version of the system-card document itself.

*Shape:*

> *This card describes the privacy-engineering posture of **{system name} v{N.M}**, deployed at **{environment}**, and specifically the RAG-and-agent slice of its data handling. It is version **{card-version}** dated **{YYYY-MM-DD}**. The document is maintained by **{team/role}**; questions about specific claims route to **{contact}**. A full changelog is at section 10. This card accompanies the DPIA (**{DPIA-reference}**) and is the authoritative statement of retrieval, logging, agent-tool, and ingest controls.*

### Section 2 — Data flow diagram

One figure, plus two or three paragraphs of caption.

The figure covers: user → gateway → agent → retriever → vector store, agent → tools, ingest source → scan → vault → index. The arrows are the data flows; each arrow is labelled with the data class flowing over it; each box is a trust boundary with an owner.

The caption calls out three things: *what is encrypted on each arrow* (TLS ≥ 1.2, mTLS, application-layer encryption), *who owns each box* (you, a sub-processor, the customer), and *which arrows cross a residency boundary*. A reader who looks only at this figure should be able to form a mental model of the privacy claims that follow.

### Section 3 — Retrieval access-control claims

This is the chapter 01 deliverable in prose. The two sub-claims:

> *3.1 Isolation shape. The retrieval index uses **{physical namespace / logical filter / hybrid RLS}** isolation across tenants. The filter predicate is enforced in **{the vector store / Postgres RLS / the application wrapper}**. Every chunk carries **{tenant_id, document_acl, sensitivity_label, source_id}** metadata; the schema is in appendix A.1.*
>
> *3.2 Caller identity. The retriever derives the authorisation predicate from the authenticated principal supplied by **{gateway/auth system}**. Request-body fields are not consulted for the tenant_id or the ACL filter. The caller-identity object is defined in appendix A.2.*

Add a paragraph on the ABAC extensions if applicable (sensitivity clearance, jurisdiction filter, erasure block-list). Name the policy document that governs each extension.

### Section 4 — Context-window-leakage test posture

Chapter 02's harness, summarised.

> *4.1 Negative test cases. A negative-test battery of **{N} example-based cases** covering cross-tenant retrieval, intra-tenant ACL escalation, request-body override, filter injection, metadata-corruption-on-ingest, and RLS bypass runs on every pull request that touches the retriever, the chunker, the ingest, or the vector-store SDK wrapper.*
>
> *4.2 Property-based tests. **{K} randomised scenarios** per CI run exercise the (caller, chunk) predicate across sampled tenants, groups, and ACLs.*
>
> *4.3 Staging canary. The battery runs against the staging environment after each deploy. The current build state is **{green / known-failing on case C since date D}**.*
>
> *4.4 Production shadow. A read-only subset of the harness runs against production every **{N minutes}**, under a service identity with normal application privileges. Last successful run: **{timestamp}**. Alerting routes to **{on-call team}**.*
>
> *4.5 Context-window-leakage paths beyond the retriever. The system defends against re-ranker-cache cross-tenancy via tenant-scoped cache keys, generation-KV-cache cross-tenancy via session-bound keys, and in-prompt tenant tagging for per-chunk origin. Each defence is tested in appendix B.*

### Section 5 — Logging and retention

Chapter 03's deliverables.

> *5.1 Log-schema classes. Operational logs are partitioned into **identifier**, **config**, **ops**, **content-redacted**, **content-metadata**, and **pointer** classes. Raw prompt and completion content is not stored in operational logs; it lives, when it lives at all, in the data-privacy vault (see 5.3).*
>
> *5.2 Redact-before-log. Prompts and completions pass through Presidio with recognisers **{list}** and operators **{list}**; the redactor fails closed on error. Residual free-text re-identification risk is managed by retention and access control.*
>
> *5.3 Data-privacy vault. Raw prompts and completions, when written for abuse investigation, are stored only in the vault at **{provider/location}**. Detokenisation requires **{role + case-ID + second-party approval}** and is itself audit-logged.*
>
> *5.4 Retention schedule. The retention table is in appendix C. Hot-store rows: identifier 90d, config 365d, ops raw 90d (aggregated 7y), content-redacted 30d, content-metadata 180d. Vault: 14 days.*
>
> *5.5 Access matrix. The matrix is in appendix D. Raw content access is just-in-time with a case ID and a second-party approval.*
>
> *5.6 Provider-side logging. We call **{model provider}**; their retention commitment is **{30d / zero-retention under enterprise tier / ...}** per their policy at **{URL}**. The contract term is **{reference}**. Training on API traffic is **{on / off}**.*

### Section 6 — Agent-tool scoping

Chapter 04's deliverables. For each tool the agent can call, one row; the table is the main artefact.

| Tool             | Identity binding | Scope requested            | Data classes touched       | Pre-invocation checks            | Logging                                | Revocation path                |
| ---              | ---              | ---                        | ---                        | ---                              | ---                                    | ---                            |
| gmail_read_unread| bind-to-user     | `gmail.readonly`           | email body, addresses      | consent valid, scope match, rate | request_id, user_id, count, latency    | user-side at Google; agent purges on refresh failure within 24h |
| calendar_write   | bind-to-user     | `calendar.events`          | event title, invitees      | consent + destructive confirm    | request_id, user_id, event_id, latency | user-side at Google            |
| crm_lookup       | bind-to-tool     | `crm:contacts:read:scoped` | contact fields             | user-claim enforced on tool side | request_id, user_id, contact_ids, latency | admin in CRM                 |
| ...              | ...              | ...                        | ...                        | ...                              | ...                                    | ...                            |

Immediately after the table, a prose paragraph names: the RFC 8693 token-exchange flow used for bind-to-user tools, the non-human identity model for the agent service, the data-boundary reviewer that prevents cross-tool data flow without consent, and the delegation-chain audit claim.

### Section 7 — Corpus ingest and provenance

Chapter 05's deliverables.

> *7.1 Sources. The pipeline ingests from **{allow-listed list of N sources}**; a per-source policy is in appendix E. The allow-list is reviewed quarterly.*
>
> *7.2 Provenance. Every chunk is linked to a document-provenance record with source_system, source_id, source_version, checksum_sha256, ingested_at, uploaded_by, mime_type, and processing_chain. Provenance records are retained independently of chunk lifetime.*
>
> *7.3 PII pre-scan. Documents are scanned with Presidio at ingest; the policy routes to **accept**, **route-restricted**, or **reject** based on the detected PII classes. High-risk classes (**{list}**) trigger rejection or restricted-index routing with tighter ACL and shorter retention.*
>
> *7.4 Change-rate / anomaly. Source change-rate is bounded at **{multiple}** of the 30-day moving average; exceedance pauses ingest and alerts **{team}**.*
>
> *7.5 Tombstoning. Takedowns write an immutable tombstone keyed on (source_system, source_id, checksum_sha256), delete chunks, purge vault records, scrub log references, invalidate caches, and schedule backup replay. The re-ingest guard rejects any attempted re-ingest of a tombstoned content hash.*

### Section 8 — Erasure and takedown

> *8.1 User erasure. A user's data-subject erasure request triggers inventory, vault purge, operational-log scrub, and backup replay as described in appendix F. Internal SLO: **24 hours for vault**, **7 days for operational log**, **90 days for backup/archive**. Current SLO attainment: **{percent / 30-day rolling}**.*
>
> *8.2 Legal takedown. Documents named in a lawful takedown notice are tombstoned per 7.5 within **{N days}** of receipt; the takedown record is retained for the regulatory period.*
>
> *8.3 Internal removal. Mis-labelled or incorrectly-ingested documents are tombstoned under the same flow, with internal-decision audit evidence.*

### Section 9 — Residual privacy risks and boundaries

The hard section. The discipline is to name what you have *not* defended against and where the complementary control lives.

> *9.1 Model-output leakage. The controls in this card are retrieval / ingest / logging / agent-tool controls. Verbatim memorisation of training data by the underlying model is the memorisation / DP story (modules 105–106 in the engineering curriculum) and is bounded by **{assumption / audit / provider-side control}**.*
>
> *9.2 Side-channel residency. Logs and vault records have a documented residency; the model provider's own compute residency is **{EU / US / configurable per region}** per **{reference}**.*
>
> *9.3 Prompt injection via retrieved content. The data-boundary reviewer does not read natural-language instructions found inside retrieved documents as authoritative scope changes. The agent's authorised scopes are derived from the user's consent, not from tool output. Tested cases in appendix G.*
>
> *9.4 Trust in sub-processors. Sub-processors named in appendix H receive data under DPA **{reference}**; retention, residency, and sub-sub-processor controls are inherited from the DPA.*
>
> *9.5 Open control boundary. Platform-level security (TEE attestation, fleet-wide key management, incident detection) is owned by **{platform-security / infra-security team}** and documented at **{reference}**. This card does not duplicate those claims.*

### Section 10 — Changelog

Datestamp each revision. Two columns: date and the material change. "Added case-6 RLS-bypass tests" is a line; "Minor typo" is not.

## The appendices

Appendices make the card checkable. Put the structured artefacts here so the prose stays short.

- **A. Schemas.** Chunk record, caller-identity object, tool scope spec, consent record, provenance record.
- **B. Test coverage.** The negative-test case list and the property-based generator spec; a pointer to the test file paths.
- **C. Retention table.** The full retention schedule by field class and store tier, with the enforcement mechanism per row.
- **D. Access matrix.** The full matrix by class and role, with just-in-time elevation protocol for sensitive classes.
- **E. Source list.** Every ingested source with its per-source policy.
- **F. Erasure runbook.** The step-by-step operational procedure for honouring a Right-to-Erasure request.
- **G. Pre-invocation checks.** The data-boundary reviewer's ruleset.
- **H. Sub-processors.** DPA-grade sub-processor list with role, data category, residency, and contract reference.

## Writing style — three small rules

Three habits that make the card survive a careful reader.

- **Claims are quantitative where they can be.** "Retention is 90 days" is a claim; "retention is appropriate" is not. If a claim resists quantification, say *why* ("we retain operational logs for 90 days to support typical abuse-investigation windows; a shorter horizon degrades our ability to ..."). The appendix is where the number lives.
- **Weasel words cost credibility.** "Industry-leading encryption" means less than "AES-256-GCM with keys held in {KMS}". The named mechanism is defensible; the adjective is not.
- **Boundary statements are a feature.** A card that says "we do not defend against X, which is owned by Y and claimed at Z" is more credible than one that implies it defends against everything. Readers trust honest boundaries.

## Review cadence

Sign the card, then review it. A reasonable cadence:

- **Each production deploy.** Build-system check: does the deploy touch anything claimed in the card? If the retriever's filter changed, the card goes back to the author.
- **Quarterly.** A full review by the privacy engineer and one other reviewer. Retention numbers get re-checked against the actual data-store configuration; the test posture section gets reconciled against the CI dashboard; the sub-processor list gets verified against procurement.
- **On regulatory change.** A new AI Act implementing act, a new ICO guidance, a new state AI law — all trigger a targeted review of the sections they bear on.

Date and signature at the top of each review; the trail of "reviewed on X by Y, no changes" is itself the compliance evidence.

## Mapping to external frameworks

For deployments that will face an external audit, add a short "framework mapping" appendix.

- **NIST AI RMF.** Map card sections to the Map / Measure / Manage / Govern functions and the subcategories (MP-3.2 for training-data governance, MS-1.1 for measurement of risks, etc.).
- **EU AI Act.** Map to Articles 10 (data governance), 12 (record-keeping), 13 (transparency), 15 (accuracy / robustness / cybersecurity). High-risk AI systems need all four; a RAG-agent deployment inside a high-risk use case typically needs the first three.
- **ISO/IEC 42001.** The emerging AI-management-system standard; mapping the card sections to its clauses helps a certification audit.
- **SOC 2 (if the system supports a service already in SOC 2 scope).** The card's logging and access-control sections support CC6 / CC7 controls.

The mapping does not change the card's content; it is a cross-reference. A single mapping table in an appendix is enough.

## A worked skeleton

```markdown
# {System Name} Privacy System Card, v1.0 (2026-{MM-DD})

## 1. Scope and version
{one paragraph}

## 2. Data flow
{figure}
{two paragraphs of caption}

## 3. Retrieval access-control
### 3.1 Isolation shape
{paragraph}
### 3.2 Caller identity
{paragraph}

## 4. Context-window-leakage test posture
### 4.1 Negative test cases
### 4.2 Property-based tests
### 4.3 Staging canary
### 4.4 Production shadow
### 4.5 Beyond the retriever

## 5. Logging and retention
### 5.1 Log-schema classes
### 5.2 Redact-before-log
### 5.3 Data-privacy vault
### 5.4 Retention schedule → appendix C
### 5.5 Access matrix → appendix D
### 5.6 Provider-side logging

## 6. Agent-tool scoping
{table + prose}

## 7. Corpus ingest and provenance
### 7.1–7.5 as above

## 8. Erasure and takedown
### 8.1–8.3 as above

## 9. Residual risks and boundaries
### 9.1–9.5 as above

## 10. Changelog

## Appendices A–H
```

Fill it in with the artefacts from chapters 01–05, and the card is the public face of the privacy-engineering work on the deployment.

## Summary — what you can do after this chapter

- Structure a nine-section privacy-slice system card that serves the customer privacy officer, the regulator's technical consultant, and the internal incident-response team at the same time.
- Translate each of the preceding chapters' engineering artefacts (chunk schema, test harness, logging policy, tool scopes, provenance records, tombstone flow) into concise, checkable claims in the card.
- Separate prose claims (three to six pages) from structured artefacts (appendices) so the document stays readable.
- State residual risks explicitly and name the boundary to the memorisation / DP / platform-security controls.
- Map the card to external frameworks (NIST AI RMF, EU AI Act, ISO/IEC 42001, SOC 2) for audit support.
- Operate a review cadence (per-deploy, quarterly, on regulatory change) and record each review as compliance evidence.

This closes the module's lecture arc. The exercises that follow — a per-tenant retrieval ACL design, a cross-tenant leakage harness, a prompt/completion logging policy, a redact-before-log pipeline, an agent-tool scoping and identity-propagation exercise, and the system-card write-up — exercise the design and the engineering end-to-end.
