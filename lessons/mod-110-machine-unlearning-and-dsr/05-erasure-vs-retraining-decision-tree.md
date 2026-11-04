# The Erasure Decision Tree — Retrain, Unlearn, or No Action

Chapters 02–04 gave you three mechanisms: SISA-style exact unlearning, approximate-unlearning on classical models, and LLM-specific gradient-ascent / task-vector / Who's-Harry-Potter techniques. Chapter 01 gave you the per-right engineering runbook. The missing piece is the decision — when a specific Article 17 erasure request lands, *which* mechanism do you actually run?

This chapter is the decision tree. Its outputs are three leaves — *retrain from scratch*, *unlearn* (with a named method), *no engineering action required against the weights*. Its inputs are six questions that a privacy engineer and an `ai-governance-analyst` walk together, each with evidence requirements. The output is logged in the audit trail (chapter 07) so a supervisory authority can cross-check the decision after the fact.

The three leaves exist because the engineering cost difference between them is three orders of magnitude, and the correctness difference is often zero. A thousand-dollar retrain when a no-action was defensible is engineering waste; a no-action when a retrain was required is a compliance failure. The decision tree is the practice that reduces both error modes.

Primary references:

- Regulation (EU) 2016/679, *GDPR*, Articles 17, 19, 23. [eur-lex.europa.eu/eli/reg/2016/679/oj](https://eur-lex.europa.eu/eli/reg/2016/679/oj). The erasure right; Art. 17(1)(a–f) grounds, Art. 17(3) exemptions.
- UK ICO — *Right to erasure* guidance. [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/the-right-to-erasure/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/the-right-to-erasure/).
- UK ICO — *Guidance on AI and data protection* — the section on *How do we comply with individual rights, including rights related to automated decision-making?*
- European Data Protection Board — *Guidelines 05/2020 on consent under Regulation 2016/679*, v1.1 (2020).
- NIST — *Privacy Framework, v1.0*, "Communicate-P" and "Control-P" categories.

## The three leaves — in one line each

- **Retrain.** The record is demonstrably in the training set, the model has demonstrably memorised something from it, and the model's remaining service lifetime is long enough that a retrain's cost is justified.
- **Unlearn.** The record is in the training set but either the memorisation is bounded (small, local, one-sided) or a retrain is infeasible within the DSR clock; apply SISA-exact (if pre-committed), approximate-unlearning (if classical), or LLM-unlearning (if transformer).
- **No action.** The record was either not in the model's training set, or was in a form the model could not meaningfully memorise (heavily de-duplicated, dominant-group-dwarfed, filtered pre-training); the engineering action is limited to corpus-level deletion (chapter 01) and log-level purge (chapter 06).

Each leaf has a different evidence requirement. The job of the decision tree is to route to the right leaf *and* to require the evidence that leaf's claim rests on.

## The six questions

### Question 1 — Was the record in the training set?

The threshold question. Everything downstream depends on this being clearly answered.

- **Evidence source:** the training-snapshot manifest and the subject-ID linkage table (chapter 01). A `grep`-equivalent across the frozen snapshot, scoped to all forms of the subject's identity.
- **Pitfalls:**
  - *Indirect inclusion.* The subject may be mentioned in free-text training data without an explicit subject-ID join. A substring search over the raw text corpus is required in addition to the structured-record search.
  - *Derivative inclusion.* The subject's data may have been included only in a derived feature (an aggregate statistic, a bucket assignment), with no row in the training set itself attributable to them. The derivative is a legitimate concern — if the aggregate carries the subject's signal, the model may still have learnt from it.
  - *Historical snapshots.* The *current* model may be trained on a snapshot that did not include the subject, but a *prior* model (still in production as a canary / A-B variant / fallback) may. Enumerate all model versions currently in service.
- **Output:** a yes/no, with the evidence referenced. If "no", the tree jumps to the no-action leaf and the runbook limits itself to the operational-DB and corpus-level actions in chapter 01.

### Question 2 — Did the model memorise content from the record?

Being in the training set is necessary but not sufficient for an unlearning action. If the record is one of 15T tokens in a frontier LLM pre-train and the subject is low-frequency, the model's weights may not have meaningfully encoded anything about them.

- **Evidence source:** a *memorisation probe*. For an LLM: an extraction-attack prompt that would surface the memorised content if present (mod-105 chapter 05). For a classical model: a membership-inference attack on the record (mod-105 chapter 03).
- **Measurement:** if the probe produces the memorised content with confidence above a threshold (defined per-model in the system card), the memorisation is non-trivial. If the probe produces generic / baseline-like content, the memorisation is below detection.
- **Pitfalls:**
  - *Probe design matters.* A weak probe that does not reach the memorised content is not evidence of absence. The probe must be adversarially designed — paraphrased, prefixed with context, prompted with tokens known to elicit the content — before "no memorisation" is a defensible conclusion.
  - *The MIA baseline.* The MIA AUC on the subject's record must be compared to a matched baseline — an AUC of 0.55 on a specific record might be consistent with "no memorisation" if the model's baseline MIA AUC is 0.55 across the distribution. Use the mod-105 chapter 03 LiRA calibration.
  - *Group-membership effects.* The model may not have memorised the subject's *record*, but may have learnt a *group signal* that identifies them. A subject with a rare condition in a medical corpus is a classic case. Check for group-level inference separately.
- **Output:** memorisation yes/no, with the probe methodology and result referenced.

### Question 3 — Does the model's remaining lifetime justify a retrain?

A retrain costs real money (GPU-time) and real time (clock-weeks). If the model is being retired in two weeks anyway, a dedicated retrain for one DSR is wasteful — the retirement handles the obligation.

- **Evidence source:** the model's deployment roadmap. Is it slated for a periodic refresh in the next N days? Is it being deprecated in favour of a successor?
- **Decision rule (common default):** if the next scheduled refresh is within the DSR clock (one month + any extension), and if the refresh will be trained on a corpus that *will* exclude the subject (i.e. the corpus-level deletion has propagated), then "wait for the refresh" is the defensible action. The audit trail notes the refresh date and the commitment.
- **Pitfalls:**
  - *Refresh slippage.* A refresh scheduled for next month may slip. The runbook must commit to a *backstop* action — if the refresh slips past the DSR clock, escalate to unlearning or an on-demand retrain.
  - *The current model continues to serve during the wait.* Serving an un-retrained model after the DSR clock has notionally expired is a compliance risk. The clock-management must be explicit.

### Question 4 — Does the model-family support a defensible unlearning method?

The three mechanism chapters each named their regime. If the model falls outside those regimes, the decision tree's unlearn leaf is not reachable and the choice collapses to retrain-or-no-action.

- **Classical convex / near-convex model (logistic regression, small MLP, OLS).** Approximate unlearning (chapter 03) is reachable. Preference: certified-removal if available, influence-function otherwise.
- **SISA-shaped pipeline.** SISA-exact unlearning (chapter 02) is reachable. First-choice if committed.
- **LLM / transformer.** Chapter 04 methods are reachable; TOFU-benchmark gating is required.
- **Vision / large deep CNN, non-SISA.** Approximate-unlearning techniques are weakly supported; retrain is the usual fallback. Some task-vector-style variants are emerging but are not yet production-defensible.
- **Non-differentiable model (random forest, GBDT, rule system).** No parametric unlearning; retrain or no-action are the options.

Decision: pick the method whose regime matches the model. If no method matches, the leaf is retrain.

### Question 5 — Does the DSR clock allow a retrain?

If retrain is the method of choice, does it fit?

- **Measure:** retrain wall-clock for the specific model (prior runs' telemetry; include data-prep, training, eval, QA; add buffer).
- **Rule:** if retrain wall-clock > (DSR clock - intake time - any contingency), the retrain cannot be the primary action within SLA. Either (a) negotiate the one-time clock extension under Art. 12(3) via `ai-governance-analyst`, (b) serve the restricted-processing state (chapter 01) during the retrain, (c) escalate to unlearning if a method is available, or (d) document in the audit trail why the extension is necessary and confirm the clock-management decision.
- **Pitfall:** compounding DSRs. Each retrain is a serial cost; if DSRs arrive faster than the retrain cadence, the queue grows. Monitor DSR cadence and plan ahead — if cadence exceeds retrain cadence, invest in SISA or approximate-unlearning before the backlog becomes a crisis.

### Question 6 — Is a retrain *required* by regulation, or does an unlearn suffice?

The regulator's bar in the GDPR Art. 17 context is "erasure". A good-faith argument that approximate unlearning achieves erasure can be made — and has been made by several large providers — but the argument is neither universal nor codified. The interpretation is a DPO / `ai-governance-analyst` decision, not an engineering one.

- **Where unlearn tends to suffice:** low-sensitivity data, post-hoc fine-tunes, deletion of memorised content that was never a primary operational asset, published commitments to a measurement-based erasure standard.
- **Where retrain tends to be required:** special-category data under Art. 9 (health, biometric, political), court-ordered erasure, regulatory enforcement actions, safety-critical systems where "mostly erased" is unacceptable.
- **The engineer's contribution:** the measurable residual-risk number (MIA AUC, TOFU Forget Quality, extraction-probe hit rate) that the DPO uses to make the call.

## The decision tree in flowchart form

```
Q1: Record in training set?
├── No → No action (confirm corpus-level deletion propagated)
└── Yes → Q2

Q2: Model memorised content from record?
├── No (probe-verified) → No action, but document the probe
└── Yes → Q3

Q3: Model being retired or refreshed within DSR clock?
├── Yes, with excluded-subject commitment → Wait for refresh
│                                             (verify at refresh time)
└── No → Q4

Q4: Model family supports a defensible unlearn method?
├── No → Retrain (go to Q5)
└── Yes → Q6

Q5: Retrain fits within DSR clock?
├── Yes → Retrain
└── No → Negotiate extension / restrict processing / escalate to unlearn

Q6: Regulatory / policy posture requires exact erasure?
├── Yes → Retrain (Q5)
└── No → Unlearn (with named method per chapters 02–04)
```

Every arrow in the tree is an audit-trail entry. Every leaf is a runbook invocation.

## The no-action leaf — defending "we did nothing to the weights"

The no-action leaf is the most contested. "You received a deletion request and did nothing" reads badly without evidence. Three defences sit under the leaf:

- **Not-in-training evidence.** The corpus search returned zero matches across all identity forms; the derivative-feature search returned zero contributions; the historical-snapshot search returned zero. Audit-trail-recorded; reproducible.
- **Not-memorised evidence.** The record was in training but the memorisation probe returned baseline-like output (for LLMs: extraction attempts produced generic completions; for classical models: MIA AUC on the record was within noise of the per-model baseline). Audit-trail-recorded with probe methodology.
- **De-duplication / aggregation evidence.** The record contributed only to an aggregate that, by construction, cannot identify them (e.g. the record was one of N = 10,000 in a mean statistic). The construction is auditable.

The no-action leaf *still* requires:

- The *operational-DB deletion* (always; chapter 01).
- The *training-corpus manifest exclusion going forward* (always; chapter 01).
- The *log purge* for the subject's prompts and model outputs about them (chapter 06).
- The *response to the subject* confirming what was done (chapter 07).

What no-action skips is the *model-weights* work. The subject is informed in plain language of the engineering action; the technical detail is in the audit trail.

## The unlearn leaf — picking the method

Once the tree reaches "unlearn", the method choice is a two-dimension selection:

- **By model family:** SISA (pre-committed classical ensemble), approximate-classical (chapter 03), LLM (chapter 04).
- **By method within family:** gradient-ascent vs. task-vector vs. Who's-Harry-Potter for LLMs; influence-function vs. certified-removal vs. Ginart-algorithm for classical.

Decision rules of thumb:

- **SISA is pre-committed or not.** If the pipeline was architected for SISA and the DSR cadence justified the investment, SISA is the first-choice. If not, SISA retrofit is not a real option; move to approximate.
- **Classical models with convex losses.** Certified-removal (Guo et al.) if the (ε, δ) budget allows and the mathematical scaffolding is available; influence-function if certified-removal is not a fit.
- **LLM with narrow deletion target.** Gradient-ascent + KL (Yao et al. variant); verify with TOFU + extraction probe.
- **LLM with named-entity deletion (common for DSRs).** Who's-Harry-Potter (Eldan & Russinovich); verify with TOFU + entity-specific extraction probe.
- **LLM with capability-shaped deletion.** Task-vector arithmetic; this is less common for DSRs — more common for safety-driven unlearning.

Every choice is logged with the per-method config in the audit trail.

## The retrain leaf — when it is right

Retrain is the *correct* answer when:

- The DSR is under Art. 17 with a regulatory emphasis on exactness (special-category data, court order, enforcement).
- The deleted record's memorisation is substantial enough that no unlearn method credibly reduces it below threshold (extraction probes keep surfacing content; MIA AUC remains above the gate after multiple unlearn passes).
- The model's remaining lifetime is long enough that the retrain cost is justified.
- The pipeline does not support a defensible unlearn method.
- Multiple DSRs have accumulated and a single retrain is cheaper than N individual unlearn passes (the "batch retrain" case).

The retrain runbook:

1. Mark the subject as excluded in the training-corpus manifest.
2. Trigger a training job on the current manifest (minus the subject).
3. Run the full eval harness — accuracy, MIA on the subject, extraction probe on known content.
4. If eval passes, promote the new model. If not, diagnose (did the exclusion propagate? did the training config change?) and re-run.
5. Decommission the old model under the retention policy (usually "retain for the Art. 33 challenge window, then destroy with evidence").
6. Record every step in the audit trail.

## The waiting-for-refresh leaf — a specific caveat

"The model will be refreshed next month and the subject will be excluded" is a *defensible* answer *only if*:

- The refresh is **genuinely scheduled** with a dated commitment, not aspirational.
- The subject's **corpus-level deletion has propagated** to the pre-refresh training snapshot — so the refresh will not re-ingest them.
- The **DSR SLA permits the wait** (usually within the one-month clock plus any agreed extension).
- The **restricted-processing interim** (chapter 01) is applied during the wait if the subject's inferences continue being produced on live data.
- The **backstop action** (unlearning or on-demand retrain) is identified in advance if the refresh slips.

The audit trail records the refresh date, the commitment, the backstop, and the actual outcome (refresh completed on time, or backstop invoked).

## The extension — negotiating the clock

Art. 12(3) permits a two-month extension for complex requests. Engineering grounds for an extension:

- *Retrain required, retrain wall-clock exceeds one month.* A large model (a 70B LLM full retrain is weeks), a compute-constrained environment, a complex eval suite.
- *Multiple deletions require coordination.* A batched retrain is more defensible than ten retrains.
- *Verification complexity.* A novel method requires standing up a benchmark suite.

The extension is a *legal* step owned by the DPO / `ai-governance-analyst`. The engineering input is the wall-clock estimate and the specific reason — both logged. The extension notice to the subject is their artefact to draft.

## Multi-corpus coordination

A subject may appear in multiple training corpora feeding multiple models. The decision tree is run *per model*, but the answers may interact:

- **Shared corpus.** Two models trained on the same corpus — the corpus-level deletion propagates to both; each model's leaf is still per-model.
- **Downstream-derived corpora.** A synthetic dataset derived from the primary corpus may need its own deletion pass (chapter 01's derivative-products row).
- **Fine-tune chain.** A base model fine-tuned into a product model — unlearning from the base model does not automatically propagate to the fine-tune; apply the decision tree to the fine-tune separately.
- **Cross-system aggregation.** A metric dashboard fed by multiple model inferences may need log-purge on each contributing system (chapter 06).

Multi-corpus coordination is where the no-action leaf is most dangerous: "no action on the model" without "action on the derivative" is incomplete. The chapter-01 inventory and the chapter-07 audit trail together are the accountable artefacts.

## The decision-tree system-card entry

A model's system card should include an "Erasure Posture" section that, for each pre-anticipated DSR shape, names the default leaf of this tree and the reasoning:

- "For single-subject Art. 17 requests where the subject was a sparse mention, default is **no action** with a memorisation-probe check; evidence fields E1, E2."
- "For single-subject Art. 17 requests where the subject was a dense mention, default is **unlearn via gradient-ascent + KL**; verification against TOFU-equivalent."
- "For court-ordered or Art. 9 special-category erasures, default is **retrain**."
- "Refresh cadence is Q, so wait-for-refresh is an option only when a request arrives within Q - 2 weeks of a refresh."

This is the artefact the DPO reviews when a DSR arrives and the engineer runs the tree.

## Summary — what you can do after this chapter

- Walk the six-question decision tree for a specific DSR: evidence for Q1 (in-training), evidence for Q2 (memorised), evidence for Q3 (refresh), feasibility for Q4 (method exists), feasibility for Q5 (retrain fits), interpretation for Q6 (retrain required).
- Reach one of three leaves (retrain, unlearn-with-method, no-action) and name the evidence required for the leaf's defensibility.
- Document the no-action leaf — the probe, the baseline, the de-duplication argument — such that a supervisory authority sees a positive answer rather than silence.
- Pick the unlearn method within the leaf, with the chapter-02/03/04 regime decision rule.
- Coordinate the tree across multiple models and derivative corpora.
- Author the "Erasure Posture" system-card entry that pre-answers the common DSR shapes for a given model.

Chapter 06 picks up a different corpus surface — RAG and agent systems — where DSR is easier than against the base model but harder than against the operational database.
