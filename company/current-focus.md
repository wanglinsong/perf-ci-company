# Current Focus — Perf CI

**Last updated:** 2026-10-07  
**Phase:** Phase 1 (Build the AI Engineering Company)

Current-phase context for every role. Update this file when the phase or priorities change; ROLE.md and AGENT.md should not carry phase-specific tasks.

---

## Current Status (2026-10-07) — Execution Round Initiated

| Area | State | Notes |
|------|-------|-------|
| **Wedge doc** (`issue-0001-wedge-validation.md`) | Draft (2026-10-02) | No execution started yet |
| **Product Lead** | 🟡 Delegated — awaiting delivery | Work item `1ae43cd3`: Discovery script + target list. Founder conducts interviews. |
| **Performance Lead** | 🟡 Delegated — awaiting delivery | Work item `e6e84dd3`: H3 Experiment Spec + H10 Algorithm Spec. Must identify version pair. |
| **Platform Lead** | 🟡 Delegated — awaiting delivery | Work item `b0af4ccc`: Pulumi spike + runner script (code/dry-run only, no cloud). |
| **QA Lead** | 🔴 Waiting on Performance Lead | Work item `366549b2`: H4 Design Challenge. Blocked until Perf spec delivered. |
| **S1 (Technical feasibility)** | Not started | Depends on Perf + Platform execution |
| **S2 (Customer problem confirmed)** | 🟡 In progress | Product Lead preparing discovery materials; founder to conduct interviews |
| **S3 (Willingness to pay)** | Not started | Requires S2 results first |

### Intake Artifacts (Complete 2026-10-03)

| Artifact | Path | Status |
|----------|------|--------|
| CEO/CTO Intake Report v1 | `company/CEO-CTO-intake-report-v1.md` | Complete |
| Product Brief v1 | `external/perf-ci-infra/product/product-brief-v1.md` | Complete |
| Experiment Methodology v1 | `external/perf-ci-infra/experiments/methodology-v1.md` | Complete |
| Platform Brief v1 | `external/perf-ci-infra/infra/platform-brief-v1.md` | Complete |
| QA Strategy v1 | `external/perf-ci-infra/qa/qa-strategy-v1.md` | Complete |

### Active Work Items (Delegated 2026-10-07)

| Work Item | Owner | Title | Status | ID |
|-----------|-------|-------|--------|----|
| H3+H10 Specs | Performance Lead | H3 Experiment Spec + H10 Algorithm Spec for Proof Experiment | Ready | `e6e84dd3` |
| Pulumi Spike | Platform Lead | Pulumi Spike for Ephemeral m7g.4xlarge + TPC-DS Runner Script | Ready | `b0af4ccc` |
| Discovery | Product Lead | Discovery Interview Scripts and Target List for S2 Validation | Ready | `1ae43cd3` |
| H4 Design Challenge | QA Lead | H4 Design Challenge of Proof Experiment Spec | Waiting on Perf | `366549b2` |

### Priority Order (CEO/CTO decision — updated 2026-10-07)

1. **Product Lead** — Prepare discovery scripts and target list; the human founder conducts customer interviews (de-risks entire wedge)
2. **Performance Lead** — Identify Gluten version pair (unblocks Platform spike + QA H4 review)
3. **Platform Lead** — Spike EC2 provisioning via Pulumi (parallel with #2 once unblocked)
4. **QA Lead** — H4 design review of experiment spec (after Perf Lead delivers)

### Decisions Recorded This Round

| Decision | Path | Status |
|----------|------|--------|
| ARM vs x86: Option A (m7g.4xlarge, ARM-only) | `company/decisions/2026-10-07-arm-vs-x86-escalation.md` | Decided |
| Cloud access: No provisioning this round — specs/code only | This delegation | Confirmed |

---

## Product Lead

### Perf CI Context

The company's current focus is Spark + Gluten performance regression detection. The initial wedge hypothesis (documented in `external/perf-ci-infra/product/issue-0001-wedge-validation.md`) proposes:

- **Customer:** Platform/data engineering teams that maintain a Gluten fork or upgrade Gluten versions in production Spark deployments
- **Problem:** No reliable automated signal that a Gluten change degraded query performance before it reaches production
- **MVP:** Single-node TPC-DS SF=10 regression detection with statistical analysis, delivered as a GitHub Actions check
- **Validation gap:** Customer confirmation (S2) and willingness-to-pay signal (S3) not yet obtained — this is the current priority

**Discovery call process:** Product Lead prepares interview scripts and target list; the human founder conducts the interviews. Scripts and founder-provided notes go in `external/perf-ci-infra/product/discovery/`. 

CEO H9 for customer access: see `company/decisions/2026-10-07-customer-access-escalation.md`

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

### ARM vs x86 Consideration

The MVP specifies m7g.4xlarge (ARM-based Graviton). This raises a question about whether benchmark results on ARM generalize to x86 production deployments. See H9 escalation in `company/decisions/2026-10-07-arm-vs-x86-escalation.md`.
