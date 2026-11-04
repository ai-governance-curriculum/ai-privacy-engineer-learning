# Exercise 06: ML-BOM Provenance Slice

**Estimated effort:** 1 hour · **Output:** `bom.json` (CycloneDX 1.6) + `bom-review.md` + one CI verification stub · **Primary chapter:** 07

## Objective

Author a **CycloneDX ML-BOM slice** for the model your module 103 artifacts describe, enumerating the datasets, the model component, the dependency graph, and the privacy metadata (lawful basis, retention, datasheet, DPIA, threat model, unlearning plan). Verify the three canonical queries (model → datasets; dataset → models; model → stack) run against your BOM.

## Prerequisites

- Chapter 07 (CycloneDX ML-BOM) read.
- Datasheets from exercise 01 and (if completed) exercise 05.
- Propagation contract from exercise 02 — the BOM references the propagation version.
- Minimisation register from exercise 03 — the BOM `properties` reference the register ID.
- Vault design from exercise 04 — the BOM references the vault policy component.
- [CycloneDX specification](https://cyclonedx.org/) open in a tab.
- `cyclonedx-cli` or `cyclonedx-python-lib` installed for validation; `jq` for queries.

## Problem statement

Produce three deliverables.

### Deliverable A — `bom.json`

A CycloneDX 1.6 JSON BOM for a specific trained model version. Must include:

- **`metadata.component`** — the root component: the deployed model (`type: machine-learning-model`) with `name`, `version`, and `bom-ref`.
- **`components`** — at minimum:
  - The root model component (also referenced from metadata).
  - One `data` component per training / fine-tune / retrieval / evaluation / synthetic dataset referenced (one per datasheet).
  - The base model (`type: machine-learning-model`) as a dependency.
  - The training pipeline component (`type: application` or `type: library`).
  - At least one `library` component for a material framework (transformers / torch / sklearn — pick what is real for your system).
  - The vault policy component (reference).
- **`dependencies`** — the graph:
  - Model → datasets.
  - Model → base model.
  - Model → training pipeline.
  - Training pipeline → frameworks.
  - Retrieval index → retrieval corpus + encoder.
- **`compositions`** — at least one assertion with `aggregate: complete` stating that the enumerated training datasets are the entire set.
- **`properties`** (per component) carrying:
  - On every `data` component: `privacy:lawful_basis`, `privacy:purpose`, `privacy:deidentification`, `privacy:retention_deadline`, `privacy:jurisdiction`, `privacy:datasheet`, `privacy:residual_risk_claim`.
  - On the root model component: `privacy:dpia`, `privacy:threat_model`, `privacy:ml_bom_fragment`, `privacy:dp_budget` (or `none`), `privacy:unlearning_plan`.

Validate against the CycloneDX 1.6 schema (`cyclonedx validate --input-file bom.json` or equivalent).

### Deliverable B — `bom-review.md`

A short Markdown walk-through of the BOM for a reader who has never seen it:

- One paragraph: what model does this BOM describe, and what deployment?
- The dependency graph diagram (Mermaid or ASCII) rendered from the BOM.
- One line per component naming what it is and where its backing datasheet / policy / plan lives.
- **Three canonical queries, run against the BOM and showing the output:**
  1. `Which datasets trained model X?` — `jq`-style one-liner + output.
  2. `Which models consumed dataset Y?` — in a multi-model BOM registry, this is a reverse walk; for a single BOM, note what the query would look like at registry scale.
  3. `What is the training-stack / framework version for model X?` — `jq`-style one-liner + output.
- A short "gaps" section — components you could not fully document (e.g. vendor base-model opacity) and how you recorded the gap.

### Deliverable C — CI verification stub

A test stub (pseudocode or your language of choice) that CI would run on every model-release PR, verifying:

- `bom.json` validates against CycloneDX 1.6 schema.
- Every `data` component carries the required `privacy:` properties (listed above).
- Every `privacy:datasheet` reference resolves to an existing datasheet in the registry.
- The root model component's `privacy:threat_model` and `privacy:dpia` references resolve.
- The `compositions.aggregate = complete` assertion matches the pipeline's declared dataset set (compare BOM dataset list to pipeline input manifest; fail on divergence).

## Starter guidance

- **Start from a published CycloneDX ML-BOM example.** The CycloneDX AI/ML examples repository and the authoritative-guide samples contain starting shapes you can edit.
- **`bom-ref` names matter.** Use stable, informative identifiers (`ds.ticket-corpus.2025-q4`, `model.support-assistant.fine-tune.v3`, `pipeline.fine-tune.v3`). The ML-BOM queries work only if `bom-ref` is readable.
- **Prefer references over inline embedding.** The BOM is a manifest; the full datasheet, DPIA, threat model, and audit live elsewhere. Use `properties` with `ref:` URLs or stable IDs.
- **Fill `compositions.aggregate = complete` only if it is true.** A partial BOM that falsely claims completeness is worse than no BOM.
- **Record base-model opacity explicitly.** Vendor base models rarely expose a BOM; record the vendor's model card URL in a `properties` entry and state the gap in `bom-review.md`.
- **Sign the BOM if your tooling supports it.** Not required for the exercise; useful in production.

## Acceptance criteria

- [ ] `bom.json` validates against the CycloneDX 1.6 schema.
- [ ] The root component is a `machine-learning-model` with `name`, `version`, `bom-ref`.
- [ ] At least three `data` components with the required `privacy:` properties.
- [ ] Dependencies walked: model → datasets → base model → training pipeline → libraries.
- [ ] `compositions.aggregate: complete` asserted for the training-dataset set, and the assertion matches reality.
- [ ] `bom-review.md` includes all three canonical queries with actual outputs.
- [ ] CI verification stub names the four checks with concrete oracles.
- [ ] Gaps section exists; any un-fillable field is explicitly called out, not quietly omitted.

## Stretch goals

- Author the BOM as part of your training pipeline's emit step so new training runs produce new BOM versions automatically.
- Sign the BOM with Sigstore / cosign and record the verification step in the review doc.
- Produce a *second* BOM for a retrieval-only system (no training) — a RAG index with no fine-tune — and compare the shape to the training-based BOM. Which components / dependencies differ?
- Load your BOM into **Dependency-Track** (open-source CycloneDX ingestion tool) and show the traversable graph in the UI.
- Author an SPDX AI-Profile equivalent of the same BOM and compare the two shapes.

## What this lands

- The BOM is the machine-readable artifact that stitches every mod-103 output together — datasheet (ex. 01), propagation contract (ex. 02), minimisation register (ex. 03), vault policy (ex. 04), synthetic audit (ex. 05).
- The BOM participates in DSR scoping (mod-110), DPIA packaging (mod-111), and release-assurance review (`ai-evaluation-engineer`, level 35).
- The CI verification stub keeps the BOM honest across future model versions.

Version the BOM per training run. Sign it. Attach it to the model artifact in your registry.
