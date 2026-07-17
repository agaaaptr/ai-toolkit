# ai-toolkit

A personal collection of Claude Code plugins/skills/hooks for AI-assisted development.

## Plugins

### ai-dev-workflow — Full AI Development Workflow

A daily, stack-agnostic dev loop that eliminates repeated context-explanation and survives context loss.

**Skills:**
- `/init` — one-time project bootstrap (detect + confirm + scaffold gaps).
- `/sync` — load + index project context at session start (explicit, read-only).
- `/flow <task>` — orchestrated 8-phase loop with anti-assumption gates; fetches a ClickUp task.
- `/wrap` — close the session (tests + docs + tidy).

**Spine (hard rule):** Investigate → Confirm → Act. Nothing executes before facts are gathered and you confirm understanding.

### Dependencies (install these first)

This plugin delegates to and relies on three other plugins. Install all three via your Claude Code plugin manager before using the workflow:

| Plugin | Role |
|---|---|
| **Superpowers** | `/flow` delegates plan/execute/debug phases to its skills (brainstorming, writing-plans, executing-plans, systematic-debugging, test-driven-development). |
| **context-mode** | `/sync` and `/flow` use `ctx_index`/`ctx_search` for just-in-time retrieval (keeps large files out of the context window). |
| **agentmemory** | cross-session `recall`/`save` of durable facts. |

**Optional companion — `tidy-session-docs`:** `/wrap` and `/flow` phase 7 use it for doc promotion + scratch cleanup **if present**; otherwise they fall back to an inline minimal tidy. Not required for the workflow to run.

### Install

**Primary — plugin marketplace:**
```
/plugin marketplace add agaaaptr/ai-toolkit
/plugin install ai-dev-workflow@ai-toolkit
```

**Secondary — npx skills (community, discoverability):**
```bash
npx skills add agaaaptr/ai-toolkit
# verify install path; if it landed in ~/.agents/skills/, symlink:
ln -s ~/.agents/skills/ai-toolkit ~/.claude/skills/ai-toolkit
```

**Fallback — git clone:**
```bash
git clone https://github.com/agaaaptr/ai-toolkit ~/.claude/skills/ai-toolkit
```

### ClickUp (for `/flow` intake)

`/flow` fetches a task via the ClickUp REST API. Set in `~/.zshrc` (never commit):
```bash
export CLICKUP_API_TOKEN="pk_..."        # ClickUp -> Settings -> Apps -> Generate
export CLICKUP_TEAM_ID="..."             # only for custom ids like #ABC-123
```
If unset, `/flow` falls back to asking you to paste the task.

### Update

Edit skills here → `git push`. Users refresh with `/plugin marketplace update ai-toolkit` (marketplace) or `git pull` (clone).
