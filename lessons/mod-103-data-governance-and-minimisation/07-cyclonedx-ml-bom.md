# CycloneDX ML-BOM for Data Provenance

The datasheet (chapter 02) is the human-readable description of a dataset. The ML-BOM is the *machine-readable* companion that lets a downstream consumer — an auditor, an acquirer, an incident responder, or a release-assurance workflow — walk from a deployed model back to its training inputs, the components that produced it, and the datasets' consent / de-identification claims, *without* reading twelve markdown files.

CycloneDX is the Linux-Foundation / OWASP standard that ships this. Since v1.5 (2023) and strengthened in v1.6 (2024), CycloneDX includes ML-BOM and dataset / model components; the AI Governance Working Group maintains ongoing evolution. SPDX AI Profile is an adjacent standard filling similar needs. This chapter is the CycloneDX-side engineering slice.

Primary references:

- **CycloneDX** — specification and tooling. [cyclonedx.org](https://cyclonedx.org/).
- **CycloneDX ML-BOM** — AI / ML BOM shape, dataset and model components. [cyclonedx.org/capabilities/mlbom](https://cyclonedx.org/capabilities/mlbom/).
- **OWASP CycloneDX Authoritative Guide** — the long-form explainer. [cyclonedx.org/guides](https://cyclonedx.org/guides/).
- **CycloneDX AI/ML examples repository** — reference `bom.json` samples.
- **NTIA SBOM Minimum Elements** — the baseline obligation shape. [ntia.gov/files/ntia/publications/sbom_minimum_elements_report.pdf](https://www.ntia.gov/files/ntia/publications/sbom_minimum_elements_report.pdf).
- **US Executive Order 14028 (2021)** — the regulatory stimulus that mainstreamed SBOMs; ML-BOM is the AI-specific extension.
- **ISO/IEC 5962:2021 (SPDX)** — adjacent SBOM standard.

## Why this matters

Three concrete questions regulators and acquirers increasingly ask that an ML-BOM answers in one call:

- "Enumerate every dataset that trained this deployed model, each dataset's consent basis, and each dataset's de-identification state." — a datasheet per corpus, cross-linked from the model record.
- "A data subject requested erasure. Which models, retrieval indexes, and derived artifacts are downstream of their records?" — a provenance traversal from a subject-indexed input to deployed artifacts.
- "We suspect this deployed model was trained on data from a feed that violated its contract. Scope the blast radius." — a reverse traversal from a dataset to every model that consumed it.

A pipeline without an ML-BOM answers these by hand. A pipeline with an ML-BOM answers them by `jq`.

The ML-BOM is also an Art. 25 accountability artifact, an EU AI Act Art. 11 technical-documentation artifact, and an SPDX-alternative compliance deliverable where SBOM obligations apply to AI systems.

## The CycloneDX ML-BOM shape at a glance

CycloneDX is JSON (and XML; use JSON). A BOM is a document with:

- **`bomFormat`** — `CycloneDX`.
- **`specVersion`** — `1.6` at time of writing.
- **`serialNumber`** — a stable URN for this BOM.
- **`version`** — integer version of this BOM.
- **`metadata`** — timestamps, tools that produced the BOM, authors, licence, and the *root component* (the thing this BOM is about — usually the deployed model).
- **`components`** — the list of things referenced by the BOM. For ML-BOM, these include components of type `machine-learning-model` and `data`, plus `library` / `application` / `container` for the training and serving stack.
- **`dependencies`** — the graph: component X depends on components Y, Z. A trained model depends on its training datasets, its base model, its training-framework, and its training-pipeline code.
- **`services`** — relevant where the model is served by or consumes external services.
- **`compositions`** — assertions about the completeness of parts of the BOM (useful for "all training datasets are enumerated above" claims).

For a privacy-focused ML-BOM, the components of interest are **model components** (type `machine-learning-model`) and **data components** (type `data`), with a dependency graph that shows which datasets fed which model.

## The dataset component — fields you must fill

A CycloneDX `data` component with the fields that carry the privacy signal:

```json
{
  "bom-ref": "ds.ticket-corpus.2025-q4",
  "type": "data",
  "name": "Customer-support ticket corpus — 2025 Q4 fine-tune slice",
  "version": "3",
  "description": "Training-corpus slice for the support-assistant fine-tune. See datasheet.v3.yaml.",
  "data": [{
    "name": "ticket-corpus-2025-q4",
    "type": "dataset",
    "contents": {
      "url": "s3://example-training/datasets/ticket-corpus/2025-q4/manifest.json",
      "attachment": { "contentType": "application/json", "content": "<redacted inline manifest>" }
    },
    "classification": "personal-data; non-special-category",
    "sensitiveData": ["name", "email", "phone", "billing_address", "free_text"],
    "graphics": [],
    "description": "412_998 ticket threads; see ds.ticket-corpus.2025-q4 datasheet section Group A.",
    "governance": {
      "owners": [{ "contact": { "name": "DPO", "email": "dpo@example.test" }}],
      "custodians": [{ "contact": { "name": "ml-platform", "email": "ml-platform@example.test" }}]
    }
  }],
  "properties": [
    { "name": "privacy:lawful_basis",         "value": "legitimate_interests" },
    { "name": "privacy:purpose",              "value": "fine_tune_support_assistant_v3" },
    { "name": "privacy:deidentification",     "value": "presidio-v2.4 + vault-tokenisation-v3" },
    { "name": "privacy:consent_token_scheme", "value": "opaque_deterministic_v3" },
    { "name": "privacy:retention_deadline",   "value": "2027-10-01" },
    { "name": "privacy:jurisdiction",         "value": "EU,EEA,CH" },
    { "name": "privacy:datasheet",            "value": "ref:ds.ticket-corpus.2025-q4@v3" },
    { "name": "privacy:residual_risk_claim",  "value": "ref:ds.ticket-corpus.2025-q4.residual-risk.md" }
  ]
}
```

Notes:

- `properties` is the CycloneDX extension point. Use a `privacy:` namespace consistently.
- `classification` and `sensitiveData` are the standard CycloneDX fields; populate them.
- Reference the full datasheet via `properties["privacy:datasheet"]` — the BOM does not embed the full datasheet (keep it a manifest, not a data lake).
- Each dataset version is a separate component (`version` field). Regenerating the dataset produces a new BOM entry.

## The model component — fields that carry data provenance

A `machine-learning-model` component for the deployed fine-tune:

```json
{
  "bom-ref": "model.support-assistant.fine-tune.v3",
  "type": "machine-learning-model",
  "name": "support-assistant-fine-tune",
  "version": "3",
  "modelCard": {
    "modelParameters": {
      "approach": { "type": "supervised" },
      "task": "chat completion for tenant-scoped support assistant",
      "architectureFamily": "transformer",
      "modelArchitecture": "base-llm-7b (ref:models/base-llm-7b@v1)",
      "datasets": [
        { "ref": "ds.ticket-corpus.2025-q4" }
      ]
    },
    "quantitativeAnalysis": {
      "performanceMetrics": [ /* link eval results */ ]
    },
    "considerations": {
      "technicalLimitations": ["tenant-scoped; cross-tenant generalisation not supported"],
      "performanceTradeoffs": ["DP not applied at this training run; see residual-risk"],
      "ethicalConsiderations": [{
        "name": "memorisation",
        "description": "Monitored per mod-105 sampler; see monthly audit report."
      }]
    }
  },
  "properties": [
    { "name": "privacy:dpia",             "value": "ref:dpia.support-assistant.v3" },
    { "name": "privacy:threat_model",     "value": "ref:threat-model.support-assistant.v3" },
    { "name": "privacy:ml_bom_fragment",  "value": "self" },
    { "name": "privacy:dp_budget",        "value": "none" },
    { "name": "privacy:unlearning_plan",  "value": "ref:mod-110.unlearning.support-assistant.v3" }
  ]
}
```

Notes:

- The `modelCard.modelParameters.datasets` list is the primary provenance pointer. One `ref` per training dataset.
- `properties` link to the DPIA, threat model, and unlearning plan — the regulator-facing bundle.
- A model fine-tuned from a base model has a `modelArchitecture` reference to the base; that base is *also* a component in the BOM (or in a BOM the base-model vendor supplies).

## The dependency graph — walkable provenance

The `dependencies` array is what makes the BOM queryable:

```json
"dependencies": [
  { "ref": "model.support-assistant.fine-tune.v3",
    "dependsOn": [
      "ds.ticket-corpus.2025-q4",
      "models/base-llm-7b@v1",
      "pipeline.fine-tune.v3",
      "framework.transformers@4.46.2"
    ]
  },
  { "ref": "pipeline.fine-tune.v3",
    "dependsOn": [
      "framework.transformers@4.46.2",
      "lib.torch@2.4.0",
      "vault.policy@v3"
    ]
  }
]
```

Three queries this enables:

- **Model → datasets**: "What did `model.support-assistant.fine-tune.v3` train on?" Walk `dependsOn` to the `data` components.
- **Dataset → models**: "What models consumed `ds.ticket-corpus.2025-q4`?" Reverse-walk the graph. Crucial for DSR / erasure scoping (mod-110).
- **Model → base-model + framework versions**: "What base-model + library stack produced this model?" Same walk.

A pipeline that emits a BOM per training run accumulates a traversable history.

## Where to attach the privacy properties

A persistent question in ML-BOM authoring is: *which fields carry the privacy signal?*. Three options, pick one per field and be consistent:

- **CycloneDX standard fields.** Use these first where they exist: `classification`, `sensitiveData`, `governance`, `data.contents`.
- **CycloneDX properties** (`privacy:lawful_basis`, …). Use for anything not covered by a standard field. Namespace consistently.
- **External reference by URN.** A `privacy:datasheet` property that resolves to `ds.ticket-corpus.2025-q4@v3` in your datasheet registry; the BOM carries a stable reference, the full detail lives in the registry.

Avoid embedding the full datasheet in the BOM — the BOM is a manifest. The datasheet, the DPIA, the threat model, and the audit reports are all separate artifacts the BOM points to.

## Generating the BOM — tooling

Three generation patterns:

- **Pipeline-integrated** — the training pipeline emits a `bom.json` as a build artifact of the trained model. Attach the BOM to the model artifact in the registry (MLflow, SageMaker, Vertex AI model registry).
- **Catalog-integrated** — a data catalog (Unity Catalog, OpenMetadata, DataHub) emits the dataset components; the training pipeline consumes the dataset BOM fragments and composes them with the model's BOM fragment.
- **Hand-authored + CI-verified** — the BOM is a reviewed YAML / JSON in the repo; CI validates it against the CycloneDX schema and asserts referenced components exist.

Tools worth knowing:

- **cyclonedx-python-lib** / **cyclonedx-bom** — reference libraries for generation and validation.
- **CycloneDX CLI** — validation, convert between formats, diff two BOMs.
- **Dependency-Track** — ingest and query CycloneDX BOMs at fleet scale.

## Versioning, signing, and distribution

Three operational rules:

- **One BOM per trained-model version.** Not one BOM for "the model" over time. Each training run produces a new BOM; old BOMs stay queryable.
- **Signed BOM.** The pipeline signs the BOM (cosign / Sigstore) before publishing. A tampered BOM is detectable at verification. CycloneDX supports JSON signatures per RFC 7515.
- **Distributed with the model.** When the model is published to an external registry, the BOM is attached as metadata (Sigstore attestation, OCI referrer, or a sidecar file). Internal registries follow the same shape.

## How this helps the DPIA, DSR, and incident response

- **DPIA (mod-111).** The DPIA's "description of processing" section lists training datasets, retrieval sources, models used, and sub-processors. The BOM is the machine-readable source for that list.
- **DSR erasure (mod-110).** Given an erasure request for subject X, scope: which datasets contain X (from capture-side provenance), which models were trained on those datasets (BOM dependency walk), which retrieval indexes consumed those datasets (BOM dependency walk), which serving endpoints host those models (deployment metadata). The BOM is one half of the DSR scope.
- **Privacy-incident response.** A corrupted upstream feed has produced malformed labels. Scope: which datasets consumed the feed, which models consumed those datasets, which deployments serve those models. Same walk.

In each case, the BOM saves a day of manual investigation on the critical path of a regulatory clock (GDPR Art. 33 — 72 hours; Art. 12(3) DSR — one month).

## Example — the end-to-end slice for a RAG system

For a tenant-scoped RAG assistant:

```json
{
  "bomFormat": "CycloneDX",
  "specVersion": "1.6",
  "serialNumber": "urn:uuid:…",
  "version": 3,
  "metadata": {
    "timestamp": "2025-10-14T12:00:00Z",
    "tools": [ { "name": "cyclonedx-python-lib", "version": "7.4.0" } ],
    "component": { "bom-ref": "model.support-assistant.fine-tune.v3", "type": "machine-learning-model", "name": "support-assistant-fine-tune", "version": "3" }
  },
  "components": [
    { /* model.support-assistant.fine-tune.v3 — as above */ },
    { /* ds.ticket-corpus.2025-q4 — as above */ },
    { /* ds.ticket-corpus-synth.v1 — synthetic eval set, chapter 06 */ },
    { /* retrieval-corpus.tenant-isolated.v2 — retrieval corpus */ },
    { /* base-llm-7b@v1 — base model (may be a reference to vendor BOM) */ },
    { /* vault.policy@v3 — the vault policy, chapter 05 */ },
    { /* pipeline.fine-tune.v3 — training pipeline component */ }
  ],
  "dependencies": [
    { "ref": "model.support-assistant.fine-tune.v3",
      "dependsOn": ["ds.ticket-corpus.2025-q4", "base-llm-7b@v1", "pipeline.fine-tune.v3", "vault.policy@v3"] },
    { "ref": "retrieval-corpus.tenant-isolated.v2",
      "dependsOn": ["ds.ticket-corpus.2025-q4", "encoder.bge-m3@v1"] }
  ],
  "compositions": [
    { "aggregate": "complete",
      "assemblies": ["model.support-assistant.fine-tune.v3"],
      "dependencies": ["ds.ticket-corpus.2025-q4", "base-llm-7b@v1"] }
  ]
}
```

The `compositions.aggregate = "complete"` assertion is the formal claim that the enumerated training datasets are the *entire* set of training datasets. An auditor relies on this.

## Common failure modes

- **Partial provenance.** The BOM enumerates the dataset you remembered; a second, forgotten dataset (the "augmentation pass we ran once last March") is silent. Combat with: pipeline-emitted BOM + `compositions.aggregate = "complete"` verified by a CI test that diffs the pipeline-declared dataset list against the BOM.
- **Dataset-version drift.** The BOM references `ds.ticket-corpus.2025-q4` but the dataset has been regenerated as v4 with the same name. Combat with: BOM references include the version suffix (`@v3`).
- **Base-model opacity.** The vendor base model's BOM is absent; your BOM says `base-llm-7b` but nothing about *its* training data. Record explicitly; many regulator-facing posters accept "vendor-supplied base model; vendor's data-governance is referenced here (vendor model card URL)".
- **Datasheet rot.** The BOM points to a datasheet URL that no longer resolves. CI test: BOM references are reachable and version-matching.

## Where SPDX AI Profile fits

SPDX's AI Profile (since SPDX 3.0) covers similar ground with a different data model. Two pragmatic observations:

- Tool support is wider for CycloneDX today; the AI Governance Working Group work is further along.
- Where your organisation already standardises on SPDX for SBOM, the AI Profile extension is the natural path. Author one or the other; converters exist.

The critical piece is not which format — it is that the provenance *is machine-readable at all*.

## Summary — what you can do after this chapter

- Author a CycloneDX ML-BOM slice that enumerates datasets, model components, and dependencies, with privacy metadata carried via standard fields + `privacy:` properties.
- Reference the datasheet, DPIA, threat model, unlearning plan, and audit reports from the BOM without embedding them.
- Use the BOM for the three canonical queries — model → datasets, dataset → models, model → stack — for DPIA, DSR, and incident response.
- Recognise and mitigate the four common failure modes (partial provenance, dataset drift, base-model opacity, datasheet rot).

The next chapter draws the coordination boundary between this role and the `training-pipeline-engineer`, `rag-engineer`, and `llm-application-developer` peer roles — who owns which half of the data-governance surface this module describes.
