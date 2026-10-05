# Perf CI — CEO/CTO Intake Synthesis Report
**Date:** 2026-10-03  
**Author:** CEO / CTO  
**Status:** Final — Intake Phase Complete  
**Audience:** Human Founder  

---

## Executive Summary

All four IC roles have completed their intake deliverables. The company has a coherent, evidence-backed product hypothesis, a rigorous performance methodology, a concrete infrastructure architecture, and an independent QA strategy — all written from scratch on a greenfield workspace.

**The core bet:** Gluten platform engineers at companies like Microsoft, Uber, ByteDance, and Meituan have no automated performance gate on Gluten version changes. They discover regressions in production. Perf CI delivers a GitHub Actions check that runs TPC-DS SF=10, applies Welch's t-test + BH FDR correction, and posts a statistically rigorous pass/fail verdict on every PR — for ~$0.86/run.

**What's ready:** Four foundational strategy documents, a proof experiment design, and a prioritized P0 build list. No code has been written yet; no AWS account is provisioned.

**What's blocking:** Three items require founder input before the team can move to execution: (1) AWS account + GitHub repo access, (2) a decision on instance type (ARM64 vs x86), and (3) customer access for discovery interviews.

---

## 1. Deliverables Produced This Intake Phase

| Role | Artifact | Lines | Status |
|------|----------|-------|--------|
| Product Lead | `product/product-brief-v1.md` | 344 | ✅ Approved |
| Performance Lead | `experiments/methodology-v1.md` | 449 | ✅ Approved |
| Platform Lead | `infra/platform-brief-v1.md` | 397 | ✅ Approved |
| QA Lead | `qa/qa-strategy-v1.md` | 373 | ✅ Approved |

All four artifacts are committed to the workspace at `/Users/lwang/gh/wls/perf-ci-company/external/OpenOPC_workplace/perf-ci/`.

---

## 2. Product Hypothesis (from Product Lead)

### The Customer
**Primary:** Gluten platform engineer (IC4–IC6) at a company running Gluten in production. Responsible for upgrading Gluten versions without breaking production SLAs. 15+ named production adopters confirmed (Microsoft, Google, Uber, ByteDance, Meituan, Alibaba, Intel, Pinterest, Tencent, Baidu, BIGO, Netease, Kyligence, IBM, Oracle).

**Secondary:** Gluten contributor who needs PR-level performance feedback before merge.

### The Pain (Evidence-Backed)
The 7-step current workflow ends in production incidents. Key evidence:
- **Gluten GitHub issue #8417** (Jan 2026, open): TPC-DS Q72 runs 40% slower than vanilla Spark — flagged as *critical* in the project's own weekly status bot.
- **Zhihu case study**: 10,000+ daily Spark SQL jobs; initial Gluten adoption revealed "serious shuffle-stability problems" caught only through manual testing.
- **Project velocity**: 47 PRs + 24 issues in one week — manual benchmarking cannot keep up.

