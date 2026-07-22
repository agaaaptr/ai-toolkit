# Flow FE Versioning + Conflict-Free Promotion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eliminate `package.json`/`CHANGELOG.md` merge conflicts on env promotion for the `@uiigateway/partnership` Angular lib by moving the version suffix out of committed files and into git tags (CI injects it at publish), and teach `/flow` to commit per-scope and to bump+tag the lib automatically.

**Architecture:** Two-repo change. (A) Partnership repo `lib-uii-gateway-partnership-angular` on `develop`: CI derives `version` from `$CI_COMMIT_TAG`, `package.json` goes plain, `CHANGELOG.md` gets a `merge=union` gitattribute, and `STANDARDS.md`/`DEVELOPMENT_WORKFLOW_GUIDE.md` are updated to match. (B) ai-toolkit repo: `/flow` skill gains a Phase-5 commit-per-scope rule and a Phase-7 conditional Angular-v13 version-bump that bumps the plain version, updates CHANGELOG, commits, and creates+pushes the env-suffixed tag.

**Tech Stack:** Angular 13.3 library, GitLab CI (`node:14.15.5`), private npm registry `npm.uii.ac.id`, Claude Code plugin skills (Markdown).

## Global Constraints

- Partnership changes land on branch `develop` only; **`git pull origin develop` before any commit**.
- Commit **per scope** (conventional-commit type): never bundle unrelated changes.
- `package.json` (lib) version is **plain** (`X.Y.Z`), never carries `-bN`/`-rcN`. Suffix lives only in the git tag.
- CI publish sets version from the tag verbatim: `npm version --no-git-tag-version --allow-same-version "$CI_COMMIT_TAG"` — `--allow-same-version` is required so master (plain tag == plain version) does not error.
- Wrapper repos are downstream (read published npm only) — not touched.
- Improve of skills other than `/flow` is **out of scope** (separate task).
- Commit message standard: `chore:` (not historical `chores:`/`bumpe`).

---

## File Structure

**Partnership repo** (`/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular`):
- Modify: `.gitlab-ci.yml` — `publish` job gains a `npm version` step.
- Modify: `projects/uiigateway/partnership/package.json` — `version` → plain.
- Create: `.gitattributes` — `CHANGELOG.md merge=union`.
- Modify: `STANDARDS.md` — plain-version model + CI derive + CHANGELOG union.
- Modify: `DEVELOPMENT_WORKFLOW_GUIDE.md` — same deltas.

**ai-toolkit repo** (`/Users/agaaaptr/Documents/Personal/Project/AI/ai-toolkit`):
- Already created (uncommitted): `docs/superpowers/specs/2026-07-22-flow-fe-versioning-design.md`.
- Modify: `plugins/ai-dev-workflow/skills/flow/SKILL.md` — Phase 5 (line 40), Phase 7 (line 44), Hard rules (lines 46-50).

---

# Part A — Partnership repo (branch `develop`)

### Task A1: Prep — sync `develop` and locate docs

**Files:** none (read-only prep).

- [ ] **Step 1: Checkout + pull develop**
```bash
git -C "/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular" checkout develop
git -C "/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular" pull origin develop
```
Expected: `Already up to date.` or fast-forward.

- [ ] **Step 2: Confirm clean tree**
```bash
git -C "/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular" status --porcelain
```
Expected: empty output. If not empty, STOP and surface to user.

- [ ] **Step 3: Locate the two doc files + confirm current CI/version**
```bash
P="/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular"
ls "$P/STANDARDS.md" "$P/DEVELOPMENT_WORKFLOW_GUIDE.md"
grep -n 'npm publish' "$P/.gitlab-ci.yml"
grep -n '"version"' "$P/projects/uiigateway/partnership/package.json"
```
Expected: both doc paths print; CI grep shows the `npm publish --registry https://npm.uii.ac.id` line; package.json shows `"version": "2.0.23-b0"`.

