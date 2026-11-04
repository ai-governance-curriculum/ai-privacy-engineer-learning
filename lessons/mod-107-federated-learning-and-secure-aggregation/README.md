# mod-107-federated-learning-and-secure-aggregation: Federated Learning, Secure Aggregation, and On-Device Privacy Engineering

**Estimated effort:** 14 hours

Federated learning (FL) is the privacy-engineering pattern that trains a shared model without centralising the training data. "The raw data stays where it is collected" is the marketing sentence; this module teaches the engineering reality. FL is really two production shapes (cross-device vs. cross-silo) with different threat models, two complementary privacy primitives (secure aggregation + differential privacy) composed on top of a federated optimiser (FedAvg / FedProx), and a governance framing (GDPR joint-controllership, lawful basis, Schrems II supplementary measures) that layers onto whichever technical design you ship. The module goes end-to-end: taxonomy → Flower code → library map → secure aggregation → DP-FL → FL-specific attacks → GDPR governance → boundary to the mlops peer.

## Learning objectives

- Distinguish cross-device FL (millions of edge devices, intermittent, non-IID) from cross-silo FL (small number of institutional parties, stable, IID-ish) — know which production shape fits which business problem.
- Engineer FedAvg and FedProx training loops with Flower, TensorFlow Federated, NVIDIA FLARE, and PySyft — client / server architecture, round-based training, heterogeneity handling, client-selection strategy.
- Engineer secure aggregation (Bonawitz et al.) on top of FL — pairwise masks, Shamir secret sharing, dropout tolerance; reason about the cost vs. the threat it defeats (honest-but-curious server, not malicious clients).
- Combine FL with DP (DP-FL) at the record / user / client level — know which DP granularity fits which threat model, what the composition cost is across rounds, and how the privacy budget interacts with the federated round cadence.
- Reason about FL-specific privacy attacks the suite does not defeat — gradient-based reconstruction (Deep Leakage from Gradients, iDLG), label-inference, membership-inference across rounds — and ship the mitigations.
- Reason about the data-governance and legal implications of FL under GDPR — joint-controller / controller-processor relationships, cross-border transfer, lawful-basis-of-each-participant, data-subject rights in a federated setting — leaving legal opinion out of scope to counsel.
- Cite the boundary to `ai-infra-mlops` (peer, level 25) on the FL-platform operational depth (client-side updater, model-store, round scheduler); this module owns the privacy-engineering slice of the design.

## Chapters

1. [Cross-device vs. cross-silo FL](01-cross-device-vs-cross-silo-fl.md) — the two production shapes, their decision checklist, the engineering realities each cascades, and the anti-patterns that recur in design docs.
2. [FedAvg and FedProx with Flower](02-fedavg-fedprox-with-flower.md) — the round as four phases, the FedProx proximal-term patch, the Flower client/server shape, and the client-selection strategy as a privacy decision.
3. [TFF, NVIDIA FLARE, and PySyft — the production-library map](03-tff-nvflare-pysyft-production-map.md) — design-axis cheat sheet, when to pick each, and the honourable-mention map (FATE, Substra, OpenFL, FedML).
4. [Bonawitz secure aggregation](04-bonawitz-secure-aggregation.md) — the rung-by-rung protocol (pairwise masks → Shamir → dropout tolerance → four-round protocol), parameter tuning, composition with DP, and the release-package SA stanza.
5. [DP-FL — record, user, and client granularity](05-dp-fl-record-user-client-granularity.md) — the granularity decision mapped to the threat model, three mechanical shapes for noise injection, composition cost across rounds, adaptive clipping, and the DP-FL stanza.
6. [FL-specific privacy attacks](06-fl-specific-privacy-attacks.md) — DLG / iDLG / Inverting Gradients, label inference, cross-round MIA, property inference, and the attack-coverage matrix per deployment.
7. [FL under GDPR](07-fl-gdpr-and-governance.md) — controller / processor / joint-controller classification, lawful basis per participant, Schrems II supplementary measures, subject rights in a federated setting, the AI Act Article 10 overlay, and the data-flow diagram.
8. [Boundary to `ai-infra-mlops`](08-boundary-to-ai-infra-mlops.md) — what the mlops peer owns vs. what this module owns, the four joint artefacts, the collision points, and the kickoff paragraph.

## Exercises

See [`exercises/`](exercises/):

1. [Cross-device vs. cross-silo design drill](exercises/exercise-01-cross-device-vs-cross-silo-design-drill.md)
2. [FedAvg with Flower, end to end](exercises/exercise-02-fedavg-with-flower-end-to-end.md)
3. [NVIDIA FLARE cross-silo training](exercises/exercise-03-nvflare-cross-silo-training.md)
4. [Bonawitz secure aggregation implementation](exercises/exercise-04-bonawitz-secure-aggregation-implementation.md)
5. [DP-FL — record vs. user vs. client DP](exercises/exercise-05-dp-fl-record-vs-user-vs-client-dp.md)
6. [Gradient-reconstruction attack and defence](exercises/exercise-06-gradient-reconstruction-attack-and-defence.md)

## Structure

- `01-…md` … `08-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs (scaffolded placeholder).
- `quizzes/`: knowledge checks (scaffolded placeholder).
- `resources.md`: external references — Kairouz 2021 survey, McMahan FedAvg, Li FedProx, Bonawitz SA, Zhu DLG, Geiping Inverting Gradients, Nasr multi-round MIA, GDPR / EDPB / AI Act, library documentation.

## Where this module sits in the track

- Upstream dependencies: mod-101 (frameworks and position — why FL is a privacy-engineer tool), mod-102 (threat modelling — the per-actor FL threat model), mod-105 (memorisation and MIA — the empirical corroborator for what the FL-DP claim bounds), mod-106 (differential privacy engineering — the accountant choice and release-package pattern this module extends).
- Downstream consumers: mod-108 (confidential computing — the TEE-attested aggregator as a complementary FL control), mod-110 (unlearning — subject-rights erasure in a federated setting), mod-111 (DPIA — quotes this module's release package and data-flow diagram), mod-112 (program and communication — the regulator-facing narrative for the FL system).
- Peer boundary: `ai-infra-mlops` (level 25) — platform-engineering depth on the FL stack. See chapter 08.
