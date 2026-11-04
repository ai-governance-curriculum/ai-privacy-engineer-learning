# Exercise 04: NIST Privacy Framework + ISO/IEC 27701 Crosswalk

**Estimated effort:** 2 hours · **Output:** one Current → Target Profile delta table, one framework-crosswalk spreadsheet, and one defence memo on a single control · **Primary chapter:** 07

## Objective

Produce the two artifacts an engineering team actually uses to run a privacy program: (1) a **NIST PF Current / Target Profile with a delta road-map** specific to your AI system, and (2) a **framework-crosswalk** that lets a single control serve multiple audits (NIST PF ↔ ISO/IEC 27701 ↔ GDPR ↔ EU AI Act ↔ sector laws). Then defend one hard control end-to-end.

## Prerequisites

- Chapter 07 (NIST PF + ISO/IEC 27701 / 29100) read.
- Chapters 02–05 read (you will cite GDPR articles, AI Act articles, and sector-law clauses in the crosswalk).
- The [NIST Privacy Framework Core](https://www.nist.gov/privacy-framework) open — specifically the current version of the Core with Functions / Categories / Subcategories.
- Access to a secondary summary of **ISO/IEC 27701** control set (Annex A / B / C control numbering). If your organisation has purchased the standard, use the official text; otherwise, a NIST-published or IAPP-published crosswalk at the Subcategory → 27701 Annex level is acceptable for this exercise, with the standard itself consulted at any point you are uncertain. <!-- needs-research: cite the specific crosswalk source used in your deliverable. -->

## Problem statement

Produce three deliverables.

### Deliverable A — Current / Target Profile with delta

For the AI system you chose in Exercise 01, author a **Profile** consisting of NIST PF Subcategories that are relevant to the system. For each selected Subcategory, record:

- **Subcategory ID + text** (e.g., `CT.DM-P1`, `GV.PO-P2`).
- **Why in scope** — one sentence linking the Subcategory to a specific obligation (GDPR / AI Act / sector / state law) your system faces.
- **Current state** — Partial / Risk-Informed / Repeatable / Adaptive, with one sentence of evidence.
- **Target state** — one of the same four tiers, with one sentence of justification.
- **Delta** — specific engineering work to close the gap (or "no gap" with evidence of adequacy).
- **Owner on the ladder** — the role (L15 / L25 / L30 / L35 / L50) that owns the delta per chapter 08.

Minimum coverage: **at least two Subcategories per Function** (IDENTIFY-P, GOVERN-P, CONTROL-P, COMMUNICATE-P, PROTECT-P) — ten rows minimum. Include at least one AI-specific Subcategory (memorisation risk, ADM meaningful-information, retrieval-ACL, DP accountant log, unlearning capability).

### Deliverable B — the framework crosswalk

A spreadsheet or Markdown table mapping each row in Deliverable A to its equivalents in other frameworks and regulations. Columns:

| NIST PF Subcategory | ISO/IEC 27701 control(s) | GDPR article(s) | EU AI Act article(s) | Sector law clauses (HIPAA / FERPA / COPPA / GLBA, as applicable) | NIST PF / 29100 principle(s) | Our evidence artifact |

Row example (illustrative shape, do not copy-paste without verifying each cross-reference against primary text):

> `PR.DS-P1 — Data-at-rest is protected consistent with the organisation's risk strategy` ↔ `27701 + 27002 Annex A controls on cryptographic measures / key management` ↔ `GDPR Art. 32(1)(a)` ↔ `EU AI Act Art. 15 cybersecurity` ↔ `HIPAA Security Rule 164.312(a)(2)(iv) encryption + decryption`, `GLBA Safeguards 314.4(c)(3) encryption at rest` ↔ `OECD "Security Safeguards" principle / 29100 "Information security"` ↔ `encryption-at-rest audit report (quarterly) stored in the privacy evidence register`

Column rules:

- Each cell cites a **specific** clause / control number, not a vague framework reference.
- Where no equivalent exists, write "no equivalent" rather than inventing one.
- The "our evidence artifact" column names a single artifact per row — the point is that **one artifact serves many columns**.

### Deliverable C — defence memo for one control

Pick **one** row from Deliverable B where you can genuinely say "one control, many audits". Write a short memo (1 page) that:

1. States the control precisely (what it is, where in the system it lives, how it is configured).
2. Explains how the control discharges each cell in its row of the crosswalk — for each framework / regulation, say *which obligation it satisfies and why*.
3. Describes the test that proves it works (pass / fail criterion, cadence, who runs it).
4. Lists the evidence it produces and where that evidence is retained.
5. Identifies the single hardest audience it would be challenged by (regulator, auditor, DPO) and the one-paragraph defence.

Best candidate controls: encryption-at-rest with KMS, DSR intake + routing, retrieval-ACL enforcement, redaction-at-ingest on prompt sinks, DP-SGD training with per-run accountant log.

## Starter guidance

- **Do not ladder every Subcategory.** The PF Core has well over 100 Subcategories; your Profile should cover the ~15–30 that *materially* apply to an AI system. Being deliberately narrow is right.
- **Use the official crosswalks where they exist.** ISO/IEC 27701 Annex D maps 27701 clauses to GDPR articles directly. NIST has published PF-to-GDPR crosswalks. Reuse them, cite them.
- **Honest tiers.** The point of marking Current = Partial is to drive the delta. If everything is Current = Repeatable, your Profile is probably self-flattering.
- **AI-specific Subcategories.** The PF is AI-agnostic; the AI specialisation lives in your *interpretation* of each Subcategory. `CT.DM-P1` (data processing permissions and consent are documented) applied to an AI system includes consent-propagation into training, retrieval, inference, and feedback-collection paths.
- **Defer out loud.** Any row where the delta involves program-leadership, cross-jurisdiction harmonisation, or SOC work should name the role per chapter 08 and stop there.

## Acceptance criteria

- [ ] Deliverable A has at least two Subcategories per Function (10+ rows minimum, all five Functions represented).
- [ ] Every row has a Current tier *and* a Target tier, with evidence sentences, *and* a specific delta or "no gap" justification.
- [ ] At least three rows reference an AI-specific concern (memorisation, DP, ADM, retrieval-ACL, unlearning, logging policy).
- [ ] Deliverable B crosswalk has specific clause / control numbers in every cell (or "no equivalent"); no hand-waving.
- [ ] The "our evidence artifact" column in Deliverable B points to at most **one** artifact per row (if more than one, consolidate — the point of the exercise is compound evidence).
- [ ] Deliverable C memo defends the control in each framework / regulation column with a specific obligation and a specific test.
- [ ] Any out-of-ladder deferral in the Profile delta is named per chapter 08.

## Stretch goals

- Extend the crosswalk with **ISO/IEC 42001:2023 (AIMS)** clauses relevant to the control (chapter 07 mentions this stack).
- Extend the crosswalk with **CSA AI Controls Matrix** rows (useful for cloud-first programs — look up current CSA AI Controls Matrix publication).
- Produce the **inverse view** — a table keyed by *your evidence artifact*, listing every obligation each artifact discharges. This is the shape an evidence-collection agent (`ai-governance-analyst` scope) files against.

## What this lands

- The Profile and delta are reusable as the privacy-engineering slice of a program-level NIST PF Target Profile that `senior-ai-governance-architect` (L50) would roll up.
- The crosswalk is the artifact that makes one engineering effort serve multiple audits — this is the compound effect chapter 07 calls out.
- The defence memo format reappears in `mod-111-dpia-and-privacy-reviews` as the "control-level narrative" section of an audit-grade DPIA.
