# mod-103-data-governance-and-minimisation: Data Governance and Minimisation for AI (Training, Retrieval, Inference)

**Estimated effort:** 14 hours

This module teaches **data governance and minimisation** as the foundation layer of AI privacy engineering — the one every later module builds on. If `mod-102` produced the privacy threat model and the risk register, this module produces the *controls on the data itself*: datasheets that describe the corpus, consent / legal-basis tags that survive ETL and vectorisation, minimisation checkpoints at ingest / processing / retention / output, a data-privacy-vault + tokenisation pattern so raw PII never reaches the model layer, synthetic-data substitutes where real data cannot be used, and a CycloneDX ML-BOM slice that lets an auditor trace a deployed model back to its training inputs and consent basis.

## Learning objectives

- Author a Datasheet-for-Datasets slice for a training corpus and a retrieval corpus — collection process, consent / legal basis, intended use, retention, de-identification state, and residual-risk claim — comfortable publishing alongside a model release.
- Engineer consent / legal-basis propagation through the data pipeline — from capture to training to retrieval to inference to logging — using column-level / document-level provenance tags that survive ETL and vectorisation.
- Engineer data-minimisation controls — collection minimisation (what not to collect), processing minimisation (minimum-necessary for the task), retention minimisation (time-boxed and purpose-bound retention), and output minimisation (what the model should refuse to generate).
- Deploy a data-privacy-vault + tokenisation pattern (Skyflow-shape) so sensitive PII is replaced with reversible tokens before it ever reaches the model layer; integrate the vault with training, retrieval, and inference flows.
- Deploy synthetic-data substitutes where real data cannot be used — tabular synth (CTGAN, TabDDPM, Gretel, MOSTLY AI, Tonic), text synth, health synth (Synthea) — and audit utility vs. privacy of the synthesised set.
- Author a CycloneDX ML-BOM slice for data provenance so an auditor can trace a deployed model back to its training inputs and consent basis.
- Coordinate with `training-pipeline-engineer` (peer, level 25) on the training-pipeline integration and with `rag-engineer` / `llm-application-developer` (peers, level 25) on the retrieval-corpus and application-side flows.

## Chapters

1. [Why data governance and minimisation is the AI-privacy foundation](./01-why-data-governance-for-ai.md) — motivation, the four controls this module delivers, and how they stack with the mod-102 threat model.
2. [Datasheets for datasets — the engineering interface](./02-datasheets-for-datasets.md) — Gebru et al.'s questionnaire adapted for training *and* retrieval corpora, with a worked slice you can publish alongside a model release.
3. [Consent and legal-basis propagation across the pipeline](./03-consent-and-legal-basis-propagation.md) — column / document / chunk-level provenance tags that survive ETL, embedding, and inference logging.
4. [Four kinds of minimisation — collection, processing, retention, output](./04-data-minimisation-controls.md) — the GDPR Art. 5(1)(b)(c)(e) obligations, each as an engineering checkpoint.
5. [Data privacy vaults and tokenisation](./05-data-privacy-vaults-and-tokenisation.md) — Skyflow-shape architecture, token types, and vault integration across training, retrieval, and inference.
6. [Synthetic-data substitutes and their utility / privacy audit](./06-synthetic-data-substitutes.md) — CTGAN, TabDDPM, Gretel, MOSTLY AI, Tonic, Synthea; the attacks that the audit must survive.
7. [CycloneDX ML-BOM for data provenance](./07-cyclonedx-ml-bom.md) — the slice of CycloneDX that carries datasets, model inputs, and consent basis to the auditor.
8. [The peer coordination boundary](./08-peer-coordination-boundary.md) — what this role owns vs. what `training-pipeline-engineer`, `rag-engineer`, and `llm-application-developer` own on the shared surface.

## Exercises

Each exercise is a hands-on build; outputs stack into a reusable data-governance bundle you will carry through the rest of the track.

1. [Datasheet authoring for one training corpus](./exercises/exercise-01-datasheet-authoring-for-one-training-corpus.md)
2. [Consent / legal-basis propagation contract](./exercises/exercise-02-consent-legal-basis-propagation-contract.md)
3. [Data minimisation checkpoint set](./exercises/exercise-03-data-minimisation-checkpoint-set.md)
4. [Data privacy vault + tokenisation pattern](./exercises/exercise-04-data-privacy-vault-tokenisation-pattern.md)
5. [Synthetic-data substitute build and audit](./exercises/exercise-05-synthetic-data-substitute-build-and-audit.md)
6. [ML-BOM provenance slice](./exercises/exercise-06-ml-bom-provenance-slice.md)

## Structure

- `01-…md` … `08-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
