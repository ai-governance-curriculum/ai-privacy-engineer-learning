# Exercise 02: CCPA / CPRA + ADMT + US State-Law Matrix Walkthrough

**Estimated effort:** 2 hours · **Output:** one state-matrix spreadsheet (or Markdown table) + one ADMT engineering-surface spec · **Primary chapter:** 04

## Objective

Produce a working **state-privacy-law matrix** keyed to the CCPA / CPRA baseline, extend it with the **CPPA ADMT** engineering surface, and prove you can diff a new state law into the matrix in under 20 minutes. The deliverable is the artifact a privacy-engineering team actually keeps — not a textbook.

## Prerequisites

- Chapter 04 (CCPA / CPRA / ADMT / state matrix) read.
- Current authoritative references open: [CCPA / CPRA](https://leginfo.legislature.ca.gov/faces/codes_displayText.xhtml?lawCode=CIV&division=3.&title=1.81.5.&part=4.&chapter=&article=), [CPPA regulations](https://cppa.ca.gov/regulations/), [IAPP US State Privacy Legislation Tracker](https://iapp.org/resources/article/us-state-privacy-legislation-tracker/).
- A specific AI system / product in mind (reuse the system you chose in Exercise 01). If you did not pick one, use the Exercise 01 default scenario.

## Problem statement

Produce two deliverables.

### Deliverable A — the state-matrix spreadsheet

A table (CSV, spreadsheet, or Markdown) with **one row per enacted state comprehensive privacy law** (CCPA / CPRA as the baseline row, then VCDPA, CPA, CTDPA, UCPA, OCPA, TDPSA, TIPA, NJDPA, FDBR, NHIPA, MDPA, MCDPA, DPDPA, RICDPA, INCDPA, and any newer laws enacted at your review date — pull current list from the IAPP tracker).

Columns (at minimum):

1. **State / law code** (e.g., `CA / CCPA+CPRA`).
2. **Effective date (consumer-rights live from)** — refresh at your review date.
3. **Applicability thresholds** — processing threshold on residents / revenue / data-sale percentage. Mark "we are in scope / not in scope" for your product.
4. **Sensitive data default** — opt-in / opt-out / limit-use. Note the delta from CCPA.
5. **Rights set delta vs. CCPA** — list any right present in this state's law that CCPA (as amended by CPRA) does not provide, or any CCPA right missing here. If identical in rights set, write "parity with CCPA + CPRA".
6. **Universal opt-out signal (UOOM) required** — yes / no (GPC or equivalent).
7. **Profiling / ADM opt-out shape** — does the law provide an opt-out of profiling in furtherance of decisions that produce legal / similar-significant effects? Note the shape difference against CPPA ADMT.
8. **DPA / risk-assessment required** — what triggers it? What minimum content? Note delta against GDPR Art. 35.
9. **Sector carve-outs** (entity-level vs. data-level for HIPAA / GLBA / FERPA / employment).
10. **Enforcement** — AG only / private right of action / cure period / specific agency.
11. **Engineering delta for us** — the *specific* pipeline change we need to satisfy this state law beyond what CCPA + our current pipeline already provides. If none, write "absorbed by existing pipeline". If something, name it (e.g., "add sensitive-data opt-in flow before processing precise geolocation").

Row 1 is CCPA + CPRA — the baseline. Every other row is a **diff**. An honest row is often one line of delta plus "absorbed by existing pipeline".

### Deliverable B — the ADMT engineering-surface spec

A short (1–2 page) specification that describes the **engineering surface you would implement to satisfy the CPPA ADMT regulations** for your chosen AI system. Must cover:

1. **In-scope decisions** — list the specific decisions your AI system makes that are covered by CPPA ADMT, and the criterion that puts each in scope (e.g., "assistant's autonomous refund action up to €50 in scope because it is a significant decision about access to essential service — document position"). Where you believe a decision is *not* in scope, state why and the test you would run to confirm.
2. **Pre-use notice** — the content template (fields, plain-language description, delivery timing before the decision), the UX surface where it renders, and the CI test that asserts it renders before any in-scope decision.
3. **Opt-out mechanism** — the endpoint / UX, the routing to a human reviewer with authority to overturn, the SLA for the review.
4. **Access-to-ADMT-information response** — the content template (output, role in decision, logic involved, categories of input, significance and envisaged consequences), the API that produces the response, and the per-decision log the response reads from.
5. **Risk assessment** — the content that satisfies the CPPA risk-assessment rule (refresh content list against the current effective text).
6. **Reuse map** — which of these controls can also serve GDPR Art. 22 (chapter 02), LGPD Art. 20 (chapter 06), and PIPL Art. 24 (chapter 06). The expected answer is: one engineering surface serves all four, with per-regime evidence slices. Show the slicing.

## Starter guidance

- **Pull the current enacted list from the IAPP tracker at your review date.** Do not trust static lists in secondary sources.
- **Write diffs, not restatements.** A row that reads "same as CCPA + opt-in for sensitive data" is better than a full rights restatement.
- **Lean on the existing pipeline.** If your product already honours GPC, UOOM is "absorbed". If your product already does Art. 22 human review, Virginia profiling opt-out may be absorbed. The exercise is to be honest about where the *real* engineering delta is.
- **Flag the state AI laws.** Colorado's AI Act and Texas TDPSA profiling provisions are the places you are most likely to add engineering, not the basic rights set.
- **Date your table.** "Refreshed against IAPP tracker on YYYY-MM-DD".

## Acceptance criteria

- [ ] Row count matches the current enacted-state count on the IAPP tracker at your review date (cite the refresh date).
- [ ] Every row has a value in column 11 (engineering delta) — either "absorbed by existing pipeline" or a specific, scoped change.
- [ ] The sensitive-data-default column distinguishes opt-in vs. opt-out states correctly against CCPA's limit-use shape.
- [ ] Deliverable B's in-scope list is defensible — every entry cites a specific criterion from the CPPA regulation and your product's specific use.
- [ ] Deliverable B's reuse map shows the four-regime composition (GDPR Art. 22, LGPD Art. 20, PIPL Art. 24, CPPA ADMT) sharing one engineering surface with per-regime evidence slices.
- [ ] A new state law dropped on your desk could be diff'd into your matrix in under 20 minutes using your existing columns.

## Stretch goals

- Add the Colorado AI Act (SB 24-205) as a **separate row group** below the comprehensive-privacy matrix. Keep the same columns where meaningful; add a column for high-risk-AI scope and anti-discrimination assessment content. Note the engineering delta against GDPR Art. 22 / CPPA ADMT.
- Add the top-3 US state **biometric privacy** laws (e.g., Illinois BIPA, Texas CUBI, Washington) as a secondary matrix. Note the engineering delta if your system uses face / voice / fingerprint / iris / vector embeddings of individuals.
- Draft the "universal DSR intake" state-table that your product's DSR portal would render — showing which state's rights are available to a given resident, with the right-specific SLA on each.

## What this lands

- The state matrix is the artifact `ai-governance-analyst` (L15) maintains on cadence and `senior-ai-governance-architect` (L50) folds into the enterprise control library. As an AI Privacy Engineer you need to **own the shape** of the matrix even if you do not update it daily — because its engineering-delta column points at your backlog.
- The ADMT engineering-surface spec is reused in `mod-111-dpia-and-privacy-reviews` and `mod-112-privacy-program-and-communication`.
