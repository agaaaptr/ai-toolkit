# SP-1: Skill Foundation — 2-Mode Framework + Progressive Disclosure

- **Date:** 2026-07-22
- **Status:** Draft (pending review)
- **Roadmap:** [skill-optimization-overview](2026-07-22-skill-optimization-overview.md)
- **Scope:** Foundation only — the shared mode/discrimination pattern + structure convention. **No skill is restructured here** (that is SP-2…5).

## Context

The plugin currently assumes Superpowers, context-mode, and agentmemory are installed. They are now **optional**: skills must work rich (plugins present) or lean (none), degrading gracefully per capability. Token usage must drop via progressive disclosure. This SP lays the shared foundation every later SP builds on.

## Goals

1. One canonical place that defines mode detection + graceful-degradation (DRY — all skills reference it).
2. A progressive-disclosure structure convention (lean `SKILL.md` + `references/`) that SP-2…5 apply.
3. Document the 2-mode philosophy + token trade-off at the plugin level.
4. Define the commit/push discipline (commit per scope; push = explicit confirm) + **CI-aware post-push** action (read the CI config → offer tag for tag-publish projects / no tag for auto-deploy / ask if unknown) — cross-cutting for all push-capable skills.

## Non-goals (SP-2…5)

- Restructuring `/sync`, `/flow`, `/init`, `/wrap` (SP-2/3/4/5).
- The new `/checkpoint` skill (SP-5).
- The `/init` memory-recall doc (SP-4).

## Design

### File 1 — `plugins/ai-dev-workflow/references/modes.md` (new, shared)

Every skill Reads this on first use. Contents:

**Operating modes.** RICH (plugins → maximal, more tokens), LEAN (no plugins → efficient fallbacks, still systematic). Default AUTO (detect).

**Mode detection (auto + override).**
1. Read override flag `ai-dev-workflow.mode: auto|rich|lean` from `CLAUDE.md`/`AGENTS.md`. If `rich`/`lean`, force it; skip detection.
2. Else (auto) probe each plugin's availability by inspecting the tool/skill list (do **not** invoke heavy tools just to detect):
   - context-mode → is `mcp__plugin_context-mode_context-mode__ctx_search` available?
   - agentmemory → is `mcp__agentmemory__memory_recall` available?
   - superpowers → is skill `superpowers:brainstorming` available?
3. Record which of the three are present.

**Detection is per-capability, not all-or-nothing.** A skill uses the rich path for each capability present and the lean fallback for the rest (e.g. context-mode present but agentmemory absent → rich retrieval, lean memory).

**Capability matrix (rich → lean).**

| Capability | Detect | Rich | Lean fallback |
|---|---|---|---|
| Retrieval | context-mode tools | `ctx_index`/`ctx_search` | `Read` on-demand + `Grep` |
| Memory | agentmemory tools | `memory_recall`/`memory_save` | native `MEMORY.md` + `workflow/<task>.md` |
| Process | superpowers skills | delegate (brainstorming/writing-plans/executing-plans/…) | inline lean version (per-skill `references/`) |

