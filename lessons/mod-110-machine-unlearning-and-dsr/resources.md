# Resources for mod-110-machine-unlearning-and-dsr

Curated reading and tooling for the DSR engineering surface, exact and approximate unlearning, LLM-specific unlearning, the erasure decision tree, RAG / agent DSR, and audit-trail packaging. Primary legal and research references first; practitioner / tooling material follows.

## Primary legal texts and regulatory guidance

- Regulation (EU) 2016/679, *General Data Protection Regulation*, Articles 12–22, 30, 33, 77. [eur-lex.europa.eu/eli/reg/2016/679/oj](https://eur-lex.europa.eu/eli/reg/2016/679/oj).
- UK *Data Protection Act 2018* and the UK GDPR. [legislation.gov.uk/ukpga/2018/12/contents](https://www.legislation.gov.uk/ukpga/2018/12/contents).
- California Consumer Privacy Act / California Privacy Rights Act. [cppa.ca.gov/regulations/](https://cppa.ca.gov/regulations/).
- Brazil — *Lei Geral de Proteção de Dados (LGPD)*. [planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm](https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm).
- European Data Protection Board — *Guidelines 01/2022 on data subject rights – Right of access*, v2.0 (2023). [edpb.europa.eu/system/files/2023-04/edpb_guidelines_202201_data_subject_rights_access_v2_en.pdf](https://www.edpb.europa.eu/system/files/2023-04/edpb_guidelines_202201_data_subject_rights_access_v2_en.pdf).
- European Data Protection Board — *Guidelines on automated individual decision-making and profiling for the purposes of Regulation 2016/679* (WP251 rev.01, endorsed 2018). [edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-automated-individual-decision-making-and_en](https://edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-automated-individual-decision-making-and_en).
- UK ICO — *Guide to the UK GDPR* (individual rights, accountability, documentation). [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/).
- UK ICO — *Guidance on AI and data protection*. [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/).
- UK ICO — *Right to erasure* guidance. [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/the-right-to-erasure/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/the-right-to-erasure/).
- CNIL — *AI practice guides*. [cnil.fr/en/artificial-intelligence](https://www.cnil.fr/en/artificial-intelligence).
- EU — *AI Act*. [artificialintelligenceact.eu/the-act/](https://artificialintelligenceact.eu/the-act/).

## Standards and frameworks

- NIST — *Privacy Framework v1.0* (2020). [nist.gov/privacy-framework](https://www.nist.gov/privacy-framework).
- NIST — *SP 800-53 Rev. 5*, especially AC (Access Control), AU (Audit and Accountability), IP (Individual Participation), PT (PII Processing and Transparency), DM (Data Minimization). [doi.org/10.6028/NIST.SP.800-53r5](https://doi.org/10.6028/NIST.SP.800-53r5).
- NIST — *SP 800-88 Rev. 1, Guidelines for Media Sanitization*. [doi.org/10.6028/NIST.SP.800-88r1](https://doi.org/10.6028/NIST.SP.800-88r1).
- NIST — *AI Risk Management Framework, AI RMF 1.0* (2023). [nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework).
- ISO/IEC 27701 — *Privacy information management extension to ISO/IEC 27001 and 27002*. [iso.org/standard/71670.html](https://www.iso.org/standard/71670.html).
- ISO/IEC 29184 — *Online privacy notices and consent*. [iso.org/standard/70331.html](https://www.iso.org/standard/70331.html).
- ISO/IEC 42001 — *Information technology — Artificial intelligence — Management system*. [iso.org/standard/81230.html](https://www.iso.org/standard/81230.html).

## Exact unlearning — SISA and its antecedents

- Bourtoule, Chandrasekaran, Choquette-Choo, Jia, Travers, Zhang, Lie, Papernot — *Machine Unlearning*, IEEE S&P 2021. [arxiv.org/abs/1912.03817](https://arxiv.org/abs/1912.03817).
- Cao, Yang — *Towards Making Systems Forget with Machine Unlearning*, IEEE S&P 2015. [cs.wisc.edu/~jerryzhu/pub/mu.pdf](https://pages.cs.wisc.edu/~jerryzhu/pub/mu.pdf).
- Ginart, Guan, Valiant, Zou — *Making AI Forget You: Data Deletion in Machine Learning*, NeurIPS 2019. [arxiv.org/abs/1907.05012](https://arxiv.org/abs/1907.05012).
- Thudi, Jia, Shumailov, Papernot — *On the Necessity of Auditable Algorithmic Definitions for Machine Unlearning*, USENIX Security 2022. [arxiv.org/abs/2110.11891](https://arxiv.org/abs/2110.11891).
- Xu, Zhu, Zhang, Wang, Zhang — *Machine Unlearning: A Survey*, ACM Computing Surveys 2023. [arxiv.org/abs/2306.03558](https://arxiv.org/abs/2306.03558).

## Approximate unlearning — influence functions and certified removal

- Koh, Liang — *Understanding Black-box Predictions via Influence Functions*, ICML 2017. [arxiv.org/abs/1703.04730](https://arxiv.org/abs/1703.04730).
- Guo, Goldstein, Hannun, van der Maaten — *Certified Data Removal from Machine Learning Models*, ICML 2020. [arxiv.org/abs/1911.03030](https://arxiv.org/abs/1911.03030).
- Izzo, Smart, Chaudhuri, Zou — *Approximate Data Deletion from Machine Learning Models*, AISTATS 2021. [arxiv.org/abs/2002.10077](https://arxiv.org/abs/2002.10077).
- Sekhari, Acharya, Kamath, Suresh — *Remember What You Want to Forget: Algorithms for Machine Unlearning*, NeurIPS 2021. [arxiv.org/abs/2103.03279](https://arxiv.org/abs/2103.03279).
- Agarwal, Bullins, Hazan — *Second-Order Stochastic Optimization for Machine Learning in Linear Time*, JMLR 2017. [arxiv.org/abs/1602.03943](https://arxiv.org/abs/1602.03943). The LiSSA Hessian-inverse-vector-product estimator Koh & Liang use.
- Neel, Roth, Sharifi-Malvajerdi — *Descent-to-Delete: Gradient-Based Methods for Machine Unlearning*, ALT 2021. [arxiv.org/abs/2007.02923](https://arxiv.org/abs/2007.02923).

## LLM unlearning

- Jang, Yoon, Yang, Cha, Lee, Logeswaran, Seo — *Knowledge Unlearning for Mitigating Privacy Risks in Language Models*, ACL 2023. [arxiv.org/abs/2210.01504](https://arxiv.org/abs/2210.01504).
- Ilharco, Ribeiro, Wortsman, Gururangan, Schmidt, Hajishirzi, Farhadi — *Editing Models with Task Arithmetic*, ICLR 2023. [arxiv.org/abs/2212.04089](https://arxiv.org/abs/2212.04089).
- Eldan, Russinovich — *Who's Harry Potter? Approximate Unlearning in LLMs* (2023). [arxiv.org/abs/2310.02238](https://arxiv.org/abs/2310.02238).
- Maini, Feng, Schwarzschild, Lipton, Kolter — *TOFU: A Task of Fictitious Unlearning for LLMs*, COLM 2024. [arxiv.org/abs/2401.06121](https://arxiv.org/abs/2401.06121). Benchmark repo: [github.com/locuslab/tofu](https://github.com/locuslab/tofu).
- Yao, Xu, Liu — *Large Language Model Unlearning* (2023). [arxiv.org/abs/2310.10683](https://arxiv.org/abs/2310.10683).
- Liu, Zhan, Zhao, Yuan, Chen, Chen, Xue, Chaudhary — *Rethinking Machine Unlearning for Large Language Models* (2024). [arxiv.org/abs/2402.08787](https://arxiv.org/abs/2402.08787).
- Si, Song, Sun, Diao, Chen, Zhang, Wen, Chen, Wu — *Knowledge Unlearning for LLMs: Tasks, Methods, and Challenges* (2023). [arxiv.org/abs/2311.15766](https://arxiv.org/abs/2311.15766).
- Barbulescu, Triantafillou — *To Each (Textual Sequence) Its Own: Improving Memorized-Data Unlearning in a Tokenized Language Model*, ICML 2024. [arxiv.org/abs/2405.03097](https://arxiv.org/abs/2405.03097).
- Carlini, Tramèr, Wallace, Jagielski, Herbert-Voss, Lee, Roberts, Brown, Song, Erlingsson, Oprea, Raffel — *Extracting Training Data from Large Language Models*, USENIX Security 2021. [arxiv.org/abs/2012.07805](https://arxiv.org/abs/2012.07805).
- Nasr, Rando, Carlini, Hayase, Jagielski, Cooper, Ippolito, Choquette-Choo, Wallace, Tramèr, Lee — *Scalable Extraction of Training Data from (Production) Language Models* (2023). [arxiv.org/abs/2311.17035](https://arxiv.org/abs/2311.17035).

## Membership-inference and verification

- Shokri, Stronati, Song, Shmatikov — *Membership Inference Attacks Against Machine Learning Models*, IEEE S&P 2017. [arxiv.org/abs/1610.05820](https://arxiv.org/abs/1610.05820).
- Carlini, Chien, Nasr, Song, Terzis, Tramèr — *Membership Inference Attacks From First Principles*, IEEE S&P 2022. [arxiv.org/abs/2112.03570](https://arxiv.org/abs/2112.03570). The LiRA attack.
- Ye, Maddi, Murakonda, Bindschaedler, Shokri — *Enhanced Membership Inference Attacks Against Machine Learning Models*, CCS 2022. [arxiv.org/abs/2111.09679](https://arxiv.org/abs/2111.09679).
- Pawelczyk, Neel, Lakkaraju — *In-context Unlearning: Language Models as Few-Shot Unlearners* (2023). [arxiv.org/abs/2310.07579](https://arxiv.org/abs/2310.07579).
- Thudi, Shintre, Steinhardt, Boneh, Papernot — *Unrolling SGD: Understanding Factors Influencing Machine Unlearning*, EuroS&P 2022. [arxiv.org/abs/2109.13398](https://arxiv.org/abs/2109.13398).

## Vector-store and corpus-deletion primitives

- Pinecone — *Delete data*. [docs.pinecone.io/guides/manage-data/delete-data](https://docs.pinecone.io/guides/manage-data/delete-data).
- Weaviate — *Delete objects*. [weaviate.io/developers/weaviate/manage-data/delete](https://weaviate.io/developers/weaviate/manage-data/delete).
- Qdrant — *Points — Delete*. [qdrant.tech/documentation/concepts/points/#delete-points](https://qdrant.tech/documentation/concepts/points/#delete-points).
- pgvector — repository README. [github.com/pgvector/pgvector](https://github.com/pgvector/pgvector).
- Elastic — *Delete by query API*. [elastic.co/guide/en/elasticsearch/reference/current/docs-delete-by-query.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-delete-by-query.html).
- OpenSearch — *Delete by query*. [opensearch.org/docs/latest/api-reference/document-apis/delete-by-query/](https://opensearch.org/docs/latest/api-reference/document-apis/delete-by-query/).

## Logging and audit

- OpenTelemetry — *Logs data model*. [opentelemetry.io/docs/specs/otel/logs/data-model/](https://opentelemetry.io/docs/specs/otel/logs/data-model/).
- Cloud provider deletion APIs — e.g. Datadog *Logs API*, [docs.datadoghq.com/api/latest/logs/](https://docs.datadoghq.com/api/latest/logs/).
- Anthropic — *Model Context Protocol (MCP)*. [modelcontextprotocol.io/](https://modelcontextprotocol.io/).

## Attack-surface and threat-modelling anchors

- OWASP — *Top 10 for Large Language Model Applications*, v2.0 (2025). [genai.owasp.org/llm-top-10/](https://genai.owasp.org/llm-top-10/).
- MITRE ATLAS. [atlas.mitre.org/](https://atlas.mitre.org/).

## Library and tooling landscape

- PyTorch — *Reproducibility* guide (determinism controls needed for SISA). [pytorch.org/docs/stable/notes/randomness.html](https://pytorch.org/docs/stable/notes/randomness.html).
- JAX — *Pseudo-random numbers*. [jax.readthedocs.io/en/latest/random-numbers.html](https://jax.readthedocs.io/en/latest/random-numbers.html).
- Hugging Face `transformers` — *Trainer* for the LLM-unlearning loops. [huggingface.co/docs/transformers/main_classes/trainer](https://huggingface.co/docs/transformers/main_classes/trainer).
- `peft` (Hugging Face) — parameter-efficient fine-tuning, useful for low-cost LLM unlearning. [github.com/huggingface/peft](https://github.com/huggingface/peft).
- Microsoft — *Presidio* for free-text subject-content matching in the chunk search. [microsoft.github.io/presidio/](https://microsoft.github.io/presidio/).
- `locuslab/tofu` — the TOFU benchmark reference implementation. [github.com/locuslab/tofu](https://github.com/locuslab/tofu).

## Related modules in this track

- `mod-101-privacy-frameworks-and-position` — DSR obligations in the regulatory map.
- `mod-103-data-governance-and-minimisation` — the data-inventory discipline that chapter 01's runbooks consume.
- `mod-104-pii-detection-and-redaction` — recogniser primitives for the free-text chunk search.
- `mod-105-memorisation-extraction-and-mia` — the attack / verification primitives chapter 03 / 04's unlearning tests depend on.
- `mod-106-differential-privacy-engineering` — the (ε, δ) accounting generalises to certified-removal in chapter 03.
- `mod-109-rag-privacy-and-agent-scoping` — the chunk-record schema and subject-ID propagation chapter 06's cascade depends on.
- `mod-111-dpia-and-privacy-reviews` — the DPIA consumes this module's Erasure Posture and per-DSR audit artefacts.
- `mod-112-privacy-program-and-communication` — the subject-facing language and the regulator-facing narrative.
