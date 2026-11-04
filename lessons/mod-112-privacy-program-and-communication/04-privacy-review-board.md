# Running a Privacy Review Board

The privacy review board is the cadence where privacy decisions are made and recorded. Chapter 01 introduced it as one of five rituals in the programme; this chapter opens the board and shows how to run it so that the decisions it produces survive audit years later and survive the memory losses of the people who attended. A review board that does not produce audit-grade minutes and a decision log that can be understood by a reviewer who was not present is a status meeting dressed in a different name.

The authoritative references:

- NIST — *Privacy Framework v1.0* (January 2020), **Govern-P** function (particularly GV.PO-P, GV.AT-P). [nist.gov/privacy-framework](https://www.nist.gov/privacy-framework).
- NIST — *AI Risk Management Framework*, **Govern** function (GOVERN 1, 2, 3, 4). [nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework).
- ISO/IEC 27701 — PIMS clauses on management review and decision records; ISO/IEC 42001 — Clause 9.3 (management review).
- ICO — *Accountability and governance* and *Senior management responsibility* guidance. [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/accountability-and-governance/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/accountability-and-governance/).
- IAPP — CIPM body of knowledge on privacy committees / boards and governance rhythms. [iapp.org/certify/cipm/](https://iapp.org/certify/cipm/).

## The board's remit — what it decides, what it does not

A team-scope privacy review board has a finite remit. The explicit statement of the remit is in the charter (chapter 01). In practice the board decides on:

- **DPIA sign-off** — initial DPIAs and material refreshes for the systems in team scope.
- **PET parameter choices** — ε / δ selection, FL mode, TEE profile, redaction thresholds, retention windows.
- **Model-card privacy-slice publication** — the external claims the team commits to.
- **Retention schedule changes** — adding stores, extending windows, decommissioning.
- **Processing-purpose additions** — a new lawful basis for an existing system, a new jurisdiction, a new user segment.
- **DSR policy exceptions** — ambiguous cases, high-sensitivity requests, items escalated from the DSR intake.
- **Control exceptions** — proposed deployments without a mandated control, with a defined duration and compensating controls.
- **Post-incident after-action acceptance** — the record from an incident after-action is acknowledged, and the LINDDUN / eval-suite deltas are scheduled.

The board does **not**:

- Make decisions that exceed team scope. Those escalate (chapter 08). Cross-product, cross-jurisdiction, or board-visible decisions go to level 50 or level 60.
- Replace the DPO's advisory duty. The DPO is a voting-equivalent voice at the board; the DPO is not the chair, not the decision-maker, not the operator.
- Replace legal counsel's privilege scope. Items that need legal privilege are flagged and handled outside the board or in an attorney-client-privilege sub-session with the minute discipline adapted accordingly.
- Replace the engineering review of technical correctness. The board decides on *privacy fitness*, not on code quality or system design in the general sense.

Being explicit about this is where you avoid the "board as everything" failure that drowns the ritual in agenda items that are not privacy decisions.

## The attendee list — who is in the room

A team-scope board is small enough to be effective and large enough to represent the partner functions (chapter 01). The default attendee list:

- **Chair (facilitator)** — the privacy engineer (team-scope). The chair runs the agenda, calls for decisions, ensures the minute discipline is kept. The chair is a facilitator, not a decider.
- **Product owner for the system(s) on the agenda** — accountable for most decisions as the controller-side representative. Must be a decision-maker, not a proxy.
- **ML lead (or engineer, depending on topic)** — brings the technical facts.
- **Platform / infra lead** — brings the deployment, logging, retention facts.
- **Security lead** — brings the threat model, incident posture, security-side mitigations.
- **DPO** — advises; the DPO's view is recorded; the DPO does not vote.
- **Legal counsel** — advises on lawful basis, jurisdictional reading, contractual terms.
- **Governance representative** — speaks to the enterprise control library and the crosswalk; may be absent at team scope if governance is small, but then the privacy engineer carries the governance connection.
- **Communications / trust & safety** — present when decisions are externally visible (model-card publication, incident after-action affecting customer narrative).

Two seats that can be observers rather than voices:

- A **customer representative** (an engineer-pair or internal customer whose workload depends on the system) when the decision affects their use.
- A **more senior privacy engineer or L50 architect** when an escalation is being pre-briefed before it goes upstream.

The size cap: around 8 to 10 named seats. Larger boards are decision-weak; smaller boards miss a perspective. If a decision needs a wider room, run a design review before the board, not a bigger board.

## The cadence — frequency and the pre-read deadline

A monthly rhythm is the common default for a team of 5 to 15 engineers shipping an AI system with quarterly-ish model retrains. Scale up to bi-weekly during a release-dense quarter; scale down to every six weeks during a maintenance period — but *name* the cadence in the charter, do not drift.

The three dates that matter:

- **Agenda-cutoff** (e.g., 7 days before the meeting) — items accepted to the agenda by this date get the pre-read and the decision; later items roll to the next board.
- **Pre-read publication** (e.g., 3 working days before) — the pre-read packet is published; attendees read before the meeting.
- **The meeting itself.**

The board that lets items onto the agenda on the day of the meeting degenerates into design review. Resist this. Items that cannot wait for the next cycle have an escalation path — most of them are incidents, which are handled by the incident response flow (chapter 05), not by a privacy review board.

## The pre-read — the document that makes the meeting work

The pre-read is the single document that distinguishes a decision-producing board from a meeting. It is published 3 working days before the board and read by every attendee before the meeting starts. The meeting is an opportunity to discuss and decide, not an opportunity to read.

The pre-read shape, per decision on the agenda:

```
# Decision brief — <decision title>
Decision class: <from the sign-off matrix>
Decision owner: <role>
Pre-read author: <privacy engineer or delegate>

## Context (half a page)
- What is the system, what has changed, why this is on the board's agenda now.
- Pointer to the DPIA / model card / runbook under review.

## Options (one paragraph per option)
- Option A: <description>. Trade-offs: <privacy>, <utility>, <cost>, <timeline>.
- Option B: <description>. Trade-offs: ...
- Option C (do-nothing / defer): <description>. Trade-offs: ...

## Measured evidence (one or two pages)
- The data that distinguishes the options. For PET parameter choices this is the accountant output
  and the utility-vs-ε curve; for a retention change this is the current-state usage data; for an
  Art. 22 posture change this is the fairness and adverse-effect measurement.
- Each number cites the artefact it came from (path, hash, commit).

## Risk register delta
- Which feared events (c-column of the DPIA) each option moves on severity / likelihood.

## Partner input
- Legal — (consulted on <date>) — <position>.
- DPO — (advised on <date>) — <position>, with any disagreement or condition noted.
- Security — <position>.
- Product — <position>.

## Recommendation
- Author's recommendation and the one-sentence reason.

## Decision requested from the board
- A specific yes-or-no or a specific choice among the options, with a date.
```

Four properties the pre-read must have:

1. **It is a document, not a slide deck.** Slides are presentation aids; the record is textual. Attach slides if useful, but the pre-read is the document.
2. **It contains the options including the do-nothing.** The board cannot ratify an option without knowing what the other options were.
3. **It cites evidence.** Every measured claim points at an artefact under version control.
4. **It names the decision requested.** "Discussion" is not a decision request. The board does not meet to discuss; it meets to decide.

If a pre-read shows up that violates any of the four, send it back and roll the item to the next board. Normalising sub-threshold pre-reads degrades the ritual to a meeting series.

## The one-pager — the executive-facing summary

A one-pager is the pre-read's summary, written for a reader who will not read the pre-read. It is produced for:

- Executive sponsors who sit on the review board as observers.
- Cross-partner leaders who did not attend but need to know the board's position.
- The decision log entry's "Context" field for a reader six months later.

The one-pager is a single page (not two, not one and a half). The shape:

```
Decision: <title>
Date decided: <date>, by <decision maker>
Why this mattered: <two sentences — the stakes if the decision were wrong>
What was chosen: <one sentence>
Why: <one paragraph — the measured trade-off>
What was rejected and why: <one short list>
What it commits us to: <the artefact changes, the SLO commitments, the published claims>
What triggers a re-visit: <list of events that bring this back to the board>
Links: <pre-read>, <DPIA §>, <model-card version>, <decision-log ID>
```

The one-pager's reader-test: *could a new engineer joining the team in six months reconstruct what we chose and why from this single page?* If not, revise.

## Running the meeting — the chair's discipline

The chair's job is to keep the meeting producing decisions. Four patterns the chair enforces:

### 1. Read-before-speak

The meeting assumes the pre-read is read. The chair opens each item with "Any clarifying questions on the pre-read?" — not with a walkthrough of the pre-read. Attendees who have not read the pre-read say so; the item rolls to the next board (public accountability keeps the discipline).

### 2. Timebox per decision

Each decision gets a timebox — 15 to 30 minutes is typical. The chair calls the timebox at the start and at the midpoint. If the discussion cannot converge inside the timebox, the chair calls a specific follow-up (more measurement, legal consultation, another option evaluation) and defers the decision to a named follow-up date.

### 3. Call the decision

After the timebox, the chair calls the decision: "Product owner, do you accept option B with the condition the DPO named?" The product owner answers yes, no, or defer. The minute-taker records the answer and the conditions. Discussion continues on the next item.

### 4. Capture the DPO's view

The DPO's position is recorded verbatim if the DPO is brief, or in the DPO's own one-paragraph summary if more detail is needed. Where the DPO's view differs from the decision made, the difference is recorded — not softened. This is the Article 38 discipline in practice (chapter 02).

## The minutes — what the record must contain

The minutes are the durable record that survives the meeting. The minimum content, per decision:

```
Decision ID: PRV-<year>-<nnn>
Date: <date>
Decision title:
Decision class: <from the sign-off matrix>
Attendees: <names and roles>
Decision made: <one sentence>
Decision maker: <role / name>
Conditions attached: <list; e.g., "re-audit MIA quarterly", "legal to confirm LIA draft by date">
Options considered:
  - Option A (chosen): <one sentence>, trade-off <one sentence>
  - Option B (rejected): <one sentence>, reason <one sentence>
  - Option C (deferred): <one sentence>, reason <one sentence>
Partner input of record:
  - DPO: <position>, agreement / disagreement, conditions
  - Legal: <position>
  - Security: <position>
  - Product: <position>
Residual-risk delta: <which feared events moved, by how much>
Evidence cited: <artefact pointers>
Review trigger: <events that bring this back>
Linked artefacts: DPIA §<ref>, model-card v<ref>, runbook <ref>
Pre-read: <path>
One-pager: <path>
```

The minutes are published within 2 working days of the meeting. Attendees have 3 working days to flag corrections. After that, the minutes are considered accepted and immutable (corrections are added as follow-up entries, not by rewriting; audit requires append-only).

### The decision log as the index of minutes

The decision log (chapter 01) is the append-only index that references the minutes. Each minute entry is a decision log row. The log is scrubbable (easy to grep) and sortable; the minute file is the long-form record that the log row cites.

Store both under version control, with access tightly controlled — the decision log is an audit artefact and tampering with it is the kind of thing a regulator notices.

## The sign-off matrix in action — the RACI on the board

Each decision class the board handles has a sign-off matrix row (chapter 01). The board's role is to be the room in which the matrix is *exercised*, not re-negotiated.

Example rows, read as "accountable / responsible / consulted / informed":

| Decision class | A | R | C | I |
|---|---|---|---|---|
| DPIA initial sign-off | Product owner | Privacy engineer | DPO, legal, security | Governance |
| PET parameter selection | Privacy engineer | ML lead | Security, DPO | Product |
| Retention schedule change | Privacy engineer | Platform lead | Legal | Security |
| Model-card privacy-slice publication | Privacy engineer | ML lead | Legal (for external wording) | Communications |
| DSR escalation (ambiguous) | Privacy engineer | DSR intake owner | Legal, DPO | Product |
| Control exception (team-scope) | Product owner | Privacy engineer | Legal, DPO | Governance |
| Processing-purpose addition | Product owner | Privacy engineer | Legal, DPO | ML, platform, security |
| Post-incident after-action acceptance | Privacy engineer | Security | DPO, legal | Product, communications |

When the matrix says Product owner is accountable, the board does not accept a product-owner proxy; the product owner is in the room or the item defers. When the matrix says the DPO is consulted, the DPO's view is captured whether or not the DPO objects to the decision. This is what transforms the board from a meeting into a governance instrument.

## The quarterly retrospective — the board reviews itself

Once a quarter, the board spends a session on itself rather than on the agenda. The retrospective asks:

- Which decisions from the last quarter would we make differently now? What have we learned?
- Which decisions were made outside the board that should have been in it? (Silent scope expansion — chapter 01.)
- Which decisions were made in the board that could have been delegated? (Theatre.)
- Where did the sign-off matrix fail — items where accountable / responsible / consulted / informed was unclear or disputed?
- What is the current open-escalation list (team-scope decisions awaiting L50 or L60 response)?
- What charter amendments does the next quarter need?

The retrospective produces a short memo that is itself in the decision log (as a meta-entry) and that may propose a charter amendment, which goes through the next board as a decision.

## The audit-readiness view of the board

A regulator or auditor can ask for the board's record at any time. The record that survives inspection has these properties:

- **Chronological completeness.** No gaps. If a month's meeting was skipped, the log says so and why.
- **Decision-maker identity.** Every decision has a named human attached.
- **Evidence traceability.** Every decision cites artefacts that still exist and still say what the decision said they said.
- **DPO engagement visibility.** Every decision class that lists the DPO as consulted has the DPO's view in the record. Where the DPO is listed as consulted but no DPO view was captured, the record is incomplete.
- **Action closure visibility.** Where a decision had conditions attached ("re-audit MIA quarterly"), the subsequent board confirms whether the condition was met.

Most audit failures of a review board are not that the wrong decisions were made; they are that the right decisions were made but the record does not prove it. The minute discipline above is what prevents that failure.

## Common failure modes of the board

To watch for in retrospectives:

- **Status-meeting creep.** Agenda items become "update on …" instead of "decide on …". Fix: enforce the pre-read's "decision requested" clause; items without a decision request are not on the agenda.
- **Pre-read neglect.** The pre-read is written the night before and no one has read it. Fix: enforce the agenda-cutoff and the pre-read publication deadline.
- **Proxy decision-makers.** The product owner sends a proxy who cannot decide. Fix: if the accountable party is not in the room, the item defers.
- **Minutes that soften disagreement.** The DPO objected; the minutes record "DPO expressed considerations." Fix: minutes record the DPO's position in the DPO's own words, verbatim or in a DPO-written paragraph.
- **One-decision-maker board.** The privacy engineer ends up being accountable for most decisions because other partners defer. Fix: run the sign-off matrix through the retrospective; if the privacy engineer is accountable on something only the product owner can own, correct the matrix.
- **Decision-log drift.** Minutes published late, log entries missing fields, links broken. Fix: the chair runs a monthly log-health check as part of the standing agenda.

## Composition with the stand-up and the retrospective

The three rituals interact:

- **Stand-up (weekly)** surfaces items; it does not decide. An item that needs a decision is added to the next review-board agenda.
- **Review board (monthly)** decides. The decision is written to the log.
- **Retrospective (quarterly)** reviews the previous three boards and the charter itself.

A decision made at the stand-up and never ratified at the board is a programme discipline failure. The board must either ratify or surface the decision; silent operational decisions that are material to privacy belong in the log.

## Summary — what you can do after this chapter

- Name the remit and non-remit of the privacy review board at team scope and distinguish board decisions from design reviews and status updates.
- Compose the attendee list from partner functions with one voice each, keeping the room small enough to decide.
- Run the agenda-cutoff / pre-read-publication / meeting cadence that makes the board decision-producing rather than discussion-producing.
- Author a pre-read that contains context, options including do-nothing, measured evidence with artefact pointers, partner input, and a specific decision request.
- Author the one-pager that summarises the pre-read for executives and for the decision-log reader six months later.
- Chair the meeting with read-before-speak, timebox-per-decision, call-the-decision, and DPO-view-of-record disciplines.
- Produce minutes that are append-only, evidence-cited, decision-maker-named, and verbatim on partner disagreement — and publish them inside two working days.
- Exercise the sign-off matrix at the board and enforce proxy-rejection and condition-tracking.
- Run the quarterly retrospective that reviews the board itself and proposes charter amendments.
- Recognise the characteristic failure modes — status-meeting creep, pre-read neglect, proxy decision-makers, minutes-softening — and apply the fix for each.

Chapter 05 picks up with the incident response flow — the out-of-band ritual that happens when the board's planning assumptions have broken.
