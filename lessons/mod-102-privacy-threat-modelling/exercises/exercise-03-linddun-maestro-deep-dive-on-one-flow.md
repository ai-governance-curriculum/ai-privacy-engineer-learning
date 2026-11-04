# Exercise 03: LINDDUN MAESTRO Deep Dive on One Flow

**Estimated effort:** 4 hours · **Output:** a sub-DFD, a completeness matrix, two threat trees, a scope-decisions log, and a controls table · **Primary chapter:** 05

## Objective

Pick **one flow** from your exercise-01 DFD that warrants depth and run a LINDDUN MAESTRO-shape deep dive on it. Produce the four artifacts chapter 05 specifies: sub-element decomposition, a completeness matrix, two refined threat trees, a scope-decisions log, and a controls table linking each surviving leaf to code and tests.

## Prerequisites

- Chapters 02, 03, 05 read.
- Exercises 01 and 02 complete — DFD and GO register in hand.
- One of the following *flow candidates* that your system actually has:
  - the training-corpus → fine-tune → model-weights path,
  - the retrieval-corpus → vector-index → inference path (multi-tenant if applicable),
  - the inference → log → fine-tune-feedback loop,
  - the inference → cache (prefix / semantic) → inference-serving path,
  - the agent → tool-call → telemetry path.

If your exercise-01 DFD does not have one of these, use the default scenario from mod-101 exercise 01 and pick the retrieval-corpus → inference path.

## Problem statement

Produce a single Markdown document with the following five sections.

### Section A — the sub-DFD

Redraw the chosen flow at a sub-element depth (chapter 05 step 1). For every data store, enumerate sub-stores (e.g. prompt log → `raw_prompt_log`, `evaluation_sample`, `abuse_review_queue`) each with its own retention, ACL, and data classes. For every process, enumerate sub-processes where trust boundaries change (e.g. inference → preprocess / provider-call / postprocess / log-write).

The sub-DFD must have strictly more DFD elements than the equivalent slice of the exercise-01 DFD. If it does not, you have not drilled deep enough.

### Section B — the completeness matrix

A Markdown table. Rows = sub-elements. Columns = the seven LINDDUN categories. Every cell contains one of:

- a **threat ID** linking to Section D's threat register, or
- a **tree ID** linking to Section C's threat tree, or
- `N/A — <explicit reason>`.

A blank cell is a fail.

### Section C — two refined threat trees

Pick two of the most significant threats from Section B (ideally those your exercise-02 top-five identified in this flow) and refine them into threat trees in the shape chapter 05 shows:

```
Root: <threat statement>
  Node 1: <first refinement>
    Leaf 1.a: <concrete sub-threat with a nameable control>
    Leaf 1.b: <concrete sub-threat with a nameable control>
  Node 2: <second refinement>
    Leaf 2.a: …
```

Rules:

- Every leaf passes the three chapter-05 heuristics (named control possible, independently scoreable, independently testable).
- Trees are ≥ 2 levels deep, ≥ 4 leaves total between the two trees.
- No leaf is "add redaction" or "add DP" without the specific library, configuration, and surface named.

### Section D — scope-decisions log

A short sequence of explicit scope decisions in the chapter-05 "Decision D-NN" format. At minimum:

- One decision referencing the security peer's STRIDE pass as the owner of a specific adjacent threat (routed *out* of your scope).
- One decision referencing `ai-risk-engineer` for a broader-harm overlap.
- One decision referencing a non-compliance threat that you cannot resolve in-engineering and that routes to the DPO / legal.

Each decision: who decided, why, when to re-open.

### Section E — controls table

A Markdown table in the chapter-05 shape:

| Threat leaf (tree ID + leaf) | Control (specific, named) | Code / config reference | Test (specific) | Evidence artifact |

Rules:

- At least one row per leaf you produced in Section C.
- "Code / config reference" is a concrete path — `services/cache/key.py#L41` is acceptable; "the cache module" is not. If your real system is private, invent plausible paths and mark them as *proposed*.
- "Test" is a named test or a clear probe — `test_cache_cross_tenant_negative` is acceptable; "unit test" is not.
- "Evidence artifact" names where the control's evidence will live (document, CI run log, audit log).

## Starter guidance

- **Drill the sub-DFD first; the matrix will fill itself out faster.** A thin sub-DFD produces a thin matrix.
- **Write the "N/A — because" entries honestly.** A category is often not applicable to a sub-element, but you must write *why* and the "why" must survive a peer review.
- **Pick threat trees worth refining.** The right two trees are high-stakes threats where GO felt insufficient; the wrong two are threats that are fine at GO depth and refining them is theatre.
- **Stop refining when the three heuristics are satisfied.** An over-refined tree with twenty untestable leaves is worse than a well-formed tree with four actionable leaves.
- **Cite the surrounding modules.** When a control is "DP-SGD", the citation is "mod-106 (DP depth)". When it is "canary evaluation", cite "mod-105". Do not re-explain those depths in this exercise.
- **The scope-decisions log is a feature, not a cop-out.** It is what makes the artifact durable and reviewable.

## Acceptance criteria

- [ ] Sub-DFD has strictly more elements than the exercise-01 slice it refines.
- [ ] Completeness matrix has no blank cells; every cell is a threat-ID / tree-ID / N/A-with-reason.
- [ ] Two threat trees are present, each ≥ 2 levels deep, with ≥ 4 leaves combined.
- [ ] Every leaf in the trees routes to a named (not family-level) control in the controls table.
- [ ] Scope-decisions log has at least three entries covering the security-peer / `ai-risk-engineer` / DPO deferral patterns.
- [ ] Controls table has specific code references (real or marked-proposed) and specific tests.
- [ ] The artifact would be reviewable by a second AI Privacy Engineer without a walkthrough from you.

## Stretch goals

- Run the completeness matrix a second time with a *second person* and compare the deltas. If you are solo, run it once, wait 24 hours, run it again. Gaps surface.
- For each threat tree leaf, score a *residual* risk assuming the proposed control ships and works. Compare to the inherent score.
- Produce a one-page summary of the deep dive suitable for a DPIA appendix. Keep it to 500 words.
- Identify one leaf that would be *impossible* to fix without re-architecture (e.g. "weights have memorised customer PHI; no DP was used at train time"). Document the compensating controls that could be shipped in the interim, and the residual risk you'd accept.

## What this lands

The MAESTRO artifact you produce here is the sort of appendix a DPIA (mod-111) reaches for when a regulator asks "how do you know the semantic cache won't leak across tenants?". It also becomes the long-lived threat-model doc for the flow you chose — kept alongside the code, updated when the design changes.
