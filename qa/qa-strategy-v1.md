# QA Strategy — Perf CI
**Version:** v1  
**Author:** QA Lead  
**Date:** 2026-10-03  
**Status:** DRAFT — pending alignment with performance-lead methodology brief and platform-lead architecture brief

---

## 0. Purpose and Scope

This document defines the Quality Assurance strategy for the Perf CI system. It covers:

- How performance experiment results are validated before any claim is published
- Key risks that could produce false positives or false negatives
- QA checkpoints at each stage of the experiment lifecycle
- Concrete, measurable result acceptance criteria
- Explicit dependencies on peer leads

This strategy does **not** define benchmark methodology (performance-lead owns that) or infrastructure provisioning (platform-lead owns that). QA's role is independent validation of results produced by those systems.

---

## 1. Guiding Principles

1. **Default to NEEDS WORK.** A performance claim is not approved until evidence answers all adversarial questions. Burden of proof is on the claimant.
2. **Never approve on a single run.** Reproducibility requires independent re-execution.
3. **AI-generated conclusions are hypotheses.** Any AI-generated root-cause explanation must be supported by observable, reproducible evidence before it becomes a claim.
4. **Zero tolerance for environment inequivalence.** Any difference in hardware, OS, JVM flags, Spark config, Gluten config, or dataset between baseline and candidate invalidates the comparison.
5. **Flag early, not late.** A methodology flaw caught at design costs nothing. The same flaw caught after a week of runs is expensive.

---

## 2. Experiment Validation Approach

### 2.1 Adversarial Question Set

For every experiment, QA must answer all nine questions with evidence. An unanswered question blocks the claim.

| # | Question | Evidence Required |
|---|----------|-------------------|
| 1 | Is the baseline valid? | Baseline run IDs, timestamps, config hash |
| 2 | Are the environments equivalent? | Hardware fingerprint, OS version, JVM flags, Spark/Gluten config diff |
| 3 | Is the result statistically meaningful? | p-value < 0.05, n ≥ 5 per configuration, confidence interval |
| 4 | Could measurement noise explain the result? | CV ≤ 5% within same configuration |
| 5 | Was the workload representative? | Query set, data scale, data distribution rationale |
| 6 | Could caching explain the result? | Cache-flush evidence or warm-cache protocol documented |
| 7 | Did configuration change between runs? | Config diff between baseline and candidate |
| 8 | Did hardware change between runs? | Hardware fingerprint comparison |
| 9 | Can another run reproduce the result? | Independent re-execution run IDs and measurements |

### 2.2 Validation Workflow

```
Experiment Design Review
        │
        ▼
Environment Equivalence Check  ◄── BLOCK if inequivalent
        │
        ▼
Baseline Stability Check       ◄── BLOCK if CV > 5%
        │
        ▼
Candidate Execution Review     ◄── BLOCK if protocol deviation
        │
        ▼
Statistical Analysis Review    ◄── BLOCK if p > 0.05 or n < 5
        │
        ▼
Independent Reproduction       ◄── BLOCK if not reproducible
        │
        ▼
AI Conclusion Review           ◄── BLOCK if unsupported by evidence
        │
        ▼
Validation Report + Verdict
```

### 2.3 Validation Report Template

Every experiment concludes with a structured validation report:

```markdown
# Validation Report — [Experiment ID]

## Adversarial Question Responses
- Baseline valid? [YES / NO / UNKNOWN — evidence]
- Environments equivalent? [YES / NO / UNKNOWN — evidence]
- Statistically meaningful? [YES / NO — p-value, n, CV]
- Noise explainable? [YES / NO — CV, confidence interval]
- Workload representative? [YES / NO — rationale]
- Caching ruled out? [YES / NO — evidence]
- Config unchanged? [YES / NO — diff]
- Hardware unchanged? [YES / NO — evidence]
- Reproduced independently? [YES / NO — run IDs]

## Findings
[Specific issues with evidence references]

## Verdict
APPROVED / NEEDS WORK / BLOCKED

## Required Fixes (if NEEDS WORK or BLOCKED)
[Specific, actionable — not vague]
```

---

## 3. Key Benchmarking Risks

### 3.1 Environment Risks (HIGH)

