# Skill Structure Convention (progressive disclosure)

Every skill uses **progressive disclosure** to stay token-efficient: `SKILL.md` is lean; detail lives in `references/`, loaded on demand. The model Reads a reference only when that step is reached.

## `SKILL.md` shape (target < ~60 lines)
- YAML frontmatter: `name`, `description`, `user-invocable`, `allowed-tools`.
- **When to use** (1–2 lines).
- **Step 0 — Detect mode:** "Read `references/modes.md`; record rich/lean per capability."
- **Core procedure:** numbered steps. Each rich/lean branch kept to **one line** (the decision point); full detail → a `references/` pointer.
- **Fallbacks + confirm gates:** called out inline (one line each); detail in references.
- **References** section listing the `references/*.md` files.

## `references/` layout
- **Cross-cutting** (shared by all skills): `plugins/ai-dev-workflow/references/` — `modes.md`, `skill-structure.md`, `commit-push.md`, `doc-structure.md`.
- **Per-skill:** `plugins/ai-dev-workflow/skills/<skill>/references/<topic>.md`.

## Rule
`SKILL.md` must NOT inline long procedures/templates — move them to `references/`. The lean body is the routing layer; references carry the substance.

## Template

```markdown
---
name: <skill>
description: <one line — when to use>
user-invocable: true
allowed-tools: Read, Edit, Write, Bash, Glob, Grep, Skill, AskUserQuestion
---

<When to use — 1-2 lines>

## Procedure
0. **Detect mode.** Read `references/modes.md` (auto + override). Record rich/lean per capability.
1. <step — one line; rich/lean branch → `references/<x>.md`>
2. <step>
…

## Fallbacks & confirms
- <one-line fallbacks; detail in references>

## References
- `references/modes.md` — mode detection + degradation
- `references/<topic>.md` — <…>
```
