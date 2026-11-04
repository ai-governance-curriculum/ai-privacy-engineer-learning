# NIST PRAM — Risk-Assessing the LINDDUN Model

LINDDUN enumerates threats. It does not, on its own, tell you which threats to spend on first. NIST's Privacy Risk Assessment Methodology (PRAM), the companion to the NIST Privacy Framework, is the overlay that turns an enumerated threat list into a *prioritised, defensible* risk ranking.

Primary sources:

- NIST Privacy Framework: [nist.gov/privacyframework](https://www.nist.gov/privacyframework)
- NIST Privacy Risk Assessment Methodology (PRAM) worksheets: [nist.gov/itl/applied-cybersecurity/privacy-engineering/collaboration-space/focus-areas/risk-assessment/resources](https://www.nist.gov/itl/applied-cybersecurity/privacy-engineering/collaboration-space/focus-areas/risk-assessment/resources)
- NIST IR 8062 (foundational concepts: *data actions*, *problematic data actions*, *privacy risks*).
- NIST SP 800-30 (likelihood × impact shape PRAM inherits).

Read the current PRAM worksheets before you commit to any scoring scheme — NIST updates them and the canonical worksheets are what regulators recognise.

## The PRAM vocabulary — four things you must name

PRAM frames privacy risk in a specific vocabulary. Internalise it before you score:

- **Data actions.** What the system *does* with personal data — collect, retain, generate, transform, disclose, transfer, retain, dispose. In an AI system: *ingest for training*, *embed into a retrieval index*, *send to inference endpoint*, *log*, *cache*, *display*, *delete*. A data action is a verb on data.
- **Data processing ecosystem.** The *who* and *where* of the data actions — operators, processors, sub-processors, downstream integrators, model providers, infrastructure providers, regulators. The DFD external-entities list (chapter 02) maps here.
- **Problematic data actions.** The subset of data actions that create privacy risk when context is considered — surveilling, unanticipated revelation, induced disclosure, stigmatisation, re-identification, inference, exclusion, loss of self-determination, loss of trust, economic / physical / dignity harms. PRAM gives a catalogue; use it.
- **Privacy risks.** A problematic data action considered against (a) the likelihood of a *problem* for the subject, (b) the impact if the problem materialises. Likelihood × impact.

LINDDUN enumerates *where* problems can arise; PRAM is *how significantly* they would hurt. They are complementary.

## Overlaying PRAM on a LINDDUN model

A workable sequence:

1. **Enumerate data actions** against your DFD. Each edge in chapter 02 produces one or more data actions. Each sink produces one or more retention-class actions ("retain prompt log for 90 days").
2. **Enumerate the data processing ecosystem.** For each data action, name the party *performing* it. Include processors and sub-processors.
3. **Classify each data action as problematic or not.** Use the PRAM catalogue (above) — do not invent your own classes without reason.
4. **For each LINDDUN threat you captured** (chapter 03/04/05), **tag it with the problematic data action(s) it manifests.** Sometimes a threat maps to a single action ("re-identification via embedding inversion" → "re-identification"); sometimes several ("prompt log retention" → "surveilling" + "induced disclosure" + "loss of self-determination").
5. **Score likelihood × impact.** For each (threat, problematic data action) pair, pick a scheme and score.
6. **Prioritise and route.** The top of the ranking goes to chapter 07's control routing.

## A defensible scoring scheme

PRAM is intentionally *scheme-agnostic* — it does not mandate a particular scoring taxonomy. A regulator does not care which scheme you chose; they care that it is *defensible, consistent, and documented*. A workable scheme has three properties:

- **Written-down anchors.** Each level ("Low", "Medium", "High") has a concrete definition that an engineer can apply without re-deciding on each threat. Anchors turn scoring from vibes into a repeatable process.
- **Separate likelihood and impact axes.** Combining them into one number early loses the diagnostic value — a "High impact / Low likelihood" threat warrants a different response than a "Medium / Medium". Keep them split until routing.
- **Explicit criteria for impact.** Privacy impact is not a single scalar — PRAM's problematic-data-actions catalogue already names the harm classes. State which harm class(es) the impact is measured against for each threat.

A reasonable baseline anchor table (adapt to your organisation):

### Likelihood anchors

| Score | Anchor |
|---|---|
| H | Credible exploitation path exists today without novel research; attacker / insider / failure mode is well-documented in the public literature *and* the control is absent or trivially bypassed. |
| M | Exploitation requires non-trivial effort or specific circumstances (e.g. a specific insider role, a specific side channel, a specific extraction technique) but is not speculative. |
| L | Exploitation requires a chain of unlikely conditions, novel research, or a party outside the modelled threat surface. |

### Impact anchors (by harm class)

| Score | Harm-class examples | Anchor |
|---|---|---|
| H | Re-identification revealing special-category data; stigmatisation at scale; unlawful ADMT with legal effect; breach of a published policy at scale. | The subject (or a population of subjects) faces a materially adverse outcome — legal, financial, reputational, physical, dignity — that they cannot readily recover from. |
| M | Induced disclosure of non-special PII; surveilling in a way the subject did not anticipate; loss of self-determination on a specific processing. | A subject has standing to complain to a regulator; corrective action is required; business implications are non-trivial. |
| L | Technical leakage of pseudonymous data that a reasonable reader would not use to harm the subject; a transparency gap that is embarrassing but not injurious. | No subject-visible harm beyond principle violation; tidy-up required. |

## Putting it all in a register

The PRAM-overlaid threat register for an AI system has these columns at minimum:

| ID | LINDDUN threat statement | Element | LINDDUN category | Data action | Problematic data action(s) | Ecosystem party | Likelihood | Impact | Harm class(es) | Control routing | Owner | Status |

Example rows (building on chapter 04's semantic-cache example):

| ID | Threat | Element | LINDDUN | Data action | Problematic | Party | L | I | Harm | Control | Owner | Status |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| T-014 | Semantic cache cross-tenant return under embedding collision. | semantic cache | L+D2 | retrieve, disclose | induced disclosure; loss of self-determination | ops | H | H | unanticipated revelation; re-identification | tenant-scoped cache key; cross-tenant negative test | retrieval-lead | Open |
| T-015 | Prompt log retains PHI with weak IAM. | prompt log | D2+U | retain | surveilling; unanticipated revelation; exclusion from rights | ops; log-reader role | M | H | HIPAA breach of confidentiality; loss of trust | redact at write; segregate log ACL to privacy-reader role | logs-lead | Open |
| T-016 | MIA on `v2` fine-tune detects training-set membership. | model weights | D1 | generate (via API) | re-identification; inference | ops | M | H | surveilling; stigmatisation (if inferred special-category) | DP-SGD retrain; confidence capping on API | ml-lead | Scoped |

This register is the artifact the DPIA (mod-111) will quote. Keep it versioned in the same repo as the DFD and the threat-tree document.

## Likelihood: what to count, what to ignore

Treat likelihood as a property of the system *as designed and operated today*, not after planned controls. "We'll add DP-SGD next quarter" does not lower the likelihood on today's model; it lowers the *residual* likelihood of the *post-control* system. Keep inherent and residual scores as separate columns if you need both.

Common AI-specific likelihood evidence:

- Published attack literature on the exact threat (e.g. Carlini et al. extraction results, LiRA MIA, vec2text inversion).
- The presence / absence of a control in your stack today.
- The size and authentication posture of the exposed surface.
- The population of plausible actors (external, downstream integrator, insider, model provider).

Resist treating likelihood as a personal guess. Anchor against the published literature and the actual state of the system.

## Impact: do not collapse harm classes

A single privacy threat can produce *several* harm classes simultaneously (surveilling + induced disclosure + stigmatisation). Record them all. The DPIA will need them; the control routing (chapter 07) frequently selects controls that address one class but not another.

Also do not anchor impact on *the operator's reputational risk*. PRAM is about harm to the *subject*. Operator cost is a business consideration that belongs in a separate business-risk register, not here.

## Residual risk and acceptance

After a control is designed, score the *residual* risk:

- **Residual likelihood** — assuming the control works as designed, how likely is the harm now?
- **Residual impact** — if the control fails, does the impact meaningfully shrink? (Some controls — redaction — shrink impact by lowering PII density. Others — DP-SGD — lower likelihood without changing the worst-case impact.)

Any threat accepted without a control (residual = inherent) needs a **named accepter** — a role with authority to accept privacy risk on behalf of the subject. On most engineering orgs, that authority belongs to the DPO (not you). Record the accepter per chapter 09.

## PRAM and the DPIA

GDPR Art. 35 DPIAs require a *risk assessment* with a defensible scoring scheme. A PRAM-shape overlay plus a LINDDUN model directly satisfies Art. 35(7)(c) (assessment of risks) and 35(7)(d) (measures envisaged to address those risks). The PRAM register is the audit trail. The EU AI Act Annex IV technical-documentation bundle re-uses the same register. Build the register once; cite it everywhere.

## Summary — what you can do after this chapter

- Name data actions, the data processing ecosystem, problematic data actions, and privacy risks in PRAM's vocabulary.
- Overlay PRAM on top of a LINDDUN threat list to produce a prioritised register.
- Design and document a defensible likelihood × impact anchor scheme separated by harm class.
- Separate inherent from residual risk and name an accepter for any unmitigated threat.
- Hand the register to a DPIA (mod-111) without rework.

The next chapter walks misuse personas against privacy and routes each threat to a concrete control family.
