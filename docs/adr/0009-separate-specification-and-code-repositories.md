# ADR-0009: specification and code live in two repositories

- Status: accepted
- Date: 2026-11-13
- Supersedes: ADR-0006

## Context
The owner instructed that the specification is managed outside the code, in `gabinety-tech-spec`; code goes to `gabinety-tech`.

## Decision
`gabinety-tech-spec` holds `docs/`, `scripts/check-spec.sh`, `scripts/check-open-markers.sh` and `PROGRESS.md`. `gabinety-tech` holds every
path of SPEC §3.2 except `docs/spec`, `docs/adr` and `docs/research`. Task `Files:` paths are relative to the repository named in the task
(code paths: `gabinety-tech`; `docs/spikes/**`, `docs/operations/**`: `gabinety-tech`; `docs/spec/**`, `docs/adr/**`: spec repository).
Implementer sessions get `gabinety-tech` as their source and a read-only copy of the task block in the brief.

## Consequences
- A contract change is two pull requests: the spec PR first, then the code PR that needs it (SUPERVISOR §4 already requires that order).
- The code repository carries a copy of `AGENTS.md` and a `spec-check` job that clones the spec repository at a pinned commit.
- Spikes are written in `gabinety-tech` (`docs/spikes/`) and linked from the spec.
