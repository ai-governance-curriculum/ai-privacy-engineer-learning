# Resources for mod-103-data-governance-and-minimisation (Data Governance and Minimisation for AI)

Primary sources first. Everything else is second-order reading.

## Datasheets for Datasets

- **Gebru, T., Morgenstern, J., Vecchione, B., Vaughan, J. W., Wallach, H., Daumé III, H., Crawford, K. — *Datasheets for Datasets*.** CACM 64(12), December 2021 (arXiv 2018, revised 2021). [arxiv.org/abs/1803.09010](https://arxiv.org/abs/1803.09010)
- **Hugging Face — Dataset Cards + `datasets.features` schema.** [huggingface.co/docs/hub/datasets-cards](https://huggingface.co/docs/hub/datasets-cards)
- **Google — Data Cards Playbook / `data cards` project.** [sites.research.google/datacardsplaybook](https://sites.research.google/datacardsplaybook/)
- **Pushkarna, M., Zaldivar, A., Kjartansson, O. — *Data Cards: Purposeful and Transparent Dataset Documentation for Responsible AI* (FAccT 2022).** [arxiv.org/abs/2204.01075](https://arxiv.org/abs/2204.01075)
- **Mitchell, M. *et al.* — *Model Cards for Model Reporting* (FAT* 2019)** (the companion that stacks on top of datasheets). [arxiv.org/abs/1810.03677](https://arxiv.org/abs/1810.03677)
- **NIST AI Risk Management Framework (AI RMF 1.0) — MAP function + Generative AI Profile (NIST AI 600-1).** [nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework)

## Consent, lawful basis, purpose limitation

- **GDPR Articles 5, 6, 7, 9, 13, 14, 17, 21, 22, 25, 28, 30, 32, 35** in the consolidated text. [eur-lex.europa.eu/eli/reg/2016/679/oj](https://eur-lex.europa.eu/eli/reg/2016/679/oj)
- **EDPB Guidelines 05/2020 on consent under Regulation 2016/679.** [edpb.europa.eu](https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-052020-consent-under-regulation-2016679_en)
- **EDPB Guidelines 4/2019 on Article 25 Data Protection by Design and by Default.** [edpb.europa.eu](https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-42019-article-25-data-protection-design-and_en)
- **EDPB Opinion 28/2024 on certain data protection aspects related to the processing of personal data in the context of AI models.** [edpb.europa.eu](https://www.edpb.europa.eu/our-work-tools/our-documents/opinion-board-art-64/opinion-282024-certain-data-protection-aspects_en)
- **Article 29 Working Party — Opinion 05/2014 on Anonymisation Techniques (WP 216).** The singling-out / linkability / inference framework that the audit in chapter 06 inherits. [ec.europa.eu/justice/article-29/documentation/opinion-recommendation/files/2014/wp216_en.pdf](https://ec.europa.eu/justice/article-29/documentation/opinion-recommendation/files/2014/wp216_en.pdf)
- **ICO (UK) — Lawful basis + Consent + Legitimate Interests Assessment guidance.** [ico.org.uk](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/lawful-basis/)
- **CNIL — AI how-to sheets ("fiches pratiques IA").** [cnil.fr/en/ai-how-sheets](https://www.cnil.fr/en/ai-how-sheets)

## EU AI Act data-governance obligations

- **EU AI Act (Regulation (EU) 2024/1689)** — the official consolidated text. [eur-lex.europa.eu/eli/reg/2024/1689/oj](https://eur-lex.europa.eu/eli/reg/2024/1689/oj) (Articles 10 and 11 are the primary anchors for this module.)

## US and sector privacy laws (fast reference)

- **California Consumer Privacy Act / California Privacy Rights Act (CCPA / CPRA).** [oag.ca.gov/privacy/ccpa](https://oag.ca.gov/privacy/ccpa)
- **HIPAA Privacy Rule — minimum necessary standard (45 CFR §164.502(b)).** [hhs.gov/hipaa](https://www.hhs.gov/hipaa/)
- **HIPAA — De-identification Guidance (Safe Harbor + Expert Determination).** [hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification](https://www.hhs.gov/hipaa/for-professionals/privacy/special-topics/de-identification/index.html)
- **NIST Privacy Framework.** [nist.gov/privacyframework](https://www.nist.gov/privacyframework)
- **NIST IR 8062 — An Introduction to Privacy Engineering and Risk Management in Federal Systems.** [doi.org/10.6028/NIST.IR.8062](https://doi.org/10.6028/NIST.IR.8062)
- **NIST SP 800-188 — Trustworthy Attribute-based Data De-identification.** [doi.org/10.6028/NIST.SP.800-188](https://doi.org/10.6028/NIST.SP.800-188)

## Data lineage and column-level provenance

- **OpenLineage** — open standard for lineage emission. [openlineage.io](https://openlineage.io/)
- **Marquez** — OpenLineage reference backend. [marquezproject.github.io/marquez](https://marquezproject.github.io/marquez/)
- **DataHub** — data-catalog with lineage + column-level governance. [datahubproject.io](https://datahubproject.io/)
- **OpenMetadata.** [open-metadata.org](https://open-metadata.org/)
- **dbt — column-level lineage + semantic model.** [docs.getdbt.com](https://docs.getdbt.com/)
- **Databricks Unity Catalog — tag-based governance, column lineage.** [docs.databricks.com/data-governance/unity-catalog](https://docs.databricks.com/data-governance/unity-catalog/index.html)
- **Snowflake — Object Tagging + Tag-Based Masking Policies.** [docs.snowflake.com](https://docs.snowflake.com/en/user-guide/object-tagging)
- **Google BigQuery — Column-level security + Policy tags.** [cloud.google.com/bigquery/docs/column-level-security-intro](https://cloud.google.com/bigquery/docs/column-level-security-intro)

## Data minimisation and retention

- **GDPR Art. 5(1)(c) + Art. 5(1)(e)** (see consolidated text above).
- **ICO (UK) — Data minimisation + storage limitation guidance.** [ico.org.uk](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/data-protection-principles/a-guide-to-the-data-protection-principles/)
- **NIST SP 800-53 Rev. 5 — Privacy control family (PT-*).** [doi.org/10.6028/NIST.SP.800-53r5](https://doi.org/10.6028/NIST.SP.800-53r5)

## Data privacy vaults and tokenisation

- **PCI Security Standards Council — Information Supplement: PCI DSS Tokenization Guidelines.** [pcisecuritystandards.org/document_library](https://www.pcisecuritystandards.org/document_library/)
- **PCI SSC — Tokenization Product Security Guidelines.** [pcisecuritystandards.org/document_library](https://www.pcisecuritystandards.org/document_library/)
- **ANSI X9.119-2** — reversible-cryptographic-token standard for financial data. [x9.org](https://x9.org/)
- **NIST SP 800-38G — Methods for Format-Preserving Encryption (FF1 / FF3-1).** [doi.org/10.6028/NIST.SP.800-38G](https://doi.org/10.6028/NIST.SP.800-38G)
- **NIST SP 800-57 Part 1 Rev. 5 — Recommendation for Key Management.** [doi.org/10.6028/NIST.SP.800-57pt1r5](https://doi.org/10.6028/NIST.SP.800-57pt1r5)
- **Skyflow — data privacy vault architecture (vendor material, useful as the reference shape).** [skyflow.com](https://www.skyflow.com/)
- **HashiCorp Vault — Transit secrets engine (FPE / encrypt / decrypt endpoints).** [developer.hashicorp.com/vault/docs/secrets/transit](https://developer.hashicorp.com/vault/docs/secrets/transit)
- **Very Good Security (VGS).** [verygoodsecurity.com](https://www.verygoodsecurity.com/)
- **Google Cloud Sensitive Data Protection (DLP) — de-identification + tokenisation primitives.** [cloud.google.com/sensitive-data-protection](https://cloud.google.com/sensitive-data-protection/)

## Synthetic data

- **Jordon, J., Szpruch, L., Houssiau, F. *et al.* — *Synthetic Data — what, why and how?*** Royal Society / Alan Turing Institute, 2022. [royalsociety.org](https://royalsociety.org/topics-policy/projects/privacy-enhancing-technologies/)
- **UK ICO — Privacy-Enhancing Technologies (PETs) guidance** (includes synthetic-data posture). [ico.org.uk](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/data-sharing/privacy-enhancing-technologies/)
- **Xu, L., Skoularidou, M., Cuesta-Infante, A., Veeramachaneni, K. — *Modeling Tabular Data using Conditional GAN*** (CTGAN). NeurIPS 2019. [arxiv.org/abs/1907.00503](https://arxiv.org/abs/1907.00503)
- **Kotelnikov, A., Baranchuk, D., Rubachev, I., Babenko, A. — *TabDDPM: Modelling Tabular Data with Diffusion Models*** (ICML 2023). [arxiv.org/abs/2209.15421](https://arxiv.org/abs/2209.15421)
- **Zhang, J., Cormode, G., Procopiuc, C. M., Srivastava, D., Xiao, X. — *PrivBayes: Private Data Release via Bayesian Networks*** (ACM TODS 2017). [dl.acm.org/doi/10.1145/3134428](https://dl.acm.org/doi/10.1145/3134428)
- **Synthetic Data Vault (SDV) — open-source orchestration layer.** [sdv.dev](https://sdv.dev/)
- **Synthea — synthetic patient-record generator (MITRE).** [synthetichealth.github.io/synthea](https://synthetichealth.github.io/synthea/)
- **Gretel.** [gretel.ai](https://gretel.ai/)
- **MOSTLY AI.** [mostly.ai](https://mostly.ai/)
- **Tonic.** [tonic.ai](https://www.tonic.ai/)
- **OpenDP SmartNoise Synth (DP synthetic data).** [smartnoise.org](https://smartnoise.org/)

## CycloneDX and ML-BOM / SBOM

- **CycloneDX — specification and tooling.** [cyclonedx.org](https://cyclonedx.org/)
- **CycloneDX ML-BOM capability page.** [cyclonedx.org/capabilities/mlbom](https://cyclonedx.org/capabilities/mlbom/)
- **OWASP CycloneDX Authoritative Guide.** [cyclonedx.org/guides](https://cyclonedx.org/guides/)
- **CycloneDX AI/ML examples repository.** [github.com/CycloneDX/bom-examples](https://github.com/CycloneDX/bom-examples)
- **NTIA — SBOM Minimum Elements.** [ntia.gov/files/ntia/publications/sbom_minimum_elements_report.pdf](https://www.ntia.gov/files/ntia/publications/sbom_minimum_elements_report.pdf)
- **Executive Order 14028 — Improving the Nation's Cybersecurity.** [federalregister.gov/d/2021-10460](https://www.federalregister.gov/documents/2021/05/17/2021-10460/improving-the-nations-cybersecurity)
- **SPDX AI Profile (SPDX 3.0).** [spdx.dev](https://spdx.dev/)
- **Dependency-Track — CycloneDX ingestion / query at scale.** [dependencytrack.org](https://dependencytrack.org/)
- **cyclonedx-python-lib.** [github.com/CycloneDX/cyclonedx-python-lib](https://github.com/CycloneDX/cyclonedx-python-lib)

## Standards relevant to the engineering interface

- **ISO/IEC 27701:2019** — Privacy Information Management System (PIMS) extension to ISO/IEC 27001. [iso.org/standard/71670.html](https://www.iso.org/standard/71670.html)
- **ISO/IEC 29100:2024** — Information technology — Security techniques — Privacy framework.
- **ISO/IEC 29134:2017** — Guidelines for privacy impact assessment. [iso.org/standard/62289.html](https://www.iso.org/standard/62289.html)
- **ISO/IEC 42001:2023** — AI management systems. [iso.org/standard/81230.html](https://www.iso.org/standard/81230.html)

## Cross-references within this track

- **mod-101** — privacy frameworks and the regulator reading. The lawful-basis + Art. 5 anchor comes from there.
- **mod-102** — privacy threat modelling. The DFD from exercise 01 feeds every exercise in this module.
- **mod-104** — PII / PHI / PCI detection + redaction. The detector feeds chapter 05 tokenisation.
- **mod-105** — memorisation / extraction / MIA. The audit in chapter 06 reuses MIA methodology.
- **mod-106** — differential privacy engineering. DP synthesis (chapter 06) and DP-training on tagged corpora both land here.
- **mod-108** — confidential computing + privacy-preserving inference. A trust-boundary-respecting vault integration crosses into mod-108 when the detokenise happens inside an attested enclave.
- **mod-109** — RAG privacy + agent scoping. Tenant-ACL + agent-tool data scoping are the depth on retrieval-side propagation.
- **mod-110** — machine unlearning + DSR. The ML-BOM makes unlearning addressable; the datasheet records the unlearning plan.
- **mod-111** — DPIAs + regulator-grade evidence. The consumer of this module's outputs.
- **mod-112** — AI privacy program + communication. The datasheet publication and the model/dataset card surfaces live there.
