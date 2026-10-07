# Perf CI — Claude Code Guide

## Repo structure

- `company/` — org design, role definitions, employee AGENT.md files
- `product/` — product discovery, issue docs, MVP specs
- `experiments/` — experiment specs and results
- `infra/` — platform infrastructure code
- `external/OpenOPC/` — OpenOPC submodule (AI agent runtime)
- `external/agency-agents/` — talent template library submodule

## Agent role

**If you are an OPC role agent** (your instructions include a Perf CI ROLE.md, e.g. "# Performance QA Lead — Role"): this repo root is your workplace. Follow your ROLE.md and AGENT.md; the rest of this section does not apply to you.

**Otherwise** (a Claude Code session started by the human founder): your job in this repo is **company structure and repo management only**. The OPC agents (CEO/CTO, Product Lead, Performance Lead, Platform Lead, QA Lead) do the actual product and engineering work. Do not implement work items directly — create issues and let OPC agents execute them.

## OpenOPC setup

### Runtime layout

All Perf CI runtime lives in this repo. `.opc/` is OpenOPC's `OPC_HOME`:

- `.opc/config/` — runtime config (tracked); `llm_config.yaml` and `channel_config.yaml` are gitignored
- `.opc/config/company_orgs/org_perf_ci_config.yaml` — the `perf_ci` org (tracked)
- `.opc/config/workplaces.yaml` — maps project `perf-ci` to this repo root, so agents work here
- `.opc/projects/perf-ci/`, `.opc/company_state/`, `*.db`, `logs/` — runtime state (gitignored)
- `.opc/skills`, `.opc/prompts/talent` — symlinks into the OpenOPC submodule

### Start the server

Always go through `bin/opc`, which sets `OPC_HOME` to this repo's `.opc`:

```bash
bin/opc ui --project perf-ci
```

Running `uv run opc` directly inside `external/OpenOPC` uses OpenOPC's own `.opc` and will not see Perf CI.

### LLM config

`.opc/config/llm_config.yaml` is gitignored. Set it up manually:

```yaml
llm:
  api_key: ''
  api_key_env: ANTHROPIC_API_KEY
  default_model: anthropic/claude-sonnet-4-6
  context_window: 200000
  context_window_overrides:
    anthropic/claude-sonnet-4-6: 200000
  max_tokens: 32768
  temperature: 0.3
  routing:
    quick_tasks: anthropic/claude-haiku-4-5-20251001
  fallback: {}
  reasoning_effort: null
```

Requires `ANTHROPIC_API_KEY` in your shell environment.

### Org config

Single source of truth: `.opc/config/company_orgs/org_perf_ci_config.yaml`. The Office UI may rewrite it when the org is edited there; review those diffs before committing.

Role ownership, decision rights, handoff contracts (H1–H11), and escalation rules: `company/org/handoff-contracts.md`.

### Agent prompts

Each role's prompt is assembled from three files, listed in that role's `prompt_refs` (paths relative to `.opc/`) — there are no copies to sync:

1. `company/employees/{role}/ROLE.md` — **what**: mission, ownership, required outputs, handoffs, rules. Wins on conflict.
2. `company/employees/{role}/AGENT.md` — **how**: persona, workflow, checklists, communication style. No ownership claims.
3. `company/current-focus.md` — current-phase context and first tasks, shared by all roles.

Put responsibilities in ROLE.md, working style in AGENT.md, and anything phase-specific in `current-focus.md`.

### Staffing — use Default Employee

When the staffing dialog appears before a company run, always pick **Default Employee** for each role. This uses the role's `prompt_refs` (ROLE.md, AGENT.md, current-focus.md) directly. Do NOT let OPC auto-recruit from the talent market — it will pick an unrelated template (e.g. "Anthropologist") that overrides the role's personality.

### Stale state cleanup

If the Office view shows wrong agents (e.g. "Anthropologist" on all roles):

```bash
# 1. Remove hired employee files
rm ./.opc/company_state/perf_ci/employees/*.yaml

# 2. Purge stale run sessions from tasks.db (replace RUN_IDs with actual stale run IDs)
python3 -c "
import sqlite3
STALE_RUNS = ['run-id-1', 'run-id-2']  # get from delegation_role_sessions
p = ','.join('?'*len(STALE_RUNS))
conn = sqlite3.connect('./.opc/projects/perf-ci/tasks.db')
cur = conn.cursor()
for t in ['delegation_role_sessions','delegation_runs','delegation_cells',
          'delegation_work_items','delegation_events','team_instances','seat_states']:
    try: cur.execute(f'DELETE FROM {t} WHERE run_id IN ({p})', STALE_RUNS)
    except: pass
conn.commit(); conn.close()
"

# Also delete orphaned work_item_runtime_links AND their tasks (causes startup crash if missed)
sqlite3 ./.opc/projects/perf-ci/tasks.db \
  "DELETE FROM work_item_runtime_links WHERE work_item_id NOT IN (SELECT work_item_id FROM delegation_work_items);"
sqlite3 ./.opc/projects/perf-ci/tasks.db \
  "DELETE FROM tasks WHERE metadata LIKE '%work_item_runtime%' AND metadata LIKE '%work_item_projection_id%' AND id NOT IN (SELECT runtime_task_id FROM work_item_runtime_links);"

# 3. Clear UI cache
python3 -c "
import sqlite3
conn = sqlite3.connect('./.opc/ui_state.db')
cur = conn.cursor()
for t in ['agents','agent_appearances','custom_agents_shadow']:
    cur.execute(f'DELETE FROM {t}')
conn.commit(); conn.close()
"

# 4. Reset staffing defaults
echo '{\"scopes\":{},\"updated_at\":\"\",\"version\":1}' > ./.opc/projects/perf-ci/company_staffing_defaults.json
```
