# Exercise 06: Privacy by Design Checkpoint Set

**Estimated effort:** 2 hours · **Output:** a Privacy-by-Design checkpoint register with enforcement, status, and evidence columns · **Primary chapter:** 08

## Objective

Convert the chapter-08 Privacy-by-Design principles into **engineering checkpoints** specific to your system, and produce the full enforcement-and-evidence register that gates the controls from exercise 05. The deliverable is the artifact that keeps the threat model from drifting between releases.

## Prerequisites

- Chapter 08 (PbD as engineering checkpoints) read.
- Exercises 01 (DFD), 04 (PRAM register), and 05 (routing register) complete.
- Access to the actual CI / release / review tooling for your system (even read-only). If you have no access, treat the checkpoints as *proposed* and mark their enforcement column accordingly.

## Problem statement

Produce a single Markdown document with the following four sections.

### Section A — the checkpoint register

A Markdown table. Columns:

| Checkpoint ID | PbD principle (1–7) | Spec (what is being enforced) | Enforcement (CI / lint / release-gate / review-gate / runtime / documented-only) | DFD element / sink / process it governs | Linked threats (exercise-04 IDs) | Owner (role) | Current status (Enforced / Instrumented / Documented / NA) | Evidence artifact(s) |

Rules:

- At least **one checkpoint per PbD principle**. More where warranted by your threat register.
- Each row names *what* is being enforced in a sentence an engineer could code against — "cache configs with `tenant_scope != true` fail CI `cache-config-lint`" is acceptable; "cache is scoped" is not.
- Enforcement mechanism names a *specific gate* in your delivery pipeline, or documents that no gate exists yet (status: Documented).
- Linked threats refer by ID to the exercise-04 PRAM register so a reader can trace the chain threat → routing → checkpoint.
- Status honestly reflects reality. "Enforced" means the gate has fired at least once; "Instrumented" means the gate exists but has not yet blocked; "Documented" means written-down expectation with no gate yet.
- Evidence names a concrete artifact per row: a CI log sample, a cache-config lint rule, a dataset-card template, a DSR SLA dashboard, a redaction false-negative sample, a design-review template field.

### Section B — gap list

Any checkpoint with status `Documented` (no gate) or `Instrumented` (gate not yet validated) appears in a short gap list with:

- The proposed enforcement mechanism (which pipeline, which stage, what signal).
- Rough effort (hours / days / weeks).
- Owner (role).
- Target release / sprint by which it is to move to Enforced status.

If you have no `Documented` or `Instrumented` rows, examine whether your status assessments are honest. In most real systems, at least a quarter of the checkpoints sit at `Documented` on first pass.

### Section C — the chain diagram

A one-paragraph narrative + a short Markdown table showing, for **five** of the top-priority threats from exercise 04, the full chain:

```
Threat (PRAM ID)
  → Primary persona (exercise 05)
    → Routed control family (exercise 05)
      → Specific control (exercise 03 or named here)
        → PbD checkpoint (exercise 06)
          → Test / evidence (this register)
```

Format as a Markdown table with columns `PRAM ID`, `Persona`, `Control family`, `Specific control`, `Checkpoint ID`, `Test`, `Evidence`. Five rows.

### Section D — handoff notes

A short section (half a page) covering:

- **Handoff to the DPIA (mod-111).** Which sections of the eventual DPIA does this register feed? Reference Art. 35(7) sub-clauses where possible.
- **Handoff to the engineering backlog.** Which `Documented`-status items are you asking the engineering lead to accept into the next planning cycle? Name them.
- **Handoff to the `ai-governance-analyst` (L15)** — what goes into the ROPA / control library / framework crosswalks from here?
- **Any items escalated to the DPO** — lawful-basis gaps, consent-propagation gaps, cross-border mechanism gaps, Art. 22 meaningful-information gaps. These are *not* fixable at the engineering layer alone and must be named for escalation.

## Starter guidance

- **Start from exercise 05's routing register.** Every row there should have *at least one* PbD checkpoint governing its control. If a row has none, you have found a gap.
- **Resist "paper checkpoints".** A checkpoint with no linked threat and no enforcement mechanism is theatre. Delete it; it adds clutter.
- **Status honesty is more valuable than coverage breadth.** Three truly `Enforced` rows beat ten `Documented` ones.
- **Group the checkpoints by principle in the register's visual layout** (headers between principle sections) so a reviewer can scan one PbD principle at a time.
- **Name the gate by its real pipeline stage.** "fails in `.github/workflows/privacy-lint.yml` step `cache-config-lint`" is the shape, not "fails CI".
- **Name evidence by its real location.** A CI log sample from a specific run is stronger than "we have CI". If there is no run, mark as planned.

## Acceptance criteria

- [ ] Each of PbD principles 1–7 has at least one checkpoint row.
- [ ] Every checkpoint is specific enough that an engineer could implement the gate without asking a clarifying question.
- [ ] Every checkpoint links to at least one threat ID from exercise 04 (so the chain is traceable).
- [ ] Status column reflects reality; at least two rows are `Documented` with a honest gap (or an explanation of why the system is already complete).
- [ ] Gap list present with proposed mechanism / effort / owner / target for each non-Enforced row.
- [ ] Chain diagram covers five top-priority threats end-to-end.
- [ ] Handoff section names DPIA sections, backlog items, analyst handoffs, and DPO escalations.
- [ ] A new engineer joining the team could read the register and tell which controls would be hard to bypass accidentally and which still rely on vigilance.

## Stretch goals

- Author one of the `Documented` gates as an actual lint / test. Commit it. Prove it fires on a known-bad config. Update the row to `Enforced`.
- Translate your register into the **NIST Privacy Framework Core subcategory** shape (ID-P / GV-P / CT-P / CM-P / PR-P). Each checkpoint maps to one or more subcategory IDs (e.g. `PR.DS-P1`). The crosswalk is reusable for the analyst role's framework table.
- Write a one-page "Art. 25 engineering interpretation" note for your system — a plain-language description of how your checkpoints discharge Art. 25(1) *by design* and Art. 25(2) *by default*. This is the DPO / auditor artifact.
- Instrument a **release-gate dashboard** showing checkpoint status at release time. Even a mocked version for one release is a useful artifact.

## What this lands

With this exercise complete, you own the full chain:

1. **DFD** (exercise 01) — what the system does with data.
2. **LINDDUN threat register** (exercise 02) — what can go wrong, by category.
3. **MAESTRO deep dive** (exercise 03) — depth where it matters.
4. **PRAM-overlaid risk register** (exercise 04) — prioritisation.
5. **Persona + control routing** (exercise 05) — the actors and controls.
6. **PbD checkpoint register** (this exercise) — the enforcement surface.

This is the privacy threat-model bundle an AI Privacy Engineer carries into a DPIA, into release reviews, into regulator queries, and into incident response. Keep it alive.
