# Progress

Measured state, updated by the supervisor after every merge and at every phase checkpoint (SUPERVISOR §9).

Rules, checked by `scripts/check-spec.sh` where a script can check them:

- Statuses: `not-started`, `in-flight`, `open-red`, `open-green`, `merged`, `assumed`, `blocked:<reason>`.
- A row with status `merged` or `open-green` carries a link to the pull request or the CI run it was read from.
- `assumed` means the work was replaced by an unverified assumption (SPEC §12.1). It is never reported as done.
- "Implemented" is not a status. Code on a branch with unit tests green is `open-green`; it says nothing about a gate.
- Every claim has the date it was measured. A claim nobody re-measured is stale, not true.

| Task | Status | Evidence (PR or run link) | Measured | Notes |
|---|---|---|---|---|
| T-00.01 | not-started | | | Owner decisions recorded |
| T-00.02 | not-started | | | Repository settings |
| T-00.03 | not-started | | | Scaffold and empty pipeline green |
| T-00.04 | not-started | | | First vertical slice against the real thing |
| T-01.01 | not-started | | | Spike: the authservice integration surface |
| T-01.02 | not-started | | | Spike: the Copilot SDK in this solution |
| T-01.03 | not-started | | | Contracts project |
| T-01.04 | not-started | | | DevIdentity and the auth contract suite |
| T-02.01 | not-started | | | Llm: contracts, fake gateway, options and health |
| T-02.02 | not-started | | | Llm: the Copilot gateway |
| T-02.03 | not-started | | | Llm: usage, token prices and bring-your-own-key models |
| T-02.04 | not-started | | | Architecture tests |
| T-02.05 | not-started | | | Live smoke command for the Assistant |
| T-02.06 | not-started | | | Live smoke run of the gateway |
| T-03.01 | not-started | | | Directory: skeleton, persistence, authentication |
| T-03.02 | not-started | | | Directory: practitioner profile |
| T-03.03 | not-started | | | Directory: taxonomy and its seed |
| T-03.04 | not-started | | | Directory: offerings |
| T-03.05 | not-started | | | Directory: verification workflow |
| T-03.06 | not-started | | | Ranking library |
| T-03.07 | not-started | | | Directory: public search, profile and sitemap |
| T-03.08 | not-started | | | Directory: internal endpoints |
| T-04.01 | not-started | | | Billing library |
| T-04.02 | not-started | | | Scheduling library |
| T-04.03 | not-started | | | Booking: skeleton and availability |
| T-04.04 | not-started | | | Booking: slot listing and the no-double-booking constraint |
| T-04.05 | not-started | | | Booking: creation, hold and the payment abstraction |
| T-04.06 | not-started | | | Booking: webhook, simulate and confirmation |
| T-04.07 | not-started | | | Booking: cancellation, dispute, no-show and resolution |
| T-04.08 | not-started | | | Booking: jobs, fee, payout and ledger export |
| T-04.09 | not-started | | | Booking: notifications |
| T-04.10 | not-started | | | Booking: stats push, reviewability, erasure, test clock |
| T-05.01 | not-started | | | Reputation: skeleton, entities and deterministic rules |
| T-05.02 | not-started | | | Reputation: create, list, reply and rating summaries |
| T-05.03 | not-started | | | Reputation: moderation |
| T-05.04 | not-started | | | Reputation: triage, summary push, erasure, test clock |
| T-06.01 | not-started | | | Assistant: skeleton, status, traffic policy, usage and budget |
| T-06.02 | not-started | | | Assistant: the brief feature |
| T-06.03 | not-started | | | Assistant: the review triage feature |
| T-06.04 | not-started | | | Evaluation suite for the two prompts |
| T-06.05 | not-started | | | Live evaluation run |
| T-07.01 | not-started | | | Web: BFF, session and generated types |
| T-07.02 | not-started | | | Web: design tokens, layout, UI primitives, Polish strings |
| T-07.03 | not-started | | | Web: public category and list pages with SEO |
| T-07.04 | not-started | | | Web: public practitioner page |
| T-07.05 | not-started | | | Web: sign-in, registration, account |
| T-07.06 | not-started | | | Web: practitioner area |
| T-07.07 | not-started | | | Web: booking flow, brief assistant, mock checkout |
| T-07.08 | not-started | | | Web: my bookings and the booking actions |
| T-07.09 | not-started | | | Web: reviews |
| T-07.10 | not-started | | | Web: administration |
| T-07.11 | not-started | | | Web: home and static pages |
| T-08.01 | not-started | | | Demo data seeder |
| T-08.02 | not-started | | | AppHost wiring of the seeder and the test clock |
| T-08.03 | not-started | | | E2E: harness and journeys J1, J2 |
| T-08.04 | not-started | | | E2E: J3 and J4 |
| T-08.05 | not-started | | | E2E: J5, J6, J7 |
| T-08.06 | not-started | | | Run guide |
| T-09.01 | not-started | | | Owner accounts and secrets |
| T-09.02 | not-started | | | Dockerfiles, fly.toml, cost and the `docker-build` job |
| T-09.03 | not-started | | | Tag-driven deploy workflow |
| T-09.04 | not-started | | | authservice and PostgreSQL apps |
| T-09.05 | not-started | | | Payment provider adapter |
| T-09.06 | not-started | | | E-mail adapter |
| T-09.07 | not-started | | | Production wiring of the Assistant and the policy guard |
| T-09.08 | not-started | | | First real booking on staging |
| T-10.01 | not-started | | | Security review |
| T-10.02 | not-started | | | Legal texts and compliance review |
| T-10.03 | not-started | | | Abuse controls and suspicious pairs |
| T-10.04 | not-started | | | Performance measurement |
| T-10.05 | not-started | | | Runbook, backups and restore |
| T-10.06 | not-started | | | Ranking changelog |
| T-10.07 | not-started | | | M3 checkpoint |

## Gates

One line per gate of SPEC §14: `not claimed`, `claimed <date>` with the proof link and the commit, or `failed <date>` with the evidence.
Say exactly what the proof covers and what it does not (platform, start command, real dependency or stand-in).

- **M0**: not claimed.
- **M1**: not claimed.
- **M2**: not claimed.
- **M3**: not claimed.

## Not shown yet

What is still unproven although the code exists (for example: three consecutive green runs on one commit; a run on a second
platform; a run against the real dependency). This list is part of every status report.

- Everything: no code exists yet; the specification was drafted on 2026-11-12 and has not been through T-00.01 (owner decisions are proposed, not decided).
- The Copilot SDK behaviours of COPILOT-SDK §9 (A-04, A-05, A-08 to A-12): carried over from the chess project's documents, never run against a real login here.
- authservice's claim names, endpoints and image (A-01, A-02): taken from its README, not yet read from source or run (T-01.01).
- Three consecutive green runs of the pipeline on one commit (T-00.03).

## Owner actions outstanding

- T-00.01: confirm the (proposed) rows of `DECISIONS.md`, state the dates of D-08 and the N of M2.
- T-00.02: repository settings (default branch, branch protection, secret scanning, GHCR).
- T-09.01: Fly.io organisation and token, payment provider account (sandbox), e-mail provider, bring-your-own-key model account, GitHub OAuth app, domain.
- T-02.06: a machine with a GitHub Copilot login for the live smoke run; read the current Copilot terms (A-04).
- T-10.02: a lawyer and an accountant for the legal texts and A-06, A-07, A-13.
