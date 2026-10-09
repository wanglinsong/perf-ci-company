# GPU Inference Host — Perf CI Agents

**Date:** 2026-10-08
**Escalation type:** H9 — Founder decision (FD-5)
**Status:** ✅ **DECIDED** — build after perf-ci-infra PR #1 merges

## Context

All OPC agents run model inference on a LAN Ollama host (qwen3.6, 64k context). Each agent turn takes 2–3 minutes. Measured: ~690 prompt tokens/s and ~65 output tokens/s. A CEO turn re-reads a ~40k-token prompt. A cloud GPU (g6e.4xlarge, 1× NVIDIA L40S 48 GB) should cut turns to tens of seconds and allow agents to run in parallel. That estimate is unbenchmarked.

## Founder Decision

1. **Budget:** the inference host is company operating cost, **not** part of perf-ci-infra work. It does **not** count toward the FD-2 $100 limit for the pilot and proof experiment. Track its spend separately. The founder sets its spend cap before the first `pulumi up`.
2. **Uptime:** the instance is **paused (EC2 stop, not terminate) when not in use**. It runs only while agents need it.

## Work Item (Platform Lead, after PR #1 merges)

**Scope:** a separate Pulumi stack under `infra/inference-node/` in perf-ci-infra. It must be independent of the benchmark-node stack and must not touch `infra/pulumi/`.

- **Instance:** g6e.4xlarge in `us-east-1`, AWS profile `perf-ci` (FD-4). Use a GPU AMI with the NVIDIA driver, e.g. AWS Deep Learning Base AMI. Root EBS must be large enough for the model weights, so they survive stop/start.
- **Model server:** vLLM, or Ollama, serving qwen3.6 or the closest vLLM-compatible build, with an OpenAI-compatible API. Enable prefix caching. Give the context window and the maximum number of parallel requests in the README.
- **Access:** no inbound ports. Reach the API only through an SSM port-forward from the founder's Mac, e.g. `aws ssm start-session --document-name AWS-StartPortForwardingSession`. Attach `AmazonSSMManagedInstanceCore`.
- **Pause when idle:** on the instance, stop it (`shutdown -h` with `instanceInitiatedShutdownBehavior: stop`) after N minutes with no API requests. N is configurable, default 30. Add scripts to start the instance and open the tunnel, and to stop it.
- **Cost:** tag every resource `Purpose: agent-inference` and `CostCenter: company-ops`. Show the hourly on-demand price (sourced) and the stopped-state EBS cost in the README.

**Done when:**

- `npm ci && npx tsc --noEmit` and `pulumi preview` pass. The real output must be in the work-item result; never simulate tool output.
- The README documents start, tunnel, stop, the idle auto-stop, and costs.
- No `pulumi up`: deploying needs the founder's explicit approval and spend cap.

**Out of scope for agents:** deploying, and switching OpenOPC to the new endpoint. The switch is founder-owned: benchmark the same 40k-token prompt on both hosts. If it's better, change `api_base` in `.opc/config/llm_config.yaml`, raise `OPC_COMPANY_MAX_CONCURRENT_TURNS` in `bin/opc`, and restart. The LAN host stays as the fallback.
