# Product Brief v1 — Perf CI
**Status:** Draft  
**Author:** Product Lead  
**Date:** 2026-10-03  
**Version:** 1.0

---

## Executive Summary

Platform and data engineering teams that run Apache Gluten in production have no automated, reliable signal that a Gluten version bump or configuration change degraded query performance before it reaches production. They discover regressions through user complaints, SLA breaches, or manual spot-checks — all of which are expensive and slow. Perf CI is a continuous performance regression detection service, delivered as a GitHub Actions check, that gives these teams a trustworthy performance gate on every Gluten-related change.

---

## 1. Customer Persona

### Primary: Gluten Platform Engineer

**Who they are:**  
A senior engineer (IC4–IC6 equivalent) on a data platform or infrastructure team at a company running Apache Gluten in production on top of Apache Spark. They maintain an internal Gluten fork, track upstream Gluten releases, and are responsible for upgrading Gluten versions across the fleet without breaking production SLAs.

**Where they work:**  
Companies confirmed to run Gluten in production include Microsoft, Google, IBM, Alibaba Cloud, ByteDance, Uber, Tencent, Meituan, Baidu, Pinterest, Oracle, Kyligence, BIGO, Netease, and Intel (15+ production adopters per the Apache Gluten project homepage, as of 2026). The Gluten GitHub repo has 1,600+ stars and 660+ forks, indicating a substantial active engineering community.

**What they care about:**  
- Shipping Gluten upgrades without production incidents
- Catching performance regressions before they affect downstream data pipelines and SLAs
- Not spending engineering time on manual benchmarking before every release
- Having a defensible, reproducible performance record they can share with stakeholders

**Their current frustration (verbatim-style):**  
> "We upgraded Gluten last quarter and TPC-DS Q72 ran 40% slower than vanilla Spark. We didn't catch it until users complained. Now we run a manual benchmark before every upgrade, but it takes two days and nobody owns it consistently."

