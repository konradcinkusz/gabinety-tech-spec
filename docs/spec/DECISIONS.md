# Owner decisions

Status: decided

> **How to read this file.** Every row below is answered. The answers marked **(given)** come from the owner's brief. The
> answers marked **(decided, default accepted by owner 2026-11-13)** were written by the author of the specification so that the director session can start; they
> are defaults, not decisions, and an agent must not treat them as taken. The director's first act (SUPERVISOR §2,
> task T-00.01) is to show the owner the **(decided, default accepted by owner 2026-11-13)** rows in one message, take corrections, write them in, and set
> `Status: decided`. From then on CI (`scripts/check-spec.sh`) fails on any unfilled placeholder left in `docs/spec/`.
>
> Why this file exists: in the project this template was extracted from, none of these were asked up front. Each one surfaced
> later as a blocked milestone, a red pipeline or a rewritten plan (see [`LESSONS.md`](LESSONS.md)).
> An agent must not guess an answer here. "Not decided yet" is a valid answer only with a date by which it will be.

## 1. Product

| # | Decision | Answer |
|---|---|---|
| D-01 | Repository name and the names derived from it (solution, namespaces, images) | **(decided, default accepted by owner 2026-11-13)** Specification repository `gabinety-tech-spec` (docs, tasks, decisions, progress); code repository `gabinety-tech` (all code, CI, Dockerfiles). Supersedes ADR-0006 (ADR-0009). Solution `GabinetyTech.sln`, projects `Gabinety.*`, image prefix `ghcr.io/konradcinkusz/gabinety-`.  |
| D-02 | One sentence: who uses it and what they get | **(given, rephrased)** Clients with a software problem find a verified programmer ("gabinet") in a niche, book one fixed-price 30 or 60 minute paid consultation in a calendar, and leave a review that only a completed consultation allows; programmers get a free public profile, bookings and payment handling, and pay a fee only on the first consultation with a new client |
| D-03 | What is explicitly **not** built (scope creep is rejected against this list) | **(decided, default accepted by owner 2026-11-13)** No scraping or copying of profiles, reviews, texts or branding of ZnanyLekarz or any other site. No video hosting (the practitioner supplies a meeting link). No project quotes, contracts, escrow for projects, invoicing automation or subscription (Pro) in the MVP. No paid placement or boost in ranking. No in-platform chat in the MVP. No mobile apps. One language (Polish UI) and one currency (PLN). No teams or agencies: one person, one profile. No use of the owner's Copilot subscription for public traffic. No bidding or hourly marketplace. No general "all programmers" catalogue: one niche first (D-22) |
| D-04 | Data sensitivity: may real user data ever be in a test, a log or a fixture? | **(decided, default accepted by owner 2026-11-13)** No. Fixtures are synthetic, in the reserved namespace `demo-`. Free text typed by users (a brief, a review) is never written to a log or a trace; only its length and a hash are |

## 2. Platforms and gates

| # | Decision | Answer |
|---|---|---|
| D-05 | Platforms a milestone gate must be shown on (Linux, Windows, macOS, WSL). List only those you will actually run | **(decided, default accepted by owner 2026-11-13)** Linux (GitHub-hosted `ubuntu-latest` and the cloud sessions). No other platform is a gate. The owner's machine is used for the live smoke run only (D-20) |
| D-06 | How the system is started for the first gate: Aspire AppHost, `docker compose`, or both. If both, which one is the gate | **(decided, default accepted by owner 2026-11-13)** Aspire AppHost, profile `fake`: `dotnet run --project src/Gabinety.AppHost --launch-profile fake` starts every service, the web app and the identity stand-in with no Docker and no credentials. The default profile needs Docker (PostgreSQL, authservice image). No `docker compose` file in the MVP |
| D-07 | Deployment target (Fly.io, Azure, on-prem compose, none yet) | **(decided, default accepted by owner 2026-11-13)** Fly.io, region `fra`, one app per service, one shared PostgreSQL app with one database per service (P7). One environment, `staging`, until M3. No PR environments in the MVP. No Azure track |
| D-08 | Anything that needs hardware or an account you do not have yet, and when it arrives | **(decided as a list; dates: see decision log)** (1) Payment provider account with split payments or connected accounts, sandbox first. (2) A domain name. (3) A transactional e-mail provider account. (4) A bring-your-own-key model account (Anthropic, OpenAI or Azure) for the Assistant. (5) A Fly.io organisation. (6) A GitHub OAuth app for authservice. (7) A GitHub Copilot login on the owner's machine for the live smoke run. (8) A lawyer or accountant for terms, privacy text and invoicing questions. M2 is dated "arrival of (1) + N weeks", never by calendar |

