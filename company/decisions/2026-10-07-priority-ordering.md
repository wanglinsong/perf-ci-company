# Decision: Phase 1 Priority Ordering

**Date:** 2026-10-07  
**Owner:** CEO / CTO  
**Status:** Active

## Context

The company is in Phase 1; intake is complete and execution of first tasks has not started. The wedge validation document (`issue-0001-wedge-validation.md`) is still a draft. Four IC roles have first tasks, but they have different dependency profiles and risk levels. Intake completed 2026-10-03: CEO/CTO Intake Report v1, Product Brief v1, Experiment Methodology v1, Platform Brief v1, QA Strategy v1.

## Decision

Priority order for starting work:

1. **Product Lead** — Prepare discovery scripts and target list; the human founder conducts customer interviews
2. **Performance Lead** — Identify Gluten version pair (unblocks Platform spike + QA H4 review)
3. **Platform Lead** — Spike EC2 provisioning via Pulumi (can begin in parallel once version pair is identified)
4. **QA Lead** — H4 design review of experiment spec (after Performance Lead delivers)

## Rationale

- **S2 is the biggest wedge risk.** If no customers have this problem, none of the technical work matters. Product discovery must not be delayed by infrastructure or experiment setup. The human founder conducts the interviews; the Product Lead prepares scripts and targets as the first task.
- **Performance Lead's version pair identification unblocks Platform spike and QA H4 review.** The Platform Lead needs to know which Gluten versions to target when designing the ephemeral benchmark environment (Spark config, JVM flags, baseline storage keys). QA cannot begin design review without an experiment spec.
- **Platform Lead can start in parallel once the version pair is identified**, but should not wait for Product Lead results — infrastructure setup is independent of customer discovery.
- **QA H4 design review must follow** the experiment spec from Performance Lead; it gates whether the proof experiment proceeds to execution.

## Open Questions

- Should the Product Lead and Performance Lead coordinate on Q4 from the wedge doc (Gluten community contacts) to accelerate discovery call scheduling?
- Is m7g.4xlarge adequate for MVP, or should Platform spike include a 3-node cluster option as a fallback?

## Follow-up

Revisit this ordering once S2 results are available. If customer validation is negative, the priority shifts from execution to pivot analysis.
