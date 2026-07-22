# Architecture

`ai-toolkit` is a **Claude Code plugin marketplace** shipping a single plugin — `ai-dev-workflow` — a daily, stack-agnostic AI development loop built to survive context loss and never act on assumptions.

## Layout

```
ai-toolkit/                          marketplace repo
├── .claude-plugin/marketplace.json  marketplace manifest (lists plugins)
├── plugins/
│   └── ai-dev-workflow/             the plugin
│       ├── skills/                  one folder per skill, each with SKILL.md
│       │   ├── init/  sync/  flow/  wrap/
│       └── templates/               state-file templates the skills emit
├── AGENTS.md                        agent guidance for developing this repo
└── docs/                            architecture, decisions, specs, plans, findings
```

A **skill** = a folder with a `SKILL.md` (YAML frontmatter `name` + `description`, then Markdown instructions). Skills may add `scripts/`, `references/`, `assets/`.

## The four skills

| Skill | Role | Mutates? |
|---|---|---|
| `/init` | One-time project bootstrap: detect the real stack, confirm, scaffold only the gaps (`CLAUDE.md`/`AGENTS.md`, `.notes/`, `GUIDE.md`, `workflow/`, `.gitignore`). | Yes (setup) |
| `/sync` | Session-start context load: index big docs for just-in-time retrieval, read git state, recall memory, print a brief. | No (read-only) |
| `/flow <task>` | The orchestrated 8-phase loop with human review gates. | Yes |
| `/wrap` | Session close: run tests, update docs, tidy, report. | Yes (docs) |

## The `/flow` loop

Eight phases, each pausing for user approval:

```
0 Context → 1 Intake → 2 Investigate → 3 Clarify & Confirm (HARD gate)
         → 4 Plan → 5 Execute → 6 Verify → 7 Document (+ conditional release)
```

`/flow` is a **thin router**: it delegates Plan/Execute to the Superpowers skills (brainstorming, writing-plans, executing-plans, TDD, systematic-debugging) rather than reimplementing them. Phase 3 is a hard anti-assumption gate — every open question is resolved with the user before any edit. On any later doubt, the loop returns to phase 3.

## The spine (hard rule)

Every skill follows **Investigate → Confirm → Act**: no edit, run, or execution happens before facts are gathered *and* the user confirms understanding.

## Dependencies

`ai-dev-workflow` orchestrates three companion plugins:

- **Superpowers** — `/flow` delegates Plan/Execute/Debug to its skills.
- **context-mode** — `ctx_index`/`ctx_search` for just-in-time retrieval (keeps large files out of the context window).
- **agentmemory** — cross-session recall/save of durable facts.

`tidy-session-docs` is an optional companion used by `/wrap` and `/flow` phase 7 if present.

## State that survives context loss

`/flow` writes `workflow/<task>.md` (from `templates/workflow-state.md.tpl`) every phase. This file — not conversation memory — is the source of truth that survives compaction and new sessions.
