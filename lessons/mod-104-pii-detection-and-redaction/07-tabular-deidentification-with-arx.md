# Tabular De-identification — k-Anonymity, l-Diversity, t-Closeness with ARX

Span-level redaction removes the obvious identifiers. A table that has already lost name, email, phone, and SSN can still re-identify individuals when `(ZIP, date-of-birth, gender)` or `(diagnosis, visit-date, hospital)` is enough to pinpoint one row. The Latanya Sweeney finding — that 87% of the US population was uniquely identifiable from `(ZIP5, date-of-birth, gender)` in the 1990 census — is the textbook case. This chapter covers the mathematics of tabular de-identification (k-anonymity, l-diversity, t-closeness), how ARX ([arx.deidentifier.org](https://arx.deidentifier.org/)) ships them as an engineering tool, and how to defend the chosen k / l / t in a DPIA.

Primary references:

- Sweeney, L. (1997). *Weaving Technology and Policy Together to Maintain Confidentiality.* Journal of Law, Medicine & Ethics. (The ZIP-DOB-gender re-ID finding.)
- Samarati & Sweeney (1998). *Protecting Privacy when Disclosing Information: k-Anonymity and Its Enforcement through Generalization and Suppression.* SRI TR.
- Sweeney, L. (2002). *k-Anonymity: A Model for Protecting Privacy.* IJUFKS.
- Machanavajjhala et al. (2007). *l-Diversity: Privacy Beyond k-Anonymity.* ACM TKDD.
- Li, Li, Venkatasubramanian (2007). *t-Closeness: Privacy Beyond k-Anonymity and l-Diversity.* ICDE.
- ARX anonymization tool — [arx.deidentifier.org](https://arx.deidentifier.org/).
- `sdcMicro` — R-based statistical disclosure control. [cran.r-project.org/package=sdcMicro](https://cran.r-project.org/package=sdcMicro).
- Python alternatives — `anonypy`, `pycanon`, `pyarxaas` (ARX via API).
- NIST SP 800-188 — *De-Identifying Government Datasets.* [doi.org/10.6028/NIST.SP.800-188](https://doi.org/10.6028/NIST.SP.800-188) — the single most readable overview.
- Dwork, C. (2006). *Differential Privacy.* — the mod-106 bridge; k-anon is not DP.

## The vocabulary

Four classes of attributes. The de-identification exercise is a decision per attribute.

- **Direct identifiers.** Name, email, SSN, MRN, phone. Removed or vault-tokenised before the de-identification step. Chapters 02-04 handle these.
- **Quasi-identifiers (QIs).** Attributes that are not individually identifying but jointly re-identify. ZIP, DOB, gender; hospital-ID + visit-date; employer + role + city. The whole de-identification mathematics is about QIs.
- **Sensitive attributes.** The thing an attacker wants to learn about the subject — diagnosis, salary, voting record. Not removed; the de-identification assumes the attacker learns the sensitive value iff the row is re-identified.
- **Non-sensitive attributes.** Attributes that are neither QI nor sensitive. Usually pass through unchanged.

Classifying each column is step zero. A QI misclassified as non-sensitive is a re-identification vector; a sensitive attribute misclassified as QI loses utility. Record the classification in a dataset dictionary — the ARX project file carries it.

## k-Anonymity

A table satisfies **k-anonymity** if, for every record, the combination of QIs is shared by at least `k` records (the record is indistinguishable from `k-1` others on the QI set). Formally: for every row `r` in the released table, `|{r' : r.QI = r'.QI}| >= k`.

Achieve k-anonymity with two operations:

- **Generalisation.** Replace a value with a less specific one that covers it. `age=41` → `age ∈ [40, 44]`; `ZIP=94040` → `ZIP=940**`.
- **Suppression.** Remove the record (or the cell) entirely.

k-anonymity's attacker model is a **linkage attack**: the attacker has a separate dataset with known individuals' QIs and tries to join. k-anonymity bounds the linkage to groups of `k`.

### Choosing k

- **k = 2 is uselessly weak** — your record is distinguishable from one other, which usually is not an indistinguishability at all.
- **k = 5** is the de facto regulator-citeable minimum. HHS OCR Expert Determination opinions (chapter 08) have accepted k=5 for public-facing releases.
- **k = 10** is the common bar for public-release health data; higher where the data is especially sensitive.
- **k = 25+** is appropriate for extremely sensitive releases or low-trust environments.

The right k depends on:

- **Sensitivity of the attributes** — a diagnosis column justifies a higher k than a hair-colour column.
- **Attacker auxiliary information** — an attacker who already has voter rolls needs a higher k to defeat.
- **Dataset size** — a small dataset at high k loses rows to suppression; the utility collapses.
- **Legal context** — some regulators / contracts specify a minimum.

Defend the chosen k in the DPIA: *what attacker model, what auxiliary data assumed available, what attack cost*.

### Failure modes of k-anonymity

Two attacks k-anonymity does *not* defend against:

- **Homogeneity attack.** All `k` records in a QI-equivalence class share the same sensitive value. Example: a k=5 class in which all 5 have the diagnosis "HIV+". The attacker learns the diagnosis without needing to pick out which record. This motivates l-diversity.
- **Background-knowledge attack.** The attacker knows, from outside the dataset, that their target is unlikely to have a specific sensitive value; this narrows the possibilities within the class. This motivates t-closeness.

## l-Diversity

A QI-equivalence class is **l-diverse** if the sensitive attribute has at least `l` well-represented values. Three versions in the literature:

- **Distinct l-diversity.** At least `l` *distinct* values of the sensitive attribute in every class.
- **Entropy l-diversity.** The entropy of the sensitive attribute distribution within every class is at least `log(l)`.
- **Recursive (c, l)-diversity.** The most-common sensitive value appears no more than `c` times the sum of the least-common `(l - 1)` values.

Recursive and entropy are stronger; distinct is what most practitioners use because it is the one ARX exposes directly.

### Choosing l

- `l = 2` prevents the trivial homogeneity attack but little else.
- `l = 3` is common for health datasets.
- `l` cannot exceed the number of distinct sensitive values; a diagnosis column with only "HIV+" / "HIV-" caps `l` at 2 regardless of what you want.

### Failure modes

- **Skewness attack.** A class with `l` distinct values but 95% of records with the same sensitive value is formally l-diverse but informationally not. Entropy l-diversity partially addresses this.
- **Similarity attack.** A class with `l` distinct but semantically similar values ("heart disease", "cardiac arrhythmia", "cardiomyopathy") lets the attacker learn the semantic category. Motivates t-closeness.

## t-Closeness

A QI-equivalence class satisfies **t-closeness** if the distribution of the sensitive attribute within the class is no more than distance `t` from the distribution of the attribute across the full table. Distance is measured by Earth Mover's Distance (EMD) or variants.

Intuition: no equivalence class is sharply different from the whole in its sensitive-attribute distribution. The attacker who identifies a class learns no more than they already knew from the overall distribution.

t-closeness is the strongest of the three; it is also the one with the highest utility cost. Pick `t ∈ [0.1, 0.3]` as a starting range; higher `t` is more permissive.

### Failure modes of t-closeness

- **Utility collapse on sparse categorical sensitive attributes.** If the global distribution is {A: 99%, B: 1%}, every class must closely reflect {99%, 1%}. Classes without any B are impossible without suppression.
- **Not a privacy guarantee against auxiliary data.** t-closeness is still only a bound over the released table; an attacker with auxiliary data still has leverage. Differential privacy (mod-106) is the stronger primitive when the attacker auxiliary is unbounded.

## Comparing the three

| Property | k-anonymity | l-diversity | t-closeness |
|----------|-------------|-------------|-------------|
| Defends against linkage attack | Yes | Yes | Yes |
| Defends against homogeneity | No | Yes | Yes |
| Defends against similarity | No | No | Yes |
| Defends against background-knowledge | No | Partial | Partial |
| Composes under multiple releases | **No** | **No** | **No** |
| Composes under auxiliary data | Weak | Weak | Weak |
| Formal privacy guarantee (DP-grade) | **No** | **No** | **No** |
| Utility cost | Low | Medium | High |
| Community adoption | Very high | High | Moderate |

The "No" row on composability is the most important bit. k-anon-style guarantees are *syntactic* — they constrain the released table. If you release a second (overlapping) table, or an attacker joins your release with an auxiliary dataset, the formal guarantee degrades or breaks. Differential privacy (mod-106) is the mathematically clean alternative. The k-anon family is the industry default because it is well-understood, ARX ships tooling, and many regulators accept it; it is not state-of-the-art privacy.

## Generalisation hierarchies

The engineering work of k-anonymity is designing the **generalisation hierarchy** per QI. A hierarchy is a tree where each node generalises the children.

For `ZIP`:

```text
*              ← fully generalised
└── 9****      (level 4)
    └── 94***  (level 3)
        └── 940**  (level 2)
            └── 94040  (level 1 / original)
```

For `age`:

```text
*
└── [0, 100)
    ├── [0, 50)
    │   ├── [0, 25)
    │   │   ├── [0, 13)
    │   │   └── [13, 25)
    │   └── [25, 50)
    └── [50, 100)
```

For `gender`:

```text
*
└── {M, F, X}
```

For `diagnosis` (sensitive, but sometimes generalised):

```text
*
└── ICD-9-root
    ├── 001-139 Infectious
    ├── 140-239 Neoplasms
    └── ...
```

Three design rules:

- **Interval hierarchies for numeric QIs.** Age, income, dates; always-contiguous intervals.
- **Prefix hierarchies for structured identifiers.** ZIP, postal codes, IPs; truncate by position.
- **Taxonomy hierarchies for categorical.** Diagnosis, occupation, education; use a published taxonomy (ICD-10, ISCO-08).

Encode each hierarchy as a CSV / Excel / ARX hierarchy file per QI. ARX reads these natively.

## The utility side — information-loss metrics

k / l / t trades privacy against utility. The DPIA cites a *measured* utility loss, not a vibe. Common metrics:

- **Precision** (Sweeney / Samarati). `1 - sum(hgt(cell)) / (hgt_max * |cells|)`, where `hgt(cell)` is the generalisation-tree height of each cell. Lower generalisation = higher precision.
- **Discernibility metric (DM).** Sum of class-size-squared, penalising large equivalence classes.
- **Entropy** of the generalised attribute compared to the raw.
- **AECS** (Average Equivalence Class Size). Ratio of row count to number of classes.
- **Task-specific utility.** The gold standard — train a downstream model on the de-identified set and measure the task-metric drop vs. the raw set. If the de-id set loses 2 points of AUC on a diabetes-risk classifier, you have a concrete answer.

Pick one or two general-purpose metrics (precision + AECS is a reasonable minimum) and *add* a task-specific metric when the dataset has a known downstream model.

## The ARX toolkit

ARX ([arx.deidentifier.org](https://arx.deidentifier.org/)) is an open-source Java-based de-identification tool that ships:

- **GUI** for interactive exploration (good for the first pass; good for defending the chosen k in a review meeting).
- **CLI + Java library** for scripted, reproducible de-identification.
- A broad catalogue of privacy models: k-anonymity, l-diversity (several variants), t-closeness, (ε, δ)-DP, δ-presence, β-likeness, etc.
- Risk analysis — prosecutor / journalist / marketer attacker models; records-at-risk probability.

Minimum reproducible pipeline shape:

```java
// ARX (Java)
Data data = Data.create("patients.csv", StandardCharsets.UTF_8, ',');
data.getDefinition().setAttributeType("zip",     AttributeType.QUASI_IDENTIFYING_ATTRIBUTE);
data.getDefinition().setAttributeType("age",     AttributeType.QUASI_IDENTIFYING_ATTRIBUTE);
data.getDefinition().setAttributeType("gender",  AttributeType.QUASI_IDENTIFYING_ATTRIBUTE);
data.getDefinition().setAttributeType("name",    AttributeType.IDENTIFYING_ATTRIBUTE);          // suppressed
data.getDefinition().setAttributeType("diagnosis", AttributeType.SENSITIVE_ATTRIBUTE);

Hierarchy zipHierarchy = Hierarchy.create("zip_hierarchy.csv", StandardCharsets.UTF_8, ',');
data.getDefinition().setHierarchy("zip", zipHierarchy);
// ... similarly for age, gender

ARXConfiguration config = ARXConfiguration.create();
config.addPrivacyModel(new KAnonymity(5));
config.addPrivacyModel(new DistinctLDiversity("diagnosis", 3));
config.addPrivacyModel(new EqualDistanceTCloseness("diagnosis", 0.2));
config.setSuppressionLimit(0.05);          // allow 5% suppression

ARXAnonymizer anonymizer = new ARXAnonymizer();
ARXResult result = anonymizer.anonymize(data, config);
DataHandle output = result.getOutput();
output.save("patients_deidentified.csv", ',');
```

The Python community uses `pyarxaas` ([github.com/navikt/arxaas](https://github.com/navikt/arxaas)) to call ARX over HTTP, which lets the data-engineering pipeline stay in Python while ARX does the Java work.

### ARX's risk analysis

ARX ships a risk-analysis layer that estimates the probability of re-identification under three attacker models:

- **Prosecutor.** The attacker knows their specific target is in the dataset. The risk is the probability of re-identifying that specific person.
- **Journalist.** The attacker is interested in any one person; they succeed if they re-identify anyone.
- **Marketer.** The attacker is interested in aggregate re-identification; risk is the expected fraction re-identified.

The output is "records at risk" at each threshold (1%, 5%, 20%). The DPIA cites the chosen attacker model + the chosen risk threshold + the measured records-at-risk.

## `sdcMicro` as a compare-point

`sdcMicro` ([cran.r-project.org/package=sdcMicro](https://cran.r-project.org/package=sdcMicro)) is the R-based statistical-disclosure-control package maintained by the Austrian statistical office. It ships a wider set of microdata-specific operations:

- **PRAM** (Post-RAndomisation Method) — swap values with controlled probability.
- **Adding noise** to numerics.
- **Rank-swapping** — perturb within rank-order neighbourhoods.
- **Microaggregation** — cluster records and replace with cluster centroid.
- **k-anon / l-div / t-closeness** — similar surface to ARX.

Use `sdcMicro` as a sanity-check: run the same dataset through ARX and sdcMicro; compare the generated anonymised datasets. Divergence is a signal to re-examine your QI classification / hierarchies / privacy-model parameters.

## Pipeline integration

In the AI data path, tabular de-identification typically shows up at two places:

- **Training-set construction.** The training corpus contains tabular rows (customer records, transaction history). Apply k-anon-style de-identification before writing to the training snapshot *and* before writing to the downstream fine-tune. The datasheet records the chosen k / l / t.
- **Analytics-side releases.** Downstream BI / analytics / external research partners receive de-identified subsets. Apply ARX; attach the risk analysis to the release.

For AI pipelines, tabular de-identification is often *combined* with record-level DP (mod-106) — ARX k-anon as a preprocessing step that reduces the privacy budget needed for DP-aggregates. Document the composition in the DPIA.

## The defence — what the DPIA cites

For every tabular release:

- **QI classification register.** Every column's class (direct ID / QI / sensitive / non-sensitive) with rationale.
- **Attacker model.** Prosecutor / journalist / marketer; auxiliary datasets assumed available; attack cost assumed.
- **Chosen k, l, t.** With rationale against the attacker model. "k=5 because the dataset is >100k rows and the attacker-auxiliary scenarios we assume require >5 candidates to be above our deniability bar."
- **Generalisation hierarchies.** The CSV / Excel hierarchies committed to the repo.
- **Information-loss metrics.** Precision / AECS / task-specific metric.
- **Records-at-risk.** Per the chosen attacker model, the prosecutor / journalist records-at-risk numbers from ARX.
- **Residual-risk statement.** The named scenarios (e.g. "an insider with access to the original dataset can trivially re-identify; k-anon does not defend against insider attackers"), each with compensating control (insider access-control + audit).
- **Composability caveat.** The statement that k-anon does not formally compose under multiple releases; the compensating mechanism (release-tracking register; re-identification canary; review cycle).

## Common mistakes

- **QIs discovered only after release.** A column assumed non-sensitive turns out to be a QI when combined with a later dataset. Mitigation: a QI-discovery checklist at every release; a "did we add any attribute we had not re-classified?" check in CI.
- **Hierarchies that do not reflect semantics.** A `zip` hierarchy that groups geographically distant ZIPs because the digits happen to prefix-match; a `diagnosis` hierarchy that groups by alphabetic sort rather than ICD class. Use published taxonomies; check with a domain expert.
- **Running without suppression.** Zero-suppression means every rare class must be generalised to the point of all records matching; the dataset collapses. Allow a small suppression budget (1-5%); record which rows were suppressed.
- **Treating k-anon as DP.** k-anonymity is not differential privacy. The guarantee is over the released table; the moment you release a second table or the attacker brings in an auxiliary, the guarantee weakens. If you need multi-release privacy, reach for mod-106.
- **Not documenting the hierarchy versions.** The hierarchy CSVs are version-controlled artifacts. A change to the hierarchy changes the de-identified output; log the version used.
- **Treating the sensitive attribute column as non-sensitive.** The reason l / t exist is that the sensitive column needs its own protection. Misclassifying it as non-sensitive is the classic beginner mistake.

## Summary — what you can do after this chapter

- Classify every column of a tabular dataset as direct ID / QI / sensitive / non-sensitive.
- Pick a k / l / t trio with a defence against a named attacker model.
- Design generalisation hierarchies per QI using published taxonomies where possible.
- Run ARX (and sdcMicro as a compare-point) and produce the anonymised dataset + risk-analysis report.
- Measure information loss against a general metric and against a task-specific metric where one exists.
- Document the release with QI register, attacker model, chosen parameters, records-at-risk, and residual-risk statement.
- Articulate the composability caveat (k-anon does not compose over releases; DP does).

The next chapter switches focus from the privacy mathematics to the regulatory frame: when HHS OCR Safe Harbor's 18-identifier list is enough and when Expert Determination (statistical disclosure control with a named expert) is required.
