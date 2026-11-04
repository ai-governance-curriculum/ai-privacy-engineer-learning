# Exercise 05: Misuse Persona and Privacy Control Routing

**Estimated effort:** 2 hours · **Output:** four persona cards + a persona-to-threat matrix + an extended routing register · **Primary chapter:** 07

## Objective

Instantiate the four canonical **privacy misuse personas** (chapter 07) against your own system, then extend the PRAM-overlaid register from exercise 04 with explicit persona and control-family routing. The deliverable is the bridge from threat register to engineering backlog.

## Prerequisites

- Chapter 07 (misuse personas + routing) read.
- Exercise 01 (DFD and external entities) complete.
- Exercise 04 (PRAM-overlaid register) complete.
- Chapter 09 (role boundary) open as a reference while routing.

## Problem statement

Produce a single Markdown document with the following three sections.

### Section A — four system-specific persona cards

One card per persona (chapter 07 catalogue: adversarial user, insider, upstream data supplier, downstream integrator). Each card is a short Markdown block with the following fields:

- **Name** — the chapter-07 persona plus a short descriptor for your system (e.g. "Adversarial user — authenticated tenant-A end-user attempting re-identification of a tenant-B subject").
- **Who they are in your system** — concrete role / relationship (tenant end-user, internal SRE, specific supplier, specific downstream API consumer).
- **Capabilities** — what access they have, what they can observe.
- **Motive(s)** — plausible reasons they'd act.
- **Primary DFD elements attacked / implicated** — reference the exercise-01 DFD IDs.
- **LINDDUN categories most implicated** — a short list from the seven.
- **Primary routed control families** — one or two from chapter 07's taxonomy.
- **Out-of-scope note** — if some aspects of this persona belong to `ai-risk-engineer` (chapter 09), route them explicitly.

The cards must be *specific to your system*. A generic "insider might extract training data" card is a fail; "insider = SRE with `prompt-log-reader` IAM role could mine the abuse-review queue for named individuals" is the shape.

### Section B — persona-to-threat matrix

A Markdown table. Rows = threats from the exercise-04 register (in-scope rows only). Columns = the four personas. Each cell contains one of:

- `primary` — this persona is the primary actor for this threat,
- `secondary` — this persona could plausibly manifest this threat but is not the most-likely actor,
- `—` — this persona is not applicable to this threat.

Rules:

- Every in-scope threat has at least one `primary` cell. If none fits, revisit the threat statement.
- At least a handful of threats have multiple `primary`/`secondary` cells (most real threats involve more than one plausible actor).
- A threat whose primary actor is *outside* the four personas (e.g. a DoS-style attacker — security peer, or a fairness attacker — `ai-risk-engineer`) is marked as **out of scope** with the target role named.

### Section C — the extended routing register

Extend the exercise-04 register with these columns:

| Primary persona(s) | Secondary persona(s) | Primary control family (chapter 07 taxonomy) | Secondary control families | Owning module (mod-10X) | Backlog ticket / RFC link |

Rules:

- `Primary control family` is exactly one of chapter 07's named families.
- `Owning module` cites the curriculum module that delivers the depth for that control.
- `Backlog ticket` is a link, or a placeholder string if the ticket does not exist yet (and that placeholder is itself a to-do you commit to filing).
- No row routes to "DP" without a specific identifiability / detectability justification.
- No row routes to "redaction" without a specific sink and purpose.
- Rows where the primary actor is outside privacy scope are marked `routed to <role>` and omitted from the control-family selection.

## Starter guidance

- **Resist generic personas.** The point is to name actors specific to *your* system — tenant end-user, SRE role name, specific supplier name, specific downstream API consumer. The register becomes actionable when the persona is specific.
- **Multiple `primary` cells are fine.** Some threats are equally driven by two personas (e.g. insider + downstream integrator on a prompt-log sink).
- **A persona that "doesn't fit" on your system is a finding.** If none of the four fits, either (a) you have missed a persona (possible but rare), (b) the "threat" is not really a privacy threat (route elsewhere), or (c) you need a fifth system-specific persona.
- **Use the control taxonomy; do not invent families.** Chapter 07's taxonomy is deliberately small. A new family needs a justification stronger than "ours is special".
- **For each threat routed to a module that is not this one, cite the chapter in that module you expect to deliver the control.** This keeps the register honest — if you cannot cite the chapter, you have not actually offloaded the work.

## Acceptance criteria

- [ ] Four persona cards present, each with all eight fields.
- [ ] Every persona card is system-specific (names roles / parties in your DFD, not abstractions).
- [ ] Persona-to-threat matrix covers all in-scope threats and has at least one `primary` cell per row.
- [ ] At least five threats carry multiple persona classifications (`primary` + `secondary`).
- [ ] At least two threats are routed out of scope with a named target role.
- [ ] Extended register has primary control family, secondary families, owning module, and backlog ticket (or placeholder) for every in-scope row.
- [ ] No row routes to "DP" or "redaction" without the sink and purpose stated.

## Stretch goals

- Add a **fifth persona** specific to your system — e.g. a "regulator submitting a Section 15 request" (mostly implicates U + N-compliance, not an attacker, but shapes the DSR surface). Or a "model-provider sub-processor" if your hosted-LLM contract has not fully pinned down their retention.
- For the strongest `primary` persona on each of the top-ten threats, write a one-paragraph "attack scenario" narrative. These are what a red-team (owned by `ai-risk-engineer`) will use as test cases.
- Translate three of your persona cards into **ai-risk-engineer**'s capability-surface shape (adversary role × capability × target × goal × observable). This makes the deferral across chapter 09's boundary explicit.

## What this lands

The persona-attached, control-routed register is:

- The direct input to exercise 06's PbD checkpoint register.
- The source the DPO uses to understand *who* each risk protects against.
- The red-team test case list for `ai-risk-engineer` (via the chapter 09 deferral contract).
- The dependency ledger for mods 103–112 — each entry tells you which later module's depth you are consuming.
