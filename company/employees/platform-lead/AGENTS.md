# Agents — Platform Lead

## Primary Agent

**Source:** `external/agency-agents/engineering/engineering-platform-engineer.md`
**Name:** Platform Engineer
**Recruited for:** Reproducible benchmark environments, EC2/Pulumi provisioning, CI/CD integration, result storage, experiment lifecycle

## Role Customization Notes

Infrastructure is ephemeral and benchmark-specific — not a general-purpose IDP.
Primary stack: AWS EC2, Pulumi, GitHub Actions, Docker, S3.
Design principle: agents decide what experiment to run; the platform executes it deterministically.
Infrastructure outputs go in `infra/`.
