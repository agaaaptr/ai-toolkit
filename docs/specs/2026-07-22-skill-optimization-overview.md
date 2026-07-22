# Skill Optimization — Overview & Roadmap

- **Date:** 2026-07-22
- **Status:** ✅ Complete — SP-1…5 implemented + pushed to `main`
- **Related:** [flow FE versioning design](2026-07-22-flow-fe-versioning-design.md), [version tracking addendum](2026-07-22-version-tracking-and-changelog-env.md)

## Context

Optimize every `ai-dev-workflow` skill for **two operating modes**:
- **Rich** — plugins installed (Superpowers, context-mode, agentmemory): maximal output, more tokens.
- **Lean** — no plugins at all: token-efficient, still systematic and good.

Every skill must **degrade gracefully** with safe fallbacks + user confirmation at gates (no assumptions — the Investigate→Confirm→Act spine holds in both modes).

## Decisions (2026-07-22)

| Area | Decision |
|---|---|
| Mode detection | **Auto (probe tool/skill availability) + override** — flag `ai-dev-workflow.mode: auto\|rich\|lean` in `CLAUDE.md`/`AGENTS.md` (default `auto`) |
| Scope | **Decompose into sub-projects** (SP-1…5), each spec → plan → implement |
| Checkpoint | **New `/checkpoint` skill + `/wrap` always checkpoints** |
| Token | **Progressive disclosure, full** — lean `SKILL.md` + per-skill `references/` |
| Doc structure | **Standard layout** (permanent `docs/{specs,plans,decisions,architecture,reference,handoffs,findings}` + `DOC-POLICY.md`; ephemeral `.session/` + `.superpowers/sdd/` cleaned each session); superpowers override → `docs/specs`,`docs/plans`; **plans permanent**; `/init` asks adapt-existing vs leave+generate |

## Capability matrix (rich → lean fallback)

| Capability | Rich (plugin) | Lean (fallback) |
|---|---|---|
| Retrieval (docs/knowledge) | context-mode `ctx_index`/`ctx_search` | `Read` on-demand + `Grep` |
| Memory (cross-session facts) | agentmemory `recall`/`save` | native `MEMORY.md` + `workflow/<task>.md` |
| Process (brainstorm/spec/plan/execute) | Superpowers skills | inline lean versions (in `/flow` `references/`) |

## Doc-structure standard

One layout for every project (set up by `/init` SP-4, curated by `/wrap` SP-5). Derived from two real `tidy-session-docs` references — BE `svc-academic-activity-go`, FE `lib-uii-gateway-academy-angular` — unified into one standard.

- **Permanent (committed, keep):** `docs/{specs,plans,decisions,architecture,reference,handoffs,findings}/` + `docs/DOC-POLICY.md`.
- **Ephemeral (gitignored, promote-or-delete each session):** `.session/`; `.superpowers/sdd/` (SDD default — contents cleaned each session end).
- **Naming:** `YYYY-MM-DD-<scope>-<slug>.md`.
- **Memory doc (`/init` output):** `docs/reference/project-memory.md` — relevant agentmemory memories + recall queries (agentmemory present); lean fallback = native `MEMORY.md`. Detailed in SP-4.
- **superpowers override:** specs → `docs/specs/`, plans → `docs/plans/` (prevents stray `docs/superpowers/`).
- **Curation rule:** only durable/permanent survives; all ephemeral deleted (promote durable content first). Stale permanent → fix or delete.
- **`/init` scan:** if existing doc structure/similar docs found → ASK adapt-to-standard vs leave+generate-new (standard always generated; existing fate = user's call).
- **Plans = permanent** (kept as records).

## Sub-projects

| SP | Focus | Output | Status |
|---|---|---|---|
| **SP-1** | Foundation: 2-mode framework + graceful-degradation + progressive-disclosure + commit/push discipline + **doc-structure standard** (DOC-POLICY, superpowers override, curation rules) | shared `references/{modes,skill-structure,commit-push,doc-structure}.md`, README/AGENTS update | ✅ done (pushed) — [spec](2026-07-22-sp1-skill-foundation.md) |
| SP-2 | `/sync` 2-mode + essential-info brief | lean `SKILL.md` + `references/` | ✅ done (pushed) |
| SP-3 | `/flow` 2-mode (delegate vs inline) + no-ClickUp intake template + checkpoint hook | lean `SKILL.md` + `references/` | ✅ done (pushed) |
| SP-4 | `/init`: memory-recall doc (2-mode) + **set up standard doc-structure + DOC-POLICY + superpowers override** + **scan existing docs (ask adapt vs leave+generate)** + remove `tidy-session-docs` init | lean `SKILL.md` + `references/` + doc outputs | ✅ done (pushed) |
| SP-5 | **`/wrap` rewrite**: integrated curation (promote durable → permanent, delete ephemeral incl `.session/` + `.superpowers/sdd/`, enforce naming, update docs to current state, context-aware, report) — supersedes `tidy-session-docs`; + new `/checkpoint` | new skill + `/wrap` rewrite | ✅ done (pushed) |
| SP-6 | ClickUp **two-way sync** (update status / create subtask / create task) — outward, confirm each | write ops + `references/clickup-write.md` | 🔜 planned — [future-work](2026-07-22-future-work.md) |
| SP-7 | ClickUp as a **standalone `/clickup` skill**? (management outside the workflow) | tbd | 🔜 planned |
| SP-8 | **Non-systematic usage resilience** (skip /init, /sync, out-of-order) — graceful per skill | per-skill fallback notes | 🔜 planned |

## No-ClickUp intake template (for SP-3)

Canonical intake structure (also the shape a ClickUp task maps onto):

```markdown
# Task Intake
- **Title**: <short>
- **Objective (tujuan)**: <outcome/goal — why it matters>
- **Problem / Need**: <current pain or gap>
- **Requirements**: <functional + non-functional>
- **Constraints**: <stack, compat, deadline, policy, deps>
- **Scope**: IN <…> / OUT <…>
- **Non-goals**: <explicitly NOT doing>
- **Acceptance criteria**: <testable "done when…">
- **Context / Background**: <related systems, history, links>
- **Priority / Urgency**: <critical/high/medium/low + deadline>
- **References**: <people, docs, repos, tickets>
- **Open questions / Assumptions**: <flagged for phase-3 clarify>
```

## Execution order

SP-1 (foundation) → SP-2, SP-3, SP-4, SP-5. Each gets its own spec → plan → implement cycle; this overview is the tracking index.
