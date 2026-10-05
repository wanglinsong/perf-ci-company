# Perf CI — Claude Code Guide

## Repo structure

- `company/` — org design, role definitions, employee AGENT.md files
- `product/` — product discovery, issue docs, MVP specs
- `experiments/` — experiment specs and results
- `infra/` — platform infrastructure code
- `external/OpenOPC/` — OpenOPC submodule (AI agent runtime)
- `external/agency-agents/` — talent template library submodule

## Agent role

Claude Code's job in this repo is **company structure and repo management only**. The OPC agents (CEO/CTO, Product Lead, Performance Lead, Platform Lead, QA Lead) do the actual product and engineering work. Do not implement work items directly — create issues and let OPC agents execute them.

## OpenOPC setup

### Start the server

```bash
cd external/OpenOPC
uv run opc ui --project perf-ci
```

### LLM config

`external/OpenOPC/.opc/config/llm_config.yaml` is gitignored. Set it up manually:

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

The Perf CI org is defined in two places (keep in sync):
- **Source of truth**: `company/org/organization.yaml`
- **OpenOPC runtime**: `external/OpenOPC/.opc/config/company_orgs/org_perf_ci_config.yaml`

### Agent prompts

Each role has an `AGENT.md` under `company/employees/{role}/AGENT.md`. Copies live in `external/OpenOPC/.opc/prompts/perf-ci/{role}.md`. When updating an AGENT.md, copy it to the OpenOPC prompts directory too.

### Staffing — use Default Employee

When the staffing dialog appears before a company run, always pick **Default Employee** for each role. This uses the role's `prompt_refs` (the AGENT.md files) directly. Do NOT let OPC auto-recruit from the talent market — it will pick an unrelated template (e.g. "Anthropologist") that overrides the role's personality.

### Stale state cleanup

If the Office view shows wrong agents (e.g. "Anthropologist" on all roles):

```bash
# 1. Remove hired employee files
rm external/OpenOPC/.opc/company_state/perf_ci/employees/*.yaml

# 2. Purge stale run sessions from tasks.db (replace RUN_IDs with actual stale run IDs)
python3 -c "
import sqlite3
STALE_RUNS = ['run-id-1', 'run-id-2']  # get from delegation_role_sessions
p = ','.join('?'*len(STALE_RUNS))
conn = sqlite3.connect('external/OpenOPC/.opc/projects/perf-ci/tasks.db')
cur = conn.cursor()
for t in ['delegation_role_sessions','delegation_runs','delegation_cells',
          'delegation_work_items','delegation_events','team_instances','seat_states']:
    try: cur.execute(f'DELETE FROM {t} WHERE run_id IN ({p})', STALE_RUNS)
    except: pass
conn.commit(); conn.close()
"

# Also delete orphaned work_item_runtime_links AND their tasks (causes startup crash if missed)
sqlite3 external/OpenOPC/.opc/projects/perf-ci/tasks.db \
  "DELETE FROM work_item_runtime_links WHERE work_item_id NOT IN (SELECT work_item_id FROM delegation_work_items);"
sqlite3 external/OpenOPC/.opc/projects/perf-ci/tasks.db \
  "DELETE FROM tasks WHERE metadata LIKE '%work_item_runtime%' AND metadata LIKE '%work_item_projection_id%' AND id NOT IN (SELECT runtime_task_id FROM work_item_runtime_links);"

# 3. Clear UI cache
python3 -c "
import sqlite3
conn = sqlite3.connect('external/OpenOPC/.opc/ui_state.db')
cur = conn.cursor()
for t in ['agents','agent_appearances','custom_agents_shadow']:
    cur.execute(f'DELETE FROM {t}')
conn.commit(); conn.close()
"

# 4. Reset staffing defaults
echo '{\"scopes\":{},\"updated_at\":\"\",\"version\":1}' > external/OpenOPC/.opc/projects/perf-ci/company_staffing_defaults.json
```
