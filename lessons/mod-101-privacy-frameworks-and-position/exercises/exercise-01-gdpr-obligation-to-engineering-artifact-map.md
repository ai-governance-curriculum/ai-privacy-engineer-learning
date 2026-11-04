# Exercise 01: GDPR Obligation → Engineering Artifact Map

**Estimated effort:** 2 hours · **Output:** one Markdown table + one one-page narrative · **Primary chapters:** 01, 02

## Objective

Produce a concrete, inspectable mapping from the eight GDPR articles an AI Privacy Engineer must read (Art. 5, 6, 9, 17, 22, 25, 32, 35) to the engineering **controls**, **tests**, and **evidence artifacts** that discharge each obligation **for a specific AI system you choose**. The deliverable is the shape every DPIA you ever write will reuse.

## Prerequisites

- Chapter 01 (how to read privacy law as an engineer) and chapter 02 (GDPR articles as engineering obligations) read.
- The GDPR text (Articles 5, 6, 9, 17, 22, 25, 32, 35) open in another tab from [eur-lex.europa.eu/eli/reg/2016/679/oj](https://eur-lex.europa.eu/eli/reg/2016/679/oj).
- A specific AI system in mind — ideally one you've worked on, built, or can read about in detail. Any of the following work: an internal LLM-powered assistant, a RAG system, a classification or recommendation model touching customer data, a healthcare triage classifier, a fine-tuned LLM you shipped to production, a published vendor case study you can treat as a known system.

If you cannot think of a system to use, use **the following standard scenario** so you can finish the exercise:

> **Scenario (default):** An EU-based retailer runs a generative customer-support assistant. It is a hosted LLM fine-tuned on their support-ticket history. It reads the authenticated customer's profile and order history, retrieves from a tenant-scoped knowledge base, and answers in natural language. All prompts and completions are logged for quality review. The assistant can issue partial refunds up to €50 autonomously; larger refunds are routed to a human agent.

## Problem statement

Produce two deliverables:

### Deliverable A — the obligation / control / test / evidence table

A Markdown table with the following columns:

| Article (and sub-clause) | Obligation (plain-language, engineer-facing) | Where it lands in the AI-system data flow | Control (specific, in-system mechanism) | Test (how we know the control works) | Evidence artifact (what we store) | Owner role (ref. chapter 08) |

Rows (at minimum):

- `Art. 5(1)(a)` — lawfulness, fairness, transparency
- `Art. 5(1)(b)` — purpose limitation
- `Art. 5(1)(c)` — data minimisation
- `Art. 5(1)(d)` — accuracy
- `Art. 5(1)(e)` — storage limitation
- `Art. 5(1)(f)` — integrity and confidentiality
- `Art. 5(2)` — accountability
- `Art. 6(1)(a)` consent, `6(1)(b)` contract, `6(1)(f)` legitimate interests — one row each, applied to the specific processing purposes in your scenario (training vs. inference vs. logging vs. feedback collection are usually different bases)
- `Art. 9(1)` — special-category data; `Art. 9(2)(a)` explicit consent — one row for each if special-category data is in scope; a single "not in scope with justification" row otherwise
- `Art. 17(1)` — right to erasure; `Art. 17(3)` — exceptions where applicable
- `Art. 22(1)` — automated decision-making; `Art. 22(2)` — exception relied on; `Art. 22(3)` — safeguards; `Art. 13(2)(f) / 15(1)(h)` — meaningful information
- `Art. 25(1)` — data protection by design; `Art. 25(2)` — by default
- `Art. 32(1)(a)` pseudonymisation / encryption; `Art. 32(1)(b)` confidentiality/integrity/availability; `Art. 32(1)(d)` regular testing
- `Art. 35(1)` — DPIA trigger analysis; `Art. 35(7)(a-d)` — DPIA content

### Deliverable B — the "defend one hard row" narrative

Pick **one row** from your table that is non-trivial for AI systems — recommended options:

- `Art. 17` right to erasure reaching model weights
- `Art. 22` meaningful-information artifact
- `Art. 32` appropriate security for memorisation / extraction risk
- `Art. 25(2)` privacy-by-default for prompt / completion logging

Write a **one-page narrative** (around 500 words) explaining:

1. What the obligation *actually* asks of your specific system.
2. Why it is non-trivial — what breaks if you build the naive version.
3. The control you'd implement, with enough detail that a senior engineer could code it.
4. The test you'd run to prove it works, with its pass / fail criterion.
5. The evidence you'd store and where (which register, which document, which retention period).
6. The deferral — what parts you cannot own as an AI Privacy Engineer and who you'd route them to (reference chapter 08).

## Starter guidance

- **Resist copying straight from chapter 02.** The chapter gives you the shape; your deliverable must show the shape *applied to your specific system*. Generic "encryption at rest" rows are a fail.
- **Name the system components.** Say "prompt-log sink in S3, 90-day retention, redaction at ingest, access via `prompt-log-reader` IAM role", not "prompt logs".
- **Cite the sub-clause.** `Art. 32(1)(a)` is different from `Art. 32(1)(b)`. Row-per-sub-clause when the obligations differ.
- **Separate processing purposes under Art. 6.** Training is one purpose; inference is another; logging-for-quality-review is another; feedback-collection-for-fine-tuning is another. Each has its own basis.
- **Write the test like a CI job.** "Query retrieval as tenant A, assert zero hits on tenant B" is the right shape. "Verify ACL" is not.

## Acceptance criteria

- [ ] Every one of the eight articles has at least one row (Art. 5 has seven — one per principle).
- [ ] Every row has a **specific-to-your-system** control, not a generic phrase.
- [ ] Every row's test states a *pass / fail criterion*, not just a test name.
- [ ] Every row's evidence artifact names *where it is stored* (document / register / log system) and the retention horizon.
- [ ] The "defend one hard row" narrative addresses all six points in the brief and references chapter 08 for the deferral.
- [ ] You can defend any row to a senior engineer in under two minutes without referencing GDPR text.
- [ ] Any row that depends on something you deliberately cannot own (e.g., the DPO drafting a lawful-basis memo) names the role that owns it per chapter 08.

## Stretch goals

- Add a second table column per row pointing to the equivalent clause(s) in **EU AI Act Art. 10 / 15 / 72** (chapter 03) and **NIST Privacy Framework Subcategories** (chapter 07). The payoff is one piece of evidence serving three audits.
- Pick a row where your control is *incomplete today* (a known gap in your scenario). Add a "gap-and-roadmap" cell describing what it would take to close it.
- Translate your Art. 22 row into both the **CPPA ADMT** (chapter 04) and the **LGPD Art. 20** (chapter 06) shapes. Note what breaks when you compose the three.

## What this lands

The obligation / control / test / evidence shape you produce here is the format every DPIA section in `mod-111-dpia-and-privacy-reviews` reuses. Save your table and the narrative — you will extend them through the rest of the module and the rest of the track.
