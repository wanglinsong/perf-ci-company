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

### Stale state cleanup

If the Office view shows wrong agents, clean up:

```bash
# Remove hired employee files
rm external/OpenOPC/.opc/company_state/perf_ci/employees/*.yaml

# Clear UI cache
python3 -c "
import sqlite3
conn = sqlite3.connect('external/OpenOPC/.opc/ui_state.db')
cur = conn.cursor()
cur.execute('DELETE FROM agents')
cur.execute('DELETE FROM agent_appearances')
cur.execute('DELETE FROM custom_agents_shadow')
conn.commit()
conn.close()
"

# Reset staffing defaults
echo '{"scopes":{},"updated_at":"","version":1}' > external/OpenOPC/.opc/projects/perf-ci/company_staffing_defaults.json
```
