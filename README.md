# gabinety-tech

A Polish marketplace where a client with a software problem books **one fixed-price, paid 30 or 60 minute consultation** with a
verified programmer (a "gabinet"), and leaves a review that only a completed consultation allows. Programmers get a free public
profile, a calendar and payment handling, and pay a fee only on the **first** consultation with a new client. Ranking follows
published rules and cannot be bought.

The model is borrowed from ZnanyLekarz (marketplace, trust mechanisms, success-based fee), not its brand or content. The business
analysis is in [`docs/research/BUSINESS-ANALYSIS.md`](docs/research/BUSINESS-ANALYSIS.md) (Polish). The working name
`znany-programista` was dropped because it sits too close to that mark.

## Status

**Specification only. No application code exists yet.** The specification was drafted on 2026-11-12 and its owner decisions are
still marked *(proposed)* in [`DECISIONS.md`](docs/spec/DECISIONS.md) until the owner confirms them (task T-00.01). Nothing described
below has been built or run; the two scripts named under "Checks" are the only commands this repository has.

## How the work is organised

The repository is built from [`spec-driven-template`](https://github.com/konradcinkusz/spec-driven-template) and the standards in
[`architecture-standards`](https://github.com/konradcinkusz/architecture-standards) (.NET Aspire, one service and one database per
bounded context, Fly.io). A **director session** reads the specification, asks the owner the open decisions once, and delegates one
task at a time to **Claude Haiku 5.5** implementer sessions, one pull request per task. The text to start it is
[`docs/spec/DIRECTOR-PROMPT.md`](docs/spec/DIRECTOR-PROMPT.md).

| Document | What it is |
|---|---|
| [`docs/spec/DECISIONS.md`](docs/spec/DECISIONS.md) | 33 owner decisions: product, platforms, CI, delegation, evidence, and the product parameters (fee, windows, niche) |
| [`docs/spec/SPEC.md`](docs/spec/SPEC.md) | The system: goals and non-goals, topology, cross-cutting rules, services, timeouts, test doubles, CI, security, assumptions, release gates |
| [`docs/spec/CONTRACTS.md`](docs/spec/CONTRACTS.md) | Data models, HTTP contracts, state machines, the fee, refund, ranking and slot formulas with test vectors |
| [`docs/spec/COPILOT-SDK.md`](docs/spec/COPILOT-SDK.md) | `Gabinety.Llm`: how to use the GitHub Copilot SDK properly, ported from [`llm-multi-agent-chess`](https://github.com/konradcinkusz/llm-multi-agent-chess) |
| [`docs/spec/TASKS.md`](docs/spec/TASKS.md) | 73 tasks in 11 phases (0 to 10), each with files, steps, acceptance proof and a verify command, and the waves that may run in parallel |
| [`docs/spec/SUPERVISOR.md`](docs/spec/SUPERVISOR.md) | The delegation protocol, with the rules for Haiku implementers and for starting and resuming sessions |
| [`docs/spec/PROGRESS.md`](docs/spec/PROGRESS.md) | Measured state. A "merged" row has a link, a gate has its proof |
| [`docs/adr/`](docs/adr/) | Eight accepted decisions and the three deviations from the standards |
| [`docs/spikes/`](docs/spikes/), [`docs/operations/`](docs/operations/) | Empty until the tasks that fill them run |

## The system in one picture

```
Browser -> web (Next.js + BFF) -> Directory   (profiles, offerings, taxonomy, verification, search, ranking)
                               -> Booking     (availability, bookings, payments, ledger, payouts, notifications)
                               -> Reputation  (reviews, moderation, rating summaries)
                               -> Assistant   (optional AI: brief assistant, review triage; Gabinety.Llm -> Copilot SDK / BYOK)
                               -> authservice (identity: tokens, JWKS)       one PostgreSQL database per service
```

The AI is optional and advisory. Public traffic never runs on the owner's Copilot subscription, only on bring-your-own-key models
([ADR-0004](docs/adr/0004-llm-in-its-own-service-copilot-sdk-and-byok.md)).

## Checks

```bash
scripts/check-spec.sh
```

```bash
scripts/check-open-markers.sh
```

They keep the specification consistent: every task has its eight fields and a unique id, every "merged" row in `PROGRESS.md` has a
link, and once `DECISIONS.md` says `Status: decided` no placeholder is left. They cannot tell whether a claim is true, only that it
carries a link and a date.

## Scope

This repository holds the specification and, once Phase 0 has run, the code (ADR-0006). Real personal data never enters it; fixtures
are synthetic. See [`docs/spec/LESSONS.md`](docs/spec/LESSONS.md) for why the template is shaped as it is.
