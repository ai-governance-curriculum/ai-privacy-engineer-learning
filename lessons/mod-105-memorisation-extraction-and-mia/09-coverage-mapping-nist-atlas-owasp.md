# The Coverage Story — NIST AI 100-2, MITRE ATLAS, OWASP LLM Top 10

The attacks in your harness are only half the story. The other half is **the mapping** — a reviewer reads your evidence by matching your attack suite to the external taxonomy they already use. If your table is laid out so a NIST AI 100-2 category or a MITRE ATLAS technique ID matches one of your attack rows at a glance, the coverage conversation ends in minutes. If the mapping is implicit or absent, the conversation becomes a line-by-line translation exercise and the reviewer concludes you have not done the mapping work.

This chapter is the engineer's reference to the three external taxonomies the privacy-side reviewer will cross-check your coverage against, and the specific mapping for the attacks mod-105 builds.

Primary references:

- **NIST AI 100-2 — Adversarial Machine Learning: A Taxonomy and Terminology of Attacks and Mitigations.** NIST, 2024. [doi.org/10.6028/NIST.AI.100-2e2023](https://doi.org/10.6028/NIST.AI.100-2e2023) (the first edition; verify the current latest edition before publication). Section on **privacy attacks**.
- **MITRE ATLAS — Adversarial Threat Landscape for Artificial-Intelligence Systems.** [atlas.mitre.org](https://atlas.mitre.org/) — the live knowledge base with tactic / technique IDs.
- **OWASP Top 10 for LLM Applications.** [owasp.org/www-project-top-10-for-large-language-model-applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/) — the privacy-relevant entry is **LLM02 — Sensitive Information Disclosure**. Verify the current list version before citing; the taxonomy has evolved between releases.

## Why three taxonomies

Each taxonomy has a different audience and answers a different reviewer's question:

- **NIST AI 100-2** is the US federal / NIST-aligned regulatory reference. A US federal agency, a NIST-aligned risk register, or a procurement review citing the AI RMF will cross-reference AI 100-2 attack categories. The taxonomy is terminology-heavy and framework-oriented.
- **MITRE ATLAS** is the security-engineer's and SOC team's reference. Mapped to TTP-shape IDs for alignment with the broader MITRE ATT&CK muscle memory. A security review or a SIEM-engineer will cross-reference ATLAS technique IDs. The taxonomy is incident-oriented and TTP-oriented.
- **OWASP LLM Top 10** is the application-security and product-team reference. A developer or appsec engineer will cross-reference the OWASP entry. Terser than the other two; focussed on the LLM application surface.

A defensible coverage story cites all three. Omitting any one is not a disqualifier but it is a visible gap to the reviewer whose world is organised around that taxonomy.

## NIST AI 100-2 — the privacy-attack categories

AI 100-2 organises adversarial-ML threats along three axes: *attacker goals*, *attacker knowledge*, and *lifecycle phase*. The privacy-attack goals map to the attack classes in this module. The relevant categories (names approximate the publication's terminology; verify current edition):

| AI 100-2 category | mod-105 attack | Chapter |
|---|---|---|
| Membership inference | Shokri shadow-model MIA | 02 |
| Membership inference | Carlini LiRA | 03 |
| Attribute inference | Attribute-inference attack (Yeom / Jayaraman variant) | 04 |
| Data reconstruction | Fredrikson-style model inversion | 04 |
| Data extraction | Carlini extraction on GPT-2 / Pythia / Llama | 05 |
| Data extraction | Nasr-2023 divergence attack on production APIs | 05 |
| Property inference | (Named for completeness, out of scope — see below) | — |
| Model extraction | (Out of scope — `ai-security-engineer`) | — |

Two things to call out when writing the mapping:

- **Property / distribution inference** is in AI 100-2's privacy-attack list but out of scope for mod-105's measurement surface. Name it; attribute it to a different role (`ai-security-engineer` scope or the peer `ai-risk-engineer`); do not pretend to measure it.
- **Model extraction** (weight / behaviour stealing) is in AI 100-2 as a *confidentiality* attack and is out of scope for this module. The `ai-security-engineer` or `ai-asset-protection-engineer` role owns it. Name the boundary.

The NIST-aligned coverage row shape (one row per attack in your harness):

```
| AI 100-2 category   | Attack                        | Our harness ref         | Last measured | Metric reported |
|---------------------|-------------------------------|-------------------------|---------------|-----------------|
| Membership inference | Shokri shadow-model MIA      | mlp-attack/shokri-v2    | 2025-10-30    | AUC + TPR@FPR   |
| Membership inference | Carlini LiRA                 | mlp-attack/lira-v3      | 2025-10-30    | AUC + TPR@FPR, log-log ROC |
| …                   | …                             | …                       | …             | …               |
```

## MITRE ATLAS — the tactic + technique mapping

ATLAS organises adversary behaviour along a lifecycle (reconnaissance → initial access → execution → exfiltration → impact) with technique and sub-technique IDs. The relevant tactics for privacy attacks on model weights are primarily **Collection** and **Exfiltration**. The relevant techniques (verify current ATLAS IDs before citing; the knowledge base evolves):

- **Membership Inference Attacks** — ATLAS technique for black-box or white-box inference that a record was in training.
- **Infer Training Data Membership** — a related finer-grained technique.
- **Reveal Training Data** — the extraction-family techniques, often with sub-techniques for different attack vectors (prompt-divergence, repeat-inducing prompts, etc.).

Rather than hard-code technique IDs in this chapter (ATLAS updates them), build your coverage row by looking up the current ID on the ATLAS site and recording it in your harness's output alongside your attack reference. The template:

```
| ATLAS tactic | ATLAS technique ID | Technique name         | Our harness ref        |
|--------------|--------------------|------------------------|------------------------|
| Collection   | AML.T####         | Membership Inference    | mlp-attack/lira-v3     |
| Exfiltration | AML.T####         | Reveal Training Data    | extract/pythia-v2      |
| …            | …                  | …                       | …                      |
```

The ATLAS mapping also connects to **real-world case studies** the knowledge base hosts (incidents and reported attacks). Where your attack surface matches a specific case study, cite it — this is the same evidence structure a SOC team uses to prioritise defence, and showing the alignment shortens the cross-disciplinary conversation.

## OWASP Top 10 for LLM Applications — LLM02

The OWASP entry closest to this module is **LLM02 — Sensitive Information Disclosure**. The entry's canonical scenarios include:

- Training-data leakage via model outputs.
- Memorised PII in completions.
- Inversion / inference leakage of sensitive attributes.
- Leakage via tools or plugin integrations.

A mod-105 coverage claim against LLM02 names which of your attacks measures each scenario:

```
| OWASP LLM scenario (LLM02)     | mod-105 attack                           | Chapter |
|-------------------------------|------------------------------------------|---------|
| Training-data leakage         | Carlini extraction + canary dose-response | 05 + 06 |
| Memorised PII in completions  | Canary measurement + shingle probing      | 05 + 06 |
| Attribute inference           | Attribute-inference measurement           | 04      |
| Inversion leakage             | Fredrikson model inversion                | 04      |
| Tool / plugin leakage         | Agent-tool extraction eval                | 05 (+ mod-109) |
```

Note that OWASP also publishes related entries that may touch privacy indirectly (LLM03 Supply-Chain and others that have shifted across versions). Scan the current OWASP list at report time; LLM02 is the primary anchor but the list evolves.

## The coverage-matrix artifact

The chapter's deliverable is a single table that encodes the three mappings together. One row per attack in the module's suite. Columns:

- **Attack name** (our label, pinned to our harness).
- **Target model class** (classifier / LLM / tabular / vision / etc).
- **NIST AI 100-2 category.**
- **MITRE ATLAS tactic + technique ID.**
- **OWASP LLM Top 10 entry (if applicable).**
- **Harness reference** — ML Privacy Meter config path + git SHA, or the external harness.
- **Last measured** — timestamp + model SHA.
- **Metric reported** — AUC, TPR@FPR, extraction rate, exposure, etc.
- **Owner** — your name / team.
- **Peer dependency** — e.g. `model-evaluation-engineer` for CI, `ai-risk-engineer` for the broader-register coverage.

This is the artefact a review consumes. A reviewer cross-references the list of attacks they expect to see (from the taxonomy they use) against your rows. Any taxonomy entry without a corresponding row is a *known gap*, documented or not.

## Known gaps to name explicitly

A defensible table also calls out what the module does *not* cover. Three gaps the module names:

- **Property / distribution inference** (AI 100-2 category, ATLAS collection sub-technique). The module does not measure "what fraction of training data has property P". Route to the `ai-risk-engineer` broader risk register or to a bespoke audit when the use case demands it.
- **Model extraction / weight stealing** (AI 100-2 confidentiality, ATLAS exfiltration sub-techniques). Out of scope — `ai-security-engineer` scope. Route via the boundary.
- **Side-channel privacy attacks** (timing-side, cache-side, electromagnetic). Hardware-security surface; `ai-infra` role family, not this one.

A named gap with a routed owner is not a disqualifier. An unnamed gap is.

## Why the mapping sits in this module rather than mod-111

mod-111 (DPIA + regulator-grade evidence) *consumes* this mapping. mod-111 is where the DPIA aggregates: data-governance evidence, PII-pipeline evidence, PET-measurement evidence, retention-and-DSR evidence, model-card evidence. The privacy-attack coverage mapping is one of those inputs. Making the mapping here (mod-105) rather than there means:

- The mapping is maintained by the person who owns the attacks. When a new attack is added to the harness (or the harness changes), the mapping change lives next to the harness change.
- The DPIA author in mod-111 reads from a pinned artefact, not from an evolving freshness problem.
- The mapping is versioned alongside the harness and the measurements. "Which attack mapping did we claim coverage on in release v2.3?" is answerable from the version-controlled artefact.

## Summary — what you can do after this chapter

- Name the three taxonomies, the reviewer audience each serves, and why coverage must map to all three.
- Build the AI 100-2 row-mapping for the module's attack suite.
- Build the ATLAS tactic + technique row-mapping (looking up current IDs on the live ATLAS site).
- Build the OWASP LLM02 scenario row-mapping and scan the current OWASP list at report time for adjacent entries.
- Produce the unified coverage-matrix artefact with the column set this chapter names.
- Name the three out-of-scope categories (property inference, model extraction, side channels) and the roles that own each.

Chapter 10 draws the depth boundary to `ai-risk-engineer`'s mod-107 so the two tracks' coverage claims are complementary, not duplicative.
