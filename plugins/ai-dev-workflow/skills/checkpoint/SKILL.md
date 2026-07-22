---
name: checkpoint
description: Save a mid-session checkpoint so work survives context loss or interruption. Updates workflow/<task>.md with phase + status + done/next, and (if agentmemory) saves a memory. Use anytime before a context-risky moment, or on interruption. Resume next session with /flow <task>. Run /checkpoint.
user-invocable: true
allowed-tools: Read, Write, Edit, Bash, AskUserQuestion, mcp__agentmemory__memory_save
---

Save a checkpoint. **Spine: Investigate → Confirm → Act.**

## Procedure
1. **Find the task.** Locate `workflow/<task>.md` (the active task state). If none, ask the user which task — or note "no active task — nothing to checkpoint".
2. **Capture state.** Update `workflow/<task>.md`: set `phase:`, `status:` (e.g. `checkpointed @ phase N`), `updated:` timestamp, append a History-log line, and fill "done so far" + "next steps" + any blockers.
3. **Memory (best-effort).** If agentmemory is available, `memory_save` the checkpoint (phase + key decisions + next). Lean: skip (the state file is the source of truth).
4. **Uncommitted work.** Note any uncommitted changes (`git status --porcelain`) in the checkpoint and advise the user (commit or stash before truly leaving).
5. **Report.** "Checkpoint saved → resume with `/flow <task>`." + the phase + the next step.

## Fallbacks
- No `workflow/<task>.md` → ask which task, or create a minimal one.
- No agentmemory → state file only (still durable).

## References
- `/flow` `references/phases.md` — checkpoint format details
