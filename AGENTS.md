# AGENTS.md

Gabinety Tech — a Polish marketplace where clients book a fixed-price paid consultation with a verified programmer and leave a review that only a completed consultation allows (DECISIONS D-02).

- Specification: [`docs/spec/SPEC.md`](docs/spec/SPEC.md), contracts: [`docs/spec/CONTRACTS.md`](docs/spec/CONTRACTS.md), LLM gateway: [`docs/spec/COPILOT-SDK.md`](docs/spec/COPILOT-SDK.md). Tasks: [`docs/spec/TASKS.md`](docs/spec/TASKS.md). Protocol: [`docs/spec/SUPERVISOR.md`](docs/spec/SUPERVISOR.md). Owner decisions: [`docs/spec/DECISIONS.md`](docs/spec/DECISIONS.md). Measured progress: [`docs/spec/PROGRESS.md`](docs/spec/PROGRESS.md).
- Standards: [`konradcinkusz/architecture-standards`](https://github.com/konradcinkusz/architecture-standards) (P1–P15). Declared in `.claude/settings.json`.
- Do not start implementation while `docs/spec/DECISIONS.md` says `Status: open`: ask the owner the open rows first.
- Fixtures are synthetic, in the namespace `demo-`. No secrets in any file. No user free text (brief, review, bio) in a log, trace or metric.
- The Copilot SDK is referenced only by `src/Gabinety.Llm`. A language model advises and never decides money, ranking, ratings, verification or removal of content.
- Money is `long` grosz. Time is UTC, read from `TimeProvider`. Constants of `CONTRACTS.md` §9 are the owner's: never change one inside a task.
- Report what was measured as measured and what was concluded as a conclusion.
- English for code, comments and docs; Polish only for user-facing text, in `web/src/i18n/pl.ts` and the notification templates.
- Verify: `scripts/check-spec.sh && scripts/check-open-markers.sh`; once code exists also `dotnet build -warnaserror && dotnet test --filter "tier!=integration"` and, for web changes, `pnpm --dir web lint && pnpm --dir web typecheck && pnpm --dir web test && pnpm --dir web build`.
