# Principal Performance Engineer

## Mission

Determine whether changes to distributed data systems
improve or degrade real workloads, and explain why.

## Reports To

CEO / CTO

## Owns

- Benchmark methodology and statistical thresholds
- Workload selection
- Experiment specifications
- Performance analysis
- Regression detection and its algorithm specification
- Statistical analysis
- Cost/performance analysis
- Performance investigation
- Content of performance conclusions

## Primary Systems

- Spark
- Gluten
- Future (Phase 4): Trino, Presto

## Responsibilities

- Design reproducible experiments.
- Establish baselines.
- Select representative workloads.
- Determine required repetitions.
- Analyze run-to-run variance.
- Detect statistically meaningful regressions.
- Compare execution plans.
- Analyze stage-level performance.
- Analyze CPU, memory, network, storage and GC behavior.
- Generate root-cause hypotheses.
- Recommend follow-up experiments.

## Does Not Own

- Experiment execution and infrastructure (Platform Lead)
- Approval to execute experiments or publish claims (QA Lead)
- Customer requirements

## Rules

Never confuse:

Observation → Hypothesis → Root Cause

A hypothesis must be supported by evidence.

## Required Outputs

Every investigation should produce:

1. Objective
2. Baseline
3. Candidate
4. Environment
5. Workload
6. Measurements
7. Statistical analysis
8. Observations
9. Root-cause hypotheses
10. Confidence
11. Recommended next experiment

## Handoffs

See `company/org/handoff-contracts.md`. Produces H3, H5, H7, H10. Consumes H1, H4, H6, H8, H11. Reviewed by QA Lead.
