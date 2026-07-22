# SP-1 Skill Foundation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development or superpowers:executing-plans. Steps use `- [ ]` checkboxes.

**Goal:** Create the shared foundation (4 reference docs + README/AGENTS update) that every later SP builds on. No skill is restructured here.

**Architecture:** Foundation = cross-cutting `references/*.md` at `plugins/ai-dev-workflow/references/` (modes, skill-structure, commit-push, doc-structure) + plugin-level README/AGENTS documenting 2-mode philosophy. Skills (SP-2…5) will Read these on first use.

**Tech Stack:** Markdown reference docs (Claude Code skill `references/` convention).

**Spec:** `docs/specs/2026-07-22-sp1-skill-foundation.md`

## Global Constraints

- These are **reference docs** (progressive disclosure) — written to be Read on demand by skills, so keep them self-contained, skimmable, no placeholders.
- Commit **per scope** (conventional commits: `docs(ai-dev-workflow): …`).
- Push only on user confirmation (SP-1 creates local commits; push is a separate user-approved step).
- No existing skill (`init`/`sync`/`flow`/`wrap` `SKILL.md`) is modified in SP-1.

---

### Task 1: Create `references/modes.md`

**Files:** Create `plugins/ai-dev-workflow/references/modes.md`.

**Content** (per spec §File 1 — expand to full prose):
- Title + purpose ("Every skill Reads this on first use to detect mode + degrade gracefully").
- **Operating modes**: RICH (plugins → maximal, more tokens) / LEAN (no plugins → efficient, still systematic). Default AUTO.
- **Mode detection (auto + override)**: (1) read override `ai-dev-workflow.mode: auto|rich|lean` from `CLAUDE.md`/`AGENTS.md` → force if rich/lean; (2) else probe each plugin by inspecting the tool/skill list (cheaply, no heavy calls): context-mode (`mcp__plugin_context-mode_context-mode__ctx_search`), agentmemory (`mcp__agentmemory__memory_recall`), superpowers (skill `superpowers:brainstorming`); (3) record which present.
- **Per-capability, not all-or-nothing**: rich path per capability present, lean fallback for the rest.
- **Capability matrix** table: Retrieval / Memory / Process × (detect, rich, lean fallback) — exact rows from spec.
- **Graceful-degradation rules**: never silent-fail; use documented fallback + tell user; probe cheaply; on uncertainty → ASK.
- **Spine (both modes)**: Investigate → Confirm → Act; confirm at gates; no execution before facts + user confirmation.

- [ ] Step 1: Write the file with the content above (full prose, ~40–60 lines).
- [ ] Step 2: Verify `grep -c 'ai-dev-workflow.mode' file` ≥1; `grep -c 'ctx_search\|memory_recall\|superpowers:brainstorming' file` ≥3; table present.
- [ ] Step 3: Commit: `git add plugins/ai-dev-workflow/references/modes.md && git commit -m "docs(ai-dev-workflow): add references/modes.md (2-mode + graceful degradation)"`

### Task 2: Create `references/skill-structure.md`

**Files:** Create `plugins/ai-dev-workflow/references/skill-structure.md`.

**Content** (per spec §File 2):
- Title + principle (progressive disclosure: lean `SKILL.md`, detail in `references/`, loaded on demand → token-efficient).
- **`SKILL.md` shape** (target <~60 lines): frontmatter (`name`, `description`, `user-invocable`, `allowed-tools`); "When to use" (1–2 lines); Step 0 detect mode (Read `references/modes.md`); core procedure (numbered, each rich/lean branch one line → pointer to `references/`); fallbacks + confirm gates 1 line each; "References" section listing `references/*.md`.
- **`references/` layout**: cross-cutting at `plugins/ai-dev-workflow/references/`; per-skill at `skills/<skill>/references/`.
- **Rule**: `SKILL.md` must NOT inline long procedures/templates — move to `references/`.
- A minimal **template** `SKILL.md` skeleton.

- [ ] Step 1: Write the file (~40–50 lines incl. template).
- [ ] Step 2: Verify `grep -c 'progressive disclosure' file` ≥1; template code-fence balanced.
- [ ] Step 3: Commit: `git add …/skill-structure.md && git commit -m "docs(ai-dev-workflow): add references/skill-structure.md (progressive disclosure convention)"`

### Task 3: Create `references/commit-push.md`

**Files:** Create `plugins/ai-dev-workflow/references/commit-push.md`.

**Content** (per spec §File 3):
- Title + scope ("every push-capable skill follows this").
- **Commit**: per scope (conventional commits); never bundle unrelated.
- **Push**: requires explicit user confirmation; never auto-push; confirm branch + commits first.
- **Post-push (CI-aware, no assumption)**: (1) detect CI (`.gitlab-ci.yml`, `.github/workflows/*.yml`, `.circleci/config.yml`, `Jenkinsfile`); (2) determine release mechanism — **tag-publish** (`$CI_COMMIT_TAG` rules / `only: tags` / `on: push: tags`) → ask "tag a version?"; **auto-deploy** (deploy on branch push) → no tag, note "CI will auto-deploy"; **both/neither/unknown** → ASK; (3) corroborate project type (FE Angular `@uiigateway/*` → tag-publish; BE `go.mod`/`composer.json` → auto-deploy); (4) never assume — unclear CI → ask.
- **Version-bump+tag flow** pointer → `/flow` Phase 7.
- Note: currently-shipped `/flow` doesn't yet follow this; SP-3 aligns it.

