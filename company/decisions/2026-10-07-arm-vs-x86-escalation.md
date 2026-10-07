# Escalation: ARM (m7g) vs x86 Benchmark Generalization

**Date:** 2026-10-07  
**Owner:** CEO / CTO  
**Status:** Open — awaiting founder decision

## Decision Requested

Should the MVP benchmark use ARM-based m7g.4xlarge instances (as currently specified) or x86-based instances, and how should we handle generalization to x86 production deployments?

## Options

### Option A: Keep m7g.4xlarge (ARM/Graviton) — Current Spec
- **Pros:** Lower cost (~30–40% cheaper than equivalent x86), aligns with industry trend toward ARM for data workloads, faster benchmark execution on Graviton 3
- **Cons:** Results may not generalize to x86 production deployments; Gluten performance characteristics (vectorized code paths, JIT compilation) can differ significantly between architectures

### Option B: Use x86 (e.g., c7i.4xlarge or c7a.4xlarge)
- **Pros:** Broader compatibility with current Spark/Gluten production deployments (most are still x86), results generalize directly to majority of target customers
- **Cons:** Higher cloud cost, slower benchmark execution

### Option C: Dual-architecture MVP (ARM + x86 comparison)
- **Pros:** Establishes baseline cross-architecture delta early; builds credibility with mixed-architecture customers
- **Cons:** Doubles infrastructure cost and execution time (~4 hours vs ~2 hours); may delay wedge validation

## Evidence

- MVP scope specifies m7g.4xlarge (ARM Graviton 3)
- Gluten is a Spark accelerator with architecture-sensitive code paths (vectorized expressions, JIT compilation, SIMD intrinsics)
- Industry trend: AWS Graviton adoption growing in data workloads, but x86 still dominates enterprise Spark deployments
- Target customers' production environments are unknown at this stage (S2 not yet validated)

## Cost Impact

> **All cost figures below are unverified estimates.** Actual cloud spend will differ based on instance pricing, data transfer, and run duration. See reconciliation note below.

- m7g.4xlarge vs c7i.4xlarge: ~$0.31/hr vs ~$0.50/hr (on-demand) — **unverified estimate**
- Per proof experiment (2 versions × 5 runs each + provisioning): ~$15–25 on ARM vs ~$25–40 on x86 — **unverified estimate**
- Dual architecture would roughly double this to ~$50–80 — **unverified estimate**

### Reconciliation with Intake Report

The CEO/CTO Intake Report v1 (`company/CEO-CTO-intake-report-v1.md`) cites a per-run cost of **~$0.86/run** for the proof experiment, which is significantly lower than the $15–40 range above. The discrepancy likely reflects different assumptions about run duration (the intake report may assume shorter warm-up/provisioning time), instance pricing tier (spot vs on-demand), or data transfer costs. These figures are unverified estimates pending actual cloud spend tracking.

## Recommendation

**Option A with a caveat:** Keep m7g.4xlarge for the proof experiment to de-risk S2 quickly and cheaply. Add a note in all artifacts that results are ARM-specific and generalization to x86 is unvalidated. Once S2 confirms customer interest, include x86 comparison in the MVP scope before any performance claims are published.

**Deadline:** Before starting Platform spike on EC2 provisioning (within 1 week). This decision affects the Pulumi infrastructure design.