No commit.

---

### Task A2: CI — derive version from tag at publish

**Files:** Modify `.gitlab-ci.yml` (the `publish` job `script` block).

- [ ] **Step 1: Insert the `npm version` step**

In the `publish` job, find this exact block:
```yaml
    - cd dist/uiigateway/partnership
    - echo '//npm.uii.ac.id/:_authToken=${NPM_AUTH_TOKEN}'>.npmrc
    - npm publish --registry https://npm.uii.ac.id
```
Replace with:
```yaml
    - cd dist/uiigateway/partnership
    - npm version --no-git-tag-version --allow-same-version "$CI_COMMIT_TAG"
    - echo '//npm.uii.ac.id/:_authToken=${NPM_AUTH_TOKEN}'>.npmrc
    - npm publish --registry https://npm.uii.ac.id
```

- [ ] **Step 2: Verify the edit**
```bash
P="/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular"
grep -n -A2 'cd dist/uiigateway/partnership' "$P/.gitlab-ci.yml"
```
Expected: the `cd`, then `npm version --no-git-tag-version --allow-same-version "$CI_COMMIT_TAG"`, then the `echo …npmrc` line.

- [ ] **Step 3: YAML lint (if a linter is available)**
```bash
P="/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular"
python3 -c "import yaml,sys; yaml.safe_load(open('$P/.gitlab-ci.yml')); print('yaml ok')" 2>&1 || echo "pyyaml unavailable — visual check only"
```
Expected: `yaml ok` (or the graceful fallback message).

- [ ] **Step 4: Commit**
```bash
P="/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular"
git -C "$P" add .gitlab-ci.yml
git -C "$P" commit -m "ci: derive lib version from git tag at publish"
```

---

### Task A3: `package.json` — plain version (strip suffix)

**Files:** Modify `projects/uiigateway/partnership/package.json`.

- [ ] **Step 1: Set version to plain**

Change:
```json
  "version": "2.0.23-b0",
```
to:
```json
  "version": "2.0.23",
```

- [ ] **Step 2: Verify**
```bash
P="/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular"
grep -n '"version"' "$P/projects/uiigateway/partnership/package.json"
```
Expected: `"version": "2.0.23",` (no suffix).

- [ ] **Step 3: Commit**
```bash
P="/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular"
git -C "$P" add projects/uiigateway/partnership/package.json
git -C "$P" commit -m "chore: set lib package.json to plain version (suffix injected via tag/CI)"
```

---

### Task A4: `.gitattributes` — union-merge CHANGELOG

**Files:** Create `.gitattributes`.

- [ ] **Step 1: Create the file**

Write `.gitattributes` (repo root) with exactly:
```
CHANGELOG.md merge=union
```

- [ ] **Step 2: Verify the merge attribute is effective**
```bash
P="/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular"
git -C "$P" add .gitattributes
git -C "$P" check-attr merge -- CHANGELOG.md
```
Expected: `CHANGELOG.md: merge: union`.

- [ ] **Step 3: Commit**
```bash
P="/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular"
git -C "$P" add .gitattributes
git -C "$P" commit -m "chore: union-merge CHANGELOG to avoid promotion conflicts"
```

---

### Task A5: Update `STANDARDS.md`

**Files:** Modify `STANDARDS.md`. Apply these deltas (grep to locate exact current text; adjust whitespace to match).

- [ ] **Step 1: §7 CI/CD — add derive-from-tag note**

After the "Tag Format for Publish" subsection (the `✅ 2.1.0-rc0` / `❌ v2.0.10` block), insert:

