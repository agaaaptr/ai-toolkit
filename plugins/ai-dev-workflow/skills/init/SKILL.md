---
name: init
description: One-time project bootstrap for the AI dev workflow. Detects (never assumes) the project's real stack and existing structure, confirms with the user, then scaffolds ONLY the gaps (CLAUDE.md/AGENTS.md/.notes/GUIDE.md/docs/vibes/workflow/.gitignore) and checks that the dependency plugins (Superpowers, context-mode, agentmemory) are installed. Run once per project with /init.
user-invocable: true
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, AskUserQuestion, mcp__plugin_context-mode_context-mode__ctx_index, mcp__agentmemory__memory_save
---

You are bootstrapping a project for the AI dev workflow. **Spine (hard rule): Investigate → Confirm → Act.** Do not create or overwrite anything until you have detected the real setup AND the user has confirmed.

## Procedure

1. **Detect (do not assume).** Scan the project root for:
   - Docs: `CLAUDE.md`, `AGENTS.md`, `README.md`, `docs/`, `vibes/`, `.notes/GUIDE.md`, `.claude/skills/`.
   - Stack manifests: `go.mod`, `package.json`, `composer.json`, `angular.json`, `requirements.txt`, `pom.xml`, `build.gradle`, `Cargo.toml`.
   - Build/test/run commands: read the detected manifest + any `Makefile`, `package.json` scripts, `run*.sh`, `artisan`.
   - Existing `.gitignore`.
   Use `Glob`/`Grep`/`Read`. Note anything **non-conventional** (custom build scripts, monorepo layout, unusual test runner).

2. **Confirm.** Present to the user, via `AskUserQuestion` or a clear summary:
   - "Detected setup: <stack, test cmd, run cmd, conventional/non-conventional notes>."
   - "Existing structure I will NOT touch: <list>."
   - "Gaps I will fill: <list of files/folders to create>."
   Ask the user to confirm or correct. **Never overwrite a mature existing file without explicit confirmation.**

3. **Scaffold only the gaps** (copy from the plugin's `templates/`):
   - `CLAUDE.md`, `AGENTS.md` — only if missing.
   - `.notes/GUIDE.md` — only if missing (from `GUIDE.md.tpl`).
   - `docs/{reference,handoffs,decisions}/`, `vibes/`, `docs/superpowers/{specs,plans}/` — only missing dirs.
   - `workflow/` — create (owned by this workflow).
   - Append `workflow/` (and `.notes/`, `vibes/` if the project uses them) to `.gitignore` using the `gitignore-entries.txt` template — do not duplicate existing entries.

4. **Dependency check.** Verify the three plugins are installed. A simple probe: check whether the `superpowers:*`, `context-mode:*`, and `agentmemory` skills/tools are available (e.g. attempt a harmless `mcp__plugin_context-mode_context-mode__ctx_stats` or check the skills list). For any missing dependency, warn and print the install instruction from the README.

5. **Save a memory** via `mcp__agentmemory__memory_save`: the detected setup + stack of this project (so `/sync`/`/flow` recall it next session).

6. **Report.** Summarize what was created, what was skipped (already present), and any missing dependencies. End by suggesting the user run `/sync`.