## 3. CI

| # | Decision | Answer |
|---|---|---|
| D-09 | Runners: GitHub-hosted, or self-hosted (then set the repository variable `RUNNER_LABELS`) | **(decided, default accepted by owner 2026-11-13)** GitHub-hosted. `RUNNER_LABELS` unset |
| D-10 | If self-hosted: what is already installed in the runner image | n/a (GitHub-hosted) |
| D-11 | If self-hosted: how many jobs may run at once, and is the Docker daemon shared between runners | n/a (GitHub-hosted) |
| D-12 | Default branch, and whether merged branches are deleted automatically | **(decided, default accepted by owner 2026-11-13)** `main`; auto-delete head branches: yes |
| D-13 | Required checks / branch protection | **(decided, default accepted by owner 2026-11-13)** On `main`: pull request required, no direct push, checks `spec-check`, `build-test`, `web`, `integration`, `e2e`, `secret-scan` required. Squash merge only |

## 4. Delegation

| # | Decision | Answer |
|---|---|---|
| D-14 | Model for the supervisor session and for implementer sub-sessions | **(given for implementers; director: see decision log)** Director (supervisor): the strongest model available to the owner, proposed `claude-opus-5-5`, chosen by the owner when the session is started. Implementers: `claude-haiku-5-5`. If the owner wants the director itself on Haiku, say so in T-00.01; SUPERVISOR §10 then applies to the director as well |
| D-15 | How sub-sessions are started (separate sessions visible to the owner, or in-process agents) | **(decided, default accepted by owner 2026-11-13)** Separate sessions visible in the Claude app, created with the `create_session` tool of the `claude-code-remote` server (`model: claude-haiku-5-5`, one session per task, the brief as `prompt`); in-process agents only for read-only research (SUPERVISOR §11) |
| D-16 | Maximum sub-sessions at once (start with 2 on self-hosted runners; raise only when CI stays stable) | **(decided, default accepted by owner 2026-11-13)** 3 on GitHub-hosted runners. 1 during Phase 0. Lower to 1 when CI is unstable (SUPERVISOR §6) |
| D-17 | Autonomy: may the supervisor merge green PRs without asking? May it archive finished sessions? | **(decided, default accepted by owner 2026-11-13)** Merge: yes, after the §7 review checklist, for tasks with `Executor: implementer`. No for `Executor: supervisor` tasks that change a contract or a standard, and no for anything that changes money rules, fees, legal text or a deviation: those wait for the owner. Archive: yes, after the PR is merged and `PROGRESS.md` is updated |
| D-18 | What the supervisor must always ask about (spend, deleting things, anything outside the repository) | **(decided, default accepted by owner 2026-11-13)** Spending money or credits (payment provider, model keys, domains, Fly resources beyond the plan in SPEC §4.3); deleting a repository, branch with unmerged work, app or database; creating repositories or accounts; publishing anything publicly or opening the site to the internet; any fee, price, legal or privacy text; any use of the owner's Copilot subscription; any real personal data |

## 5. Evidence

| # | Decision | Answer |
|---|---|---|
| D-19 | What counts as proof that a gate passed (CI run link on a named commit, a recording, a measured number) | **(decided, default accepted by owner 2026-11-13)** A CI run link on a named commit for every automated criterion; for a user-visible gate also the Playwright report and its trace artifact of the journey that passed; for a performance target the measured number with the command and environment; for an owner action the owner's confirmation in the PR |
| D-20 | First real-world result required before any feature beyond the first vertical slice (e.g. one real model answer, one real customer file, one deploy) | **(decided, default accepted by owner 2026-11-13)** Two. (a) A token from a real authservice instance (container image, RS256) is accepted by Directory and rejected when altered, in CI (T-00.04). (b) One real model answer to a brief, produced through `Gabinety.Llm` with the owner's Copilot login, recorded by the owner's live smoke run (T-02.06). Until (b) is recorded the Copilot behaviours in `COPILOT-SDK.md` §9 stay assumptions |

## 6. Product parameters

Added to the template: the questions this product raises that the 20 above do not cover.

