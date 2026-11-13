# ADR-0006: the specification and the code live in one repository

- Status: accepted
- Date: 2026-11-12

## Context
The spec-driven template puts the specification, the tasks and the code in the same repository, and the checks
(`check-spec.sh`, `check-open-markers.sh`) assume that. The repository is named `gabinety-tech-spec`.

## Decision
Code is added to this repository by Phase 0 task T-00.03. The solution is named `GabinetyTech` whatever the repository is called.
Renaming the repository to `gabinety-tech` is the owner's option and changes no path in the documents.

## Consequences
- A pull request can change a contract and its specification together (SPEC amendments land in the same PR).
- The director session needs access to one repository only.

## Alternatives considered
- A separate code repository: rejected, two repositories need two sets of settings, two access grants and a sync rule.
