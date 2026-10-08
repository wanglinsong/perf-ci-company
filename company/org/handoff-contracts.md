# Handoff Contracts

Every handoff between roles is a written artifact with a fixed set of fields.
A handoff missing a required field is incomplete; the receiver returns it
instead of guessing.

Thresholds (repetitions, CV limits, significance, multiple-comparison
correction) are **not** restated here or in role prompts. The single source is
`external/perf-ci-infra/experiments/methodology-v1.md` (or its successor).

## Decision Rights

| Decision | Proposes | Approves / Decides | Veto | Final |
|---|---|---|---|---|
| Customer problem, MVP scope | Product Lead | CEO/CTO | — | CEO/CTO |
| Benchmark methodology & thresholds | Performance Lead | QA Lead (review) | QA Lead | CEO/CTO arbitrates |
| Experiment design | Performance Lead | QA Lead (H3 → H4) | QA Lead | CEO/CTO arbitrates |
| How an experiment executes | Platform Lead | — | QA Lead (if spec deviates) | Platform Lead |
| Performance conclusion content | Performance Lead | — | — | Performance Lead |
| Publishing a performance claim | Performance Lead | QA Lead (H8) | QA Lead | CEO/CTO arbitrates |
| Product components (analyzer, CI check, report) | Platform Lead builds | Performance Lead accepts algorithm; QA Lead validates | QA Lead | CEO/CTO |
| Cloud spend inside approved envelope | Platform Lead | — | — | Platform Lead |
| Cloud spend / destructive action outside envelope | Platform Lead | CEO/CTO | — | Human founder |
| Release | CEO/CTO (after QA verdict) | Human founder | QA Lead (BLOCKED) | Human founder |

## Flows

**Experiment lifecycle**

```
Performance ──H3 Experiment Spec──▶ QA ──H4 Design Challenge──▶ Performance
Performance ──H5 Run Request─────▶ Platform ──H6 Run Record──▶ Performance + QA
Performance ──H7 Investigation───▶ QA ──H8 Validation Report──▶ Performance + CEO/CTO
CEO/CTO ──H9 Release / Escalation──▶ Human founder
```

**Product component delivery**

```
Product ──H2 Requirement Brief──▶ CEO/CTO ──H1 Assignment──▶ Performance / Platform
Performance ──H10 Algorithm Spec──▶ Platform ──H11 Component Delivery──▶ Performance + QA
```

All assignments come from the CEO/CTO (H1). IC roles do not dispatch work to
each other; they produce the artifacts below and the CEO/CTO routes them. In
the runtime, QA is the configured reviewer for Performance and Platform work.

---

## Repository Layout

**The parent repo (`perf-ci-company`) defines and governs the company; `external/perf-ci-infra` holds the work it produces.**

| Role | `external/perf-ci-infra` | Parent repo |
|---|---|---|
| CEO / CTO | — (coordinates; produces no artifacts) | `company/decisions/`, `company/reviews/` (item status), `company/current-focus.md`, own `AGENT.md`, skills |
| Product Lead | `product/` (H2, discovery) | own `AGENT.md`, skills |
| Performance Lead | `experiments/` (H3, H5, H7, H10) | own `AGENT.md`, skills |
| Platform Lead | `infra/` (code, H6, H11) | own `AGENT.md`, skills |
| QA Lead | `qa/` (H4, H8, component validation) | own `AGENT.md`, skills |

"Own `AGENT.md`" is `company/employees/<role>/AGENT.md`; skills are `.opc/projects/perf-ci/skills/`. ROLE.md files, this file, `.opc/config/`, and submodule pointers are founder-owned.

Commits (enforced by `.githooks/commit-msg` in both repos):

- Commit on the `staging` branch only; one commit per work item per repo.
- Stage only your role's paths (`git add <paths>`; never `git add -A`, `git add .`, or `git commit -a`).
- Message: `<role>: <work item title> (<contract>)`, ending with trailers:

  ```
  Work-Item: <work item id>
  Contract: <H1–H11 or none>
  Role: <role id>
  ```
- Never commit a submodule pointer; the founder bumps it when opening the parent PR.
- Agents commit and push as the bot account `jason-nb-wang`. Only the CEO/CTO pushes (`git push agent staging`) and opens PRs, once a run's work items are approved; nobody but the founder approves or merges. `.githooks/pre-push` blocks agent pushes to `main`, to other branches, or through `origin`.

## Artifact Ownership

The runtime enforces one writer per file per run:

- A file belongs to the work item (step) that first writes it in a run.
- A later step of the **same role** may revise it only once the owning step
  is finished (approved, failed, or cancelled) — e.g. the CEO/CTO's delivery
  step correcting files written at intake.
- Files owned by **another role** are never edited directly. Request the
  change from the owning role through the CEO/CTO (H1), or record the needed
  correction in your own artifact.
- If a write is refused as "owned by another durable WorkItem", do not retry
  or try to delete work items; report it in your result (or as an H9 if it
  blocks delivery).

## H1 — Work Assignment (CEO/CTO → any IC)

- **ID** and **owner role** (exactly one)
- **Objective** — the question or outcome, one sentence
- **Inputs** — artifacts the owner must read (paths)
- **Deliverable** — which contract (H2–H11) or file, and output path
- **Constraints** — what the owner may and may not decide alone; budget
- **Reviewer** — QA Lead for experiment or component work; CEO/CTO otherwise
- **Done when** — acceptance criteria

## H2 — Requirement Brief (Product Lead → CEO/CTO)

