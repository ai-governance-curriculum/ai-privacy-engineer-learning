# AI-Specific Privacy Incident Response

A privacy incident is the point at which the programme's prevention story stops and the programme's response story begins. For an AI system, the incidents that matter include categories that a traditional data-breach playbook does not quite cover: a suspected PII leak through model output, an LLM that reproduces training data verbatim, a RAG system that crosses a tenant boundary under prompt injection, a DSR that the engineering stack cannot honour inside the statutory window. This chapter gives you the AI-specific incident-response flow — triage, reportability decision, coordination, and feedback into the LINDDUN model and the privacy-eval regression suite.

The authoritative references — read them before the first incident, not during one:

- Regulation (EU) 2016/679 (*GDPR*), **Article 33** (notification of a personal data breach to the supervisory authority), **Article 34** (communication of a personal data breach to the data subject), **Article 4(12)** (definition of "personal data breach"), **Recitals 85–88**. [eur-lex.europa.eu/eli/reg/2016/679/oj](https://eur-lex.europa.eu/eli/reg/2016/679/oj).
- Article 29 Working Party — *Guidelines on Personal Data Breach Notification under Regulation 2016/679*, WP250 rev.01, endorsed by the EDPB (2018). [ec.europa.eu/newsroom/article29/items/612052](https://ec.europa.eu/newsroom/article29/items/612052).
- EDPB — *Guidelines 09/2022 on personal data breach notification under GDPR* (updating WP250). [edpb.europa.eu](https://www.edpb.europa.eu/).
- UK ICO — *Personal data breaches* guidance. [ico.org.uk/for-organisations/report-a-breach/personal-data-breach/personal-data-breaches-a-guide/](https://ico.org.uk/for-organisations/report-a-breach/personal-data-breach/personal-data-breaches-a-guide/).
- California — *Cal. Civ. Code §§ 1798.82 and 1798.29* (breach notification obligations under California law, separate from CCPA/CPRA's privacy obligations). [leginfo.legislature.ca.gov](https://leginfo.legislature.ca.gov/).
- California — *Insurance Code § 791.26* and Attorney General breach-notification resources. [oag.ca.gov/privacy/databreach](https://oag.ca.gov/privacy/databreach).
- US HHS — *HIPAA Breach Notification Rule*, **45 CFR §§ 164.400–414**. [hhs.gov/hipaa/for-professionals/breach-notification/](https://www.hhs.gov/hipaa/for-professionals/breach-notification/).
- SEC — *Form 8-K Item 1.05 cybersecurity incident disclosure* (public-company specific). [sec.gov/corpfin/announcement/cybersecurity-rules](https://www.sec.gov/).
- NIST — *SP 800-61 Rev. 2, Computer Security Incident Handling Guide*. [csrc.nist.gov/publications/detail/sp/800-61/rev-2/final](https://csrc.nist.gov/publications/detail/sp/800-61/rev-2/final).
- NIST — *SP 800-184, Guide for Cybersecurity Event Recovery*. [csrc.nist.gov/publications/detail/sp/800-184/final](https://csrc.nist.gov/publications/detail/sp/800-184/final).

A scope note: this chapter covers *privacy* incident response. It composes with — but does not replace — the organisation's security incident response (owned by the CISO / security function) and legal's regulator liaison. The privacy engineer is a specialist voice inside the broader incident command, not a replacement for it.

## What a privacy incident actually is

GDPR Article 4(12) defines a personal data breach as:

> *a breach of security leading to the accidental or unlawful destruction, loss, alteration, unauthorised disclosure of, or access to, personal data transmitted, stored or otherwise processed.*

The WP250 guidance decomposes this into three categories:

- **Confidentiality breach** — unauthorised or accidental *disclosure* or *access* to personal data.
- **Integrity breach** — unauthorised or accidental *alteration* of personal data.
- **Availability breach** — accidental or unauthorised *loss of access* to, or *destruction* of, personal data.

For an AI system, the three categories map onto specific engineering failure modes:

| GDPR category | AI-specific incident type |
|---|---|
| Confidentiality | Prompt / completion log exposure; model output that reproduces training data; RAG cross-tenant leakage; agent tool-call exfiltration; MIA that is successful against production data; sub-processor leak. |
| Integrity | Training-data poisoning; retrieval-corpus poisoning that alters subject-facing outputs; a system change that silently corrupts identity-propagation. |
| Availability | Prolonged outage of the DSR intake; loss of a retention store that holds evidence a subject needs; irrecoverable loss of a model-version that was serving per-subject preferences. |

A fourth, AI-specific failure mode that does not cleanly fit the three but must be tracked:

- **DSR failure** — the engineering stack cannot complete a validated subject request inside the statutory window. WP250 treats delayed response as a *procedural* failing rather than always a breach under Article 4(12), but the response discipline is close enough that the incident channel is the right surface — and in some cases the DSR failure co-exists with a confidentiality or integrity breach that caused it.

Keep the AI-specific list on hand; the triage step (next section) uses it.

## The incident flow at a high level

An AI privacy incident runs through five phases. Each phase has owners, inputs, outputs, and a target duration. The phases compose with the organisation's security incident response — this is the privacy slice, not a replacement.

```
0. Signal           — hour 0
   |
1. Triage           — hour 0 to hour 2
   |
2. Containment      — hour 0 to hour 24 (overlapping with triage)
   |
3. Reportability    — hour 2 to hour 72 for GDPR; different clocks for US statutes
   decision
   |
4. Notification     — on reportability decision; may re-fire as facts evolve
   & communication
   |
5. After-action     — within 10 working days of incident closure
   & feedback
```

The sections below walk each phase.

## Phase 0 — signal, and the privacy-incident intake

Signals that trigger the privacy-incident channel:

- An automated alert from a privacy guardrail (retention-audit failure, redaction-FP spike, log-access anomaly, memorisation canary hit in production output).
- A security team hand-off (a security investigation that has turned up personal data exposure).
- A customer / user report (support ticket, CISO report from a customer, researcher disclosure).
- A sub-processor notification.
- A regulator notification.
- A DSR intake flag (request cannot be honoured in time).
- A DPIA refresh that discovered a previously unnoticed exposure.

The intake surface collects every signal into a single **suspected incident** file. A suspected incident exists until Phase 1 reclassifies it. Operating pattern: *when in doubt, open the file*. Opening a file is cheap; failing to open one and then discovering a reportable breach is the failure the audit will catch.

Where the signal comes from an external party, the receipt is also the clock start for Article 33's "without undue delay and, where feasible, not later than 72 hours **after having become aware** of it". WP250 is specific — *awareness* is when the controller has a reasonable degree of certainty that a breach has occurred that compromises personal data. The triage phase is partly about establishing that awareness threshold, and partly about escalating from suspicion to confirmation.

## Phase 1 — triage, the first two hours

Triage establishes four facts so that the right people are in the room and the right clock has started:

1. **Is this a privacy incident, a security incident, both, or neither?** A privacy incident involves personal data; a security incident involves the organisation's systems. AI incidents are often both.
2. **What personal data is affected, in what state, in what jurisdiction?** Personal data (Art. 4(1)), special-category data (Art. 9), personal health information (HIPAA PHI), personal information under CCPA — the applicable regime depends on this.
3. **What is the plausible mechanism?** The hypothesis that will drive containment and the eventual after-action.
4. **What is the scope?** How many subjects, approximately; what is the upper bound.

The output of triage is the **incident scoping note** — a short document (one page is typical) that the incident commander and the privacy engineer co-author. Its shape:

```
# Incident scoping note — INC-<year>-<nnn>
Incident commander: <security role>
Privacy lead: <privacy engineer>
DPO notified at: <timestamp>
Legal notified at: <timestamp>

Signal: <what triggered this; link to the original signal>
Awareness established at: <timestamp — this is the Art. 33 clock start>
Affected system(s): <names, with pointers to DPIA and system card>
Suspected mechanism(s) (one or more; each is a working hypothesis):
  - Hypothesis A: <description, evidence so far>
  - Hypothesis B: ...
Personal data in scope:
  - Categories: <list>
  - Special-category (Art. 9) involvement: <yes / no / unknown>
  - Jurisdictional applicability: <EU / UK / US-state / HIPAA / other>
Approximate scope:
  - Subjects affected (upper bound, with uncertainty): <number>
  - Volume (records, tokens, inferences): <number>
Containment status: <what is already stopped>
Notification clocks triggered:
  - GDPR Art. 33 — <clock start, 72-hour deadline>
  - CCPA / Cal. Civ. Code §§ 1798.82 / 29 — <applicability>
  - HIPAA Breach Notification Rule — <applicability>
  - Other (SEC 8-K, sectoral) — <applicability>
Decisions requested in the next 6 hours: <list>
```

The scoping note is the first piece of audit evidence the incident produces. It is updated as facts come in; each update is a new version in the file (append-only, never overwritten).

### AI-specific triage questions

Beyond generic breach triage, an AI privacy incident has questions that only make sense for AI systems:

- **If the exposure is a model output**, is the exposed data (a) training data verbatim or near-verbatim (memorisation hit), (b) retrieval corpus content that should not have been in that user's context (cross-tenant leak), or (c) a model-inferred attribute (inferred-attribute disclosure)? The three have different mitigation paths and different reportability analyses.
- **If the exposure is a log**, which log class — prompts, completions, intermediates, metadata — and did the redact-before-log pipeline fail (claim 2 of the model-card privacy slice, chapter 03) or was it bypassed?
- **If the trigger is prompt injection**, is the source of the injection a user, a third party via retrieval content, or a system-prompt element? Indirect prompt injection (via the retrieval corpus) is a particular AI-specific vector the OWASP LLM Top 10 names.
- **If the trigger is a model retrain**, did new training data ingress without the DPIA refresh? If so, Phase 4 includes a DPIA refresh and possibly an Art. 36 prior consultation, not only a breach notification.
- **If the trigger is unlearning failure**, is the subject's data still influencing outputs after the SISA retrain or the approximate-unlearning procedure claimed to have erased it? This may be a procedural rather than a strict-breach incident, but it is a material privacy event for the subject.

The triage runbook lists these questions explicitly so the responder does not have to generate them during an incident.

## Phase 2 — containment

Containment is run by the security incident commander; the privacy engineer contributes specifics for the AI-specific exposure. Patterns for AI systems:

- **Stop the leak path.** Disable the affected endpoint; roll back to a previous model version where feasible; quarantine the affected log store from export; pause a data-pipeline that is writing unredacted data.
- **Preserve evidence.** Snapshot the log store at the moment of discovery; archive the model version (weights + system prompt + retrieval-index version + config); checkpoint the retrieval corpus state. The forensic trail must survive remediation.
- **Isolate without destroying.** Resist the temptation to delete the exposure (log lines, exposed completions) — once deleted, you have lost the ability to notify affected subjects and to prove the scope.
- **Communicate containment to the responder set.** Status updates at an agreed cadence (hourly is typical in the first 12 hours, every 4 hours subsequently).

The privacy-engineering contribution during containment is to protect the evidence that enables the reportability decision and the subject-notification process. That means: *don't delete; don't let auto-remediation delete; don't let the security team delete out of habit*. The retention-audit discipline (chapter 01) that would normally enforce deletion has a carve-out for live incident evidence — know how to invoke it.

## Phase 3 — the reportability decision

This is the privacy-specific phase. The decision is normally made by legal and the DPO, with the privacy engineer providing the facts. The clocks in play:

### GDPR Article 33 and 34

**Article 33(1)**: *In the case of a personal data breach, the controller shall without undue delay and, where feasible, not later than 72 hours after having become aware of it, notify the personal data breach to the supervisory authority competent in accordance with Article 55, **unless the personal data breach is unlikely to result in a risk to the rights and freedoms of natural persons**.*

**Article 34(1)**: *When the personal data breach is likely to result in a **high risk** to the rights and freedoms of natural persons, the controller shall communicate the personal data breach to the data subject without undue delay.*

The two thresholds — "likely to result in a risk" (triggers Art. 33) and "high risk" (triggers Art. 34) — are the WP250 analysis. For an AI incident, the risk-to-rights assessment uses the same severity × likelihood framing the DPIA does (mod-111 ch. 01), applied to the actually-exposed data in the specific incident.

WP250's indicative criteria for the risk analysis (summarised; the guidance is the authoritative source):

- **Type of breach** — confidentiality / integrity / availability. Confidentiality breaches involving sensitive data trend toward high risk.
- **Nature, sensitivity, and volume of personal data** — special-category data, financial data, large numbers of subjects.
- **Ease of identification** — direct identifiers vs. pseudonymised data; data that can be re-identified.
- **Severity of consequences for individuals** — risk of identity fraud, financial loss, reputational damage, discrimination.
- **Special characteristics of the individual** — vulnerability, child subjects.
- **Special characteristics of the data controller** — healthcare, financial, public body.
- **Number of affected individuals.**

For AI-specific incident types, the risk analysis adapts:

- **Training-data extraction exposing an identifiable record** — if the record is identifiable and the data is sensitive, this trends toward Art. 34 territory. The number of subjects affected may be small (the specific identified subject) but severity is high.
- **Cross-tenant RAG leak** — risk depends on what crossed the boundary. A business-document leak between two B2B tenants affects two legal entities; the affected natural persons are the individuals named in the documents. Scope and severity depend on the specific content.
- **Prompt / completion log exposure** — depends on what the logs contained. If redact-before-log was in place (claim 2 of chapter 03) and worked, the exposure is pseudonymised and lower-risk; if it bypassed, the exposure may be severe.
- **DSR failure to meet the window** — WP250 treats this as a procedural failing rather than a breach under Art. 4(12) in most cases; however, the facts matter, and some DSR failures co-exist with a confidentiality or integrity breach.

The reportability decision is captured in writing:

```
# Reportability decision — INC-<year>-<nnn>
Decision made by: <legal lead> and <DPO>
Decision owner (for the record): <controller representative — product owner at team scope>
Date and time: <timestamp>
Awareness established at: <from scoping note>
GDPR Art. 33 — notify SA? <yes / no>, with reason
GDPR Art. 34 — notify data subjects? <yes / no>, with reason
Risk analysis (summary):
  - Likely to result in a risk: <yes / no>, factors cited
  - If yes: likely to result in a high risk: <yes / no>, factors cited
Jurisdictional applicability and other clocks:
  - UK ICO: <same as GDPR>
  - Cal. Civ. Code 1798.82 / 29: <trigger assessment>
  - HIPAA Breach Notification Rule (45 CFR 164.400): <trigger assessment, including the safe-harbor
    encryption criterion>
  - SEC 8-K Item 1.05: <materiality assessment — this is a legal / executive call>
  - Sectoral or contractual notification clocks: <list>
Supervisory authority to notify (if applicable): <authority, with reason — lead SA under Art. 56>
Target notification time: <timestamp — within 72 hours of awareness>
Draft notification: <path>
Evidence pack supporting the notification: <path>
```

### US-state and federal breach notification

Fifty US states plus DC have breach notification laws; the trigger definitions differ. For California — the most consequential for most AI products — Cal. Civ. Code §§ 1798.82 (for private entities) and 1798.29 (for state agencies) require notification to California residents whose *personal information* (as defined in those sections) was, or is reasonably believed to have been, acquired by an unauthorised person. The California AG's office maintains breach reporting pages and publishes breach reports over 500 residents on their site.

HIPAA's Breach Notification Rule (45 CFR 164.400–414) applies where protected health information (PHI) held by a covered entity or business associate is involved:

- Notification to affected individuals: within 60 days of discovery.
- Notification to HHS: within 60 days for breaches affecting ≥ 500 individuals; annually for smaller breaches.
- Notification to media: within 60 days where ≥ 500 residents of a state or jurisdiction are affected.
- **Safe-harbor exception**: unsecured PHI has specific technical definitions in the HHS guidance; encryption at the levels specified by HHS can place data outside the "unsecured PHI" scope and remove the notification obligation. The privacy engineer must know whether the data at the moment of exposure was in a form covered by the safe harbor.

Public companies subject to SEC regulation have Form 8-K Item 1.05 disclosure obligations for *material* cybersecurity incidents, generally within four business days of determining materiality. The materiality determination is a legal and executive call; the privacy engineer contributes the facts.

Contractual notification obligations (to customers, to sub-processors, to insurers) often sit alongside the statutory ones and have their own clocks. Legal owns the inventory; the incident file references it.

### The decision tree — one page the responder can hold

For the privacy engineer, the useful artefact is a decision tree the responder can walk at 2 AM:

```
1. Is personal data affected?
   - No  → close the privacy-incident file; feed back to security as a security-only incident.
   - Yes → continue.

2. Is PHI affected (HIPAA scope)?
   - Yes → engage the HIPAA breach-notification workflow; check the safe-harbor encryption
           criterion; consider the 60-day individual, 500-person HHS, and media triggers.
   - Either way → continue.

3. Is California personal information affected (Cal. Civ. Code § 1798.82)?
   - Yes → engage the California notification workflow; note the attorney-general report
           threshold (> 500 residents).
   - Either way → continue.

4. GDPR analysis:
   a. Is this a personal data breach (Art. 4(12))?
      - No  → not Art. 33, document reasoning.
      - Yes → continue.
   b. Is the breach likely to result in a risk to the rights and freedoms of natural persons?
      - No  → not Art. 33 notifiable, document reasoning.
      - Yes → notify SA within 72 hours of awareness (Art. 33).
   c. Is the breach likely to result in a high risk?
      - No  → not Art. 34 notifiable, document reasoning.
      - Yes → notify data subjects without undue delay (Art. 34).

5. Other applicable regimes (SEC 8-K, sectoral, contractual): engage per legal's inventory.

6. Prepare and send notifications; coordinate communications (Phase 4).
```

The tree lives in the incident runbook; walking it is a documented activity in Phase 3's evidence record.

## Phase 4 — notification and communication

The notification artefact itself has statutory content requirements. GDPR Article 33(3) names what must be in the supervisory-authority notification; Article 34(2) names what must be in the data-subject communication. The privacy engineer provides the technical narrative; legal does the drafting. The supervisory-authority notification can be done in stages when all the facts are not known in time ("phased notification" — Art. 33(4)).

Content the notification carries, mapped to Art. 33(3):

| Art. 33(3) element | What the privacy engineer provides |
|---|---|
| (a) nature of the breach, including categories and approximate number of data subjects and records | The scoping note's affected-data categories and the measured upper-bound counts. |
| (b) name and contact of the DPO | Standing information (not per-incident). |
| (c) likely consequences | The risk analysis from Phase 3. |
| (d) measures taken or proposed to address the breach | The containment and remediation actions, with estimated completion. |

For the Art. 34 data-subject communication, Article 34(2) requires the same but *in clear and plain language*. The privacy engineer's input here is to ensure the technical facts are translated accurately without losing the material specifics. "Our systems were affected by a security incident" is rarely sufficient; the affected subject is entitled to know what data of theirs is exposed and what they should do.

Communications (the function, not the ritual) coordinates the external narrative. The discipline the privacy engineer brings: *the communications draft must not contradict the notification*. A public statement that minimises what the Art. 33 filing concedes is the kind of divergence that turns a reportable incident into a regulator action.

## Phase 5 — after-action and feedback

Within 10 working days of the incident being considered closed (containment complete, notifications sent, subject queries processed), the team runs an after-action. This is not optional for anything above a minor incident.

The after-action produces a structured record that feeds four downstream artefacts:

1. **The incident file archive** — the full chronology, decisions made, evidence captured, lessons learned. Retained for the longer of local regulatory retention requirements and internal incident-retention policy (often seven years).
2. **The LINDDUN model update** — what threat did the LINDDUN model miss (mod-102)? The after-action proposes LINDDUN deltas and names an owner to work them.
3. **The privacy-eval regression suite update** — what test did the eval suite not have that would have caught this? The after-action proposes new tests (memorisation canary type, redaction case, cross-tenant negative test, retention-audit rule) and schedules them.
4. **The DPIA refresh if needed** — did the incident reveal a feared event not in the DPIA's risk register? Refresh the DPIA; re-assess residual risk.

The after-action shape:

```
# After-action — INC-<year>-<nnn>
Date closed: <date>
Date of after-action: <date>
Participants: <list>

## Timeline
<minute-by-minute or hour-by-hour, keyed to the scoping note's timestamps>

## What happened — the final narrative
<two to four paragraphs — the story the organisation will tell itself>

## What the detection missed
<what signal arrived late, what signal was present but ignored, what signal was absent>

## What the containment missed
<what evidence was almost destroyed, what action took too long, what coordination gap appeared>

## What the reportability decision got right / wrong
<retrospective on the Phase 3 decision>

## What the LINDDUN model missed
<threats the LINDDUN model did not capture; proposed deltas; owner>

## What the eval suite missed
<tests that would have caught this; proposed additions; owner>

## What the DPIA must now reflect
<risk-register deltas; mitigation deltas; residual-risk re-assessment>

## Charter / runbook changes proposed
<edits to the incident runbook, the programme charter, the sign-off matrix>

## Dates for follow-up verification
<when we check that the deltas are landed>
```

The after-action is reviewed at the next privacy review board (chapter 04) as a decision-of-record, and the LINDDUN / eval-suite / DPIA deltas become board-tracked action items.

## Coordination during an incident — who talks to whom

An AI privacy incident has too many stakeholders to run on ad-hoc coordination. The incident file names the roster:

- **Incident commander** — security (usually), owns the overall response; the privacy engineer is a specialist voice reporting into the commander.
- **Privacy lead** — the AI privacy engineer, owns the privacy-specific analysis and the facts underlying the reportability decision.
- **DPO** — advises on reportability, is the regulator contact (Art. 39(1)(e)); must be notified at triage, not at Phase 3.
- **Legal counsel** — advises on reportability, drafts notifications, manages contractual notification obligations.
- **Communications** — manages external narrative when the incident goes public.
- **Executive sponsor** — the controller-side authority for board-visible incidents; this is the sign-off for the Art. 33 filing and for any SEC 8-K materiality determination.
- **Affected product / team leads** — operate containment and remediation inside their systems.
- **Sub-processor contacts** — where a sub-processor is involved, their incident contact per the DPA.
- **Regulator liaison counsel** — brought in where the matter is heading into formal regulator engagement.

The healthy pattern: all written communication during the incident is in a dedicated channel and all key decisions are written down and signed at the time, not reconstructed after the fact. Verbal decisions that are not transcribed into the incident file are invisible to the audit later.

## The three incident scenarios every AI privacy programme must rehearse

The incident runbook is not a document; it is a document plus a set of rehearsals. The three scenarios below are the minimum rehearsal set — exercise 05 for this module runs one of them.

### Scenario A — a memorisation leak

A customer reports that a model completion contains the name, email, and medical condition of a person they do not know. The team reproduces the prompt; the completion is reproducible. Investigation shows the triggering prompt is a common stem, and the exposed data appears to be a near-verbatim training-corpus match. The team must decide: how many subjects are affected (one? several? more?), is the leak ongoing (is the model still live?), what is the reportability posture, and what communication is appropriate.

Privacy-engineering role: establish the extraction surface, estimate the affected-subject population, author the Art. 33(3) narrative, contribute to the model-card privacy-slice update, and feed the eval suite a new memorisation canary.

### Scenario B — a cross-tenant RAG leak

A tenant reports that their internal chat assistant returned a document that is not theirs. The team reproduces; a specific prompt triggers retrieval of another tenant's document. Investigation points at a configuration error in the retrieval ACL for a recent deployment. The team must decide: how many tenants and documents are affected, whether re-indexing resolves the exposure, and how to notify the affected tenants and their data subjects.

Privacy-engineering role: trace the exposure through the retrieval logs, bound the exposure set, coordinate with legal on the contractual notification obligations, and feed the eval suite a new cross-tenant negative test.

### Scenario C — a DSR that cannot be completed in time

A subject submits an erasure request; the DSR intake accepts and validates. The engineering stack cannot complete the erasure inside the GDPR one-month window (the model is not retrained on demand; the retrieval corpus erasure requires a cross-system scan that has failed twice). The team must decide: is this a procedural failure, is it a breach, what to communicate to the subject under Art. 12(3), and how to close the engineering gap.

Privacy-engineering role: establish the gap, communicate the extension and the reasons under Art. 12(3), feed the DSR runbook with the operational fix, and consider whether the unlearning capability statement (claim 7, chapter 03) needs an update.

## The scaling boundary — when the team-scope programme escalates

Some incidents exceed team scope and must escalate upstream (chapter 08):

- **A regulator has made contact** — the matter goes through the DPO and legal; the level 60 head of AI governance is informed and often the executive point.
- **The incident is cross-product or cross-jurisdiction** — the senior AI governance architect (level 50) coordinates the response across product lines.
- **The incident is material for public disclosure (SEC 8-K)** — executive leadership and the board communications function take over external messaging; the privacy engineer continues to supply facts.
- **The incident has criminal-law implications** — legal and (sometimes) outside counsel own the engagement with law enforcement.

Knowing where your scope ends is part of running the response competently. Do not attempt to manage a cross-jurisdictional, regulator-active, public-company-material incident out of a team-scope response.

## Common failure modes in AI privacy incident response

- **Awareness denial.** The team hopes the signal is a false positive and does not start the clock. Fix: a rule that any credible signal opens the file, even if the file is later downgraded.
- **Containment overreach.** The team deletes the exposed logs during containment, destroying the evidence needed for the Art. 33 notification. Fix: the runbook's "preserve evidence" step.
- **Siloed triage.** Security runs the triage without the privacy lead; the privacy facts are discovered too late. Fix: the DPO and privacy lead are on the triage call from hour zero.
- **Notification-contradiction with communications.** The Art. 33 filing says one scope, the external statement says a smaller scope. Fix: communications and legal review each other's drafts before send.
- **No after-action.** The team closes the incident and moves on. Fix: the after-action is a board-visible commitment; missing it is an audit flag.
- **DPIA drift post-incident.** The after-action recommends a DPIA refresh; three months later the DPIA has not been refreshed. Fix: the after-action's deltas are tracked at the review board until closed.
- **Rehearsal avoidance.** The team runs the response for the first time during an actual incident. Fix: the quarterly rehearsal is non-optional.

## Summary — what you can do after this chapter

- Define a personal data breach in the three GDPR categories (confidentiality, integrity, availability) and map each onto AI-specific incident types.
- Run the five-phase flow (signal, triage, containment, reportability, notification, after-action) with the artefacts each phase produces.
- Walk the reportability decision tree across GDPR Art. 33 and 34, CCPA / Cal. Civ. Code breach statutes, HIPAA Breach Notification Rule, and SEC 8-K Item 1.05.
- Author the scoping note, the reportability decision record, the Art. 33(3) notification content, and the after-action record.
- Coordinate with the DPO (the regulator contact), legal (drafting), security (incident commander), communications (external narrative), and the executive sponsor (sign-off) through the written incident channel.
- Feed the LINDDUN model, the privacy-eval regression suite, and the DPIA with the lessons from each incident.
- Recognise the escalation triggers (regulator contact, cross-jurisdiction, public-disclosure materiality) that take the incident above team scope.
- Rehearse the three canonical AI incident scenarios (memorisation leak, cross-tenant leak, DSR failure) at least quarterly.

Chapter 06 picks up with the evidence crosswalk — how the artefacts that incidents, DPIAs, and model cards produce serve more than one audit at once.