**Graceful-degradation rules.**
- Never fail silently: if a rich tool is unavailable, use the documented fallback **and** tell the user (e.g. "agentmemory not installed — using MEMORY.md").
- Probe cheaply (introspect the list, don't call heavy tools to detect).
- On any uncertainty about availability → ASK the user (no assumption).

**Spine (both modes).** Investigate → Confirm → Act. Confirm at each gate. No execution before facts + user confirmation. Holds rich and lean.

### File 2 — `plugins/ai-dev-workflow/references/skill-structure.md` (new, shared)

The progressive-disclosure convention SP-2…5 apply. Contents:

**Principle.** `SKILL.md` stays lean (core + decision points); detail (long procedures, templates, fallback tables, examples) lives in `references/`, loaded on demand. Saves tokens (model Reads a reference only when that step is reached).

**`SKILL.md` shape (target < ~60 lines).**
- YAML frontmatter: `name`, `description`, `user-invocable`, `allowed-tools`.
- "When to use" (1–2 lines).
- Step 0 — Detect mode: "Read `references/modes.md`; record rich/lean per capability."
- Core procedure: numbered steps. Each rich/lean branch kept to **one line** (the decision point); full detail → a `references/` pointer.
- Fallbacks + confirm gates called out inline (1 line each); detail in references.
- "References" section listing the `references/*.md` files.

**`references/` layout.**
- Cross-cutting (shared by all skills): `plugins/ai-dev-workflow/references/{modes,skill-structure}.md`.
- Per-skill: `plugins/ai-dev-workflow/skills/<skill>/references/<topic>.md`.

**Rule.** `SKILL.md` must NOT inline long procedures/templates — those go to `references/`. The lean body is the routing layer; references carry the substance.

### File 3 — `plugins/ai-dev-workflow/references/commit-push.md` (new, shared)

Cross-cutting discipline every push-capable skill (`/flow`, `/wrap`, future `/checkpoint`) follows.

**Commit.** Per scope (conventional commits). Never bundle unrelated changes.

**Push.** Push **requires explicit user confirmation** — never auto-push. Confirm branch + commits before each push.

**Post-push action (CI-aware, no assumption).** After a confirmed push, read the project's CI config to decide the relevant next action:
1. Detect CI: `.gitlab-ci.yml`, `.github/workflows/*.yml`, `.circleci/config.yml`, `Jenkinsfile`, …
2. Determine the release mechanism from the config:
   - **Tag-publish** (publish/deploy triggered by a version tag): GitLab `rules: $CI_COMMIT_TAG` / `only: tags`; GitHub `on: push: tags:`. → ask "tag a version?" (run the version-bump+tag flow).
   - **Auto-deploy** (deploy triggered by branch push): deploy job on `develop` / `staging` / `master`. → do **not** ask about a tag; note "CI will auto-deploy".
   - **Both / neither / unknown** → ASK the user what the release flow is.
3. Corroborate with project type:
   - FE Angular `@uiigateway/*` library → tag-publish → offer version-bump + tag.
   - BE service (`go.mod` → Go; `composer.json` → PHP / Lumen / Laravel) → auto-deploy → no tag.
4. Never assume — if the CI config is unclear, ask.

**Version-bump + tag flow** (tag-publish projects) → the `/flow` Phase 7 procedure (tag-first, plain version, counter, env suffix).

> Note: the **currently shipped** `/flow` Phase 7 pushes + tags inline without the explicit push-confirm / CI-driven post-push question. SP-3 (the `/flow` restructure) brings it in line with this reference.

### File 4 — README.md + AGENTS.md updates

Add a "Two modes (rich / lean)" section to both:
- Explain rich vs lean + token trade-off.
- The `ai-dev-workflow.mode` override flag.
- Pointers to `references/modes.md` and `references/skill-structure.md`.
- Note the three plugins are now **optional** (was: required).

### File 5 — `plugins/ai-dev-workflow/references/doc-structure.md` (new, shared)

The standard project doc layout + curation rules. Derived by analyzing two real `tidy-session-docs` references (BE `svc-academic-activity-go`, FE `lib-uii-gateway-academy-angular`) into ONE standard. Applied by `/init` (SP-4) and `/wrap` (SP-5).

**Permanent (committed, keep — never delete):**
- `docs/specs/` (design specs), `docs/plans/` (implementation plans — **kept as records**), `docs/decisions/` (ADRs), `docs/architecture/`, `docs/reference/` (durable: DB, conventions), `docs/handoffs/` (BE↔FE), `docs/findings/` (validation/incident).
- `docs/DOC-POLICY.md` — declares this classification + naming + red-flags (the FE pattern).
- `docs/reference/project-memory.md` — `/init`-generated **memory-recall doc** (relevant agentmemory memories + their recall queries); lean fallback (no agentmemory) = native root `MEMORY.md`. (Detail in SP-4.)

**Ephemeral (gitignored, promote-or-delete each session — only durable survives):**
- `.session/` — neutral session scratch.
- `.superpowers/sdd/` — SDD default scratch (not overrideable); **contents cleaned each session end**.

**Naming:** `YYYY-MM-DD-<scope>-<slug>.md`.

**superpowers override (committed docs):** specs → `docs/specs/`, plans → `docs/plans/` (AGENTS.md preference; prevents stray `docs/superpowers/`).

**Curation rules (for `/wrap`):** permanent = keep; ephemeral = promote durable content → permanent, delete the rest; stale/inaccurate permanent → fix or delete (don't keep garbage); sole record of a non-obvious decision → promote to `decisions/`, never delete.

**`/init` scan behavior (for SP-4):** when `/init` scans the project + docs, if it finds an **existing doc structure or similar docs**, ASK the user: (a) adapt the existing to this standard, or (b) leave existing as-is and generate the standard docs alongside. ai-dev-workflow standard docs are **always generated**; the fate of existing docs is the user's call (move / leave) — no assumption.

## Files affected (SP-1)

- Create: `plugins/ai-dev-workflow/references/modes.md`
- Create: `plugins/ai-dev-workflow/references/skill-structure.md`
- Create: `plugins/ai-dev-workflow/references/commit-push.md`
- Create: `plugins/ai-dev-workflow/references/doc-structure.md`
- Modify: `README.md` (2-mode section; deps optional)
- Modify: `AGENTS.md` (2-mode section; deps optional)

## Acceptance criteria

- [ ] `references/modes.md` defines detection (auto+override), per-capability matrix, graceful-degradation rules, spine.
- [ ] `references/skill-structure.md` defines the lean-`SKILL.md` + `references/` convention with a template.
- [ ] `references/commit-push.md` defines commit-per-scope, push=explicit-confirm, CI-aware post-push (tag-publish → offer tag; auto-deploy → no tag; unknown → ask).
- [ ] `references/doc-structure.md` defines the standard layout (permanent/ephemeral), DOC-POLICY, naming, superpowers override, curation rules, /init scan behavior.
- [ ] README + AGENTS document 2-mode + optional plugins + the override flag + pointers.
- [ ] No existing skill is changed (deferred to SP-2…5).
- [ ] Markdown valid; no placeholders.

## Decision log

| # | Decision |
|---|---|
| Mode detection | Auto (probe tool/skill list) + override flag (`ai-dev-workflow.mode`) |
| Granularity | Per-capability rich/lean (not all-or-nothing) |
| Structure | Progressive disclosure: lean `SKILL.md` + `references/` |
| Commit/push | Commit per scope; push = explicit confirm; post-push = CI-aware (read CI config: tag-publish → offer tag / auto-deploy → no tag / unknown → ask) |
| Doc structure | Standard layout: permanent `docs/{specs,plans,decisions,architecture,reference,handoffs,findings}` + `DOC-POLICY.md`; ephemeral `.session/` + `.superpowers/sdd/` (cleaned each session); superpowers override → `docs/specs`,`docs/plans`; plans permanent; `/init` asks adapt-existing vs leave+generate |
| Scope | Foundation only; skill restructure = SP-2…5 |
