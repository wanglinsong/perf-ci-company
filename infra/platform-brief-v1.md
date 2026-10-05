# Platform Architecture Brief — v1
**Perf CI | Platform Engineering Lead**
**Date:** 2026-10-03
**Status:** Draft — Intake / Greenfield

---

## 1. Current Workspace State

| Dimension | Status |
|---|---|
| Workspace root | `/perf-ci/` |
| Existing infrastructure code | **None** — greenfield |
| Existing CI/CD pipelines | **None** |
| Existing cloud resources | **None** (assumed) |
| Existing benchmark data | **None** |
| Git history | Empty repository |

**Assessment:** This is a clean-slate build. No legacy constraints, no migration risk. Every infrastructure decision is a first-principles choice.

---

## 2. Product Context (from `issue-0001-wedge-validation.md`)

The first product milestone is **Gluten wedge validation**: determine whether Apache Gluten accelerates TPC-DS workloads on Spark 3.5 and by how much, with enough statistical rigor to support a customer-facing claim.

### MVP Experiment Specification

| Parameter | Value |
|---|---|
| Workload | TPC-DS SF=10 |
| Engine | Spark 3.5 + Gluten (latest stable) |
| Baseline | Spark 3.5 vanilla (no Gluten) |
| Instance type | m7g.4xlarge (single node) |
| Cluster topology | Single-node (no cluster for MVP) |
| Result storage | S3, keyed by Gluten version tag |
| CI trigger | GitHub Actions check on Gluten PR |
| Provisioning | Pulumi — fully ephemeral |
| Reporting | GitHub Actions summary, per-query latency comparison |

This specification drives every infrastructure priority in this brief.

---

## 3. Execution Environment Design

### 3.1 Instance Selection Rationale

**Primary:** `m7g.4xlarge` (AWS Graviton3, ARM64)

| Property | Value | Rationale |
|---|---|---|
| vCPUs | 16 | Sufficient for single-node Spark with parallelism headroom |
| Memory | 64 GiB | Fits TPC-DS SF=10 working set in memory; avoids disk spill noise |
| Network | Up to 12.5 Gbps | Adequate for S3 data reads at SF=10 |
| Storage | EBS gp3 (provisioned) | Deterministic IOPS; avoids instance-store variance |
| Architecture | ARM64 (Graviton3) | Eliminates Intel-specific JIT noise; consistent clock behavior |
| On-demand cost | ~$0.6720/hr | ~$0.80 total per SF=10 run (est. 70 min) |

**Escape hatch:** If a workload requires x86-specific JVM behavior or Gluten native libraries are not ARM-compatible, fall back to `m7i.4xlarge` (Intel Ice Lake). This must be documented as an environment variable in the experiment spec.

### 3.2 OS and System Configuration

| Layer | Choice | Pin Strategy |
|---|---|---|
| AMI | Amazon Linux 2023 (ARM64) | Pinned by AMI ID per experiment run |
| Kernel | AL2023 default | Recorded in run metadata |
| JVM | Amazon Corretto 17 (LTS) | Pinned by exact version string |
| Spark | 3.5.x | Pinned by artifact SHA |
| Gluten | Latest stable tag | Pinned by Git tag + artifact SHA |
| Python | 3.11 (for PySpark) | Pinned via conda/venv lockfile |

**Noise controls applied at boot:**
- CPU governor set to `performance` (disable frequency scaling)
- Transparent huge pages: `madvise`
- NUMA: single-node, no NUMA policy needed for m7g.4xlarge
- Swap: disabled
- JVM warmup: 1 dry run before timed measurement (methodology decision — deferred to Performance Lead)

### 3.3 Storage Layout

```
/data/tpcds/sf10/          # TPC-DS dataset (pre-loaded from S3 on boot)
/opt/spark/                # Spark installation
/opt/gluten/               # Gluten native library
/tmp/spark-work/           # Spark work dir (tmpfs or gp3 EBS)
/results/                  # Per-run output, uploaded to S3 on completion
```

### 3.4 Spark Configuration Baseline

```properties
# Resource allocation (single-node)
spark.master                        local[14]
spark.driver.memory                 48g
spark.driver.memoryOverhead         4g
spark.executor.memory               48g

# Determinism controls
spark.sql.shuffle.partitions        200
spark.sql.adaptive.enabled          false   # AQE disabled for reproducibility
spark.sql.adaptive.coalescePartitions.enabled  false

# Gluten plugin (when enabled)
spark.plugins                       io.glutenproject.GlutenPlugin
spark.gluten.sql.columnar.backend.lib  velox

# Result serialization
spark.sql.parquet.compression.codec snappy
```