- [ ] Step 1: Write the file (~40–50 lines).
- [ ] Step 2: Verify `grep -c 'tag-publish' file` ≥1; `grep -c 'auto-deploy' file` ≥1; CI filenames present.
- [ ] Step 3: Commit: `git add …/commit-push.md && git commit -m "docs(ai-dev-workflow): add references/commit-push.md (commit/push discipline + CI-aware post-push)"`

### Task 4: Create `references/doc-structure.md`

**Files:** Create `plugins/ai-dev-workflow/references/doc-structure.md`.

**Content** (per spec §File 5):
- Title + source ("derived from BE + FE tidy-session-docs references → one standard").
- **Permanent (committed, keep)**: `docs/{specs,plans,decisions,architecture,reference,handoffs,findings}/` + `docs/DOC-POLICY.md` + `docs/reference/project-memory.md` (`/init` memory-recall doc). Plans = permanent.
- **Ephemeral (gitignored, promote-or-delete each session)**: `.session/`; `.superpowers/sdd/` (SDD default, contents cleaned each session end).
- **Naming**: `YYYY-MM-DD-<scope>-<slug>.md`.
- **superpowers override**: specs → `docs/specs/`, plans → `docs/plans/` (AGENTS.md; prevents `docs/superpowers/`).
- **Curation rules** (for `/wrap`): permanent keep; ephemeral promote-durable-then-delete; stale permanent → fix or delete; sole-record decision → promote to `decisions/`.
- **`/init` scan behavior**: if existing doc structure/similar docs → ASK adapt-to-standard vs leave+generate-new; standard always generated.
- **DOC-POLICY.md template** (classification + naming + red-flags, FE-style).

- [ ] Step 1: Write the file (~60–80 lines incl. DOC-POLICY template).
- [ ] Step 2: Verify `grep -c 'DOC-POLICY' file` ≥1; permanent/ephemeral sections present; `.session/` + `.superpowers/sdd/` mentioned.
- [ ] Step 3: Commit: `git add …/doc-structure.md && git commit -m "docs(ai-dev-workflow): add references/doc-structure.md (standard layout + DOC-POLICY + curation)"`

### Task 5: Update `README.md` (2-mode + optional plugins)

**Files:** Modify `README.md` (the ai-toolkit plugin README).

**Content**:
- Change the Prerequisites section: the three plugins are now **optional** (was: required). Rich mode if present, lean if absent.
- Add a **"Two modes (rich / lean)"** subsection: explain rich (maximal, more tokens) vs lean (efficient, still systematic) + the `ai-dev-workflow.mode: auto|rich|lean` override flag.
- Pointers to `plugins/ai-dev-workflow/references/{modes,skill-structure,commit-push,doc-structure}.md`.
- Remove the `tidy-session-docs` "optional companion" line (superseded — integrated into `/wrap` in SP-5). *[SP-5 will fully wire /wrap; here we just drop the companion mention.]*

- [ ] Step 1: Edit README (Prerequisites → optional; add Two-modes subsection; drop tidy-session-docs companion line; add references pointers).
- [ ] Step 2: Verify `grep -c 'optional' README.md` increased; `grep -c 'tidy-session-docs' README.md` == 0; `grep -c 'ai-dev-workflow.mode' README.md` ≥1.
- [ ] Step 3: Commit: `git add README.md && git commit -m "docs: README — 2-mode (rich/lean), plugins optional, drop tidy-session-docs companion"`

### Task 6: Update `AGENTS.md` (2-mode + override + pointers)

**Files:** Modify `AGENTS.md`.

**Content**:
- Add a **"Two modes (rich / lean)"** section near the top: override flag `ai-dev-workflow.mode`, per-capability rich/lean, pointer to `references/modes.md`.
- Update the "Do not" / conventions: note push = explicit confirm (pointer `references/commit-push.md`); docs go to the standard layout (pointer `references/doc-structure.md`).
- Note the three plugins are optional.

- [ ] Step 1: Edit AGENTS.md (add Two-modes section + pointers; note optional plugins + push-confirm).
- [ ] Step 2: Verify `grep -c 'ai-dev-workflow.mode' AGENTS.md` ≥1; pointers to the 4 references present.
- [ ] Step 3: Commit: `git add AGENTS.md && git commit -m "docs: AGENTS.md — 2-mode + override + pointers to references/"`

### Task 7: Final verification

- [ ] Step 1: `ls plugins/ai-dev-workflow/references/` → modes.md, skill-structure.md, commit-push.md, doc-structure.md all present.
- [ ] Step 2: Cross-ref consistency: each reference + README + AGENTS mutually point correctly (`grep -rn 'references/' plugins/ai-dev-workflow/references/*.md README.md AGENTS.md`).
- [ ] Step 3: Markdown sanity (code fences balanced in each new file).
- [ ] Step 4: `git log --oneline <base>..HEAD` → 6 commits (4 references + README + AGENTS). No skill `SKILL.md` touched.
- [ ] Step 5: Push is **not** done here — await user confirmation (per commit-push discipline).

---

## Self-Review

**Spec coverage:** modes.md (§File1)→T1; skill-structure.md (§File2)→T2; commit-push.md (§File3)→T3; doc-structure.md (§File5)→T4; README/AGENTS (§File4)→T5/T6. All acceptance criteria from the spec mapped. No skill restructured (non-goal honored). ✓

**Placeholder scan:** none — each task carries its content outline + verification; no TBD/TODO.

**Consistency:** the 4 reference names are identical across spec, plan, README/AGENTS pointers. `ai-dev-workflow.mode` flag string consistent everywhere.

## Execution Handoff

Plan saved to `docs/plans/2026-07-22-sp1-skill-foundation.md`. Two options:
1. **Subagent-Driven** — fresh subagent per task, review between.
2. **Inline** — execute in this session via executing-plans.