| Risk | Description | Mitigation |
|------|-------------|------------|
| **Hardware drift** | Baseline and candidate run on different physical nodes or VM sizes | Require hardware fingerprint in every run artifact; block comparison if fingerprints differ |
| **JVM flag divergence** | GC policy, heap size, or JIT flags differ between runs | Config hash must be identical; diff must be zero |
| **Spark/Gluten version skew** | Dependency version changes between baseline and candidate | Lock versions in experiment spec; verify at execution time |
| **OS-level interference** | Background processes, kernel updates, NUMA effects | Require dedicated execution nodes; record `/proc/loadavg` and CPU steal at run time |
| **Filesystem cache effects** | Warm page cache inflates read performance | Document cache-flush protocol; enforce it before every run |

### 3.2 Measurement Risks (HIGH)

| Risk | Description | Mitigation |
|------|-------------|------------|
| **JVM warmup artifacts** | Early iterations are slower due to JIT compilation | Require warmup runs excluded from measurement; document warmup protocol |
| **Single-run conclusions** | One run cannot distinguish signal from noise | Minimum n=5 runs per configuration; compute CV |
| **Clock resolution** | Wall-clock timing has OS scheduler jitter | Use monotonic clocks; report median and p95, not just mean |
| **Outlier contamination** | A single slow run inflates mean | Report median; flag and investigate outliers before excluding them |
| **Metric selection bias** | Choosing only favorable metrics post-hoc | Metrics must be declared in experiment spec before execution begins |

### 3.3 Statistical Risks (MEDIUM)

| Risk | Description | Mitigation |
|------|-------------|------------|
| **Underpowered test** | Too few runs to detect real difference | Power analysis before experiment; minimum n=5, prefer n=10 |
| **Multiple comparisons** | Testing many queries inflates false-positive rate | Apply Bonferroni correction or equivalent when testing > 5 queries |
| **Wrong test choice** | Parametric test on non-normal data | Check normality; use Mann-Whitney U if distribution is skewed |
| **p-hacking** | Running more iterations until p < 0.05 | Fix n before execution; do not add runs after seeing results |

### 3.4 Workload Risks (MEDIUM)

| Risk | Description | Mitigation |
|------|-------------|------------|
| **Non-representative queries** | Benchmark queries don't reflect real workload | Query set must be justified against real usage patterns (product-lead input required) |
| **Data scale mismatch** | Small data hides I/O bottlenecks; large data hides CPU bottlenecks | Document target data scale and rationale in experiment spec |
| **Data skew** | Uniform synthetic data doesn't stress real join/shuffle patterns | Use production-representative data distributions or document deviation |

### 3.5 AI Conclusion Risks (MEDIUM)

| Risk | Description | Mitigation |
|------|-------------|------------|
| **Plausible but unsupported root cause** | AI generates a convincing explanation not backed by execution plans or profiles | Require CPU profile, GC log, or shuffle metric to support any AI-generated root cause |
| **Confirmation bias** | AI explanation matches the expected narrative | QA independently reviews raw evidence before accepting AI conclusion |
| **Overgeneralization** | Single-query finding generalized to all workloads | Scope claims to the specific queries and data scale tested |

---

## 4. QA Checkpoints

QA participates at four stages. Each checkpoint has a defined gate — work does not proceed past a gate without QA sign-off.

### Checkpoint 1: Experiment Design Review (BEFORE execution)

**Trigger:** Performance Lead submits experiment specification  
**QA Actions:**
- Verify all nine adversarial questions are addressed in the spec
- Confirm metrics are declared before execution
- Confirm warmup protocol is specified
- Confirm minimum run count (n ≥ 5) is planned
- Confirm cache-flush protocol is specified
- Confirm hardware/environment equivalence plan exists
- Confirm statistical test is pre-selected and appropriate

**Gate:** QA signs off on design OR returns specific required changes  
**Output:** Design review comment on experiment spec  
**Block condition:** Any adversarial question unanswerable from the spec

---

### Checkpoint 2: Environment Equivalence Validation (AT execution start)

**Trigger:** Platform Lead signals environment is provisioned  
**QA Actions:**
- Compare hardware fingerprints (CPU model, core count, memory, storage type)
- Compare OS version and kernel parameters
- Compare JVM flags (heap, GC policy, JIT settings)
- Compare Spark configuration (executor count, memory, shuffle settings)
- Compare Gluten/plugin configuration and version
- Verify dataset identity (checksum or version tag)
- Verify cache-flush was executed