> **Note:** Final Spark config is owned by Performance Lead. Platform provides the baseline and guarantees it is applied identically across runs. Any deviation from baseline must be declared in the experiment spec.

---

## 4. Key Infrastructure Components

### 4.1 Component Map

```
┌─────────────────────────────────────────────────────────────────┐
│                        GitHub Actions                           │
│  PR trigger → workflow dispatch → experiment-runner job         │
└────────────────────────┬────────────────────────────────────────┘
                         │ HTTP POST (experiment spec JSON)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Experiment Orchestrator                        │
│  (GitHub Actions job or lightweight Lambda)                     │
│  - Validates experiment spec                                    │
│  - Calls Pulumi to provision                                    │
│  - Polls for completion                                         │
│  - Triggers teardown                                            │
└────────────────────────┬────────────────────────────────────────┘
                         │ Pulumi up
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   AWS EC2 (m7g.4xlarge)                         │
│  Ephemeral benchmark node                                       │
│  - Provisioned fresh per run                                    │
│  - Runs benchmark harness via SSM or cloud-init                 │
│  - Emits results to S3                                          │
│  - Self-terminates on completion (TTL guard)                    │
└────────────────────────┬────────────────────────────────────────┘
                         │ Results upload
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                        AWS S3                                   │
│  s3://perf-ci-results/                                          │
│  ├── runs/{run-id}/                                             │
│  │   ├── spec.json          # Experiment spec (immutable)       │
│  │   ├── env.json           # Captured environment metadata     │
│  │   ├── results.json       # Per-query latencies               │
│  │   ├── spark.log          # Full Spark driver log             │
│  │   ├── system.json        # CPU/mem/disk metrics              │
│  │   └── cost.json          # AWS cost estimate for this run    │
│  └── baselines/             # Pinned baseline results by tag    │
│      └── gluten-{version}/results.json                         │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Component Descriptions

| Component | Technology | Responsibility |
|---|---|---|
| **Provisioner** | Pulumi (TypeScript) | Declare and apply EC2 + VPC + IAM + S3 resources |
| **Benchmark Node** | EC2 m7g.4xlarge | Run Spark + Gluten + TPC-DS harness |
| **Benchmark Harness** | Shell + PySpark | Execute queries, capture timing, emit results |
| **Result Store** | S3 | Immutable per-run artifact storage |
| **Orchestrator** | GitHub Actions | Trigger, monitor, report, teardown |
| **Cost Reporter** | AWS Cost Explorer API | Emit per-run cost estimate to `cost.json` |
| **TTL Guard** | EC2 instance tag + Lambda | Auto-terminate instances older than 4 hours |

### 4.3 IAM Boundary

The benchmark node runs with a minimal IAM instance profile:
- `s3:PutObject` on `s3://perf-ci-results/runs/*`
- `s3:GetObject` on `s3://perf-ci-datasets/*`
- No EC2 control plane permissions
- No cross-account access

Pulumi runs with a separate role that has EC2 + VPC + IAM provisioning rights, scoped to the `perf-ci` resource tag.

---

## 5. CI/CD Integration

### 5.1 GitHub Actions Workflow Design

```
Trigger: PR to apache/gluten (or perf-ci repo)
         Manual dispatch with experiment spec override

Jobs:
  validate-spec      → Parse and validate experiment spec JSON
  provision          → pulumi up (creates EC2 node)
  await-results      → Poll S3 for results.json (timeout: 90 min)
  compare-baseline   → Fetch baseline from S3, compute regression delta
  report             → Post GitHub Actions summary + PR comment
  teardown           → pulumi destroy (always runs, even on failure)
```

### 5.2 GitHub Actions Summary Format

```markdown
## Perf CI — TPC-DS SF=10 | Gluten {version} vs Baseline

| Query | Baseline (ms) | Gluten (ms) | Delta | Status |
|-------|--------------|-------------|-------|--------|
| Q1    | 1,240        | 890         | -28%  | ✅     |
| Q7    | 3,100        | 2,950       | -5%   | ✅     |
| Q19   | 8,400        | 9,100       | +8%   | ⚠️     |
| ...   | ...          | ...         | ...   | ...    |

**Geometric mean speedup:** 1.18×
**Run cost:** $0.82
**Environment:** m7g.4xlarge | AL2023 | Corretto 17 | Spark 3.5.x | Gluten {sha}
**Run ID:** run-20261003-abc123
**Artifacts:** [S3 link]
```