- **Customer** — persona and named example teams
- **Problem** — current workflow and the specific pain
- **Evidence** — each claim labeled `public-source` (with link), `founder-interview`,
  or `assumption`. Agents do not conduct interviews; they prepare scripts and
  analyze founder-provided notes.
- **Requirement** — what the product must do, as user stories
- **Success metric** — measurable, with time horizon
- **Out of scope**
- **Open questions** for Performance / Platform

Path: `external/perf-ci-infra/product/`

## H3 — Experiment Spec (Performance Lead → QA Lead)

Submitted **before** any run.

- **Experiment ID**, **objective** (question answered)
- **Baseline** — engine versions, commit SHAs, full config
- **Candidate** — engine versions, commit SHAs, full config, and the diff vs. baseline
- **Workload** — suite, scale factor, query list, data format, data location
- **Environment** — instance type, OS, JVM, Spark and Gluten config
- **Measurement plan** — metrics, repetitions, warmup handling, run ordering
- **Analysis plan** — statistical test and thresholds (by reference to methodology)
- **Known confounds** and how each is controlled
- **Expected result** — what outcome would confirm vs. refute the hypothesis
- **Cost estimate** — must fit the approved envelope

Path: `external/perf-ci-infra/experiments/<experiment-id>/spec.md`

## H4 — Design Challenge (QA Lead → Performance Lead)

- **Experiment ID** and spec revision reviewed
- **Adversarial question responses** — answerable from the design or not
- **Findings** — specific, with reference to spec section
- **Verdict** — `APPROVED FOR EXECUTION` / `NEEDS WORK` / `BLOCKED`
- **Required changes** if not approved

Only an `APPROVED FOR EXECUTION` spec may be sent to Platform (H5).

Path: `external/perf-ci-infra/qa/<experiment-id>/design-challenge.md`

## H5 — Run Request (Performance Lead → Platform Lead, via CEO/CTO)

- **Experiment ID** and **approved spec revision** (link to H3 + H4)
- **Run matrix** — configurations × repetitions × ordering
- **Artifacts to collect** — latencies, plans, event logs, GC logs, profiles, metrics
- **Deadline** and **cost cap**

Platform executes exactly the approved spec. Any deviation needed for
feasibility goes back to Performance and QA before execution.

## H6 — Run Record (Platform Lead → Performance Lead + QA Lead)

- **Run IDs** and the **experiment ID / spec revision** executed
- **Environment fingerprint** — instance IDs and types, AMI, kernel, JVM,
  engine builds (SHAs), full effective config, dataset checksum
- **Execution log** — start/end per run, failures, retries
- **Raw measurements** — per query, per repetition (not only summaries)
- **Collected artifacts** — paths in result storage
- **Deviations from spec** — none, or each one listed
- **Cost** — actual spend per run and total
- **Teardown confirmation** — no resources left running

Path: `external/perf-ci-infra/experiments/<experiment-id>/runs/`

## H7 — Investigation Report (Performance Lead → QA Lead)

The 11-point output: objective, baseline, candidate, environment, workload,
measurements (link to H6 raw data), statistical analysis, observations,
root-cause hypotheses (each with evidence), confidence, recommended next
experiment. Observations, hypotheses and root causes are labeled separately.

Path: `external/perf-ci-infra/experiments/<experiment-id>/report.md`

## H8 — Validation Report (QA Lead → Performance Lead + CEO/CTO)

- **Experiment ID** and report revision reviewed
- **Adversarial question responses** — YES / NO / UNKNOWN with evidence
- **Independent checks** — recomputed statistics from raw data; reproduction run IDs
- **Findings**
- **Verdict** — `APPROVED` / `NEEDS WORK` / `BLOCKED`
- **Required fixes**

A performance claim may be published or used in a release decision only with
an `APPROVED` verdict.

Path: `external/perf-ci-infra/qa/<experiment-id>/validation.md`

## H9 — Release / Escalation Request (CEO/CTO → Human founder)

- **Decision requested** — one sentence
- **Options** with recommendation
- **Evidence** — links to H8 verdicts and other artifacts
- **Cost, risk, reversibility**
- **Deadline** — when the decision blocks work

## H10 — Algorithm Spec (Performance Lead → Platform Lead)

For product components that encode methodology (regression detector,
baseline selection, report contents).

- **Component** and the requirement (H2) it serves
- **Inputs / outputs** — data shapes
- **Algorithm** — exact statistical procedure and thresholds (by reference)
- **Reference cases** — inputs with expected verdicts, including a known
  regression and a known no-change case
- **Acceptance criteria**

## H11 — Component Delivery (Platform Lead → Performance Lead + QA Lead)

- **Component**, **version / commit**
- **How to run it** — one command
- **Test results** — including all H10 reference cases
- **Known limitations**
- **Cost and runtime** per invocation

Performance Lead confirms the algorithm matches H10; QA Lead validates it
against known regressions before it gates anything.

## Escalation Rules

| Condition | Action |
|---|---|
| QA `BLOCKED` and Performance disagrees | CEO/CTO arbitrates with both reports; may order a reproduction run |
| Same work item fails review twice | CEO/CTO replans or rescopes |
| Run needs spend or resources outside the approved envelope | CEO/CTO → human founder (H9) |
| Destructive cloud action outside automated ephemeral lifecycle | Human founder approval |
| Platform must deviate from an approved spec | Back to Performance + QA before execution |
| Required evidence depends on customers or external access | CEO/CTO → human founder (H9) |
| Release decision | QA verdict → CEO/CTO recommendation → human founder |
