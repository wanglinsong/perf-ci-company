# Performance Methodology Brief — v1

**Author:** Principal Performance Engineer  
**Date:** 2026-10-03  
**Status:** Active — v1 (greenfield)  
**Scope:** Spark + Gluten (Velox native execution) on AWS EC2; Trino/Presto secondary  

---

## 1. Objective

Define the repeatable, statistically rigorous methodology by which Perf CI determines whether a change to Spark or Gluten improves or degrades real workloads. This document covers:

- Benchmark workload selection and rationale
- Key metrics and measurement approach
- Regression detection criteria and statistical standards
- First experiment design (proof experiment)
- Platform dependencies required to execute experiments

---

## 2. Primary Systems Under Test

| System | Scope |
|--------|-------|
| **Apache Spark** | Catalyst planner, AQE, shuffle, whole-stage codegen, Tungsten memory |
| **Gluten (Velox backend)** | Native columnar execution, JVM/native boundary, fallback behavior, columnar reader |
| **Trino / Presto** | Distributed exchange, join strategies, connector behavior (secondary) |

---

## 3. Benchmark Workloads

### 3.1 Primary: TPC-DS

| Property | Value |
|----------|-------|
| Suite | TPC-DS |
| Scale factors | SF=10 (CI fast path), SF=100 (regression confirmation), SF=1000 (scale validation) |
| Query set | All 99 standard TPC-DS queries |
| Data format | Parquet (snappy compression), ORC as secondary |
| Partitioning | Date-partitioned fact tables (standard TPC-DS layout) |
| Storage | S3-compatible object store or local NVMe depending on environment |

**Rationale:** TPC-DS is the industry standard for analytical query engines. It exercises a wide range of operators (joins, aggregations, window functions, subqueries) and is the most commonly cited benchmark for Spark and Gluten comparisons in published literature and vendor reports.

### 3.2 Secondary: TPC-H

| Property | Value |
|----------|-------|
| Suite | TPC-H |
| Scale factors | SF=10 (CI), SF=100 (confirmation) |
| Query set | All 22 standard TPC-H queries |
| Data format | Parquet (snappy) |

**Rationale:** TPC-H is simpler and faster to run than TPC-DS. It is useful for isolating join and aggregation regressions quickly. Used as a fast-feedback complement to TPC-DS.

### 3.3 Tertiary: Real-Workload Replay (future)

When customer workloads are available (anonymized query logs), replay them as a third workload tier. This tier is not required for v1 but is the long-term target for customer-relevant signal.

### 3.4 Workload Exclusions

- Streaming workloads (Spark Structured Streaming) — out of scope for v1
- ML workloads (MLlib) — out of scope for v1
- Write-heavy workloads (INSERT/UPDATE/MERGE) — secondary priority; may be added in v2

---

## 4. Key Metrics

### 4.1 Primary Metrics

| Metric | Unit | Collection Method |
|--------|------|-------------------|
| **Query wall-clock latency** | seconds | Measured at driver: `System.currentTimeMillis()` around `df.collect()` / `spark.sql().show()` |
| **Total benchmark wall time** | seconds | Sum of all query latencies in a single run |
| **Geometric mean latency** | seconds | Geomean across all queries in a run (primary single-number summary) |

### 4.2 Secondary Metrics (diagnostic)

| Metric | Unit | Purpose |
|--------|------|---------|
| CPU utilization (user + sys) | % | Detect compute regressions vs. I/O regressions |
| JVM GC pause time | ms | Detect GC-driven latency spikes |
| Shuffle bytes written | bytes | Detect plan changes that increase data movement |
| Shuffle spill (disk) | bytes | Detect memory pressure regressions |
| Peak executor memory | bytes | Detect memory footprint changes |
| Gluten fallback operator count | count | Detect native→JVM fallback regressions |
| Task deserialization time | ms | Detect serialization overhead changes |

### 4.3 Cost Metrics (secondary)