### 5.3 Pass/Fail Gate (placeholder — methodology TBD by Performance Lead)

The platform will enforce a configurable gate. Default (placeholder):
- **FAIL** if any query regresses > 10% vs baseline
- **WARN** if geometric mean speedup < 1.0×
- **PASS** otherwise

> **Dependency:** Final regression thresholds and statistical criteria are owned by Performance Lead. Platform will implement whatever gate spec is provided.

---

## 6. Experiment Lifecycle

```
Phase 1: PROVISION (est. 4 min)
  - Pulumi up: VPC, security group, IAM role, EC2 instance
  - cloud-init: install JVM, Spark, Gluten, dataset sync from S3
  - Health check: SSH/SSM reachable, Spark version confirmed

Phase 2: EXECUTE (est. 55–70 min at SF=10)
  - Dry run (1×): JVM warmup, filesystem cache warm
  - Timed run (1×): all TPC-DS queries, per-query timing captured
  - System metrics: CPU, memory, disk I/O sampled every 30s

Phase 3: COLLECT (est. 2 min)
  - results.json, env.json, cost.json, spark.log uploaded to S3
  - Run ID stamped on all artifacts

Phase 4: TEARDOWN (est. 90 sec)
  - pulumi destroy: EC2 terminated, EBS deleted, security group removed
  - VPC retained (shared, not per-run) — or destroyed if isolated VPC chosen
  - Teardown confirmation written to S3 run record

Total wall time: ~65–80 min
Total cost per run: ~$0.80 (est.)
```

---

## 7. Cost Controls

| Control | Mechanism | Threshold |
|---|---|---|
| Per-run spend cap | Instance TTL tag + Lambda terminator | 4 hours max runtime |
| Monthly budget alert | AWS Budgets | $200/month alert |
| Cost per run reporting | Cost Explorer API in teardown phase | Emitted to `cost.json` |
| Spot instance option | Pulumi parameter `use_spot=true` | Available for non-gated runs |
| Resource tagging | All resources tagged `project=perf-ci`, `run-id={id}` | Enables cost attribution |

**Estimated cost breakdown per run (SF=10, m7g.4xlarge on-demand):**

| Item | Duration | Rate | Cost |
|---|---|---|---|
| EC2 m7g.4xlarge | 75 min | $0.6720/hr | $0.84 |
| EBS gp3 100GB | 75 min | $0.08/GB-mo | $0.01 |
| S3 PUT/GET | ~500 ops | negligible | <$0.01 |
| **Total** | | | **~$0.86** |

---

## 8. Reproducibility Guarantees

Every experiment run records and pins the following to ensure bit-for-bit reproducibility:

| Variable | Control Method |
|---|---|
| AMI | Pinned AMI ID in Pulumi config |
| JVM version | Exact version string in cloud-init |
| Spark version | Artifact SHA in cloud-init |
| Gluten version | Git tag + native library SHA |
| Dataset | S3 URI + ETag (immutable) |
| Spark config | Embedded in experiment spec JSON |
| OS kernel | Recorded in `env.json` at runtime |
| CPU governor | Set to `performance` in cloud-init |
| Swap | Disabled in cloud-init |
| THP | Set to `madvise` in cloud-init |

**Variance target:** CV ≤ 5% across back-to-back identical runs. Any run pair exceeding this threshold is flagged as a reproducibility failure and escalated as P0.

---

## 9. Prioritized Build List (Sprint 1)

Priority order is set to unblock the first experiment run as fast as possible.

### P0 — Unblocks First Run

| # | Deliverable | Output | Est. Effort |
|---|---|---|---|
| 1 | **Pulumi EC2 provisioner spike** | `infra/pulumi/` — VPC + EC2 + IAM + S3 bucket | 2 days |
| 2 | **cloud-init bootstrap script** | `infra/scripts/bootstrap.sh` — JVM, Spark, Gluten install | 1 day |
| 3 | **Benchmark harness (shell)** | `infra/scripts/run-tpcds.sh` — query execution + timing | 1 day |
| 4 | **Result schema + S3 upload** | `infra/scripts/collect-results.sh` + `infra/schema/results.json` | 0.5 day |
| 5 | **GitHub Actions workflow (manual trigger)** | `.github/workflows/perf-run.yml` | 1 day |

### P1 — Required for CI Gate

