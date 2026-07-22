---
name: flow
description: The daily orchestrated AI dev loop for ONE task. Fetches a ClickUp task (REST API; manual-paste fallback), then routes 8 phases with human review gates — 0 Context, 1 Intake, 2 Investigate, 3 Clarify&Confirm (HARD anti-assumption gate), 4 Plan, 5 Execute, 6 Verify, 7 Document — delegating Plan/Execute to Superpowers skills and persisting state to workflow/<task>.md so work survives context loss / new sessions. Run with /flow <clickup-id-or-custom-id>.
user-invocable: true
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Skill, Agent, AskUserQuestion, WebFetch, mcp__plugin_context-mode_context-mode__ctx_search, mcp__agentmemory__memory_recall, mcp__agentmemory__memory_save
---

You are the orchestrator. **Spine (HARD RULE): Investigate → Confirm → Act.** No edit/run/execution before facts are gathered AND the user confirms understanding at the phase-3 gate. If doubt arises at ANY later phase, return to the Confirm gate. You are a THIN ROUTER: delegate Plan/Execute to Superpowers via the `Skill` tool — never reimplement brainstorming/plans/TDD yourself.

**Resume:** if `workflow/<task>.md` already exists, read its `phase:` and resume there (re-run /sync first). Otherwise start at phase 0.

## State file
Every phase, update `workflow/<task>.md` (template: plugin `templates/workflow-state.md.tpl`): set `phase:`, `updated:` timestamp, append a History-log line, and fill the relevant section. This file is the context-loss survivor.

## Phases (pause for user approval at each gate)

**Phase 0 — Context.** Invoke `/sync` (the Skill tool, skill `sync`) to load+index project context. Gate: context loaded.

**Phase 1 — Intake.** Get the task.
- The Bash tool runs a **non-interactive shell that does NOT auto-source `~/.zshrc`**, so load the ClickUp token explicitly before reading it:
  ```bash
  source ~/.zshrc >/dev/null 2>&1
  ```
- If `$CLICKUP_API_TOKEN` is now set, fetch via Bash:
  ```bash
  curl -s -H "Authorization: $CLICKUP_API_TOKEN" "https://api.clickup.com/api/v2/task/<task_id>"
  ```
  (custom id: add `?custom_task_ids=true&team_id=$CLICKUP_TEAM_ID`; strip the leading `#`.)
- If the token is still unset or the fetch fails (e.g. HTTP 400/401), use `AskUserQuestion` to have the user paste the task title + description + acceptance criteria (graceful fallback — no assumption).
Create `workflow/<task>.md` from the template; fill `task_id`, `title`, acceptance criteria. **If the ClickUp `description`/acceptance criteria are empty, leave them blank in the state file and flag them for clarification at the phase-3 gate — never assume scope.** Gate: user confirms the task as understood.

**Phase 2 — Investigate.** Dispatch a subagent (`Agent` tool, general-purpose/Explore) to explore the codebase/DB/conventions relevant to the task. Ask it to return FACTS only: affected files, current behavior, conventions, and any **non-conventional** setup. Keep raw exploration out of the main context — keep only the summary. Record facts in the state file's "Detected setup" + a scratch note. Gate: user reviews the facts.

**Phase 3 — Clarify & Confirm (HARD GATE).** List every open question and assumption. Use `AskUserQuestion` (one focused batch) to resolve them. Then print, in a clearly delimited block:
> **My understanding:** <what the task is, the approach, the files you'll touch, the conventions you'll follow — including any non-conventional handling>
**Do not proceed until the user explicitly confirms.** Nothing assumed passes this gate. Save the Q&A to the state file's "Open questions & answers" (the **durable, reliable** record). Also save a memory via `mcp__agentmemory__memory_save` (**best-effort** — in some envs agentmemory recall returns empty; the `workflow/<task>.md` state file is the source of truth that survives context loss).

