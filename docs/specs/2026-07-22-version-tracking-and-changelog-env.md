# Spec Addendum: Version Tracking + CHANGELOG Env Note

- **Date:** 2026-07-22
- **Status:** Accepted
- **Related:** [2026-07-22-flow-fe-versioning-design.md](2026-07-22-flow-fe-versioning-design.md) (the base plain-version reform)
- **Repos:** partnership `lib-uii-gateway-partnership-angular`; ai-toolkit (`/flow`, `/sync`)

## Context

After deploying the plain-version reform across develop/staging/master, six edge-case questions were raised about the suffix/CI logic. Analysis verdict:

- **Solid (no action):** accidental suffix in `package.json` cannot double (`npm version "$CI_COMMIT_TAG"` sets, not appends — publish is always tag-driven); counter resets on any version digit change (patch/minor/major); no hard limit on N.
- **Gap — version tracking:** no in-repo view of what's live per branch; devs must check verdaccio repeatedly.
- **Gap — CHANGELOG informativeness:** plain entries don't show which env suffix was published.
- **Caveats (noted, low practical impact):** counter relies on the b-on-develop/rc-on-staging tagging convention; `-bN` format sorts lexically in semver (`b10 < b9`), which only matters for range queries (wrappers pin exact versions).

## Decisions

| # | Decision |
|---|---|
| Q2 | CHANGELOG = plain heading + a `Published as: <ver>-<suffix> (<branch>)` line (informative, no cross-branch edit → no conflict). |
| Q3 source | Tracking data from the npm registry (`npm view … versions --json`) — authoritative (what actually published). |
| Q3 updater | Generated on-demand by a script; `VERSIONS.md` is **committed** (repo info), regenerated each run. |
| Q3 integration | Also surfaced in `/sync` (Angular lib detected → latest-per-env in the brief). |

## Design

### A — CHANGELOG `Published as:` note (`/flow` Phase 7)
- The Phase 7 version-bump procedure is **reordered**: compute the tag (branch + counter N) **before** writing the CHANGELOG, so the suffix is known.
- CHANGELOG entry: `## <today> (<plain version>)` + bullets + `- Published as: <tag> (<branch>)`.
- No cross-branch CHANGELOG edits (the multi-env picture lives in `VERSIONS.md`).

### B — On-demand version tracking (partnership repo)
- `scripts/published-versions.mjs` + `npm run versions`.
- Queries `npm view @uiigateway/partnership versions --json`, classifies by suffix convention (`-bN`→develop, `-rcN`→staging, plain→master), sorts **numerically** (handles the `b10 > b9` lexical caveat for display), prints a table **and writes `VERSIONS.md`**.
- `VERSIONS.md` is committed (repo info); regenerated (overwritten) on each run; header records "last generated" date + "run `npm run versions` to refresh". Conflicts (derived snapshot) resolve by re-running the script.

### B-integration — `/sync` (ai-toolkit)
- When `/sync` detects an Angular v13 `@uiigateway/*` library, it runs `npm view <pkg> versions --json` and includes a latest-per-env line in the session brief (read-only).

## Edge cases addressed
- Q1 (double suffix): N/A — CI sets version from tag verbatim; no double possible.
- Q6 (lexical sort): the tracking script sorts numerically, so display is correct despite npm's lexical prerelease ordering.
- Q4 (per-branch counter): unchanged — suffix is tag-only, each branch tags independently.
- Q5 (merge conflicts): unchanged — plain version + `.gitattributes union` already deployed; this addendum adds no conflict surface.

## Files affected
**Partnership (`develop`):** `scripts/published-versions.mjs` (new), root `package.json` (`versions` script), `VERSIONS.md` (new, committed), `STANDARDS.md` + `DEVELOPMENT_WORKFLOW_GUIDE.md` (how to check versions).
**ai-toolkit (`main`):** `plugins/ai-dev-workflow/skills/flow/SKILL.md` (Phase 7 reorder + note), `plugins/ai-dev-workflow/skills/sync/SKILL.md` (published-versions in brief).

## Execution order
1. Partnership: script + npm script → run → `VERSIONS.md` → docs → push `develop`.
2. ai-toolkit: `/flow` Phase 7 reorder+note; `/sync` published-versions; commit (push pending user GO — `main` already has unpushed commits).
