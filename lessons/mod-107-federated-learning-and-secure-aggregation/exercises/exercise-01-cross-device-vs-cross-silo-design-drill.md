# Exercise 01: Cross-Device vs. Cross-Silo Design Drill

**Estimated effort:** 2 hours · **Output:** `fl-shape-drill/` directory with three completed design memos, a critique of a provided design doc, and the shape-decision defence paragraph · **Primary chapters:** 01

## Objective

Internalise the cross-device vs. cross-silo decision as the first and most expensive design choice in any FL project. You will classify three realistic business scenarios, critique a design doc that has the shape wrong, and produce the defence paragraph a privacy engineer reads out at a kick-off. This exercise is a pre-code exercise — it forces the decision-order before the implementation of exercises 02–06.

## Prerequisites

- Chapter 01 read.
- Working knowledge of GDPR controller / processor concepts (chapter 07 will deepen; a working definition is enough here).
- No code or special environment required.

## Problem statement

You are the privacy-engineer lead on a team that is running three internal design reviews this week, plus one external consulting engagement. Each has a decision to make; you will produce a decision memo for each and then critique a fourth design.

### Scenarios

#### Scenario A — "Smart Ring Sleep Model"

A consumer wearable company (12 million shipped rings, iOS + Android companions) wants to train a next-generation sleep-stage classifier. Each ring collects PPG, movement, and temperature time series. The PM proposes training the model federatedly so the time-series signals never leave the ring. Rings connect to the phone twice daily when the user wakes and when the user sleeps. Battery and cellular upload are the tightest constraints.

#### Scenario B — "Multi-Hospital Sepsis Early-Warning"

A consortium of eight academic medical centres (4 EU, 3 US, 1 UK) wants to train a 48-hour sepsis early-warning model on ICU data. Each site has 20 000 – 150 000 labelled ICU admissions. Legal counsel at each hospital has indicated they cannot share raw PHI with the other hospitals or with a central aggregator. One hospital's research IT team has volunteered to host the aggregator; the others are reluctant to see it hosted at a peer site. Each site runs one FL training round per week; the full training run is expected to take 6 months.

#### Scenario C — "Enterprise Help-Desk LLM Fine-Tune"

A 70 000-employee enterprise wants to fine-tune a general-purpose language model on employee help-desk interactions. Each business unit has its own help-desk system with ~10⁵ – 10⁶ anonymised transcripts. The IT security team mandates that no business unit's transcripts leave the unit's own storage; a centralised training on pooled data is not allowed. There are 14 business units, each on the corporate VPN with stable connectivity. The CISO cares about demonstrating that fine-tuning cannot leak one unit's transcripts to another unit's inference team.

#### Scenario D — the design doc to critique

You will be given a one-page design doc (produce a reasonable example of one yourself, or re-use a colleague's) that proposes an FL system. The doc argues for cross-device FL across 50 institutional partners, selected one partner per round, with a global model refresh every day. Your job is to walk the six-question decision checklist from chapter 01 and produce the critique.

### Deliverables

#### `scenario-A-decision.md`, `scenario-B-decision.md`, `scenario-C-decision.md`

For each scenario, produce a one-page decision memo with the following sections:

- **Shape chosen.** Cross-device, cross-silo, or hybrid (name the layers).
- **The six-question walk.** Answer chapter 01's six questions (endpoint count, participant legal type, data volume, aggregator operator, privacy unit, round cadence). Each answer in one sentence, with the data point from the scenario.
- **The engineering realities that follow.** Pick three from the chapter-01 list (intermittency, non-IID, slice size, untrusted client, scheduled rounds, site-level shift, silo-as-unit, contractually bounded aggregator) and name which apply.
- **Privacy unit.** Record-level, user-level, or client-level — the setting that chapter 05 will develop.
- **The two likely anti-patterns.** For each scenario, name two anti-patterns the design review must guard against.
- **Open questions for counsel / product / mlops.** List the questions you would not resolve as the privacy engineer. For each, who is the right owner.

The three memos together are the first part of the deliverable.

#### `scenario-D-critique.md`

Produce a 2–3 page critique of the design doc. Structure:

- **Shape mismatch summary.** One paragraph identifying the mismatch between the business problem and the proposed shape.
- **Six-question walk.** Walk the chapter-01 checklist against the proposed design; identify which answers the doc gets right and which it gets wrong.
- **Risks the current design walks into.** At least four — SA parameter mismatch, DP accounting mismatch, governance ambiguity, dropout-tolerance failure, etc.
- **Recommended re-design.** One paragraph describing what you would propose instead. Keep it one paragraph; this is not a re-architecture document.
- **Review-meeting talking points.** A bulleted list of five things you would say in the review meeting, in order.

#### `kickoff-paragraph.md`

Pick one of A / B / C and write the one-paragraph FL-shape defence paragraph (see chapter 01's closing) you would read at the project kick-off with mlops, legal, and the business owner in the room. Must explicitly state:

- The shape and the one reason it is right for this business problem.
- The privacy unit.
- The three biggest engineering realities the team must plan for.
- The one question counsel must answer before the project proceeds.

Keep it to 150 words or fewer.

## Starter guidance

- **Walk the six questions in order.** Resist the urge to jump to a conclusion. The order forces you to notice contradictions — e.g., "the PM wants cross-device but the number of endpoints is 15".
- **Watch for the hybrid signal.** Smart-ring data that starts on the ring but is ultimately pooled at a lab for research is a hybrid system — two FL layers chained. Name it as two systems, not one.
- **Call out the "pseudonymous vs. identified" ambiguity.** In a cross-device deployment, the participants are device IDs; in cross-silo, they are named entities. The mismatch is often where the legal basis conversation gets stuck.
- **Do not re-design.** The deliverable is the shape *decision* and its defensible justification, not a full system design. One paragraph of re-design is enough.
- **Reference the chapter.** Each decision memo should cite chapter 01 for the invariants it relies on.

## Acceptance criteria

- [ ] Three decision memos complete, each with the six-question walk, the three engineering realities, the privacy unit, two anti-patterns, and open questions.
- [ ] One critique of the design doc, with shape-mismatch summary, six-question walk, four named risks, re-design paragraph, and five talking points.
- [ ] One kickoff paragraph for one of the scenarios, under 150 words, naming shape, privacy unit, three engineering realities, and one counsel question.
- [ ] Each memo is one page or less (< 500 words). Length is a discipline check.
- [ ] Every claim about an engineering reality is traceable to a line in chapter 01.

## Stretch goals

- **Peer review.** Swap your decision memos with another learner's and critique each other's. Note where the two memos differ and reconcile.
- **Multi-jurisdictional elaboration.** Pick Scenario B and write a one-paragraph note on the Schrems II implications of the EU / US / UK silo split, in terms a counsel would consume. Flag where SA + DP would act as the supplementary measures (preview of chapter 07).
- **Add a Scenario E you invent.** A plausible FL deployment at your own org — a hypothetical if you cannot cite a real one. Walk the six questions and attach the decision memo.
- **"Why not FL" alternative paragraph.** For one of A / B / C, produce a one-paragraph counter-proposal that argues against FL — e.g., "this is better solved by centralised training on de-identified data with a DPIA". State the trade-offs honestly.

## What this lands

- A reflexive decision ordering — shape first, implementation second — that makes every downstream exercise cheaper.
- A re-usable memo template you will copy into every future FL design review.
- The kickoff paragraph is the first draft of what you will read at your first real-world FL project kick-off. Pin it to your project folder.
- The critique is the muscle of "polite but specific pushback in a design review". It is a career skill, not just a module exercise.
