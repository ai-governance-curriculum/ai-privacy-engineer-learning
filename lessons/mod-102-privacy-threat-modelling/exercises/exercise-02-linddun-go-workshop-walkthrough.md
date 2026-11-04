# Exercise 02: LINDDUN GO Workshop Walkthrough

**Estimated effort:** 3 hours (2 hours facilitation + 1 hour preparation and write-up) · **Output:** a Markdown threat-model register with at least 25 triaged threats · **Primary chapters:** 03, 04

## Objective

Run (or simulate) a **LINDDUN GO workshop** against the DFD you produced in exercise 01 and capture the output in the register shape chapter 04 specifies. If you can gather real participants, do so; if not, you can run the exercise solo with the DFD and the card deck. The deliverable is a register that would be useful input to a DPIA.

## Prerequisites

- Chapter 03 (seven LINDDUN categories) and chapter 04 (GO playbook) read.
- Exercise 01 complete — you have a labelled DFD and element / edge tables.
- The LINDDUN GO card deck downloaded from [linddun.org](https://linddun.org/). Print or load on a tablet.
- Participants if available: a product engineer, an ML / inference engineer, an infra / platform engineer, a security peer. A DPO delegate is a bonus.

If you cannot gather participants, you can simulate the workshop solo: treat each card as a self-prompt and discipline yourself to read it, consider the DFD element, and capture candidate threats. The quality bar is lower solo (you lose the social surface-area of a real room) but the exercise is still useful.

## Problem statement

Produce a single Markdown file with:

### Section A — workshop setup

- The DFD (image / Mermaid inline) and a link to exercise 01's file.
- The participant list (real or "run solo") with roles.
- The agenda you used, with actual start / end times per block.
- Any scope exclusions you declared up-front and why.

### Section B — the threat register

A Markdown table. One row per threat captured. Columns:

| ID | DFD element | LINDDUN category (L / I / N-rep / D1 / D2 / U / N-comp) | Threat statement (one sentence) | In scope? | Likelihood (H / M / L) | Impact (H / M / L) | Control family routing | Owner | Status |

Rules:

- At least **25** rows in total.
- Every DFD element touching personal data has at least one row (one category minimum).
- At least one row per LINDDUN category (seven minimum) *somewhere* in the register.
- At least one row is marked **Out of scope** with a documented reason (routed to the security peer, routed to `ai-risk-engineer`, or marked "not a threat on this system because …").
- Threat statements are specific. "The log is leaky" is not; "`prompt-log-raw` retains `user_id` and raw prompt text with 90-day retention and no redaction; cross-session linkability threat for any subject with more than one prompt" is.
- Likelihood and impact are coarse at this stage (the PRAM overlay in exercise 04 formalises them).

### Section C — top-five triage

- The top five threats ranked by `likelihood × impact`.
- For each: the control-family routing and the named owner.
- A one-paragraph rationale for the ranking (what makes these five worse than the rest).

### Section D — facilitator reflections

A short section (200–400 words) covering:

- Which DFD elements generated the most threats? Why?
- Which LINDDUN category took the room the longest to internalise?
- Which of the AI-specific pitfalls from chapter 04 (model weights, caches, telemetry, tenant boundary, legitimate interests) tripped you up? How did you resolve it?
- What would you change about the agenda next time?
- Any threats that you captured but could not resolve into the right category — these are candidates for threat-tree refinement in the MAESTRO pass (exercise 03).

## Starter guidance

- **Resist the urge to tidy the register before you have 25 rows.** Capture first; dedupe and reword after.
- **Treat the cards as prompts, not as checklists.** Read the prompt; look at the element; speak the candidate threat out loud. If nothing comes, write "N/A — because …" explicitly rather than skipping.
- **Expect weights, logs, retrieval corpus, and semantic cache to dominate.** Budget time accordingly.
- **Do not over-score at this stage.** Likelihood / impact here are *triage* scores. Exercise 04 re-scores with PRAM anchors.
- **Mark the scope decisions explicitly.** "Authentication surface = out of scope, routed to security peer" is a *positive* artifact, not a cop-out.
- **Capture each threat's owner by role, not by person.** "Retrieval-lead" survives personnel changes; "Alice" does not.

## Acceptance criteria

- [ ] At least 25 threats captured.
- [ ] Every LINDDUN category has at least one row.
- [ ] Every row has a specific, element-level threat statement (no "logs are leaky"-style rows).
- [ ] At least three rows explicitly out of scope with a documented reason.
- [ ] Each row routed to one control family from chapter 07's taxonomy (or marked "out of scope" with a target role).
- [ ] Top-five triage section named and ranked.
- [ ] Facilitator reflections section addresses all five bullet questions.
- [ ] The register is in a form you could hand to a DPIA (mod-111) without rework.

## Stretch goals

- Run the workshop *twice*: once with the team, once with the DPO delegate afterwards. Compare the deltas. The DPO will usually surface Non-compliance threats the engineers will not.
- For any row marked "likelihood: high", write a one-paragraph note explaining what in the published literature or your system's current state supports the "high" rating. This is what chapter 06 (PRAM) calls defensible anchoring.
- Translate three rows from this register into **STRIDE** categories (Spoofing / Tampering / Repudiation / Info-disclosure / DoS / Elevation-of-privilege) to see where STRIDE under-covers. Document the gap.
- Produce a second "LINDDUN GO on only the backups / snapshots sink" micro-pass. This sink is the most-forgotten and the micro-pass often surfaces real findings.

## What this lands

The register you produce here feeds directly into:

- **Exercise 03** (MAESTRO) — picks the top-ranked threats for deep refinement.
- **Exercise 04** (PRAM overlay) — re-scores the register with anchored likelihood / impact.
- **Exercise 05** (misuse personas) — attaches personas to the register rows.
- **Exercise 06** (PbD checkpoints) — gates the engineering work that the register demands.

Keep the register alive — it is meant to be updated whenever the DFD changes.