**Gate:** QA confirms environment equivalence OR blocks execution  
**Output:** Environment equivalence checklist (pass/fail per item)  
**Block condition:** Any non-trivial difference between baseline and candidate environments

---

### Checkpoint 3: Execution Integrity Review (DURING/AFTER execution)

**Trigger:** Raw run results are available  
**QA Actions:**
- Verify run count matches spec (n ≥ 5 per configuration)
- Compute CV for each configuration — flag if CV > 5%
- Check for outliers (> 2σ from median) — investigate before excluding
- Verify warmup runs are excluded from measurement
- Verify no configuration changes occurred mid-experiment
- Cross-reference raw per-run measurements against any summary statistics

**Gate:** QA confirms execution integrity OR requires re-run  
**Output:** Execution integrity checklist with raw CV values  
**Block condition:** CV > 5%, run count below spec, or evidence of mid-run config change

---

### Checkpoint 4: Result and Conclusion Validation (AFTER analysis)

**Trigger:** Performance Lead submits analysis and conclusions  
**QA Actions:**
- Verify statistical test was applied correctly (check inputs, not just p-value)
- Verify p-value < 0.05 for any claimed regression or improvement
- Verify confidence intervals are reported
- Verify multiple-comparison correction if > 5 queries tested
- Independently reproduce claimed regressions (re-run or re-analyze raw data)
- Review any AI-generated root-cause hypotheses for supporting evidence
- Verify claims are scoped to the tested workload and data scale

**Gate:** QA issues APPROVED / NEEDS WORK / BLOCKED verdict  
**Output:** Validation report (see Section 2.3 template)  
**Block condition:** Any automatic block trigger (see Section 5)

---

## 5. Automatic Block Triggers

Stop and escalate immediately if any of the following are true:

1. Performance Lead cannot provide raw per-run measurements (only summary statistics)
2. CV > 5% within the same configuration
3. Baseline and candidate ran on different hardware or different days without a controlled re-run
4. Statistical test was not applied, or p-value > 0.05 for a claimed regression
5. AI-generated root cause has no supporting execution plan, CPU profile, GC log, or shuffle metric
6. A "pass" was claimed on the first attempt without independent reproduction
7. Metrics were selected or changed after execution began
8. Run count is below the pre-specified minimum

**Escalation path:** QA Lead → CEO/CTO with written BLOCKED report

---

## 6. Result Acceptance Criteria

These are the minimum measurable bars for a performance claim to be approved. All criteria must be met simultaneously.

### 6.1 Statistical Criteria

| Criterion | Threshold | Measurement |
|-----------|-----------|-------------|
| Minimum runs per configuration | n ≥ 5 | Count of valid (non-warmup) runs |
| Within-configuration stability | CV ≤ 5% | Coefficient of variation of run times |
| Statistical significance | p < 0.05 | Two-sided t-test or Mann-Whitney U |
| Effect size | ≥ 5% relative change | Median candidate vs. median baseline |
| Confidence interval | Reported and non-overlapping | 95% CI for both configurations |

> **Rationale for 5% effect size floor:** Changes smaller than 5% are within typical measurement noise for distributed systems and should not be published as performance claims without extraordinary evidence.

### 6.2 Reproducibility Criteria

| Criterion | Threshold |
|-----------|-----------|
| Independent re-execution | At least one full re-run of the experiment by QA or an independent process |
| Direction consistency | All re-runs show the same direction of change (improvement or regression) |
| Magnitude consistency | Re-run median within 10% of original median |

### 6.3 Environment Criteria

| Criterion | Requirement |
|-----------|-------------|
| Hardware equivalence | Identical CPU model, core count, memory, storage type |
| OS equivalence | Identical OS version and kernel |
| JVM equivalence | Identical flags, heap size, GC policy |
| Spark/Gluten config | Zero diff between baseline and candidate configs |
| Dataset | Identical dataset version/checksum |

### 6.4 Documentation Criteria

| Criterion | Requirement |
|-----------|-------------|
| Raw measurements | Per-run timing data available (not just summary) |
| Config artifacts | Full config files for both baseline and candidate |
| Environment fingerprint | Hardware and software fingerprint recorded at run time |
| Warmup protocol | Documented and verifiable from run logs |
| Cache-flush evidence | Logged before each run |

---

## 7. Tooling and Process Dependencies

### 7.1 Dependencies on Performance Lead

