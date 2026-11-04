# How to Read Privacy Law as an Engineer

Privacy regulation is written for lawyers, not engineers. You need to translate it. This chapter is the mental model you will use for every statute, framework, and regulator opinion in the rest of this module — and in the rest of the track.

## Why this matters

As an AI Privacy Engineer (level 30, AI Governance family) you do **not** issue legal opinion. The DPO and legal counsel do that. You *consume* their reading of the text and convert it into:

- **Controls** — mechanisms in the system that make the obligation enforceable (ACLs, DP noise, redaction, retention jobs, consent-propagation metadata, attestation-bound inference).
- **Tests** — repeatable checks that the control works (unit tests, integration tests, red-team probes, MIA / extraction harnesses, DPIA walkthroughs).
- **Evidence artifacts** — durable records a supervisory authority or auditor can read later (DPIA documents, ROPA entries, DP accountant logs, PII-pipeline F1 reports, attestation quotes, consent-register dumps).

Everything you read in this module should land in that three-column shape: *obligation → control → test → evidence*. When the shape is incomplete you have a gap, not an opinion.

## The engineer's reading pass

When you first read a privacy statute or framework, run this pass:

1. **Identify the actors.** Who is the *data subject* (GDPR), *consumer* (CCPA), *individual* (HIPAA), *data principal* (PIPL), *titular* (LGPD), *student* (FERPA), *child* (COPPA)? Who is the *controller / processor* (GDPR), *business / service provider / contractor / third party* (CCPA), *covered entity / business associate* (HIPAA)? Who is the *supervisory authority / regulator*? The actor model determines where in your architecture an obligation lands.
2. **Identify the processing scope.** What triggers the law? Residency, nationality, sector, purpose, volume, revenue thresholds, is-it-automated-decision-making, is-it-high-risk-AI. In US state laws this is usually a revenue / records / sale-of-data threshold. In GDPR it is EU-based data subjects or EU establishment. Know the trigger so you can tell product "this ships us into Article 22" or "this is below the Colorado threshold".
3. **Identify the obligation class.** Most obligations fall into a handful of engineering-recognisable shapes:
   - *Lawful basis* — do you have the right to process at all?
   - *Transparency* — can the data subject see what you did?
   - *Rights-handling* — can the data subject act on their data (access, erase, rectify, port, object, opt-out, opt-out-of-ADMT)?
   - *Purpose / minimisation / retention* — did you collect only what you need and dispose of it?
   - *Security* — are the technical and organisational measures appropriate?
   - *Risk assessment* — have you documented your reasoning before high-risk processing?
   - *Breach response* — can you detect, notify, and remediate on the statutory clock?
   - *Cross-border* — do you have a mechanism (SCCs, adequacy, binding corporate rules) for the data leaving the jurisdiction?
4. **Find the article hook.** Every obligation points to one or more specific article / rule / section. Cite it. "Right to erasure" is `GDPR Art. 17`; "Automated decision-making" is `GDPR Art. 22` and (separately) the CPPA ADMT regulations; "DPIA" is `GDPR Art. 35` + `ISO/IEC 29134` + `NIST PRAM`. If you cannot cite the hook, you do not understand the obligation yet.
5. **Map to the AI-system data flow.** The system is training corpora → retrieval corpora → prompts → model → completions → logs → telemetry → derived artifacts (embeddings, caches, fine-tune deltas, LoRA adapters, memory). Overlay the obligation onto that flow. "Right of erasure" is cheap on an application DB and expensive on model weights. "Lawful basis for training" is a different fight than "lawful basis for inference". "Security of processing" is different between a hosted LLM API and an on-prem fine-tune.

## Three worked examples

### Example 1 — GDPR Art. 25 "Data Protection by Design and by Default"

- **Obligation class:** engineering-process obligation; defaults must be the privacy-preserving option.
- **Article hook:** GDPR Art. 25.
- **AI-flow mapping:** training-corpus intake (collect the minimum), retrieval (scope per tenant), prompt (do not log raw PII by default), logging (retention + redaction), model (DP / minimisation / anonymisation where feasible).
- **Control:** *default-off* logging of raw prompts, *default-on* redaction in prompt and completion sinks, per-tenant retrieval ACL, documented data-minimisation review before any new field enters training.
- **Test:** CI job that greps staging logs for a canary PII string and fails the build if it appears; integration test that queries retrieval as tenant A and asserts zero hits on tenant B's documents.
- **Evidence artifact:** data-minimisation review document per model version, PII-pipeline F1 report, retrieval-ACL negative-test log, DPIA Section 25 with a cross-reference table.

