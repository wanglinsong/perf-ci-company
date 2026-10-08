# ARM vs x86 Decision — Perf CI MVP

**Date:** 2026-10-07  
**Escalation type:** H9 — Founder decision  
**Status:** ✅ **DECIDED**

## Decision Requested

Which instance architecture to use for the MVP proof experiment and initial benchmark baseline.

## Options

| Option | Instance | Pros | Cons |
|--------|----------|------|------|
| **A — m7g.4xlarge (ARM/Graviton)** | AWS EC2 m7g.4xlarge | Lower cost (~30% vs x86), aligns with industry trend, Gluten has Velox native deps that benefit from ARM | Results labelled ARM-only; may not generalize to x86 production deployments |
| B — c6i.4xlarge (x86) | AWS EC2 c6i.4xlarge | Directly matches most current customer production hardware | Higher cost, slower industry trajectory |

## CEO/CTO Recommendation

**Option A — m7g.4xlarge (ARM), results labelled ARM-only.**

Rationale:
- The MVP's goal is technical feasibility validation (S1), not production benchmarking. ARM provides a valid proof of concept at lower cost.
- Gluten's Velox backend has native code that runs on both architectures; the detection mechanism (regression flagging) is architecture-independent even if absolute latency numbers are not.
- We can add x86 baseline in Phase 2 once S1 is validated and customer interviews confirm x86 dominance in their deployments.
- The wedge doc already specified m7g.4xlarge as the MVP instance type.

## Founder Decision

**Approved Option A — m7g.4xlarge (ARM), results labelled ARM-only.**

## Notes

- All benchmark results from this experiment will be clearly labelled "ARM (m7g.4xlarge) only — not generalizable to x86."
- Phase 2 planning should include x86 baseline comparison if customer interviews confirm x86 production dominance.
- This decision does not preclude adding x86 instances later; it only sets the MVP scope.