QA cannot operate without the following from the Performance Lead:

| Dependency | Required For | Status |
|------------|-------------|--------|
| Experiment specification format | Checkpoint 1 design review | ⚠️ PENDING — methodology brief not yet available |
| Defined metric set (latency, throughput, CPU, GC, shuffle) | Metric validation | ⚠️ PENDING |
| Statistical test selection and rationale | Statistical review | ⚠️ PENDING |
| Warmup protocol definition | Execution integrity check | ⚠️ PENDING |
| Minimum run count and power analysis | Acceptance criteria calibration | ⚠️ PENDING |
| Raw per-run output format | Raw data cross-reference | ⚠️ PENDING |
| AI conclusion format and evidence requirements | AI conclusion review | ⚠️ PENDING |

**Action required:** QA Lead will refine Sections 6.1 and 6.2 once the performance-lead methodology brief is available. If the brief changes the minimum run count, effect size floor, or statistical test, acceptance criteria will be updated accordingly.

### 7.2 Dependencies on Platform Lead

QA cannot operate without the following from the Platform Lead:

| Dependency | Required For | Status |
|------------|-------------|--------|
| Hardware fingerprint format and collection mechanism | Environment equivalence check | ⚠️ PENDING — architecture brief not yet available |
| Config artifact storage location and format | Config diff at Checkpoint 2 | ⚠️ PENDING |
| Run log format and storage location | Raw data access | ⚠️ PENDING |
| Cache-flush mechanism and log evidence | Cache-flush verification | ⚠️ PENDING |
| Environment provisioning signal (when ready for Checkpoint 2) | Checkpoint 2 trigger | ⚠️ PENDING |
| Experiment run ID scheme | Run traceability | ⚠️ PENDING |

**Action required:** QA Lead will integrate with platform-lead's observability and artifact storage design once the architecture brief is available. QA tooling for environment equivalence checks will be built on top of platform-provided fingerprints.

### 7.3 Dependencies on Product Lead

| Dependency | Required For | Status |
|------------|-------------|--------|
| Workload representativeness rationale | Adversarial question #5 | ⚠️ PENDING |
| Definition of "real workload" for the target customer | Workload risk assessment | ⚠️ PENDING |

---

## 8. QA Tooling Plan

The following tooling is required to execute this strategy. Tooling will be built incrementally as platform-lead's infrastructure becomes available.

| Tool | Purpose | Owner | Status |
|------|---------|-------|--------|
| Environment equivalence checker | Diffs hardware fingerprints and config files between baseline and candidate | QA Lead (built on platform artifacts) | PLANNED |
| CV calculator | Computes coefficient of variation from raw run data | QA Lead | PLANNED |
| Statistical test runner | Applies t-test / Mann-Whitney U to raw measurements | QA Lead | PLANNED |
| Validation report generator | Produces structured validation report from checklist inputs | QA Lead | PLANNED |
| Regression gate | Automated check: blocks CI pipeline if acceptance criteria not met | QA Lead + Platform Lead | PLANNED |

---

## 9. Open Questions and Risks

| # | Question | Blocking? | Owner |
|---|----------|-----------|-------|
| 1 | What is the target data scale (GB/TB) for benchmarks? | Yes — affects workload risk assessment | Product Lead + Performance Lead |
| 2 | What distributed system is being benchmarked (Spark + Gluten? Other)? | Yes — affects environment equivalence checks | Performance Lead |
| 3 | Will experiments run on cloud VMs or bare metal? | Yes — affects hardware equivalence criteria | Platform Lead |
| 4 | What is the CI trigger model (PR-gated, nightly, on-demand)? | Yes — affects Checkpoint 2 timing | Platform Lead |
| 5 | Is there a pre-existing baseline dataset or will one be established? | Yes — affects baseline validity | Performance Lead |
| 6 | What is the acceptable regression threshold for CI gates? | Yes — affects acceptance criteria | CEO/CTO + Performance Lead |
| 7 | Will AI-generated root-cause analysis be part of the standard output? | Yes — affects AI conclusion review scope | Performance Lead |

---

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| v1 | 2026-10-03 | QA Lead | Initial draft — written before peer briefs available; assumptions explicit in Section 7 |

---

*This document will be revised once performance-lead's methodology brief and platform-lead's architecture brief are available. All PENDING items in Section 7 represent required inputs before QA checkpoints can be fully operationalized.*
