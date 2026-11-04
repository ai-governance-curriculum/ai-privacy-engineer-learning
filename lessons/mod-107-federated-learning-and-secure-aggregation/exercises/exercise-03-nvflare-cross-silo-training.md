# Exercise 03: NVIDIA FLARE Cross-Silo Training

**Estimated effort:** 2 hours · **Output:** `nvflare-cross-silo/` directory with a provisioned workspace, a working FedAvg job on 3 silos, the per-site startup kits, the admin-server audit excerpt, and a `results.md` write-up that compares FLARE vs. Flower on the privacy-engineer workflow axis · **Primary chapters:** 03, 01

## Objective

Stand up a cross-silo FL training run on NVIDIA FLARE, operate the provisioning console, observe the admin-server audit log, and understand which pieces of the operational scaffolding make FLARE the mainstream pick for institutional deployments. By the end, you have ground-truth-level awareness of what is and is not covered by the "pick FLARE" decision in chapter 03.

## Prerequisites

- Chapter 03 (library map) read.
- Python 3.10+.
- `pip install nvflare[all]` (or the appropriate NVFlare install per its current docs).
- Docker (optional, for containerised sites).
- Exercise 02 completed (you re-use the FedAvg trainer).

## Problem statement

You will simulate a 3-silo cross-silo federation using NVFlare's local simulation. The silos train on MNIST or CIFAR-10 — either the exercise-02 partitioning adapted, or NVFlare's example partitioning. Pick one; keep the model and data identical to exercise 02 so the comparison is like-for-like.

### Required deliverables

#### `provision/` — the provisioned workspace

Use the NVFlare provisioning tool (`nvflare provision` or the Python API in recent versions) to create a workspace with:

- One **server** entity (your admin / aggregator).
- Three **site** entities (Silo-A, Silo-B, Silo-C).
- One **admin** entity (the user running the training jobs).

Commit the provisioned startup kits under `provision/workspace/` (DO NOT commit the private keys — put them under `.gitignore`; commit only the YAML / JSON manifest files and a `README.md` describing what a real deployment would do with the kits).

#### `jobs/fedavg_mnist/` — the FedAvg job

Define the FLARE job that runs FedAvg across the 3 silos. Structure:

```
jobs/fedavg_mnist/
├── app_server/
│   └── config/config_fed_server.conf
├── app_site/
│   └── config/config_fed_client.conf
├── meta.json
└── custom/        # the training code
    ├── net.py
    ├── mnist_dataset.py
    └── mnist_fedavg_learner.py
```

- Use the `FedAvg` controller and `ScriptRunner` or `PT*` executor the NVFlare docs describe for your installed version.
- Run 20 rounds, all 3 sites each round (cross-silo default; `min_clients = 3`).
- Log per-round metrics in the admin-server audit stream.

#### `run.sh` — the orchestration script

Script that:

- Starts the server, three sites, and an admin process locally (e.g. via NVFlare's POC mode or the appropriate dev-run command for your installed version).
- Submits the FedAvg job through the admin API.
- Tails the admin audit log into `admin_audit.jsonl`.
- Collects per-site stdout into `site_logs/`.

#### `admin_audit.jsonl` — the server-side audit excerpt

Capture the first 20 rounds of the admin audit stream. Verify:

- Each round has a timestamped job-progress event.
- Site participation per round is logged.
- Aggregation step is logged.
- The admin command stream (job submission, abort, resume if applicable) is logged.

#### `results.md` — the write-up

Up to two pages covering:

- The provisioning flow. Write the one paragraph you would show a site admin: "here is your startup kit, here is how you join the federation, here is what we do and do not see from your site".
- The per-round metrics across 20 rounds. One plot.
- A **FLARE vs. Flower** comparison table on the following axes: provisioning story, admin / audit log richness, DP-FL support, SA support, framework agnosticism, production-readiness, teaching-friendliness, time-to-first-working-run. Write one sentence per axis on each library.
- A one-paragraph "which production shape does FLARE serve well, and when would you still pick Flower?".
- A one-paragraph "what mlops owns vs. what this exercise forced *you* to do" — a preview of chapter 08's boundary.

## Starter guidance

- **Start with NVFlare's `hello-world` or `hello-pt` example** before adding your own model. The provisioning / job-graph concepts take a reading pass to absorb.
- **POC mode / dev mode is sufficient** — you are not running against real hospital servers. Everyone on localhost.
- **Keep the model and data identical to exercise 02** — the comparison in `results.md` is only honest if the only variable is the library.
- **Read the admin audit log live.** FLARE's audit stream is more operational than Flower's; a chunk of this exercise is the *reading of the log*, not the training itself. Open it in a second terminal.
- **Note what you did not have to code.** Provisioning, certs, admin console, audit trail — mlops's scope in chapter 08. Write that down while it is fresh.

## Acceptance criteria

- [ ] Provisioned workspace committed (manifests only, not private keys).
- [ ] FedAvg job runs 20 rounds across 3 sites, visible in the admin audit.
- [ ] Per-round metrics (train loss, val metrics) captured in the audit stream.
- [ ] `admin_audit.jsonl` committed with the first 20 rounds of events.
- [ ] `results.md` under two pages, with the provisioning paragraph, the FLARE vs. Flower comparison table, the production-shape paragraph, and the mlops-boundary paragraph.
- [ ] `README.md` explains how to re-run the setup locally.

## Stretch goals

- **Enable NVFlare's built-in DP filter.** Add the DP-FedAvg aggregator (chapter 05) and re-run with `noise_multiplier = 0.5, 1.0, 2.0`. Observe the convergence hit and the per-round DP ε in the audit stream.
- **Simulate a dropped site.** Kill Silo-B mid-round. Observe how FLARE handles it. Compare to how Flower would behave.
- **Add a FLARE filter for per-site telemetry stripping.** Write a small server-side filter that redacts any field you don't want to leak into the audit log. Verify the filter is in the chain.
- **Produce a one-page "site-admin runbook"** that would onboard a new site: how to deploy the startup kit, how to join a job, how to see what the aggregator is doing.
- **Compare the admin-audit format to Flower's** on richness — what fields are present in FLARE's and absent in Flower's.

## What this lands

- First-hand familiarity with FLARE's provisioning + admin model; a working baseline to critique a real cross-silo deployment.
- A concrete answer to "when do we pick FLARE over Flower?" — not a repeated claim from the chapter, but an experience report.
- The two-paragraph writeup separating mlops scope (which FLARE heavily occupies) from privacy scope (what you had to configure or verify) is the first draft of the kickoff paragraph you will carry into chapter 08's boundary discipline.
