# Exercise 03: Data Minimisation Checkpoint Set

**Estimated effort:** 2 hours · **Output:** `minimisation-register.md` + `checkpoints.yaml` + at least two CI / lint enforcement artifacts · **Primary chapter:** 04

## Objective

Produce a **minimisation checkpoint set** covering all four dimensions — collection, processing, retention, output — for one real AI system, with each checkpoint named, specified, enforced, owned, and evidenced. The register is one of the artifacts the DPIA (mod-111) consumes directly.

## Prerequisites

- Chapter 04 (data minimisation controls) read.
- A privacy DFD of the system (mod-102 exercise 01 output is ideal) so the sinks the checkpoints apply to are already enumerated.
- The propagation contract from exercise 02 — the matcher is one of the enforcement surfaces this register references.
- Access to at least one real CI / lint / release-gate / runtime-monitor surface in the system; if you have none, pick a target CI system and mock the enforcement.

## Problem statement

Produce three deliverables.

### Deliverable A — `minimisation-register.md`

A Markdown document introducing the register, the four dimensions, and the enforcement categories, followed by **at least twelve** checkpoints (three per dimension minimum) using the table shape from chapter 04 and mod-102 chapter 08:

| Checkpoint ID | Dimension | Spec | Enforcement | Owner | Current status | Evidence |

- `Checkpoint ID` — e.g. `M-COL-01`, `M-PROC-03`, `M-RET-02`, `M-OUT-01`. Stable and referenceable.
- `Dimension` — Collection / Processing / Retention / Output.
- `Spec` — one or two sentences on what the checkpoint enforces.
- `Enforcement` — the concrete mechanism: CI config lint / CI schema lint / release gate / runtime monitor / review gate / contract clause. Include the file / tool.
- `Owner` — a team or role (`data-platform`, `ml-platform`, `app-team`, `security-peer`, `this-role`).
- `Current status` — Enforced / Instrumented / Documented / Not applicable, per chapter 04.
- `Evidence` — a link or a description of the artifact that proves the gate fires (a CI run; a dashboard; a runbook).

A short "tensions" section should name any of the three chapter-04 tensions (coverage, researcher freedom, safety-review retention) that apply to your system and the trade-off you made.

### Deliverable B — `checkpoints.yaml`

A machine-readable version of the register. One YAML list; each item has `id`, `dimension`, `spec`, `enforcement.type` (`config_lint | schema_lint | release_gate | runtime_monitor | review_gate | contract`), `enforcement.ref` (file / tool / doc link), `owner`, `status`, `evidence_ref`. The YAML is consumed by the DPIA tooling to render the Art. 32 "measures" section.

### Deliverable C — enforcement artifacts

Author at least **two** of the following, actually usable:

- A **config lint rule** — e.g. an OPA / Rego policy / Terraform Sentinel policy / custom CI script that rejects an infra-as-code change that introduces a sink without a retention policy.
- A **schema lint rule** — e.g. a dbt / sqlfluff / Checkpoint-style test that rejects a table definition lacking a `data_class` column tag.
- A **release gate** — a CI job that blocks deployment of a model version if its ML-BOM is missing or its datasheet reference does not resolve.
- A **runtime monitor** — a scheduled job / dashboard definition that samples prompt-completion log age and alerts if oldest-record-age exceeds the retention deadline.
- A **CI test** — a canary record seeded at retention expiry; test asserts the retention job deletes within SLA.

Pseudocode is acceptable where real tooling is unavailable, but the artifact must be specific enough that an engineer could implement from it in under a day.

## Starter guidance

- **Use the DFD sinks from mod-102 exercise 01.** Walk them and assign each sink to at least one checkpoint in each dimension. A sink absent from the register is a gap.
- **Not every checkpoint is enforceable yet.** Some will be `Documented` or `Instrumented`. That is fine — the register is honest about the gap, and the DPIA presents the plan to close it.
- **Prefer code-enforced over review-enforced.** A policy that lives in a style guide has no mechanism. A policy that lives as a CI lint has one.
- **Pair the checkpoint ID with the PbD checkpoint register from mod-102 ch. 08 where there is overlap.** Cross-reference rather than duplicate.
- **For output minimisation, think belt-and-braces.** The retrieval ACL (mod-109) is the primary control; the output filter is the secondary. Both appear in the register with the same event (cross-tenant leak) but different checkpoints.
- **Retention is often the quickest to fix and the most impactful.** If three sinks have no retention policy, those three are the first three checkpoints with `Documented` status.

## Acceptance criteria

- [ ] At least twelve checkpoints, with at least three per dimension (Collection / Processing / Retention / Output).
- [ ] Each checkpoint names a specific enforcement mechanism (not "code review will catch it").
- [ ] Each checkpoint names an owner who could actually be paged.
- [ ] Each checkpoint names an evidence source (a CI run link, a dashboard panel, a runbook ID).
- [ ] At least two checkpoints are enforced in code — the two Deliverable-C artifacts.
- [ ] At least one of the three standing tensions is named and resolved in the "tensions" section.
- [ ] YAML and Markdown are synchronised.
- [ ] A reader (DPO or an auditor) can walk the register and understand the current enforcement posture of each dimension.

## Stretch goals

- Author a *drift report* workflow: a scheduled check that snapshots the register, diffs against the previous snapshot, and files a review when the number of `Documented`-only checkpoints grows (controls not being implemented) or when enforced checkpoints go un-fired for >N days (controls going silent).
- Add a checkpoint for **agent-tool argument scoping** that stitches into mod-109: a lint that rejects tool definitions whose argument schema permits passing the entire user record when only an identifier is needed.
- Author a checkpoint pairing for **the three tensions** — one checkpoint that makes the trade-off visible rather than silent (e.g. a release-gate that requires a signed "coverage trade-off" doc when an ingest schema narrows).
- Produce a one-page summary suitable for a product-leadership review: four dimensions, three bars each (Enforced / Instrumented / Documented), as visual evidence of posture.

## What this lands

- The register is the *measures* section of the DPIA (mod-111 Art. 35(7)(d) in GDPR terms).
- The enforcement artifacts are durable — they continue to protect the system after you move on.
- The tensions section is often the most-read piece by legal / DPO; it is where the honest trade-offs live.

Version the register. Date the status column. Review on the same cadence as the datasheet.
