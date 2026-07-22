# ADR-0001: Documentation layout and spec/plan paths

- **Status:** Accepted
- **Date:** 2026-07-22

## Context

The repo accumulated flat docs (`docs/<file>.md`) alongside Superpowers-generated `docs/superpowers/{specs,plans}/`. The result was hard to track and inconsistent with how professional AI-skill repositories organize their documentation.

## Decision

Adopt a typed `docs/` layout: `architecture/`, `decisions/` (ADRs), `findings/`, `specs/`, `plans/`, with a `docs/README.md` index. Design specs and implementation plans are written to `docs/specs/` and `docs/plans/` (dated `YYYY-MM-DD-<topic>.md`), **overriding** the Superpowers default of `docs/superpowers/`. Local session scratch (`.superpowers/`) is gitignored. Add a root `AGENTS.md` for agent-guided development of the repo itself.

## Consequences

- One predictable home per document type; easier to find and maintain.
- Brainstorming/writing-plans sessions must be told (via `AGENTS.md`) to use `docs/specs` / `docs/plans`. Both skills accept a user-preferred location, so this is supported.
- Future repo-level decisions are recorded in `decisions/` as numbered, append-only ADRs.
