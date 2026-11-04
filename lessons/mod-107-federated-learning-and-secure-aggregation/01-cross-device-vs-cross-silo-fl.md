# Cross-Device vs. Cross-Silo Federated Learning — Which Production Shape Fits Which Business Problem

Federated learning (FL) is "train on data that stays where it was collected". The one-sentence version is a marketing sentence; the engineering reality is that FL comes in two distinct production shapes, and the privacy, networking, trust, and operational problems you will have to solve are not the same across the two. Treating them as the same shape is the first and most expensive design mistake. Before any Flower code, any NVIDIA FLARE deployment, any secure-aggregation protocol, you need the taxonomy that the rest of the module assumes.

The two shapes have standard names. *Cross-device FL* is the Google-Keyboard archetype — millions of edge devices, intermittent, non-IID, mobile phones or embedded sensors. *Cross-silo FL* is the hospital-consortium archetype — a small number of institutional parties (hospitals, banks, pharma companies), stable servers, IID-ish partitioning, legal contracts underpinning the collaboration. The McMahan et al. 2017 FedAvg paper launched the cross-device framing; the subsequent industry deployments (NVIDIA Clara/FLARE, Owkin, Melloddy, FedML Industrial) built the cross-silo stack. The Kairouz et al. 2021 *Advances and Open Problems in Federated Learning* survey is the canonical reference that formalised the split.

Primary references — read them in this order:

