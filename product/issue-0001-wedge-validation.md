# Perf CI — Initial Product Wedge Validation

**Date:** 2026-10-02
**Participants:** Product Lead, Performance Lead, Platform Lead, QA Lead, CEO/CTO
**Status:** Draft — awaiting participant review
**Closes:** GitHub Issue #1

---

## Framing

This document answers the question: what is the narrowest valuable
Spark + Gluten performance engineering problem worth building an MVP around?

Narrowest = fewest assumptions, fastest to validate.
Valuable = a real team will change their workflow because of it.

---

## 1. Customer / Problem Hypothesis

**Target customer:**
A platform or data engineering team that maintains a fork of
[Gluten](https://github.com/apache/incubator-gluten) or upgrades
Gluten versions in a production Spark deployment.

**Problem statement:**
When a Gluten change (version upgrade, PR, config change) lands in a
Spark deployment, teams have no reliable automated signal that tells
them whether query performance improved, degraded, or stayed within
acceptable variance — before it reaches production.

**Why this customer:**
Gluten introduces a JVM → native execution boundary that makes
performance behavior unpredictable compared to vanilla Spark.
Small changes to query planner rules, fallback thresholds, or native
library versions can cause silent 10–50% regressions on production
workloads. Teams operating at this boundary are uniquely exposed and
have no commercial tooling to protect them.

---

## 2. Current Workflow

1. Engineer opens a Gluten PR or plans a version upgrade.
2. They manually trigger a TPC-DS or TPC-H benchmark run in a
   shared dev cluster — if they run one at all.
3. Results are collected as raw query runtimes, compared by hand
   in a spreadsheet or notebook.
4. A judgment call is made ("it looks about the same") without
   statistical analysis.
5. PR merges. No benchmark result is recorded.
6. If a regression surfaces, it is caught in staging or production.
7. Root cause requires manual bisection across Gluten versions,
   Spark configs, and execution plans.

**Time from change to confident benchmark verdict:** days to weeks,
or never.

---

## 3. Pain Points

**P1 — No reproducible environment**
Dev clusters share resources, have variable load, and differ in
JVM flags, CPU, and memory from the environment that runs production.
Results are not comparable across runs or across engineers.

**P2 — No statistical rigor**
Single or dual benchmark runs cannot distinguish signal from noise.
A 5% change looks identical to measurement variance. Teams either
ignore small regressions or over-react to noise.

**P3 — No baseline management**
There is no defined "what good looks like." Without a committed
baseline, there is nothing to regress against. Every comparison is
relative to the last run someone remembered.

**P4 — No CI integration**
Benchmarks are not part of the PR workflow. There is no automated
gate. Performance regressions are not blocking.

**P5 — Gluten native boundary obscures root cause**
When a regression is found, determining whether it originates in the
JVM planner, native code, the fallback path, or a config change
requires deep engine expertise and manual execution plan comparison.
This expertise is rare and slow.

**P6 — Knowledge is lost**
Benchmark results live in personal notebooks, Slack threads, and
shared spreadsheets. Findings are not retained as organizational
memory. The next engineer starts from scratch.

---

## 4. Competitive Alternatives

| Alternative | What it does | Why it falls short |
|---|---|---|
| Apache Spark built-in benchmarks | TPC-DS/TPC-H scripts in the Spark repo | No CI integration, no statistical analysis, no Gluten-specific support, no baseline management |
| DIY benchmark scripts + Grafana | Custom scripts, ad-hoc dashboards | High maintenance, no reproducibility guarantees, requires dedicated infra expertise |
| Databricks Runtime benchmarks | Proprietary performance tracking inside DBR | Locked to Databricks; not available to open-source Gluten users |
| Intel / Kyligence internal Gluten CI | Internal CI for upstream Gluten | Not productized, not available to downstream users, no customer-facing API |
| Cloud provider cost dashboards (AWS Cost Explorer, etc.) | Cost tracking | Cost only — no performance measurement, no causal analysis |

**Gap:** No commercial product provides reproducible, statistically
rigorous, CI-integrated performance regression detection for
Spark + Gluten.

---

## 5. MVP Hypothesis

**The smallest useful system:**

> Given a Gluten version diff or config change, run a defined
> benchmark workload in an isolated, reproducible AWS environment,
> compare results against a committed baseline with statistical
> analysis, and report a pass/fail signal as a GitHub Actions check.

**Scope constraints (MVP only):**

- Workload: TPC-DS at scale factor 10 (22 queries, representative coverage)
- Engine: Spark 3.5 + Gluten (latest stable release)
- Environment: Single EC2 instance type (m7g.4xlarge or equivalent) — no cluster for MVP
- Baseline: Stored per Gluten version tag in a managed artifact store
- Statistics: Welch's t-test per query; flag regression at p < 0.05 and ≥ 5% mean change
- Reporting: GitHub Actions summary + per-query flame chart diff (optional stretch)
- Infrastructure: Pulumi-provisioned, fully ephemeral — torn down after each run

**What MVP does not include:**
- Cluster-mode Spark (deferred to Phase 2)
- Trino / Presto (deferred to Phase 2)
- Root-cause AI analysis (deferred to Phase 2)
- Custom customer workloads (deferred to Phase 2)
- SaaS UI (deferred to Phase 3)

---

## 6. Proof Experiment

**Objective:**
Verify that the MVP concept is technically feasible and that
the detection mechanism reliably distinguishes a real regression
from noise before any production code is written.

**Setup:**
- Use two known Gluten releases with a documented performance
  difference (e.g., a release that introduced a known planner
  regression and the release that fixed it).
- Run TPC-DS SF=10 on both versions: 5 repetitions each.
- Execute on identical, freshly provisioned EC2 instances
  (m7g.4xlarge) with identical JVM flags and Spark config.

**What we measure:**
- Per-query mean latency and coefficient of variation (CV) across 5 runs
- Statistical significance of the difference (Welch's t-test, p < 0.05)
- Detection accuracy: does the system flag the known-bad version?
- Reproducibility: re-run the experiment on consecutive days —
  do results stay within ±3% of each other?
- End-to-end wall-clock time from "trigger" to "result"

**Pass criteria for the proof experiment:**
1. Known regression detected with p < 0.05 on ≥ 3 of the affected queries
2. No false positives on the non-regressed queries at p < 0.05 threshold
3. Reproducibility: CV ≤ 5% per query across same-version runs
4. End-to-end time ≤ 2 hours from trigger to result

**Estimated effort:** 1–2 weeks (infrastructure setup + experiment execution)

---

## 7. Success Criteria

The product wedge is validated when all three of the following are true:

**S1 — Technical feasibility confirmed**
The proof experiment passes all four pass criteria above.
The system can detect a ≥ 5% real regression with statistical
confidence in ≤ 2 hours on reproducible infrastructure.

**S2 — Customer problem confirmed**
At least two engineers from distinct Gluten-operating teams
independently describe the current workflow pain (unprompted) and
confirm they would use an automated regression gate if one existed.

**S3 — Willingness to pay signal**
At least one team agrees to run the MVP on a real PR workflow
(pilot agreement, even at no cost), and at least one decision-maker
from that team states what the regression-free confidence signal
is worth to their team in terms of engineering time saved.

---

## Open Questions

1. Is m7g.4xlarge (single node) representative enough for the target
   customer's cluster workload? Should we validate with a 3-node cluster instead?
2. Is TPC-DS SF=10 representative of real customer query complexity,
   or should we use SF=100 from the start?
3. Which Gluten version pair has the clearest documented regression
   for the proof experiment?
4. Are there existing Gluten community contacts who could serve
   as early customer discovery interviews?

---

## Recommended Next Step

1. **Performance Lead:** Identify the Gluten version pair
   with the clearest documented regression for the proof experiment.
   Output: version pair + regression query(ies) documented in `experiments/`.

2. **Platform Lead:** Spike EC2 provisioning via Pulumi for a single
   ephemeral benchmark node. Output: working Pulumi script in `infra/`.

3. **Product Lead:** Schedule 3–5 customer discovery calls with
   engineers at teams known to operate Gluten in production.
   Output: interview notes in `product/discovery/`.