### Example 2 — GDPR Art. 17 "Right to Erasure"

- **Obligation class:** rights-handling + storage.
- **Article hook:** GDPR Art. 17 (and Art. 12 for the handling clock).
- **AI-flow mapping:** operational databases (easy), object store blobs, backup / snapshots (mind the retention tail), training corpora (hard — have you snapshotted the corpus?), **model weights** (very hard — this is `mod-110-machine-unlearning-and-dsr` depth), retrieval index + embeddings (medium — re-index on erasure), logs / traces (medium — redact or expire), derived LoRA deltas / fine-tunes (hard — retrain or unlearn).
- **Control:** DSR intake + routing, per-subject index of their data-presence across every sink, erasure job per sink, unlearning decision tree for weights, deletion-attestation for processors.
- **Test:** end-to-end DSR dry run with a synthetic subject; cross-sink audit that counts residual records; on-model residual membership-inference check if weights were in scope.
- **Evidence artifact:** DSR runbook, deletion-attestation log, unlearning decision record, post-erasure MIA report.

### Example 3 — EU AI Act Art. 10 "Data and Data Governance"

- **Obligation class:** high-risk-AI data-governance obligation.
- **Article hook:** EU AI Act (Regulation 2024/1689) Art. 10.
- **AI-flow mapping:** training / validation / test datasets for high-risk AI systems; the governance applies regardless of whether personal data is present.
- **Control:** dataset statement (Datasheet-shape), documented data-collection processes, relevance / representativeness / accuracy / completeness assessment, examination for bias, documented data-preparation, and — specifically — handling of special-category personal data to the extent *strictly necessary* for bias monitoring, with safeguards.
- **Test:** dataset-card completeness check in CI; bias examination report; special-category-data access-log audit.
- **Evidence artifact:** per-dataset Datasheet, bias-examination report, data-governance section of the EU AI Act Annex IV technical documentation (owned next-up by `ai-evaluation-engineer`, level 35).

## Primary text first, commentary second

Do not read privacy law via blog posts. Read:

- The statute text itself (eur-lex for EU, govinfo / state AG sites for US, ANPD for LGPD, NPC for PIPL).
- The regulator guidance that the statute points to (EDPB opinions, CNIL AI sheets, UK ICO AI and PETs guidance, HHS OCR guidance, FTC business guidance).
- The applicable standard (NIST Privacy Framework, NIST SP 800-226, ISO/IEC 27701, ISO/IEC 29134, ISO/IEC 42001).

Everything else — IAPP reference articles, vendor whitepapers, standard textbooks — is useful second-order reading. It is not the authority you cite when you write a DPIA or defend an audit finding.

## What a mature reading looks like

A senior AI privacy engineer, when asked "does this ship us into GDPR Art. 22?", does not say "yes" or "no". They say:

> "The use-case produces an automated decision with legal or similarly significant effects for the data subject under Art. 22(1). We can keep it inside Art. 22(2)(a) (contract-necessary), 22(2)(b) (authorised by law), or 22(2)(c) (explicit consent). We've picked 22(2)(c). Legal has drafted the consent language; I own the engineering surface to prove consent was collected, that the subject can obtain human intervention under 22(3), and that we can show the logic involved per Art. 13(2)(f) / 15(1)(h). My DPIA section on this is here, my test harness for the human-intervention button is here, my meaningful-information artifact is here."

The rest of this module teaches you to speak like that.

## Summary — what you can do after this chapter

- Convert any privacy obligation into the *obligation → control → test → evidence* shape.
- Identify the five actors in any statute and the trigger for processing scope.
- Classify an obligation into lawful-basis / transparency / rights / minimisation / security / assessment / breach / cross-border.
- Cite the article hook for any claim you make.
- Overlay the obligation onto the AI-system data flow and know where it lands cheaply vs. expensively.

The next chapter does this for GDPR end-to-end.
