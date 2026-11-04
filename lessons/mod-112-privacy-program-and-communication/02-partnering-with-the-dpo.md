# Partnering with the Data Protection Officer

The Data Protection Officer is a statutory role under GDPR Articles 37 to 39. It is the single most structurally misunderstood relationship in a privacy programme: engineers assume the DPO is a legal reviewer who signs off their work; lawyers assume the DPO is an engineering-adjacent compliance specialist; executives assume the DPO reports somewhere in the organisational hierarchy. All three readings are wrong. This chapter names what the DPO actually is in law, what the DPO is required to do, what the DPO is prohibited from doing, and how an AI privacy engineering function partners with — rather than reports into or manages — the DPO.

The authoritative sources — read them before the first DPO conversation, not after:

- Regulation (EU) 2016/679 (*GDPR*), **Article 37** (designation of the DPO), **Article 38** (position of the DPO), **Article 39** (tasks of the DPO), and the relevant **Recitals 97**. [eur-lex.europa.eu/eli/reg/2016/679/oj](https://eur-lex.europa.eu/eli/reg/2016/679/oj).
- Article 29 Working Party — *Guidelines on Data Protection Officers ('DPOs')*, WP243 rev.01, endorsed by the EDPB (2017). [ec.europa.eu/newsroom/article29/items/612048](https://ec.europa.eu/newsroom/article29/items/612048).
- UK ICO — *Data protection officers*. [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/accountability-and-governance/data-protection-officers/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/accountability-and-governance/data-protection-officers/).
- EDPB — *Guidelines 05/2021 on the Interplay between the application of Article 3 and the provisions on international transfers* and related DPO practice notes on the EDPB site. [edpb.europa.eu](https://www.edpb.europa.eu/).
- CNIL — *Data Protection Officer (DPO)* guidance pages. [cnil.fr/en/designation-dpo](https://www.cnil.fr/en/designation-dpo).

A note of scope: the DPO is an EU / UK / EEA statutory role. Organisations outside that regulatory perimeter may employ a *privacy officer* or *chief privacy officer*; the partnership principles below largely translate, but the statutory independence requirements in Article 38 are specific to the GDPR DPO and must not be assumed for other titles.

## The statutory DPO — what Article 37 to 39 actually say

### Article 37 — when a DPO must be designated

Article 37(1) requires the controller and the processor to designate a DPO in three cases:

- The processing is carried out by a **public authority or body**.
- The core activities of the controller or processor consist of **processing operations which require regular and systematic monitoring of data subjects on a large scale**.
- The core activities consist of **processing on a large scale of special categories of data** under Article 9 or personal data relating to criminal convictions and offences under Article 10.

AI systems that profile users, score risk, operate recommender surfaces at scale, or handle health / financial / biometric / political / orientation data will almost always trigger the second or third limb. For AI privacy engineers: assume a DPO is designated, and if your organisation has not designated one, flag the gap to legal.

Article 37(5) requires that the DPO be designated **on the basis of professional qualities and, in particular, expert knowledge of data protection law and practices**. The DPO is a professional role, not a rotated administrative duty.

Article 37(6) allows the DPO to be **an employee or an external appointment under a service contract**. Both arrangements are common; the independence requirements in Article 38 apply equally to both.

### Article 38 — the position of the DPO

Article 38 is where the misunderstanding about reporting lines gets settled. The key provisions:

- Article 38(1): *The controller and the processor shall ensure that the data protection officer is **involved, properly and in a timely manner, in all issues** which relate to the protection of personal data.*
- Article 38(3): *The controller and processor shall ensure that **the data protection officer does not receive any instructions regarding the exercise of those tasks**. He or she shall not be dismissed or penalised by the controller or the processor for performing his tasks. **The data protection officer shall directly report to the highest management level** of the controller or the processor.*
- Article 38(5): *The data protection officer shall be bound by secrecy or confidentiality concerning the performance of his or her tasks, in accordance with Union or Member State law.*
- Article 38(6): *The data protection officer may fulfil other tasks and duties. **The controller or processor shall ensure that any such tasks and duties do not result in a conflict of interests**.*

The three load-bearing sentences for the privacy-engineering partnership:

1. The DPO **receives no instructions on the exercise of their tasks**. Not from legal. Not from the CISO. Not from the chief privacy officer. Not from you. The DPO decides what to advise and what to escalate.
2. The DPO **cannot be dismissed or penalised for performing their tasks**. Even if their advice is inconvenient. Attempting to retaliate is itself a breach the regulator will catch.
3. The DPO **reports directly to the highest management level** — board, CEO, equivalent. Not to legal, not to the CISO, not to compliance.

A privacy engineering function that treats the DPO as a reviewer-who-signs-the-DPIA or as a member of the privacy engineering team has already breached Article 38. This is a common mistake in organisations that are new to the DPO role; make sure your organisational chart does not place the DPO under the privacy engineer or vice versa.

### Article 39 — the tasks of the DPO

Article 39(1) is the DPO's statutory job description:

- **(a)** inform and advise the controller or the processor and the employees who carry out processing of their obligations under the GDPR and other data protection provisions.
- **(b)** monitor compliance with the GDPR, with other Union or Member State data protection provisions, and with the policies of the controller or processor in relation to the protection of personal data, including the assignment of responsibilities, awareness-raising and training of staff, and related audits.
- **(c)** provide advice where requested as regards the data protection impact assessment and monitor its performance pursuant to Article 35.
- **(d)** cooperate with the supervisory authority.
- **(e)** act as the contact point for the supervisory authority on issues relating to processing, including the prior consultation referred to in Article 36, and to consult, where appropriate, with regard to any other matter.

Article 39(2): *The data protection officer shall in the performance of his or her tasks have due regard to the risk associated with processing operations, taking into account the nature, scope, context and purposes of processing.*

The engineering reading of Article 39 — the tasks sort into **advise**, **monitor**, and **be the regulator's contact point**. None of them is "operate" — the DPO does not run the privacy programme. The AI privacy engineer runs the programme; the DPO advises on it and monitors it from a statutory independence position.

## The DPO is advisory, not operational — what that means in practice

The WP243 guidelines are explicit (section 4): the DPO **advises the controller** on the controller's compliance obligations; the DPO **does not personally fulfil those obligations**. The controller remains accountable under Article 24. In an AI privacy programme:

- The **controller** is the organisation (often represented operationally by the product owner / business unit lead who makes the processing purpose decision).
- The **privacy engineer** is the technical author who produces the artefacts that demonstrate compliance — the DPIA, the data flow diagram, the PET deployment claim, the DSR runbook, the model-card privacy slice.
- The **DPO** advises on each of those artefacts, monitors that the programme is delivering them in good faith, and tells the controller when the controller is heading into a compliance failure.

Three common operational patterns this clarifies:

1. **"The DPO should write the DPIA."** No. Article 35(2) says the controller shall **seek the advice** of the DPO when carrying out a DPIA; Article 39(1)(c) says the DPO **provides advice where requested**. The DPO advises on the DPIA that the controller (via the privacy engineer) writes.
2. **"The DPO should sign off on the model card privacy slice."** No. The DPO reviews it and advises on it. The sign-off is the controller's — in your sign-off matrix (chapter 01), the DPO is Consulted, not Accountable, on model-card publication.
3. **"The DPO should run the privacy review board."** No. The DPO is a participant and adviser. The board is run by the controller side (most often facilitated by the privacy engineer). The DPO's role at the board is to advise and to record their opinion in the decision log.

Breaking this pattern — delegating operational ownership to the DPO — compromises the DPO's independence and leaves the controller with less accountability, not more. When the regulator inspects, the regulator will look for evidence that the controller took responsibility for compliance, not that the DPO did.

## What the DPO cannot do — the conflict-of-interest guardrails

Article 38(6) prohibits conflicts of interest. WP243 section 3.5 names the shape: the DPO **cannot hold a position that leads them to determine the purposes and means of the processing of personal data**. The CNIL maintains a list of positions that are structurally incompatible with the DPO function:

- Head of the processing (chief executive, chief operating officer, chief financial officer, head of the marketing department, head of the human resources department, head of the IT department).
- Any role with operational decision-making authority over the purposes and means of the processing.

For an AI privacy programme, two situations to watch:

- **The DPO is also the privacy engineer.** This is a conflict — the privacy engineer determines means (which PET at which parameters, which retention mechanism, which logging policy). The two roles must be separate.
- **The DPO is also the CISO.** The ICO and CNIL have flagged this as a potential conflict; it is permissible in small organisations with careful boundary management, but in an AI-heavy enterprise it is a structural risk. Flag it; do not quietly accept it.

Where a conflict exists, the appropriate escalation is to legal and to the highest management level. This is not a privacy-engineering decision; it is a governance decision that belongs upstream of the programme.

## The privacy engineer / DPO partnership — the running interface

With the statutory shape named, the operational partnership has a predictable shape. In order of recurring touchpoints:

### Monthly 1:1 — the briefing

A standing 30-to-60-minute meeting between the privacy engineer and the DPO. The agenda is predictable and the output is a short note in the decision log or a DPO-file:

- **DPIA deltas** — which DPIAs changed since last meeting (new systems, refreshes, scope extensions), and the DPO's view on each.
- **Incoming DPIAs** — which systems are heading into the DPIA pipeline, and whether the DPO's advice is needed in design rather than at review.
- **Incident file review** — which privacy incidents are open or closed, the DPO's read on reportability (chapter 05), and any items still outstanding from after-actions.
- **DSR pattern review** — volumes, failure modes, SLA attainment, items the DPO may want to cite in a regulator engagement.
- **Regulator-shift briefing** — new EDPB opinion, national DPA guidance, enforcement action — DPO brings the regulatory horizon, privacy engineer brings the implementation implication.

### Review-board attendance — the advice-of-record

The DPO attends the monthly review board (chapter 04) and speaks to the decisions on the agenda. The DPO's view on each material decision is captured in the minutes. Where the DPO disagrees with a proposed decision, that disagreement is recorded — not softened, not negotiated. The controller then decides, and the controller's decision in the face of DPO disagreement is itself on record.

The disagreement-recording discipline is where DPOs earn their independence. A review board where the DPO has never disagreed with anything is a review board where the DPO either is not engaged or is being effectively suppressed. An auditor who sees a two-year DPO file with zero recorded disagreements will investigate.

### DPIA consultation — the Article 35(2) record

For each DPIA the privacy engineer authors (mod-111 ch. 01), there is a formal DPO consultation record:

- Date the DPO was consulted.
- Materials the DPO was given (DPIA draft, data flow diagram, PET parameters, memorisation-audit result, etc.).
- The DPO's opinion on each of the six DPIA elements — systematic description, necessity and proportionality, risk assessment, mitigations, residual-risk statement, DPO consultation section itself (which the DPO sees in draft).
- Any points of disagreement and the controller's response.
- The DPO's view on whether Article 36 prior consultation is required.
- The DPO's view on the DPIA's refresh triggers.

The record is kept inside the DPIA document (or as a linked appendix) and is one of the first items a regulator inspection will examine.

### Prior consultation (Article 36) — the DPO as the regulator contact

Where a DPIA concludes that residual risk is high *in the absence of mitigation measures that could further reduce it*, Article 36 requires the controller to consult the supervisory authority before processing. The DPO is the organisation's contact point for that consultation (Article 39(1)(e)).

For an AI system this is a real-and-present scenario, not a theoretical one. A biometric classifier for a public-facing use case, a model that infers health attributes from behavioural data, a system that supports Article 22 automated decisions in a highly consequential domain — these are the cases where the DPIA's residual-risk determination may credibly be "high", and Article 36 is the gate. The partnership discipline:

- The privacy engineer brings the facts and the mitigation measurements.
- The DPO's advice drives the residual-risk determination; in the event of a dispute, the controller decides but the DPO's opinion is on record.
- If Article 36 is triggered, the DPO owns the regulator interface; the privacy engineer produces the technical materials for the submission (a role chapter 06 of mod-111 covers for the regulator-facing bundle shape).

### Incident interface — the reportability decision

Article 33 (notification to the supervisory authority) and Article 34 (communication to the data subject) are statutory obligations with 72-hour (and "without undue delay") clocks. The reportability decision is normally made by legal and the DPO; the privacy engineer provides the facts. Chapter 05 covers the full decision tree.

The partnership discipline during an incident:

- The privacy engineer brings the technical narrative (what was exposed, by what mechanism, to how many subjects, with what certainty).
- The DPO brings the regulatory posture (whether the facts meet the Article 33(1) threshold — "likely to result in a risk to the rights and freedoms of natural persons" — and the Article 34(1) threshold for data-subject notification — "high risk").
- Legal brings the drafting discipline for the notification itself.
- The controller (highest management level) is the sign-off.

Pre-rehearsing this during a quarterly incident drill is where the partnership is proofed. The first time the four of you coordinate should not be at hour zero of an actual incident.

## What the DPO expects from the privacy engineer — the inputs

The DPO's statutory role under Article 39 depends on receiving the facts in a form the DPO can rely on. Concretely, the DPO expects from the privacy engineer:

- **DPIA drafts that are evidence-cited**, not prose claims. The DPO's monitoring task becomes tractable when every mitigation in the DPIA points to an artefact under version control.
- **Timely notification of changes that may require DPIA refresh** — new training data category, new deployment region, new user segment, new processing purpose. "Timely" here means before the change ships, not after.
- **Access to the artefact set** — the DPO must be able to read the model cards, the system cards, the DSR runbooks, the retention schedule, the decision log, without asking for individual permissions.
- **Clear, de-jargoned technical briefings** — the DPO is a privacy professional, not necessarily an ML engineer. Translate PET parameters, MIA advantage figures, FL protocols, TEE attestations into the risk language the DPO can carry to the regulator.
- **Prompt incident facts** — in the first hour of a suspected incident, the DPO needs the facts you have, however incomplete, with the uncertainty flagged. Waiting for a clean narrative compresses the 72-hour window.

## What the privacy engineer expects from the DPO — the inputs

The reverse interface. From the DPO, the privacy engineer expects:

- **Regulatory horizon** — new EDPB opinions, national DPA guidance, enforcement actions, regulator priorities. The DPO is the privacy engineer's primary source for what is coming next.
- **Jurisdictional reading** — where a technical choice has a jurisdictional dimension (cross-border transfer, lawful basis differences between EU and UK, divergence between EEA and Swiss practice), the DPO's reading is load-bearing.
- **Clear advice on DPIA residual-risk language** — the terms "high risk" and "likely to result in a risk" are statutory; the DPO calibrates what the organisation can say.
- **Participation, not surveillance** — the DPO sits inside the programme as a partner, not alongside as an auditor. The healthy pattern is the DPO engages early (at the design review), not only at the DPIA review.
- **Recorded disagreement where it exists** — disagreement is not failure of the partnership; disagreement being swept under the rug is.

## The organisational position — what the chart should look like

The healthy organisational shape at team scope:

```
                     (Board / highest management level)
                             |              |
                        (DPO) ------>  (CEO / COO)
                                            |
                              (Chief Privacy Officer / Head of Governance, L60)
                                            |
                                 (Senior AI Governance Architect, L50)
                                            |
                          (AI Privacy Engineer, L30 — this role)
                                            |
                                   (Team: ML, platform, security, product)
```

The DPO sits on a direct reporting line to the highest management level (per Article 38(3)) and is organisationally *alongside* the privacy-engineering chain, not inside it. The partnership happens horizontally: the privacy engineer briefs the DPO; the DPO advises the privacy engineer and the controller; the controller remains accountable.

If your organisation's chart has the DPO reporting to the CISO, to the chief privacy officer, or to the privacy engineer, flag the structural problem to legal and to the highest management level. This is not a cultural quibble; it is a GDPR Article 38 compliance issue.

## Common failure modes in the partnership

The patterns to catch:

- **DPO-as-signatory.** The privacy engineer drafts a DPIA; the DPO signs it the same day with no recorded dialogue. The DPIA fails the dialogue test; the DPO's independence is compromised; the audit will catch it.
- **DPO-as-operator.** The DPO is asked to run the review board, to maintain the risk register, to update the retention schedule. Each is operational; each compromises the DPO's advisory independence.
- **DPO-as-silent-objector.** The DPO disagrees in private conversation but allows the DPIA or decision to be signed without the disagreement in the record. Three months later, when the decision is questioned, there is no record. This hurts both parties.
- **DPO-as-firewall.** The engineering team routes communication with legal and with the regulator exclusively through the DPO, bypassing direct legal involvement. The DPO becomes a bottleneck; legal cannot do its job; the DPO's regulator-facing role gets entangled with routine legal review.
- **DPO-as-tourist.** The DPO attends the review board but does not engage with the technical materials. The privacy engineer's job here is to produce de-jargoned briefings that make engagement possible; the DPO's job is to engage even when the material is uncomfortable.
- **DPO-reports-to-privacy-engineer.** The actual Article 38 breach. Fix the chart.

## Summary — what you can do after this chapter

- Recite the three load-bearing provisions of Article 38 (no instructions, no retaliation, direct report to highest management) and explain each in one sentence.
- Enumerate the DPO's five Article 39 tasks and map each to a partnership touchpoint with the privacy engineer.
- Place the DPO on the organisational chart in a position compatible with Article 38(3) — alongside, not under, the privacy engineering chain.
- Run the monthly 1:1, the review-board advice-of-record, the DPIA consultation, and the incident interface as four distinct rhythms, each producing a durable record.
- Recognise and flag conflict-of-interest patterns where the DPO role has been placed under operational decision-making authority.
- Distinguish DPO *advice* and *monitoring* (statutory) from *operation* (controller / privacy-engineer responsibility).
- Catch the common partnership failure modes and name the fix for each.

Chapter 03 picks up with the model / system / dataset card privacy slice — the external-facing artefact that communicates the programme's technical claims to engineers, customers, regulators, and the public.
