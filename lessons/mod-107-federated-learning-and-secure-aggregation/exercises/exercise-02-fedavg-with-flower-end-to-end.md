# Exercise 02: FedAvg with Flower, End to End

**Estimated effort:** 3 hours · **Output:** `flower-fedavg/` directory with runnable Flower simulation, non-IID partitioning script, FedAvg → FedProx comparison, per-round audit log, and a `results.md` write-up · **Primary chapters:** 02, 01

## Objective

Stand up a working Flower federation end-to-end on a non-IID partitioning of a public dataset, run FedAvg and FedProx with a well-specified client-selection strategy, emit the per-round audit log fields the release package will consume, and compare FedAvg vs. FedProx convergence under non-IID. This is the foundational training loop for every subsequent FL exercise.

## Prerequisites

- Chapter 01 (shape decision) and 02 (FedAvg/FedProx) read.
- Python 3.10+.
- `pip install flwr[simulation] torch torchvision numpy pandas matplotlib`.
- Familiarity with PyTorch training loops.

## Problem statement

You will build a Flower simulation of a 20-client federation on either MNIST or CIFAR-10 (your choice — MNIST is faster, CIFAR-10 is closer to realistic). The federation will be non-IID; you will pick FedAvg as a baseline, run it to convergence (or 100 rounds, whichever comes first), then switch to FedProx with `μ ∈ {1e-3, 1e-2, 1e-1}` and compare. You will emit the per-round audit log fields from chapter 02's logging list.

### Required deliverables

#### `partition.py` — non-IID client partitioning

Partition the training set across 20 clients. Use the Dirichlet partitioning scheme from Yurochkin et al. 2019 / Hsu et al. 2019 ("Measuring the effects of non-identical data distribution for federated visual classification", [arxiv.org/abs/1909.06335](https://arxiv.org/abs/1909.06335)): for each class, draw a Dirichlet(α) distribution over clients, and allocate that class's examples proportionally. Use `α ∈ {0.1, 0.5, 100}` (small α → highly non-IID; large α → ~IID).

- Emit three partition files `partition_alpha_0.1.pkl`, `partition_alpha_0.5.pkl`, `partition_alpha_100.pkl`.
- For each partition, emit a one-plot visualisation `partition_alpha_*.png` showing per-client class histograms.

#### `client.py` — the Flower client

Implement a `fl.client.NumPyClient` (chapter 02). The client:

- Loads its shard from the partition file.
- Trains a small model (CNN for CIFAR-10 / MLP for MNIST) — your pick, document the choice.
- Supports both FedAvg (plain `fit`) and FedProx (adds proximal term). Switch via `config["proximal_mu"]`.
- Reports correct `example_count` and `{train_loss, val_accuracy}` metrics.

#### `server.py` — the Flower server

Use `fl.simulation.start_simulation` to run the 20-client federation locally. The server:

- Picks the `FedAvg` or `FedProx` strategy via a CLI flag.
- `fraction_fit = 0.5` (sample 10 of 20 clients per round).
- 50 rounds.
- On every round, writes a JSONL line to `audit_log.jsonl` with the fields from chapter 02's logging list (round number, clients selected, clients reported, aggregation weights summary, train-loss distribution, val metrics, round duration, global model hash).

#### Run the sweep

Run six training runs, 3 × α × {FedAvg, FedProx(μ=0.01)}:

| Run | α | Strategy |
| --- | --- | --- |
| 1 | 0.1 | FedAvg |
| 2 | 0.1 | FedProx μ=0.01 |
| 3 | 0.5 | FedAvg |
| 4 | 0.5 | FedProx μ=0.01 |
| 5 | 100 | FedAvg |
| 6 | 100 | FedProx μ=0.01 |

#### `plots.py` — convergence pictures

Produce:

- One line plot per α (3 total): val-accuracy vs. round, FedAvg and FedProx overlaid. Save as `plots/convergence_alpha_*.png`.
- One scatter plot: per-client reported sample count distribution per round, for α = 0.1. Save as `plots/client_sizes_alpha_0.1.png`.

#### `results.md` — the write-up

Up to one page covering:

- The partition configuration used and what α ≈ 0.1 vs. 100 mean operationally.
- The six runs' final val-accuracy, mean round time, and clients-reported rate.
- An observation on where (if anywhere) FedProx out-performs FedAvg. The expected result from Li et al. 2020 is FedProx helps at low α and is roughly neutral at high α — confirm or contradict against your data.
- A one-paragraph "the audit log satisfies chapter 02's requirements" attestation. If any field is missing, state why.
- A one-paragraph "what this would have to change to be production-grade" closure, referencing SA (chapter 04) and DP-FL (chapter 05).

## Starter guidance

- **Use `start_simulation` with ray, not `start_server` + 20 shell sessions.** Local simulation is the right teaching tool; real deployment is exercise 03's scope.
- **Keep the model small.** The point is the FL mechanics, not the state-of-the-art accuracy. A 100k-parameter CNN is plenty.
- **Set a seed.** Reproducibility is release-package discipline. Log the seed in the audit log.
- **The FedProx proximal term bug.** Chapter 02 warned that snapshotting the live parameters instead of the round-start parameters makes the proximal term zero. Verify by logging `mean(||w_current - w_global||)` over the local epoch — it should grow non-trivially if FedProx is wired correctly.
- **Non-IID partitioning is where most convergence problems come from.** If α=0.1 FedAvg shows catastrophic oscillation, that is the expected lesson. FedProx is supposed to damp it; measure how much.
- **Pin library versions.** Capture in a `requirements.txt` or `pyproject.toml`.

## Acceptance criteria

- [ ] Three partition files generated with visualisation plots that show the expected pattern (single-class dominance at α=0.1, uniform at α=100).
- [ ] `client.py` implements both FedAvg and FedProx (same class, config-switched), with the live-vs-snapshot bug avoided.
- [ ] `server.py` emits `audit_log.jsonl` with at least the chapter-02 field list per round.
- [ ] Six runs completed, with per-run checkpoints, logs, and accuracy curves.
- [ ] Three `convergence_alpha_*.png` plots and one `client_sizes_alpha_0.1.png` plot generated.
- [ ] `results.md` under one page, with the six-run results, the FedProx observation, and the audit-log + production-grade paragraphs.
- [ ] `README.md` explains how to reproduce the runs on the pinned versions.

## Stretch goals

- **Add FedAvgM and FedAdam** as two more server strategies; include them in one α = 0.1 run and discuss.
- **Measure dropout tolerance.** Simulate each client failing with probability `p ∈ {0, 0.2, 0.5}` by raising exceptions in `fit()`. Observe how FedAvg's aggregation weight compensates (or doesn't).
- **Participation cap.** Enforce "no client participates in more than 40 rounds". Observe the convergence impact.
- **Save the global model after each run.** Compute a mod-105-style simple MIA on the final model — the baseline attack surface before SA + DP compose into chapter 04 and 05.
- **Instrument with TorchDynamo / torch.compile** to see if client-side training speedups are safe under Flower's serialisation (preview of mlops concerns from chapter 08).

## What this lands

- A runnable, reproducible Flower federation you can extend in later exercises.
- A felt sense of FedAvg's non-IID failure mode and FedProx's mitigation.
- The per-round audit log skeleton that becomes the release-package input downstream.
- Discipline on partition visualisation and seed logging — these show up in every release review.

Version your artefacts. Pin your library versions. Commit the audit log. Everything in exercises 03, 04, 05, 06 assumes you have a working FedAvg loop and the audit log pipeline from here.