| Metric | Unit | Purpose |
|--------|------|---------|
| EC2 instance-hours per benchmark run | instance-hours | Cost efficiency tracking |
| Cost per query (USD) | USD | Normalize across instance types |

### 4.4 Metrics NOT Used as Primary Signals

- Throughput (queries/hour) — derived from latency; not independently measured
- CPU instructions retired — too environment-sensitive for CI use
- Network bytes — too variable across S3 regions

---

## 5. Measurement Approach

### 5.1 Run Structure

```
For each (engine_version, configuration):
  1. Provision fresh instance (no warm state)
  2. Run data generation / validation (skip if data already staged)
  3. Warm-up: 1 full pass through all queries (results discarded)
  4. Measurement: 5 independent full passes (results recorded)
  5. Collect Spark event logs, GC logs, executor metrics
  6. Terminate instance
```

**Minimum repetitions:** 5 measurement runs per configuration.  
**Warm-up:** 1 run discarded to eliminate JIT and OS page-cache cold-start effects.  
**Instance lifecycle:** Fresh instance per experiment pair (baseline + candidate) to eliminate cross-contamination.

### 5.2 Timing Boundaries

- **Start:** Immediately before `spark.sql(query)` is submitted
- **End:** Immediately after the result action (`.collect()` or `.show(1)`) returns
- **Excluded:** Data generation, Spark session startup, result serialization to disk

### 5.3 Environment Controls

| Control | Requirement |
|---------|-------------|
| Instance type | Identical for baseline and candidate |
| OS image | Same AMI; no OS updates between runs |
| JVM version | Pinned (e.g., OpenJDK 11.0.x — exact build) |
| Spark version | Pinned per experiment |
| Gluten version | Pinned per experiment |
| Spark config | Identical across baseline and candidate (documented in experiment spec) |
| Data | Same dataset, same S3 path or local path |
| Network | Same VPC/AZ; no cross-AZ traffic |
| Time of day | Runs scheduled to avoid shared-resource contention (off-peak preferred) |

### 5.4 Data Collection

- Spark event logs: `spark.eventLog.enabled=true`, `spark.eventLog.dir=<experiment_dir>/event_logs/`
- GC logs: `-Xlog:gc*:file=<experiment_dir>/gc.log:time,uptime:filecount=5,filesize=20m`
- Executor metrics: collected via Spark REST API at job completion
- Raw timing: written to `<experiment_dir>/timings/<run_id>.jsonl` (one JSON object per query per run)

---

## 6. Statistical Standards

### 6.1 Summary Statistics (per query, per configuration)

| Statistic | Requirement |
|-----------|-------------|
| Mean | Required |
| Standard deviation | Required |
| Coefficient of variation (CV = stddev/mean) | Required |
| Min / Max | Required |
| 95% confidence interval | Required (t-distribution, n=5) |

### 6.2 Regression Detection