```markdown
### Version injection at publish

`npm publish` reads `version` from `package.json`. To keep `package.json` **plain** (no `-bN`/`-rcN`) across branches and avoid promotion conflicts, the `publish` job sets the version from the tag right before publishing:

\`\`\`yaml
- cd dist/uiigateway/partnership
- npm version --no-git-tag-version --allow-same-version "$CI_COMMIT_TAG"
- npm publish --registry https://npm.uii.ac.id
\`\`\`

`$CI_COMMIT_TAG` already carries the suffix (e.g. `2.0.26-b1`), so the published artifact is correct while the committed `package.json` stays suffix-free. `--allow-same-version` prevents an error on master, where the plain tag equals the plain version.
```

- [ ] **Step 2: §8.2 — clarify the source of the suffix**

Find the comment `# Edit version in package.json` / `"version": "2.1.1",  # Update this` block. After it, add:

```markdown
> The `version` in `package.json` is always **plain** (`X.Y.Z`). The env suffix (`-bN`/`-rcN`) is NOT committed — it lives only in the git tag and is injected by CI at publish (see §7). This is what prevents version-line conflicts on promotion.
```

- [ ] **Step 3: §9.3 step 2 — fix the confusing comment**

Find:
```markdown
# Edit package.json: ubah version dari "2.1.5" → "2.1.5" (tetap)
# (suffix sudah di-setup saat tagging)
```
Replace with:
```markdown
# package.json version stays PLAIN ("2.1.5") on every branch.
# The -rcN suffix is NOT written to package.json; it lives in the tag (2.1.5-rc0)
# and is injected by CI at publish. No version-line edit needed for promotion.
```

- [ ] **Step 4: §8.3 CHANGELOG — plain, one entry per version**

In the CHANGELOG format subsection, add:

```markdown
> CHANGELOG entries use the **plain** version (`## DD Month YYYY (X.Y.Z)`), one entry per version. Do NOT create separate `-b0`/`-rc0` entries per env — the entry is written once (typically on `develop` at bump time) and carries through `staging → master`. A `.gitattributes` rule (`CHANGELOG.md merge=union`) auto-resolves any residual top-of-file conflict during promotion.
```

- [ ] **Step 5: Verify**
```bash
P="/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular"
grep -n 'Version injection at publish' "$P/STANDARDS.md"
grep -n 'always \*\*plain\*\*' "$P/STANDARDS.md"
grep -n 'merge=union' "$P/STANDARDS.md"
```
Expected: each grep prints at least one matching line.

- [ ] **Step 6: Commit**
```bash
P="/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular"
git -C "$P" add STANDARDS.md
git -C "$P" commit -m "docs(STANDARDS): plain-version model, tag-suffix, CI derive, CHANGELOG union"
```

---

### Task A6: Update `DEVELOPMENT_WORKFLOW_GUIDE.md`

**Files:** Modify `DEVELOPMENT_WORKFLOW_GUIDE.md`. Mirror the A5 deltas into this doc's equivalent sections.

- [ ] **Step 1: CI section — add the same "Version injection at publish" block** (the yaml snippet + explanation from A5 Step 1). Locate the publish/CI section via:
```bash
P="/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular"
grep -n 'npm publish' "$P/DEVELOPMENT_WORKFLOW_GUIDE.md"
```

- [ ] **Step 2: Version Control Standards — add the "package.json is plain, suffix in tag" note** (A5 Step 2 text).

- [ ] **Step 3: Practical Scenarios (Scenario 3 / release-to-production) — fix any "edit package.json suffix" wording** to state the version stays plain and the suffix is tag-only / CI-injected (A5 Step 3 text).

- [ ] **Step 4: CHANGELOG guidance — add the plain, one-entry-per-version note** (A5 Step 4 text).

- [ ] **Step 5: Verify**
```bash
P="/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular"
grep -cn 'plain' "$P/DEVELOPMENT_WORKFLOW_GUIDE.md"
grep -n 'CI_COMMIT_TAG' "$P/DEVELOPMENT_WORKFLOW_GUIDE.md"
```
Expected: grep counts > 0; `CI_COMMIT_TAG` appears in the new CI block.

- [ ] **Step 6: Commit**
```bash
P="/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular"
git -C "$P" add DEVELOPMENT_WORKFLOW_GUIDE.md
git -C "$P" commit -m "docs(GUIDE): plain-version model, tag-suffix, CI derive, CHANGELOG union"
```

---

### Task A7: Push `develop` + watch the pipeline

**Files:** none.

- [ ] **Step 1: Push**
```bash
P="/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular"
git -C "$P" log --oneline origin/develop..develop
git -C "$P" push origin develop
```
Expected: push succeeds; `git log` beforehand shows the 5 new commits (ci, chore, chore, docs, docs).

- [ ] **Step 2: Note pipeline impact**

Pushing commits to `develop` triggers the `analysis`/`compile`/`build`/`deploy` jobs (branch pipelines) — these do NOT publish (publish is tag-only). The derive-from-tag change only takes effect on the next tag push. Surface this to the user; do NOT create a tag here.

---

# Part B — ai-toolkit repo

### Task B1: Branch + commit the spec

**Files:** already-created `docs/superpowers/specs/2026-07-22-flow-fe-versioning-design.md` (and this plan).

- [ ] **Step 1: Create a feature branch (currently on `main`)**
```bash
cd "/Users/agaaaptr/Documents/Personal/Project/AI/ai-toolkit"
git checkout -b feat/flow-fe-versioning
```

- [ ] **Step 2: Commit spec + plan**
```bash
git add docs/superpowers/specs/2026-07-22-flow-fe-versioning-design.md docs/superpowers/plans/2026-07-22-flow-fe-versioning.md
git commit -m "docs(ai-dev-workflow): flow FE versioning design spec + plan"
```

---

### Task B2: `/flow` Phase 5 — commit-per-scope rule + Hard rule

**Files:** Modify `plugins/ai-dev-workflow/skills/flow/SKILL.md` (Phase 5 at line 40; Hard rules at lines 46-50).

- [ ] **Step 1: Rewrite the Phase 5 paragraph**

Replace:
```markdown
**Phase 5 — Execute.** Invoke `superpowers:executing-plans` (TDD, checkpoint review). Update the state file after each plan task. **If any new doubt or non-conventional behavior surfaces, STOP and return to the phase-3 Confirm gate** — do not guess. Gate: per-checkpoint review.
```
with:
```markdown
**Phase 5 — Execute.** Before delegating, state this **binding rule** to the execution: *commit per scope — group changes by conventional-commit type/scope (`feat`/`fix`/`refactor`/`docs`/`test`/`chore`), commit each scope separately; never bundle unrelated changes into one commit; combine only changes that form one logical unit.* Then invoke `superpowers:executing-plans` (TDD, checkpoint review). Update the state file after each plan task. **If any new doubt or non-conventional behavior surfaces, STOP and return to the phase-3 Confirm gate** — do not guess. Gate: per-checkpoint review.
```

- [ ] **Step 2: Add a Hard rule**

In the `## Hard rules` list, after `- On doubt at any phase → back to Confirm.`, add:
```markdown
- Commit per scope: never bundle unrelated changes into one commit — group by conventional-commit type/scope.
```

- [ ] **Step 3: Verify**
```bash
grep -n 'commit per scope' "plugins/ai-dev-workflow/skills/flow/SKILL.md"
```
Expected: at least two matches (Phase 5 paragraph + Hard rule).

- [ ] **Step 4: Commit**
```bash
git add plugins/ai-dev-workflow/skills/flow/SKILL.md
git commit -m "feat(ai-dev-workflow): /flow Phase 5 commit-per-scope rule"
```

---

### Task B3: `/flow` Phase 7 — conditional Angular v13 version-bump + tag

**Files:** Modify `plugins/ai-dev-workflow/skills/flow/SKILL.md` (Phase 7 at line 44).

- [ ] **Step 1: Insert the conditional release step**

