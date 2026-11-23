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
| T-00.01 | merged | https://github.com/konradcinkusz/gabinety-tech-spec/pull/2 (check `spec` green, run 43439862658) | 2026-11-13 | Owner decisions recorded; defaults accepted by owner; ADR-0009 splits repos |
| T-00.02 | not-started | | | Repository settings |
| T-00.03 | merged | PRs https://github.com/konradcinkusz/gabinety-tech/pull/1, /pull/2, /pull/3, /pull/4. Three green runs of `ci` on commit 5645ebb: https://github.com/konradcinkusz/gabinety-tech/actions/runs/43908188697 (attempts 1-3); `secret-scan` https://github.com/konradcinkusz/gabinety-tech/actions/runs/43908188672 (attempts 1-3) | 2026-11-16 | Shells, kernel (telemetry, JWT, internal key, rate limit, origin guard, health, features), AppHost fake profile, Next.js shell, jobs build-test/web/integration/e2e/secret-scan, `docs/operations/CI.md`. Started before T-00.02 on the owner's instruction. Caveats: Copilot runtime bundle download (A-12) not measured; `RUNNER_LABELS` switch never exercised with a variable set; `e2e` job checks only `/health` per service
| T-00.04 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/9 and /pull/10 (CI green, `integration` job: 3 tests passed, 0 skipped, run https://github.com/konradcinkusz/gabinety-tech/actions/runs/43933246622); note PR https://github.com/konradcinkusz/gabinety-tech/pull/11 | 2026-11-16 | First vertical slice against the real thing: `ThinSlice.Real_authservice_token_is_accepted` and `Altered_token_is_rejected` on the real image pinned by digest. Result: accepted; altered token 401. Revealed: no `email_verified` claim (CONTRACTS 1 amended). Not shown: administrator and multi-role tokens, e-mail confirmation required, arm64 |
| T-01.01 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/6 (CI green: build-test, web, integration, e2e, secret-scan on 0e94bd1); session session_01RjKhVNPnMWpiAFM9UDzmun | 2026-11-16 | Spike: the authservice integration surface. Director spot-checked the file:line claims against authservice commit d748af5 (`/auth/me` has no GitHub login; role claim is the long URI). A-01 differs, A-02 confirmed; amendments D1-D7 decided in the spec PR of 2026-11-16. Digest read from the registry, not from the repository; multi-role claim shape and RS256 `kid` header are conclusions not shown by a running container |
| T-01.02 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/7 (CI green: all 5 jobs on b9036fa); session session_01RjKhVPJmFvTdrsJiUXTnBr | 2026-11-16 | Spike: the Copilot SDK in this solution. Compile-only probe of all 27 members, 0 errors and 0 warnings on SDK 1.0.28 and 1.0.34 with the GHCP001 pragma; recommendation: keep 1.0.28, bump to 1.0.34 after T-02.06. A-12 confirmed by the session (CLI 1.0.105, 89,395,448 bytes for SDK 1.0.28; SHA-256 checked against the release); A-05 unknown. The director could not re-measure the bundle sizes (the `github/copilot-cli` repository is outside the session's scope); the 89 MB agrees with the chess project's documents |
| T-01.03 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/5 (CI green: all 5 jobs) | 2026-11-16 | Contracts project: records of CONTRACTS 3.2, 3.3, 4.2, 4.4, 5.2, 6; wire format pinned by tests (Money, string enums, UTC instants with Z). `github-link` records follow with T-03.05 |
| T-01.04 | merged | PRs https://github.com/konradcinkusz/gabinety-tech/pull/12 (DevIdentity and the suite; CI green) and /pull/13 (real-image wiring); `integration` run https://github.com/konradcinkusz/gabinety-tech/actions/runs/43935407280: `RealAuthServiceContractTests` 9 of 9 passed, 0 skipped; session session_01RjKhVQJQSgGBuAmUZ7iNHT | 2026-11-16 | DevIdentity (in-memory, RS256, 10 seeded synthetic users, refuses to start without GABINETY_FAKE=1) and the AuthContract suite, which passes against DevIdentity in `build-test` (37 tests) and against the real authservice v0.3.4 in `integration`. Tests first (36 of 37 failed before the code). The suite asserts the exact claim-name set of the real token. A-03 shown for the contract the suite covers; not shown: 2FA, 202 register, lockout, GitHub sign-in, administrator and multi-role claims |
| T-02.01 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/17 (CI green: all 5 jobs); main after the three merges: all 5 jobs green on 25b9cfb | 2026-11-16 | Llm: contracts, fake gateway, options, validator, health. 44 Llm tests; test-first (42 of 44 failed before the code); a mutation showed one test too weak and it was tightened. 833 lines, over the 500 limit, accepted unsplit. Keys never appear in messages or ToString (tested). Copilot gateway is a placeholder reporting Unavailable until T-02.02 |
| T-02.02 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/19 (CI green: all 5 jobs) | 2026-11-16 | Llm: the Copilot gateway ported from the chess project; 58 Llm tests; R4 (start never throws), R8 (timeout applied twice, abort, TimeoutException), token redacted from the Unavailable detail (tested); a mutation removing `AvailableTools = []` failed a test. Never run against a live runtime or login (T-02.06) |
| T-02.03 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/21 (CI green: all 5 jobs) | 2026-11-16 | Llm: usage, token prices, bring-your-own-key models; 91 Llm tests; 3 mutations each failed a test. Deviation from the porting map: `Llm:UsdPerAiCredit` was not ported (custom model prices are USD per million tokens, so no conversion is needed here); it returns when the Assistant needs a USD estimate for Copilot credits. Live behaviour not shown (T-02.06) |
| T-02.04 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/20 (CI green: all 5 jobs); main after both merges green on 101f106 | 2026-11-16 | Architecture tests, one per rule; each rule shown failing with a deliberate violation in the PR. One documented exception as data: AppHost -> DevIdentity (see decision log). No Dockerfile exists yet, so the Dockerfile half of the DevIdentity rule was shown only with a temporary file |
| T-02.05 | not-started | | | Live smoke command for the Assistant |
| T-02.06 | not-started | | | Live smoke run of the gateway |
| T-03.01 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/22 (CI green on c62fcdd: all 5 jobs; `integration` ran `SkeletonMigrationIntegrationTests.Migration_runs_once_and_a_second_start_is_a_no_op` on PostgreSQL 17, passed in 9 s); main green on a77d0c6 | 2026-11-16 | Directory skeleton: DbContext with provider switch, migration hosted service after ApplicationStarted, empty baseline migration, policies Authenticated/Admin/Internal, OpenAPI at /openapi/v1.json, /health/details listing `database` and `identity`; Program.cs 27 lines. Findings: MSB3277 on EF Relational 10.0.4 vs 10.0.15 fixed by pinning (code PR #23); the kernel's /health lists no check names, so /health/details exists. Not shown: `/health/details` is unauthenticated and reveals only Healthy/Degraded names |
| T-03.02 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/25 (CI green on 3a1daf0, all 5 jobs; `integration` ran the slug-collision test on PostgreSQL 17); follow-up https://github.com/konradcinkusz/gabinety-tech/pull/27 | 2026-11-16 | Directory: practitioner profile, NIP, contact detector, slug, state machine. CI first failed because the T-03.01 migration test expected one history row; the director made it compare against every declared migration (stronger, deterministic order). Review found the phone rule flagged year ranges such as "2015-2023"; fixed in #27 (tests first). `GET` and `submit` call authservice `/auth/me` for `emailConfirmed` with a 5 s budget (the access token has no `email_verified`). Not shown: detector false positives beyond year ranges; the identity lookup against the real authservice |
| T-03.03 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/31 (CI green: all 5 jobs; `integration` ran `TaxonomyIntegrationTests`: 20 categories and one meta row after the first and the second start) | 2026-11-16 | Directory taxonomy: `categories`, `taxonomy_meta` (key `id = 1` with a CHECK), idempotent seeder after the migration service, deactivate-not-delete, no `HasData` (file scan test). 2 mutations each failed a test. The seeder's unique-key retry path has no test |
| T-03.04 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/35 (CI green: all 5 jobs; `integration` ran the filter, the migration and 6 concurrent creates: exactly 5 created and 1 `too_many_offerings`) | 2026-11-16 | Directory offerings; the 5-active limit is serialised by `SELECT ... FOR UPDATE` on the practitioner row inside the retrying execution strategy (the pattern for T-04.05). Tests remove the global rate limiter (60 per minute per address would throttle a big suite) |
| T-03.05 | in-flight | session_01RjKhVM6P4gJ28WmAakKMbx (claude-sonnet-5-5, effort medium), branch task/T-03.05-verification | 2026-11-16 | Directory: verification workflow |
| T-03.06 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/14 (CI green: all 5 jobs) | 2026-11-16 | Ranking library, done by the director (all 3 session slots were in use). 14 tests: the three CONTRACTS 9.3 vectors within 0.005, monotonicity grids, bounds, ties, display rule; 4 mutations each failed a test. Not used by any service yet |
| T-03.07 | not-started | | | Directory: public search, profile and sitemap |
| T-03.08 | not-started | | | Directory: internal endpoints |
| T-04.01 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/16 (CI green: all 5 jobs) | 2026-11-16 | Billing library: Money, Fees, RefundPolicy, BookingStateMachine; 36 tests. A mutation (+5000 to +4999) showed the CONTRACTS 9.1 vectors do not pin the half-up tie; a test derived from the formula was added (10 -> fee 2). Follow-up: `Gabinety.Billing.BookingStatus` and `Gabinety.Contracts.Booking.BookingStatus` are two enums with the same members; the Booking service (T-04.03+) needs a mapping test |
| T-04.02 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/15 (CI green: all 5 jobs) | 2026-11-16 | Scheduling library: SlotGenerator; 10 tests incl. both DST vectors, 12 h and 60 day boundaries, buffer, exception, d=60, property over 200 seeded cases. Limit: rules use TimeOnly, so an end of 24:00 cannot be expressed (an availability window may end at 23:30 at the latest); T-04.03 validation must say so |
| T-04.03 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/28 (CI green: all 5 jobs; `integration` ran 3 Booking tests on PostgreSQL 17) | 2026-11-16 | Booking skeleton, availability API, `DirectoryClient` (3 s, one retry on GET), `directory` health check; 48 Booking tests; 4 mutations. Known: a midnight end is not expressible; concurrent PUTs of the same new exception date can return 500; DirectoryClient bypasses the kernel's retry pipeline while Directory's e-mail lookup removes it: two ways to the same end, to be unified in the kernel |
| T-04.04 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/32 (CI green; the concurrent-insert test (10 rounds) passed 3 of 3 runs of the same commit, run 43962140318) | 2026-11-16 | Booking slots and the no-double-booking constraint. FOUND BY THE REAL DATABASE: the loser of two concurrent overlapping inserts sometimes gets `40P01` (deadlock) instead of `23P01`, and without a retrying strategy EF wraps it as `InvalidOperationException` (transient failure); `BookingSlotGuard.IsSlotTaken` now searches the whole exception chain for either. An expired hold still blocks in the database until `ExpireStaleHoldsAsync` runs: the creation flow must call it first |
| T-04.05 | in-flight | session_01RjKhVPU94WLgwf1GLHXA2q (claude-sonnet-5-5, effort medium), branch task/T-04.05-booking-create | 2026-11-16 | Booking: creation, hold and the payment abstraction |
| T-04.06 | not-started | | | Booking: webhook, simulate and confirmation |
| T-04.07 | not-started | | | Booking: cancellation, dispute, no-show and resolution |
| T-04.08 | not-started | | | Booking: jobs, fee, payout and ledger export |
| T-04.09 | not-started | | | Booking: notifications |
| T-04.10 | not-started | | | Booking: stats push, reviewability, erasure, test clock |
| T-05.01 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/26 (CI green: all 5 jobs; `integration` ran the Reputation migration test, 12 s) | 2026-11-16 | Reputation skeleton, three tables, the five deterministic rules and `PublicationDecider`; 68 Reputation tests; 3 mutations each failed a test. Rules read text with control and zero-width characters removed (cannot be evaded by them); a URL with user-info trick is handled. A false positive only holds a review for moderation |
| T-05.02 | not-started | | | Reputation: create, list, reply and rating summaries |
| T-05.03 | not-started | | | Reputation: moderation |
| T-05.04 | not-started | | | Reputation: triage, summary push, erasure, test clock |
| T-06.01 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/34 (CI green: all 5 jobs; `integration` ran the Assistant migration test) | 2026-11-16 | Assistant skeleton, status, `ITrafficPolicy`, usage and the budget guard; 31 Assistant tests; 2 mutations. The status fails closed (`Disabled`) when the budget check throws. The AppHost has no `assistantdb` resource yet (default profile only; the service falls back to InMemory without it) |
| T-06.02 | not-started | | | Assistant: the brief feature |
| T-06.03 | not-started | | | Assistant: the review triage feature |
| T-06.04 | not-started | | | Evaluation suite for the two prompts |
| T-06.05 | not-started | | | Live evaluation run |
| T-07.01 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/30 and /pull/33 (CI green: web job) | 2026-11-16 | Web BFF: runtime config, session cookies, catch-all proxy (header allow-list, path safety, 1 MB, SPEC 5.1 timeouts), edge proxy `src/proxy.ts` (Next 16 renamed `middleware`) verifying RS256 with `jose`, origin guard, generated types for Directory and Reputation. 104 vitest tests; 3 security mutations. #33 renews the session on navigation instead of logging out every hour. Not shown: a browser flow, the real authservice. Staging must set `Identity__Issuer` and `Identity__Audience`, else only signature and expiry are verified |
| T-07.02 | in-flight | session_01RjKhVMmMbPSrVsW8bE2L6a (claude-sonnet-5-5, effort medium), branch task/T-07.02-web-ui-base | 2026-11-16 | Web: design tokens, layout, UI primitives, Polish strings |
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
| T-11.01 | merged | https://github.com/konradcinkusz/gabinety-tech-spec/pull/20 | 2026-11-22 | Contracts, ADR-0013 and decisions for wave 3 |
| T-11.02 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/166 | 2026-11-22 | Directory: technologies on the card, price sort |
| T-11.03 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/174 | 2026-11-22 | Directory: next free time stored, exposed, sortable |
| T-11.04 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/177 | 2026-11-22 | Booking: pushes the next free time (earliest slot of the 14-day window) |
| T-11.05 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/179 | 2026-11-22 | Web: sort, technologies, next free time on the list (time shown in Warsaw time, not the visitor's) |
| T-11.06 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/169 | 2026-11-22 | DevIdentity: forgot, reset, change password; /dev/last-reset-token only with GABINETY_FAKE=1 |
| T-11.07 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/172 | 2026-11-22 | Web: forgot and reset password pages; the reset page sends `strict-origin` since PR 187 (was `no-referrer`, which made the form post fail the origin guard) |
| T-11.08 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/178 | 2026-11-22 | Web: change password on /konto; password fields are never echoed back |
| T-11.09 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/165 | 2026-11-22 | Billing: the reschedule rule |
| T-11.10 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/175 | 2026-11-22 | Booking: reschedule endpoint, migration (hand-written), Rescheduled notification; practitioner of the booking gets 403, a stranger 404 |
| T-11.11 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/180 | 2026-11-22 | Web: reschedule page and entry |
| T-11.12 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/181 | 2026-11-22 | Booking: notification preferences, migration (hand-written), erased with the subject |
| T-11.13 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/185 | 2026-11-22 | Web: notification preferences on /konto |
| T-11.14 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/167 | 2026-11-22 | Booking: from and to on the bookings list |
| T-11.15 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/183 | 2026-11-22 | Booking: earnings endpoint (period rules decided in the PR) |
| T-11.16 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/184 | 2026-11-22 | Web: practitioner calendar (week view) |
| T-11.17 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/186 | 2026-11-22 | Web: earnings card on the dashboard, links to the calendar |
| T-11.18 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/176 | 2026-11-22 | Web: help page and support address |
| T-11.19 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/168 | 2026-11-22 | Web: GitHub avatar proxy |
| T-11.20 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/187 | 2026-11-22 | Journeys J8 and J9 (J9 found and fixed the reset-form origin bug); UI gate and audit cover the wave 3 routes |
| T-11.21 | merged | https://github.com/konradcinkusz/gabinety-tech/pull/187 (last PR of the wave) | 2026-11-22 | Wave 3 checkpoint. Every PR of the wave passed `ci` (build-test, integration, e2e, secret-scan) on its head before merge; the full `integration` and `e2e` jobs were not re-run on `main` after the last merge in this pass |

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

- All behaviour: the code is shells with health endpoints. No feature exists. Decisions were recorded 2026-11-13 (defaults); a lawyer has not seen them.
- The Copilot SDK: members compile (T-01.02); behaviours of COPILOT-SDK §9 (A-04, A-05 sign-in half, A-08 to A-11) were never run against a real login. A-12 (bundle download) is reported by the T-01.02 session, not re-measured by the director.
- authservice: claim names, endpoints and image were read from source by T-01.01 (2026-11-16); never run. The multi-role claim shape, the RS256 `kid` header and the body of `POST /auth/2fa/login` are not shown (T-00.04 runs the real image).
- Pipeline stability over time: three green runs of one commit on one day (2026-11-16) are shown; a longer history is not.
- The default profile (PostgreSQL and authservice in Docker) has not been run; the fake profile was run locally and in CI (`e2e` job) with only a `/health` check per service.
- The booking exclusion constraint is shown on PostgreSQL 17 (T-04.04, 3 of 3 runs). The booking creation flow that uses it end to end is not built yet (T-04.05).
- Follow-ups recorded, not yet tasks: make the hard-coded migration lists of the Booking and Reputation integration tests dynamic like Directory's; unify the "client without the kernel's retry pipeline" approach (Booking `DirectoryClient`, Directory e-mail and GitHub lookups); the concurrent `PUT /me/availability` of one new exception date can return 500; wire `assistantdb`, `bookingdb`, `reputationdb`, `directorydb` into the AppHost default profile.

## Owner actions outstanding

None of these block work; the director continues around them (DECISIONS log, 2026-11-16).

- D-08 dates and D-26 provider: state by 2026-12-31 (decision log). N of M2 = 4 weeks is a default.
- Approve or reject spec PR https://github.com/konradcinkusz/gabinety-tech-spec/pull/6 (CONTRACTS 4.1 constraint expression); T-04.04 is blocked on it.
- T-00.02 (no longer blocks the scaffold, which is merged, but protects `main`): on BOTH repos set default branch `main`, auto-delete head branches, branch protection per D-13, secret scanning and push protection, GHCR. I cannot read or set these.
- T-09.01: Fly.io organisation and token, payment provider account (sandbox), e-mail provider, bring-your-own-key model account, GitHub OAuth app, domain.
- T-02.06: a machine with a GitHub Copilot login for the live smoke run; read the current Copilot terms (A-04).
- T-10.02: a lawyer and an accountant for the legal texts and A-06, A-07, A-13.

## Owner-only items after wave 3 (2026-11-22)

- authservice: set the SMTP secrets so the password-reset mail is really sent (until then the NoOp mail service only logs it).
- Set `Support:Email` on the web app so the help page shows the contact address (the page works without it).
- Send one real reset mail and check that the link starts with the site address (`FrontendBaseUrl`, `deploy/fly/authservice.fly.toml`; replace the `<web-public-host>` placeholder with the real host).
- Not run here: Docker-based integration tests of the hand-written migrations `T-11.03`, `T-11.10`, `T-11.12` ran only in CI; check them again on the first deploy.
- Known gap: the next free slot on the list is shown in Warsaw time (no client-side local time, `enhance.js` has a size budget).
