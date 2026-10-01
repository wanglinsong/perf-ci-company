# Strategy

## Phase 1 — Build the AI Engineering Company

Objective:

Validate that an AI-native engineering organization can investigate
distributed-system performance problems more effectively than a
single-agent workflow.

Focus:

* Validate OpenOPC workflows
* Establish company memory
* Establish engineering roles and handoffs
* Define experiment and evidence standards
* Develop the first benchmark methodology
* Identify the highest-value Spark + Gluten performance problem

Success is measured by validated engineering outcomes, not by the
number of agents, documents, or features created.

---

## Phase 2 — Build the Perf CI Engine

Objective:

Build the smallest system that can continuously answer:

* Is this change faster?
* Is this change slower?
* Is this change cheaper?
* Why did performance change?

Initial scope:

* Spark
* Gluten
* Reproducible benchmark environments
* Baseline management
* Regression detection
* Performance analysis
* Cost/performance analysis
* Experiment history and company memory

Core architecture:

**Deterministic execution + AI investigation + organizational memory**

---

## Phase 3 — Validate With Real Users

Objective:

Determine whether engineering teams will use and pay for
continuous performance engineering.

Initial customer profile:

Teams operating distributed data infrastructure where changes to
engines, workloads, configurations, or hardware can materially affect
performance or infrastructure cost.

Initial deployment model:

* Customer-controlled infrastructure
* BYOC
* Secure benchmark execution
* SaaS control plane / APIs

Validate:

* painful problem
* measurable ROI
* workflow adoption
* willingness to pay
* security requirements
* deployment requirements

Do not optimize for scale before validating the problem.

---

## Phase 4 — Expand the Platform

Potential expansion:

* Trino
* Presto
* Additional native execution engines
* Workload replay
* Hardware optimization
* Cloud cost optimization
* Automated experiment design
* Additional distributed data systems

Expansion should follow validated customer demand and reusable
technical capabilities rather than platform breadth for its own sake.

