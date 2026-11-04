# The Boundary to Senior AI Governance Architect (L50) and Head of AI Governance (L60)

The AI privacy engineer role this track prepares you for is level 30. It is a *team-scope* role: one product line, one ML platform, one business unit. The programme you run (chapter 01), the DPIAs you author (mod-111), the model cards you publish (chapter 03), the review board you facilitate (chapter 04), the incidents you respond to (chapter 05), and the crosswalk you maintain (chapter 06) are all team-scope artefacts. Attempting to operate those at enterprise scope — across product lines, across jurisdictions, with regulator engagement and board-level reporting — is not a stretch; it is a scope violation that will produce weaker outcomes than passing the work upward to the right role.

This chapter names what level 50 (senior AI governance architect) and level 60 (head of AI governance) do that is not yours, what the hand-off packets to those roles look like, and the escalation triggers that tell you to pass. The purpose is not to make you small; it is to make the full ladder work. A privacy engineer who confidently defers enterprise architecture to level 50 and board reporting to level 60 is a stronger operator than one who takes on both and does neither well.

A scope note: the level numbers (30 / 50 / 60) are the role grades defined in this track's role model. Organisations use different grade labels; the level numbers in this chapter refer to the role shapes, not to internal title numbering at any specific company.

## The ladder — a one-paragraph map

- **Level 30 — AI privacy engineer (this role).** Owns the team-scope privacy programme and its artefacts.
- **Level 50 — Senior AI governance architect.** Owns enterprise control-library architecture, cross-jurisdiction reconciliation, framework-version migrations, and the design of the privacy / AI governance interface across product lines. Does not own the day-to-day of any one programme.
- **Level 60 — Head of AI governance.** Owns board-level reporting, regulator engagement, strategic posture toward law and policy, and executive accountability for AI governance outcomes. Often sits adjacent to the chief privacy officer and the general counsel; in some organisations the head of AI governance *is* the chief privacy officer with an AI emphasis.

Between you and level 50 there is often a level 40 role (programme lead, head of privacy engineering). Between level 50 and level 60 there is typically a management layer. The specific intermediate shape depends on the organisation; the two roles this chapter names are the ones whose boundary most matters for your scope decisions.

## Level 50 — Senior AI Governance Architect

### What level 50 owns

The role's remit is architectural and cross-product:

- **The control library at enterprise scope.** The set of controls the organisation expects every AI system to implement, versioned, catalogued, and mapped to the frameworks (NIST, ISO, CSA, AI Act). The crosswalk you maintain at team scope rolls up into this library.
- **Cross-jurisdiction reconciliation.** Where EU practice (GDPR + AI Act), UK practice (UK GDPR + ICO + emerging UK AI framework), US state practice (CCPA/CPRA + Colorado + Connecticut + Virginia + Utah + the growing cohort), US federal practice (HIPAA, GLBA, SP 800-53, SEC rules), APAC practice (Japan APPI, Singapore PDPA, Korea PIPA), Brazilian LGPD, etc. are in tension, the architect reconciles — usually by designing a control that satisfies the strictest requirement and deploys uniformly.
- **Framework-version migration.** When NIST publishes PF v2, when ISO 27701 moves from 2019 to 2025, when 42001 is amended, when the AI Act implementing acts land — the architect owns the migration plan at enterprise scope.
- **Enterprise standards for programme shape.** The template of charters, decision logs, sign-off matrices that team-scope programmes (yours) are expected to implement. You customise within the template; the template is the architect's.
- **Technical-strategy input on PETs.** Which PET approaches (DP accountant choice, FL framework choice, TEE platform choice, redaction-tooling choice) the enterprise standardises on. Your team may vary from the standard with justification; the standard itself is the architect's.
- **The bridge to peer functions at enterprise scope.** The architect's counterparts in enterprise security, enterprise data governance, enterprise legal, and enterprise ML platform.

### What level 50 does not own

- **Day-to-day operation of any one team's programme.** That is yours (and the counterparts of yours on other teams).
- **Writing DPIAs for specific systems.** Teams write; the architect reviews patterns.
- **Routine DPO engagement.** The DPO works with the teams through the privacy engineers; the architect engages the DPO on cross-product patterns and on exception requests.
- **Board-level reporting.** That is level 60.
- **Regulator engagement.** That is primarily the DPO (statutorily, Article 39(1)(e)) and the head of governance (level 60); the architect contributes the technical materials.

### Escalation triggers — when a decision belongs at L50

You pass a decision to level 50 when any of the following is true:

