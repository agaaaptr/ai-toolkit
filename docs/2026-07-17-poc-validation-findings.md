# ai-dev-workflow — PoC Validation Findings

- **Date:** 2026-07-17
- **PoC project:** `svc-academic-activity-go` (Go backend, branch `experiment/ai-dev-workflow`)
- **Testers:** User + Claude
- **Purpose:** find gaps/bugs/unexpected behaviors BEFORE others consume the plugin, so it ships mature. Each finding → a plugin improvement (issue/fix).

## Summary

| ID | Finding | Severity | Status |
|---|---|---|---|
| F1 | `tidy-session-docs` was a hard dependency (not bundled/declared) → `/flow` phase 7 + `/wrap` hard-fail in projects lacking it | High | **FIXED** (graceful invocation + declared optional) |
| F2 | ClickUp token invisible to the non-interactive Bash shell (no `source ~/.zshrc`) → `/flow` auto-fetch silently fell back to manual paste | High | **FIXED** (`/flow` phase 1 now `source ~/.zshrc`) |
| F3 | `agentmemory` save→recall returns nothing (`memory_save` returns IDs but `memory_sessions=[]`, `recall`/`smart_search` empty) → cross-session memory promise unmet | **Critical** | **OPEN** |
| F4 | Docs referenced plugin path `/AI` instead of the real `/AI/ai-toolkit` | Low | **FIXED** (spec/handoff/plan corrected) |
| F5 | (appended during `/flow` dry-run) | — | — |

## Details

### F1 — tidy-session-docs hard dependency (FIXED)
- **Where:** `/flow` phase 7, `/wrap` step 4.
- **Symptom:** invoked `tidy-session-docs`, which exists only as a project-scope skill in the PoC repo — absent in any other project → those two doc steps hard-failed.
- **Root cause:** design referenced it but never declared it a dependency (spec §6.4) nor bundled it.
- **Fix applied:** graceful invocation ("invoke `tidy-session-docs` if available, else inline minimal convention-detecting tidy + confirm with user"); declared optional companion in spec §6.4 + README. Commits `40bb761` (plugin) + `375673b` (spec).

### F2 — ClickUp token invisible to non-interactive Bash shell (FIXED)
- **Where:** `/flow` phase 1 (`$CLICKUP_API_TOKEN`).
- **Symptom:** the Claude Code Bash tool runs a **non-interactive** shell that does **not** auto-source `~/.zshrc`; the token (exported there) was invisible → curl sent an empty `Authorization` header → ClickUp `400 OAUTH_017` → `/flow` silently fell back to manual paste even though ClickUp was correctly set up.
- **Verified:** `source ~/.zshrc >/dev/null 2>&1` makes the token visible (HTTP 200 on a real task).
- **Fix applied:** `/flow` phase 1 now runs `source ~/.zshrc` before reading the token. Commit `80aab81`.
- **Note:** this is a general gotcha — ANY env var the user sets in `~/.zshrc` is invisible to the Bash tool. The workflow should source it (or the user should put exports in `~/.zshenv`). Worth documenting in README.

### F3 — agentmemory save→recall broken (OPEN, CRITICAL)
- **Where:** `/sync` step 5 (recall), `/flow` phase 3 (save memory), and the cross-session/cross-directory transfer promise (handoff doc).
- **Symptom:** `mcp__agentmemory__memory_save` consistently returns IDs (`mem_…`), BUT `memory_sessions` = `[]`, and `memory_recall` / `memory_smart_search` return **empty** for any query. Saves are accepted but **not retrievable**.
- **Impact (severe):** the workflow's core promise — "context survives via agentmemory across sessions / across the directory switch" — is currently **unmet**. `/sync` recall is empty; `/flow` phase-3 memory saves can't be recalled; the handoff's "recall in the new dir" mechanism won't work through agentmemory.
- **Mitigating fact:** the harness's **native** file-based memory (`~/.claude/.../memory/MEMORY.md`) DOES work (auto-loaded this session).
- **Recommendation:** (a) investigate the agentmemory backend in this env (is it a stub / no DB / project-key scoping mismatch?), and/or (b) **pivot `/sync` + `/flow` to the native MEMORY.md** (file-based, reliable) as the primary memory, keeping agentmemory optional. Update spec §10 + the skills accordingly.
- **Status:** OPEN — blocks the "survive context loss" claim until resolved.

### F4 — doc path mismatch (FIXED)
- **Where:** spec §6, handoff, plan.
- **Symptom:** all referenced the plugin repo as `/Users/.../AI`; the real path is `/Users/.../AI/ai-toolkit` (subdir).
- **Fix applied:** corrected across all docs; verified against the actual repo. Commit `375673b`.

## `/flow` dry-run validation results (2026-07-17, task 86ey844rx)

- [x] Running `/flow` loads the NEW cache `80aab816624f` (both fixes present). **Plugin marketplace-update + reload correctly swaps the active skill to the new commit.** ✓
- [x] Phase 1 ClickUp auto-fetch works end-to-end — token visible after `source ~/.zshrc`, HTTP 200, task fields populated. **F2 fix validated.** ✓
- [x] Phase 2 subagent investigate returns compact facts; main context kept clean. ✓
- [x] Phase 3 HARD gate stops before any execution. ✓
- [x] `workflow/86ey844rx.md` state file written + updated per phase. ✓ (minor lint — see F6)

### F5 — sparse ClickUp task handling (OPEN, Low)
- **Symptom:** task 86ey844rx has empty `description` + `text_content` (only title + status "ready to test"). `/flow` intake had no acceptance criteria to record.
- **Impact:** `/flow` must treat empty ClickUp descriptions as "needs user clarification at phase 3," never assume scope.
- **Recommendation:** add an explicit note to `/flow` phase 1 ("if description empty, flag for phase-3 clarification") and to the `workflow-state.md.tpl` "Acceptance criteria" section.

### F6 — workflow-state template markdown lint (OPEN, Trivial)
- **Symptom:** `templates/workflow-state.md.tpl` puts lists directly under headings without blank lines → generated `workflow/<task>.md` triggers markdownlint MD022/MD032.
- **Impact:** cosmetic (state file is personal/gitignored), but noisy in linting editors.
- **Recommendation:** add blank lines between headings and lists in `templates/workflow-state.md.tpl`.

### Positive findings (mechanisms that work)
- Plugin marketplace update + reload swaps the active skill to the new commit cache (`80aab816624f`). ✓
- `source ~/.zshrc` makes the ClickUp token visible to the Bash tool. ✓
- Graceful tidy invocation is present in the loaded `/flow` (F1 fix live). ✓
- Subagent delegation (phase 2) returns a clean summary, keeping the orchestrator context lean. ✓

## Open improvement backlog (for the ai-toolkit plugin)
1. **F3 (Critical):** agentmemory save→recall broken in this env — investigate backend OR pivot `/sync`+`/flow` to the harness native `MEMORY.md`. Until resolved, the "survive context loss via agentmemory" claim is unmet.
2. **F5 (Low):** bake "empty ClickUp description → clarify at phase 3" into `/flow` phase 1 + the state template.
3. **F6 (Trivial):** fix `workflow-state.md.tpl` markdown spacing.
4. **Docs:** add the "Bash tool doesn't source ~/.zshrc" gotcha to the README (general env-var note).
