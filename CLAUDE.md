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

- `.opc/config/` — runtime config, all tracked (including `llm_config.yaml` and `channel_config.yaml`). Credential fields stay empty: put keys in the repo-root `.env` (gitignored; template in `.env.example`), which `bin/opc` loads, and reference them by variable name (e.g. `api_key_env`). The `.githooks/pre-commit` hook blocks commits that fill in a key/token/secret/password field — enable it once per clone with `git config core.hooksPath .githooks`
- `.opc/config/company_orgs/org_perf_ci_config.yaml` — the `perf_ci` org (tracked)
- `.opc/config/workplaces.yaml` — maps project `perf-ci` to this repo root, so agents work here
- `.opc/projects/perf-ci/`, `*.db`, `logs/` — runtime state (gitignored), except the tracked files below
- `.opc/skills/` — system skills (tracked; seeded from OpenOPC's built-in skills, now owned here)
- Employee experience (tracked — review diffs like AGENT.md edits; commit while the server is idle):
  - `.opc/company_state/perf_ci/employees/*.yaml` — hired employees
  - `.opc/company_state/perf_ci/employee_experience/*.json` — per-employee learned profile
  - `.opc/projects/perf-ci/employee_evolution.json`, `.opc/evolution/employees.json` — experience scores and reflections
  - `.opc/projects/perf-ci/skills/` — learned playbook skills promoted from repeated reflections
  - `.opc/projects/perf-ci/company_staffing_defaults.json` — saved staffing choices
- `.opc/prompts/talent/` — talent templates imported from `external/agency-agents` (gitignored cache). After updating the agency-agents submodule, refresh with `bin/opc talent import "$PWD/external/agency-agents"`

### Start the server

Always go through `bin/opc`, which sets `OPC_HOME` to this repo's `.opc`:

```bash
bin/opc ui --project perf-ci
```

Running `uv run opc` directly inside `external/OpenOPC` uses OpenOPC's own `.opc` and will not see Perf CI.

### Execution agent — opencode

OpenOPC and all agent tool use (file edits, shell) run on this Mac, in this repo. Only model inference runs on host `opencode` (192.168.68.58), which serves Ollama at `http://ollama:11434` (`/etc/hosts`: `192.168.68.58 opencode` and `192.168.68.58 ollama`).

- All roles run on `opencode` (`preferred_external_agent` in the org config; first in `preferred_order` in `agent_config.yaml`), using the local opencode CLI with model `ollama/qwen3.6:latest`.
- The `ollama` provider lives in your user opencode config (`~/.config/opencode/opencode.json`, `baseURL: http://ollama:11434/v1`).
- On host `opencode`, Ollama must listen on the network: start it with `OLLAMA_HOST=0.0.0.0:11434`. Ollama has no auth — anyone on the LAN can use it.
- The OpenCode desktop app's server on `opencode:4096` is not used by OpenOPC.

### LLM config

`.opc/config/llm_config.yaml` (tracked; never put a key in `api_key`). OpenOPC's own LLM drives intake, coordination, gates, and auto-recruit — if it fails, runs fail at CEO intake. Point it at Ollama's OpenAI-compatible API (`http://ollama:11434/v1`). Do not use the `ollama_chat/` LiteLLM route: when streaming, it merges parallel tool calls into one call with invalid arguments, and agents stall.

```yaml
llm:
  default_model: openai/qwen3.6:latest     # Ollama's OpenAI-compatible API
  api_base: http://ollama:11434/v1
  api_key: ''
  api_key_env: OLLAMA_API_KEY             # bin/opc defaults it to "ollama"
  context_window: 32768
  max_tokens: 8192
  temperature: 0.3
  routing: {}
  fallback: {}
  reasoning_effort: null
```

Or use Anthropic (requires API credit; set `ANTHROPIC_API_KEY` in `.env`, never in the file):

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

Requires `ANTHROPIC_API_KEY` in `.env` (or your shell environment).

### Org config

Single source of truth: `.opc/config/company_orgs/org_perf_ci_config.yaml`. The Office UI may rewrite it when the org is edited there; review those diffs before committing.

Role ownership, decision rights, handoff contracts (H1–H11), and escalation rules: `company/org/handoff-contracts.md`.

### Agent prompts

Each role's prompt is assembled from three files, listed in that role's `prompt_refs` (paths relative to `.opc/`) — there are no copies to sync:

1. `company/employees/{role}/ROLE.md` — **what**: mission, ownership, required outputs, handoffs, rules. Wins on conflict.
2. `company/employees/{role}/AGENT.md` — **how**: persona, workflow, checklists, communication style. No ownership claims.
3. `company/current-focus.md` — current-phase context and first tasks, shared by all roles.

Put responsibilities in ROLE.md, working style in AGENT.md, and anything phase-specific in `current-focus.md`.

Agents maintain their own "how": each AGENT.md ends with **Learning and Self-Maintenance** and **Tool notes** sections, and agents record tool lessons there (or as skills under `.opc/projects/perf-ci/skills/`). They must not edit ROLE.md, other roles' files, or `handoff-contracts.md`. Review agent edits to AGENT.md and skills in `git diff` before committing; revert anything wrong or overreaching.

### Talent profiles vs. AGENT.md

- **ROLE.md** (what) always applies to whoever fills the seat.
- **AGENT.md** (how) is the role's baseline working style; a Default Employee gets only this.
- **A hired talent profile** (from `external/agency-agents`) is added on top as an "Employee Persona" — an extra specialization, not a replacement. It also accumulates an experience profile across runs.
- Hire talent deliberately via **Org → Hire Talent**, choosing a profile that fits the role. Never let chat "recruit" auto-match by keywords.
- To add a Perf CI–specific profile, add it to the agency-agents fork, then re-import. Profiles must not claim ownership; ROLE.md wins on conflict.

### Staffing — use Default Employee

When the staffing dialog appears before a company run, pick **Default Employee** for each role unless you deliberately hired a talent profile for that role (see above). Default Employee uses only the role's `prompt_refs` (ROLE.md, AGENT.md, current-focus.md). Do NOT click **Auto recruit** (the UI recommends it whenever templates exist): it depends on OpenOPC's LLM, and when that fails it assigns the alphabetically first template ("Anthropologist") to every role. Do NOT let OPC auto-recruit from the talent market — keyword matching picks unrelated templates (e.g. "Anthropologist", or "Performance Benchmarker" for QA).

### Artifact ownership conflicts

OpenOPC records one owner per file per run (`company_artifact_ownership` in `tasks.db`). An agent write refused with "artifact path is owned by another durable WorkItem" means another step owns the file. The OpenOPC fork lets a later same-role step take over once the owner is approved/failed/cancelled; cross-role writes stay blocked by design (see `company/org/handoff-contracts.md`, Artifact Ownership).

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
          'delegation_work_items','delegation_events','team_instances','seat_states',
          'role_runtime_sessions']:  # role_runtime_sessions: else an orphan character shows in Office
    try: cur.execute(f'DELETE FROM {t} WHERE run_id IN ({p})', STALE_RUNS)
    except: pass
conn.commit(); conn.close()
"

# Also delete orphaned work_item_runtime_links AND their tasks (causes startup crash if missed)
sqlite3 ./.opc/projects/perf-ci/tasks.db \
  "DELETE FROM work_item_runtime_links WHERE work_item_id NOT IN (SELECT work_item_id FROM delegation_work_items);"
sqlite3 ./.opc/projects/perf-ci/tasks.db \
  "DELETE FROM tasks WHERE metadata LIKE '%work_item_runtime%' AND metadata LIKE '%work_item_projection_id%' AND id NOT IN (SELECT runtime_task_id FROM work_item_runtime_links);"

# Also remove leftovers keyed by the stale runs' task IDs (from delegation_work_items before
# deleting, or from runtime_sessions): runtime_sessions, execution_checkpoints, agent_messages
# (tasks.db) and task_progress (ui_state.db). Keep session_messages if the chat session is
# still in use. Back up .opc/*.db and tasks.db first.

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
