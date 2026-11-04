# TensorFlow Federated, NVIDIA FLARE, and PySyft — the Production-Library Map

Chapter 02 used Flower because it is the cleanest teaching stack. A production FL project rarely stops at Flower. The three libraries worth knowing, by the production shape they serve best, are TensorFlow Federated (TFF), NVIDIA FLARE, and PySyft. Each has a different design centre, a different operational envelope, and a different natural match to cross-device or cross-silo. This chapter maps them — you are not implementing the whole deployment stack, but when a design doc says "we're going to use TFF" or "we're migrating from PySyft to FLARE", you need to know what that implies about the system architecture, the SA story, the DP-FL story, and the operational handoffs to `ai-infra-mlops` (chapter 08).

The library is not the system. Each of these is a *framework* that wraps the algorithms from chapter 02 with a client-server runtime, an aggregation protocol, a job-orchestration layer, and a privacy stack of varying completeness. Pick with care — migration between frameworks is expensive and the embedded assumptions differ.

Primary references:

- TensorFlow Federated — [www.tensorflow.org/federated](https://www.tensorflow.org/federated), source at [github.com/google-parfait/tensorflow-federated](https://github.com/google-parfait/tensorflow-federated), the `tff.learning` and `tff.aggregators` modules.
- NVIDIA FLARE — [nvflare.readthedocs.io](https://nvflare.readthedocs.io), source at [github.com/NVIDIA/NVFlare](https://github.com/NVIDIA/NVFlare), the "Federated Learning with NVIDIA FLARE" medical-imaging whitepapers.
- PySyft and OpenMined — [openmined.org](https://openmined.org), [github.com/OpenMined/PySyft](https://github.com/OpenMined/PySyft).
- Flower — [flower.ai/docs](https://flower.ai/docs) — covered end-to-end in chapter 02; included here for comparison.
- Kairouz et al. 2021 §9 — the Appendix inventorying FL frameworks (TFF, LEAF, FATE, FedML, PySyft) with their design axes.

## Design-axis cheat sheet

The four libraries differ on a small number of axes that cascade into everything downstream. The axes:

| Axis | Flower | TFF | NVIDIA FLARE | PySyft |
| --- | --- | --- | --- | --- |
| Primary target | General FL; teaching | Cross-device simulation & research; some production | Cross-silo production, especially healthcare | Research on privacy-preserving ML; MPC/HE emphasis |
| ML framework | Framework-agnostic | TensorFlow-centric | PyTorch-centric; TF supported | PyTorch; numpy for MPC |
| Programming model | Client + server Python classes | Functional, federated-computations as first-class values | Controller + executor + components, job-graph model | Duet pattern: data-owner vs. data-scientist Python processes |
| Simulation vs. real deployment | Both (first-class) | Primarily simulation; production "TFF-on-K8s" available | Primarily real deployment; simulator included | Both |
| Built-in Secure Aggregation | Experimental (`SecAgg+` strategy) | Yes, `tff.aggregators.secure_aggregator` | Yes, "Secure XGBoost" + add-on homomorphic encryption | Yes (via SyMPC / CrypTen bridge); research-grade |
| Built-in DP-FL | Opacus/TF-Privacy integration patterns, not native | Yes, `tff.aggregators.DifferentiallyPrivateFactory` | Yes, "FL + DP" tutorials; Opacus bridge | Partial; relies on external DP libraries |
| Governance tooling | Minimal | Minimal | Full admin console, provisioning, audit logging | Minimal |
| Typical scale | Up to ~10⁴ simulated / ~10² real | Up to ~10⁸ simulated / production via Google's stack | Up to ~10² real silos | Up to ~10¹ for MPC, 10³ for pure FL |
| License | Apache 2.0 | Apache 2.0 | Apache 2.0 | Apache 2.0 |

Internalise this table before reading the per-library sections. If a design doc proposes a mismatch — "we'll use TFF to coordinate 15 hospitals in production", say — the first question is whether the author is aware of the design axis they are swimming against.

## TensorFlow Federated (TFF)

TFF is Google's open-source analogue to the internal FL stack that runs Gboard. Its design centre is *federated computations as first-class values* — a `@tff.federated_computation` decorator turns a Python function into a strongly-typed federated program, which TFF can then compile to a local simulator or (in principle) a production runtime. The library's roots are in simulation-at-scale: the common use is running thousands of simulated clients inside one Python process to iterate on an algorithm before any production deployment.

What makes TFF worth knowing, beyond the simulation role, is that it ships the first-party productionisation of two primitives the rest of the ecosystem has to bolt on:

- `tff.aggregators.secure_aggregator` — the first-class Bonawitz-style SA aggregator, batteries-included. Chapter 04 refers to it as the production reference implementation.
- `tff.aggregators.DifferentiallyPrivateFactory` — the DP-FL aggregator with record / user / adaptive clipping, tied to the `dp_accounting` library. Chapter 05 shows how it maps to the granularity decision.

The TFF programming model has a learning curve. Instead of a client class and a server class (Flower's shape), TFF has `tff.learning.algorithms.build_weighted_fed_avg` (and friends) that construct a federated computation out of composable *process* objects — `aggregation_process`, `model_update_aggregation_factory`, `model_distributor`. The design is powerful; it is also unfamiliar to engineers used to the client/server split. The TFF tutorials (`Federated Learning for Image Classification`, `Building Your Own Federated Learning Algorithm`) are the canonical on-ramp.

**When to pick TFF.** Cross-device simulation at scale; research on new FL algorithms; any project that wants Google-grade reference implementations of SA and DP-FL out of the box. A good match for "we want to prototype the algorithm and later port to an internal production stack" workflows.

**When not to.** Multi-institutional cross-silo medical FL where you also need a provisioning console, audit log, and deployed server infrastructure — FLARE is the stack designed for that. Pure PyTorch shops will find TFF's TF-centrism costly. TFF's production-deployment story is weaker than its simulation story; running thousands of real clients against a TFF-backed service requires Google-flavoured infrastructure (K8s + gRPC + Google's streaming aggregator) that is possible but not out-of-the-box.

**A conceptual slice of TFF code:**

```python
import tensorflow_federated as tff
import tensorflow as tf


def model_fn():
    # A Keras model wrapped to satisfy TFF's expected interface.
    keras_model = make_keras_mlp()
    return tff.learning.models.from_keras_model(
        keras_model,
        input_spec=train_spec,
        loss=tf.keras.losses.SparseCategoricalCrossentropy(),
        metrics=[tf.keras.metrics.SparseCategoricalAccuracy()],
    )


# Build the DP-aggregated FedAvg process.
dp_aggregator = tff.learning.dp_aggregator(
    noise_multiplier=1.1,
    clients_per_round=100,
    expected_clients_per_round=100,
    target_unclipped_quantile=0.5,     # adaptive clipping
    clipped_count_stddev=0.1,
)

training_process = tff.learning.algorithms.build_weighted_fed_avg(
    model_fn=model_fn,
    client_optimizer_fn=lambda: tf.keras.optimizers.SGD(learning_rate=0.02),
    server_optimizer_fn=lambda: tf.keras.optimizers.SGD(learning_rate=1.0),
    model_aggregator=dp_aggregator,
)

state = training_process.initialize()
for round_num in range(100):
    sampled_clients = sample(client_data, num_clients=100)
    state, metrics = training_process.next(state, sampled_clients)
```

Note how DP is a property of the *aggregator*, not of the client's optimiser. That is TFF's architectural assertion: DP-FL is about how the server combines updates, not about how the client trains. Chapter 05 unpacks the record-vs-user-vs-client implication.

## NVIDIA FLARE

NVIDIA FLARE (Federated Learning Application Runtime Environment) is the mainstream cross-silo production stack. Its design centre is *multi-institutional deployment* — the framework ships a provisioning console (define participating sites with public-key infrastructure), an administrative server, encrypted site-to-server communication, a job graph (controller → executor → components), and a full operational audit log. The healthcare whitepapers published by NVIDIA with partners (Mass General Brigham, Cleveland Clinic, King's College London, NIH, Stanford) are built on FLARE.

Four things make FLARE the right pick for cross-silo:

- **Provisioning is first-class.** You run a provisioning CLI, generate a startup kit per site (certificates, config, scripts), and hand each institution a reproducible deployment. Nobody is cobbling together ad-hoc gRPC configs.
- **The job model is operational.** A FLARE "job" is a declarative YAML / Python description of controllers, executors, filters, and components. Running a training job is "submit the job to the admin server", which then orchestrates it across sites. Reproducibility is the default.
- **Built-in privacy primitives.** `nvflare.app_opt.pt.differential_privacy` (DP-FL), `nvflare.app_common.workflows.sag_optional_dp` (secure aggregation with optional DP), homomorphic-encryption-based secure aggregation via the `HE` module. These are the "we need to run this in production tomorrow" building blocks.
- **Admin + audit console.** Every round is logged on the admin server. Each site has its own audit log. For a regulated cross-silo collaboration, that log is a non-negotiable artefact.

**When to pick FLARE.** Cross-silo medical imaging, cross-silo pharma / finance, enterprise-FL inside a large corporation across sub-business-units, any FL project where a provisioning and audit story matters as much as the training algorithm.

**When not to.** Cross-device simulation at 10⁶ clients — FLARE is not designed for that density. Research projects where the operational scaffolding is overhead. Pure Google-ecosystem shops that prefer TFF.

**The shape of FLARE code:**

```python
# A FLARE controller (server-side). Simplified.
from nvflare.apis.impl.controller import Controller
from nvflare.app_common.workflows.fedavg import FedAvg
from nvflare.app_common.aggregators.dp_fedavg_aggregator import DPFedAvgAggregator

# A FLARE job is normally expressed via a job-config YAML/JSON that references
# component class paths. The controller orchestrates per-round broadcast and
# aggregation; the aggregator does the privacy-sensitive combining.

class MyFedJob:
    def __init__(self):
        self.controller = FedAvg(
            min_clients=5,
            num_rounds=50,
            aggregator_id="dp_aggregator",
        )
        self.aggregator = DPFedAvgAggregator(
            noise_multiplier=1.0,
            clip_norm=1.0,
            dp_mode="client_level",   # see chapter 05
        )
```

On the executor (client) side, FLARE provides a `ScriptRunner` or `Executor` that wraps your training function; much like Flower, you write a local-training step and FLARE handles the round orchestration.

Two FLARE-specific terms worth knowing because they recur in cross-silo design docs:

- **Startup kit.** The per-site provisioning bundle (certificates, scripts, config). The security story depends on each site keeping its startup kit confidential — a leaked kit is a site compromise.
- **Filter.** A pluggable transform that runs at the server or client before / after an operation. DP noise addition, SA mask application, model-input sanitisation, and privacy-leakage inspection are all expressed as filters. The filter pattern is where you insert the mod-107 privacy hooks.

## PySyft and the OpenMined ecosystem

PySyft, from the OpenMined community, is the research-first library with the strongest cryptography story. Its design centre is *privacy-preserving computation* broadly — FL is one application; secure multi-party computation (via SyMPC), homomorphic encryption (via TenSEAL), and private set intersection are first-class. The "duet" pattern — a data-owner Python process and a data-scientist Python process sharing tensors via a secure channel — is PySyft's distinctive abstraction.

PySyft is where you go if the research question is "can we run this computation across parties without any one party seeing the others' data in the clear?" rather than "can we train a model with FedAvg?". Its SA implementations are closer to the cryptographic literature than to Bonawitz; its DP story depends on external DP libraries. The implementation has gone through several architectural generations (v0.2, v0.4, v0.8), and production stability has historically lagged academic claims. The project remains the main open-source hub for MPC-first FL research.

**When to pick PySyft.** Research on MPC-heavy architectures, PSI (private set intersection) between two or three parties, teaching the cryptographic primitives behind FL, projects where the budget is "one FL module in a broader privacy-preserving system".

**When not to.** Any cross-device or cross-silo production deployment where operational stability dominates algorithmic novelty. Training a BERT-scale model in federated fashion across silos. Any setting where the audit log, provisioning console, and admin UX matter.

## The honourable-mention map

A larger map, for when you see these names in design docs or vendor pitches:

- **FATE (Federated AI Technology Enabler)** — WeBank-led, strong in Chinese finance / insurance deployments, mature feature set, Java-flavoured runtime, used in the EU AI Act compliance-adjacent industrial-finance world. ([fate.fedai.org](https://fate.fedai.org)).
- **FedML / TensorOpera** — Commercial-flavoured research framework with a managed-service offering. ([fedml.ai](https://fedml.ai)).
- **IBM Federated Learning (IBMFL)** — IBM Research's library, academic / industrial pilots, somewhat niche.
- **Substra (Owkin)** — The pharma-consortium stack behind MELLODDY; cross-silo-first, strong on data governance primitives. ([substra.ai](https://substra.ai)).
- **OpenFL (Intel + Linux Foundation)** — Enterprise cross-silo framework, SGX integration via Gramine, workgroup charter under the LF AI & Data umbrella.
- **FedScale / FedBench / LEAF** — Benchmarking harnesses, not production frameworks.
- **TensorFlow Federated's `tff.analytics`** — A separate sub-API for federated analytics (counts, histograms, heavy-hitters) rather than model training. Worth knowing for the mod-106-adjacent analytics case.

Mention any of these by their actual role; conflating them produces review-time confusion.

## Choosing the library — a decision tree

For a privacy-engineer-run project, the practical ordering:

1. **Is this a research or teaching project?** Pick Flower. The clean abstractions and framework-agnosticism make experimentation fast.
2. **Is this a cross-device simulation at scale (10⁴–10⁸ clients) with first-party SA and DP-FL?** Pick TFF. The primitives are there; the simulation tooling is first-class.
3. **Is this a cross-silo multi-institutional production deployment, especially healthcare / finance?** Pick NVIDIA FLARE. The provisioning + admin + audit story will save you months.
4. **Is this an MPC-forward research project or a privacy-preserving analytics system where FL is a component?** Pick PySyft or Substra.
5. **Is this an enterprise FL project inside one large organisation across business units?** OpenFL or FLARE both fit; the SGX integration path favours OpenFL, the healthcare-adjacent patterns favour FLARE.
6. **Is this an existing FATE / Substra / FedML deployment inheriting the choice?** Keep the existing stack; migration is expensive and buys you no privacy advantage.

If your team's answer is "we'll build it ourselves", call the question out explicitly in the design review. There are legitimate reasons to roll your own — specific security envelope, specific hardware environment, an existing internal RPC substrate — but each of them is a budget decision that is usually underestimated.

## What this means for the privacy engineer, operationally

- You own the **privacy primitives** the library exposes — the SA aggregator, the DP-FL aggregator, the filter chain, the audit-log writer. You verify they are configured correctly. You do not own the orchestration runtime; that belongs to `ai-infra-mlops` (chapter 08).
- You own the **threat-model note** per library. Each library's SA primitive makes different threat-model assumptions (honest-but-curious server? number of colluding clients? dropout tolerance?). The model in the release package is library-specific.
- You own the **attestation** that the library version is pinned, the privacy-relevant configuration is logged, and the library has not been monkey-patched to downgrade SA or DP. This is the FL-shaped equivalent of the mod-106 accountant-state-saved-on-checkpoint attestation.
- You consume the **library-level audit trail** that `ai-infra-mlops` produces (round logs, provisioning events, admin actions). You do not operate the admin console; you read from it.

The division of labour between you and `ai-infra-mlops` is explicit in chapter 08. This chapter gives you the vocabulary; chapter 08 gives you the handshake.

## Summary — what you can do after this chapter

- Name the four main FL libraries (Flower, TFF, FLARE, PySyft) and which production shape each primarily serves.
- Walk the design-axis table and use it to critique a proposed library choice.
- Explain why TFF is "simulation-first" and what that implies for a production deployment plan.
- Explain why NVIDIA FLARE is "cross-silo-first" and what the startup-kit / filter concepts contribute.
- Explain why PySyft is "research-first" and when its MPC-forward posture is the right trade-off.
- List five honourable-mention libraries by role (FATE, FedML, OpenFL, Substra, LEAF).
- Run the library-choice decision tree against a new project and defend the result.
- Articulate the privacy engineer's ownership boundary vs. `ai-infra-mlops`'s on top of whichever library is picked.

Chapter 04 is where the SA protocol shows up at the depth you need to defend it.
