# FedAvg and FedProx with Flower — the Round, the Client, and the Heterogeneity Knob

Chapter 01 taxonomised. This chapter codes. FedAvg is the baseline federated-training algorithm introduced by McMahan et al. 2017; FedProx (Li et al. 2020) is the first-line patch when client heterogeneity makes vanilla FedAvg diverge or oscillate. Flower ([flower.ai](https://flower.ai), [arxiv.org/abs/2007.14390](https://arxiv.org/abs/2007.14390), Beutel et al. 2020) is the library this chapter uses to make the abstractions concrete. Flower is framework-agnostic (PyTorch, TensorFlow, JAX, scikit-learn, Hugging Face), has a clean client / server split that maps directly to the algorithm's abstractions, and is the easiest stack to use for the exercises. It is not necessarily what you would use in the largest cross-device deployments — Google and Apple use internal stacks, and NVIDIA FLARE dominates production cross-silo — but it is the correct teaching tool.

Primary references:

- McMahan, Moore, Ramage, Hampson, Agüera y Arcas — *Communication-Efficient Learning of Deep Networks from Decentralized Data*. AISTATS 2017. [arxiv.org/abs/1602.05629](https://arxiv.org/abs/1602.05629) — the FedAvg paper.
- Li, Sahu, Zaheer, Sanjabi, Talwalkar, Smith — *Federated Optimization in Heterogeneous Networks*. MLSys 2020. [arxiv.org/abs/1812.06127](https://arxiv.org/abs/1812.06127) — FedProx.
- Beutel, Topal, et al. — *Flower: A Friendly Federated Learning Research Framework*. 2020. [arxiv.org/abs/2007.14390](https://arxiv.org/abs/2007.14390).
- Flower documentation — [flower.ai/docs/framework/](https://flower.ai/docs/framework/).
- Karimireddy, Kale, Mohri, Reddi, Stich, Suresh — *SCAFFOLD: Stochastic Controlled Averaging for Federated Learning*. ICML 2020. [arxiv.org/abs/1910.06378](https://arxiv.org/abs/1910.06378) — the next-rung variance-reduction algorithm, mentioned for the heterogeneity lever map.

## The FedAvg round, in one diagram

A single FedAvg round is four phases. Burn this into your mental model:

```
   SERVER                                 SELECTED CLIENTS (k of N)
   ------                                 ------------------------
1. Select k clients from eligible pool    (standing by)
2. Broadcast current global weights w_t  ──────────────────────▶
                                          3. Local training: w_t → w_t^i
                                             for E local epochs
                                             over this client's shard
   4. Aggregate updates:          ◀────── Upload Δ_i = w_t^i - w_t
      w_{t+1} = w_t + Σ (n_i / n) · Δ_i   (or upload w_t^i directly)
```

Four moving pieces. Each has a privacy-engineering consequence:

- **Phase 1 (client selection).** Which clients get picked each round shapes the DP sampling rate `q` (chapter 05). Non-uniform selection — picking only "high-quality" clients — breaks DP's uniform-sampling assumption and inflates the privacy guarantee.
- **Phase 2 (model broadcast).** The server broadcasts the global model in the clear. That weight checkpoint is public to every participating client and may leak information about training data from earlier rounds (chapter 06). If the model itself is treated as sensitive (e.g. proprietary architecture + weights), that becomes a confidentiality-of-model problem orthogonal to privacy-of-data.
- **Phase 3 (local training).** `E` local epochs of SGD on the client's shard. The per-client update `Δ_i` is what the gradient-leakage attacks of chapter 06 target.
- **Phase 4 (aggregation).** The server computes a weighted average of the updates. This is the step that secure aggregation (chapter 04) wraps so the server sees only the sum, not any individual `Δ_i`.

The FedAvg update rule is a weighted sum where the weights are the clients' sample counts `n_i` divided by the total sample count `n = Σ n_i` of the participating cohort. "Participating cohort", not "total population" — clients that dropped during the round are not in the denominator. That detail is where secure aggregation (chapter 04) and dropout-tolerance come in.

## When FedAvg breaks — and why FedProx exists

FedAvg's correctness proof (Stich 2018, Khaled-Mishchenko-Richtárik 2019 for the convex case; Li-Huang-Yang-Wang-Zhang 2019 for the non-convex case under smoothness) assumes two things that production FL violates.

- **Clients' local objectives are close to the global.** If `L_i(w)` is client `i`'s local loss and `L(w) = Σ p_i L_i(w)` is the population loss, FedAvg assumes `|L_i - L|` is bounded. Under non-IID data, that bound can be enormous. When it is, the local SGD update on client `i` drifts the weights toward `arg min L_i`, which may be far from `arg min L`.
- **System homogeneity.** Each client runs roughly the same number of local steps. In cross-device, battery / connectivity / CPU heterogeneity makes clients run *very* different amounts of local work before returning — some finish 1 epoch, some finish 10, some drop mid-training.

Under both violations, vanilla FedAvg can oscillate, converge slowly, or get stuck at a local minimum that looks right for the most common client distribution but is wrong for the population.

FedProx's patch: add a *proximal term* to each client's local loss, penalising deviation from the global model:

```
L_i^prox(w) = L_i(w) + (μ/2) · ||w - w_t||²
```

With `μ > 0`, each client's optimiser is pulled back toward the broadcasted global weights `w_t`. The result:

- **Robust to partial local work.** A client that only runs a fraction of its local steps still produces a meaningful `Δ_i` because the proximal term keeps the partial-step solution anchored.
- **Robust to non-IID data.** The client's local-minimum drift is damped by the proximal regulariser; the global aggregate is less distorted by heterogeneous local optima.
- **A single hyper-parameter.** `μ` is the one knob. Li et al. 2020 give practical guidance (`μ ∈ {1e-3, 1e-2, 1e-1, 1}`); the choice is a per-setup sweep. `μ = 0` recovers FedAvg exactly, so FedProx is strictly more general.

FedProx is the **first thing to try when FedAvg plateaus** on a non-IID or heterogeneous federation. It is not a universal solution; the heterogeneity lever map (below) is where SCAFFOLD, FedNova, MIME, and friends come in.

## A minimal Flower FedAvg example

Flower splits the world into a `Client` (runs on each participant) and a `Server` with a `Strategy` (runs on the aggregator). A minimal FedAvg trainer looks like this. The example is PyTorch; the same shape holds for TensorFlow / JAX by swapping the training primitives.

```python
# client.py — runs on each participating node
import flwr as fl
import torch
from torch.utils.data import DataLoader


class MnistClient(fl.client.NumPyClient):
    def __init__(self, model, train_loader: DataLoader, val_loader: DataLoader,
                 device: str = "cpu"):
        self.model = model
        self.train_loader = train_loader
        self.val_loader = val_loader
        self.device = device

    def get_parameters(self, config):
        # Convert torch state_dict to list[np.ndarray]; Flower serialises these.
        return [v.cpu().numpy() for v in self.model.state_dict().values()]

    def set_parameters(self, parameters):
        state_dict = {
            k: torch.tensor(v)
            for k, v in zip(self.model.state_dict().keys(), parameters)
        }
        self.model.load_state_dict(state_dict, strict=True)

    def fit(self, parameters, config):
        self.set_parameters(parameters)
        local_epochs = int(config.get("local_epochs", 1))
        lr = float(config.get("lr", 0.01))
        optimizer = torch.optim.SGD(self.model.parameters(), lr=lr)
        criterion = torch.nn.CrossEntropyLoss()
        self.model.train()
        for _ in range(local_epochs):
            for xb, yb in self.train_loader:
                xb, yb = xb.to(self.device), yb.to(self.device)
                optimizer.zero_grad()
                loss = criterion(self.model(xb), yb)
                loss.backward()
                optimizer.step()
        # Return the updated weights, the training-set size for aggregation
        # weighting, and any metrics you want the server to log.
        return self.get_parameters(config={}), len(self.train_loader.dataset), {}

    def evaluate(self, parameters, config):
        self.set_parameters(parameters)
        self.model.eval()
        criterion = torch.nn.CrossEntropyLoss()
        total, correct, total_loss = 0, 0, 0.0
        with torch.no_grad():
            for xb, yb in self.val_loader:
                xb, yb = xb.to(self.device), yb.to(self.device)
                logits = self.model(xb)
                total_loss += criterion(logits, yb).item() * yb.size(0)
                correct += (logits.argmax(1) == yb).sum().item()
                total += yb.size(0)
        return total_loss / total, total, {"accuracy": correct / total}
```

```python
# server.py — runs on the aggregator
import flwr as fl


def fit_config(server_round: int):
    # Config passed to each client's fit() call this round.
    return {"local_epochs": 1, "lr": 0.01}


strategy = fl.server.strategy.FedAvg(
    fraction_fit=0.3,              # pick 30% of eligible clients each round
    min_fit_clients=5,             # abort round if fewer than 5 respond
    min_available_clients=10,      # require 10 clients in the pool
    on_fit_config_fn=fit_config,   # hook for per-round hyper-parameters
)

fl.server.start_server(
    server_address="0.0.0.0:8080",
    config=fl.server.ServerConfig(num_rounds=50),
    strategy=strategy,
)
```

Three things to internalise from the code shape:

- **The client's `fit` return tuple is (weights, example_count, metrics).** The `example_count` is what the server uses to compute the sample-size-weighted average in Phase 4. Lying about it (reporting a bigger `n_i` than you actually trained on) is a classical attack — see chapter 06's note on *weight poisoning*.
- **The server never touches raw training data.** Flower's abstractions enforce that. The server sees weights and scalar metrics; nothing else. That is the invariant you lean on when you describe the system to a DPO.
- **`strategy` is where the aggregation happens.** Flower ships `FedAvg`, `FedAvgM`, `FedProx`, `FedAdam`, `FedYogi`, `QFedAvg`, `FedTrimmedAvg`, `Krum`, `Bulyan`, and more. Swap the strategy and you swap the aggregation algorithm without changing the clients.

## Switching to FedProx in Flower

FedProx has two pieces: a **strategy change** on the server (so the per-round client config carries `proximal_mu`) and a **client change** that adds the proximal term to the local loss. Flower ships the server side as `fl.server.strategy.FedProx`; on the client you implement the regulariser yourself.

```python
# server.py — swap FedAvg for FedProx
strategy = fl.server.strategy.FedProx(
    fraction_fit=0.3,
    min_fit_clients=5,
    min_available_clients=10,
    on_fit_config_fn=fit_config,
    proximal_mu=0.01,              # the one hyper-parameter
)
```

```python
# client.py — add the proximal term inside fit()
def fit(self, parameters, config):
    self.set_parameters(parameters)
    # Snapshot the global weights BEFORE local training; we regularise toward
    # these values across every step of the local update.
    global_params = [torch.tensor(p, device=self.device) for p in parameters]
    proximal_mu = float(config.get("proximal_mu", 0.0))

    optimizer = torch.optim.SGD(self.model.parameters(),
                                lr=float(config.get("lr", 0.01)))
    criterion = torch.nn.CrossEntropyLoss()
    self.model.train()
    for _ in range(int(config.get("local_epochs", 1))):
        for xb, yb in self.train_loader:
            xb, yb = xb.to(self.device), yb.to(self.device)
            optimizer.zero_grad()
            loss = criterion(self.model(xb), yb)
            if proximal_mu > 0.0:
                prox = sum(
                    ((p - gp) ** 2).sum()
                    for p, gp in zip(self.model.parameters(), global_params)
                )
                loss = loss + (proximal_mu / 2.0) * prox
            loss.backward()
            optimizer.step()
    return self.get_parameters(config={}), len(self.train_loader.dataset), {}
```

Two engineering foot-guns to watch for:

- **Snapshot the global parameters once per `fit()` call.** The proximal term regularises toward the *round-start* global model, not the current model-being-trained. A common bug is to compute the proximal term against `self.model.parameters()` (the live, locally-updating copy), which collapses the term to zero. The snapshot is what makes FedProx FedProx.
- **Do not include frozen / BN-stat parameters in the proximal sum.** Batch-normalisation running statistics are not trained via gradient descent; penalising their drift against the round-start values is at best meaningless and at worst inhibits legitimate adaptation. The bug is silent — the training does not crash, it just gets slower. Match the parameter iteration to `.parameters()` (trainable only), not `.state_dict()`.

## The client-selection strategy — the knob that's secretly a privacy decision

Flower exposes client selection as the `configure_fit` method of a `Strategy`. The default is uniform random sampling across available clients. Non-uniform strategies ("pick the clients with the lowest current validation loss", "pick clients who have not participated recently to encourage coverage", "pick clients whose uploaded update sizes last round were above a threshold") are tempting as optimisation levers. Each one has a privacy consequence.

- **Uniform random selection.** The clean case. The DP accountant for DP-FL (chapter 05) is calibrated for Poisson sampling; uniform-with-replacement is close enough that production accountants will accept it. The sampling rate `q = k / N` for `k` picked out of `N` available is the amplification parameter you quote.
- **"Oracle" or score-conditioned selection.** You pick clients based on something the server has learned from them. This *breaks* the DP amplification argument because the sampling is not independent of the data; privacy accountants that assume uniform sampling will be *wrong* (silently anti-conservative). Chapter 05 treats this as a hard rule: if the selection is data-dependent, the DP claim must be re-derived.
- **Coverage-stratified selection.** Divide clients into strata (by locale, by device tier, by join date) and sample uniformly within each. This can be DP-safe if the strata are a *fixed* partitioning not derived from training behaviour. The accountant adjusts to the per-stratum rate.
- **Participation-capped selection.** A cross-device best practice: each client may participate in at most `M` rounds over the lifetime of the training run. This bounds a single user's total exposure for user-level DP composition (chapter 05). Flower does not enforce this out of the box; you own the cap in a stateful strategy.

**Rule of thumb.** If a product manager or a PM-adjacent-ML engineer asks for a "smarter" client selection algorithm to speed up convergence, the first question is "does the algorithm use any signal derived from client data?". If yes, the DP accountant has to change, and that conversation involves the privacy engineer (you).

## The heterogeneity lever map

FedProx is the simplest patch. Several other algorithms exist and the choice is a trade-off of complexity, stability, and the specific flavour of heterogeneity you face. The map:

| Algorithm | What it adds | Costs | When to pick |
| --- | --- | --- | --- |
| **FedAvg** | Baseline, nothing | — | Low heterogeneity, same-sized clients, you are prototyping |
| **FedProx** | Proximal term `μ‖w − w_t‖²` per client | One hyper-parameter | First upgrade when FedAvg oscillates or diverges |
| **FedAvgM** | Server-side momentum | Server-side state across rounds | Modest extra convergence stability; simple to add |
| **SCAFFOLD** | Client + server control variates to correct drift | Doubles communication per round; stateful clients | Severe non-IID; you can afford the comm cost |
| **FedNova** | Normalise updates by effective number of local steps | Requires knowing each client's step count | System-heterogeneous clients with widely different local work |
| **FedAdam / FedYogi / FedAdagrad** | Server-side adaptive optimisers | More hyper-parameters on the server | Large-batch-flavoured convergence behaviour |
| **MIME** | Server-side momentum + per-client reset | Doubles communication; complicated | Research setting; rarely a first choice |
| **Ditto / FedPer / FedRep** | Per-client personalisation head | Not a global-model algorithm | When each client needs its own model layer |

The pragmatic ordering for a privacy-engineer-run project: start FedAvg, escalate to FedProx when you see oscillation, escalate to FedAvgM or FedAdam if the plateau is at the server side, and only reach for SCAFFOLD / FedNova when you have evidence the heterogeneity is severe enough to justify the operational complexity. Every extra knob is one more thing to defend in a code review and one more thing to break DP's clean sampling story.

## Metrics you should log every round

The server-side logs are the forensic record for everything downstream — DP accounting, incident review, attack reproduction. Log at least the following per round, and persist them in a way the audit trail can quote:

- **Server metadata.** `round_number`, `strategy_name`, `strategy_hyperparameters`, `global_model_hash` (after aggregation).
- **Participation.** `clients_available`, `clients_selected`, `clients_reported`, `clients_dropped`. The selected / reported gap is the dropout rate and it drives the SA parameterisation (chapter 04) and the DP sampling-rate accounting (chapter 05).
- **Aggregation accounting.** `total_samples_in_round`, `per-client reported sample counts` (hashed or bucketed, not raw, for cross-device), the aggregation weights used.
- **Metrics.** Per-round training-loss and validation-metric summaries from clients that reported. Distributional, not raw-string — min / mean / max, or a histogram.
- **Timings.** `round_duration`, `aggregation_duration`. Spiky aggregation durations are an early indicator of the SA protocol scaling with client count faster than expected.
- **DP state (if DP-FL is enabled).** The DP accountant's current `(ε, δ)` after the round — see chapter 05.

Push the logs into whatever your mod-106 release-package audit log mechanism is. The FL round log is a first-class artefact in the mod-107 release package (the equivalent of the DP-SGD accountant audit log from mod-106).

## The "did we train with FedAvg correctly?" debugging checklist

Five recurring failure modes. If convergence looks wrong, walk this list before anything else.

- **Client-reported sample counts are wrong.** `n_i` should be the number of training examples the client actually used this round. Off-by-one errors (counting batches not examples, double-counting via data augmentation) produce biased aggregation weights. Verify against a known-size dataset first.
- **Batch normalisation on non-IID clients.** BN's running statistics are per-client-specific and the aggregated state is meaningless on the global model. Switch to GroupNorm or LayerNorm, or exempt the BN stats from aggregation (`fl.common.NDArrays` cherry-picking). This is the same bug as the mod-106 BN-vs-DP-SGD foot-gun, differently shaped.
- **Local optimiser state is carried across rounds.** `torch.optim.SGD` with momentum stores per-parameter buffers. If you re-instantiate the client's optimiser each round (the usual pattern) the buffers are reset to zero — which may or may not be what you want. Document the convention.
- **The server's initial weights are random per server restart.** If you restart the server mid-training, you lose state. Checkpoint the server's global weights *and* the strategy's internal state, including DP accountant state, every round.
- **Clients are picked before they are "ready".** `min_available_clients` is the pool, not the responding set. A round that selects clients which then fail to produce an update inflates the dropout rate. Watch the `clients_selected` / `clients_reported` ratio and tune `fraction_fit` so that the responding set hits your target.

## What this chapter does and does not do

This chapter is enough to get a FedAvg / FedProx training run working on Flower against a simulated federation of 10–1000 clients (the cross-silo end of the scale). It is not enough to run a Gboard-scale cross-device deployment — for that, you want the industrial stack discussed in chapter 03 (NVIDIA FLARE for cross-silo; TFF or an internal framework for cross-device). And none of this chapter is yet *private*; the server sees every individual update. Chapter 04 (secure aggregation) wraps the aggregation step so the server sees only the sum; chapter 05 (DP-FL) injects noise into the updates themselves. Both chapters assume you already own the FedAvg / FedProx loop from here.

## Summary — what you can do after this chapter

- Draw the FedAvg round as four phases and name the privacy-engineering consequence of each.
- Explain why FedAvg can fail under non-IID and system heterogeneity, and the FedProx proximal term's role in patching both.
- Stand up a Flower FedAvg trainer end-to-end (client + server + strategy) in PyTorch or TF.
- Switch to FedProx by changing both the server strategy and the client loss — and avoid the two silent bugs (live-vs-snapshot parameters; BN stats in the proximal sum).
- Reason about client-selection strategy as a privacy decision, not just a utility decision.
- Pick the right heterogeneity lever from the FedAvg → FedProx → FedAvgM → SCAFFOLD escalation map.
- List the five round-level metrics every FL system should log and connect them to downstream audit artefacts.
- Walk the five-item correctness checklist when a training run misbehaves.

Chapter 03 broadens the library picture — TensorFlow Federated, NVIDIA FLARE, PySyft — and the operational stakes that force the choice between them.