1. **The decision affects more than one product line.** A PET parameter choice that will set a precedent across the enterprise; a retention schedule change that must apply across teams for coherence; a DSR runbook variation that other teams must inherit.
2. **The decision requires cross-jurisdiction reconciliation.** A single technical choice that must behave differently in EU, UK, and US; or must be designed to satisfy the strictest regime uniformly. You can describe the tension; the architect decides the resolution.
3. **The decision is a control-library addition.** You identified a control that should become enterprise standard, not just your team's local choice. The architect accepts or rejects the promotion.
4. **The decision is a framework-version migration.** The next version of a framework the crosswalk tracks has landed; the migration has implications beyond your team.
5. **The decision is a control exception that crosses the team boundary.** Deploying without a mandated control for a defined period, where the mandate is enterprise-level and the exception precedent matters.
6. **A dispute between your programme and another team's programme requires architectural adjudication.** Not a political dispute (that goes to management); a technical disagreement about the right privacy pattern.

### The hand-off packet — what you give the architect

When you escalate, the hand-off is a documented packet, not a conversation. The shape:

```
# Escalation to L50 — <issue title>
From: <privacy engineer>
Date: <date>
Sponsor: <team product owner / exec sponsor>

## Problem
<two to four paragraphs: what is the decision, what forces it, why it exceeds team scope>

## Options considered at team scope
<list: what did we try, what fell short>

## Cross-team / cross-jurisdiction dimensions
<the specifics that push the decision upward>

## Recommended direction (if any)
<team's view; not a decision, but a input>

## Specific request to the architect
<what decision are we asking for; what architectural product (new library row,
 migration plan, exception approval, reconciliation design)>

## Timing
<when we need a decision; what breaks if we miss the date>

## Linked artefacts
<pre-reads, DPIAs, measurements, incident records, crosswalk rows>
```

The packet goes to the architect; the architect may ask for additional material, may call a working session with multiple team privacy engineers, may propose a cross-team pattern. The decision the architect makes lands in the enterprise control library and is reflected back down into your team's programme as a standard you adopt.

### The running relationship with L50

Beyond escalations, the healthy running relationship:

- A monthly or quarterly architect-sync (depending on organisational scale) where team privacy engineers brief the architect on their patterns, deltas, and pain points. The architect sees the ground truth across teams.
- A standing review channel where cross-team patterns are proposed and discussed.
- The architect sitting as an observer at your review board when a cross-team decision is on the agenda.
- Your participation in enterprise-level working groups on framework migration, policy drafting, and tooling selection.

## Level 60 — Head of AI Governance

### What level 60 owns

The role's remit is strategic and board-facing:

- **Board-level reporting.** Quarterly or semi-annual reporting to the board or board committee on the state of AI governance, including privacy. The content, not just the delivery — the narrative the organisation tells its board about AI risk.
- **Regulator engagement.** Formal engagement with supervisory authorities, sectoral regulators, standards bodies, and (in some organisations) policy-making engagement with government.
- **Strategic posture.** Where the organisation stands on emerging regulation, voluntary frameworks, industry self-governance, and public commitments.
- **Executive accountability for AI governance outcomes.** The "buck stops here" seat. In many organisations, this is formally the chief privacy officer; in some, a dedicated chief AI officer or head of responsible AI.
- **Cross-functional authority.** Legal, security, engineering, product, and governance teams across the enterprise coordinate through this role on AI governance matters that are strategic (not just operational).
- **Executive-level incident sign-off.** For incidents that reach the executive threshold (reportable breaches of significant scope, regulator contact, public-material incidents), level 60 is the sign-off for the organisation's position.

### What level 60 does not own

- **The technical artefacts.** DPIAs, model cards, crosswalks, runbooks — these are team-scope (level 30) and enterprise-pattern (level 50) products.
- **Day-to-day DPO engagement.** The DPO is statutorily independent (chapter 02); the head of governance works alongside the DPO, respecting the independence boundary.
- **Operational incident response.** Incidents run by the incident commander (security) and the privacy engineer (facts); level 60 is informed, briefed, and sign-off for the externally-visible position, not the operator.
- **Specific control choices.** Level 60 sets posture, not parameters.

### Escalation triggers — when a decision belongs at L60

You pass a decision to level 60 (usually through your line of management or through the architect) when:

