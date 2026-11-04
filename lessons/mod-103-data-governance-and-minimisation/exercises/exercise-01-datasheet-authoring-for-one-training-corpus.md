# Exercise 01: Datasheet Authoring For One Training Corpus

**Estimated effort:** 2 hours · **Output:** two files — `datasheet.md` (human-readable) + `datasheet.yaml` (machine-readable manifest) · **Primary chapter:** 02

## Objective

Author a **Datasheet-for-Datasets** slice for one real-or-representative training corpus — covering Gebru et al.'s seven sections, the AI-specific consent / lawful-basis / de-identification / retention extensions, and a signed-ready residual-risk claim. The datasheet is the first artifact that is reviewed when a regulator, auditor, or acquirer asks about a corpus; this exercise is where you build the muscle to produce one you would be comfortable publishing.

## Prerequisites

- Chapter 02 (datasheets for datasets) read; Gebru et al. (2021) read at least once.
- Chapter 01 (motivation) read so the stack of outputs this module produces is clear.
- A training corpus you can describe at provenance / composition / use / maintenance depth. Candidates:
  - A corpus you've shipped or shipped-against at work (preferred — the exercise is highest-value here).
  - A public corpus with enough documentation to answer most questions (Common Crawl slice; MIMIC-III / MIMIC-IV for health; HELM / BIG-bench slices; a Hugging Face dataset with a dataset card).
  - If you have no corpus in mind, use **the default scenario** from mod-102 exercise 01 — the tenant-scoped RAG customer-support assistant's ticket corpus — so you can finish.

## Problem statement

Produce two files.

### `datasheet.md` — the human-readable document

A Markdown document covering Gebru et al.'s seven sections, each with the AI-specific extensions from chapter 02:

- **§1 Motivation** — who created the dataset, for what purpose, funded by whom.
- **§2 Composition** — instance representation; counts; sampling; splits; relationships; duplicates; demographic representativeness.
- **§3 Collection Process** — timeframe; mechanism; consent + lawful basis; cross-border transfers; sensitive-data classes; children's data; opt-outs.
- **§4 Preprocessing / Cleaning / Labeling** — de-identification state (which techniques, which fields, residual re-id risk); labeller access; vendor processors.
- **§5 Uses** — intended uses; *un*intended uses; licence; downstream distribution; known biases; representativeness gaps.
- **§6 Distribution** — recipients; agreements; cross-border mechanism; access controls.
- **§7 Maintenance** — owner; versioning; retention; rectification + erasure routing; residual-risk claim with signatories.

The **residual-risk claim** (end of §7) is a short paragraph — name the specific residual scenarios, the compensating control for each, the trigger that would re-open the assessment, and the signing authorities (DPO + data owner + privacy engineer).

### `datasheet.yaml` — the machine-readable manifest

A YAML file matching the shape in chapter 02, filling every field with a *real* value (use `unknown` or `tbd:<reason>` where you genuinely do not know — the point is that the gaps are named, not filled in with lies).

Required top-level keys at minimum: `id`, `title`, `version`, `owners`, `motivation`, `creator`, `collection_period`, `sampling`, `instance`, `size`, `splits`, `duplicates`, `personal_data_classes`, `special_category`, `children_data`, `lawful_basis`, `lia_doc` (if applicable), `consent_available_for`, `deidentification`, `labelers`, `cross_border`, `intended_use`, `unintended_use`, `licence`, `retention`, `rectification_sla`, `erasure_sla`, `residual_risk`.

## Starter guidance

- **Write §7 first.** The residual-risk claim is the hardest section; writing it first forces you to confront the honest gaps earlier and the rest of the datasheet becomes shaped by what you choose to defend.
- **Be specific about lawful basis.** "Consent" or "legitimate interests" is not enough — name the LIA document reference, the consent-text version, the opt-out mechanism.
- **If you cannot name a controller / processor, that is the first finding.** An unowned dataset is a finding.
- **De-identification state matters at the field level.** "Redacted" is not a state; "Presidio-v2.4 + vault tokenisation of {name,email,phone}, residual re-id risk in free-text" is.
- **Include a known-biases / representativeness statement.** EU AI Act Art. 10(3) requires it for high-risk systems; it also makes the datasheet useful to the downstream model-card author.
- **Keep the markdown and YAML synchronised.** Any machine-readable field in YAML should have a human-readable paragraph in the Markdown. The YAML is a manifest; the Markdown is the review document.

## Acceptance criteria

- [ ] All seven Gebru sections present and non-empty (use `tbd:<reason>` for honest gaps; do not silently omit).
- [ ] The dataset is named with a stable ID (date / version / name convention declared).
- [ ] Lawful basis is specifically identified (consent / contract / legitimate interests / legal obligation / public task / vital interest), with references to consent-text version or LIA document.
- [ ] Each personal-data class present is named; special-category / PHI / PCI / children's data explicitly flagged (or explicitly excluded with a mechanism).
- [ ] De-identification state is per-field — not a global claim.
- [ ] Intended use section names at least three *un*intended uses the dataset is not released for.
- [ ] Retention is a specific timeframe with a deletion mechanism.
- [ ] Residual-risk claim: at least two named residual scenarios, each with a compensating control and a re-opening trigger; three signatories (DPO + data owner + privacy engineer) with roles identified.
- [ ] YAML and Markdown are consistent — any YAML field is traceable to a Markdown paragraph.
- [ ] Datasheet version is referenceable from the ML-BOM you will author in exercise 06.

## Stretch goals

- Produce a second datasheet for a **retrieval corpus** derived from the training corpus — using the retrieval-specific fields (tenant scope, chunking, encoder version, author-indexing consent, right-to-be-forgotten path) from chapter 02.
- Produce a third datasheet for an **evaluation set** sampled from production — with explicit sampling procedure, freeze policy, and eval-specific residual-risk statement.
- Author a `redaction / sanitisation` diff — the Markdown datasheet with and without the sensitive composition statistics, intended for *external publication*. The public version should retain the residual-risk claim and signatories.
- Peer-review a classmate's / colleague's datasheet. Use the chapter 02 completeness checklist as the review template.

## What this lands

- The datasheet is the primary input to exercise 06 (ML-BOM provenance slice) — the BOM's `data` component will reference this datasheet by ID + version.
- The YAML manifest is machine-readable by the training pipeline so changes to the dataset's lawful-basis / retention trigger CI alerts.
- The residual-risk claim becomes evidence in the DPIA (mod-111).
- A stable datasheet convention for your team outlives any individual dataset.

Version the two files. Date the signatures. Plan for the datasheet to be regenerated when the dataset is.