*(Composite from Gluten GitHub issue #8417: "TPCDS q72 uses gluten calculation much worse than spark" — flagged as a critical performance issue in the Jan 2026 weekly status update.)*

### Secondary: Spark Performance Engineer / Gluten Contributor

**Who they are:**  
An engineer contributing to the Apache Gluten open-source project or an internal Gluten fork. They need to validate that their PRs don't introduce performance regressions before merging.

**What they care about:**  
- Fast feedback on PR performance impact (hours, not days)
- Reproducible benchmark results they can cite in PR reviews
- Not blocking other contributors with slow or flaky performance gates

---

## 2. Problem Statement

### The Core Pain

**Teams that operate Gluten in production have no automated performance gate.** When they upgrade Gluten versions, apply patches, or change Spark/Gluten configurations, they have no reliable, automated way to know whether query performance regressed before the change reaches production.

### Current Workflow (What They Do Today)

1. **Upgrade decision:** Team decides to upgrade Gluten (e.g., from 1.2.0 → 1.3.0) or apply a patch.
2. **Manual benchmark (if it happens):** One engineer manually runs TPC-DS or TPC-H on a dev cluster. This takes 1–3 days to set up and run. Results are stored in a spreadsheet or Confluence page.
3. **Comparison:** Engineer eyeballs the numbers against a previous run. No statistical significance testing. No automated comparison.
4. **Judgment call:** Team decides "looks OK" based on intuition and time pressure.
5. **Deploy to staging:** Change goes to staging. If something looks wrong, it gets caught here — or not.
6. **Production incident:** If a regression slipped through, it surfaces as user complaints, SLA breaches, or a data pipeline slowdown. Root cause analysis takes hours to days.
7. **Post-mortem:** Team resolves to "add better benchmarking." The cycle repeats.

### Specific Pain Points (Prioritized)

| # | Pain Point | Severity | Frequency |
|---|-----------|----------|-----------|
| 1 | **No automated regression signal on Gluten changes** — teams discover regressions in production | Critical | Every upgrade cycle (monthly–quarterly) |
| 2 | **Manual benchmarking is inconsistent and unowned** — results vary by who runs it, on what hardware, with what data | High | Every manual benchmark attempt |
| 3 | **No statistical rigor** — teams can't distinguish real regressions from benchmark noise | High | Every comparison |
| 4 | **No historical baseline** — no queryable record of "what did performance look like at version X?" | Medium | Post-incident investigation |
| 5 | **Benchmark setup is expensive** — generating TPC-DS data, configuring Spark+Gluten, and running 99 queries takes significant engineering time | Medium | Initial setup per team |
| 6 | **No PR-level feedback for contributors** — Gluten contributors don't know if their change regressed performance until after merge | Medium | Per PR (for active contributors) |

### Evidence

- **Gluten GitHub issue #8417** (open as of Jan 2026): "TPCDS q72 uses gluten calculation much worse than spark" — flagged as a *critical performance issue* requiring immediate attention in the project's own weekly status bot.
- **Gluten weekly status (Jan 2026)**: 47 PRs and 24 issues in one week; active performance regressions and OOM issues cited as priority items. The project is moving fast with no automated performance gate.
- **Zhihu case study**: Zhihu runs 10,000+ daily Spark SQL jobs and adopted Gluten after extensive manual validation. Initial experiments revealed "serious shuffle-stability problems" — caught only through manual testing.
- **Apache Comet** (the DataFusion-based Spark accelerator) runs TPC-DS/TPC-H benchmarks manually on EC2 and publishes results in a static file — no automated PR-level regression detection.
- **Gluten's own `gluten-it` tool** provides a CLI for benchmarking but requires manual invocation and produces no automated comparison or historical tracking.

---

## 3. Product Hypothesis

### The Bet

> **If we give Gluten platform teams a GitHub Actions check that automatically runs TPC-DS SF=10 on every Gluten-related change and reports statistically significant regressions as a PR status check, they will adopt it as their performance gate and pay for reliable, dedicated execution infrastructure.**

### Why Now

1. **Gluten has crossed the production threshold.** 15+ named production adopters (Microsoft, Google, Uber, ByteDance, etc.) means there are real teams with real SLAs depending on Gluten performance. The stakes are high enough to pay for tooling.
2. **The project is moving fast.** 47 PRs in one week means the regression surface is large and growing. Manual benchmarking cannot keep up.
3. **The general-purpose CI benchmarking market is nascent but proving out.** CodSpeed ($15/user/month, Techstars-backed) and Bencher ($100/month flat) have demonstrated willingness-to-pay for automated performance regression detection in CI — but neither supports Spark/Gluten workloads. The gap is real.
4. **No incumbent owns this space for data infrastructure.** Datadog's Spark observability is production-runtime focused, not pre-production regression detection. spark-perf-lint is static analysis only. There is no "CodSpeed for Spark/Gluten."

---

## 4. Competitive Landscape

### Direct Competitors (General CI Benchmarking)

| Tool | Positioning | Pricing | Spark/Gluten Support | Gap |
|------|------------|---------|---------------------|-----|
| **CodSpeed** | Continuous benchmarking for any language; bare-metal runners; PR checks | Free (OSS) / $15/user/month / Enterprise custom | ❌ None | No JVM/Spark workload support; runners optimized for micro-benchmarks, not 30-min TPC-DS runs |
| **Bencher** | Bare-metal CI benchmarking; language-agnostic; self-hostable | Free (OSS) / $100/month flat / Enterprise custom | ❌ None | Same gap; no Spark-aware statistical model; 5-min job timeout on free tier |

### Adjacent Tools (Spark-Specific, Not Regression Detection)

| Tool | What It Does | Gap |
|------|-------------|-----|
| **Datadog Spark integration** | Production runtime observability; job-level metrics | No pre-production regression detection; no TPC-DS baseline comparison |
| **Gluten `gluten-it` CLI** | Manual TPC-DS/TPC-H benchmarking for Gluten | No automation, no historical tracking, no PR integration, no statistical analysis |
| **Apache Comet benchmarks** | Manual TPC-DS on EC2; results in static files | No CI integration; no regression detection; Comet-specific, not Gluten |
| **spark-perf-lint** | Static analysis linter for PySpark anti-patterns | No runtime performance measurement |
| **IBM spark-tpc-ds-performance-test** | TPC-DS benchmark scripts for Spark | No CI integration; no regression detection |

### Competitive Conclusion

The market has general-purpose CI benchmarking tools (CodSpeed, Bencher) that prove willingness-to-pay, and Spark-specific tools that prove the problem exists — but no tool that combines automated CI integration with Spark/Gluten-aware workloads and statistical regression detection. **This is the gap Perf CI fills.**

---

## 5. MVP Feature List

The MVP is the narrowest system a real Gluten platform team will change their workflow to use. It must deliver a trustworthy performance gate with minimal setup friction.

### MVP Features (Prioritized)

**F1 — GitHub Actions Workflow Integration** *(Must-have)*  
A reusable GitHub Actions workflow (`.github/workflows/perf-ci.yml`) that teams add to their Gluten fork or Spark+Gluten repo with minimal configuration. Triggers on PR or push to main. Outputs a pass/fail status check on the PR.

*Why:* GitHub Actions is the default CI surface for Gluten contributors and most platform teams. Zero-friction integration is the adoption gate.

**F2 — TPC-DS SF=10 Benchmark Execution** *(Must-have)*  
Automated execution of a representative TPC-DS query subset (e.g., 20–30 queries covering the most regression-sensitive patterns) at Scale Factor 10 on a single-node Spark + Gluten environment. Produces per-query wall-clock latency results.

*Why:* TPC-DS is the industry-standard benchmark for Spark/Gluten (used by Gluten's own performance claims: "3.02x TPC-DS speedup"). SF=10 is large enough to be meaningful, small enough to run in under 60 minutes on a single node. The Gluten project itself uses TPC-DS for performance validation.

**F3 — Statistical Regression Detection** *(Must-have)*  
Comparison of current run results against a stored baseline (last known-good commit on main). Flags queries where the performance change exceeds a configurable threshold with statistical significance (e.g., >5% regression with p<0.05). Reports per-query results in the PR comment.

*Why:* Without statistical rigor, teams can't trust the signal. Noise-driven false positives destroy adoption. This is the core value differentiator vs. manual eyeballing.

**F4 — Baseline Storage and History** *(Must-have)*  
Persistent storage of benchmark results keyed by commit SHA, branch, and Gluten version. Enables comparison against any historical baseline, not just the immediately preceding run.

*Why:* Teams need to answer "when did this regress?" not just "did this regress?" Historical baselines also enable trend analysis and release-over-release comparison.

**F5 — PR Comment Report** *(Must-have)*  
Automated PR comment summarizing: overall pass/fail, per-query latency delta vs. baseline, queries that regressed beyond threshold, and a link to full results. Human-readable, actionable.

*Why:* The PR comment is the primary UX surface. If the report is confusing or noisy, engineers will ignore it. Clear signal → adoption.

---

## 6. Out of Scope for MVP

- Multi-node cluster benchmarking (single-node first)
- TPC-H support (TPC-DS first)
- Gluten ClickHouse backend (Velox backend first)
- Custom workload support (TPC-DS only for MVP)
- SaaS dashboard / web UI (GitHub Actions output only)
- Alerting / notification integrations (Slack, PagerDuty)
- Cost attribution or resource optimization recommendations
- Support for non-Gluten Spark accelerators (Comet, Photon)

---

## 7. Pricing Hypothesis

### Hypothesis

> **Teams will pay $200–$500/month for a managed Perf CI service that provides dedicated, noise-isolated benchmark runners and hosted result storage. Open-source projects get a free tier.**

### Rationale

- **CodSpeed** charges $15/user/month for general CI benchmarking with bare-metal runners at $0.032/min. A 60-minute TPC-DS run costs ~$1.92/run. At 2 runs/day, that's ~$115/month in runner costs alone — before the per-user seat fee.
- **Bencher** charges $100/month flat for up to 250 benchmark series. This is the closest pricing analog for a flat-fee model.
- **The Gluten use case requires dedicated hardware.** Standard GitHub Actions runners (2-core, 7GB RAM) cannot run a meaningful Spark+Gluten benchmark. Self-hosted runners or dedicated cloud instances are required. This is a real infrastructure cost that justifies a managed service premium.
- **The customer's alternative cost** is 1–3 days of senior engineer time per upgrade cycle. At $200–300/hour fully-loaded, even one avoided incident pays for a year of the service.

### Pricing Tiers (Hypothesis, Not Validated)

| Tier | Price | Target | Limits |
|------|-------|--------|--------|
| **Open Source** | Free | OSS Gluten contributors | Public repos only; shared runners; 3-month history |
| **Team** | $299/month | Platform teams at mid-size companies | Private repos; dedicated runners; unlimited history; 5 concurrent runs |
| **Enterprise** | Custom (~$1,500–3,000/month) | Large adopters (Microsoft, Uber, ByteDance scale) | On-prem runner support; SSO; SLA; custom workloads; API access |

*Note: These tiers are hypotheses. Validation requires customer interviews and willingness-to-pay probes.*

---

## 8. Success Criteria

### MVP Validation (Before Full Build)

| Signal | Target | Method |
|--------|--------|--------|
| **S1 — Problem confirmation** | 5+ Gluten platform engineers independently describe the regression detection pain without prompting | Customer discovery interviews |
| **S2 — Workflow adoption** | 3+ teams add the GitHub Actions workflow to their repo and run it on a real PR | Pilot program with 3–5 target teams |
| **S3 — Willingness to pay** | 2+ teams express intent to pay $200+/month for managed runners + history | Pricing conversation in discovery interviews |
| **S4 — Signal trust** | Teams act on a regression flag (investigate, revert, or consciously accept) rather than ignoring it | Behavioral observation during pilot |

### Post-Launch (90 Days)

| Metric | Target |
|--------|--------|
| Active repos running Perf CI | 10+ |
| Paying customers | 3+ |
| Regressions caught before production | ≥1 confirmed customer story |
| False positive rate (noise-driven flags) | <10% of all flags |
| Benchmark run completion rate | >95% (no flaky infrastructure) |

---

## 9. Open Questions

These are the highest-priority unknowns that the CEO/CTO should help prioritize for follow-up:

### Q1 — Customer Access (Blocking for S1 validation)
**Question:** Do we have warm introductions to engineers at any of the 15+ named Gluten production adopters (Microsoft, Google, Uber, ByteDance, Alibaba Cloud, etc.)?  
**Why it matters:** Customer discovery interviews are the critical path. Without access to real Gluten platform engineers, we cannot validate the problem hypothesis or pricing.  
**Recommended action:** CEO/CTO to identify 3–5 warm contacts at Gluten production adopters for Product Lead to schedule discovery calls.

### Q2 — Infrastructure Feasibility for MVP (Pending platform-lead input)
**Question:** What is the realistic wall-clock time and hardware requirement for TPC-DS SF=10 on a single-node Spark + Gluten environment? Can this run in under 60 minutes on a machine we can provision cost-effectively?  
**Why it matters:** If a single run takes 3+ hours, the PR-check use case is broken and we need to rethink the MVP scope (nightly job instead of PR check, or a smaller query subset).  
**Recommended action:** Platform Lead to provide a hardware spec and estimated run time. Product Lead will adjust MVP scope accordingly.

### Q3 — Statistical Methodology (Pending performance-lead input)
**Question:** What is the minimum number of repeated runs needed to produce a statistically reliable regression signal at TPC-DS SF=10? (1 run? 3 runs? 5 runs?)  
**Why it matters:** More runs = more cost and longer wall-clock time. If we need 5 runs to get a reliable signal, the PR-check use case may be infeasible and we need a different execution model.  
**Recommended action:** Performance Lead to advise on minimum run count and recommended statistical test.

### Q4 — Open Source vs. Proprietary Wedge
**Question:** Should the GitHub Actions workflow be open-source (free, drives adoption) with a paid managed runner/storage backend, or should the entire product be proprietary from day one?  
**Why it matters:** Open-source workflow drives adoption in the Gluten community but may reduce pricing power. Proprietary risks slow adoption in a community-driven ecosystem.  
**Recommended action:** CEO/CTO to make a strategic call. Product Lead recommends open-source workflow + paid managed infrastructure (CodSpeed model) as the default hypothesis.

### Q5 — Gluten vs. Comet as Initial Target
**Question:** Should the MVP target Gluten (Velox backend) specifically, or should we also support Apache Comet (DataFusion-based) from day one?  
**Why it matters:** Comet has a different benchmark setup (TPC-DS via spark-sql-perf, runs on EC2) and a different community. Supporting both doubles the integration surface. However, Comet's community is also active and has no automated regression detection.  
**Recommended action:** Start with Gluten only. Comet is a natural expansion after MVP validation. Revisit after S2 (workflow adoption) is achieved.

### Q6 — Pricing Validation Method
**Question:** How do we probe willingness-to-pay without a working product? (Fake door test? Pricing page? Direct ask in interviews?)  
**Why it matters:** We need WTP signal before investing in managed infrastructure.  
**Recommended action:** Product Lead to include a direct pricing probe in discovery interviews: "If this existed today and cost $X/month, would you expense it?" and observe reaction to $99, $299, and $499 price points.

---

## 10. Recommended Next Steps

| Priority | Action | Owner | Timeline |
|----------|--------|-------|----------|
| 1 | CEO/CTO identifies 3–5 warm contacts at Gluten production adopters | CEO/CTO | This week |
| 2 | Product Lead schedules 5 customer discovery interviews | Product Lead | Within 2 weeks |
| 3 | Platform Lead provides hardware spec + estimated TPC-DS SF=10 run time | Platform Lead | This week |
| 4 | Performance Lead advises on minimum run count for statistical reliability | Performance Lead | This week |
| 5 | CEO/CTO makes open-source vs. proprietary strategic call (Q4) | CEO/CTO | This week |
| 6 | Product Lead writes discovery interview guide | Product Lead | Before first interview |
| 7 | Product Lead runs 5 discovery interviews and synthesizes findings | Product Lead | Within 4 weeks |
| 8 | Team reviews discovery findings and makes build/explore/defer decision | All | Week 5 |

---

## Appendix A: Peer Input Received

### From Performance Lead (2026-10-04)

*Received in response to Product Lead's consultation request on performance signals.*

**Key signals for platform teams:**
- **Primary metric:** Query wall-clock latency (p50/p95). This is the go/no-go signal for Gluten upgrades.
- **Secondary metric:** Gluten fallback operator count — a jump in native→JVM fallbacks is often a leading indicator of latency regression and cheaper to collect than profiling.
- CPU efficiency and memory pressure are diagnostic (explain *why*), not the action signal.

**Minimum viable regression threshold:**
- ≥5% mean latency increase, p<0.05 (Welch's t-test, Benjamini-Hochberg FDR-corrected across all queries)
- For sub-10s queries, 5% may be noise; 10% practical threshold may be needed. **Customer validation needed on whether their SLA makes 5% vs. 10% matter.**

**Most regression-sensitive TPC-DS queries for Gluten:**
- Join-heavy: Q17, Q64, Q95 (multi-way joins, large intermediate results)
- Window functions: Q12, Q20, Q36, Q49 (Velox window operator coverage varies)
- Subquery-heavy: Q41, Q47, Q57 (correlated subqueries stress the planner)
- Aggregation-heavy: Q7, Q27, Q67 (grouping sets, rollup)

**Statistical method:** Welch's t-test (two-sample, unequal variance) with BH FDR correction across all 99 queries. Mann-Whitney U is a valid alternative for non-normal distributions.

**Open questions from Performance Lead (Section 11 of methodology brief):** Query weighting, acceptable regression threshold, cost-per-query priority, Gluten feature dependencies, CI SLA. Product Lead to address these in customer discovery interviews.

---

### From Platform Lead (2026-10-04)

*Received in response to Product Lead's consultation request on infrastructure constraints.*

**Minimum hardware for meaningful single-node benchmark:**
- CPU: 16 vCPUs minimum (m7g.4xlarge / Graviton3)
- RAM: 64 GiB minimum for TPC-DS SF=10 (32 GiB causes disk spill, high variance)
- Disk: 100 GB gp3 EBS with provisioned IOPS
- **Summary:** m7g.4xlarge (16 vCPU / 64 GiB) is the minimum for clean, interpretable results

**GitHub Actions hosted runners are not suitable.** They are shared, noisy-neighbor environments with no CPU governor control or NUMA isolation. Dedicated ephemeral EC2 instances are required. GitHub Actions is the orchestrator; benchmark runs on EC2.

**Expected wall-clock time for TPC-DS SF=10:**
- Full suite (99 queries, 1 warmup + 1 timed run): **65–80 minutes end-to-end**
- Provisioning + teardown: ~6 minutes additional
- **Implication:** Full-suite PR check blocks a PR for 65–80 minutes. **Recommended pattern: reduced-subset PR check (10–15 queries, ~15 min) + full nightly run.**

**Gluten build constraints:**
- Pre-built Gluten release artifacts: no compilation needed (MVP should pin to pre-built releases)
- Custom Gluten builds: require C++ toolchain + 30–60 min compilation (out of scope for MVP)
- ARM64 (Graviton3) pre-built artifact availability: **open risk** — may need to fall back to m7i.4xlarge (x86) if ARM64 artifacts aren't available for target Gluten version
- JVM: pin to Amazon Corretto 17 (LTS)
- Spark version coupling: Gluten artifacts are tightly coupled to specific Spark minor versions (configuration management concern, not a blocker)

**MVP infrastructure conclusion:** Feasible as a nightly job with a reduced-subset PR check. Full-suite PR gating adds 65–80 min to PR cycle time. Hardware must be dedicated EC2.

---

### Impact on MVP Scope (Updated Based on Peer Input)

The peer responses refine F2 (benchmark execution) and F3 (statistical detection):

- **F2 update:** MVP PR check should run the ~20 most regression-sensitive queries (Performance Lead's list: Q7, Q12, Q17, Q20, Q27, Q36, Q41, Q47, Q49, Q57, Q64, Q67, Q72, Q95 + 6 additional) rather than all 99. Full 99-query suite runs nightly.
- **F3 update:** Statistical method confirmed as Welch's t-test + BH FDR correction. Threshold: ≥5% regression, p<0.05. Fallback operator count added as a secondary signal.
- **Infrastructure:** MVP requires dedicated EC2 (m7g.4xlarge or m7i.4xlarge fallback). GitHub Actions is the trigger/orchestrator only.

---

## Appendix B: Research Sources

- Apache Gluten homepage (gluten.apache.org) — production adopter list, performance claims
- Apache Gluten GitHub discussion #11483 (Jan 2026 weekly status) — active regression issues, project velocity
- Apache DataFusion Comet benchmarking guide — manual benchmark process, no CI integration
- CodSpeed pricing page (codspeed.io/pricing) — competitive pricing model
- Bencher pricing page (bencher.dev/pricing) — competitive pricing model
- Zhihu Gluten case study (BestHub) — real-world adoption pain points
- SparkDoctor blog — Spark event log as performance record concept
- Gluten GitHub issue #8417 — TPC-DS Q72 regression as concrete evidence of the problem
