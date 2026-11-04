# DP Analytics — PipelineDP, Google DP, and SmartNoise Analytics

Not every DP release is a model or a synthetic dataset. The most common DP release pattern in practice is *DP aggregates*: counts, sums, means, quantiles, histograms — the numbers that populate a dashboard, a research paper, or a regulator's required report. DP analytics is the alternative to DP training when only the aggregates leave your trust boundary. It is often *simpler* than DP-SGD, *tighter* in its ε claims for the same utility, and *cheaper* to operate. Picking it over DP training when the use case fits is the single highest-leverage architectural choice in this module.

Primary references:

- Google Differential Privacy library — [github.com/google/differential-privacy](https://github.com/google/differential-privacy). C++, Go, Java mechanism library; Python front-end via PipelineDP.
- PipelineDP — [pipelinedp.io](https://pipelinedp.io/) and [github.com/OpenMined/PipelineDP](https://github.com/OpenMined/PipelineDP).
- OpenDP (Core library) — [github.com/opendp/opendp](https://github.com/opendp/opendp) and [docs.opendp.org](https://docs.opendp.org/).
- SmartNoise Analytics — [docs.smartnoise.org/sql/](https://docs.smartnoise.org/sql/) and [github.com/opendp/smartnoise-sdk](https://github.com/opendp/smartnoise-sdk).
- Dwork, McSherry, Nissim, Smith — *Calibrating Noise to Sensitivity in Private Data Analysis*. TCC 2006. [link.springer.com/chapter/10.1007/11681878_14](https://link.springer.com/chapter/10.1007/11681878_14) — the Laplace mechanism paper, still foundational.
- McSherry, Talwar — *Mechanism Design via Differential Privacy*. FOCS 2007. [www.cis.upenn.edu/~aaroth/courses/slides/Lecture3.pdf](https://www.cis.upenn.edu/~aaroth/courses/slides/Lecture3.pdf) summary; original via FOCS proceedings — the exponential mechanism.
- Wilson et al. — *Differentially Private SQL with Bounded User Contribution*. PETS 2020. [arxiv.org/abs/1909.01917](https://arxiv.org/abs/1909.01917) — the mechanism behind ZetaSQL-DP and later PipelineDP.
- Garfinkel et al. — *Issues Encountered Deploying Differential Privacy*. WPES 2018. [arxiv.org/abs/1809.02201](https://arxiv.org/abs/1809.02201) — real deployment lessons.

## When to pick DP analytics over DP training

Three triggering conditions, any of which flips the design toward DP analytics:

- **The release is fundamentally aggregate.** Hourly active-user counts, per-region revenue sums, campaign click-through rates, feature-usage histograms. You never intended to release the raw records.
- **The downstream consumers want queries, not a model.** A dashboard that answers "how many users did X in region Y last week" is better served by DP SQL than by a DP-synth dataset plus a non-DP query engine.
- **Low-dimensional, well-defined query surface.** A fixed set of aggregations known in advance. You can allocate ε across the specific queries that matter rather than spraying noise across a whole distribution.

If all three hold, skip DP training entirely. The audit is shorter, the ε is tighter, and the utility is better.

## The two production stacks

The DP-analytics ecosystem has coalesced around two stacks. Both are production-grade; the choice depends on what your data infrastructure looks like.

### PipelineDP (Google / OpenMined)

- **Front-end:** Python, with Apache Beam and Spark backends (plus a local/`pandas` mode for small workloads).
- **Mechanism library under the hood:** Google's C++ DP library (Laplace, Gaussian, bounded sum / count / mean / quantile / variance, approximate-bounds).
- **Key abstractions:** `DataExtractors` (functions to extract a key, value, and privacy_id from each input record), `AggregateParams` (specifies the aggregation + noise mechanism + bounds), `DPEngine.aggregate`.
- **Strength:** scales to Beam / Spark workloads cleanly; the mechanism library is battle-tested across Google's production DP releases.
- **Weakness:** a Pythonic API on top of a C++ mechanism library — some mechanisms the C++ library supports are not yet exposed in Python.

A minimal PipelineDP program:

```python
import pipeline_dp

backend = pipeline_dp.LocalBackend()      # or BeamBackend / SparkBackend
budget_accountant = pipeline_dp.NaiveBudgetAccountant(total_epsilon=1.0, total_delta=1e-6)
dp_engine = pipeline_dp.DPEngine(budget_accountant, backend)

params = pipeline_dp.AggregateParams(
    noise_kind=pipeline_dp.NoiseKind.LAPLACE,
    metrics=[pipeline_dp.Metrics.COUNT],
    max_partitions_contributed=1,      # one partition per user per release
    max_contributions_per_partition=1, # at most one contribution per user per partition
)

data_extractors = pipeline_dp.DataExtractors(
    privacy_id_extractor=lambda row: row.user_id,
    partition_extractor=lambda row: row.region,
    value_extractor=lambda row: 1,
)

counts = dp_engine.aggregate(
    col=input_data, params=params, data_extractors=data_extractors,
)
budget_accountant.compute_budgets()        # finalise ε allocation across aggregations
result = list(counts) if isinstance(backend, pipeline_dp.LocalBackend) else counts.collect()
```

Four things in that block that are the whole engineering story:

- **Contribution bounding.** `max_partitions_contributed` and `max_contributions_per_partition` enforce per-user contribution bounds. This is the sensitivity of the mechanism and the reason the ε claim is honest. If a user can appear in arbitrarily many partitions or contribute arbitrarily many records to one partition, the sensitivity is unbounded and no finite noise gives DP.
- **Privacy ID extraction.** The `privacy_id_extractor` names what one "person" is for the purposes of DP. Match this to the neighbouring relation in your release package (chapter 05). User-level DP means the privacy_id is the user; record-level DP means the privacy_id is the record (and the contribution bound is set to match).
- **Budget allocation.** `NaiveBudgetAccountant` splits the total (ε, δ) across the aggregations in the program. More sophisticated accountants (`PLDBudgetAccountant`, `RDPBudgetAccountant`) are available for tighter composition.
- **Backend agnosticism.** The same code runs on a local pandas-like backend for testing, Apache Beam for production pipelines, and Spark for Databricks-style workloads. This is the production virtue of PipelineDP.

### OpenDP / SmartNoise Analytics

- **Front-end:** Python, with a SmartNoise SQL layer that accepts SQL and translates to a sequence of OpenDP measurements.
- **Mechanism library under the hood:** OpenDP's Rust core (Laplace, Gaussian, discrete Laplace / Gaussian, exponential, geometric, DP aggregations).
- **Key abstractions:** `Measurement` (a DP mechanism with a stated privacy loss), `Transformation` (a non-DP data operation), type-checked composition.
- **Strength:** the type-system approach to composition is harder to misuse; the same primitives work for synth data (chapter 07) and analytics; strong research backing from Harvard Privacy Tools / OpenDP.
- **Weakness:** SQL translation surface still catching up to analyst expectations; some complex queries have to be re-expressed.

A SmartNoise SQL example:

```python
from snsql import Privacy, from_connection

privacy = Privacy(epsilon=1.0, delta=1e-6)
reader = from_connection(
    pyodbc.connect(...), privacy=privacy, metadata="hospital.yaml",
)

q = """SELECT region, COUNT(*) AS n_patients
         FROM patients
        WHERE admission_year = 2024
        GROUP BY region"""
dp_result = reader.execute(q)
```

The `metadata` YAML defines the per-column bounds, per-user contribution bounds, and privacy ID — same engineering as PipelineDP, expressed as config. A SmartNoise release package is the YAML + the SQL + the ε + the audit log.

### Which to pick

| | PipelineDP | SmartNoise Analytics |
|---|---|---|
| Native backend | Beam, Spark, local | Database connector (ODBC, pandas) |
| Interface | Python API | SQL (plus Python API) |
| Config granularity | Per-aggregation Python object | Per-dataset YAML |
| Mechanism library | Google DP (C++ core) | OpenDP (Rust core) |
| Analyst audience | Data engineers | SQL analysts + data engineers |
| Best fit | Scale-out batch analytics pipelines | Analyst-driven dashboards, ad-hoc |

Both are production-grade. If your pipeline is Beam / Spark-shaped, PipelineDP is the simpler fit. If your users think in SQL, SmartNoise is the simpler fit. They are not mutually exclusive — a shop can run both.

## The mechanism primitives you will use

The DP-analytics libraries wrap a handful of primitives. Know what each does; the release package names which one every query used.

- **Laplace mechanism.** Add noise from a Laplace distribution with scale `Δ / ε`. For integer-valued sensitivities, the preferred integer variant is the **discrete Laplace** (geometric mechanism). Use for counts and bounded sums at small ε.
- **Gaussian mechanism.** Add noise from `N(0, (Δσ)²)` with σ calibrated for `(ε, δ)`. Use for continuous aggregates and when composition matters (Gaussian composes more tightly than Laplace via Rényi / PRV).
- **Discrete Gaussian.** Integer-valued variant; useful for integer counts where the Gaussian's continuous support is awkward.
- **Exponential mechanism.** Private selection — pick the "best" element from a set under a utility function, with noise on the utility. Use for DP mode / majority / argmax queries and for DP threshold selection.
- **Bounded-sum / bounded-mean.** Composite mechanisms that enforce a per-record value bound (clipping) before the noise addition. The clipping is an implicit Laplace mechanism parameter; the library handles it.
- **Quantile tree / bounded-quantile.** DP quantile estimation; Dwork-Roth has the theory, the libraries ship the code.

Which mechanism is right for a given query is often not obvious. The libraries have sensible defaults (Laplace for counts; Gaussian for continuous), and the release package should quote the mechanism explicitly: "monthly active users: Laplace mechanism with ε = 0.1 and user-level sensitivity 1".

## Contribution bounding — the sensitivity story for aggregates

The DP-SGD sensitivity story was "clip per-example gradient to C". The DP-analytics sensitivity story is "bound how much any one user can change the aggregate". It has two knobs:

- **Per-partition contribution limit** (`max_contributions_per_partition`). One user's influence on any single aggregate value. For a COUNT aggregate with a per-user limit of 1, one user adds at most 1 to any count. Sensitivity is 1.
- **Cross-partition contribution limit** (`max_partitions_contributed`). How many distinct aggregate values a user can affect at all. If a user can appear in at most 5 partitions (e.g., 5 of the 50 regions), their total contribution is bounded in L1 / L2 terms.

Three failure modes:

- **No contribution bound set.** The library's default sensitivity is 1 per partition, but if a user appears in 50 partitions you have L1 sensitivity 50 and the ε you report is 50× looser than it looks. Set both knobs explicitly.
- **Bound set too loose for utility.** If you set `max_contributions_per_partition = 1000` because some users contribute that many, the sensitivity is 1000 and the noise is 1000× larger. Pick a bound that trades utility for coverage — users above the bound are clipped. Document the clipping impact.
- **Bound set *after* the fact.** The contribution bound is a pre-processing step; it is not an after-the-mechanism filter. If your pipeline sees all a user's records, picks the first K, and then calls the DP mechanism, you have *leaked* information (which records were first depended on the data). Pre-process the data to randomly sample K contributions per user before the mechanism.

The engineering audit note: "User contribution bounded to 5 partitions, 1 record per partition. Users with more records are randomly subsampled to 5 partitions (reservoir sampling, seed X). Users with more than 1 record per partition are randomly subsampled to 1."

## The budget-allocation story

A DP-analytics program typically computes more than one aggregate at once. The budget accountant decides how to split the total ε across the aggregations.

Three strategies:

- **Uniform split.** Equal ε per aggregation. Easy; ignores that some aggregations matter more.
- **Analyst-weighted split.** The analyst picks a weight per aggregation; higher-priority metrics get more budget and less noise.
- **Variance-minimising split.** An optimal allocation under a utility model. PipelineDP's `PLDBudgetAccountant` approximates this; OpenDP has similar machinery.

For a first deployment, analyst-weighted is the pragmatic default: the dashboard product owner tells you which three KPIs are the headline metrics and those get 50% of the budget collectively; the rest split uniformly.

## Partition selection — the Wilson-et-al trick

A naïve DP COUNT query by region computes a noisy count for *every* region — including ones where zero users appear. Releasing the noisy zero counts tells an attacker which regions have no users (useful side information for an adversary). Partition selection is the DP-aware way to decide which partitions to release.

Wilson et al. 2020 define the mechanism: for each candidate partition, compute a noisy count; release the partition only if the noisy count exceeds a threshold calibrated for a target δ. PipelineDP and SmartNoise both implement this.

The release discipline: "Partitions with fewer than T users are suppressed via Wilson-et-al 2020 partition-selection at threshold-δ = 1e-7. Reported ε includes the partition-selection cost."

## The dashboard pattern — standing DP vs one-shot

Two patterns in the wild:

- **One-shot release.** Run the DP mechanism once; publish the aggregates; spend ε once. Example: a research paper's single table.
- **Standing release (refreshed dashboard).** Run the DP mechanism periodically (hourly, daily, weekly); publish updated aggregates each time. Each run composes with the previous.

The standing pattern is where DP analytics gets expensive. A daily-refreshed dashboard for a year composes 365 times. Three mitigations:

- **Renew the user pool.** If each day's data is a (near-)disjoint set of users (for example, "today's new signups"), the composed ε is the single-run ε, not 365 × single-run ε. The audit note: "Daily aggregates are over records with ingestion_date = today; user_id observed on earlier days is excluded from future days by construction."
- **Fresh ε allocation per window.** Set an annual budget (e.g., ε = 10) and allocate 10/365 per daily refresh. The utility per day degrades, but the total ε stays bounded. Document the expiry.
- **Streaming DP mechanisms.** Tree-aggregation (Chan-Shi-Song, Dwork-Naor-Pitassi-Rothblum 2010) and DP-FTRL (McMahan et al. 2022) handle continual release with sub-linear composition. More complex to deploy; appropriate when the dashboard is high-frequency and the data stream is ongoing.

Chapter 09's release package has a "release cadence" section exactly for this.

## Failure modes that are specific to DP analytics

The shared failure modes with DP-SGD — accountant mismatch, sensitivity bug, sampling-regime drift — all apply. The ones specific to analytics:

- **Private-ID leak via side channels.** The DP aggregate is clean; the system log that records "query run on cohort X, returned Y partitions" is not. Any logging around the DP query has to be audited for per-record information.
- **Analyst running one query a thousand times with tiny changes.** Each run spends budget. Policy: the ε accountant is enforced *across sessions*, not per session. SmartNoise's YAML and PipelineDP's `NaiveBudgetAccountant` both support persistent budget tracking; wire it up.
- **Metadata leaks.** Reporting "COUNT over 50 partitions with Wilson-et-al threshold T = 20" tells an attacker that at least one partition had fewer than 20 users. If the fact of the partition's existence is sensitive, the partition-selection mechanism has to use a larger threshold.
- **Public auxiliary data joins.** If the analyst joins a DP aggregate with a non-DP public table, differential-privacy guarantees are preserved by post-processing, but the *utility* often collapses because the join amplifies noise. Flag loudly in release notes.
- **Re-executing a query after a bug.** Running the DP query, finding a bug, fixing, re-running, publishing both: you have spent 2× ε. The policy has to be "every published number comes from a single accounted run; replay is forbidden."

## The release package from an analytics run

Chapter 09's release package for an analytics run looks like:

```jsonc
{
  "release_id": "dashboard-hospital-weekly-2026-11-01",
  "release_cadence": "weekly, every Monday at 09:00 UTC",
  "total_epsilon_per_release": 1.0,
  "total_delta_per_release": 1e-6,
  "annual_budget_claim": "ε = 52.0, δ = 5.2e-5 under naive composition; under PRV: ε ≈ 12.4 at user-level, assuming non-disjoint user pool across weeks",
  "accountant": "PipelineDP NaiveBudgetAccountant (per-release), cross-checked by google.dp_accounting for the annual PRV number",
  "privacy_id": "user_id",
  "neighbouring_relation": "add_or_remove_one_user",
  "contribution_bounds": {
    "max_partitions_contributed": 5,
    "max_contributions_per_partition": 1,
    "subsampling_strategy": "reservoir, seed=X"
  },
  "aggregates": [
    {
      "name": "weekly_active_users_by_region",
      "mechanism": "laplace",
      "sensitivity": 5,
      "epsilon": 0.5
    },
    {
      "name": "weekly_sum_revenue_by_region",
      "mechanism": "laplace",
      "value_bounds": [0, 1000],
      "sensitivity": 5000,
      "epsilon": 0.5
    }
  ],
  "partition_selection": {
    "mechanism": "Wilson-et-al 2020",
    "threshold_delta": 1e-7,
    "cost_epsilon": 0.0
  },
  "engineering_attestations": [
    "Users with more than 5 partitions subsampled to 5 via reservoir, seed X, deterministic per weekly run.",
    "DP-aggregate re-execution forbidden by policy P; audit log Q confirms single execution per published value."
  ]
}
```

## Summary — what you can do after this chapter

- Decide when DP analytics is the right release shape (aggregate-only; dashboard / query; well-defined aggregation surface) and when it is not.
- Pick between PipelineDP and SmartNoise based on backend (Beam / Spark vs. database-connector) and user audience (Python-engineer vs. SQL-analyst).
- Name the five mechanism primitives (Laplace, Gaussian, discrete variants, exponential, bounded-sum / mean / quantile) and know which one to pick for a given aggregate.
- Set contribution bounds correctly: `max_partitions_contributed` and `max_contributions_per_partition`, pre-process the data to enforce them.
- Allocate the ε budget across aggregations using analyst-weighted or variance-minimising strategies.
- Apply partition selection (Wilson et al. 2020) to avoid releasing noisy zero counts for sparse partitions.
- Reason about one-shot vs. standing (dashboard-refresh) release patterns and the three mitigations for the lifetime-ε explosion.
- Produce the analytics-release package JSON with contribution bounds, aggregate specs, partition selection, and engineering attestations.

Chapter 09 is the module's capstone — the DP release package, consolidating everything from DP-SGD (chapters 03–04), PATE (chapter 06), DP synth (chapter 07), and DP analytics (this chapter) under the NIST SP 800-226 evaluation framework.
