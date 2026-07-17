---
name: wrap
description: Close a work session cleanly. Runs the stack-detected test command, updates API-CONTRACT/docs/decisions/docs/handoffs as needed, then invokes the tidy-session-docs skill to promote durable docs and delete ephemeral scratch. Call from any /flow phase to stop cleanly, or at true session end. Run with /wrap.
user-invocable: true
allowed-tools: Bash, Read, Edit, Write, Skill, Glob
---

You are closing a session. **Spine: Investigate → Confirm → Act** — confirm what changed before claiming it's done.

## Procedure

1. **Investigate what changed.** Run `git status` + `git diff --stat`. Identify which docs/contracts the session's work touched.

2. **Run tests.** Determine the test command from `CLAUDE.md` or the detected manifest and run it. Show the output. If tests fail, STOP and report — do not declare success (verification-before-completion).

3. **Update docs.** Prompt the user (or infer from the diff) whether `API-CONTRACT.md`, `docs/decisions/`, or `docs/handoffs/` need updates for this session's changes. Make the edits.

4. **Tidy.** Invoke the `tidy-session-docs` skill to promote durable docs from `vibes/` → `docs/` and delete ephemeral scratch.

5. **Report.** Summarize: test result (with evidence), docs updated, files promoted/deleted, and any unfinished work carried in `workflow/<task>.md`.

## Hard rules
- Never claim tests pass without showing their output.
- Do not delete anything in `docs/` (stable) — only `tidy-session-docs` decides promotions.