| # | Decision | Answer |
|---|---|---|
| D-21 | Product name and domain | **(given, name)** gabinety-tech, renamed from znany-programista because of the closeness to the ZnanyLekarz mark. **(decided, default accepted by owner 2026-11-13)** Display name "Gabinety Tech"; domain not chosen; the owner runs a trademark search before buying one (D-08) |
| D-22 | First niche | **(decided, default accepted by owner 2026-11-13)** .NET: legacy migration, and AI integration and automation. Other categories exist in the taxonomy but are not marketed. Supply target before launch: at least 15 approved practitioners in the niche (owner-run outreach, not a build task) |
| D-23 | Platform fee | **(decided, default accepted by owner 2026-11-13)** 15 % of the price of the first completed consultation of a client-practitioner pair, rounded half up to a grosz; nothing on later consultations; payment-provider costs absorbed by the platform. The brief gives no rate; this one is a placeholder for the owner's number |
| D-24 | Consultation product | **(decided, default accepted by owner 2026-11-13)** Duration 30 or 60 minutes. Fixed price per practitioner offering, between 50 PLN and 1500 PLN. Held in the practitioner's own video tool; the link is shown to the client only after payment |
| D-25 | Cancellation, completion and payout rules | **(decided, default accepted by owner 2026-11-13)** Client cancels 24 h or more before start: full refund. Less than 24 h: no refund, treated as completed for money, no review. Practitioner cancels at any time: full refund. Practitioner no-show: client reports within 72 h: full refund and a ranking penalty. Client no-show: practitioner reports within 72 h: treated as completed for money. Client may open a dispute within 48 h after the end; otherwise the booking completes and the payout is released. An administrator resolves disputes |
| D-26 | Payment provider | **(open, owner's choice)** Candidates, none verified here: Stripe Connect, Przelewy24, Tpay. Required: split payments or connected accounts, refunds, signed webhooks, a sandbox. MVP gates M0 and M1 use the mock |
| D-27 | Language and locale | **(decided, default accepted by owner 2026-11-13)** User-facing text Polish, in one place (`web/src/i18n/pl.ts` and the e-mail templates). Code, comments and docs English. Stored times UTC, shown in Europe/Warsaw. Currency PLN |
| D-28 | Minimum verification before a profile is public | **(decided, default accepted by owner 2026-11-13)** E-mail verified (authservice) + GitHub account linked through authservice + valid NIP (format and checksum; no registry lookup in the MVP) + a portfolio or repository link + manual approval by an administrator. Each is shown as its own badge |
| D-29 | Review rules | **(decided, default accepted by owner 2026-11-13)** Only the client of a booking that ended as completed (not late-cancelled), once per booking, within 14 days; rating 1 to 5 and text of 20 to 2000 characters; the practitioner may reply once and may not remove a review; the aggregate rating is shown only from 3 reviews, below that the page says "Nowy" |
| D-30 | Ranking | **(decided, default accepted by owner 2026-11-13)** The formula and weights in CONTRACTS §9.3, published at `/zasady-rankingu` with a version and date. No paid input |
| D-31 | AI features and their limits | **(decided, default accepted by owner 2026-11-13)** Two: a brief assistant for clients and an advisory review triage for the moderator. Public traffic only on bring-your-own-key models. A daily budget of 5 USD for the deployed Assistant; when it is spent the features switch off and the plain form is used |
| D-32 | Frontend | **(decided, default accepted by owner 2026-11-13)** Next.js with a BFF (ADR-0001) |
| D-33 | Who moderates and approves at the start | **(decided, default accepted by owner 2026-11-13)** The owner, as authservice `Admin`. No other moderators in the MVP; the moderator role exists in Directory for later |

## Decision log

Decisions taken after the start go here, newest first, with the date and the reason. A decision that changes the specification
lands in the same pull request as the change to `SPEC.md`.

| Date | Decision | Reason |
|---|---|---|
| 2026-11-13 | D-14: the director runs on `claude-sonnet-5-5` (the model the owner started the session on); implementers `claude-haiku-5-5`. Not Opus as proposed | Owner kept the session model |
| 2026-11-13 | D-08: no account exists yet and no date is known. Each item is "not decided"; the owner states a date before T-09.01 starts, and in any case by 2026-12-31 | Owner asked for defaults; a date cannot be invented for an account |
| 2026-11-13 | M2 N = 4 weeks after arrival of the payment sandbox account (default, owner may change) | Owner asked for defaults |
| 2026-11-13 | D-26 payment provider stays open with the default mock for M0 and M1; decided by 2026-12-31 or before T-09.05 starts | Provider choice needs the owner's research; mock needs none |
| 2026-11-13 | D-23 fee 15 % accepted as the owner's number by default; D-25 and D-24 accepted unchanged | Owner asked for defaults. The fee and legal texts are still reviewed by a lawyer at T-10.02 |
| 2026-11-13 | D-01: two repositories (ADR-0009 supersedes ADR-0006) | Owner instruction: spec outside the code repository |
| 2026-11-12 | Specification drafted; rows marked (proposed) await the owner (T-00.01) | The brief answered the product and the delegation model but not the numbers |