- McMahan, Moore, Ramage, Hampson, Agüera y Arcas — *Communication-Efficient Learning of Deep Networks from Decentralized Data*. AISTATS 2017. [arxiv.org/abs/1602.05629](https://arxiv.org/abs/1602.05629) — the FedAvg paper; introduces the cross-device setting as a first-class production shape.
- Kairouz, McMahan, et al. — *Advances and Open Problems in Federated Learning*. Foundations and Trends in Machine Learning, 2021. [arxiv.org/abs/1912.04977](https://arxiv.org/abs/1912.04977) — the 100-author canonical survey. Section 2.1 formalises the cross-device vs. cross-silo split; the whole document is the module's standing reference.
- Bonawitz, Eichner, et al. — *Towards Federated Learning at Scale: System Design*. MLSys 2019. [arxiv.org/abs/1902.01046](https://arxiv.org/abs/1902.01046) — Google's description of the production cross-device system (Gboard). Read for the operational reality of running FL at the millions-of-clients scale.
- Rieke, Hancox, et al. — *The Future of Digital Health with Federated Learning*. npj Digital Medicine, 2020. [nature.com/articles/s41746-020-00323-1](https://www.nature.com/articles/s41746-020-00323-1) — the cross-silo medical-imaging reference; the business framing for institutional FL.

## The two shapes, side by side

| Dimension | Cross-device | Cross-silo |
| --- | --- | --- |
| Number of clients | 10⁶ – 10⁹ | 2 – ~100 |
| Client type | Mobile phones, IoT, edge devices | Hospitals, banks, pharma, institutions |
| Availability | Intermittent (battery, Wi-Fi, idle, charging) | ~100% server uptime, scheduled windows |
| Data partitioning | Non-IID, user-specific, often tiny per device | Larger shards, closer to IID or IID-per-site |
| Client identifiability | Pseudonymous (install ID, hashed device ID) | Named legal entities with contracts |
| Lawful basis per client | Consent (click-through, privacy policy) | Contract / joint-controller agreement |
| Trust model | Clients untrusted; server operates infra | Clients semi-trusted; server often a neutral orchestrator |
| Privacy unit | The user / device (user-level DP) | The institution's dataset (silo-level) and its subjects (record / user-level) |
| Communication constraint | Severe — upload budget is the bottleneck | Modest — datacentre-to-datacentre links |
| Round cadence | Many rounds (10³ – 10⁵+), small slice per round | Few rounds (10 – 10³), many clients each round |
| Client selection | Server picks ~0.1% of eligible clients per round | All or almost-all silos every round |
| Dropout rate per round | High (30-70%) | Low (near zero) |
| Audit footprint per client | Minimal — thin client, tamper-resistant where possible | Full — each silo runs an auditable server |
| Typical governance regime | Product T&Cs + consumer-privacy regulators (GDPR, CCPA) | Multi-party DPA / joint-controller agreement; sectoral rules (HIPAA, PSD2) |

Nothing on this table is optional. The business problem and the production shape are joined at the hip. If a team tells you they are "building federated learning" without first telling you which shape, your first question is which shape. The follow-up design conversations differ in almost every dimension.

## Cross-device: the archetype and the engineering reality

The archetype is Google's Gboard — the keyboard that learns the next-word distribution from text you have typed on-device. The data never leaves the phone as text. Instead, each selected device computes a gradient update against the locally-cached training examples, uploads the update, and the server averages a few thousand such updates into a global model, round after round, for weeks. The current serving model is pushed back to devices on the next check-in. Apple runs the same shape for the on-device models that power QuickType, Siri personalisation, and (as of recent iOS releases) parts of Apple Intelligence; Samsung and Xiaomi run it for keyboards and app recommenders; health wearables (Fitbit, Oura's clinical research arm) and smart-home vendors run it for activity and anomaly detection.

Four engineering realities define the cross-device shape, and they ripple into every downstream design decision in this module.

- **Intermittency is the default.** A device joins the round when it is (a) idle, (b) on Wi-Fi, (c) plugged in, and (d) picked by the server. Those four predicates reduce the eligible population by ~100× in practice. The round finishes when either a target number of participating devices has reported their updates (e.g. 100–5000) or a timeout elapses. Devices that have not yet reported are dropped. The secure-aggregation protocol (chapter 04) must tolerate **dropout rates of 30–70% within a single round**; the Bonawitz paper calibrates its Shamir-secret-sharing parameters directly against that reality.
- **Non-IID is not an academic footnote.** The empirical distribution of one user's keyboard inputs — language mix, frequency, lexicon, typing cadence, emoji ratio — is radically different from the population's. FedAvg's straight-average aggregation of client updates assumes the per-client gradients are unbiased noisy estimates of the population gradient; under extreme non-IID they are not. FedProx (chapter 02), SCAFFOLD, MIME, and FedNova exist specifically to patch that assumption at scale.
- **Per-round slice is tiny.** A typical production round samples 100 – 5 000 devices out of ~10⁸ eligible. The sampling rate is `q ≈ 10⁻⁵` to `10⁻⁴`. That is far smaller than any DP-SGD sampling rate — and the resulting privacy amplification by subsampling is a central privacy lever (chapter 05). But it also means the *effective sample size per round* is small enough that single-device outliers can skew the aggregate without careful clipping.
- **The client is untrusted.** A user's phone is not a trusted execution environment. The server operator must assume some clients are compromised, rooted, running an instrumented FL stack, or outright adversarial. The threat model for the SA protocol assumes the server is *honest-but-curious* (not actively malicious) and that a bounded fraction of clients may be malicious or dropping. Byzantine-robust aggregation (Krum, median, trimmed-mean, FoolsGold — all referenced in Kairouz 2021 §6) is an orthogonal class of defence this module names but does not own; see chapter 06 for the boundary.

**Which business problems fit.** Cross-device is correct when (a) the data is generated and held on consumer devices, (b) the volume per device is small but the population is enormous, (c) upload cost of raw data would be prohibitive or regulatorily impossible, and (d) a slightly-stale global model is acceptable because the system re-trains continuously. Keyboard next-word, speech-recognition personalisation, on-device recommender personalisation, mobile camera / vision personalisation, wearable anomaly detection — all fit. FL on 1 000 hospitals' CT scans does not fit this shape, even though each hospital holds data locally.

## Cross-silo: the archetype and the engineering reality

The archetype is a medical-imaging consortium — a handful of hospitals (3 – 30 typically) that each hold a few thousand to a few million labelled studies behind their own firewalls. The clinical PHI never crosses the hospital's network boundary; instead, each hospital runs a local training job, uploads the model-weight update to a neutral aggregator (a trusted third party or a rotating role), and the aggregator produces an updated global model. The data-sharing agreement, not product T&Cs, is the governance substrate. NVIDIA FLARE's deployments at Mass General Brigham / Cleveland Clinic / King's College London for COVID-19 chest-X-ray classification, Owkin's MELLODDY consortium with ten pharma companies (AstraZeneca, Bayer, Boehringer, GSK, Janssen, Novartis, Merck KGaA, Servier, Sanofi, Amgen) training on each company's proprietary ADMET compound libraries, and the Featurecloud / FeatureCloud oncology projects are the published cross-silo exemplars.

Four engineering realities define the cross-silo shape.

- **Availability is scheduled, not opportunistic.** Each silo runs a server-grade participant. Rounds are planned; failure rates per round are low; a dropped silo is incident-reviewed, not shrugged off. The SA protocol you ship can tolerate ~0–10% dropout; if you ship Bonawitz-SA here, the Shamir-threshold parameters tune very differently from cross-device.
- **IID-ish, but with site-level distribution shift.** Each hospital's imaging distribution reflects its vendor (Siemens vs. GE scanners), its protocols, its patient demographics, its disease prevalence. The distribution is *not* IID across sites, but each site's shard is much larger than a cross-device client's. The practical fix is usually *FedProx or FedAvg with careful normalisation per site*, and in many consortia it is enough; more aggressive personalisation layers (FedPer, Ditto) are the escalation.
- **The privacy unit is the silo and the subject.** Two concurrent privacy layers apply. At the silo level, the data-sharing agreement defines what the aggregator is allowed to learn about the silo (ideally nothing beyond the trained model). At the subject level, each hospital is still a GDPR / HIPAA controller for its patients, and the subject's data must be protected against reconstruction via the gradient updates it influences. Chapter 05 ties this to the record / user / client DP-FL granularity choice.
- **Aggregator-trust is contractually bounded.** Unlike cross-device, where the aggregator is a product owner, cross-silo aggregators are often neutral — a hosting academic group, a regulator-appointed trustee, an escrow service, or a rotating role across the silos. The legal structure (joint controllership, processor-controller, or peer-to-peer) determines which aggregator-protection mechanisms are mandatory. Chapter 07 is where the governance implications land.

**Which business problems fit.** Cross-silo is correct when (a) the data is held by a small number of named organisations, (b) each silo has substantial data volume that it is legally prohibited from sharing in raw form, (c) the collaboration has a business rationale (a shared research consortium, a pooled risk model, a drug-discovery joint venture) with a signed agreement, and (d) the latency and cost of a coordinated round every day/week/month is operationally feasible. Multi-hospital medical imaging, inter-bank fraud detection, inter-insurer actuarial modelling, pharma ADMET prediction, multi-manufacturer defect detection in semiconductor yield — all fit. Keyboard next-word on consumer phones does not.

## The hybrid "cross-device but limited to a vetted fleet"

Two real-world shapes blur the boundary and are worth naming so you recognise them in design docs.

- **Enterprise-device FL.** A single large enterprise runs FL across its own employee-issued laptops or fleet-managed devices. Thousands of clients (closer to cross-device than cross-silo in scale), but the fleet is legally and operationally controlled by one entity, and the clients are vetted corporate endpoints running an IT-managed OS image. Governance is simpler than cross-device (no consumer consent surface) but the engineering problems (intermittency, non-IID per employee, dropout) are still cross-device-flavoured.
- **Wearable + hospital hybrid.** The classical shape for clinical digital-health studies — a cross-device layer that collects wearable sensor data from study participants, personalised on-device, then a cross-silo layer where participating study sites (hospitals, labs) jointly train a research model. Each layer carries its own privacy design; the module treats them as two separate FL systems chained via a documented aggregator.

Design these as composed-but-separate systems. The temptation is to run one FL stack across both layers. Don't — the threat models and governance regimes do not combine cleanly.

## Decision checklist — which shape, by the question the business is asking

Use this as a design-review artefact. Run it on day zero of any FL project proposal.

- **Q1 — How many participating endpoints?** Fewer than ~100 named entities: cross-silo. More than ~10 000 anonymous endpoints: cross-device. Between: the project is likely enterprise-fleet (cross-device-flavoured) or a federation of federations.
- **Q2 — What is a participant legally?** A named organisation bound by a contract: cross-silo. A natural person whose device executes a client: cross-device. If you cannot answer this cleanly, the lawful-basis question (chapter 07) has no answer yet.
- **Q3 — How much data does each participant hold?** Less than 10 MB and intermittent connectivity: cross-device. More than 100 GB and stable connectivity: cross-silo. In between, decide by availability, not volume.
- **Q4 — Who operates the aggregator?** A product team at a platform owner (Google, Apple, OEM): cross-device. A neutral or rotating third party agreed by contract: cross-silo. If no one has an answer yet, that is the first thing to decide; the privacy design depends on it.
- **Q5 — What is the privacy unit you need to protect?** A user's on-device history: user-level DP across all of that user's rounds. An institution's entire dataset: silo-level + per-subject. See chapter 05 for the mechanical consequences.
- **Q6 — What is the round cadence the business needs?** Daily or faster re-training against a drifting population: cross-device (continuous round cadence, wide client pool). Weekly to monthly research-grade iterations: cross-silo (scheduled rounds, all silos each round).

If any answer is ambiguous, that is a design-document gap, not a flexibility feature. Push back to the product manager, DPO, or legal counsel until the ambiguity resolves.

## Common anti-patterns, by shape

These are the design mistakes that recur enough to be worth naming.

- **"Cross-device, with each user explicitly identified and logged".** If the server-side observability is per-user with the user's identity, you have spent the engineering cost of FL without getting the privacy benefit. The point of cross-device FL is that the aggregator sees only aggregates. If you need per-user telemetry, you are shipping a differently-architected system.
- **"Cross-silo, but treat each hospital's dataset as public to the other hospitals".** A cross-silo collaboration is between controllers who do *not* get to see each other's data — the aggregator must enforce that invariant through the SA protocol or the trust model. A design that lets hospital A see hospital B's gradient update in the clear defeats the collaboration's legal predicate.
- **"Cross-device with two clients per round".** Flower will let you train in this configuration. Nothing stops you. But two-client FedAvg is a thin wrapper over distributed SGD without any of the privacy amplification by subsampling, and the per-round noise structure is terrible for secure aggregation. If you cannot reach the ~50-client-per-round minimum, you are prototyping a cross-silo system in miniature — plan accordingly.
- **"Cross-silo without an aggregator-trust story".** "The aggregator is honest" is a *conclusion*; the question is *what makes it honest*. Contractual obligation, HSM-anchored keying, running in a TEE (mod-108), or rotating-third-party role — pick at least one. A design that silently assumes a benevolent aggregator cannot survive the first security review.
- **"We will decide later whether this is cross-device or cross-silo".** The two systems share the FL vocabulary and very little else. "Later" means you will pay the migration cost in production. Decide now.

## What the rest of the module assumes about the shape split

The chapters that follow do not re-derive the split. Each chapter makes its applicability explicit:

- **Chapter 02 (FedAvg / FedProx with Flower).** Mechanics applicable to both shapes; the examples are cross-silo for ease of running locally; the client-selection strategy is where the shape actually matters at runtime.
- **Chapter 03 (TFF / FLARE / PySyft).** TensorFlow Federated is historically Google's cross-device simulation stack; NVIDIA FLARE is the mainstream cross-silo production stack; PySyft bridges both with a stronger research / cryptographic focus. The library you pick is often the strongest signal for which shape you have decided you are building.
- **Chapter 04 (Bonawitz secure aggregation).** Exactly the protocol designed for the *cross-device* threat model (honest-but-curious server, high dropout, untrusted clients). We ship the protocol and then explicitly contrast when it is overkill for cross-silo.
- **Chapter 05 (DP-FL granularity).** Record-level DP is a cross-silo default when each silo pools many users' records; user-level DP is a cross-device default when the user is the privacy unit; client-level DP is the cross-silo answer when the silo itself is the unit protected. The granularity choice *is* a shape choice.
- **Chapter 06 (FL-specific attacks).** DLG/iDLG are catastrophic in cross-silo without SA; they are rate-limited by sampling in cross-device but still viable. Cross-round MIA is a cross-silo-flavoured concern (stable participation); label inference hits both.
- **Chapter 07 (FL + GDPR governance).** Cross-device vs. cross-silo determines the lawful-basis story and the controller-processor-vs-joint-controller question. Different legal structure, different artefacts.

In every downstream chapter, any time a design choice is sensitive to the shape, we mark it explicitly: *"Cross-device: X. Cross-silo: Y."* Expect to see that framing throughout.

## Summary — what you can do after this chapter

- Name the two FL shapes, their clients, their aggregators, their availability profile, and their typical governance regime.
- State the McMahan / Kairouz / Bonawitz / Rieke reference anchors from memory.
- Walk the six-question decision checklist against a proposed FL project and defend the shape conclusion in a design review.
- Recognise and name the four common anti-patterns in design docs.
- Articulate which engineering realities cascade from cross-device (intermittency, non-IID, tiny slice, untrusted client) and which from cross-silo (scheduled rounds, site-level shift, silo-as-unit, contractually bounded aggregator).
- Connect each downstream chapter to the shape it primarily serves.

Chapter 02 goes to code — FedAvg and FedProx under Flower, with the client-selection and heterogeneity knobs the production shape requires.
