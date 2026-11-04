# The Boundary to `ai-evaluation-engineer` — Contributing Into Release Assurance

The AI privacy engineer owns a specific, well-defined contribution: the DPIA, the PRAM, the EDPB / CNIL / ICO-aligned risk analysis, the sectoral cross-tag, and the regulator-facing bundle (chapter 06). That contribution is substantial — it is this module's output. It is not, however, the whole of release assurance. The role that owns release assurance, audit-trail methodology, and regulator-facing release decisions at the full-time depth the biggest AI deployments demand is a different role: **`ai-evaluation-engineer` (level 35)**, a peer / next-up role in the curriculum. This final chapter names that boundary cleanly, so the privacy engineer knows what they contribute, what they do not, and how to hand off into the broader release-assurance interface.

The chapter is short by design. The core content lives in `ai-evaluation-engineer`'s own curriculum; this chapter's job is to establish the interface — what the privacy bundle looks like as an input to that role, what expectations the evaluation engineer places on it, and what the privacy engineer should not try to own.

## Why there is a boundary at all

Three forces push the full-depth release-assurance work to a distinct role:

- **Scope** — release assurance covers safety, fairness, robustness, security, performance, cost, and privacy. Privacy is one of seven top-level domains; a role that owns all seven at depth cannot also own the engineering practice of any one. At level 35, release assurance is the primary job.
- **Methodology** — release assurance is a methodological discipline. It owns the evaluation methodology (how an eval set is constructed, how metrics are pre-registered, how the evaluation is reproduced), the audit-trail methodology (how the evidence of a release decision is recorded), and the regulator-facing methodology (how release evidence is translated to a supervisory authority's expected shape). The methodology is distinct from the privacy-engineering practice.
- **Decision authority** — the release-go/no-go decision carries accountability. It belongs with a role that has the organisational seniority and the cross-domain evidence. Level 35 is where the decision sits, with contributions from level-30 specialists (the privacy engineer, the safety engineer, the fairness engineer, the robustness engineer) and sign-off from the product owner.

A healthy programme organises around this division. An unhealthy programme asks the privacy engineer to make the release-go/no-go decision; the privacy engineer should notice when the ask arrives and redirect it.

## The interface — what the privacy engineer sends into release assurance

The release-assurance process consumes evidence from each specialist domain. The privacy engineer's contribution, in the form the evaluation engineer expects:

- **The DPIA** (chapter 01, chapter 02), with Step 7 approval chain complete and change log current to the release candidate.
- **The PRAM output** (chapter 03), with Worksheets 2–3 reflecting the release candidate's data actions and Worksheet 4 reflecting the controls applied.
- **The EDPB / CNIL / ICO artefact set** (chapter 04), with the anonymity claim (if any), the per-phase LI test, the training-data provenance chain, and the per-right DSR artefact set.
- **The sectoral cross-tag** (chapter 05), with HIPAA / GLBA / SP 800-53 PT coverage per applicable overlay.
- **The bundle** (chapter 06), with the eight artefact classes versioned to the release candidate and the manifest reviewed.
- **The enterprise-overlay cross-tag** (chapter 07), with AICM, 42001, AI RMF, 27701 identifiers populated.
- **A release-recommendation note** — the privacy engineer's one-page recommendation: proceed / proceed with conditions / block. The recommendation cites the artefacts and names the residual risk. It is the privacy engineer's input to the release decision, not the decision itself.

The hand-off is a package the evaluation engineer reviews alongside packages from safety, fairness, robustness, security, and performance specialists. The evaluation engineer synthesises, surfaces open questions, and presents the composite release view.

## What the evaluation engineer owns that the privacy engineer does not

Items the evaluation engineer owns at depth and the privacy engineer contributes to without owning:

- **Release-gate methodology** — the pass/fail criteria for the release decision as a whole, the pre-registration of criteria, the governance around changing criteria.
- **Audit-trail schema design** — the shape of the audit trail across all domains (not only privacy). mod-110 chapter 07 is privacy's contribution; the broader audit-trail design is level-35 work.
- **Regulator-facing release methodology** — the translation of release evidence into the shape the supervisory authority (or the AI Act's notified body, where relevant) expects. The chapter-06 bundle is an input; the release-level translation is level-35 work.
- **Cross-domain evidence synthesis** — the composite view that combines privacy residual risk with safety residual risk with fairness residual risk. A release where privacy residual risk is low but fairness residual risk is high requires cross-domain adjudication.
- **Shipping-cadence governance** — the cadence at which releases are shipped and the gate at each cadence. Privacy contributes to each release; the cadence itself is a release-assurance decision.
- **Post-release monitoring design** — the production-monitoring plan. The privacy engineer specifies what privacy signals are monitored (extraction-attempt indicators, DSR SLA breach, unusual-access patterns); the overall monitoring architecture is level-35 work.
- **Regulator-liaison methodology** — the ongoing engagement with supervisory authorities, the Art. 36 prior-consultation process at the methodological level, the escalation pathways. The privacy engineer is a subject-matter expert in those engagements; the methodology is level-35.

## What the privacy engineer owns that the evaluation engineer defers to

Reciprocally, the privacy engineer owns at depth the following items that the evaluation engineer defers to:

- **The DPIA and PRAM content** — the evaluation engineer does not re-author the DPIA; they consume it and challenge it where it under-specifies. The privacy engineer is the DPIA's author.
- **The PET deployment claims** — DP parameters, federated-aggregation protocol specifications, TEE attestation evidence are the privacy engineer's technical content.
- **The DSR runbook set** — mod-110's output is the privacy engineer's territory.
- **The lawful-basis analysis per phase** — the EDPB Opinion 28/2024 three-step LI test, the Art. 9 handling, the special-category reasoning. The evaluation engineer accepts the privacy engineer's analysis, subject to challenge if it under-specifies.
- **The sectoral obligations cross-tag** — HIPAA / GLBA / Reg S-P / SP 800-53 PT coverage.
- **The privacy-specific incident-response branches** — the AI-specific scenarios the breach-response plan must cover.
- **The data-flow diagram's content** — the evaluation engineer may own the broader architectural-documentation discipline; the DFD's content and accuracy from the privacy perspective is the privacy engineer's responsibility.

The boundary is clean when both sides respect it. Where it blurs — the evaluation engineer tries to rewrite the DPIA, or the privacy engineer tries to set the release-gate criteria — the programme has a scoping problem that should be addressed explicitly rather than negotiated incident by incident.

## When the escalation goes privacy → release-assurance

Specific scenarios where the privacy engineer escalates into release assurance:

- **The chapter-01 residual-risk determination is "high" after mitigations** — Art. 36 prior consultation is engaged. The release-assurance process must absorb the implication: the release cannot ship without the supervisory authority's consultation outcome. The privacy engineer flags the condition; the release-assurance process decides how to operate.
- **A measured MIA / extraction advantage exceeds the pre-committed threshold** — the chapter-06 bundle's acceptance-threshold discipline flags the release. The privacy engineer proposes mitigations; the release-assurance process decides whether to defer the release, strengthen mitigations, or accept.
- **A DSR capability gap is identified for a specific right** — if a planned release introduces a corpus that cannot be erased under the SLA, the privacy engineer flags; the release-assurance process decides whether to defer.
- **An EDPB position shifts** — the privacy engineer briefs on the implication; the release-assurance process decides whether to replan.
- **A sectoral obligation newly applies** — a product is extended to a regulated customer whose data-type brings HIPAA or GLBA into scope; the privacy engineer brings the applicable controls; release assurance decides on the scope expansion.
- **An incident reveals a capability gap** — the breach-response plan drill surfaces that a specific scenario cannot be handled; the privacy engineer proposes investment; release assurance prioritises.

In each case the privacy engineer provides the specialist judgement; the release-assurance role makes the composite decision.

## When the escalation comes release-assurance → privacy

Reciprocally, the release-assurance process puts questions back to the privacy engineer:

- **A customer's third-party-risk assessment names a specific AICM or 42001 control the current bundle under-serves** — the privacy engineer scopes the delta and proposes the work.
- **A new feature changes the data-action set** — the PRAM must be updated to cover the new action. The release-assurance process notes the feature; the privacy engineer runs the update.
- **A new jurisdiction opens** — a release into a new region brings new regulators. The privacy engineer runs the regulatory-gap analysis; release assurance incorporates.
- **A release depends on a vendor change** — a model vendor change, a cloud change, a hosted-API change. The privacy engineer updates the vendor-oversight artefacts; release assurance integrates.

## What the privacy engineer should not try to own

To be explicit — attempting to own these items at the privacy engineer's level is a scoping error:

- The release-gate pass criteria for the composite risk picture.
- The post-release monitoring architecture for non-privacy signals.
- The eval-set construction methodology for non-privacy evaluations.
- The compute and cost optimisation of the release-assurance pipeline.
- The DPO-adjacent regulatory-engagement strategy for the organisation as a whole (this is the DPO's role; the privacy engineer is a technical support).
- The product roadmap. Privacy input matters; roadmap ownership is elsewhere.

Attempts to expand the privacy engineer's scope into these areas typically happen because the broader release-assurance function is under-resourced. The right response is to name the scoping problem and advocate for the investment, not to absorb the work and burn out.

## A hand-off checklist — the artefact the privacy engineer sends on each release

For each release candidate, the privacy engineer sends to the release-assurance process:

```
Release: <system-and-version>
Target ship date: <date>
Privacy engineer: <name>
DPO: <name>

Artefacts attached:
  [ ] DPIA v<X>, signed <date>, change log current to this release
  [ ] PRAM v<X>, Worksheets 1-4 current
  [ ] EDPB / CNIL / ICO artefact set v<X>
  [ ] Sectoral cross-tag v<X> (applicable overlays: ___)
  [ ] Bundle v<X>, manifest reviewed <date>, hashes verified
  [ ] Enterprise overlay cross-tag v<X>
  [ ] Release-recommendation note v<X>

Residual risk summary:
  - Overall privacy residual risk: Low / Medium / High
  - Art. 36 prior consultation required: Y / N
  - Sectoral breach-risk escalations: Y / N (if Y, which overlay)
  - Open items blocking recommendation: <list>
  - Mitigations deferred to next release: <list with owner and date>

Recommendation:
  [ ] Proceed
  [ ] Proceed with conditions: <list>
  [ ] Block pending: <list>
```

The checklist is a one-page document that chains to the bundle. The release-assurance process consumes it alongside the parallel artefacts from other specialists.

## Summary — what you can do after this chapter

- Name the boundary between the AI privacy engineer (level 30, this track) and `ai-evaluation-engineer` (level 35) as the boundary between privacy-engineering contribution and release-assurance ownership.
- Produce the hand-off artefact set (DPIA, PRAM, EDPB/CNIL/ICO artefacts, sectoral cross-tag, bundle, enterprise overlay, release-recommendation note) that release assurance expects on each release.
- Escalate to release assurance when the privacy residual-risk determination, the measured MIA / extraction advantage, the DSR capability gap, or a regulatory / sectoral change requires a release-level decision.
- Receive escalations from release assurance for control deltas, feature changes, jurisdictional expansions, and vendor changes, and respond with the scoped privacy-engineering work.
- Avoid scope creep into release-gate design, cross-domain evidence synthesis, release-monitoring architecture, and DPO-adjacent regulatory strategy; name the scoping problem rather than absorb the work.
- Ship the one-page release-recommendation note as the privacy-engineering input to the release-assurance decision.

This chapter closes the module. The curriculum's next step for a privacy engineer who wants to own release assurance at depth is to progress toward `ai-evaluation-engineer`; the curriculum's parallel step is mod-112, which covers the privacy-program and communication obligations around all of this — the subject-facing language, the regulator-facing narrative, and the internal discipline that keeps the programme operating.
