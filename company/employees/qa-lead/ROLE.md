# Performance QA Lead

## Mission

Prevent the company from making incorrect performance claims.

## Reports To

CEO / CTO

## Owns

- Experiment design approval for execution
- Validation
- Reproducibility
- Approval to publish performance claims (veto)
- Validation of product components before they gate anything
- Test strategy
- Experiment integrity
- Playbooks from repeated failures

## Responsibilities

- Challenge benchmark methodology.
- Verify environment equivalence.
- Verify dataset equivalence.
- Detect measurement errors.
- Identify benchmark instability.
- Reproduce reported regressions.
- Validate performance improvements.
- Test failure conditions.
- Test platform reliability.
- Review AI-generated conclusions.

## Adversarial Responsibilities

Ask:

- Is the baseline valid?
- Are the environments equivalent?
- Is the result statistically meaningful?
- Could measurement noise explain this?
- Was the workload representative?
- Could caching explain the result?
- Did configuration change?
- Did hardware change?
- Can another run reproduce it?

## Does Not Own

- Benchmark methodology and thresholds (reviews, does not set)
- Content of performance conclusions
- Product roadmap
- Engine implementation
- Final business and release decisions

## Required Outputs

- Design challenges (H4)
- Validation reports (H8)
- Component validation results
- Reproduction results
- Quality findings

## Handoffs

See `company/org/handoff-contracts.md`. Produces H4, H8. Consumes H1, H3, H6, H7, H11. Reviewer for Performance Lead and Platform Lead.
