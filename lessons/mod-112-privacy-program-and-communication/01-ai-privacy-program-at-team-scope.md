# Owning an AI Privacy Program at Team Scope

The preceding eleven modules produced the technical artefacts an AI privacy engineer makes — the threat model, the data inventory, the redaction pipeline, the memorisation audit, the DP accountant output, the FL protocol, the TEE attestation, the RAG system card, the unlearning capability, the DPIA and regulator bundle. This chapter is where you stop producing artefacts in isolation and start running them as a *programme* — a recurring, cross-functional operating rhythm that your team, your partner functions, and your executive sponsors can rely on.

An AI privacy programme at level 30 is scoped to **a team** — one product line, one ML platform, one business unit — not an enterprise. Enterprise control-library architecture belongs to the senior AI governance architect (level 50; see chapter 08); board-level reporting belongs to the head of AI governance (level 60). Your remit is the operating cadence that makes that larger apparatus work for the team you own. The deliverable this chapter produces is a **programme charter** — a one-page document that names the cadence, the artefacts the cadence produces, the decision log that records what was decided, the sign-off routes that make decisions enforceable, and the owners the charter binds to each.

The authoritative references you work from:

- NIST — *Privacy Framework v1.0* (January 2020), especially the **Govern-P** function and the **Control-P** function. [nist.gov/privacy-framework](https://www.nist.gov/privacy-framework).
- NIST — *AI Risk Management Framework* (AI RMF 1.0, January 2023), **Govern** function. [nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework).
- ISO/IEC 27701:2019 — *Security techniques — Extension to ISO/IEC 27001 and ISO/IEC 27002 for privacy information management — Requirements and guidelines* (PIMS), clauses on the privacy information management system and the roles and responsibilities a PIMS assigns. [iso.org/standard/71670.html](https://www.iso.org/standard/71670.html). (ISO/IEC 27701:2025 is the revised edition; both are cited where organisations straddle the transition.)
- ISO/IEC 42001:2023 — *Information technology — Artificial intelligence — Management system* (AIMS), Clause 5 (Leadership), Clause 9 (Performance evaluation), Clause 10 (Improvement). [iso.org/standard/81230.html](https://www.iso.org/standard/81230.html).
- IAPP — *Privacy Operational Life Cycle* framework (the CIPM body of knowledge treats operational privacy as a lifecycle: assess, protect, sustain, respond). [iapp.org/certify/cipm/](https://iapp.org/certify/cipm/).

## What "programme" means — the five commitments

A privacy programme at team scope is not a backlog of tickets. It makes five explicit commitments that distinguish it from ad-hoc privacy work:

1. **A cadence** — there is a recurring meeting (or ritual) where privacy decisions are made, with a published schedule and a standing agenda.
2. **An artefact set** — each cadence produces and updates a defined set of documents (DPIA, model-card privacy slice, decision log, risk register, incident file). The programme's state lives in those artefacts, not in Slack or chat threads.
3. **A decision log** — every material privacy decision is captured in a durable record with a date, an owner, a rationale, the option chosen, the options rejected, and the evidence cited. The log is the audit trail.
4. **A sign-off routing matrix** — for each class of decision, the charter names who signs, who is consulted, who is informed, and what it takes to escalate. Ambiguity about who signs is a programme failure.
5. **An escalation path** — when a decision exceeds team scope (cross-product, cross-jurisdiction, board-visible, regulator-engaging), the charter names the destination (level 50 or level 60) and the hand-off packet shape.

Any one of the five missing breaks the programme. A cadence without a decision log produces discussions that no one can audit; a decision log without a sign-off matrix produces decisions no one is accountable for; a sign-off matrix without an escalation path produces team-scope decisions that silently exceed their scope. The charter the exercise for this module produces is the document that commits all five.

## The partner functions — who sits at the table

An AI system touches enough of a company that a privacy programme that runs only out of your team will fail. The partners you must engage, with what each owns:

- **Product management** — owns the purpose, the user commitments, the roadmap. The privacy programme's input to product is the per-purpose lawful-basis map and the by-design constraints; the privacy programme's output from product is the schedule of system changes that trigger DPIA refresh.
- **ML engineering / research** — owns the training data choice, the model architecture, the training run. The privacy programme's input is the minimisation checkpoints, the DP / FL / TEE decision, the memorisation-audit requirement; the output is the model card's privacy slice.
- **Platform / infra / SRE** — owns the deployment surface, the log pipelines, the retention controls, the access control. The privacy programme's input is the logging policy and the retention schedule; the output is the audit of actual retention versus policy.
- **Security** — owns the threat model's security half, the pentest, the breach-detection controls, and (in most organisations) the incident-response commander role. The privacy programme's input is the privacy-specific feared-event register (mod-102); the output is the privacy slice of the incident-response playbook (chapter 05).
- **Governance (AI governance, data governance, if separate)** — owns the enterprise control catalogue, the risk appetite statement, the policy library. The privacy programme's input is evidence from the team's artefacts; the output is the mapping from the team's artefacts into the enterprise catalogue.
- **Legal** — owns the lawful-basis analysis, the contractual terms with processors and sub-processors, the regulator engagement. The privacy programme's input is the facts of the processing; the output is a documented lawful-basis analysis legal can sign.
- **DPO** (where designated under GDPR Art. 37) — advises and monitors; is independent from the controller. The DPO's role is covered in chapter 02; the one sentence here is that the DPO is a *partner*, not a reporting line up or down. The privacy engineer briefs the DPO on technical artefacts; the DPO's advice is recorded (not ignored and not laundered).
- **Communications / trust & safety** — owns external-facing narrative when something goes public (an incident, a regulator action, a customer-facing commitment). The privacy programme's input is the technical truth; the output is the external statement, reviewed for technical accuracy before release.

The RACI shape the programme charter commits to is covered later in this chapter. The first discipline is naming the partners and the artefacts that flow between you — before the first cadence, before the first decision log entry.

## The cadence — ritual, frequency, and what each ritual produces

A healthy team-scope programme runs something close to the following cadence. The specific frequencies are illustrative — adapt to deployment pace and risk level, but *name* a cadence in the charter rather than relying on "as needed":

| Ritual | Frequency | Participants | Standing agenda | Artefact produced |
|---|---|---|---|---|
| **Privacy stand-up** | Weekly | Privacy engineer, ML lead, platform lead, security lead | Open incidents, DSRs in flight, DPIA refreshes queued, next week's risk | Stand-up notes appended to decision log |
| **Privacy review board** | Monthly (or per-release in a slow-release org) | Privacy engineer (facilitator), product, ML, platform, security, governance, legal, DPO | Pre-read deck; specific decisions on queue; minutes | Decision log entries, signed off |
| **DPIA refresh review** | Per material change; quarterly backstop | Privacy engineer, ML lead, DPO, legal | Trigger analysis, scope deltas, residual-risk re-estimate | DPIA version bump |
| **Model card re-publication** | Per model release, plus quarterly backstop | Privacy engineer, ML lead | Privacy slice diff since last version | Published model card version |
| **Incident after-action** | Within 10 working days of a privacy incident closing | Privacy engineer, security, legal, DPO, affected product owner | Timeline, decisions made, what the LINDDUN model missed, what the eval suite missed | After-action record, LINDDUN update, eval-suite ticket |
| **Programme retrospective** | Quarterly | Programme owner + all partner functions | Previous quarter's decisions revisited, SLAs met vs. missed, charter deltas | Charter amendment, backlog deltas |
| **Audit-readiness review** | Semi-annually, plus pre-audit | Privacy engineer, governance, legal | Evidence pack walkthrough, crosswalk check (chapter 06), sign-off currency | Crosswalk refresh, outstanding-evidence ticket set |

Two notes on this cadence that distinguish a programme from a meeting series:

- **Each ritual has a specific decision class.** The review board decides; stand-ups surface issues but do not decide. Collapsing the two produces decisions no one remembers making.
- **Each ritual produces a durable artefact.** If the ritual did not produce a document, the ritual did not happen for audit purposes. "We discussed it" is not evidence.

## The artefact set — the programme's state

A team-scope programme's running state lives in a defined set of documents, each with an owner and a known location. The minimum set, each of which is cross-referenced from the others:

- **The programme charter** (this chapter's exercise output) — one page, lists cadence, artefacts, decision log, sign-off routes, partner RACI, escalation path.
- **The decision log** — append-only record of material privacy decisions (chapter 04 covers the schema).
- **The risk register** — the (c)-column of the DPIA (mod-111 ch. 01), kept current, scored on severity × likelihood, cross-referenced to mitigations.
- **The DPIA(s)** — one per system, versioned (mod-111 ch. 01 and 02).
- **The model-card privacy slices** — one per model, versioned (chapter 03 of this module).
- **The system cards** — one per deployed system (mod-109 ch. 06).
- **The retention schedule** — table mapping each data store to its retention window, its lawful-basis citation, and its enforcement mechanism.
- **The access matrix** — who can read, write, export each data store, with the review date and the reviewer.
- **The DSR runbook set** — one per right (mod-110 ch. 01).
- **The incident file** — the active incident file plus the after-action archive (chapter 05).
- **The evidence crosswalk** — the mapping from these artefacts to NIST PF, 27701, 42001, CSA AICM (chapter 06).
- **The credentials portfolio plan** — individual (per engineer on the team) and team-level (chapter 07).

Each artefact has a known path, a known owner, and a known review cadence. "I know where it is" is not enough — a team-mate, a new hire on week one, the DPO, or an inspector must be able to find it without you.

## The decision log — the audit backbone

A decision log is an append-only record of material privacy decisions. It is *the* audit backbone of the programme — the artefact a regulator will ask for first after the DPIA. A minimum schema (chapter 04 covers the review-board variant in more detail):

```
Decision ID: PRV-2026-0042
Date: 2026-04-11
Decision title: Approve DP-SGD ε = 3.0, δ = 1e-5 for recommendation model v4
Decision class: PET parameter selection
Owner: <privacy-engineer handle>
Decision maker: <sign-off role per matrix>
Pre-read: docs/prb/2026-04-11/pre-read.md
Option considered 1: ε = 1.0 (rejected — utility regression > 15 pp)
Option considered 2: ε = 3.0 (chosen)
Option considered 3: ε = 8.0 (rejected — exceeds team's ε cap in charter)
Evidence cited: runs/recsys-v4/dp-accountant.json; MIA audit mia-recsys-v4.md
Partner input:
 - DPO (consulted): advised quarterly MIA re-run, recorded
 - Legal (consulted): confirmed consistency with privacy notice
 - Security (informed): no objection
Residual-risk delta: feared-event FE-003 (membership inference) severity unchanged, likelihood N→L
Review trigger: next model retrain; or Opacus accountant version bump
Linked artefacts: DPIA §4 row R-14, model-card v4 §Privacy
```

The three disciplines the log enforces, each of which is a hard-won lesson:

- **Rejected options are recorded.** A decision log that records only the chosen option is a notarisation, not a decision log. The reviewer's job six months later is to re-examine whether the right option was chosen; that requires seeing what else was on the table.
- **Evidence is cited by path, not summarised.** "The DP accountant ran fine" is not citable; `runs/recsys-v4/dp-accountant.json` is. The log points at the evidence; the evidence is checked in.
- **Partner input is attributed.** "Legal was okay with it" is not usable; "Legal (consulted): confirmed consistency with privacy notice" is. The RACI distinction (responsible, accountable, consulted, informed) must be visible in the log.

A decision log that meets these three disciplines passes audit. A log that merely records that a decision was made does not.

## Sign-off routing — the RACI matrix the charter commits to

The sign-off matrix is where the programme decides *in advance* who signs what. Deciding at the moment of the decision produces delay, politics, and audit-weak outcomes. The matrix is a two-dimensional table: decision class on one axis, partner function on the other, RACI letter in each cell.

Decision classes the matrix must cover, at minimum:

- **Lawful-basis determination for a new processing purpose** — accountable: legal; responsible: privacy engineer; consulted: DPO; informed: product, ML.
- **DPIA sign-off (initial)** — accountable: controller (product owner at team scope); responsible: privacy engineer; consulted: DPO; informed: legal, security, governance.
- **DPIA sign-off (refresh)** — same, slightly lighter DPO engagement where the refresh is scoped.
- **PET parameter choice (ε / δ, FL mode, TEE attestation profile, redaction threshold)** — accountable: privacy engineer; responsible: ML lead; consulted: security, DPO; informed: product.
- **Model-card privacy-slice publication** — accountable: privacy engineer; responsible: ML lead; consulted: legal (for external-facing wording); informed: communications.
- **Retention schedule change** — accountable: privacy engineer; responsible: platform lead; consulted: legal; informed: security.
- **DSR escalation (volume spike, ambiguous request, cross-jurisdiction)** — accountable: privacy engineer; responsible: DSR intake owner; consulted: legal, DPO; informed: product.
- **Incident reportability decision (Art. 33 / CCPA / HIPAA)** — accountable: DPO (or legal, depending on structure); responsible: privacy engineer (facts) + security (facts); consulted: communications, regulator-liaison counsel; informed: executive sponsor. (Chapter 05 covers the decision tree.)
- **PET / control exception (deploying without a mandated control for a defined period)** — accountable: head of AI governance (level 60); responsible: privacy engineer (packaging the exception request); consulted: legal, DPO. **This escalates out of team scope**; the charter documents the hand-off.
- **Cross-jurisdiction reconciliation (a decision that behaves differently in EU vs. US vs. UK)** — escalates to level 50 (senior architect) per chapter 08.

The matrix is not a bureaucracy — it is the device that makes "who owns this" not a question. The first time you need it is when an incident is unfolding at 2 AM; the matrix must be legible then, which means it must be legible now.

## Partner RACI — the running relationship, not the RACI letter

The RACI letter is the sign-off shape. The running relationship is separate: how often you meet, what you brief, what they brief. Ground-truth running relationships the team-scope programme owner should run:

- **With the DPO — monthly 1:1 plus review-board attendance.** The DPO is independent; your relationship is informational, not managerial in either direction. Brief: artefact deltas, incoming DPIAs, incidents, DSR patterns. Receive: regulator shift, DPO's consultation priorities, DPO's view on residual risk. Chapter 02 covers this in depth.
- **With legal — standing weekly or bi-weekly working time.** Privacy law is lawyer territory; you are the engineer. Brief: facts, measurements, feasibility. Receive: current jurisdictional reading, upcoming law shifts, reg-engagement signals.
- **With security — joint incident-response rehearsal quarterly, plus participation in security review of ML systems.** You and security share an incident surface. Rehearse together.
- **With governance — quarterly crosswalk check.** Governance runs the control library at enterprise scale. Your crosswalk (chapter 06) is the API between your team's artefacts and the library.
- **With ML engineering — embedded, not consultative.** The AI privacy engineer belongs in the ML planning meetings. Privacy as a review gate at the end of the sprint is a programme failure.

## The failure modes a programme must guard against

A programme fails in characteristic ways that the charter can forestall. Name them in the retrospective and look for them:

- **Theatre.** The cadence runs, minutes are kept, but decisions are not made or are made outside the cadence. Sign: the review-board meeting consists of status reports and ends on time with nothing decided.
- **Ownership inflation.** The privacy engineer becomes the single point of accountability for everything, including things only legal or product can actually decide. Sign: the decision log is 95% signed by the privacy engineer.
- **Evidence backlog.** Decisions are made but the supporting evidence drifts — PET parameters updated but the DP accountant output not re-saved, retention policy updated but the retention schedule not. Sign: the audit-readiness review finds evidence that lags decisions by months.
- **Silent scope expansion.** The team ships a system variant in a new jurisdiction or a new user segment without the DPIA refresh. Sign: the system map diverges from the DPIA system description.
- **Cadence drift.** The weekly stand-up becomes bi-weekly, then monthly, then dormant, with no corresponding re-work of the charter. Sign: the stand-up notes dwindle, incidents start arriving as surprises.
- **Escalation phobia.** Team-scope decisions silently include things that exceed the scope (cross-jurisdiction reconciliation, board-visible risk, regulator-visible commitments). Sign: level 50 or level 60 is surprised by a commitment the team made. Chapter 08 covers the escalation triggers you must respect.

The retrospective (quarterly) is where you catch these. The charter amendment the retrospective produces is where you fix them.

## The programme charter — the one-page deliverable

Exercise 01 produces a programme charter. The one-page shape it commits to:

```
# Privacy programme charter — <team / product line>
Owner: <privacy engineer handle>, with executive sponsor <role>
Scope: <system(s) and purpose(s) in scope; what is explicitly out of scope>
Cadence:
  - Weekly stand-up: <day, time, standing agenda>
  - Monthly review board: <day, time, pre-read deadline>
  - Quarterly retrospective: <month week>
  - Per-release / per-change rituals: <triggers>
Artefact set:
  - Decision log: <path>
  - Risk register: <path>
  - DPIA(s): <paths>
  - Model-card privacy slices: <paths>
  - Retention schedule: <path>
  - Access matrix: <path>
  - DSR runbooks: <path>
  - Incident file (active): <path>
  - Evidence crosswalk: <path>
Partner RACI matrix: <path to full matrix; this page lists classes>
Sign-off escalation:
  - Team-scope decisions: <sign-off role>
  - Cross-product / cross-jurisdiction: escalate to level 50 <role>
  - Board-visible / regulator-engaging: escalate to level 60 <role>
Current open escalations: <list>
Version: v<n>, <date>, <reviewer>
```

The charter is a living document — it belongs in version control, with a change log. A charter that has not been edited in a year signals either perfect stability or programme drift. In practice, almost always the latter.

## Summary — what you can do after this chapter

- Name the five commitments that distinguish a privacy programme from ad-hoc privacy work: cadence, artefact set, decision log, sign-off routing, escalation path.
- Enumerate the partner functions (product, ML, platform, security, governance, legal, DPO, communications) and the artefacts that flow between you and each.
- Propose a team-scope cadence that pairs each ritual with its decision class and its durable artefact.
- Structure a decision log entry that records rejected options, cites evidence by path, attributes partner input, and links to artefacts.
- Build a sign-off RACI matrix that covers the full set of team-scope decision classes and names the escalation destinations for decisions that exceed team scope.
- Recognise the characteristic failure modes — theatre, ownership inflation, evidence backlog, silent scope expansion, cadence drift, escalation phobia — and plan the retrospective that catches them.

Chapter 02 picks up with the DPO partnership — the statutory role the programme must coordinate with, not report to.
