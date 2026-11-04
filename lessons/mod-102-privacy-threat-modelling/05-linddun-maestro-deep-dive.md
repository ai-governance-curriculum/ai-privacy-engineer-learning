# LINDDUN MAESTRO — When the Depth Is Warranted

LINDDUN GO (chapter 04) gives you a working privacy threat model in a workshop sitting. LINDDUN MAESTRO is the deeper, more methodological pass you run on the hot spots the GO workshop surfaced — or on a production system where the threat model needs to hold up to regulator scrutiny.

The depth is where this chapter earns its keep. MAESTRO forces per-element enumeration, threat-tree refinement, and documented reasoning for every rejected candidate. You pay more; you get an artifact that survives an audit.

> **Terminology note.** The LINDDUN family's canonical methodological-depth variant is sometimes referred to in current literature as LINDDUN PRO; this module uses "LINDDUN MAESTRO" per the curriculum spec to refer to the same deeper-pass methodology. Confirm the exact method name against [linddun.org](https://linddun.org/) before publishing an artifact externally. <!-- needs-research: confirm whether the canonical current name is LINDDUN PRO, LINDDUN MAESTRO, or a distinct methodology; align terminology across all mod-102 chapters once confirmed -->

## When MAESTRO is worth the cost

MAESTRO is a multi-session exercise. Budget days of analyst time, not hours. Reach for it when:

- GO surfaced a threat whose *cost of mishandling* dominates the model (training-data memorisation on a published fine-tune; cross-tenant RAG leak; special-category data in a scraped corpus).
- You are preparing a DPIA (mod-111) for *high-risk* processing under GDPR Art. 35 — regulator eyes will be on the reasoning.
- An EU AI Act Annex IV technical-documentation bundle (mod-101 chapter 03) requires a documented privacy-risk analysis.
- An incident has pushed the team into a "we need to rebuild the threat model from scratch" moment and GO is too shallow to reach consensus.

The time cost is rewarded when the artifact is *re-used* — DPIA, Annex IV, audit response, regulator query, incident post-mortem. If the artifact is single-use, GO probably suffices.

## The MAESTRO shape

MAESTRO's depth over GO comes from four additions:

1. **Systematic per-element coverage.** Every DFD element times every LINDDUN category gets a cell. Cells are either (a) a documented threat, (b) a documented refinement into sub-threats, or (c) a documented "not applicable because …". GO allows a cell to go blank by default; MAESTRO does not.
2. **Threat-tree refinement.** A threat that is too abstract to act on gets refined into a tree of concrete sub-threats, each with its own likelihood / impact. The tree is the audit trail — it shows a reader *how* you reasoned from "L on the semantic cache" down to "cross-tenant cache hit under embedding-collision when `k ≥ 3`".
3. **Documented assumptions and scope decisions.** Every "out of scope" assertion names *who decided* and *why*, and references the chapter 09 role-boundary or an explicit governance document.
4. **Linked controls, not just control families.** GO ends at "redaction family"; MAESTRO ends at "`presidio-redactor@2.4.1` configured with recogniser set X in `redact.py#L112`, exercised by the `test_redact_cache_write` integration test".

## Running a MAESTRO pass

### Step 1 — Rebuild / verify the DFD at depth

The GO DFD was fit-for-purpose for a workshop. For MAESTRO:

- Drill every data store into sub-stores (e.g. "prompt log" → `raw_prompt_log` (S3 bucket, 90d) / `evaluation_sample` (S3 bucket, 2y) / `abuse_review_queue` (DynamoDB, indefinite)). Each sub-store is its own DFD element.
- Expand every process into its sub-processes where the trust boundary differs (e.g. "inference" → `prompt_preprocess`, `model_call` crossing to the provider, `postprocess`, `log_write`).
- Document every assumption about data classes — if "prompt" sometimes carries PHI and sometimes does not, split the flow by class.

### Step 2 — Build the completeness matrix

A table with rows = DFD elements (sub-stores and sub-processes included), columns = the seven LINDDUN categories. Every cell gets one of:

- a **threat ID** (link to an entry in the threat register), or
- a **refinement ID** (link to a threat tree), or
- **N/A: `<explicit reason>`**.

A blank cell is a gap. Teams often find that after step 1 the matrix is 40–120 cells; this is normal.

### Step 3 — Refine threats into trees where needed

A threat tree starts with a root statement — the GO-level one-liner — and refines downward until each leaf is specific enough to:

- route to a *named* control (not a family),
- score *independently* on likelihood and impact,
- test with a *specific* probe or integration test.

Example tree for the semantic-cache linkability threat from chapter 04:

