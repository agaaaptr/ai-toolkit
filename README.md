# ai-toolkit

> A Claude Code plugin marketplace shipping **`ai-dev-workflow`** — a daily, stack-agnostic AI development loop that survives context loss and never acts on assumptions.

`ai-dev-workflow` orchestrates the full lifecycle of a single task — intake, investigation, confirmation, planning, execution, verification, documentation — delegating the heavy lifting to specialized plugins and persisting state so work survives context loss across sessions.

## Why

Two failure modes waste the most time when pair-programming with an agent:

1. **Context loss** — long sessions get summarized; the agent forgets what it was doing.
2. **Assumptions** — the agent acts on a guess instead of confirming, and you discover it three steps later.

This toolkit is built around a single hard rule that addresses both: **Investigate → Confirm → Act**. Nothing executes before facts are gathered *and* you confirm understanding, and every phase is persisted to a state file so the loop can resume anywhere.

## Skills

| Command | What it does |
|---|---|
| `/init` | One-time project bootstrap — detect the real stack, confirm with you, scaffold only the missing gaps (`CLAUDE.md`/`AGENTS.md`, `GUIDE.md`, `workflow/`, `.gitignore`). |
| `/sync` | Session-start context load — index big docs for just-in-time retrieval, read git state, recall memory, print a brief. Read-only. |
| `/flow <task>` | The orchestrated 8-phase loop with human review gates. Fetches a ClickUp task (or accepts a paste). Persists state to `workflow/<task>.md`. |
| `/wrap` | Session close — run tests, update docs, tidy scratch, report. |

## How `/flow` works

Eight phases, each pausing for your approval:

```
0 Context → 1 Intake → 2 Investigate → 3 Clarify & Confirm (HARD gate)
         → 4 Plan → 5 Execute → 6 Verify → 7 Document (+ conditional release)
```

`/flow` is a **thin router**: it delegates Plan/Execute to the Superpowers skills (TDD, executing-plans) rather than reimplementing them. Phase 3 is a hard anti-assumption gate — every open question is resolved with you before any code is touched. Phase 7 commits per scope and, for Angular FE v13 `@uiigateway/*` libraries, offers a version bump + env-suffixed tag.

## Repository structure

```
ai-toolkit/
├── .claude-plugin/marketplace.json   marketplace manifest
├── plugins/ai-dev-workflow/
│   ├── skills/{init,sync,flow,wrap}/ one SKILL.md per skill
│   └── templates/                    state-file templates
├── AGENTS.md                         agent guidance for developing this repo
└── docs/                             architecture, decisions, specs, plans, findings
```

See [`docs/architecture/`](docs/architecture/) for the full system overview.

## Prerequisites (optional)

The three companion plugins are **optional** — `ai-dev-workflow` runs with or without them (see Two modes below):

| Plugin | Role | If absent |
|---|---|---|
| **Superpowers** | `/flow` delegates plan/execute/debug to its skills. | inline lean phases |
| **context-mode** | JIT retrieval (`ctx_index`/`ctx_search`) — keeps large files out of context. | `Read` + `Grep` |
| **agentmemory** | Cross-session recall/save of durable facts. | native `MEMORY.md` |

## Two modes (rich / lean)

- **Rich** — plugins installed: maximal capability (more tokens).
- **Lean** — no plugins: token-efficient fallbacks, still systematic.
- Default **auto** (each skill probes what is available, per capability). Override in the project's `CLAUDE.md`/`AGENTS.md`:
  ```
  ai-dev-workflow.mode: auto | rich | lean
  ```
- Internals: [`references/modes.md`](plugins/ai-dev-workflow/references/modes.md) · skill authoring [`references/skill-structure.md`](plugins/ai-dev-workflow/references/skill-structure.md) · commit/push [`references/commit-push.md`](plugins/ai-dev-workflow/references/commit-push.md) · doc layout [`references/doc-structure.md`](plugins/ai-dev-workflow/references/doc-structure.md).

## Installation

**Plugin marketplace (primary):**
```
/plugin marketplace add agaaaptr/ai-toolkit
/plugin install ai-dev-workflow@ai-toolkit
```

**npx skills (community discoverability):**
```bash
npx skills add agaaaptr/ai-toolkit
# if it lands in ~/.agents/skills/, symlink:
ln -s ~/.agents/skills/ai-toolkit ~/.claude/skills/ai-toolkit
```

**Git clone (fallback):**
```bash
git clone https://github.com/agaaaptr/ai-toolkit ~/.claude/skills/ai-toolkit
```

## Configuration (ClickUp, for `/flow`)

`/flow` fetches the task via the ClickUp REST API. Export in `~/.zshrc` (never commit):
```bash
export CLICKUP_API_TOKEN="pk_..."     # ClickUp → Settings → Apps → Generate
export CLICKUP_TEAM_ID="..."          # only for custom ids like #ABC-123
```
If unset, `/flow` falls back to asking you to paste the task.

## Usage

```
/init                      # once per project
/sync                      # at the start of each session
/flow <clickup-id>         # run the loop for one task
/wrap                      # close the session
```

## Development

This repo is itself developed with Claude Code. [`AGENTS.md`](AGENTS.md) holds the conventions for editing skills safely (SKILL.md format, commit-per-scope, where specs/plans go, how to validate). Edit skills here, then users refresh via `/plugin marketplace update ai-toolkit` (or `git pull`).

## Known caveats

- **`agentmemory` recall may return empty** in some environments. When it does, `/sync` falls back to `ctx_search` (the context-mode KB) and the native `MEMORY.md`. The durable record for any task is always `workflow/<task>.md`.
- **Non-interactive shell:** the Claude Code Bash tool does not source `~/.zshrc`, so `/flow` phase 1 runs `source ~/.zshrc` to load `CLICKUP_API_TOKEN` (or put exports in `~/.zshenv`, which non-interactive zsh does source).
- **Sparse ClickUp tasks** (empty description) are flagged for clarification at the phase-3 gate — scope is never assumed.

## Documentation

- [Architecture](docs/architecture/) · [Decision records (ADRs)](docs/decisions/)
- [Design specs](docs/specs/) · [Implementation plans](docs/plans/) · [Validation findings](docs/findings/)
