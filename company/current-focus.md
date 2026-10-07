# Current Focus — Perf CI

Current-phase context for every role. Update this file when the phase or priorities change; ROLE.md and AGENT.md should not carry phase-specific tasks.

## Product Lead

### Perf CI Context

The company's current focus is Spark + Gluten performance regression detection. The initial wedge hypothesis (documented in `external/perf-ci-infra/product/issue-0001-wedge-validation.md`) proposes:

- **Customer:** Platform/data engineering teams that maintain a Gluten fork or upgrade Gluten versions in production Spark deployments
- **Problem:** No reliable automated signal that a Gluten change degraded query performance before it reaches production
- **MVP:** Single-node TPC-DS SF=10 regression detection with statistical analysis, delivered as a GitHub Actions check
- **Validation gap:** Customer confirmation (S2) and willingness-to-pay signal (S3) not yet obtained — this is the current priority

Customer discovery interviews should target engineers at teams known to operate Gluten in production. You prepare the interview scripts and target list; the human founder conducts the interviews. Scripts and founder-provided notes go in `external/perf-ci-infra/product/discovery/`.

## Principal Performance Engineer

### Perf CI Context

The current proof experiment (from `external/perf-ci-infra/product/issue-0001-wedge-validation.md`) requires:

1. Identify two Gluten releases with a documented performance difference — one with a known planner regression, one without.
2. Run TPC-DS SF=10 on both: 5 repetitions each, on identical freshly provisioned EC2 instances.
3. Verify: known regression detected at p < 0.05 on ≥ 3 affected queries; no false positives on non-regressed queries; CV ≤ 5% within same-version runs; end-to-end time ≤ 2 hours.

First task: identify the Gluten version pair and document it in `external/perf-ci-infra/experiments/proof-experiment-setup.md`.

## Platform Engineering Lead

### MVP Scope (Current Phase)

From `external/perf-ci-infra/product/issue-0001-wedge-validation.md`:

- **Instance type:** m7g.4xlarge (single node, no cluster for MVP)
- **Workload:** TPC-DS SF=10
- **Engine:** Spark 3.5 + Gluten (latest stable)
- **Baseline storage:** S3, keyed by Gluten version tag
- **CI integration:** GitHub Actions check on a Gluten PR
- **Provisioning:** Pulumi — fully ephemeral, torn down after each run
- **Reporting:** GitHub Actions summary with per-query latency comparison

First task: spike EC2 provisioning via Pulumi for a single ephemeral benchmark node. Output: working Pulumi script in `external/perf-ci-infra/infra/`.