| # | Deliverable | Output | Est. Effort |
|---|---|---|---|
| 6 | **Baseline comparison script** | `infra/scripts/compare-baseline.py` | 1 day |
| 7 | **GitHub Actions PR integration** | `.github/workflows/perf-pr-check.yml` | 1 day |
| 8 | **Cost reporting** | `infra/scripts/emit-cost.py` | 0.5 day |
| 9 | **TTL guard Lambda** | `infra/lambda/ttl-guard/` | 1 day |

### P2 — Hardening and Observability

| # | Deliverable | Output | Est. Effort |
|---|---|---|---|
| 10 | **Reproducibility validation** | Back-to-back run comparison, CV check | 1 day |
| 11 | **System metrics collection** | `infra/scripts/collect-metrics.sh` (CPU/mem/disk) | 0.5 day |
| 12 | **Pulumi destroy validation** | Automated teardown confirmation + S3 record | 0.5 day |
| 13 | **AWS Budgets alert** | `infra/pulumi/budget.ts` | 0.5 day |

### P3 — Phase 2 (Post-MVP)

| # | Deliverable | Notes |
|---|---|---|
| 14 | EKS cluster provisioner | Multi-node experiments |
| 15 | Spot instance support | Cost reduction for non-gated runs |
| 16 | Experiment queue / scheduler | Parallel experiment runs |
| 17 | Metrics dashboard | Grafana or CloudWatch |

---

## 10. Open Dependencies and Risks

### Dependencies on Other Roles

| Dependency | From | Blocking? | Status |
|---|---|---|---|
| Final Spark config baseline | Performance Lead | Yes — needed before P0-3 | ⏳ Pending |
| Regression gate thresholds | Performance Lead | Yes — needed before P1-7 | ⏳ Pending |
| TPC-DS dataset location / format | Performance Lead | Yes — needed before P0-2 | ⏳ Pending |
| Gluten artifact source (Maven / GitHub Releases) | Performance Lead | Yes — needed before P0-2 | ⏳ Pending |
| AWS account + credentials | CEO/CTO | Yes — needed before P0-1 | ⏳ Pending |
| GitHub repo + Actions secrets | CEO/CTO | Yes — needed before P0-5 | ⏳ Pending |

### Open Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Gluten native libs not available for ARM64 | Medium | High — forces x86 fallback | Verify Gluten ARM64 release artifacts before committing to m7g |
| TPC-DS SF=10 dataset generation time | Low | Medium — adds 30 min to first run | Pre-generate and cache in S3 |
| CV > 5% on Graviton3 due to thermal throttling | Low | High — reproducibility failure | Pin CPU governor; add thermal monitoring |
| AWS account limits (EC2 m7g.4xlarge quota) | Low | Medium — blocks provisioning | Request quota increase proactively |
| Gluten query failures on subset of TPC-DS | Medium | Medium — partial results | Harness must handle per-query failures gracefully |

---

## 11. Immediate Next Actions

1. **Verify Gluten ARM64 artifact availability** — confirm m7g.4xlarge is viable before writing Pulumi scripts
2. **Obtain AWS account credentials and region** — required to run `pulumi up`
3. **Sync with Performance Lead** on: Spark config baseline, TPC-DS dataset source, Gluten version pin, regression gate spec
4. **Begin Pulumi EC2 provisioner spike** (P0-1) — can start with placeholder values, finalize once dependencies resolved
5. **Stand up S3 bucket structure** — can be done immediately once AWS creds available

---

## 12. Decisions Log

| Decision | Rationale | Escape Hatch |
|---|---|---|
| m7g.4xlarge (ARM64) | Eliminates Intel JIT noise; consistent Graviton3 clock behavior | Switch to m7i.4xlarge if Gluten ARM64 not available |
| Single-node (no cluster) | MVP scope; eliminates network variance; simpler provisioning | EKS multi-node in Phase 2 |
| Pulumi (TypeScript) | Type-safe IaC; good AWS provider; team familiarity assumed | Terraform if Pulumi blocked |
| Ephemeral per-run provisioning | Eliminates noisy-neighbor and state accumulation variance | Persistent dev cluster for iteration only, never for measurement |
| AQE disabled | Removes adaptive plan changes that introduce run-to-run variance | Re-enable only if Performance Lead explicitly requests it |
| S3 for result storage | Durable, cheap, versioned, accessible from GitHub Actions | RDS for structured querying in Phase 2 |

---

*This brief will be updated as dependencies are resolved and the first experiment run is completed. Version 2 will incorporate Performance Lead methodology brief inputs.*

**Author:** Platform Engineering Lead
**Next review:** After Performance Lead methodology brief is available