- **Root L.cache.1** — Semantic cache enables cross-session linkability on a subject.
  - **L.cache.1.a** — Cache keyed on raw prompt-embedding similarity below threshold `θ = 0.92` returns another user's record.
    - **L.cache.1.a.i** — Threshold too loose for prompts containing shared boilerplate (e.g. "my order number is …").
    - **L.cache.1.a.ii** — Collision rate scales with prompt length < 50 tokens.
  - **L.cache.1.b** — Cache retains cross-session records beyond a session TTL.
    - **L.cache.1.b.i** — Default TTL inherited from the pre-AI service is `3600s`; stays live across typical session-switch intervals.
  - **L.cache.1.c** — Cache key omits tenant scope.
    - **L.cache.1.c.i** — When tenants share the same retrieval provider, the shared cache is the concrete leak path.

Each leaf gets scored (chapter 06), each gets a control routed to it, each gets an acceptance test.

### Step 4 — Document assumptions and scope decisions

A separate "decisions" section with entries of the form:

> **Decision D-12 — Agent authentication surface is out of scope for this privacy threat model.** Owner: `security-peer`. Reason: authentication is under the security peer's STRIDE pass on this system (ref SEC-TM-2026-04). This privacy model assumes the security pass is complete and trusts its authentication findings. Re-open if the security pass finds a gap that affects per-subject disclosure risk.

Scope decisions are what turn a model into a maintainable artifact. Without them, the next person who inherits the model does not know what was deliberately excluded vs. simply missed.

### Step 5 — Link controls to code and tests

Every surviving leaf threat gets a row in a controls table:

| Threat leaf | Control (specific) | Code / config reference | Test (specific) | Evidence artifact |
|---|---|---|---|---|
| L.cache.1.a.i | Raise embedding-similarity threshold to `θ = 0.98`; add tenant-salt to cache key | `services/cache/key.py#L41`, `infra/cache/config.yaml` | `test_cache_cross_tenant_negative` | integration-test log; cache-dump audit |
| L.cache.1.b.i | Cache TTL reduced to `60s`; add per-tenant TTL override | `infra/cache/config.yaml` | `test_cache_ttl_expiry` | `retention-cycle` CI job log |
| L.cache.1.c.i | Cache keyed on `(tenant_id, embedding, request_hash)` | `services/cache/key.py#L41` | `test_cache_key_tenant_scope` | design-review doc, code-owner sign-off |

This is the shape chapter 06 (PRAM) prioritises and chapter 08 (Privacy by Design) enforces as release gates.

## Threat-tree refinement discipline

Three heuristics for stopping refinement at the right depth:

- **Named control.** Can you write the control sentence without a wildcard? "Redact PII" is not named; "Presidio with the `PERSON`, `EMAIL`, `PHONE_NUMBER`, `CREDIT_CARD` recognisers at cache-write time" is.
- **Independent score.** Can two leaves plausibly have different (likelihood × impact) scores? If yes, they belong as separate leaves; if no, collapse.
- **Independent test.** Can the leaf be covered by its own test / probe? If yes, keep; if not, refine until it can.

Over-refinement is a real failure mode. A tree with 50 leaves where 40 are untestable is worse than a tree with 15 leaves where each is actionable.

## AI-specific MAESTRO depth areas

Some DFD elements routinely require MAESTRO depth because GO cannot reach the specificity they need:

- **Model weights** — threats here fan out into memorisation, extraction, membership inference, model inversion, side-channel. Each has distinct controls (DP-SGD, output filtering, confidence capping, access control on model API). mod-105 is the depth source.
- **Training corpora** — lawful-basis and consent-propagation threats are rarely resolvable at GO depth; the tree extends into per-source consent, per-source special-category screening, per-source retention. mod-103 is the depth source.
- **Retrieval corpora in multi-tenant systems** — cross-tenant threats fan out by index topology, by ACL propagation, by failure-mode catch (what happens on an unauthenticated request). mod-109 is the depth source.
- **Downstream integrator surface** — threats where another party accumulates traces against subjects. Chapter 07 of this module (misuse personas) is the depth source.
- **Agent tool-call plane** — tool-argument linkability, cross-tool identifier leak, memory contents. mod-109 is the depth source.

A MAESTRO pass without a plan for which of these you will deepen is going to run long and shallow. Plan the deepening before the pass.

## Review and sign-off

A MAESTRO artifact is reviewed by:

- **The product engineering lead** — do they recognise the system in the DFD?
- **The security peer** — is the out-of-scope-to-security list correct?
- **The DPO delegate** — are the non-compliance / lawful-basis threats correctly routed?
- **A second AI Privacy Engineer** (peer review) — is the reasoning defensible?
- The artifact is **signed-off** with a version and a date. Re-run on a trigger (new fine-tune, new corpus, new tenant pattern, incident).

## Summary — what you can do after this chapter

- Decide whether a system warrants MAESTRO depth or stays at GO.
- Rebuild the DFD at sub-store / sub-process depth and build the completeness matrix.
- Refine a threat into a threat tree with named controls, independent scores, independent tests.
- Document scope decisions and assumptions so the artifact is maintainable.
- Link every surviving leaf threat to specific controls, code locations, and tests.

The next chapter takes the MAESTRO (or GO) output and overlays a NIST PRAM-shape risk assessment so engineering can prioritise defensibly.
