# Data Privacy Vaults and Tokenisation

Minimisation says *do not pass the data when the task does not need it*. Vault + tokenisation goes further: when the data *is* needed downstream but the downstream is untrusted (a third-party model provider; a vector DB; an analytics tool), replace the data with a *token* and keep the raw value in an isolated vault that only authorised callers can detokenise. The model layer, the retrieval layer, and the log layer see tokens; the raw PII never crosses the vault boundary.

This chapter is the engineer's reference for the pattern: what a data privacy vault *is*, what token types exist, how to integrate the vault with training / retrieval / inference / logging, and the invariants you enforce on top.

## Why this matters

Three reasons the vault pattern is the right default for AI pipelines:

- **Model providers are a trust boundary that your controls do not cross.** When you send a prompt containing `alice@example.com` to a hosted LLM, the raw email has left your environment. Contract terms (Art. 28) help; they do not substitute for not sending the value. A token `email_tok_8f3c…` sent instead makes the vendor incident less damaging.
- **Vector DBs are derivative sinks that outlive retention.** Even after you delete the source document, the embedding and its metadata live in the vector index until a reindex runs. A vault token in the chunk keeps the raw value out of the index in the first place.
- **Prompt / completion logs are the single most consistently leaky sink.** Tokenising PII before the prompt reaches the model means the log row — intentional or accidental — contains tokens, not identifiers.

The pattern is known commercially as a **data privacy vault** and is associated with Skyflow's architecture (whose "Polymorphic Data Encryption" marketing-slide shapes the industry reference). The pattern itself is older — it traces to PCI DSS tokenisation (NIST SP 800-57 / PCI SSC guidance / ANSI X9.119) — and open-source / self-hostable variants exist. Vendor-neutral, the pattern is what matters.

Primary references:

