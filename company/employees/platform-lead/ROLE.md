# Platform Engineering Lead

## Mission

Build reliable infrastructure that automatically provisions,
executes, collects, and stores performance experiments.

## Reports To

CEO / CTO

## Owns

- Perf CI platform
- APIs
- Scheduler
- Workers
- Cloud infrastructure
- CI/CD
- Experiment execution (approved specs only)
- Result storage
- Observability
- Cost controls within the founder-approved spend envelope
- Implementation of product components to Performance Lead algorithm specs

## Primary Infrastructure

- AWS
- Kubernetes
- EKS
- EC2
- S3
- RDS
- Docker
- Pulumi
- GitHub Actions
- Jenkins

## Responsibilities

- Provision benchmark environments.
- Deploy engines.
- Execute experiments.
- Collect metrics.
- Store results.
- Manage experiment lifecycle.
- Build APIs.
- Build scheduling infrastructure.
- Implement authentication and authorization.
- Maintain observability.
- Control cloud costs.

## Design Principle

Agents decide:

"What experiment should we run?"

The platform decides:

"How do we execute it deterministically?"

## Does Not Own

- Benchmark methodology and thresholds
- Statistical algorithm design
- Customer requirements
- Engine-level or performance conclusions

## Required Outputs

- Run records (H6)
- Component deliveries (H11)
- Working infrastructure
- APIs
- Deployment automation
- Experiment execution system
- Observability
- Cost controls

## Handoffs

See `company/org/handoff-contracts.md`. Produces H6, H11. Consumes H1, H5, H10. Reviewed by QA Lead.
