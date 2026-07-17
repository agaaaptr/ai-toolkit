---
name: sync
description: Load and index project context at the start of a session (explicit trigger, read-only, idempotent). Reads CLAUDE.md/AGENTS.md, indexes .notes/GUIDE.md and docs/ into context-mode for just-in-time retrieval (no large dumps into the context window), reads git state, detects and reports the project's real setup for confirmation, recalls relevant facts from agentmemory, and prints a session brief (with an offer to resume any in-progress task). Run with /sync whenever you begin a session or need a project refresher.
user-invocable: true
allowed-tools: Read, Bash, Glob, Grep, mcp__plugin_context-mode_context-mode__ctx_index, mcp__plugin_context-mode_context-mode__ctx_search, mcp__agentmemory__memory_recall, mcp__agentmemory__memory_smart_search
---

You are loading project context. This skill is **read-only** — do not edit project files. **Spine: Investigate → Confirm → Act** applies to *reporting* setup, not mutating.

## Procedure

1. **Read the anchors.** Read `CLAUDE.md` and `AGENTS.md` (these are always-on but re-read to ground the brief).

2. **Index big knowledge for JIT retrieval** (keep it OUT of the context window). Call `mcp__plugin_context-mode_context-mode__ctx_index` on:
   - `.notes/GUIDE.md` (if present) — `source: "GUIDE.md"`.
   - `docs/` directory (if present) — `source: "docs"`.
   Do NOT print their contents. Subsequent lookups use `mcp__plugin_context-mode_context-mode__ctx_search`.

3. **Read git state.** Run via Bash:
   ```bash
   git rev-parse --abbrev-ref HEAD
   git status --porcelain
   git log --oneline -3
   ```

4. **Detect + report setup.** Determine stack + test/run commands (from manifests/CLAUDE.md). Flag anything non-conventional. This is informational (no mutation here).

5. **Recall memory (resilient).** Try `mcp__agentmemory__memory_recall` (or `memory_smart_search`) with the project name/concepts to surface prior decisions and gotchas. **If it returns empty** (agentmemory recall is observed empty in some environments), fall back to `mcp__plugin_context-mode_context-mode__ctx_search` — the persistent context-mode KB auto-captures session decisions and is reliable — and check the native `MEMORY.md`. Surface whatever prior facts you find.

6. **Check for in-progress work.** If `workflow/` exists and contains `<task>.md` files, read their `phase:` frontmatter and surface: "In-progress task(s): <id> at phase <n> — resume with /flow <id>."

7. **Dependency check.** If context-mode or agentmemory tools are unavailable, warn (they are required for full `/sync`).

8. **Print the session brief** (concise):
   - Detected setup (stack, test/run cmd, conventional/non-conventional).
   - Current branch + uncommitted file count + last 3 commits.
   - Top recalled facts (2-4 bullets).
   - In-progress task offer (if any).
   - Recommended next step (e.g. "/flow <task>" or "describe what you want to do").

End by awaiting the user's direction. Do not auto-start work.