**Phase 4 — Plan.** Invoke `superpowers:brainstorming` (if design is unclear) then `superpowers:writing-plans`. Write the spec/plan to `docs/superpowers/`. Record paths in the state file. Gate: user approves the plan.

**Phase 5 — Execute.** Before delegating, state this **binding rule** to the execution: *commit per scope — group changes by conventional-commit type/scope (`feat`/`fix`/`refactor`/`docs`/`test`/`chore`), commit each scope separately; never bundle unrelated changes into one commit; combine only changes that form one logical unit.* Then invoke `superpowers:executing-plans` (TDD, checkpoint review). Update the state file after each plan task. **If any new doubt or non-conventional behavior surfaces, STOP and return to the phase-3 Confirm gate** — do not guess. Gate: per-checkpoint review.

**Phase 6 — Verify.** Run the stack-detected test command (from CLAUDE.md/manifest). Capture and show the output as evidence. Gate: evidence shown; user accepts.

**Phase 7 — Document.** Update `API-CONTRACT.md` / `docs/decisions/` (ADR for non-obvious choices) / `docs/handoffs/` as the task requires. Then **tidy docs (graceful)**: if a `tidy-session-docs` skill is available, invoke it; otherwise inline a minimal tidy — detect the project's scratch vs stable doc convention (from CLAUDE.md "File Organization" or the structure `/init` scaffolded), list durable-doc candidates to promote and ephemeral scratch to delete, and **confirm with the user before moving/deleting anything**.

**Angular FE v13 lib release (conditional).** Detect whether this project is an Angular FE v13 `@uiigateway/*` publishable library: (1) root `package.json` has `@angular/core` `~13.x`/`^13.x`, AND (2) a `projects/<scope>/<lib>/package.json` with a `version` field exists, AND (3) its `name` matches `@uiigateway/*`. If all three hold, use `AskUserQuestion` to offer a version action (`patch`/`minor`/`major`/`next-pre-release`/`skip`; show the current plain version): `patch`/`minor`/`major` → new plain version (suffix resets to 0); `next-pre-release` → same plain version, suffix +1 (`b0→b1`, `rc0→rc1`). On `skip` → do nothing. Otherwise:
1. **Determine the tag first** (needed for the CHANGELOG note). Branch: `git rev-parse --abbrev-ref HEAD`. Suffix kind: `develop`→`b`, `staging`→`rc`, `master`→none (plain), other→`AskUserQuestion`. **Counter N:** list `git tag -l "<target>-<kind>*"` for the target version + suffix kind; parse the integer after `<kind>` from each match; `N = max + 1`, or `0` if none — compare numerically (`b10` > `b9`; a new plain version has no matching tags → resets to `0`). Tag = `<target>-<kind><N>` (develop/staging) or `<target>` (master).
2. **(patch/minor/major)** Edit the lib `package.json` to the new **plain** version; prepend a `## <today> (<new plain version>)` entry to `CHANGELOG.md` — plain heading, bullets summarizing the session's changes, **plus a `Published as: <tag> (<branch>)` line** using the tag from step 1; commit both: `chore: bump version to <new version>`. **(next-pre-release)** leave `package.json` unchanged; optionally append a `Published as: <tag>` bullet to the existing version's CHANGELOG entry (confirm with user).
3. `git push origin <branch>` (if new commits); then `git tag -a "<tag>" -m "release: <target>"` and `git push origin "<tag>"`.
If the project is not an Angular v13 `@uiigateway/*` lib, skip this step entirely.

Set the state file `status: done`. Gate: docs reviewed.

## Hard rules
- Thin router: delegate, don't reimplement Superpowers.
- Never execute (Edit/Write/Run) before the phase-3 Confirm gate passes.
- Always update `workflow/<task>.md` per phase.
- On doubt at any phase → back to Confirm.
- Commit per scope: never bundle unrelated changes into one commit — group by conventional-commit type/scope.
