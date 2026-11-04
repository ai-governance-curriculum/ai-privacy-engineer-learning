# Exercise 04: NIST PRAM Overlay on the LINDDUN Model

**Estimated effort:** 3 hours · **Output:** a documented scoring scheme + a PRAM-overlaid risk register derived from exercise 02 · **Primary chapter:** 06

## Objective

Convert the LINDDUN threat register from exercise 02 into a **prioritised privacy-risk register** using NIST PRAM vocabulary (data actions / data processing ecosystem / problematic data actions / privacy risks) and a *documented* likelihood × impact anchor scheme. The output is the register shape a DPIA (mod-111) will quote directly.

## Prerequisites

- Chapter 06 (NIST PRAM overlay) read.
- NIST PRAM worksheets downloaded from [nist.gov/itl/applied-cybersecurity/privacy-engineering/collaboration-space/focus-areas/risk-assessment/resources](https://www.nist.gov/itl/applied-cybersecurity/privacy-engineering/collaboration-space/focus-areas/risk-assessment/resources).
- Exercise 02 complete — the LINDDUN GO register is the input.
- Exercise 03 optional but useful — adds depth on one flow that will likely dominate the top of the ranking.

## Problem statement

Produce a single Markdown document with the following four sections.

### Section A — the data-actions inventory

A Markdown table listing every **data action** your system performs on personal data. Enumerate by walking each edge in your exercise-01 DFD:

| Action ID | Verb (collect / retain / generate / transform / disclose / transfer / dispose) | Subject data class | Source (DFD element) | Destination (DFD element) | Party performing action (processing-ecosystem role) | Notes |

Rules:

- Every edge in the exercise-01 DFD that carries personal data produces at least one data action.
- Every data store produces at least one *retention* action.
- Each action names the party *performing* it — "ops", "model-provider", "downstream-integrator", "upstream-supplier". Do not collapse into "the system".

### Section B — the problematic-data-action classification

For each data action from Section A, mark whether it is *problematic* per the PRAM catalogue (surveilling, unanticipated revelation, induced disclosure, stigmatisation, re-identification, inference, exclusion, loss of self-determination, loss of trust, economic / physical / dignity harms). A single action can have *multiple* problematic classifications.

Format: add columns to Section A's table — `Problematic? (Y/N)` and `Problematic categories` (comma-separated from the catalogue).

### Section C — the documented scoring scheme

A short section (at most one page) specifying your chosen likelihood × impact anchor scheme. At minimum:

- **Likelihood anchors** for H / M / L — each with a concrete, written-down meaning (chapter 06 gives a baseline; adapt).
- **Impact anchors** for H / M / L — organised **by harm class**, not a single scalar. A row of the impact table per harm class from the PRAM catalogue.
- An **explanation** of why your anchors are appropriate for *your system's subjects* — e.g. "our subjects include minors; we weight dignity / exclusion harms upward from the baseline".
- An **approver** — the role (DPO / privacy lead / head of AI governance) that signs off on the scheme. Document even if you are the sole approver for this exercise.

The scheme must be *re-usable*. A regulator reading it should be able to reproduce your scores from the register row data.

### Section D — the PRAM-overlaid register

Take the exercise-02 threat register (at least 25 rows) and extend it with these columns:

| Original Threat ID | LINDDUN category | Element | Linked data action(s) from Section A | Problematic-data-action tag(s) from Section B | Ecosystem party | Inherent likelihood (H/M/L) | Inherent impact (H/M/L) | Harm class(es) | Residual likelihood after routed control | Residual impact after routed control | Accepter (role) |

Rules:

- Every row in the exercise-02 register (minus explicitly out-of-scope rows) appears here.
- *Inherent* scores reflect the system **as designed and operated today**, not after planned controls. Use your scoring scheme.
- *Residual* scores assume the chapter-07 routed control works as designed; leave blank if there is no routed control.
- *Accepter* is the role authorised to accept the residual risk on behalf of the subject (DPO in most cases). "Not accepted" is also a valid state and should be marked.
- Explicitly list any threats where you refuse to assign an inherent score and why (e.g. "requires specialist MIA testing — exercise 03 of mod-105 produces that score").

## Starter guidance

- **Write the scheme (Section C) before you score.** Scoring first and then writing anchors produces post-hoc rationalisation.
- **Anchor likelihood to evidence, not to vibes.** If you say H, name the published attack / the stated system state / the specific absence of a control. Chapter 06 is clear on this.
- **Do not collapse harm classes.** A single row can carry multiple; record them. They route to different controls.
- **Treat "loss of self-determination" seriously.** It is easy to deprioritise this harm class because it has no direct measurable consequence; the EU AI Act Art. 22-adjacent GDPR obligations exist precisely to protect it.
- **Residual = inherent is a signal.** Threats with no control planned (residual = inherent) need an accepter named. If you cannot name one, you have an open item for the DPO.
- **The PRAM vocabulary wins.** When you capture a threat, prefer the PRAM word — "induced disclosure", "re-identification" — over a colloquial one. The register must be reviewable by a regulator.

## Acceptance criteria

- [ ] Data-actions inventory covers every personal-data-carrying edge of the exercise-01 DFD.
- [ ] Each data action is classified `problematic Y/N` with one or more PRAM categories where Y.
- [ ] Scoring scheme is one page, with anchors that are concrete and separate by harm class.
- [ ] The scheme has a named approver role.
- [ ] Every in-scope row from exercise 02 is in the register with linked data action(s), inherent L / I, harm class(es), and (where a control is routed) residual L / I.
- [ ] At least three rows have *residual = inherent* and a named accepter (or an explicit "no accepter yet — open item").
- [ ] At least five rows reference Section A data actions correctly.
- [ ] The register is sortable by `inherent likelihood × inherent impact` and the top ten are still *defensible* under your scheme.

## Stretch goals

- Produce a *second* scoring scheme with tighter impact anchors (e.g. splitting H into H / critical). Re-score and compare. If several rows change rank significantly, your anchors are not yet well-tuned.
- For every row with residual score < inherent score, identify the single *most important* control evidence that must exist for the residual to hold. These evidence items are the top entries in exercise 06's checkpoint register.
- Produce a 10-line "executive summary" of the top five privacy risks, each in a sentence, suitable for a privacy-committee readout. This is the DPO's audience.
- Translate your top three rows into the ISO/IEC 29134 DPIA risk-matrix shape (likelihood axis labels: `rare / unlikely / likely / highly likely`; consequence axis labels: `negligible / limited / significant / maximum`). Document the mapping so the two schemes stay in sync.

## What this lands

The register you produce is the direct input to:

- **Exercise 05** — attaching misuse personas to the ranked rows.
- **Exercise 06** — gating the controls each row routes to.
- **mod-111 DPIA** — the register fills Art. 35(7)(c) (assessment of risks) and 35(7)(d) (measures envisaged).
- **EU AI Act Annex IV** — the register feeds Section 2(b) / 2(c) / 2(d) of the technical documentation bundle.

You will carry this register forward through the rest of the track; design it to live for years.
