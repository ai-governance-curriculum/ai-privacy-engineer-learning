# HIPAA Safe Harbor vs. Expert Determination

HIPAA's Privacy Rule gives two paths to produce *de-identified* health data — data that is no longer Protected Health Information (PHI) and can therefore be used, shared, and (importantly for AI) *trained on* without the Privacy Rule applying. The paths are named in 45 CFR § 164.514(b):

- **Safe Harbor** (§ 164.514(b)(2)) — a checklist. Remove the 18 listed identifiers; attest you have no actual knowledge the remaining information could re-identify. Done.
- **Expert Determination** (§ 164.514(b)(1)) — a statistical argument. An "expert in statistical / scientific principles and methods for rendering information not individually identifiable" certifies that the risk of re-identification is "very small".

For AI workloads, the Safe Harbor / Expert Determination choice is one of the most consequential governance decisions on the training corpus. This chapter covers both paths, when each applies, how to run them, and the AI-era pitfalls that are not in the 2003 rule text.

Primary references:

- 45 CFR § 164.514 — De-identification of protected health information. [ecfr.gov/current/title-45/.../section-164.514](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164/subpart-E/section-164.514).
- HHS OCR (2012). *Guidance Regarding Methods for De-identification of Protected Health Information in Accordance with the HIPAA Privacy Rule.* [hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification](https://www.hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification/index.html).
- NIST SP 800-188 — *De-Identifying Government Datasets.* [doi.org/10.6028/NIST.SP.800-188](https://doi.org/10.6028/NIST.SP.800-188).
- El Emam, K. (2013). *Guide to the De-Identification of Personal Health Information.* CRC Press.
- Narayanan & Shmatikov (2008). *Robust De-anonymization of Large Sparse Datasets* (Netflix Prize attack) — the canonical reason naive de-identification underperforms its claims.
- Sweeney, L. (2015). *Only You, Your Doctor, and Many Others May Know.* Technology Science — zip + DOB + gender re-identification updated.

## Why PHI is its own problem

PHI is a defined category under HIPAA — individually identifiable health information held by a covered entity or business associate. The Privacy Rule generally prohibits use / disclosure without authorisation; the de-identification paths are the carve-outs that let the data be used for research, analytics, AI training, population health, and secondary research without individual authorisation.

For an AI privacy engineer working on a health-AI product, the question "is this PHI?" comes up on:

- The training corpus (de-identified notes, imaging, labs, claims).
- The fine-tune corpus (post-release user traffic that may contain PHI).
- The retrieval index (clinical knowledge base + per-patient context).
- The log sink (prompt / completion of a clinical-LLM product).

Each is a candidate for de-identification *in-pipeline* so the downstream is not PHI. Chapter 09 covers the integration; this chapter covers the de-identification *methodology* HHS recognises.

## Safe Harbor — the 18-identifier list

Safe Harbor is a checklist:

1. **Names.**
2. **All geographic subdivisions smaller than a state**, including street address, city, county, precinct, **ZIP code** — with one exception: the initial three digits of the ZIP may be kept if the three-digit ZIP covers >20,000 people (there is an HHS list of restricted three-digit ZIPs).
3. **All elements of dates (except year)** for dates directly related to an individual, including birth date, admission / discharge / death dates, and all ages over 89 (and dates indicative of such ages).
4. **Telephone numbers.**
5. **Fax numbers.**
6. **Email addresses.**
7. **Social security numbers.**
8. **Medical record numbers.**
9. **Health plan beneficiary numbers.**
10. **Account numbers.**
11. **Certificate / license numbers.**
12. **Vehicle identifiers and serial numbers**, including license plate numbers.
13. **Device identifiers and serial numbers.**
14. **Web URLs.**
15. **IP addresses.**
16. **Biometric identifiers**, including finger and voice prints.
17. **Full-face photographs and any comparable images.**
18. **Any other unique identifying number, characteristic, or code**, except as permitted for re-identification code (§ 164.514(c)).

Plus the **actual-knowledge clause**: the covered entity must not have actual knowledge that the information could be used *alone or in combination* with other information to identify an individual.

If you remove all 18 and have no actual knowledge of residual risk, the data is not PHI.

### Engineering the Safe Harbor pass

For a text-heavy corpus (clinical notes), the Presidio-style pipeline is the mechanism. Map each of the 18 to one or more recognisers:

| Safe Harbor identifier | Primary recogniser(s) |
|------------------------|------------------------|
| Names | `PERSON` (NER) + custom provider-list deny-list |
| Geographic smaller than state | `LOCATION` (NER) + ZIP regex (with 3-digit-ZIP carve-out) |
| Dates (except year) | `DATE_TIME` + date-parser that extracts only year; age > 89 rule |
| Phone / fax | `PHONE_NUMBER` |
| Email | `EMAIL_ADDRESS` |
| SSN | `US_SSN` |
| MRN | Custom pattern recogniser |
| Health plan / beneficiary numbers | Custom pattern per payer format |
| Account numbers | Custom pattern per system |
| Licence / certificate numbers | `MEDICAL_LICENSE` + custom |
| Vehicle IDs | Custom pattern + licence-plate image detector |
| Device IDs | Custom pattern per device vocabulary |
| URLs | `URL` |
| IP addresses | `IP_ADDRESS` |
| Biometric identifiers | Image / audio pipelines (chapter 06) |
| Full-face photographs | Face detector (chapter 06) |
| "Any other unique identifier" | Residual catch-all — org-specific IDs |

Two engineering specifics that bite:

- **The "year of birth for ages >89" rule.** A 93-year-old is identifiable by the age in a small enough population. Clinical data pipelines commonly top-code age at 89 ("90+"), and *only* keep the year for everyone under that.
- **The 3-digit-ZIP carve-out list.** The HHS list of excluded 3-digit ZIPs changes with the census. Hard-code the current list; version it; re-check on each census release.

Produce a **Safe Harbor attestation** as the output artifact:

```yaml
# safe-harbor-attestation.yaml
dataset: clinical-notes-2025-q4
version: 2025-11-04
method: HIPAA Safe Harbor § 164.514(b)(2)
pipeline:
  detector: presidio-2.2.354
  recognisers_catalogue: clinical-recognisers-v8.2
  image_face_detector: retinaface-2024.04
  operators:
    text: replace with entity-type token
    image_face: solid-box
    audio_voice: transcript-only release
identifiers_removed:
  - names: PERSON + provider-deny-list
  - geographic: LOCATION + ZIP (3-digit carve-out applied)
  - dates: DATE_TIME → year only; age > 89 top-coded as "90+"
  - phone: PHONE_NUMBER
  - fax: PHONE_NUMBER (separate pattern)
  - email: EMAIL_ADDRESS
  - ssn: US_SSN
  - mrn: CUSTOM_MRN
  - health_plan: CUSTOM_HEALTH_PLAN
  - account: CUSTOM_ACCOUNT
  - license: MEDICAL_LICENSE
  - vehicle: CUSTOM_VEHICLE + licence-plate image detector
  - device: CUSTOM_DEVICE
  - url: URL
  - ip: IP_ADDRESS
  - biometric: NOT PRESENT in corpus
  - face: face detector pass on attached images
  - other_unique_id: custom pattern recogniser for institution codes
actual_knowledge_review:
  reviewed_by: alice@provider.example
  reviewed_at: 2025-11-03
  statement: "Reviewed a 1000-row random sample for residual re-identification
             signals; see attached review-log. No residual identification path
             identified under the review scope."
eval:
  per_identifier_recall:
    PERSON: 0.94
    DATE_TIME: 0.98
    MRN: 0.96
    # ...
```

The attestation is the DPIA / release-document artifact.

### Known Safe Harbor failures

Safe Harbor is a necessary-but-not-sufficient standard for many AI contexts:

- **Longitudinal data.** Safe Harbor removes direct identifiers but a long sequence of events on the same patient can still re-identify. "Patient presented with left-arm fracture on 2024-02-X, admitted for pneumonia on 2024-06-X, had thyroid surgery on 2024-11-X" is uniquely identifying for most individuals even without name / MRN / ZIP.
- **Free-text narrative.** Clinical notes contain quasi-identifiers embedded in prose — occupation ("truck driver"), family status ("5 children"), unique conditions ("recently returned from the Congo"). Safe Harbor's checklist does not list these; the actual-knowledge clause is the only backstop.
- **Linkage with external datasets.** Netflix Prize taught that aggregate behaviour (ratings over time) re-identifies against external data. A Safe Harbor'd prescription-fill timeline re-identifies against known surveillance-marketing data. The 2008 Narayanan / Shmatikov result is the canonical demonstration.
- **Images / audio that pass the face / voice detector.** A face detector with 95% recall misses 5% of faces. Those are PHI in the released set.
- **Rare conditions.** Safe Harbor does not require removing diagnosis. A rare-disease cohort of 50 patients nationwide is identifying by the diagnosis alone.

Where any of these apply, Safe Harbor is not enough. The regulator's recourse when this becomes an incident is the actual-knowledge clause — "you should have known". The engineering recourse is Expert Determination.

## Expert Determination — the statistical-disclosure path

Expert Determination is a *risk-based* alternative. A person with "appropriate knowledge of and experience with generally accepted statistical and scientific principles and methods for rendering information not individually identifiable" applies such principles and methods and determines that:

- The risk is "very small" that the information could be used, alone or in combination with other reasonably available information, by an *anticipated recipient* to identify an individual.
- The determination is documented in writing.

The expert's determination is time-bound (typically three years) and dataset-bound. A new dataset / a new recipient / three years later requires a new determination.

### What "very small" means in practice

HHS does not define a numeric threshold. The conventional engineering interpretations:

- **Records-at-risk ≤ 5-10%** at the chosen attacker model, with the attacker having realistic (not maximalist) auxiliary data.
- **Expected number of re-identifications ≤ some small fraction** (often 5%) of records — see El Emam et al.'s framework.
- **No single record with re-ID probability > 10%**.

Pick the numeric bar as part of the expert's determination; document the attacker-model assumptions that led to the bar.

### What the expert actually does

A typical Expert Determination engagement covers:

- **Data inventory.** What fields are in the dataset; what each means; distributions.
- **Threat / attacker model.** Who might try to re-identify; what auxiliary data they have; what cost / sophistication.
- **Risk analysis.** Prosecutor / journalist / marketer records-at-risk from a statistical-disclosure-control tool (ARX, sdcMicro, μ-ARGUS).
- **Mitigations.** Generalisation (k-anon-style); perturbation; cell suppression; synthetic substitution; aggregation.
- **Residual-risk statement.** The named scenarios that remain; the threshold applied; the "very small" judgement.
- **Expert's written report.** Signed, dated, credentials documented, methodology detailed.
- **Recipient controls (optional).** The expert may condition the "very small" on contractual recipient controls (no linkage attempts; return / destroy at end; audit right).

For AI training specifically, the expert must also address:

- **Memorisation risk.** A trained model may regurgitate rare patterns from the training set. The expert's analysis should address what the model exposes, not only what the dataset exposes.
- **Fine-tune leakage.** Downstream fine-tunes on user prompts may add new PHI that is not covered by the original determination.
- **RAG retrieval.** A retrieval pipeline over the dataset exposes individual records at query time. The expert's risk model must reflect retrieval access.

### When to go Expert Determination instead of Safe Harbor

- **Rich longitudinal data.** Multiple visits per patient.
- **Rare diseases / small cohorts.**
- **Geographic data finer than 3-digit ZIP needed for utility.**
- **Imaging / audio that cannot be reliably redacted.**
- **Datasets intended for ML training** where memorisation is a genuine risk.
- **Datasets released outside the originating organisation** to parties with potentially rich auxiliary data.

A reasonable org heuristic: if the dataset is for internal analytics on structured data where the Safe Harbor list covers every PHI exposure, Safe Harbor is fine. If the dataset is for LLM training, cross-org sharing, or population health research with external partners, go Expert Determination.

## Beyond HIPAA — other jurisdictions' de-identification regimes

HIPAA is the most engineered of the US regimes. The AI privacy engineer should also know:

- **GDPR anonymous data.** "Information that does not relate to an identified or identifiable natural person" or data rendered anonymous such that the data subject is not or no longer identifiable. GDPR does not have a Safe Harbor list; the Art. 29 WP Opinion 05/2014 on anonymisation techniques is the reference. Three tests: *singling out, linkability, inference*. A dataset that fails any of the three is not anonymous under GDPR.
- **UK ICO Anonymisation Code of Practice** (2012 / 2021 updates) — detailed engineering guidance. See [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/data-sharing/anonymisation](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/data-sharing/anonymisation/).
- **Health Canada / PIPEDA guidance** — similar shape to HIPAA Expert Determination.
- **Japan APPI** — "anonymously processed information" category with specific technical measures required.
- **Australia Privacy Act** — "de-identified information" with similar risk-based framing.
- **State laws (CMIA, My Health My Data Act WA, Health Data Act NY).** Each has its own disclosure control footprint.

For a multi-jurisdiction product, the de-identification pipeline usually targets the strictest applicable regime; the DPIA documents the per-jurisdiction compliance claim.

## The pipeline — making Safe Harbor / Expert Determination composable

An AI pipeline often needs both. Three composable patterns:

### Pattern A — Safe Harbor at ingest; Expert Determination at release

- Ingest-side detector removes the 18 identifiers (chapters 02-04).
- Training proceeds on Safe-Harbor'd data.
- Any downstream release (dataset share; model publication) goes through an additional Expert Determination step, which may include aggregation / further generalisation / synthetic substitution.

This is the conservative composition — Safe Harbor is the *floor*, Expert Determination is the *release gate*.

### Pattern B — Expert Determination end-to-end

- The dataset is de-identified by Expert Determination once, at ingest.
- The de-identified version is what the training / retrieval / inference pipeline sees.
- The original PHI is held in a separate PHI vault with Art. 28 processor controls.

This is the stricter composition — the pipeline never sees PHI. More expensive to set up; cleaner to audit.

### Pattern C — Synthetic-data substitution

- The real PHI dataset trains a generative model (CTGAN / TabDDPM for tabular; sequence model for text; see mod-103 chapter 06).
- The generative model emits synthetic PHI-shaped data for the pipeline.
- Expert Determination certifies that the synthetic dataset's re-identification risk is very small (which typically requires measuring membership inference against the generative model's outputs — a mod-105 technique).

Synthetic substitution has the strongest privacy story when done right and the weakest when done wrong. The MIA audit (mod-105) is the *only* way to make the privacy claim defensible.

## AI-specific pitfalls

Six AI-era pitfalls Safe Harbor / Expert Determination did not originally contemplate:

- **Model memorisation.** The Carlini et al. extraction attacks show that even "de-identified" training data can leak verbatim content. DP-SGD (mod-106) is the mechanism; Expert Determination should require DP for release-time model artifacts.
- **Retrieval pipelines.** A RAG system over a Safe-Harbor'd corpus still exposes individual records (minus the 18) at retrieval time. If an attacker's query targets a specific rare condition, the retrieved chunk may uniquely identify. Per-query retrieval ACL (mod-109) is the complementary control.
- **Fine-tune drift.** User-submitted prompts + completions are fine-tune fodder. PHI enters the pipeline through the user channel; Expert Determination from month 0 does not cover month 6's corpus. Ongoing surveillance of the fine-tune corpus is mandatory.
- **Prompt and completion logs.** Chapter 01's choke-point 4 — logs are the leak. A logged completion that quotes a Safe-Harbor'd patient's year-of-visit + diagnosis + 3-digit-ZIP is re-identifying. The log sanitisation discipline (chapter 09) is the compensating control.
- **Multi-modal release.** A de-identified tabular + unredacted image dataset. The image is PHI; the tabular is not. Compose the two and the image re-identifies the tabular row. Multi-modal datasets need per-modality de-identification pass.
- **Cross-dataset linkage over time.** A longitudinal release with month-level dates on medications tracks the same individual across quarters; the attacker joins the two releases and the Safe Harbor "no year" rule no longer protects. Composability across releases is the formal weakness that differential privacy (mod-106) is the strong answer to.

## The decision matrix

Given a dataset, which path do you choose? A heuristic decision matrix:

| Characteristic | Safe Harbor | Expert Determination |
|----------------|-------------|----------------------|
| Structured tabular, no free-text | Common | Common |
| Rich free-text (clinical notes) | Risky | Standard |
| Longitudinal / multiple visits | Risky | Standard |
| Imaging / audio | Required: image / audio redaction | Common |
| ZIP finer than 3-digit needed | Not possible | Possible |
| Full dates needed | Not possible | Possible with controls |
| Rare conditions (cohorts <~100 nationwide) | Risky | Standard |
| External sharing | Common | Preferred |
| Used for AI training | Insufficient alone | Insufficient alone — compose with DP + memorisation audit |
| Public release | Rare | Common |

## Summary — what you can do after this chapter

- List the 18 Safe Harbor identifiers and map each to one or more recognisers in your detection pipeline.
- Produce a Safe Harbor attestation YAML + a review log.
- Articulate the known failure modes of Safe Harbor (longitudinal data, free-text QIs, external linkage, imaging, rare conditions).
- Describe the Expert Determination process: inventory, threat model, statistical analysis, mitigations, residual-risk, written report.
- Choose between Safe Harbor, Expert Determination, and synthetic substitution based on the dataset's characteristics and downstream use.
- Compose HIPAA-style de-identification with GDPR anonymous-data requirements and with the AI-pipeline controls (DP, MIA audit, RAG ACL, log sanitisation).

The next chapter takes everything in chapters 02-08 and wires it into the AI data path — the five choke-points from chapter 01, now with concrete operators, latency budgets, and FP / FN measurement.
