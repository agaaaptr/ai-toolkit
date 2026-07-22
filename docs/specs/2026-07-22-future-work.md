# Future Work — Backlog (SP-6, SP-7, SP-8)

- **Date:** 2026-07-22
- **Status:** Planned (not started) — recorded before session end
- **Roadmap:** [skill-optimization-overview](2026-07-22-skill-optimization-overview.md)

Candidate sub-projects for the next session. Each → spec → plan → implement. Recorded so context survives.

## SP-6 — ClickUp two-way sync

The plugin currently only **fetches** from ClickUp (GET). ClickUp API v2 supports **write** with the same `CLICKUP_API_TOKEN`. All three use cases below are feasible.

| Use case | Endpoint | Where |
|---|---|---|
| Update task status (e.g. on `/flow` done) | `PUT /api/v2/task/{task_id}` (`{"status": ...}`) | `/wrap` (ask first) |
| Create subtask from session work | `POST /api/v2/list/{list_id}/task` + body `parent: <parent_task_id>` | `/wrap` / `/flow` |
| Create new task from a no-ClickUp-id task | `POST /api/v2/list/{list_id}/task` (need target list_id); save returned id to `workflow/<task>.md` | `/flow` phase 7 / `/wrap` |

Sources: [Create Task](https://developer.clickup.com/reference/createtask), [Tasks docs](https://developer.clickup.com/docs/tasks).

### Clarifying questions (resolve before building — no assumptions)
1. **Scope:** all three (update-status + create-subtask + create-task), or subset?
2. **Confirmation:** ask before each write-op (recommended — outward action) or a flag?
3. **Target list for create-task:** env `CLICKUP_DEFAULT_LIST_ID` (one default) or ask per-create (lists differ per task)?
4. **Subtask source:** plan tasks / session commits / user-specified at `/wrap`?
5. **Status mapping:** ClickUp status for "done" — env config (`CLICKUP_DONE_STATUS`) or ask each time?
6. **Token:** does the token have write access to the target workspace? (verify scope)

### Design notes
- Write = outward (like push) → confirm each (spine no-asumsi).
- Non-interactive shell: same `source ~/.zshrc` pattern.
- Likely a shared `references/clickup-write.md` (endpoints + JSON bodies) used by `/wrap` + `/flow`.

## SP-7 — ClickUp as a standalone skill? (`/clickup`)

**Question:** should ClickUp ops be a **separate skill** for management OUTSIDE the dev workflow? E.g., a `/clickup` (or `/cu`) skill for ad-hoc ClickUp management (list / get / update / create tasks) independent of `/flow` + `/wrap`. The workflow skills would call into it.

- **Explore:** is there a need for ClickUp management beyond the workflow? (standalone list/update/create/delete)
- **Scope of standalone ops:** list tasks, get task, update status, create task/subtask, (delete?).
- **Decide:** standalone `/clickup` skill vs keep ClickUp ops embedded in `/flow` + `/wrap` only.
- **Clarifying:** what ClickUp management do you do outside the dev workflow?

## SP-8 — Non-systematic skill usage resilience

**Question:** how should each skill behave when used **out of the normal order** (`/init` → `/sync` → `/flow` → `/wrap`)? Each must **degrade gracefully + guide the user** (no break, no assumption).

**Cases to handle:**
- `/sync` **without `/init`** (no `CLAUDE.md` / standard structure): detect missing setup → suggest `/init`; still produce a brief from what's available.
- `/flow` **without `/sync`**: phase 0 invokes `/sync` (self-heal). **Without `/init`** (no `workflow/`): `/flow` creates `workflow/<task>.md` itself.
- `/wrap` **without `/flow`** (no active task): handle "no active task" — just tidy/commit the session's work.
- **Only `/wrap`** (no /init/sync/flow): tidy whatever's there; don't assume a task.
- `/checkpoint` **without `/flow`**: handle "no active task" (already does).

**Design:** each skill detects missing prerequisites → graceful fallback + suggests the right next step. Document the expected behavior per out-of-order case (probably a `references/usage-order.md` or per-skill fallback notes). Verify at the live PoC.
