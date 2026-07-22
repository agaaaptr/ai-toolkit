# Documentation

Everything that isn't the runnable skills: the system's architecture, the decisions behind it, the validated designs, the execution plans, and the validation findings.

## Structure

| Path | Purpose |
|---|---|
| [`architecture/`](architecture/) | How the marketplace, plugins, and skills fit together. Start here. |
| [`decisions/`](decisions/) | Architecture Decision Records (ADRs) — *why* a choice was made. |
| [`specs/`](specs/) | Design specs — validated design before implementation. |
| [`plans/`](plans/) | Implementation plans — task-by-task execution plans. |
| [`findings/`](findings/) | Validation findings — PoC results, incident notes, retrospectives. |

## Conventions

- **Specs** → `specs/` and **plans** → `plans/`, both dated `YYYY-MM-DD-<topic>.md`. This overrides the Superpowers default of `docs/superpowers/` (see [ADR-0001](decisions/0001-doc-layout-and-spec-plan-paths.md) and [`AGENTS.md`](../AGENTS.md)).
- **ADRs** are numbered `NNNN-<slug>.md` and append-only — supersede, don't rewrite.
- Cross-link liberally: a spec should link to its plan and any related ADR.
