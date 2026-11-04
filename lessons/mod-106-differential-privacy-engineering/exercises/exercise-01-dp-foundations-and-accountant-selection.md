# Exercise 01: DP Foundations and Accountant Selection

**Estimated effort:** 2 hours · **Output:** `dp-accountant-bench/` directory with a cross-accountant comparison script, a `results.md` report, and a one-paragraph defence of your chosen production accountant · **Primary chapters:** 01, 02

## Objective

Internalise the accountant choice as a reasoned trade-off rather than a library default. You will compute (ε, δ) for a fixed mechanism specification across four accountants (RDP, PRV, Gaussian-DP, and advanced composition on raw (ε, δ)), observe the tightness gap, run the sanity checks from chapter 02, and write the one-paragraph defence of your production choice. This is the groundwork for every ε you will quote in the rest of the module.

## Prerequisites

- Chapter 01 (DP foundations) read.
- Chapter 02 (accountants and library mapping) read.
- Python 3.10+ environment.
- `pip install opacus prv-accountant dp-accounting matplotlib` (plus `tensorflow-privacy` if you want the TF cross-check).

## Problem statement

You are given a target DP-SGD mechanism specification:

```
mechanism:        subsampled Gaussian
sampling rate q:  1.25e-3   (batch 256, dataset N = 204,800)
noise multiplier: σ ∈ {0.7, 1.0, 1.3, 1.6}
step count T:     ∈ {5_000, 20_000, 80_000}
target δ:         1e-6
neighbouring:     add/remove one record
```

Produce:

### `compute_eps.py` — the cross-accountant script

For every `(σ, T)` pair in the sweep, compute ε at δ = 1e-6 under four accountants:

- Opacus `RDPAccountant`.
- Opacus `PRVAccountant` (or Microsoft `prv_accountant` as an equivalent — verify agreement).
- `dp_accounting.rdp.RdpAccountant` (Google's, as a second RDP implementation for cross-check).
- Advanced composition on raw (ε, δ): treat the per-step mechanism as (ε_step, δ_step)-DP, then compose T steps via advanced composition. This is the loose upper bound.

Emit a CSV `eps_table.csv` with columns `(sigma, T, accountant, epsilon)`.

### `sanity_checks.py` — the accountant unit tests

Run the five sanity checks from chapter 02 against your primary accountant of choice:

- Monotonicity in T (fix σ, δ; vary T; ε should be non-decreasing).
- Monotonicity in σ (fix T, δ; vary σ; ε should be non-increasing).
- Monotonicity in q (fix σ, T, δ; vary q; ε should be non-decreasing).
- Cross-library agreement to within 10% between Opacus-PRV and Microsoft `prv_accountant` (or `dp_accounting`) on the same mechanism.
- Degenerate cases (σ → ∞ should give ε → 0; σ → 0 should give ε → ∞; q → 0 should give ε → 0).

Run the tests as `pytest` or `unittest` and emit a pass/fail summary.

### `plots.py` — the tightness picture

Produce two figures:

- **Tightness gap plot.** X: T (log scale). Y: ε. One line per accountant. Fix σ = 1.0, δ = 1e-6. Save as `plots/tightness_vs_T.png`.
- **ε heatmap.** Heatmap of ε values over the (σ, T) grid for the chosen primary accountant. Save as `plots/eps_heatmap.png`.

### `results.md` — the human-readable write-up

Up to one page covering:

- The table of ε values at a representative (σ, T) (pick σ = 1.0, T = 20,000 as the headline).
- The relative tightness of each accountant (PRV should be tightest; advanced composition loosest; RDP between them; GDP close to PRV).
- The sanity-check pass/fail summary.
- **The one-paragraph production defence:** "For this training recipe we will use the {X} accountant implemented by {library} v{version}. It is tighter than RDP by a factor of ~{gap}% at our target (σ, T), passes the five sanity checks, and agrees with an independent implementation ({Y}) within 2%. The audit log will record both the PRV ε (primary) and the RDP ε (cross-check)."

## Starter guidance

- **Start with just one (σ, T) pair.** Get the four accountants all computing one number before you loop. The APIs differ slightly; a single run shakes out the basic translation.
- **For advanced composition on raw (ε, δ).** The per-step Gaussian mechanism has analytic ε at fixed δ via Balle-Wang 2018's analytic Gaussian mechanism. Use that as the per-step (ε, δ); compose T steps via the Dwork-Rothblum-Vadhan advanced-composition theorem. This should be *dramatically* worse than the Rényi-based numbers; that is the point.
- **The Opacus RDPAccountant exposes alphas.** The optimisation over α during the RDP → (ε, δ) conversion is the common source of small discrepancies between RDP implementations. Set the same α-grid across implementations before claiming they disagree.
- **Pin your library versions.** The results you produce are only comparable if the readers of your `results.md` know what versions you ran.
- **Expect PRV and GDP to be close.** They are both central-limit-flavoured; they should agree to within a few percent. If they do not, one of your accountants is mis-specified.

## Acceptance criteria

- [ ] `eps_table.csv` populated across all 12 `(σ, T)` cells and 4 accountants.
- [ ] Sanity-check script passes all five tests on the primary accountant.
- [ ] `plots/tightness_vs_T.png` shows the four accountants' ε as a function of T, log scale, at σ = 1.0.
- [ ] `plots/eps_heatmap.png` shows the ε surface over the (σ, T) grid for the primary accountant.
- [ ] `results.md` names the chosen production accountant, states its version, and explains the choice in ≤ 150 words.
- [ ] Reproducible: a `README.md` explains how to re-run on the pinned library versions.

## Stretch goals

- **Add TF-Privacy's RDP accountant as a third cross-check** and compare its ε to Opacus-RDP. Reconcile any discrepancy (sampling-rate convention, step-count convention).
- **Add a user-level amplification cell.** If a user contributes k = 5 records, compute the user-level ε under group privacy at δ = 1e-6; plot user-level ε vs k.
- **Add a zCDP cross-check.** Convert the Gaussian mechanism's zCDP ρ to (ε, δ)-DP via the zCDP conversion formula; compare to PRV.
- **Explore the PRV accountant's numerical parameters** (`eps_error`, `delta_error` in the Microsoft implementation) — show how tighter numerics change the reported ε by a sub-percent amount.
- **Add a DP-FTRL (tree aggregation) accountant** for a streaming-mechanism cross-check; useful for the mod-107 federated setting.

## What this lands

- A working intuition for the tightness gap between accountant families — not a textbook claim, a measured one.
- A reusable `compute_eps.py` + `sanity_checks.py` that you will re-run as a CI check for every DP release in later exercises.
- The production defence paragraph becomes the first row of your `claim.md` in exercise 08.
- The sanity-check suite lives alongside your training code; you will adopt it as a release gate.

Version the artefact. Pin the library versions. The one-paragraph production defence is the single most valuable output; write it as if a regulator will read it, because in exercise 04 and exercise 08, one will (in role play).