| Criterion | Threshold |
|-----------|-----------|
| Minimum detectable effect | ≥ 5% change in mean latency |
| Statistical significance | p < 0.05 (Welch's two-sample t-test, two-tailed) |
| Acceptable within-config variance | CV ≤ 5% per query |
| Minimum repetitions | 5 per configuration |

**Regression declared when:** mean change ≥ 5% AND p < 0.05.  
**Improvement declared when:** mean change ≤ −5% AND p < 0.05.  
**Inconclusive when:** |mean change| ≥ 5% but p ≥ 0.05, OR |mean change| < 5% regardless of p-value.

### 6.3 Variance Handling

- If CV > 5% for a query within a single configuration, flag the query as **high-variance** and do not declare regression/improvement for that query without additional runs.
- If CV > 10%, the query result is **unreliable** and must be excluded from the geomean summary.
- Outlier detection: flag any run where a single query latency exceeds mean ± 3σ; investigate before including.

### 6.4 Multiple Comparisons

When testing all 99 TPC-DS queries simultaneously, apply Benjamini-Hochberg FDR correction at q = 0.05 to control false discovery rate. Report both raw p-values and BH-adjusted p-values.

### 6.5 Practical vs. Statistical Significance

A result is **actionable** only when both conditions hold:
1. Statistically significant (p < 0.05 after BH correction)
2. Practically significant (≥ 5% mean change)

A result that is statistically significant but < 5% change is noted but not acted upon unless it affects a customer-critical query.

---

## 7. Regression Detection Pipeline

```
Raw timings (JSONL)
  → Compute per-query stats (mean, stddev, CV, CI)
  → Flag high-variance queries (CV > 5%)
  → Welch's t-test per query (baseline vs. candidate)
  → BH FDR correction across all queries
  → Classify each query: REGRESSION | IMPROVEMENT | INCONCLUSIVE | HIGH_VARIANCE
  → Compute geomean delta (excluding HIGH_VARIANCE queries)
  → Generate experiment report (11-point format)
```

The pipeline is implemented as a Python script at `tools/analyze.py` (to be built by Platform Lead per platform dependency list in Section 10).

---

## 8. Experiment Report Format (11-Point Standard)

Every experiment must produce all 11 of the following sections:

1. **Objective** — what question this experiment answers
2. **Baseline** — engine version, config, commit SHA
3. **Candidate** — engine version, config, commit SHA
4. **Environment** — instance type, OS, JVM flags, Spark config, Gluten config
5. **Workload** — benchmark suite, scale factor, queries, data format
6. **Measurements** — per-query latency across all repetitions, raw data (link to JSONL)
7. **Statistical analysis** — mean, stddev, CV, Welch's t-test p-value, BH-adjusted p-value, 95% CI
8. **Observations** — what the data shows, stated as observations not conclusions
9. **Root-cause hypotheses** — engine-level explanations supported by evidence
10. **Confidence** — how confident, and what evidence would change the conclusion
11. **Recommended next experiment** — what to test next and why

Output location: `experiments/<experiment-id>/report.md`

---

## 9. First Experiment Design: Proof Experiment

### 9.1 Goal

Validate that the Perf CI pipeline can detect a known, documented performance regression in Gluten with statistical confidence, and does not produce false positives on non-regressed queries.

### 9.2 Version Pair Selection Criteria

Select two Gluten releases where:
- A documented planner regression exists between them (release notes, GitHub issues, or community reports)
- The regression affects a subset of TPC-DS queries (not all queries — to enable false-positive testing)
- Both releases are buildable from source or available as pre-built artifacts

**Candidate version pair (to be confirmed in `experiments/proof-experiment-setup.md`):**  
Gluten v1.1.x (baseline, pre-regression) vs. Gluten v1.2.x (candidate, with known regression) — exact versions to be identified by reviewing Gluten release notes and GitHub issue tracker.

### 9.3 Experiment Specification

| Property | Value |
|----------|-------|
| Benchmark | TPC-DS SF=10 |
| Queries | All 99 (regression expected on subset; rest serve as false-positive control) |
| Repetitions | 5 per configuration (10 total runs) |
| Warm-up | 1 discarded run per configuration |
| Instance type | AWS EC2 (see Platform Dependencies, Section 10) |
| Parallelism | 1 benchmark run at a time (no concurrent runs) |
| Data format | Parquet, snappy |
| Max wall time | ≤ 2 hours end-to-end |

### 9.4 Success Criteria

| Criterion | Threshold |
|-----------|-----------|
| Known regression detected | p < 0.05 on ≥ 3 affected queries (BH-corrected) |
| False positive rate | 0 false positives on confirmed non-regressed queries |
| Within-config variance | CV ≤ 5% per query |
| End-to-end wall time | ≤ 2 hours |

### 9.5 Failure Modes and Mitigations

| Failure Mode | Mitigation |
|-------------|------------|
| CV > 5% on many queries | Investigate instance noise; consider dedicated tenancy or bare-metal |
| Known regression not detected at p < 0.05 | Increase repetitions to 10; re-evaluate version pair |
| False positives on non-regressed queries | Review environment equivalence; check for data or config differences |
| Run time > 2 hours | Reduce query set to top-20 TPC-DS queries for CI fast path |

### 9.6 Next Experiment (after proof)

If the proof experiment succeeds: run the same methodology on a real Gluten PR under review to validate the pipeline on a live change.  
If the proof experiment fails: diagnose variance source first (instance type, JVM flags, data locality) before proceeding.

---

## 10. Platform Dependencies

The following items are required from the **Platform Lead** to execute experiments. This list is the handoff artifact for platform-lead.

### 10.1 Compute Infrastructure

| Dependency | Specification | Priority |
|-----------|---------------|----------|
| EC2 instance type | Recommended: `r6i.4xlarge` (16 vCPU, 128 GB RAM) or `r6i.8xlarge` (32 vCPU, 256 GB RAM) for SF=100+. Must be consistent across baseline and candidate. | P0 |
| AMI | Amazon Linux 2023 or Ubuntu 22.04 LTS, pinned AMI ID | P0 |
| Instance lifecycle | Fresh instance per experiment run (no reuse); terminated after run | P0 |
| Dedicated tenancy | Preferred for low-variance runs; evaluate cost vs. shared tenancy | P1 |
| NVMe local storage | For shuffle spill and local data staging (if not using S3) | P1 |

### 10.2 Software Environment

| Dependency | Specification | Priority |
|-----------|---------------|----------|
| JVM | OpenJDK 11 (exact build pinned in experiment spec) | P0 |
| Spark | Version pinned per experiment; pre-installed on AMI or installed via provisioning script | P0 |
| Gluten | Pre-built JAR for each version under test; stored in S3 artifact bucket | P0 |
| Python | 3.10+ with `scipy`, `pandas`, `numpy` for analysis pipeline | P0 |
| TPC-DS data generator | `tpcds-kit` compiled and staged; data pre-generated at SF=10 and SF=100 | P0 |

### 10.3 Storage

| Dependency | Specification | Priority |
|-----------|---------------|----------|
| S3 bucket (data) | TPC-DS Parquet data at SF=10 and SF=100; same bucket/path for all runs | P0 |
| S3 bucket (artifacts) | Gluten JARs, Spark distributions, experiment outputs | P0 |
| S3 bucket (results) | Raw timing JSONL, Spark event logs, GC logs | P0 |
| IAM roles | EC2 instance role with read access to data bucket, read/write to results bucket | P0 |

### 10.4 Orchestration

| Dependency | Specification | Priority |
|-----------|---------------|----------|
| Experiment runner script | Shell or Python script that: provisions instance, runs benchmark, collects logs, uploads results, terminates instance | P0 |
| Analysis script (`tools/analyze.py`) | Python script implementing the regression detection pipeline (Section 7) | P0 |
| CI trigger | GitHub Actions or equivalent: trigger experiment on PR label or schedule | P1 |
| Result storage schema | Defined JSONL schema for raw timings (see Section 10.5) | P0 |

### 10.5 Raw Timing Schema

Each timing record written to `<experiment_dir>/timings/<run_id>.jsonl`:

```json
{
  "experiment_id": "string",
  "run_id": "string (UUID)",
  "run_index": "integer (0=warmup, 1-5=measurement)",
  "config": "baseline | candidate",
  "engine": "spark | gluten",
  "engine_version": "string",
  "query_id": "string (e.g. q1, q7, q42)",
  "start_ts_ms": "integer (epoch ms)",
  "end_ts_ms": "integer (epoch ms)",
  "latency_s": "float",
  "status": "success | failed | timeout",
  "error": "string | null"
}
```

### 10.6 Networking

| Dependency | Specification | Priority |
|-----------|---------------|----------|
| VPC / AZ | All instances in same AZ as S3 endpoint (or same region with VPC endpoint) | P0 |
| No cross-AZ shuffle | Spark executor placement must be single-AZ | P0 |
| S3 VPC endpoint | Required to avoid internet egress and reduce latency variance | P1 |

---

## 11. Open Questions for Product Lead

The following questions are sent to product-lead to align on which performance signals matter most to target customers:

1. **Which query patterns matter most?** Are customers primarily running long-running complex joins (TPC-DS Q17, Q64, Q95) or simpler aggregations? This affects which queries we weight in the geomean.
2. **What is the acceptable regression threshold for customers?** Is 5% the right bar, or do customers care only about regressions ≥ 10%?
3. **Is cost-per-query a primary signal?** Should we weight cost metrics equally with latency, or is raw latency the primary customer concern?
4. **Are there specific Gluten features customers rely on?** (e.g., specific operators, specific data formats) — affects which fallback metrics to track.
5. **What is the target SLA for CI feedback?** ≤ 2 hours is the current target; is this acceptable for the product roadmap?

---

## 12. Methodology Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| EC2 instance noise causes CV > 5% | Medium | High | Use dedicated tenancy; run during off-peak hours; increase repetitions |
| TPC-DS does not reflect customer workloads | Medium | Medium | Add real-workload replay tier in v2 |
| Gluten version pair lacks documented regression | Low | High | Identify version pair from GitHub issues before running proof experiment |
| Analysis pipeline bugs produce false regression signals | Low | High | QA Lead independently validates analysis script against synthetic data |
| S3 latency variance contaminates timing | Medium | Medium | Use S3 VPC endpoint; measure and report S3 read time separately |
| JVM JIT non-determinism | Low | Medium | Warm-up run discarded; 5 measurement runs provide sufficient averaging |

---

## 13. Versioning and Change Control

- This document is `methodology-v1.md`. Changes that affect regression thresholds, statistical methods, or workload selection require a new version (`methodology-v2.md`) and a note in the experiment log.
- Experiments reference the methodology version they were run under.
- Retroactive changes to methodology are not applied to completed experiments.

---

## Appendix A: Welch's t-Test Implementation Reference

```python
from scipy import stats
import numpy as np

def welch_ttest(baseline_latencies: list[float], candidate_latencies: list[float]):
    """
    Returns (t_stat, p_value, mean_delta_pct, ci_95_low, ci_95_high)
    """
    t_stat, p_value = stats.ttest_ind(baseline_latencies, candidate_latencies, equal_var=False)
    mean_b = np.mean(baseline_latencies)
    mean_c = np.mean(candidate_latencies)
    mean_delta_pct = (mean_c - mean_b) / mean_b * 100
    
    # 95% CI on the difference
    se = np.sqrt(np.var(baseline_latencies, ddof=1)/len(baseline_latencies) +
                 np.var(candidate_latencies, ddof=1)/len(candidate_latencies))
    df = len(baseline_latencies) + len(candidate_latencies) - 2
    t_crit = stats.t.ppf(0.975, df)
    diff = mean_c - mean_b
    ci_95_low = diff - t_crit * se
    ci_95_high = diff + t_crit * se
    
    return t_stat, p_value, mean_delta_pct, ci_95_low, ci_95_high
```

## Appendix B: BH FDR Correction Reference

```python
from statsmodels.stats.multitest import multipletests

def apply_bh_correction(p_values: list[float], alpha: float = 0.05):
    """
    Returns (reject_array, corrected_p_values)
    reject_array[i] = True means query i is significant after correction
    """
    reject, p_corrected, _, _ = multipletests(p_values, alpha=alpha, method='fdr_bh')
    return reject, p_corrected
```

## Appendix C: Geomean Latency Computation

```python
import numpy as np

def geomean(latencies: list[float]) -> float:
    """Geometric mean of latencies. Excludes any HIGH_VARIANCE queries before calling."""
    return float(np.exp(np.mean(np.log(latencies))))
```

---

*End of Performance Methodology Brief v1*
