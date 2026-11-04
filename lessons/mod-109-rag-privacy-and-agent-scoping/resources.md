# Resources for mod-109-rag-privacy-and-agent-scoping

Curated reading and tooling for RAG retrieval access control, cross-tenant leakage testing, prompt / completion logging, agent-tool scoping, corpus poisoning defence, and system-card authoring. Primary references first; practitioner / tooling material follows.

## Standards and guidance

- OWASP — *Top 10 for Large Language Model Applications*, v2.0 (2025). [genai.owasp.org/llm-top-10/](https://genai.owasp.org/llm-top-10/). Especially LLM02 (Sensitive Information Disclosure), LLM04 (Data and Model Poisoning), LLM06 (Excessive Agency), and LLM08 (Vector and Embedding Weaknesses).
- OWASP — *Non-Human Identities* material. [owasp.org/www-project-non-human-identity-management/](https://owasp.org/www-project-non-human-identity-management/). <!-- needs-research: confirm the exact title and publication year of the OWASP NHI Top 10 for AI/ML before citing specific entries. -->
- NIST — *AI Risk Management Framework, AI RMF 1.0* (2023). [nist.gov/itl/ai-risk-management-framework](https://www.nist.gov/itl/ai-risk-management-framework). The Measure / Manage / Govern functions anchor the regulator-facing posture a system card speaks to.
- NIST — *SP 800-53 Rev. 5*, especially the AU (Audit) and AC (Access Control) control families. [doi.org/10.6028/NIST.SP.800-53r5](https://doi.org/10.6028/NIST.SP.800-53r5).
- NIST — *SP 800-162, Guide to Attribute Based Access Control* (2014). [doi.org/10.6028/NIST.SP.800-162](https://doi.org/10.6028/NIST.SP.800-162). ABAC vocabulary used in the retrieval-predicate design.
- NIST — *SP 800-63-3 Digital Identity Guidelines* and the 63A / 63B / 63C companions. [doi.org/10.6028/NIST.SP.800-63-3](https://doi.org/10.6028/NIST.SP.800-63-3). Authentication and federation vocabulary the system card quotes.
- MITRE ATLAS — *Adversarial Threat Landscape for AI Systems*. [atlas.mitre.org/](https://atlas.mitre.org/). Especially AML.T0010 (ML Supply Chain Compromise), AML.T0020 (Poison Training Data), AML.T0051 (Prompt Injection).
- EU — *AI Act*. [artificialintelligenceact.eu/the-act/](https://artificialintelligenceact.eu/the-act/). Articles 10 (data governance), 12 (record-keeping), 13 (transparency), 15 (accuracy / robustness / cybersecurity).
- UK ICO — *Guidance on AI and data protection*. [ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/artificial-intelligence/).
- GDPR Articles 5, 17, 25, 32. [gdpr-info.eu/](https://gdpr-info.eu/).

## Protocols and standards for identity propagation

- IETF RFC 6749 — *The OAuth 2.0 Authorization Framework*. [datatracker.ietf.org/doc/html/rfc6749](https://datatracker.ietf.org/doc/html/rfc6749).
- IETF RFC 8693 — *OAuth 2.0 Token Exchange*. [datatracker.ietf.org/doc/html/rfc8693](https://datatracker.ietf.org/doc/html/rfc8693). The standards-track mechanism for on-behalf-of tool invocation.
- IETF RFC 7519 — *JSON Web Token (JWT)*. [datatracker.ietf.org/doc/html/rfc7519](https://datatracker.ietf.org/doc/html/rfc7519).
- OpenID Connect — *Core 1.0*. [openid.net/specs/openid-connect-core-1_0.html](https://openid.net/specs/openid-connect-core-1_0.html).
- OpenID — *RISC Profile* (Security Event Tokens for session / credential revocation). [openid.net/specs/openid-risc-profile-specification-1_0.html](https://openid.net/specs/openid-risc-profile-specification-1_0.html).
- Anthropic — *Model Context Protocol (MCP)*. [modelcontextprotocol.io/](https://modelcontextprotocol.io/) and the specification at [spec.modelcontextprotocol.io/](https://spec.modelcontextprotocol.io/).
- SPIFFE — *Secure Production Identity Framework for Everyone*. [spiffe.io/docs/latest/spiffe-about/overview/](https://spiffe.io/docs/latest/spiffe-about/overview/). Workload-identity model that agent NHI can bind to.
- Google — *Workload Identity Federation*. [cloud.google.com/iam/docs/workload-identity-federation](https://cloud.google.com/iam/docs/workload-identity-federation).

## Vector stores and retrieval access control

- Pinecone — *Namespaces*. [docs.pinecone.io/guides/indexes/data/understanding-namespaces](https://docs.pinecone.io/guides/indexes/data/understanding-namespaces). *Metadata filtering*. [docs.pinecone.io/guides/index-data/indexing-overview#metadata](https://docs.pinecone.io/guides/index-data/indexing-overview#metadata).
- Weaviate — *Multi-tenancy*. [weaviate.io/developers/weaviate/concepts/data#multi-tenancy](https://weaviate.io/developers/weaviate/concepts/data#multi-tenancy).
- Qdrant — *Payload filtering*. [qdrant.tech/documentation/concepts/filtering/](https://qdrant.tech/documentation/concepts/filtering/). *Multi-tenancy guide*. [qdrant.tech/documentation/guides/multiple-partitions/](https://qdrant.tech/documentation/guides/multiple-partitions/).
- pgvector — repository README with filtering notes. [github.com/pgvector/pgvector](https://github.com/pgvector/pgvector).
- PostgreSQL — *Row Security Policies*. [postgresql.org/docs/current/ddl-rowsecurity.html](https://www.postgresql.org/docs/current/ddl-rowsecurity.html).
- OpenSearch — *k-NN search* and *security plugin's document-level security*. [opensearch.org/docs/latest/search-plugins/knn/](https://opensearch.org/docs/latest/search-plugins/knn/), [opensearch.org/docs/latest/security/access-control/document-level-security/](https://opensearch.org/docs/latest/security/access-control/document-level-security/).

## PII detection / redaction

- Microsoft — *Presidio* documentation. [microsoft.github.io/presidio/](https://microsoft.github.io/presidio/). Analyzer, Anonymizer, custom recognisers.
- Microsoft — *Presidio recognisers reference*. [microsoft.github.io/presidio/supported_entities/](https://microsoft.github.io/presidio/supported_entities/).
- spaCy — *Documentation*. [spacy.io/usage](https://spacy.io/usage). The default NLP backend for Presidio's NER-based recognisers.
- Private AI, Scrubadub, Aho-Corasick-based scanners — alternative or complementary tooling; evaluate against your data distribution.

## Data-privacy vaults and tokenisation

- Skyflow — *What is a Data Privacy Vault*. [skyflow.com/post/what-is-a-data-privacy-vault](https://www.skyflow.com/post/what-is-a-data-privacy-vault). *The Case for a Data Privacy Vault*. [skyflow.com/post/the-case-for-a-data-privacy-vault](https://www.skyflow.com/post/the-case-for-a-data-privacy-vault).
- NIST — *SP 800-38G, Recommendation for Block Cipher Modes of Operation: Methods for Format-Preserving Encryption*. [doi.org/10.6028/NIST.SP.800-38G](https://doi.org/10.6028/NIST.SP.800-38G). Format-preserving tokenisation reference.

## Model-provider policies (quote in the system card)

- Anthropic — *Privacy policy* and API data-usage terms. [anthropic.com/legal/privacy](https://www.anthropic.com/legal/privacy).
- OpenAI — *Business terms* and *Enterprise privacy*. [openai.com/policies/business-terms/](https://openai.com/policies/business-terms/), [openai.com/enterprise-privacy/](https://openai.com/enterprise-privacy/).
- Google — *Vertex AI data governance*. [cloud.google.com/vertex-ai/docs/generative-ai/data-governance](https://cloud.google.com/vertex-ai/docs/generative-ai/data-governance).
- Microsoft — *Azure OpenAI Service data privacy*. [learn.microsoft.com/en-us/azure/ai-services/openai/concepts/data-privacy](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/data-privacy).

## Corpus provenance and supply chain

- SLSA — *Supply-chain Levels for Software Artifacts*, v1.0. [slsa.dev/spec/v1.0/](https://slsa.dev/spec/v1.0/).
- in-toto — *Specification*. [in-toto.io/](https://in-toto.io/). Attestation primitives for a RAG ingest pipeline.
- NIST — *SP 800-218, Secure Software Development Framework (SSDF)*. [doi.org/10.6028/NIST.SP.800-218](https://doi.org/10.6028/NIST.SP.800-218).
- Carlini et al. — *Poisoning Web-Scale Training Datasets is Practical*, USENIX Security 2024. [arxiv.org/abs/2302.10149](https://arxiv.org/abs/2302.10149).

## System-card and model-card patterns

- Mitchell et al. — *Model Cards for Model Reporting*, FAccT 2019. [arxiv.org/abs/1810.03993](https://arxiv.org/abs/1810.03993).
- Gebru et al. — *Datasheets for Datasets*, CACM 2021. [arxiv.org/abs/1803.09010](https://arxiv.org/abs/1803.09010).
- OpenAI — *GPT-4 System Card*. [openai.com/index/gpt-4-system-card/](https://openai.com/index/gpt-4-system-card/).
- Anthropic — *Claude 3 Family model card / addenda*. [anthropic.com/news/claude-3-family](https://www.anthropic.com/news/claude-3-family).
- ISO/IEC 42001 — *Information technology — Artificial intelligence — Management system*. [iso.org/standard/81230.html](https://www.iso.org/standard/81230.html).

## Testing tools

- pytest. [docs.pytest.org/en/stable/](https://docs.pytest.org/en/stable/).
- Hypothesis — property-based testing for Python. [hypothesis.readthedocs.io/](https://hypothesis.readthedocs.io/).
- Testcontainers — Postgres / Elasticsearch / Redis in CI. [testcontainers.com/](https://testcontainers.com/).
- Mutmut — mutation testing. [github.com/boxed/mutmut](https://github.com/boxed/mutmut).

## Primary attack-surface reading

- Simon Willison — *Prompt injection* notes series. [simonwillison.net/tags/prompt-injection/](https://simonwillison.net/tags/prompt-injection/). Practitioner-facing accumulation of the attack surface, useful for red-team case design.
- Greshake et al. — *Not what you've signed up for: Compromising Real-World LLM-Integrated Applications with Indirect Prompt Injection* (2023). [arxiv.org/abs/2302.12173](https://arxiv.org/abs/2302.12173).
- Carlini et al. — *Extracting Training Data from Large Language Models* (USENIX Security 2021). [arxiv.org/abs/2012.07805](https://arxiv.org/abs/2012.07805). Boundary read for the model-output-leakage discussion in the residual-risks section.

## Related modules in this track

- `mod-104-pii-detection-and-redaction` — deeper Presidio coverage, recogniser authoring, pipeline-position decisions.
- `mod-105-memorisation-extraction-and-mia` — the model-output-leakage half of the residual-risk statement.
- `mod-106-differential-privacy-engineering` — DP-SGD and the alternative when the index-side controls are not enough.
- `mod-108-confidential-computing-and-ppi` — TEE / HE / MPC when the deployment surface itself has to be removed from the trust boundary.
- `mod-110-machine-unlearning-and-dsr` — the operational companion to the erasure / tombstoning claims here.
- `mod-111-dpia-and-privacy-reviews` — the DPIA document the system card feeds into.