In the Phase 7 paragraph, find:
```markdown
list durable-doc candidates to promote and ephemeral scratch to delete, and **confirm with the user before moving/deleting anything**. Set the state file `status: done`. Gate: docs reviewed.
```
Replace with:
```markdown
list durable-doc candidates to promote and ephemeral scratch to delete, and **confirm with the user before moving/deleting anything**.

**Angular FE v13 lib release (conditional).** Detect whether this project is an Angular FE v13 `@uiigateway/*` publishable library: (1) root `package.json` has `@angular/core` `~13.x`/`^13.x`, AND (2) a `projects/<scope>/<lib>/package.json` with a `version` field exists, AND (3) its `name` matches `@uiigateway/*`. If all three hold, use `AskUserQuestion` to offer a version action (`patch`/`minor`/`major`/`next-pre-release`/`skip`; show the current plain version): `patch`/`minor`/`major` → new plain version (suffix resets to 0); `next-pre-release` → same plain version, suffix +1 (`b0→b1`, `rc0→rc1`). On `skip` → do nothing. Otherwise:
1. **(patch/minor/major)** Edit the lib `package.json` to the new **plain** version; prepend a `## <today> (<new plain version>)` entry to `CHANGELOG.md` (plain, match the latest entry's style, summarize the session's changes); commit both: `chore: bump version to <new version>`. **(next-pre-release)** leave `package.json` unchanged; optionally append a bullet to the existing version's CHANGELOG entry (confirm with user).
2. Branch: `git rev-parse --abbrev-ref HEAD`. Suffix kind: `develop`→`b`, `staging`→`rc`, `master`→none (plain), other→`AskUserQuestion`.
3. **Counter N (deterministic):** list `git tag -l "<target>-<kind>*"` for the target version + suffix kind; parse the integer after `<kind>` from each match; `N = max + 1`, or `0` if none. Compare numerically (`b10` > `b9`). A new plain version has no matching tags → N resets to 0, so `2.1.0-b0 → 2.1.0-b1 → 2.1.1-b0` works with no special case.
4. Tag: `develop`→`<target>-b<N>`; `staging`→`<target>-rc<N>`; `master`→`<target>`.
5. `git push origin <branch>` (if new commits); then `git tag -a "<tag>" -m "release: <target>"` and `git push origin "<tag>"`.
If the project is not an Angular v13 `@uiigateway/*` lib, skip this step entirely.

Set the state file `status: done`. Gate: docs reviewed.
```

- [ ] **Step 2: Verify the edit**
```bash
grep -n 'Angular FE v13 lib release' "plugins/ai-dev-workflow/skills/flow/SKILL.md"
grep -n 'git tag -l' "plugins/ai-dev-workflow/skills/flow/SKILL.md"
```
Expected: both greps match.

- [ ] **Step 3: Verify the counter algorithm (next-pre-release increments; version-bump resets; numeric sort)**

Encodes the exact `nextN` rule the skill text must follow — reset on version change is automatic (a new version has no matching tags), no special-case needed. Expected output: `2` / `0` / `1` / `11`.
```bash
node -e '
function nextN(ver, kind, tags){
  const prefix = ver + "-" + kind;
  const ns = tags.filter(t => t.startsWith(prefix)).map(t => Number(t.slice(prefix.length))).filter(n => !isNaN(n));
  return ns.length ? Math.max(...ns) + 1 : 0;
}
console.log(nextN("2.1.0","b",["2.1.0-b0","2.1.0-b1"])); // 2
console.log(nextN("2.1.1","b",["2.1.0-b0","2.1.0-b1"])); // 0  (patch bump resets to b0)
console.log(nextN("2.1.0","rc",["2.1.0-rc0"]));          // 1
console.log(nextN("2.1.0","b",["2.1.0-b9","2.1.0-b10"]));// 11 (numeric, not lexical)
'
```
If any line differs, the skill text's counter rule is wrong — fix before committing.

- [ ] **Step 4: Dry-run the detection predicate against the partnership repo**

Run from the partnership repo root; expect `DETECT: yes`:
```bash
P="/Users/agaaaptr/Documents/Work/BSI/Project/Front-end/BPM - Army/UIIKemitraan/lib-uii-gateway-partnership-angular"
node -e '
const fs=require("fs");
const root=JSON.parse(fs.readFileSync(process.argv[1]+"/package.json","utf8"));
const ang=(root.dependencies&&root.dependencies["@angular/core"])||(root.devDependencies&&root.devDependencies["@angular/core"]);
const v13=!!(ang&&/^[\^~]?13\./.test(ang));
const libPkg=process.argv[1]+"/projects/uiigateway/partnership/package.json";
const hasLib=fs.existsSync(libPkg);
let nameOk=false;
if(hasLib){const lib=JSON.parse(fs.readFileSync(libPkg,"utf8"));nameOk=/^@uiigateway\//.test(lib.name||"");}
console.log("DETECT:",(v13&&hasLib&&nameOk)?"yes":"no",{v13,hasLib,nameOk,ang});
' "$P"
```
Expected: `DETECT: yes { v13: true, hasLib: true, nameOk: true, ang: '~13.3.x' }`.

- [ ] **Step 5: Commit**
```bash
git add plugins/ai-dev-workflow/skills/flow/SKILL.md
git commit -m "feat(ai-dev-workflow): /flow Phase 7 Angular v13 version-bump + env-tag"
```

---

### Task B4: Final verification (ai-toolkit)

**Files:** none (verification).

- [ ] **Step 1: Markdown lint the edited skill**
```bash
npx --yes markdownlint-cli "plugins/ai-dev-workflow/skills/flow/SKILL.md" 2>&1 || echo "markdownlint unavailable — visual check only"
```
Expected: no errors (or graceful-fallback message). Fix MD022/MD032 (blank lines around headings/lists) if flagged.

- [ ] **Step 2: Re-read both edited regions** to confirm coherence (Phase 5, Phase 7, Hard rules).

- [ ] **Step 3: Commits on this branch**
```bash
git log --oneline main..HEAD
```
Expected: spec+plan commit, Phase 5 commit, Phase 7 commit.

- [ ] **Step 4 (optional, manual integration):** install the plugin and run `/flow <task>` in the partnership repo to confirm the version-bump offer + tag appear end-to-end. Skip if time-boxed; the predicate dry-run (B3 Step 3) covers detection.

---

### Task B5: Push / PR (confirm with user)

**Files:** none.

- [ ] **Step 1: Surface to user before pushing** (ai-toolkit push was not explicitly authorized; partnership push was). Offer: push branch, or open PR.
```bash
git push -u origin feat/flow-fe-versioning   # only after user confirms
```

---

## Self-Review (run after writing, before handoff)

**Spec coverage:**
- §3.1 CI derive-from-tag → Task A2. ✓
- §3.2 plain package.json → Task A3. ✓
- §3.2 CHANGELOG plain → Task A5/A6 (doc) + B3 (skill writes plain entries). ✓
- §3.3a Phase 5 commit-per-scope → Task B2. ✓
- §3.3b/§3.4 Phase 7 version-bump + tag procedure → Task B3. ✓
- §3.5 `.gitattributes merge=union` → Task A4. ✓
- §5 doc deltas → Tasks A5/A6. ✓
- §6 execution order (partnership then ai-toolkit) → Part A then Part B. ✓
- Detection = Angular 13 + lib + `@uiigateway/*` → B3 Step 1 + verified B3 Step 3. ✓

**Placeholder scan:** none — all steps carry concrete content/commands.

**Type/name consistency:** tag counter logic (`git tag -l "<ver>-b*"`/`-rc*`) matches between spec §3.4 and plan B3. Commit prefix `chore:` consistent. `--allow-same-version` present in A2 and documented in A5.

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-07-22-flow-fe-versioning.md`. Two execution options:

1. **Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration.
2. **Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints.

Which approach?