- Skyflow — *data privacy vault* architecture. [skyflow.com](https://www.skyflow.com/) (vendor material; useful as the shape).
- PCI Security Standards Council — *Tokenization Product Security Guidelines* and *Information Supplement: PCI DSS Tokenization Guidelines*. [pcisecuritystandards.org](https://www.pcisecuritystandards.org/).
- NIST SP 800-57 Part 1 Rev. 5 — key-management recommendations. [doi.org/10.6028/NIST.SP.800-57pt1r5](https://doi.org/10.6028/NIST.SP.800-57pt1r5).
- ANSI X9.119-2 (reversible-cryptographic-token standard for payment data).

## What a data privacy vault is

A privacy vault is a *storage service with four properties*:

- **Isolation** — a dedicated store with its own authentication, authorisation, logging, and network boundary. Not a table in the production warehouse.
- **Policy-controlled detokenisation** — detokenisation is an authenticated, authorised, audited API call; direct database access to the raw columns is not available even to privileged operators.
- **Fine-grained ACL** — the policy evaluates *who, from where, for what purpose, which fields, in which format*. A caller who can read `email` in `cleartext` for `customer_support` may only read `email` in `masked` form for `analytics`.
- **Audit trail by default** — every detokenisation call is logged with caller, purpose, requested fields, response format, and record identifier. The audit is retained for the Art. 30 and DSR access responses.

What the vault is *not*:

- Not a KMS. The vault uses a KMS underneath; it is not a replacement for one.
- Not a general-purpose database. The schema is narrow — it stores sensitive fields, not the whole transactional record.
- Not an encryption-at-rest checkbox. An RDS with KMS encryption is not a vault, because the rest of the application can still SELECT the plaintext.

## Token types

Three token shapes cover most of the pipeline. Pick per-field based on what the consumer needs.

### Opaque reversible token

- Shape: `email_tok_8f3c6e2a2f…`
- No semantic / structural relation to the underlying value.
- Resolved by vault API call: `vault.detokenise([email_tok_8f3c…])` → `["alice@example.com"]`.
- Best for: free-text embedding consumers; log sinks; prompt bodies.
- Deterministic variant (same value → same token) supports joins and equality checks downstream without detokenisation; the trade-off is that the mapping leaks equality information to anyone who sees the tokens.

### Format-preserving token (FPE-shape)

- Shape: a string that *passes the receiving system's validation* — `fpe-email-dp1k2@tok-1a.example` validates as an email; `4012…345-FPE` validates as a Luhn-passing PAN.
- Implementation: format-preserving encryption (FPE) per NIST SP 800-38G (FF1 / FF3-1), domain-specific schemes, or structure-preserving pseudorandom mapping.
- Best for: cases where downstream validation / schema constraints would reject an opaque token (an email-validator field; a legacy system expecting a 10-digit phone).
- Risk: format-preservation can leak length / character-class information. Record the leakage in the datasheet.

### Masked / redacted (irreversible)

- Shape: `al***@example.com` or `***-***-1234`.
- Not reversible at all; the raw value is still stored in the vault, but the consumer receives a projection.
- Best for: UIs rendering to a human; analytics that need not resolve to individuals; cross-tenant dashboards.

Choose the shape *per-field, per-consumer*. The vault policy records which consumer gets which shape; the datasheet records which shape each downstream saw.

## The pattern in an AI pipeline

A reference sequence for a training / retrieval / inference AI system that uses a vault:

```text
┌─────────────┐        ┌─────────────────────────┐
│  capture    │──┬──▶ │  vault  (raw PII)       │
│  (ingest)   │  │    │  key-management: KMS     │
└─────────────┘  │    │  audit sink              │
                 │    └─────────────────────────┘
                 │                 ▲
                 │                 │  detokenise
                 │     tokens      │  (audit + policy)
                 ▼                 │
            production tables, logs, retrieval chunks, prompts, telemetry
                 │
                 ▼
          ┌──────────────┐        ┌────────────┐         ┌────────────┐
          │ trainer      │        │ retrieval  │         │ inference  │
          │ (tokenised)  │        │ (tokenised │         │ (tokenised │
          └──────────────┘        │  chunks +  │         │  prompt +  │
                                  │  embeds)   │         │  tokens)   │
                                  └────────────┘         └────────────┘
```

Three integration points matter:

### Integration with training

- At ingest, each sensitive field is replaced with a token. The downstream training view (chapter 03) reads the tokenised column; the raw column is not visible to the trainer's service account at all.
- For free-text fields, the training pipeline runs mod-104 PII detection, and *every* detected span becomes a vault token before the text is written to the training snapshot. The snapshot stores tokens inline (e.g. `My name is ⟨name_tok_8f3c⟩ and I live at ⟨addr_tok_3d2a⟩`).
- Deterministic tokens let the trainer learn relational signals (same email in two tickets is the same token) without memorising the raw value. The *de-anonymisation* risk shifts from "what did the model memorise" to "who holds detokenise permission on these tokens".
- The datasheet records the token scheme and the de-anonymisation residual risk.

### Integration with retrieval

- Documents are tokenised at ingest; the chunk, the embedding, and the metadata entry in the vector DB carry tokens rather than raw PII.
- At query time, the user's query may contain raw PII (their own email, their own order ID). The orchestrator decides whether the query crosses the vault boundary at all:
  - Preferred: tokenise the user's query values (using the same scheme) and let the vector DB match tokens against tokens.
  - Alternative: embed the raw query and let the retrieval match semantically on the token embeddings' neighbourhood; this works less well.
- The retrieved chunks are detokenised *selectively* at render — the user's own name / email in a retrieved chunk is detokenised back; another customer's that appears in the chunk is not detokenised and is rendered as the opaque token.
- See mod-109 for the tenant-ACL layer that stacks on top.

### Integration with inference

- The prompt built for the model is a composition of system prompt + retrieved chunks + user input. Each component has been tokenised where sensitive values appear.
- The model provider receives the tokenised prompt. A model-provider-side cache that retains the prompt retains tokens, not raw PII.
- The completion is received in tokenised form — the model, prompted with tokens, emits tokens. The egress proxy detokenises the completion for the authorised consumer (the end-user; the support agent). Log sinks can be configured to log the tokenised completion rather than the detokenised one.
- Where the model would have memorised an email address, it instead memorises a token — which is useless without detokenise permission (mod-105 depth).

### Integration with logging

- Logs are the single easiest integration: write the tokenised payload (because the pipeline already produced it). The log sink receives tokens; a reviewer who holds the detokenise permission can resolve for a specific investigation.
- DSR erasure of a logged row becomes: delete the log row *and* delete the vault entry for its tokens. A log row that outlives its vault entry is already detoxified (the token no longer resolves).

## The vault's own obligations

The vault is a *high-value sink*. It is now holding the raw PII for the whole pipeline. Treat it accordingly:

- **KMS-backed field-level encryption.** The underlying storage is encrypted; keys are rotated; HSM-backed where the compliance tier requires.
- **Separate network segment.** The vault is reachable by the ingest, detokenise, and admin paths — nothing else.
- **Strict least-privilege IAM.** Service accounts hold *purpose-specific* detokenisation roles. A support-agent terminal that resolves a customer's email has a different role from an analytics job that only needs a masked format.
- **Rate-limited and anomaly-detected detokenisation.** A sudden detokenisation of 10,000 emails is an incident, not a feature. Thresholds + alerting are mandatory.
- **Full audit on every detokenise call** — caller, purpose, format, record IDs, response-size, outcome.
- **Backup / restore under the same ACL.** A vault backup restored into a staging environment is a risk; a signed procedure is required.
- **Attestation / confidential compute for high-stakes use cases** — mod-108 covers.

The vault is also the target of the DPIA's Art. 32 "appropriate technical and organisational measures" paragraph. Document in the DPIA that the vault exists and summarise its controls — the control is not *only* real if the regulator can read it.

## Policy — who can detokenise what, under what purpose

A vault's power lives in its policy engine. A worked policy fragment:

```yaml
# vault-policy.yaml
roles:
  - name: support-agent
    description: A support agent handling a live ticket.
    allowed:
      - fields: [name, email, phone]
        format: cleartext
        purpose: customer_support
        condition: request.ticket_id matches token.subject_ticket_id
      - fields: [billing_address]
        format: masked
        purpose: customer_support
  - name: fine-tune-trainer
    description: ML platform account that runs training.
    allowed: []        # trainer never detokenises; it only sees tokens
  - name: analytics
    description: BI / analytics job.
    allowed:
      - fields: [email]
        format: masked
        purpose: analytics
        condition: result.aggregation_size >= 100   # k-anon-ish guard
  - name: dsr-processor
    description: DSR erasure/access job.
    allowed:
      - fields: [name, email, phone, billing_address]
        format: cleartext
        purpose: dsr_access | dsr_erasure
        condition: request.dsr_token is valid
```

Three features of the policy worth copying:

- **Purpose binding.** A role's entry carries a `purpose`; a request without a matching purpose header is rejected. This is the enforcement side of chapter 03's propagation contract.
- **Condition clauses.** The support agent can detokenise a customer's email only when the current ticket's subject matches the token's subject. Cross-customer detokenisation by a support agent is a separate, signed-off path (and should page).
- **Aggregation guards.** Analytics gets masked values *and* only when the result aggregates many records. A guard that requires `aggregation_size >= 100` is a weak k-anonymity-shape guard; combine with mod-104 tabular de-identification for depth.

## Failure modes to design against

Five known weak points:

- **Detokenisation egress.** An employee with a legitimate detokenise role exports 50,000 rows into a spreadsheet and emails it. Rate-limits + anomaly detection + DLP on the egress edge mitigate; the datasheet records the residual risk.
- **Token-leak via joins.** Deterministic tokens enable joins across sinks; an attacker with tokens from two different sinks can link records even without detokenisation. Recorded in the datasheet; mitigate with per-sink tokenisation keys where join-across-sinks is not needed.
- **Format-preserving leakage.** FPE preserves length, character class, structure. A length-sensitive field (zip codes; small-cardinality categorical values) is close to re-identifiable via FPE alone. Pick opaque tokens where downstream validation permits.
- **Silent plaintext fallthrough.** An old code path reads from the raw table rather than the token column. The bug is invisible until the raw column is deleted. CI test: assert the raw column is unreachable by the consumer service accounts.
- **Model-provider retention of tokens.** If the model provider retains inference payloads for its own fine-tuning, your tokens are in their training set. The vendor contract and the provider's zero-retention mode (where available) are the controls; confidential-compute inference (mod-108) is the hard-fallback.

## Build-vs-buy

Open-source primitives:

- **HashiCorp Vault** — a secrets manager with transit / format-preserving encryption endpoints. Build-your-own-vault lego; not a drop-in data privacy vault.
- **OpenFHE, Microsoft SEAL** — homomorphic encryption primitives for compute-on-encrypted-tokens (mod-108 depth).
- **Google Cloud DLP, AWS Macie + KMS, Azure Purview** — cloud-native tokenisation / discovery + encryption; close to the pattern but not identical.

Managed:

- **Skyflow** — the reference vendor; defines many of the engineering idioms this chapter uses.
- **Very Good Security (VGS), Evervault, Protegrity, Piiano, Transcend Data Vault.**

Build vs. buy is a cost + control trade-off. Record the choice in the datasheet (what vault / vendor is in scope) and the DPIA (the Art. 28 processor agreement).

## The datasheet entry for a vault-integrated dataset

The chapter 02 datasheet gets a `tokenisation` subsection:

```yaml
tokenisation:
  vault: skyflow-eu-tenant-1          # or self-hosted-vault-v2
  scheme:
    name: opaque_deterministic_v3
    key_rotation: 90d
    cross-sink_key_separation: true    # separate keys per sink
  fields_tokenised: [name, email, phone, billing_address, order_id]
  fields_format_preserving: [phone, billing_address_zip]
  fields_masked_for_analytics: [email]
  detokenisation_roles: [support-agent, dsr-processor]
  audit_sink: vault-audit-2025
  residual_risk_notes: |
    Deterministic tokens permit equality-join across sinks; see cross-sink_key_separation
    policy above. FPE on phone leaks length (always 10 digits for US tenants); accepted.
```

The datasheet entry is what the DPIA cites.

## Summary — what you can do after this chapter

- Design a data privacy vault + tokenisation integration for training, retrieval, inference, and logging paths of an AI system.
- Pick a token shape (opaque / FPE / masked) per field based on the downstream consumer's requirements.
- Write a vault policy that binds detokenise to purpose, role, and condition clauses.
- Recognise the five standard failure modes (detokenise egress, token-leak via joins, FPE leakage, plaintext fallthrough, provider-side retention) and the mitigations.
- Record the vault + tokenisation choice in the datasheet and the DPIA.

The next chapter picks up the complementary move: *synthetic* data as a substitute for real data where the vault pattern is not enough (or where the product team wants volume for an evaluation set that would never consent to being on the eval set).