1. **The decision is board-visible.** A material privacy risk the board must know about, an incident with board-notification implications (including SEC 8-K materiality considerations for public companies), a strategic commitment the organisation is contemplating.
2. **The decision is regulator-facing.** Formal engagement with a supervisory authority beyond routine communication; a prior-consultation submission under Article 36; a regulator inquiry; an enforcement action.
3. **The decision is a public commitment.** An external pledge, a public policy on AI use, a disclosure to customers that will be material to them.
4. **The decision shapes external posture.** The organisation's position on emerging regulation; participation in industry self-governance; voluntary audit programmes.
5. **A level 50 decision is contested at the executive level.** The architect proposed a direction; a product leader or executive disagrees; resolution requires executive accountability.

### The hand-off — what goes up to L60

Usually not directly from you. The normal path is level 30 → level 40 (programme lead) or level 50 (architect) → level 60. What you contribute to the level-60 packet:

- **Factual materials.** The incident's technical narrative, the DPIA's residual-risk claim, the measurement that distinguishes options, the crosswalk evidence.
- **Scenario analysis.** What happens under option A versus option B, in the specific terms the board will understand (customer impact, regulator posture, financial implications where relevant).
- **The "least-sanitised version".** The honest technical read, without the softening that sometimes happens in upward passes. Level 60's job is to carry the hard news if it is hard news; sanitised inputs produce sanitised outputs.

The healthy pattern: the privacy engineer's technical narrative in level-60 packets is attributed and preserved as a layer of the packet, not rewritten into prose that loses the measurement specifics.

### The running relationship with L60

At team scope, you may have little direct interaction with level 60. The healthy pattern is:

- A yearly briefing where team privacy engineers present their programmes to the head of governance (often as part of a wider governance review).
- Incident-response rehearsals (chapter 05) that include the executive sign-off role, so the first time you brief level 60 on an incident is not during a live incident.
- Invitation to the head of governance's office hours or open sessions, if the role runs them.

Your main window into level 60 is through level 40 / 50 and through the DPO's monthly briefings (chapter 02) — the DPO's view of regulator posture is level 60's primary input source as well.

## When the boundary breaks — characteristic failures

The failure modes are symmetric: either you take on what is not yours (scope creep up) or you hand off what is yours (scope abandonment down).

### Scope creep up — taking on what is not yours

- **You negotiate directly with a regulator.** Even if a regulator contacts your team directly, the response goes through the DPO and legal; the privacy engineer provides facts, not organisational positions.
- **You commit to a cross-team control change.** A commitment that binds other teams requires the architect's agreement, not just yours.
- **You brief the board.** If you are asked to brief the board directly, confirm the ask with level 50 / 60; the brief should be co-authored and the delivery context agreed.
- **You draft an external public commitment.** External-public is level 60's territory; the privacy engineer contributes the technical content; communications and legal draft; level 60 approves.

The cost of scope creep up is not usually a formal failure (nobody gets fired); it is that the decision becomes brittle because it was made without the context and authority the larger role brings. A commitment made by level 30 that binds level 60 is a commitment that will be renegotiated under less favourable conditions later.

### Scope abandonment down — handing off what is yours

- **You escalate a team-scope PET parameter choice.** Choosing ε = 3.0 versus ε = 5.0 for your team's model is yours (with partner input per the sign-off matrix). Pushing this decision to the architect is a scope abandonment that burns the architect's time on a decision the team should own.
- **You delegate the DPIA authorship to legal.** The DPIA is the engineer's document in close partnership with the DPO; legal reviews but does not author.
- **You defer the model-card privacy slice to the ML team.** The privacy slice is yours; the ML team authors the technical-details sections but the privacy claims are yours to author and own.
- **You skip the quarterly retrospective.** The retrospective is a team-scope ritual (chapter 01); not running it is a programme failure that no one else owns.

The cost of scope abandonment is weaker team-scope outcomes — decisions not made, artefacts not maintained, incidents not learned from. The upward roles cannot fill the gap because they have their own scope.

## The hand-off packet taxonomy — what goes to whom

A reference table the exercise for chapter 06 (crosswalk) and 07 (portfolio) can use as a scope check:

| Decision class | Owner at team scope (you at L30) | When it escalates | Escalation destination |
|---|---|---|---|
| PET parameter choice for one system | You | Sets enterprise precedent | L50 |
| DPIA for one system | You | Residual risk is "high" after mitigation | DPO (Art. 36 consultation) → regulator (L60 relationship) |
| Model-card privacy slice | You | External commitment is material | L50 / L60 for review |
| Retention schedule for one team | You | Cross-team coherence required | L50 |
| DSR runbook for one team | You | Cross-product pattern needed | L50 |
| Incident facts and reportability input | You (facts) + DPO + legal (decision) | Public-material or regulator-facing | L60 for external posture |
| Crosswalk for your team's artefacts | You | Enterprise rollup | L50 |
| Programme charter for your team | You | Template change | L50 |
| Credentials pursuit (personal) | You | Role transition (into L40 / 50) | Line management |
| Credentials pursuit (team) | You (team lead) | Hiring and certification scope | L40 / L50 |

