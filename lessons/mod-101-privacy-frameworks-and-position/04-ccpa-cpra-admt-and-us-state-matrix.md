# CCPA / CPRA, the CPPA ADMT Rules, and the US State Privacy Matrix

The US has no federal comprehensive privacy law. In its absence, California set the baseline and twenty-plus other states have followed. An AI Privacy Engineer must be able to read the California baseline — including the California Privacy Protection Agency's **Automated Decisionmaking Technology (ADMT)** regulations — and then diff every other state law against it instead of reading each state from scratch.

Primary sources:

- **CCPA / CPRA** — [California Consumer Privacy Act / California Privacy Rights Act (Cal. Civ. Code § 1798.100 et seq.)](https://leginfo.legislature.ca.gov/faces/codes_displayText.xhtml?lawCode=CIV&division=3.&title=1.81.5.&part=4.&chapter=&article=) and the [CPPA regulations](https://cppa.ca.gov/regulations/).
- **CPPA ADMT rulemaking** — the CPPA has issued regulations on automated decisionmaking technology, risk assessments, and cybersecurity audits; the regulation text and effective dates move on the CPPA's calendar. **Always pull the current effective text from [cppa.ca.gov/regulations](https://cppa.ca.gov/regulations/) before you cite a specific clause** — do not rely on a date-stamped version from training data or a secondary summary. <!-- needs-research: refresh ADMT effective date and clause-numbering against the then-current CPPA regulation text at every curriculum review cycle. -->
- **IAPP US State Privacy Legislation Tracker** — [iapp.org/resources/article/us-state-privacy-legislation-tracker](https://iapp.org/resources/article/us-state-privacy-legislation-tracker/). The industry reference for cross-state comparison.

> This chapter does not deliver legal opinion. Count thresholds, applicability, effective dates, and clause-level language *change*. Always refresh against the authoritative source at each review.

## The California baseline

### Who is covered

The CCPA (as amended by CPRA, in force since 2023) applies to a **business** that does business in California, collects consumers' personal information, determines the purposes and means of processing, and meets at least one applicability threshold:

- gross annual revenue above a statutory amount (the dollar figure is periodically adjusted — refresh the current figure from the CPPA site before citing), **or**
- buys, sells, or shares the personal information of a threshold number of California consumers or households, **or**
- derives a threshold percentage of annual revenue from selling or sharing consumers' personal information.

Three secondary roles defined by CCPA:

- **Service provider** — processes personal information on behalf of a business under a written contract that satisfies CCPA requirements.
- **Contractor** — similar to service provider but under a different contract form; same effect.
- **Third party** — anything else that receives personal information.

### Consumer rights (CCPA / CPRA core)

- **Right to know** — categories of personal information collected, sources, business / commercial purposes, categories of recipients.
- **Right to delete** — subject to enumerated exceptions (fraud detection, security, legal compliance, internal use compatible with the context of collection, etc.).
- **Right to correct** (CPRA addition) — inaccurate personal information.
- **Right to know specific pieces of personal information collected**.
- **Right to opt out of sale / sharing** — "sharing" was added by CPRA specifically to cover cross-context behavioural advertising, i.e. targeted ads.
- **Right to limit use and disclosure of sensitive personal information** (CPRA addition).
- **Right to opt out of ADMT and to access information about ADMT** (added via CPPA ADMT regulations).
- **Right to non-discrimination** for exercising rights.

### Sensitive personal information (CPRA)

CPRA defines "sensitive personal information" (SSN, driver's license, financial account, precise geolocation, racial / ethnic origin, religious / philosophical beliefs, union membership, mail / email / text contents not addressed to the business, genetic data, biometric data for unique identification, health, sex life / sexual orientation). Consumers can limit its use. Engineer consequence: your sensitive-PII lane (`mod-104-pii-detection-and-redaction`) needs California scope, not just GDPR Art. 9 scope — the lists overlap but are not identical.

### Core engineer-facing obligations

- **Privacy notice** — your notice-at-collection and privacy policy content (legal / governance owns the drafting, you own the data-sources that make the notice accurate).
- **Service-provider contracts** — purpose-limited, no-sale, confidentiality, right-to-audit.
- **Opt-out signals** — honour the **Global Privacy Control (GPC)** as an opt-out-of-sale/sharing signal. Engineer the browser-signal path.
- **Risk assessments** — the CPPA requires risk assessments for processing presenting significant risk to consumers' privacy; AI / ADMT processing is specifically called out. The content is adjacent to a GDPR-shape DPIA.
- **Cybersecurity audits** — annual cybersecurity audits are required for businesses whose processing presents significant risk; the CPPA regulation sets the content requirements.

## Automated Decisionmaking Technology (ADMT)

The CPPA ADMT regulations are the California analogue of GDPR Art. 22 — with important shape differences.

### The engineer's reading

- **Scope.** The regulations cover automated decisionmaking technology used to make a *significant decision* about a consumer — including decisions regarding financial / lending services, housing, insurance, education, employment, healthcare, essential goods or services, criminal-justice-adjacent contexts — *and* certain uses like behavioural advertising / profiling of minors / extensive profiling / training ADMT on personal information. The exact in-scope list is set out in the CPPA regulation — read the current effective text.
- **Pre-use notice.** The business must issue a pre-use notice describing the ADMT — the purpose, how it works at a plain-language level, and consumer rights.
- **Right to opt out.** Consumers have a right to opt out of ADMT use for significant decisions, with narrow exceptions (e.g., human appeal available; security / fraud prevention).
- **Right to access ADMT information.** Consumers can request information about the ADMT used — the output, its role in the decision, the key parameters / logic, and (in some iterations of the regulation) training-data characteristics.
- **Risk assessment.** ADMT in scope triggers a specific risk-assessment requirement with CPPA-prescribed content and retention.

### Engineering consequence

ADMT shares the **human-review + contestation + meaningful-information** engineering surface with GDPR Art. 22 — build it once and route both statutes to it. Specific additions for California:

- **ADMT inventory** — register each in-scope use of automated decisionmaking with the pre-use notice content baked in.
- **Pre-use notice delivery** — the notice is a product surface, not a buried privacy-policy paragraph.
- **Opt-out mechanism** — a product path for the consumer to opt out of ADMT or request human review, with routing to a human reviewer with authority to overturn.
- **Access to ADMT information** — a response template with the required elements. (Analogous to GDPR Art. 15(1)(h) "meaningful information about the logic involved".)
- **Risk assessment** — authored to CPPA content requirements, retained for the CPPA-prescribed period.

### Control / test / evidence

| ADMT requirement | Control | Test | Evidence |
|---|---|---|---|
| Pre-use notice | Notice generation per decision type; A/B-free rollout gate | UI-rendering test; audit that the notice renders before the decision | Rendered-notice sample per release |
| Opt-out of ADMT | Opt-out endpoint; routing to human reviewer | E2E test: opt-out request → human decision path | Opt-out-event log |
| Access to ADMT information | Response template; per-decision explanation generation | Response-content check against CPPA fields | Sample response + content-check log |
| Risk assessment | ADMT risk-assessment document per decision type | Peer review of content against CPPA content list | The risk-assessment document |
| Cybersecurity audit | Annual audit per CPPA | Audit workpaper review | Audit report |

## The 20+ state matrix

As of the research date, every state comprehensive privacy law shares a common skeleton — some variation of CCPA's rights, notice, contract, and risk-assessment shape. The IAPP tracker lists the enacted laws; the set this track calls out:

**VCDPA** (Virginia), **CPA** (Colorado), **CTDPA** (Connecticut), **UCPA** (Utah), **OCPA** (Oregon), **TDPSA** (Texas Data Privacy and Security Act), **TIPA** (Tennessee Information Protection Act), **NJDPA** (New Jersey), **FDBR** (Florida Digital Bill of Rights), **NHIPA** (New Hampshire), **MDPA** (Maryland Online Data Privacy Act), **MCDPA** (Minnesota Consumer Data Privacy Act), **DPDPA** (Delaware Personal Data Privacy Act), **RICDPA** (Rhode Island), **INCDPA** (Indiana). More states are landing every session; **always refresh the current enacted set via the IAPP tracker before you rely on a count** — specific names, abbreviations, and effective dates move. <!-- needs-research: keep the enacted-state list, abbreviations, effective dates, and distinctive-clause notes current against the IAPP tracker at each review cycle. -->

### The shared skeleton — what to expect in almost every state law

- **Applicability thresholds** — processing threshold on number of residents (often 100,000 residents or 25,000+ residents if a percentage of revenue is from sale) and sometimes revenue; exemptions for small business. HIPAA-, GLBA-, FERPA-regulated processing is commonly carved out (entity-level or data-level — differs by state).
- **Consumer rights** — access, deletion, correction, data-portability, opt-out of sale, opt-out of targeted advertising, opt-out of profiling in furtherance of decisions that produce legal or similarly significant effects.
- **Sensitive data** — concept exists in most state laws with slightly different definitions. **Most state laws require opt-in consent to process sensitive data**, which is a meaningful shape difference from CCPA's opt-out-of-use.
- **Controller / processor model** — most state laws use GDPR-style controller / processor terminology rather than CCPA's business / service provider.
- **Data protection assessment (DPA)** — required for high-risk processing including targeted advertising, sale, sensitive data, profiling with significant effects. Content is adjacent to GDPR-shape DPIA.
- **Universal opt-out mechanism (UOOM)** — several states (CO, CT, OR, TX, MT, NH, NJ, DE, MD, MN) require honouring universal opt-out signals such as GPC for sale / targeted advertising. CCPA requires honouring GPC specifically.
- **Profiling restrictions** — most state laws give consumers an opt-out of profiling in furtherance of decisions that produce legal or similarly significant effects. This is the GDPR Art. 22 / CPPA ADMT shape in a different legislative frame.

### Where the states diverge from CCPA

The CCPA baseline is distinctive in several ways; other states usually look *more like each other* and *less like CCPA*:

- **Private right of action.** CCPA grants a private right of action for a narrow class of data-breach incidents involving unencrypted or non-redacted personal information. Most other state laws vest enforcement solely in the state AG (or the CPPA for California).
- **Business / service provider vs. controller / processor.** Shape difference in contracts and in who-is-liable. In practice the engineer still needs purpose-limited, auditable contracts.
- **Sensitive data default.** CCPA = right to limit use. Most others = opt-in to process.
- **Age-related opt-in / opt-out of sale.** CCPA has opt-in to sale for ages 13–16 and parental consent for under 13. Other states differ.
- **Cure period / enforcement discretion.** Some state laws include a cure period before enforcement; several of the newer laws are sunsetting the cure period.
- **Dedicated data-broker registration.** Several states (CA, VT, TX, OR) have separate data-broker-registration laws on top of the comprehensive privacy regime.

### State-specific privacy-engineering flags for AI

Several recent laws name AI or profiling specifically:

- **Colorado AI Act (SB 24-205)** — the first US *state* general-purpose AI law of consequence, covering high-risk AI systems with a GDPR-adjacent risk-assessment and anti-discrimination framing. Relevant to AI Privacy Engineering where high-risk AI touches a Colorado consumer. <!-- needs-research: refresh effective date and current regulation / amendment status for the Colorado AI Act. -->
- **Texas TDPSA** — opt-out of profiling in furtherance of solely-automated decisions.
- **Several laws** list "profiling" in their DPA-required list, which is your DPIA hook for AI.

### Diffing a new state law

When a new state law lands, run this diff pattern (10–20 minutes):

1. **Applicability** — do we meet the threshold?
2. **Carve-outs** — HIPAA / GLBA / FERPA / employment data entity vs. data level?
3. **Rights set** — any right not already covered by our existing CCPA pipeline?
4. **Sensitive data** — opt-in or opt-out? Which of our processing touches it?
5. **DPA / risk-assessment** — is the content substantively different from our DPIA template?
6. **Universal opt-out mechanism** — required? Already honoured?
7. **Profiling / ADMT** — in scope? Overlap with our Art. 22 / ADMT surface?
8. **Enforcement** — private right of action? AG only? Cure period?
9. **Timeline** — effective date?

That diff becomes the state's row in your state-matrix spreadsheet. The engineer's job is: *add to the existing pipeline the one or two things the state law wants that CCPA + our pipeline do not already give*.

## Control / test / evidence for the state matrix

The pattern is "**one pipeline, many statutes**":

| Capability | Serves | Engineering control | Test | Evidence |
|---|---|---|---|---|
| Rights-handling intake | CCPA, all state laws, GDPR, PIPL, LGPD | DSR intake + routing + clock | E2E DSR dry-run per statute | DSR audit trail |
| Universal opt-out (GPC) | CCPA, CO, CT, OR, TX, NH, NJ, DE, MD, MN | Signal detector at edge; downstream propagation | GPC header replay test | GPC-honoured log |
| Sale / sharing control | CCPA, all | Opt-out-of-sale flag on consumer record | Vendor-outbound filter test | Suppression-list audit |
| Sensitive-data processing | CPRA, state matrix (opt-in), GDPR Art. 9 | Separate access path; opt-in record | Per-tenant opt-in audit | Consent register |
| Profiling / ADMT opt-out | CPPA ADMT, state matrix, GDPR Art. 22 | ADMT inventory; human-review path; meaningful-information artifact | E2E opt-out → human decision | Decision log |
| Data protection assessment | CPPA, state DPAs, GDPR Art. 35 | DPIA template that absorbs every required content element | Content-completeness check against the longest-form requirement | DPIA document |

The deliverable is **one pipeline that satisfies all applicable statutes**, instrumented with per-statute evidence slices.

## Depth boundary

- **Operational state-law tracking** — "when did New Hampshire's law take effect, what changed in Colorado last month" — is `ai-governance-analyst` (level 15) scope and consumed by this role. You should know the shape; your analyst colleagues should own the daily-tracking legwork.
- **Cross-jurisdiction policy reconciliation** — authoring a control library that discharges CCPA + CPRA + state matrix + GDPR + sector law + PIPL coherently — defers up to `senior-ai-governance-architect` (level 50).

## Summary

- CCPA / CPRA is the US baseline. The CPPA ADMT regulations are California's answer to GDPR Art. 22 with a specific pre-use-notice, opt-out, access, and risk-assessment shape.
- The 20+ US state laws share a common skeleton — rights, opt-out of sale / targeted advertising, DPA for high-risk processing, often opt-in to sensitive data — with specific per-state wrinkles.
- Build *one* pipeline for rights, opt-out, DPA, and ADMT and instrument it with per-statute evidence slices. Do not build a California pipeline and a separate Virginia pipeline.
- Refresh counts, dates, and clause citations against primary sources every review cycle.

Next: the sector-specific US laws — HIPAA, FERPA, COPPA, GLBA.