### The MVP (5 Must-Have Features)
1. **F1 — GitHub Actions workflow integration** (zero-friction adoption gate)
2. **F2 — TPC-DS SF=10 execution** (20–30 regression-sensitive queries; <60 min)
3. **F3 — Statistical regression detection** (Welch's t-test + BH FDR; ≥5% + p<0.05)
4. **F4 — Baseline storage and history** (keyed by commit SHA + Gluten version)
5. **F5 — PR comment report** (per-query delta, pass/fail, link to full results)

### Pricing Hypothesis
| Tier | Price | Target |
|------|-------|--------|
| Free | $0 | OSS contributors; self-hosted |
| Team | $299/month | Platform teams at production adopters |
| Enterprise | Custom | Large-fleet operators (Uber, ByteDance scale) |

*Anchored to CodSpeed ($15/user/month) and Bencher ($100/month flat). Not yet validated with any customer.*

### Competitive Gap
No tool combines CI automation + Spark/Gluten-aware workloads + statistical regression detection. CodSpeed and Bencher prove willingness-to-pay but don't support JVM/Spark workloads. Gluten's own `gluten-it` CLI is manual, untracked, and has no statistical analysis.

---

## 3. Performance Methodology (from Performance Lead)

### Benchmark Stack
- **Primary:** TPC-DS (all 99 queries; SF=10 for CI, SF=100 for confirmation, SF=1000 for scale)
- **Secondary:** TPC-H (22 queries; fast-feedback complement)
- **Future:** Real-workload replay (customer query logs; v2)

### Statistical Standards (non-negotiable)
| Standard | Value |
|----------|-------|
| Primary metric | Query wall-clock latency (geomean across queries) |
| Minimum runs | 5 per configuration (+ 1 discarded warmup) |
| Variance gate | CV ≤ 5% per query; CV > 10% = unreliable, excluded |
| Significance test | Welch's two-sample t-test (two-tailed) |
| Multiple comparisons | Benjamini-Hochberg FDR correction at q=0.05 |
| Regression threshold | ≥5% mean change AND p<0.05 (both required) |
| Improvement threshold | ≤−5% mean change AND p<0.05 |

### Proof Experiment Design
The first experiment validates the pipeline itself: detect a *known* Gluten regression with statistical confidence, zero false positives on non-regressed queries.
- **Workload:** TPC-DS SF=10, all 99 queries
- **Version pair:** Gluten v1.1.x vs v1.2.x (exact versions TBD — requires Gluten release note review)
- **Wall time target:** ≤2 hours end-to-end
- **Success bar:** p<0.05 on ≥3 affected queries (BH-corrected); 0 false positives

### Diagnostic Metrics (secondary)
CPU utilization, JVM GC pause, shuffle bytes/spill, peak executor memory, **Gluten fallback operator count** (key signal for native→JVM regressions).

---

## 4. Platform Architecture (from Platform Lead)

### Execution Environment
| Layer | Choice | Rationale |
|-------|--------|-----------|
| Instance | `m7g.4xlarge` (Graviton3, ARM64, 16 vCPU, 64 GiB) | Eliminates Intel JIT noise; ~$0.67/hr |
| OS | Amazon Linux 2023 (ARM64), pinned AMI | Reproducibility |
| JVM | Amazon Corretto 17 (LTS) | Platform choice |
| Spark | 3.5.x, pinned by artifact SHA | Reproducibility |
| Provisioning | Pulumi (TypeScript), fully ephemeral | Fresh instance per run |
| Orchestration | GitHub Actions | Zero-friction CI integration |
| Result store | S3 (`perf-ci-results/`, `perf-ci-datasets/`, `perf-ci-artifacts/`) | Immutable per-run artifacts |

**Cost per run:** ~$0.86 (75 min on m7g.4xlarge on-demand + EBS + S3)  
**Monthly budget alert:** $200/month (AWS Budgets)  
**TTL guard:** Lambda auto-terminates instances older than 4 hours

### Experiment Lifecycle
```
PROVISION (4 min) → EXECUTE (55–70 min) → COLLECT (2 min) → TEARDOWN (90 sec)
Total: ~65–80 min wall time
```

### P0 Build List (unblocks first run, ~5.5 days effort)
1. Pulumi EC2 provisioner spike (`infra/pulumi/`)
2. cloud-init bootstrap script (`infra/scripts/bootstrap.sh`)
3. Benchmark harness (`infra/scripts/run-tpcds.sh`)
4. Result schema + S3 upload (`infra/scripts/collect-results.sh`)
5. GitHub Actions workflow — manual trigger (`.github/workflows/perf-run.yml`)

### P1 Build List (CI gate, ~3.5 days effort)
6. Baseline comparison script (`infra/scripts/compare-baseline.py`)
7. GitHub Actions PR integration (`.github/workflows/perf-pr-check.yml`)
8. Cost reporting (`infra/scripts/emit-cost.py`)
9. TTL guard Lambda (`infra/lambda/ttl-guard/`)

---

## 5. QA Strategy (from QA Lead)

### Guiding Principles
- Default to **NEEDS WORK** — burden of proof is on the claimant
- Never approve on a single run — reproducibility requires independent re-execution
- AI-generated root causes are hypotheses until supported by profiles/logs
- Zero tolerance for environment inequivalence between baseline and candidate

### Four QA Checkpoints
| Checkpoint | Trigger | Gate |
|-----------|---------|------|
| 1 — Design Review | Performance Lead submits spec | QA signs off or returns required changes |
| 2 — Environment Equivalence | Platform Lead signals provisioned | QA confirms equivalence or blocks execution |
| 3 — Execution Integrity | Raw results available | QA confirms CV ≤ 5%, n ≥ 5, no mid-run config change |
| 4 — Result Validation | Performance Lead submits analysis | QA issues APPROVED / NEEDS WORK / BLOCKED |

### 8 Automatic Block Triggers (escalate to CEO/CTO immediately)
1. Raw per-run measurements unavailable (only summary stats)
2. CV > 5% within same configuration
3. Baseline and candidate ran on different hardware/days without controlled re-run
4. Statistical test not applied, or p > 0.05 for a claimed regression
5. AI root cause has no supporting execution plan, CPU profile, GC log, or shuffle metric
6. "Pass" claimed on first attempt without independent reproduction
7. Metrics selected or changed after execution began
8. Run count below pre-specified minimum

### Result Acceptance Criteria
| Criterion | Threshold |
|-----------|-----------|
| Minimum runs | n ≥ 5 per configuration |
| Variance | CV ≤ 5% |
| Significance | p < 0.05 (two-sided t-test or Mann-Whitney U) |
| Effect size | ≥ 5% relative change |
| Confidence interval | 95% CI reported and non-overlapping |
| Reproducibility | Independent re-run; magnitude within 10% of original |

---

## 6. Cross-Role Conflict Register

Three conflicts emerged between IC briefs. All require resolution before execution begins.

### CONFLICT-1: Instance Type — ARM64 vs x86 ⚠️ UNRESOLVED

| Role | Position |
|------|----------|
| Platform Lead | `m7g.4xlarge` (Graviton3, ARM64, 64 GiB) — lower cost, eliminates Intel JIT noise |
| Performance Lead | `r6i.4xlarge` (Intel Ice Lake, x86, 128 GiB) — more RAM for SF=100+; x86 JVM behavior |

**Root cause:** Platform Lead wrote the brief before receiving Performance Lead's dependency list. Performance Lead recommends r6i for SF=100+ RAM headroom; Platform Lead chose m7g for noise reduction and cost.

**CEO/CTO Decision Required:**
- **Option A — m7g.4xlarge (ARM64):** Lower cost (~$0.67/hr), better noise profile, but Gluten ARM64 artifact availability is **unverified**. If Gluten doesn't publish ARM64 JARs, this forces a source build, adding significant complexity.
- **Option B — r6i.4xlarge (x86):** Higher cost (~$1.01/hr), proven x86 JVM behavior, 128 GiB RAM handles SF=100 without spill, Gluten x86 artifacts are confirmed available.
- **Recommendation:** **Start with r6i.4xlarge (x86)** for the proof experiment. Verify Gluten ARM64 artifact availability in parallel. Switch to m7g if ARM64 JARs are confirmed and cost becomes a priority. Do not let ARM64 uncertainty block the first run.

### CONFLICT-2: JVM Version — OpenJDK 11 vs Corretto 17 ⚠️ UNRESOLVED

| Role | Position |
|------|----------|
| Performance Lead | OpenJDK 11 (exact build pinned) |
| Platform Lead | Amazon Corretto 17 (LTS) |

**Root cause:** Same sequencing issue as CONFLICT-1.

**CEO/CTO Decision Required:**
- **Option A — OpenJDK 11:** Matches the JVM version most Gluten production deployments use (Gluten's own CI uses JDK 11). More representative of customer environments.
- **Option B — Corretto 17:** LTS, AWS-supported, better long-term maintenance. Gluten supports JDK 17 as of recent releases.
- **Recommendation:** **Use OpenJDK 11 for the proof experiment** to maximize representativeness of production Gluten deployments. Plan a JDK 17 migration experiment once the proof experiment succeeds.

### CONFLICT-3: Run Count vs Wall Time Budget ⚠️ NEEDS MONITORING

| Role | Position |
|------|----------|
| Performance Lead | 5 measurement runs + 1 warmup per configuration = 10 total runs |
| Platform Lead | Lifecycle estimate: 55–70 min for 1 timed run; 65–80 min total |

**Issue:** Platform Lead's lifecycle estimate covers 1 warmup + 1 timed run. Performance Lead requires 5 timed runs. Actual wall time for 5 measurement runs at SF=10 is likely 5–6 hours, not 65–80 minutes — well over the ≤2 hour proof experiment target.

**CEO/CTO Decision Required:**
- **Option A — Reduce query set for CI fast path:** Use 20–30 regression-sensitive queries (not all 99) for the 5-run CI check. Full 99-query suite reserved for nightly/confirmation runs. Product Lead already scoped F2 to "20–30 queries" — this is consistent.
- **Option B — Reduce repetitions for CI:** Use 3 runs for the PR-gated check (accepting lower statistical power); use 5 runs for nightly/confirmation. QA's minimum is n≥5, so this requires QA sign-off.
- **Recommendation:** **Option A** — adopt the 20–30 query reduced subset for the CI fast path. Performance Lead should define the specific query list (drawing from the sensitive query list in product-brief-v1.md Appendix A). Full 99-query suite runs nightly. This preserves the ≤2 hour wall time and the n≥5 statistical requirement simultaneously.

---

## 7. Open Decisions for Founder

These items cannot be resolved by the IC team without founder input or external action.

### D1 — AWS Account + GitHub Repo Access 🟡 PARTIALLY RESOLVED

**GitHub repo:** ✅ **https://github.com/wanglinsong/perf-ci-infra** — confirmed by founder (2026-10-05). All infra code, scripts, and documents go here.

Platform Lead still needs:
- AWS account credentials (EC2, S3, IAM, Budgets) — **still blocking**
- GitHub Actions secrets configured in the repo

**Action:** Founder provisions AWS account credentials and configures GitHub Actions secrets in `wanglinsong/perf-ci-infra`. The repo itself is now unblocked — Platform Lead can begin scaffolding the repo structure immediately.

### D2 — Customer Access for Discovery Interviews 🔴 BLOCKING VALIDATION

Product Lead's S2 validation gate (customer confirmation of pain) requires 3–5 discovery interviews with Gluten platform engineers at production adopters (Meituan, Alibaba, Intel, Microsoft, LinkedIn are the top targets).

No customer has independently confirmed the pain or expressed intent to pay. The product hypothesis is evidence-backed but unvalidated.

**Action:** Founder provides warm introductions to Gluten platform engineers at 2–3 production adopters. This is the highest-priority ask from the product brief.

### D3 — OSS vs. Proprietary Positioning 🟡 STRATEGIC

The pricing model ($299/month Team tier) assumes a proprietary SaaS offering. An alternative is to open-source the core pipeline and monetize managed hosting + enterprise support (CodSpeed's model).

**Tradeoffs:**
- **Proprietary:** Faster revenue, harder adoption in OSS community
- **OSS core + managed:** Slower revenue, faster adoption, community moat

**Action:** Founder decides positioning before go-to-market work begins. Does not block technical execution.

### D4 — Gluten ARM64 Artifact Verification 🟡 TECHNICAL

Resolves CONFLICT-1. Platform Lead needs to verify whether Gluten publishes pre-built ARM64 JARs for the target release before committing to m7g.4xlarge.

**Action:** Performance Lead or Platform Lead checks Gluten GitHub Releases for ARM64 artifacts. If available, m7g.4xlarge is viable. If not, r6i.4xlarge is the default. This can be done without founder input — flagged here for visibility.

---

## 8. Risk Register (Top 5)

| # | Risk | Severity | Owner | Status |
|---|------|----------|-------|--------|
| R1 | No customer validation — product hypothesis unconfirmed | HIGH | Product Lead | Open — blocked on D2 |
| R2 | Gluten ARM64 artifacts unavailable — forces x86 fallback | HIGH | Platform Lead | Open — needs verification |
| R3 | 5-run × 99-query wall time exceeds ≤2h proof experiment target | HIGH | Performance Lead + Platform Lead | Mitigated by reduced query set (D3 decision) |
| R4 | AWS account not provisioned — blocks all P0 platform work | HIGH | Founder | Open — blocked on D1 |
| R5 | Pricing hypothesis ($299/month) unvalidated with any customer | MEDIUM | Product Lead | Open — blocked on D2 |

---

## 9. Recommended Next Actions (Prioritized)

### Founder Actions (this week)
1. **Provision AWS account + GitHub repo** (D1) — unblocks all platform P0 work immediately
2. **Provide 2–3 warm intros to Gluten platform engineers** (D2) — unblocks customer validation
3. **Decide OSS vs. proprietary positioning** (D3) — needed before go-to-market work

### CEO/CTO Decisions (this week)
4. **Resolve CONFLICT-1:** Adopt r6i.4xlarge (x86) as the default instance type for the proof experiment; verify ARM64 in parallel
5. **Resolve CONFLICT-2:** Adopt OpenJDK 11 for the proof experiment; plan JDK 17 migration experiment post-proof
6. **Resolve CONFLICT-3:** Adopt 20–30 query reduced subset for CI fast path; Performance Lead to define the specific query list

### IC Team Actions (next sprint, after D1 resolved)
7. **Platform Lead:** Begin P0 build list — Pulumi spike, bootstrap script, benchmark harness, result schema, GitHub Actions workflow (manual trigger). ~5.5 days.
8. **Performance Lead:** Identify exact Gluten version pair with documented regression; write `experiments/proof-experiment-setup.md`. Define 20–30 query CI fast-path subset.
9. **Product Lead:** Conduct 3–5 customer discovery interviews (pending D2). Answer Performance Lead's 5 open questions in `experiments/methodology-v1.md §11`.
10. **QA Lead:** Update `qa/qa-strategy-v1.md` to align with confirmed methodology (instance type, JVM, run count, query set). Conduct Checkpoint 1 design review on the proof experiment spec once Performance Lead delivers it.

---

## 10. Architecture Decisions Log

| Decision | Choice | Rationale | Status |
|----------|--------|-----------|--------|
| Primary benchmark | TPC-DS | Industry standard for Spark/Gluten; used in Gluten's own performance claims | ✅ Locked |
| Scale factor (CI) | SF=10 | Meaningful signal; fits in memory on single node; <2h wall time | ✅ Locked |
| Statistical test | Welch's t-test + BH FDR | Handles unequal variance; controls false discovery rate across 99 queries | ✅ Locked |
| Regression threshold | ≥5% + p<0.05 | Dual requirement prevents noise-driven false positives | ✅ Locked |
| Provisioning | Pulumi (TypeScript) | Declarative, ephemeral, reproducible | ✅ Locked |
| Result format | JSONL per query per run | Enables per-query statistical analysis; platform-agnostic | ✅ Locked |
| Instance type | **PENDING** | CONFLICT-1 unresolved (m7g vs r6i) | ⚠️ Pending D4 |
| JVM version | **PENDING** | CONFLICT-2 unresolved (JDK 11 vs 17) | ⚠️ Pending |
| CI query set | **PENDING** | CONFLICT-3 — reduced subset size TBD | ⚠️ Pending |

---

## 11. Verification Status

| Artifact | Verified | Method |
|----------|----------|--------|
| `product/product-brief-v1.md` (344 lines) | ✅ | Direct file read; all 13 sections confirmed present |
| `experiments/methodology-v1.md` (449 lines) | ✅ | Direct file read; all 11 sections confirmed present |
| `infra/platform-brief-v1.md` (397 lines) | ✅ | Direct file read; all 10+ sections confirmed present |
| `qa/qa-strategy-v1.md` (373 lines) | ✅ | Direct file read; all 10 sections confirmed present |
| Cross-role conflicts identified | ✅ | Synthesized from direct brief comparison |
| No code written yet | ✅ | Workspace contains only markdown documents |
| No AWS resources provisioned | ✅ | Confirmed greenfield; no infra code exists |

---

*This report synthesizes the intake phase outputs of all four IC roles. The team is ready to move to execution as soon as the three founder-blocking items (D1, D2, D3) are resolved and the three CEO/CTO decisions (CONFLICT-1, -2, -3) are made.*