The table is not exhaustive; use it as the shape of the check — "is this decision in my column or is it pointing right?"

## The one-pager escalation standard

A consistent escalation discipline across teams makes the upward roles work. The one-pager the architect and the head of governance expect from a level-30 escalation:

```
# Escalation one-pager
From: <you>
To: <L50 / L60 / both>
Subject: <one line>
Why upward: <one sentence — which escalation trigger from chapter 08>
What we decided and can act on: <at team scope>
What we need the recipient to decide: <specific ask>
Options: <listed with trade-offs>
Measured evidence: <path pointers>
Timing: <when / what breaks>
```

This one-pager is the same shape as the review-board one-pager (chapter 04) with a different top matter. The consistency is the point — any level-30 engineer in the enterprise producing escalations in the same shape lets the architect and the head of governance operate at scale.

## The career-ladder view — getting to L50 and L60

A brief orientation for the engineer whose five-year plan includes stepping upward:

- **Level 30 to level 40 (programme lead):** the move is from operating one programme to operating multiple, usually across a business unit. CIPM becomes load-bearing (chapter 07). The operating experience that counts: having run the programme at level 30 for long enough to have seen multiple incident after-actions, multiple DPIA refreshes, multiple review-board quarters.
- **Level 40 to level 50 (senior architect):** the move is from operating programmes to designing the standard. Architectural thinking, cross-jurisdiction fluency, and the ability to see patterns across many team-scope programmes. ISO LI credentials, enterprise-scale project experience, and often an organisational move from engineering-adjacent to governance-adjacent.
- **Level 50 to level 60 (head of governance):** the move is from architectural leadership to executive leadership. Board-adjacent experience, regulator-engagement experience, and the strategic capacity to carry the organisation's public position on AI governance. AIGP becomes a near-standard credential at this level; CIPM / CIPP are assumed baseline; CPO adjacency often matters.

The lesson for level 30: do not try to perform level 50 or level 60 from your seat. Do the work of your seat well; the next level becomes possible because the artefacts you produced at level 30 are what qualify you to design for others at level 50.

## Composition with the preceding chapters

This chapter closes the module and composes with the ones before:

- The programme charter (chapter 01) names the escalation path; the destinations are the roles chapter 08 defines.
- The DPO partnership (chapter 02) is horizontal; the escalation to level 60 is vertical and goes through the DPO's regulator contact role for regulator-facing matters.
- The model-card privacy slice (chapter 03) the privacy engineer owns; level 50 owns the enterprise template it conforms to; level 60 owns the public-posture layer when the card is publicly released.
- The review board (chapter 04) exercises the sign-off matrix; the "escalate" row of the matrix points at the roles this chapter names.
- Incident response (chapter 05) has specific triggers (regulator contact, SEC materiality, public-material incident) that escalate to level 60.
- The crosswalk (chapter 06) rolls up into the enterprise control library owned by level 50.
- The credentials portfolio (chapter 07) names the credentials that become load-bearing at each level.

The chapters are designed to compose; this closing chapter is the scope-check on the composition.

## Summary — what you can do after this chapter

- State the role shape at level 30, level 50, and level 60 in a sentence each and name what each level does *not* own.
- Identify the six escalation triggers that take a decision to level 50 (cross-product, cross-jurisdiction, control-library addition, framework-migration, cross-team exception, cross-team technical dispute).
- Identify the five escalation triggers that take a decision to level 60 (board-visible, regulator-facing, public commitment, external posture, contested L50 decision).
- Author the escalation hand-off packet and the one-pager that makes the escalation legible to the upward role.
- Recognise and avoid scope creep up (taking on regulator negotiation, cross-team binding commitments, board briefings, external commitments) and scope abandonment down (delegating PET parameters, DPIA authorship, model-card slice, retrospectives).
- Use the running relationships — monthly architect-sync, yearly governance briefing, incident rehearsals with the executive sign-off — to keep the three levels coherent without collapsing the boundary.
- See the career ladder as a sequence of scope expansions, each earned by the quality of artefacts at the current level.

This completes the chapter set. The module's exercises (01 through 07) operationalise each chapter into a producible deliverable. The resources file lists the authoritative references the chapters cite.
