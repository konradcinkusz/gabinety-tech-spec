# Gabinety Tech — Phases and atomic tasks

> Companion to [`SPEC.md`](SPEC.md) and [`CONTRACTS.md`](CONTRACTS.md). Execution protocol: [`SUPERVISOR.md`](SUPERVISOR.md).
>
> Every task is sized for **one sub-session, one branch, one pull request** and for an implementer on **Claude Haiku 5.5**
> (SUPERVISOR §10): about 300 changed lines at most, not counting generated migrations and fixtures. A task is done when its
> **Accept** criteria hold and its **Verify** command passes in CI, not when the code is written.

## How to read a task

```
### T-PP.NN Title
- Executor: implementer | supervisor | owner     who does it (owner = a person; an agent cannot)
- Milestone: M0 | M1 | M2 | M3
- Depends: task ids that must be merged first, or —
- Read: SPEC sections and standards guides to load before editing
- Files: paths it may create or modify (anything else is out of scope)
- Do: the work, as numbered steps
- Accept: observable conditions, each one testable
- Verify: the command CI runs and the sub-session runs before opening the PR
```

`scripts/check-spec.sh` fails when a task block lacks one of these eight fields or when two tasks share an id.

Standards guides are named by skill from `architecture-standards` (for example `reference-architecture`, `repo-baseline`,
`service-api-patterns`, `testing-strategy`, `e2e-acceptance-testing`, `frontend-bff`, `identity-and-accounts`,
`payments-and-monetization`, `ai-evals`, `security-review`, `fly-io-deployment`, `init-generic-template`, `implementation-phase`).
Every implementation task follows `implementation-phase`.

Common **Verify** for a .NET task unless the task says otherwise (called "common verify" below):
`dotnet build -warnaserror && dotnet test --filter "tier!=integration"`.
Common **Verify** for a web task: `pnpm --dir web install --frozen-lockfile && pnpm --dir web lint && pnpm --dir web typecheck && pnpm --dir web test && pnpm --dir web build`.

Rules for writing tasks:

- **Executor `owner`** marks what only a person can do (repository settings, accounts, hardware, a signature). These are
  collected early so they do not block a gate at the end.
- **Files are disjoint** between tasks that may run in parallel. Shared files (package versions, the solution, the CI
  workflow, the AppHost, `Program.cs` of a service, shared contracts, `CONTRACTS.md`, `SPEC.md`) belong to the supervisor.
  Every project of SPEC §3.2 exists as a buildable shell after T-00.03, so no implementer task adds a project or edits the solution.
- **Features register themselves.** A service feature is a folder `Features/<Name>/` with a class implementing `IServiceFeature`
  (kernel: `Add(builder)` and `Map(endpoints)`); `Program.cs` calls `AddFeatures()` once and is never edited by a feature task.
- **A schema change is serial within a service.** Tasks of one service that add a migration depend on each other in a chain; one
  migration per task, named `<TaskId>_<Name>`. Migrations are provider-specific (PostgreSQL); tests on InMemory do not use them.
- **Query code has an integration test.** A task that adds a list, filter, sort or constraint adds a `tier=integration` test on a real PostgreSQL.
- **A spike** produces a note under `docs/spikes/` with exact versions and a recommendation. If a spike is skipped, the task
  is not "done": it is replaced by a row in SPEC §12.1 (assumption, not verified).
- **Accept criteria name their proof**: a test name, a command with its expected output, or a CI job.
- **Constants of the owner** (fee rate, ranking weights, price bounds, windows) are read from CONTRACTS and configuration; a task
  never invents one (DECISIONS D-17).

---

## Phase 0 — Ground truth before code

These four tasks are fixed. They exist because skipping them cost the most in the project this template came from.

### T-00.01 Owner decisions recorded
- Executor: owner · Milestone: M0 · Depends: —
- Read: `DECISIONS.md`
- Files: `docs/spec/DECISIONS.md`
- Do: 1) The director shows the owner every row marked (proposed) in one message. 2) The owner corrects or accepts each row and states the dates of D-08 and the N of M2. 3) Replace each "(proposed)" with "(decided)" for accepted rows, record corrections in the decision log, set `Status: decided`.
- Accept: no "(proposed)" and no placeholder left in `docs/spec/DECISIONS.md`; status line reads `Status: decided`; changes to a number of CONTRACTS (fee, windows, bounds) are carried into CONTRACTS in the same PR.
- Verify: `scripts/check-spec.sh`

### T-00.02 Repository settings
- Executor: owner · Milestone: M0 · Depends: T-00.01
- Read: DECISIONS D-09–D-13; `repo-baseline`
- Files: —
- Do: 1) Default branch as in D-12. 2) Automatically delete head branches. 3) Branch protection and required checks as in D-13 (the checks exist after T-00.03; until then protect with `spec-check`). 4) Secret scanning, push protection and code scanning on. 5) Enable GitHub Packages (GHCR) for the owner. 6) Confirm the director session has write access to the repository and can create pull requests.
- Accept: each setting is visible in the repository settings; a manually dispatched workflow on the default branch starts.
- Verify: owner confirms in the PR that closes this task (settings are not readable by CI).

### T-00.03 Scaffold and empty pipeline green
- Executor: supervisor · Milestone: M0 · Depends: T-00.02
- Read: SPEC §3, §4.1, §6.0, §9; DECISIONS D-05–D-13; `init-generic-template`, `reference-architecture`, `repo-baseline`, `frontend-bff`
- Files: `*.sln`, `global.json`, `Directory.Build.props`, `Directory.Packages.props`, `.editorconfig`, `.config/dotnet-tools.json`, `src/**` (shells and kernel), `web/**` (scaffold), `tools/**` (shells), `tests/**` (shells), `.github/workflows/**`, `scripts/**`, `.gitleaks.toml`, `docs/operations/CI.md`
- Do: 1) Scaffold with `init-generic-template`, then arrange to SPEC §3.2: every project, test project and the Next.js app exists as a shell that builds, with the allowed references only. 2) Kernel `Gabinety.ServiceDefaults` per SPEC §6.0 including `IServiceFeature` and `AddFeatures()`, `/health` and `/alive`. 3) AppHost with both profiles of SPEC §4.1; the fake profile starts every shell. 4) Create every job of SPEC §9 that can exist now (not `docker-build`), plus the guards (kernel size, SDK isolation grep, every csproj in the solution, clock grep) in `scripts/` and the `RUNNER_LABELS` switch. 5) Pre-commit secret scan. 6) Pin `GitHub.Copilot.SDK` in `Directory.Packages.props` at the version of the chess project (1.0.28) until T-01.02 says otherwise. 7) Record per job duration, what it downloaded and what it needed from the runner image, including the Copilot runtime bundle download (A-12) and the Aspire JavaScript integration (A-16).
- Accept: every pull-request job is green **three times in a row on one commit**; `docs/operations/CI.md` holds the measured table; `dotnet run --project src/Gabinety.AppHost --launch-profile fake` starts and every service answers `/health` 200 (shown by a script `scripts/check-fake-profile.sh` that the `e2e` job runs); no job installs a toolchain at run time that DECISIONS D-10 says is in the runner image.
- Verify: links to the three runs in the PR description.

### T-00.04 First vertical slice against the real thing
- Executor: supervisor · Milestone: M0 · Depends: T-00.03, T-01.01
- Read: SPEC §7, §11, §14; DECISIONS D-20; `docs/spikes/authservice.md`; `identity-and-accounts`
- Files: `src/Gabinety.Directory/Features/Me/**`, `tests/Gabinety.Integration.Tests/ThinSlice/**`, `.github/workflows/**` (the `integration` job), `docs/spikes/first-real-result.md`
- Do: 1) `GET /api/v1/me` in Directory behind `AddJwtAuthentication`. 2) The `integration` job starts PostgreSQL and the pinned real authservice image, registers a user, logs in, calls `/api/v1/me` on Directory with the token. 3) Alter one character of the token signature and call again. 4) Write down what the stand-in (DevIdentity) will not show and what the real image revealed.
- Accept: test `ThinSlice.Real_authservice_token_is_accepted` and `ThinSlice.Altered_token_is_rejected` pass in the `integration` job; `GET /api/v1/me` answers `{ userId }` equal to the token subject; the note is in `docs/spikes/first-real-result.md`.
- Verify: `dotnet test --filter "tier=integration&FullyQualifiedName~ThinSlice"`

---

## Phase 1 — Spikes, contracts and the identity stand-in

### T-01.01 Spike: the authservice integration surface
- Executor: implementer · Milestone: M0 · Depends: T-00.02
- Read: SPEC §12.1 A-01, A-02; CONTRACTS §1; ADR-0002; `identity-and-accounts`; the `authservice` repository (public, attach read-only: `git clone --depth 1 https://github.com/konradcinkusz/authservice`)
- Files: `docs/spikes/authservice.md`
- Do: 1) From authservice's source and docs record: the claim names in the access token (subject, e-mail, e-mail verified, roles); the platform role names; the endpoints the web app needs (register, login, refresh, logout, me, GitHub sign-in start and exchange, consent versions and acceptance, export, delete) with method, path, request and response fields; the health path; the published image name, tag and digest and the port; the configuration keys needed to run it with RS256 behind Directory (private key, public base URL, database) and how a key pair is generated on Linux. 2) Mark each of A-01 and A-02 `confirmed` or `differs`, citing the authservice file and line. 3) List every difference from CONTRACTS §1, §3.3 (`github-link`) and SPEC §6.1 as a proposed amendment (do not edit those files).
- Accept: the note answers every item with a file and line reference into the authservice repository; a section "Proposed amendments" exists (possibly empty, saying so).
- Verify: `scripts/check-spec.sh`

### T-01.02 Spike: the Copilot SDK in this solution
- Executor: implementer · Milestone: M0 · Depends: T-00.02
- Read: COPILOT-SDK.md §1–§3, §9; ADR-0004; the `llm-multi-agent-chess` repository, `src/Chess.Llm/README.md` (attach read-only)
- Files: `docs/spikes/copilot-sdk.md`
- Do: 1) In a scratch project outside the repository run `dotnet add package GitHub.Copilot.SDK` and record the latest stable version and the version the chess project pins. 2) Compile a probe that touches every member of the COPILOT-SDK §3 table; record each as present or missing at that version, and any new compiler warning or experimental marker. 3) Record whether the first `dotnet build` downloads a runtime bundle, its size, host and checksum behaviour, and the folder it is cached in (A-12). 4) From the SDK documentation record what is said about a custom provider (`ProviderConfig`) and whether a session with it needs a running runtime and a sign-in (A-05); mark unknowns as unknown. 5) Recommend the version to pin.
- Accept: the note has the member table with one row per §3 member; a recommended pin; A-05 and A-12 each with `confirmed`, `differs` or `unknown` and the source; nothing in the note claims behaviour that needs a login.
- Verify: `scripts/check-spec.sh`

### T-01.03 Contracts project
- Executor: supervisor · Milestone: M0 · Depends: T-00.03
- Read: CONTRACTS §0, §3.2, §3.3, §4.2, §4.4, §5.2, §6
- Files: `src/Gabinety.Contracts/**`
- Do: 1) One record per request and response of CONTRACTS, grouped by service in folders `Directory`, `Booking`, `Reputation`, `Assistant`, `Common` (Money, Problem details extension, Page). 2) Enums as in CONTRACTS with `JsonStringEnumConverter` attributes. 3) No logic beyond constructors and validation attributes; no reference to any package.
- Accept: the project builds with no package reference; a test in `Gabinety.Architecture.Tests` asserts it has no dependency except the framework (added by T-02.04; until then checked by reading the csproj).
- Verify: common verify

### T-01.04 DevIdentity and the auth contract suite
- Executor: implementer · Milestone: M0 · Depends: T-01.01, T-00.03
- Read: ADR-0003; SPEC §6.7; `docs/spikes/authservice.md`
- Files: `tools/Gabinety.DevIdentity/**`, `tests/Gabinety.Integration.Tests/AuthContract/**`
- Do: 1) Implement the endpoints the spike lists as needed by the web app (register, login, refresh, logout, me, JWKS, OIDC discovery) on an in-memory user list with RS256 tokens whose claim names are those of the spike. 2) Seed the synthetic users of SPEC §6.7 (passwords are fixed development values stated in the README of the tool, not secrets). 3) Refuse to start unless `GABINETY_FAKE=1` and say why. 4) Write an auth contract suite as an abstract class parameterised by a base URL: register, login, wrong password, refresh rotation, replayed refresh token, me, JWKS contains one key, discovery names the JWKS. 5) Run the suite in the unit tier against DevIdentity (in process) and in the `tier=integration` tier against the real image when `AUTH_BASE_URL` is set.
- Accept: `AuthContract.*` pass against DevIdentity in `build-test`; the same tests pass against the real authservice in the `integration` job (supervisor wires the env var); starting DevIdentity without `GABINETY_FAKE=1` exits non-zero (test).
- Verify: common verify

---

## Phase 2 — The LLM gateway (`Gabinety.Llm`)

### T-02.01 Llm: contracts, fake gateway, options and health
- Executor: implementer · Milestone: M0 · Depends: T-01.02, T-00.03
- Read: COPILOT-SDK §1, §2, §6, §10, §11; `testing-strategy`
- Files: `src/Gabinety.Llm/LlmContracts.cs`, `src/Gabinety.Llm/FakeLlmGateway.cs`, `src/Gabinety.Llm/LlmOptions.cs`, `src/Gabinety.Llm/CustomModelOptions.cs`, `src/Gabinety.Llm/LlmOptionsValidator.cs`, `src/Gabinety.Llm/LlmGatewayHealthCheck.cs`, `src/Gabinety.Llm/LlmServiceCollectionExtensions.cs`, `src/Gabinety.Llm/AssemblyInfo.cs`, `tests/Gabinety.Llm.Tests/Fake*`, `tests/Gabinety.Llm.Tests/Options*`, `tests/Gabinety.Llm.Tests/Health*`
- Do: 1) Port the files named in COPILOT-SDK §11 that need no SDK type (attach the chess repository read-only and copy, then adapt). 2) The fake gateway replies per COPILOT-SDK §10. 3) Options and validator per §6 (`ToString()` without the key; messages name the setting, never a value). 4) Health check never Unhealthy. 5) `AddLlmGateway` registers the fake for `Llm:Provider=Fake` and, for `Copilot`, a placeholder registration that T-02.02 completes (the switch throws for an unknown provider).
- Accept: tests: every row of the §10 table has a test; `[[slow]]` ends in the caller's timeout and is cancelled; a bad custom model entry stops validation with a message that contains the setting path and not the key (test uses a recognisable fake key and asserts it is absent); `CustomModelOptions.ToString()` omits the key; `LlmGatewayHealthCheck` returns Degraded for `NotAuthenticated` and `Unavailable`.
- Verify: common verify

### T-02.02 Llm: the Copilot gateway
- Executor: implementer · Milestone: M0 · Depends: T-02.01
- Read: COPILOT-SDK §1 R1–R9, §3, §4, §5, §11
- Files: `src/Gabinety.Llm/CopilotLlmGateway.cs`, `src/Gabinety.Llm/CopilotSessionSettings.cs`, `src/Gabinety.Llm/CopilotClientHostedService.cs`, `src/Gabinety.Llm/CopilotOptions.cs`, `src/Gabinety.Llm/ReplyReasoning.cs`, `src/Gabinety.Llm/Gabinety.Llm.csproj`, `tests/Gabinety.Llm.Tests/Copilot*`
- Do: 1) Port the five files from the chess project, renaming namespaces and the activity source, removing the custom-model and price parts (T-02.03 adds them). 2) Complete the `Copilot` branch of `AddLlmGateway` (singleton gateway, hosted service, options binding). 3) Keep the `#pragma warning disable GHCP001` scoped to single methods with a reason. 4) Reference `GitHub.Copilot.SDK` in the csproj only. 5) Unit tests need no runtime: a missing `Copilot:CliPath` and a refused loopback `Copilot:CliUrl` both end in `Unavailable` with no exception and no network; `CopilotSessionSettings` tests of COPILOT-SDK §4.
- Accept: tests `Start_with_missing_cli_path_is_Unavailable`, `Start_with_refused_url_is_Unavailable`, `Session_has_no_tools_and_rejects_permissions`, `Session_replaces_the_system_message`; the build passes with `-warnaserror`; no `GitHub.Copilot` namespace is referenced outside `src/Gabinety.Llm`.
- Verify: common verify

### T-02.03 Llm: usage, token prices and bring-your-own-key models
- Executor: implementer · Milestone: M0 · Depends: T-02.02
- Read: COPILOT-SDK §6, §8, §11; the chess project's `UsageTally.cs`, `TokenPriceMapping.cs`, `CustomModelPricing.cs` and their tests
- Files: `src/Gabinety.Llm/UsageTally.cs`, `src/Gabinety.Llm/TokenPriceMapping.cs`, `src/Gabinety.Llm/CustomModelPricing.cs`, `src/Gabinety.Llm/CopilotLlmGateway.cs`, `src/Gabinety.Llm/CopilotSessionSettings.cs`, `tests/Gabinety.Llm.Tests/Usage*`, `tests/Gabinety.Llm.Tests/Price*`, `tests/Gabinety.Llm.Tests/Custom*`
- Do: 1) Port the three files and their tests. 2) Wire the usage and reasoning events into `AskAsync` as in the chess gateway; abort on timeout. 3) `ListModelsAsync` returns Copilot models with prices (a failure of the unit listing leaves models listed without prices and logs a warning) followed by the active custom models; a custom id equal to a Copilot id is not listed and an error is logged naming it. 4) `ForCustomModel` builds the `ProviderConfig` for `anthropic`, `openai`, `azure`.
- Accept: tests: `Unavailable` credits add no cost and no zero; reasoning tokens are not added twice; a price quoted in nano-AIU is divided by 1e9; a model with no price entry gets no `Pricing`; a custom model has `Custom = true`; `ForCustomModel` leaves the model id null and sets provider type and wire model; an `anthropic` provider sends no `WireApi`.
- Verify: common verify

### T-02.04 Architecture tests
- Executor: implementer · Milestone: M0 · Depends: T-00.03, T-01.03
- Read: SPEC §3.2, §5 rules 2 and 4, §6.0; COPILOT-SDK §1 R1, R2; the chess project's `PublicSurfaceTests`
- Files: `tests/Gabinety.Architecture.Tests/**`
- Do: 1) Test: only `Gabinety.Llm` references the package `GitHub.Copilot.SDK` (read the csproj files); no source file outside `src/Gabinety.Llm` contains `using GitHub.Copilot`. 2) Reflection test: no public member of `Gabinety.Llm` exposes a type from an assembly whose name starts with `GitHub.Copilot`. 3) Project reference test: the allowed edges of SPEC §3.2 and no other. 4) Kernel purity: `Gabinety.ServiceDefaults` references no domain project. 5) `tools/Gabinety.DevIdentity` is referenced by no project under `src/`, and is in no Dockerfile. 6) Clock guard: no `DateTime.Now`, `DateTime.UtcNow`, `DateTimeOffset.UtcNow` in `src/` outside `Gabinety.AppHost`. 7) Free-text guard: no logging call in `src/` passes a member named `Brief`, `BriefText`, `Bio`, `Text` or `Reply`. 8) `Gabinety.Contracts` has no package reference.
- Accept: each test named for its rule; each fails when its rule is violated (the PR shows, for each, the failing output of a deliberate temporary violation, then reverted).
- Verify: common verify

### T-02.05 Live smoke command for the Assistant
- Executor: implementer · Milestone: M1 · Depends: T-06.02
- Read: COPILOT-SDK §7, §11 (last row); the chess project's `scripts/smoke-copilot.sh`
- Files: `src/Gabinety.Assistant/Features/OwnerSmoke/**`, `scripts/smoke-llm.sh`, `tests/Gabinety.Assistant.Tests/OwnerSmoke/**`
- Do: 1) A command-line verb of the Assistant (`dotnet run --project src/Gabinety.Assistant -- smoke`), **not** an HTTP endpoint, that runs the brief feature once on a fixed synthetic Polish brief with the owner-run traffic policy (it may use a Copilot model) and prints model id, latency, input and output tokens and AI credits; no brief or reply text is printed beyond the parsed suggestion's slugs. 2) `scripts/smoke-llm.sh` wraps it: refuses when `CI` is set, refuses when `Llm:Provider` would be `Fake`, needs `dotnet` and says what it spends. 3) The verb is not reachable when the host runs as a web server (a test starts the web host and asserts no route exists).
- Accept: tests: under `CI=1` the script exits 2 with the refusal text; with `Llm:Provider=Fake` it exits 2; the verb never starts Kestrel; the verb refuses to run a public-traffic request path (the policy of §7 still applies to HTTP, shown by an existing test staying green).
- Verify: common verify

### T-02.06 Live smoke run of the gateway
- Executor: owner · Milestone: M1 · Depends: T-02.05
- Read: COPILOT-SDK §9; DECISIONS D-20
- Files: `docs/spikes/first-real-result.md`
- Do: 1) The owner runs `scripts/smoke-llm.sh` on a machine with a Copilot login (and, separately, against a bring-your-own-key model once T-09.01 has the key). 2) The director pastes the output into `docs/spikes/first-real-result.md` and closes A-08 to A-11 as confirmed or corrects COPILOT-SDK §9. 3) The owner reads the current Copilot terms about serving end users and records the conclusion for A-04.
- Accept: the note contains the output of one real run (no token, no personal data); each of A-08 to A-11 is `confirmed` or `differs` with the evidence; A-04 has the owner's recorded conclusion.
- Verify: owner confirms in the PR; `scripts/check-spec.sh`

---

## Phase 3 — Directory service

### T-03.01 Directory: skeleton, persistence, authentication
- Executor: implementer · Milestone: M0 · Depends: T-00.04, T-01.03
- Read: SPEC §6.0, §6.1; CONTRACTS §0, §3.1; `service-api-patterns`, `reference-architecture` P3, P4, P9
- Files: `src/Gabinety.Directory/Persistence/**`, `src/Gabinety.Directory/Features/Health/**`, `src/Gabinety.Directory/Program.cs`, `tests/Gabinety.Directory.Tests/Skeleton*`
- Do: 1) `DirectoryDbContext` with `ApplyConfigurationsFromAssembly`, provider switch (`Database:Provider`), PostgreSQL retry policy, InMemory fallback. 2) Migration hosted service that runs `MigrateAsync` after Kestrel starts (never `EnsureCreated` on PostgreSQL), with an empty baseline migration `T-03.01_Baseline`. 3) `Program.cs` as a manifest (service defaults, JWT authentication, feature discovery, OpenAPI at `/openapi/v1.json`). 4) Policies `Authenticated`, `Admin` (authservice role), `Internal`. 5) `/health` reports `database` and `identity` (empty JWKS makes it Degraded).
- Accept: `Program.cs` is at most 60 lines; tests: unauthenticated request to a protected endpoint is 401, a token with the wrong audience is 401; `/health` 200 and lists `database` and `identity`; the migration service runs once and is idempotent (second start no-op) on the integration tier.
- Verify: common verify; `dotnet test --filter "tier=integration&FullyQualifiedName~Directory"`

### T-03.02 Directory: practitioner profile
- Executor: implementer · Milestone: M1 · Depends: T-03.01
- Read: CONTRACTS §3.1, §3.3, §3.5, §8.2; SPEC §6.1
- Files: `src/Gabinety.Directory/Features/Profiles/**`, `src/Gabinety.Directory/Persistence/Migrations/T-03.02*`, `tests/Gabinety.Directory.Tests/Profiles/**`
- Do: 1) Entity, configuration and migration for `practitioners` (columns of §3.1 used so far). 2) Endpoints `POST /api/v1/me/practitioner`, `GET` and `PUT /api/v1/me/practitioner`, `POST /api/v1/me/practitioner/submit`. 3) Slug generation with unique suffix. 4) Validation of §3.5: lengths, https URLs and meeting-host allow-list, NIP checksum (`INipValidator`), contact-detail detector (`IContactDetailDetector`). 5) `profile_completeness` computed on write. 6) State machine of §8.2 as a pure class with a transition table.
- Accept: tests: NIP vectors `5260250995` valid and `5260250996` invalid; contact detector rejects an e-mail, a phone number with spaces and an `@handle` and accepts "dotnet 8 i EF Core 9"; duplicate create is 409 `practitioner_exists`; slug collision gets a suffix; submit with an unmet checklist item is 422 `submit_incomplete` listing it; every illegal transition is rejected; a second user cannot read the first user's private profile (404).
- Verify: common verify

### T-03.03 Directory: taxonomy and its seed
- Executor: implementer · Milestone: M1 · Depends: T-03.02
- Read: CONTRACTS §2, §3.2 (`GET /api/v1/taxonomy`); `reference-architecture` P4 (reference data is seeded separately)
- Files: `src/Gabinety.Directory/Features/Taxonomy/**`, `src/Gabinety.Directory/Persistence/Migrations/T-03.03*`, `src/Gabinety.Directory/Persistence/Seeds/**`, `tests/Gabinety.Directory.Tests/Taxonomy/**`
- Do: 1) Entities and migration for `categories` and `taxonomy_meta`. 2) A versioned seed script (an idempotent hosted service reading an embedded JSON of the seed set, upserting by slug, raising the version when the set changes); **no `HasData`**. 3) `GET /api/v1/taxonomy`.
- Accept: tests: seeding twice changes nothing; the endpoint returns the 8 problems and 12 techs of CONTRACTS §2 with Polish names; removing a slug from the seed deactivates it (`is_active = false`) and does not delete it; the migration files contain no seed rows.
- Verify: common verify

### T-03.04 Directory: offerings
- Executor: implementer · Milestone: M1 · Depends: T-03.03
- Read: CONTRACTS §3.1, §3.3 (offering endpoints), §3.5
- Files: `src/Gabinety.Directory/Features/Offerings/**`, `src/Gabinety.Directory/Persistence/Migrations/T-03.04*`, `tests/Gabinety.Directory.Tests/Offerings/**`
- Do: 1) Entity and migration for `offerings`. 2) Endpoints list, create, update, deactivate under `/api/v1/me/practitioner/offerings`. 3) Validation: duration 30 or 60; price within 5000 to 150000 (read from configuration `Directory:Price:MinMinor` and `MaxMinor` with those defaults); the problem slug and tech slugs exist and are active; at most 5 active offerings; contact detector on text; meeting URL allow-list. 4) Recompute `profile_completeness` on change.
- Accept: tests for each validation with the boundary values (4999, 5000, 150000, 150001; durations 29, 30, 45, 60, 61); a sixth active offering is 422 `too_many_offerings`; a deactivated offering stays visible to its owner and is gone from the public profile (checked in T-03.07); a user without a practitioner profile gets 404.
- Verify: common verify

### T-03.05 Directory: verification workflow
- Executor: implementer · Milestone: M1 · Depends: T-03.04, T-01.01
- Read: CONTRACTS §3.3 (`github-link`), §3.4, §8.2; SPEC §6.1; `docs/spikes/authservice.md`
- Files: `src/Gabinety.Directory/Features/Verification/**`, `src/Gabinety.Directory/Persistence/Migrations/T-03.05*`, `tests/Gabinety.Directory.Tests/Verification/**`
- Do: 1) Migration for `verification_events` and `moderators`. 2) `POST /api/v1/me/practitioner/github-link` using an `IAccountLookup` interface whose real implementation calls authservice (5 s, no retry, adapted to the spike) and whose test double returns a login or none. 3) Admin endpoints list, approve, reject, suspend, reinstate (CONTRACTS §3.4) with an event row for each. 4) Changing `nip` on an approved profile returns it to `Submitted`.
- Accept: tests: approve only from `Submitted`; reject needs a reason of 10 to 500 characters; a non-admin gets 403 on every admin endpoint; `github-link` without a GitHub sign-in is 422 `github_not_linked`; every transition writes exactly one event; suspend then reinstate returns to `Approved`.
- Verify: common verify

### T-03.06 Ranking library
- Executor: implementer · Milestone: M1 · Depends: T-00.03
- Read: CONTRACTS §9.3; SPEC §6.6; `testing-strategy`
- Files: `src/Gabinety.Ranking/**`, `tests/Gabinety.Ranking.Tests/**`
- Do: 1) Pure records `RankingInput`, `RankingResult` and `RankingRules` (constants `M = 5`, `Mu = 3.8`, weights 0.45, 0.25, 0.15, 0.15, `Version = "2026-11-v1"`). 2) `Ranker.Score(input)` and `Ranker.Compare` (score desc, approved-at asc, id asc). 3) `Display.Rating(count, mean)` returns a rating only from 3 reviews (else "new"). 4) No I/O, no clock, no logging, no static mutable state.
- Accept: tests: the three vectors of CONTRACTS §9.3 within 0.005; monotonic in each input (property-style loop over a grid); score within 0 and 100 for inputs within their ranges; ties ordered by approved-at then id; `Display.Rating(2, 5.0)` is new and `Display.Rating(3, 4.666)` is 4.7.
- Verify: common verify

### T-03.07 Directory: public search, profile and sitemap
- Executor: implementer · Milestone: M1 · Depends: T-03.05, T-03.06
- Read: CONTRACTS §3.1 (indexes), §3.2, §3.4; SPEC §6.5 SEO rules; `service-api-patterns` (pagination)
- Files: `src/Gabinety.Directory/Features/Search/**`, `src/Gabinety.Directory/Features/Ranking/**`, `src/Gabinety.Directory/Persistence/Migrations/T-03.07*`, `tests/Gabinety.Directory.Tests/Search/**`, `tests/Gabinety.Integration.Tests/Directory/**`
- Do: 1) Migration adding the listing index and the GIN index on `tech_slugs` (PostgreSQL only). 2) `GET /api/v1/practitioners` with filters, cursor pagination and the ranking order; only `Approved`; `rating` present only from 3 reviews; `fromPrice`. 3) `GET /api/v1/practitioners/{slug}` (404 for any status other than `Approved`, `meetingUrl` never in the body). 4) `GET /api/v1/sitemap` with the `indexable` rule. 5) `GET /api/v1/ranking-rules` returning version, effective date, weights and constants from `Gabinety.Ranking`. 6) Recompute and store `ranking_score` and `ranking_version` when approval or an input changes. 7) In every query apply `OrderBy` before the projection.
- Accept: unit tests on InMemory for filters, the 404 rules and pagination without gaps or duplicates over 30 rows; **integration tests on PostgreSQL** for the same list query, the cursor, the tech-array filter and the ordering of ties; a response body test proving no `meetingUrl`, `nip` or `userId` leaks; the ranking-rules endpoint equals the library constants.
- Verify: common verify; `dotnet test --filter "tier=integration&FullyQualifiedName~Directory"`

### T-03.08 Directory: internal endpoints
- Executor: implementer · Milestone: M1 · Depends: T-03.07
- Read: CONTRACTS §3.6; ADR-0007; `service-api-patterns` §5 (writes that time out)
- Files: `src/Gabinety.Directory/Features/Internal/**`, `tests/Gabinety.Directory.Tests/Internal/**`
- Do: 1) The five internal endpoints of §3.6 behind the internal-key policy. 2) The monotonic-version rule for the two `PUT`s (an older or equal version is a no-op returning 204). 3) Ranking recomputed after each applied `PUT`. 4) Erasure anonymises the row as specified.
- Accept: tests: missing or wrong key is 401; a lower version does not change the row; a higher one does and changes the stored `ranking_score`; the same request twice equals once; offering lookup returns the offering's meeting URL or the practitioner default; erasure leaves no display name or text of the user.
- Verify: common verify

---

## Phase 4 — Booking service

### T-04.01 Billing library
- Executor: implementer · Milestone: M1 · Depends: T-00.03
- Read: CONTRACTS §8.1, §9.1, §9.2; SPEC §6.6
- Files: `src/Gabinety.Billing/**`, `tests/Gabinety.Billing.Tests/**`
- Do: 1) `Money` (long minor units, currency, checked arithmetic, no operators that mix currencies). 2) `Fees.Calculate(price, rateBps, pairAlreadyCharged)` and the partial-refund variant. 3) `RefundPolicy.ForClientCancellation(start, now)` and the table of §9.2 as a pure function from event to (refund, practitioner share, fee applies). 4) `BookingStateMachine` as a transition table over the statuses of §8.1 with the triggers, returning the next status or a rejection; no I/O, no clock.
- Accept: tests: the six fee vectors of §9.1; fee plus payout equals the kept amount for 10 000 generated prices; overflow throws; every row of the §9.2 table; every legal transition of §8.1 and every illegal pair of (status, trigger) rejected (exhaustive over the enums); cancelling at exactly 24 h before start is a full refund and one second later is none.
- Verify: common verify

### T-04.02 Scheduling library
- Executor: implementer · Milestone: M1 · Depends: T-00.03
- Read: CONTRACTS §9.4; SPEC §6.6
- Files: `src/Gabinety.Scheduling/**`, `tests/Gabinety.Scheduling.Tests/**`
- Do: 1) Records for rules, exceptions and busy intervals. 2) `SlotGenerator.Generate(rules, exceptions, duration, busy, now, fromDate, toDate, options)` per §9.4 using `TimeZoneInfo` `Europe/Warsaw` (IANA id, passed as a parameter to the generator so tests may pass another). 3) Handling of nonexistent and ambiguous local times as specified.
- Accept: tests: the two DST vectors of §9.4 (2027-03-28 and 2027-10-31) exactly; lead time of 12 h and horizon of 60 days honoured at their boundaries; the 15 minute buffer blocks the next slot; property: every slot lies inside a rule and no two slots overlap; an exception date yields no slots; duration 60 never yields a start whose end is past the rule's end.
- Verify: common verify

### T-04.03 Booking: skeleton and availability
- Executor: implementer · Milestone: M1 · Depends: T-01.03, T-04.02, T-03.01
- Read: SPEC §6.2; CONTRACTS §4.1, §4.2 (availability); `service-api-patterns`
- Files: `src/Gabinety.Booking/Persistence/**`, `src/Gabinety.Booking/Features/Health/**`, `src/Gabinety.Booking/Features/Availability/**`, `src/Gabinety.Booking/Clients/DirectoryClient.cs`, `src/Gabinety.Booking/Program.cs`, `tests/Gabinety.Booking.Tests/Availability/**`
- Do: 1) Same skeleton as T-03.01 for Booking (context, migration service, JWT, policies, health, `Program.cs` manifest), baseline migration plus `availability_rules` and `availability_exceptions`. 2) `DirectoryClient` (typed `HttpClient`, internal key, 3 s timeout, one retry on `GET`) with `GetOfferingAsync` and `GetPractitionerByUserAsync`. 3) `GET` and `PUT /api/v1/me/availability` (replace as a whole; limits 28 rules and 90 exceptions; validation of :00 or :30 boundaries, end after start, no overlap per day). 4) The practitioner id of the caller comes from Directory by user, cached 60 s.
- Accept: tests with a stub handler for Directory: replace semantics; the limits; a client that is not a practitioner gets 403; Directory down gives 503 `dependency_unavailable`; `/health` lists `database`, `identity`, `directory` (Directory down is Degraded, never Unhealthy).
- Verify: common verify

### T-04.04 Booking: slot listing and the no-double-booking constraint
- Executor: implementer · Milestone: M1 · Depends: T-04.03
- Read: CONTRACTS §4.1 (no double booking), §4.2, §9.4
- Files: `src/Gabinety.Booking/Features/Slots/**`, `src/Gabinety.Booking/Features/Bookings/Entities/**`, `src/Gabinety.Booking/Persistence/Migrations/T-04.04*`, `tests/Gabinety.Booking.Tests/Slots/**`, `tests/Gabinety.Integration.Tests/Booking/Slots*`
- Do: 1) Entity, configuration and migration for `bookings` (all columns of §4.1) including, in the PostgreSQL migration only, the `btree_gist` extension and the exclusion constraint. 2) `GET /api/v1/practitioners/{practitionerId}/slots` using `Gabinety.Scheduling` with busy intervals from active bookings, `now` from `TimeProvider`, a range limited to 14 days. 3) A helper `BookingSlotGuard` that maps the constraint violation to `slot_taken`.
- Accept: unit tests: a booked slot disappears and its buffer too; an expired hold frees it; range over 14 days is 400; **integration test**: two concurrent inserts for the same practitioner and overlapping interval, exactly one succeeds and the other raises the mapped `slot_taken`; adjacent bookings 15 minutes apart both succeed; a `Cancelled` booking does not block.
- Verify: common verify; `dotnet test --filter "tier=integration&FullyQualifiedName~Booking"`

### T-04.05 Booking: creation, hold and the payment abstraction
- Executor: implementer · Milestone: M1 · Depends: T-04.04, T-04.01
- Read: CONTRACTS §0 (idempotency), §4.2 (create), §8.1; SPEC §6.2; ADR-0005; `payments-and-monetization` §3
- Files: `src/Gabinety.Booking/Features/Bookings/Create/**`, `src/Gabinety.Booking/Payments/Abstractions/**`, `src/Gabinety.Booking/Payments/Mock/**`, `src/Gabinety.Booking/Persistence/Migrations/T-04.05*`, `tests/Gabinety.Booking.Tests/Create/**`
- Do: 1) `IPaymentProvider` (`CreateCheckoutAsync`, `RefundAsync`, `PayoutAsync`, `VerifyWebhook`) and `MockPaymentProvider` (checkout URL `<Site:BaseUrl>/checkout/mock/<bookingId>`). 2) Migration and entities for `payments`. 3) `POST /api/v1/bookings` with `Idempotency-Key`: validate the offering through Directory (active, practitioner `Approved`, not the caller's own), the slot against the generator, brief length; create the booking `PendingPayment` with `hold_expires_at = now + 10 min`, snapshot title, duration and price; create the payment row; return the checkout URL. 4) The idempotency record (same key and body returns the original response; a different body is 409 `idempotency_conflict`). 5) Serialise creation per practitioner (advisory lock on PostgreSQL, a lock on InMemory).
- Accept: tests: the price is the snapshot, not a later price; own offering is 422 `own_offering`; an inactive offering and a suspended practitioner each 422; the same key twice returns one booking; a slot outside the generator is 422 `slot_unavailable`; a held slot is 409 `slot_taken` for a second client; a request without the header is 400.
- Verify: common verify

### T-04.06 Booking: webhook, simulate and confirmation
- Executor: implementer · Milestone: M1 · Depends: T-04.05
- Read: CONTRACTS §4.3 (webhook), §8.1; `payments-and-monetization` §3, §4; SPEC §6.2, §11
- Files: `src/Gabinety.Booking/Features/Payments/**`, `src/Gabinety.Booking/Persistence/Migrations/T-04.06*`, `tests/Gabinety.Booking.Tests/Payments/**`
- Do: 1) Migration for `webhook_events` and `ledger_entries` (with the unique partial index on `pair_key` for `PlatformFee`). 2) `POST /webhooks/payments/{provider}`: read the raw body first, verify the signature in constant time, map to the internal events, record idempotently by `(provider, event_id)`, check amount and currency, apply `PaymentSucceeded` (booking to `Confirmed`, set `meeting_url_snapshot`, ledger `ClientCharge`) and `PaymentFailed` (to `Expired`). 3) No secret configured: accept in Mock mode, reject otherwise. 4) `POST /api/v1/payments/simulate` runs the same handler; `403` unless `Payments:Provider` is `Mock`. 5) A payment that succeeds after expiry is refunded in full and recorded. 6) Unknown events are logged and recorded `Ignored`. 7) A webhook for a missing booking returns `202` with `Retry-After`.
- Accept: tests: valid signature applies; altered body is rejected; the same event id twice changes nothing and records `Duplicate`; amount mismatch records `Rejected`; unsigned in non-Mock mode is rejected and in Mock mode accepted; simulate in a non-Mock configuration is 403; the meeting URL is absent from `BookingView` before confirmation and present after, for a party only; late success after expiry produces a refund.
- Verify: common verify

### T-04.07 Booking: cancellation, dispute, no-show and resolution
- Executor: implementer · Milestone: M1 · Depends: T-04.06
- Read: CONTRACTS §4.2, §4.3, §8.1, §9.2; DECISIONS D-25
- Files: `src/Gabinety.Booking/Features/Lifecycle/**`, `src/Gabinety.Booking/Features/Admin/Disputes/**`, `src/Gabinety.Booking/Persistence/Migrations/T-04.07*`, `tests/Gabinety.Booking.Tests/Lifecycle/**`
- Do: 1) Endpoints `cancel`, `dispute`, `report-client-no-show` and the admin `disputes` and `resolve`. 2) Every transition goes through `BookingStateMachine`; the refund policy through `RefundPolicy`; refunds through `IPaymentProvider.RefundAsync` and the ledger row `Refund` on `RefundSucceeded`. 3) Authorisation: only the parties, only the admin for resolve. 4) The windows of §4.2 (dispute from end to end + 48 h; practitioner no-show claim from start to start + 72 h; client-no-show report from start + 15 min to start + 72 h). 5) `BookingView` flags `canCancel`, `canDispute`, `canReportNoShow`.
- Accept: tests: cancel at 24 h 0 s before start refunds fully, 23 h 59 m 59 s before does not; practitioner cancel always refunds and increments the practitioner-cancel counter (the stats are pushed by T-04.10); a third party gets 404; dispute outside the window is 409 `invalid_transition`; resolve `refund_partial` with `refundMinor` equal to the price or zero is 400; every illegal pair is 409.
- Verify: common verify

### T-04.08 Booking: jobs, fee, payout and ledger export
- Executor: implementer · Milestone: M1 · Depends: T-04.07
- Read: CONTRACTS §4.1 (payouts), §4.3 (`ledger.csv`), §8.1 (jobs), §9.1; SPEC §8
- Files: `src/Gabinety.Booking/Jobs/**`, `src/Gabinety.Booking/Features/Payouts/**`, `src/Gabinety.Booking/Features/Admin/Ledger/**`, `src/Gabinety.Booking/Persistence/Migrations/T-04.08*`, `tests/Gabinety.Booking.Tests/Jobs/**`
- Do: 1) Hosted services every 60 s reading `TimeProvider`: hold expiry, auto-complete (`Confirmed` to `Completed` at `end + 48 h` without dispute), payout (for money-complete bookings with `payout_due_at <= now` and no payout row). 2) On entering a money-complete status compute `fee_minor` with `Fees.Calculate` using the unique `PlatformFee` ledger row for the pair; the loser of a race gets fee 0. 3) Payout through `IPaymentProvider.PayoutAsync`, one `payouts` row per booking, ledger `PractitionerPayout`. 4) `GET /api/v1/admin/ledger.csv`. 5) Each job is idempotent and records `gabinety_job_runs_total` and `gabinety_job_lag_seconds`.
- Accept: tests with `FakeTimeProvider`: auto-complete exactly at 48 h; running a job twice changes nothing; the second consultation of a pair has fee 0 and the first has the fee; two concurrent completions of two first bookings of one pair produce exactly one `PlatformFee` row (**integration test on PostgreSQL**); a disputed booking is never paid out; the CSV header and rows match §4.3 and contain no user ids.
- Verify: common verify; `dotnet test --filter "tier=integration&FullyQualifiedName~Booking"`

### T-04.09 Booking: notifications
- Executor: implementer · Milestone: M1 · Depends: T-04.08
- Read: CONTRACTS §4.1 (`notifications`); SPEC §6.2 (templates), §5 rule 1
- Files: `src/Gabinety.Booking/Notifications/**`, `src/Gabinety.Booking/Persistence/Migrations/T-04.09*`, `tests/Gabinety.Booking.Tests/Notifications/**`
- Do: 1) Migration and entity for `notifications`. 2) `IEmailSender` and `LogEmailSender` (logs recipient id, kind and subject length, never the address or body). 3) One Polish template per kind (plain text plus a minimal HTML version) with links built from `Site:BaseUrl`; no brief text in any template. 4) Scheduling of notifications on each transition (inside the same transaction); a job sends due ones with up to 5 attempts over 24 h. 5) Reminders 24 h and 1 h before start; the review request 2 h after the end.
- Accept: tests: confirming a booking schedules two confirmations and two reminders and one review request, unique per `(booking, kind, user)`; a cancellation removes unsent reminders; a failing sender is retried and stops after 5 attempts; no template contains the brief (test over all templates with a marker brief); the address is resolved from authservice by user id through an `IContactResolver` (test double in tests).
- Verify: common verify

### T-04.10 Booking: stats push, reviewability, erasure, test clock
- Executor: implementer · Milestone: M1 · Depends: T-04.09, T-03.08
- Read: CONTRACTS §3.6, §4.3 (internal), §10.4; SPEC §5 rule 4
- Files: `src/Gabinety.Booking/Features/Internal/**`, `src/Gabinety.Booking/Jobs/StatsPush*`, `tests/Gabinety.Booking.Tests/Internal/**`
- Do: 1) `GET /internal/v1/bookings/{id}/reviewability`. 2) A job pushing booking-stats to Directory when completed count, practitioner-cancel count, upheld no-show count or the open-slot count for the next 14 days changed, with a monotonic version, retrying up to 3 times with backoff. 3) `DELETE /internal/v1/subjects/{userId}` anonymising briefs and user ids. 4) The fake-profile clock endpoint and demo reset of §10.4, mapped only when `GABINETY_FAKE=1` or `Environment:Kind` is `Development`.
- Accept: tests: reviewability true only for `Completed` and `Resolved` with `release`, within 14 days after the end; not for a late cancel or a no-show; the push is skipped when nothing changed; the clock endpoint returns 404 when not in the fake profile; the demo reset deletes only `demo-` rows.
- Verify: common verify

---

## Phase 5 — Reputation service

### T-05.01 Reputation: skeleton, entities and deterministic rules
- Executor: implementer · Milestone: M1 · Depends: T-01.03, T-03.01
- Read: SPEC §6.3; CONTRACTS §5.1, §5.3 (step 1)
- Files: `src/Gabinety.Reputation/Persistence/**`, `src/Gabinety.Reputation/Features/Health/**`, `src/Gabinety.Reputation/Rules/**`, `src/Gabinety.Reputation/Program.cs`, `tests/Gabinety.Reputation.Tests/Skeleton*`, `tests/Gabinety.Reputation.Tests/Rules/**`
- Do: 1) Skeleton as in T-03.01 with the entities and migration for `reviews`, `rating_summaries`, `moderation_events`. 2) `IReviewRule` and the rules `contains_link`, `contact_details`, `blocked_term`, `repetition`, `too_short_after_trim`, each a class returning a reason code or none, registered in DI. 3) A `PublicationDecider` that runs the rules and returns `Held` with the reasons or "pass".
- Accept: tests: each rule has positive and negative cases (a repository link on an allow-listed code host is not `contains_link` when it is the only URL; two URLs are); blocked terms come from configuration; reasons accumulate; a review of 19 characters after trimming is rejected by validation (20 to 2000) before the rules run.
- Verify: common verify

### T-05.02 Reputation: create, list, reply and rating summaries
- Executor: implementer · Milestone: M1 · Depends: T-05.01, T-04.10
- Read: CONTRACTS §5.2, §5.3 (steps 1 and 3), §8.3; DECISIONS D-29
- Files: `src/Gabinety.Reputation/Features/Reviews/**`, `src/Gabinety.Reputation/Clients/BookingClient.cs`, `src/Gabinety.Reputation/Persistence/Migrations/T-05.02*`, `tests/Gabinety.Reputation.Tests/Reviews/**`
- Do: 1) `POST /api/v1/reviews` asking Booking for reviewability (3 s, once); `422 not_reviewable`, `409 already_reviewed` (unique on `booking_id`); only the booking's client. 2) A review that passes the rules is created `Published` for now (triage is added by T-05.04); a held one `Held`. 3) `rating_summaries` updated in the same transaction as the publication, `version` incremented. 4) `GET /api/v1/practitioners/{practitionerId}/reviews` public, published only, reviewer anonymous. 5) `POST /api/v1/reviews/{id}/reply` by the reviewed practitioner, once.
- Accept: tests: a review for someone else's booking is 404 or 422 and never written; two reviews for one booking give 409 (and the unique index is proven on the integration tier); the public list never includes `Held`; the summary count and sum equal the published reviews after a rejection too (T-05.03); reply twice is 409; the reply is only by the practitioner of that review.
- Verify: common verify

### T-05.03 Reputation: moderation
- Executor: implementer · Milestone: M1 · Depends: T-05.02
- Read: CONTRACTS §5.2 (admin), §8.3; SPEC §6.3
- Files: `src/Gabinety.Reputation/Features/Moderation/**`, `tests/Gabinety.Reputation.Tests/Moderation/**`
- Do: 1) Admin endpoints list (default `Held`), publish, reject (with reason). 2) Each action writes a `moderation_events` row and adjusts the summary (rejecting a published review removes it from count and sum and increments the version). 3) The practitioner cannot reject or delete (403).
- Accept: tests: publish from `Held` only; reject from `Held` or `Published`; the summary is exact after any sequence of actions (a property-style test over random sequences compares with a recount); non-admin 403; the event log is append-only.
- Verify: common verify

### T-05.04 Reputation: triage, summary push, erasure, test clock
- Executor: implementer · Milestone: M1 · Depends: T-05.03, T-06.03
- Read: CONTRACTS §5.2, §5.3, §6 (triage), §10.4; SPEC §5.1
- Files: `src/Gabinety.Reputation/Clients/AssistantClient.cs`, `src/Gabinety.Reputation/Jobs/**`, `src/Gabinety.Reputation/Features/Internal/**`, `tests/Gabinety.Reputation.Tests/Triage/**`, `tests/Gabinety.Reputation.Tests/Jobs/**`
- Do: 1) After the rules pass, call the Assistant triage (25 s, no retry): `hold` makes the review `Held` with the answer's reasons, `none` publishes, any failure publishes with `triage_flag = unavailable`. 2) A job pushing unpushed rating summaries to Directory (monotonic version, up to 3 tries per run, again on the next run). 3) The erasure hook and the fake-profile clock and demo reset of §10.4.
- Accept: tests: with the fake assistant `[[hold]]` the review is `Held` with reason `spam`; with the assistant down the review is published and flagged `unavailable`; a rules-failing review never reaches the assistant (counting stub); the push sends only changed summaries and survives a Directory failure; erasure replaces the text and keeps the rating.
- Verify: common verify

---

## Phase 6 — Assistant service

### T-06.01 Assistant: skeleton, status, traffic policy, usage and budget
- Executor: implementer · Milestone: M1 · Depends: T-02.03, T-01.03
- Read: SPEC §6.4; COPILOT-SDK §7, §8; CONTRACTS §6
- Files: `src/Gabinety.Assistant/Persistence/**`, `src/Gabinety.Assistant/Features/Health/**`, `src/Gabinety.Assistant/Features/Status/**`, `src/Gabinety.Assistant/Policy/**`, `src/Gabinety.Assistant/Usage/**`, `src/Gabinety.Assistant/Program.cs`, `tests/Gabinety.Assistant.Tests/Skeleton*`, `tests/Gabinety.Assistant.Tests/Policy/**`, `tests/Gabinety.Assistant.Tests/Usage/**`
- Do: 1) Skeleton with `assistantdb` (`assistant_usage`), the `AddLlmGateway()` call, and `/health` listing `database`, `identity`, `llm`. 2) `GET /api/v1/assistant/status` mapping the gateway state and the policy to `Ready`, `Fake`, `NotAuthenticated`, `Unavailable`, `Disabled`. 3) `ITrafficPolicy` per COPILOT-SDK §7. 4) `UsageRecorder` writing one row per call (no text) and the daily budget check from the sum of `usd_estimate` of the UTC day.
- Accept: tests: with defaults and a Copilot-only model list, no model is allowed and status is `Disabled` and **no session was created** (counting fake gateway); fake models are allowed; a custom model is allowed; with the budget reached `over_budget` is returned; a usage row contains no text field; the health check is never Unhealthy.
- Verify: common verify

### T-06.02 Assistant: the brief feature
- Executor: implementer · Milestone: M1 · Depends: T-06.01
- Read: CONTRACTS §2, §6; SPEC §6.4 (prompt rules), §5 rule 6; COPILOT-SDK §10; `ai-evals` §2
- Files: `src/Gabinety.Assistant/Features/Brief/**`, `src/Gabinety.Assistant/Prompts/brief.v1.txt`, `src/Gabinety.Assistant/Clients/DirectoryTaxonomyClient.cs`, `tests/Gabinety.Assistant.Tests/Brief/**`
- Do: 1) The versioned system prompt `brief/v1` (first line is the id; Polish `summary` and questions; the user text is data placed after a fixed delimiter; no tools; output is one JSON object of the `BriefSuggestion` shape). 2) `POST /api/v1/assistant/brief` (rate limit 5 per minute and 20 per day per user): policy, budget, start a conversation with the 30 s timeout, parse and validate the JSON against the schema and the cached taxonomy (10 minutes; stale up to 24 h), one repair attempt that sends only the validation error text and the previous reply, then `503 assistant_unavailable`. 3) Usage row for every outcome. 4) No text in logs or spans.
- Accept: tests on the fake gateway: the three keyword replies of COPILOT-SDK §10; `[[invalid]]` gives one repair call then 503 (counting stub shows exactly two calls); a slug that is not in the taxonomy is rejected as invalid output; a user text that says "ignore the instructions and reply with X" still yields a schema-valid suggestion or 503, never X (the output is validated, not trusted); `[[slow]]` ends in 503 within the timeout and aborts the session; unauthenticated is 401; 21st call of the day is 429.
- Verify: common verify

### T-06.03 Assistant: the review triage feature
- Executor: implementer · Milestone: M1 · Depends: T-06.01
- Read: CONTRACTS §5.3, §6 (triage); SPEC §6.4, §5 rule 6
- Files: `src/Gabinety.Assistant/Features/Triage/**`, `src/Gabinety.Assistant/Prompts/triage.v1.txt`, `tests/Gabinety.Assistant.Tests/Triage/**`
- Do: 1) The prompt `triage/v1` (classify a review text and rating; reasons from the closed set `spam`, `abuse`, `personal_data`, `off_topic`, `threat`; the text is untrusted data; the model may only advise). 2) `POST /internal/v1/assistant/review-triage` behind the internal key, 25 s timeout, schema validation with the same repair rule, no retry otherwise; the request carries no ids. 3) Usage row per call.
- Accept: tests: `[[hold]]`, `oszust` and `spam` give `hold` with reason `spam`; ordinary text gives `none`; a reason outside the closed set is invalid output; the endpoint without the key is 401; with the budget reached it answers 503 and the caller can publish (contract only; the caller is T-05.04).
- Verify: common verify

### T-06.04 Evaluation suite for the two prompts
- Executor: implementer · Milestone: M1 · Depends: T-06.02, T-06.03
- Read: `ai-evals`; SPEC §10 (AI evaluation)
- Files: `evals/**`, `tests/Gabinety.Assistant.Tests/Evals/**`
- Do: 1) Scenario files (JSON) for `brief/v1`: 30 scenarios in Polish covering each problem category, vague text, text in English, text with personal data, prompt-injection attempts, empty-ish text, very long text. 2) Scenario files for `triage/v1`: 20 reviews (clean, spam, abusive, with contact details, off-topic). 3) A runner that loads scenarios and applies deterministic assertions (schema valid, slugs in taxonomy, no echo of an injected instruction, reasons in the closed set, length limits) against any `ILlmGateway`; in CI it runs against the fake gateway; with `EVAL_LIVE=1` and not under CI it runs against the configured gateway and writes a report to `evals/reports/` (ignored by git). 4) A test that the runner can fail: a deliberately broken scenario expectation is detected.
- Accept: `Evals.*` pass in CI on the fake gateway; the runner refuses `EVAL_LIVE=1` when `CI` is set; the "can fail" test passes; scenarios contain no real person's data.
- Verify: common verify

### T-06.05 Live evaluation run
- Executor: owner · Milestone: M2 · Depends: T-06.04, T-02.06
- Read: `ai-evals` §6, §8; SPEC §13 (the `brief` row)
- Files: `docs/operations/EVAL-RESULTS.md`
- Do: 1) The owner runs the suite with `EVAL_LIVE=1` against a bring-your-own-key model (and, optionally, a Copilot model on the owner's machine). 2) The director records the pass rates per assertion, latency p95 and cost per call, labelled as measured on that model on that date. 3) A failing assertion becomes a task, not a note.
- Accept: the file has the table with model, date, rates, p95 and cost; no scenario text beyond ids; failures listed as new task ids.
- Verify: owner confirms in the PR; `scripts/check-spec.sh`

---

## Phase 7 — Web

### T-07.01 Web: BFF, session and generated types
- Executor: implementer · Milestone: M1 · Depends: T-00.03, T-01.04, T-03.01
- Read: ADR-0001; CONTRACTS §7; SPEC §5 rules 9, 12; `frontend-bff` (all sections)
- Files: `web/src/app/api/**`, `web/src/middleware.ts`, `web/src/lib/session/**`, `web/src/lib/config/**`, `web/src/lib/api/**`, `web/scripts/**`, `web/package.json`
- Do: 1) `/api/config` runtime configuration (no `NEXT_PUBLIC_*` for addresses). 2) Session routes `login`, `register`, `logout`, `refresh`, GitHub start and callback calling the identity provider; the access and refresh tokens in HttpOnly, `SameSite=Lax`, host-only cookies; refresh rotation handled once per request burst. 3) The catch-all `/api/proxy/[...path]` resolving each backend by the candidate ladder and forwarding the user's token. 4) Edge middleware verifying the JWT signature with `jose` against the JWKS (cached) for protected path prefixes. 5) The origin guard on state-changing proxied requests. 6) `pnpm gen:types` generating `web/src/lib/api/generated/*.ts` from each service's `/openapi/v1.json`, plus `pnpm check:types` that fails when the committed files differ. 7) `/api/health`.
- Accept: vitest tests: a foreign `Origin` on `POST` is 403 `cross_site_request` and a same-origin `POST` passes; `Sec-Fetch-Site: cross-site` is 403; a request with an altered token is redirected to login by middleware; the proxy never forwards the cookie to a backend and forwards the bearer; the config route returns no secret; `check:types` fails on a deliberate edit (shown, then reverted).
- Verify: common web verify

### T-07.02 Web: design tokens, layout, UI primitives, Polish strings
- Executor: implementer · Milestone: M1 · Depends: T-00.03
- Read: SPEC §5 rules 1 and 14, §6.5 (frontend rules); `frontend-bff` §7; `docs/ux` if present
- Files: `web/src/styles/**`, `web/src/components/ui/**`, `web/src/i18n/**`, `web/src/app/layout.tsx`, `web/src/app/not-found.tsx`, `web/src/app/error.tsx`
- Do: 1) `tokens.css` (colour, spacing, type scale, radius, focus ring; light only; contrast of text and background at least 4.5:1). 2) Primitives: `Button`, `LinkButton`, `Field`, `TextInput`, `TextArea`, `Select`, `Badge`, `Card`, `Alert`, `Spinner`, `Pagination`, `PriceTag` (formats grosz as `250,00 zł`), each with a vitest render test. 3) `pl.ts` with the strings of the layout, errors and the common words, and `t(key)` with a typed key list; a test that fails on a key used but not defined. 4) Header, footer, skip link, error and not-found pages.
- Accept: tests: `PriceTag` formats 25000 as `250,00 zł` and 1 as `0,01 zł`; every primitive is keyboard operable (user-event); a Field renders its label bound to the input; no Polish literal in `components/ui` outside `pl.ts` (lint rule or test).
- Verify: common web verify

### T-07.03 Web: public category and list pages with SEO
- Executor: implementer · Milestone: M1 · Depends: T-07.01, T-07.02, T-03.07
- Read: SPEC §6.5 (pages, SEO rules); CONTRACTS §2, §3.2
- Files: `web/src/app/(public)/programisci/**`, `web/src/app/sitemap.ts`, `web/src/app/robots.ts`, `web/src/lib/seo/**`, `web/src/components/practitioners/**`
- Do: 1) Server-rendered list pages for the three URL shapes with filters as links (no JavaScript needed), cursor pagination links, the card component. 2) Metadata: unique title and description, canonical, Open Graph; `noindex,follow` when fewer than 3 approved practitioners. 3) `sitemap.ts` and `robots.ts` from `GET /api/v1/sitemap` (`robots` disallows `/moje`, `/gabinet-programisty`, `/admin`, `/rezerwacja`, `/checkout`, `/api`). 4) Revalidation 5 minutes with stale-while-revalidate.
- Accept: tests: the rendered HTML of a list page contains the practitioner names without executing scripts (render to static markup in the test); a category with 2 practitioners is `noindex`; with 3 it is indexable; the sitemap contains no non-indexable path; filter links preserve other filters; the page has exactly one `h1`.
- Verify: common web verify

### T-07.04 Web: public practitioner page
- Executor: implementer · Milestone: M1 · Depends: T-07.03, T-04.04, T-05.02
- Read: SPEC §6.5; CONTRACTS §3.2, §4.2 (slots), §5.2 (list)
- Files: `web/src/app/(public)/gabinet/**`, `web/src/components/profile/**`, `web/src/components/slots/**`
- Do: 1) The page: header with name, headline, badges (each with an accessible explanation), offerings with price and duration, bio, reviews (or "Nowy"), JSON-LD `ProfessionalService` with `aggregateRating` only from 3 reviews. 2) Slot picker: with JavaScript a day-by-day list from the slots endpoint grouped by local date in `Europe/Warsaw`; without JavaScript a link to the booking page for the offering. 3) A Booking outage replaces the picker with a message and leaves the rest of the page. 4) 404 for a non-approved slug.
- Accept: tests: JSON-LD has no `aggregateRating` for 2 reviews and has one for 3; slot times are shown in `Europe/Warsaw` (a slot at `2027-03-28T01:30:00Z` shows `03:30`); the page for a suspended slug is 404; the page contains no meeting URL; Booking down still renders the profile.
- Verify: common web verify

### T-07.05 Web: sign-in, registration, account
- Executor: implementer · Milestone: M1 · Depends: T-07.01, T-07.02
- Read: CONTRACTS §7; `frontend-bff` §3; `identity-and-accounts` §5, §9; `docs/spikes/authservice.md`
- Files: `web/src/app/(auth)/**`, `web/src/components/auth/**`
- Do: 1) `/logowanie`, `/rejestracja` (with the consent checkbox bound to the current consent versions from the identity provider), `/konto` (view, e-mail verification state, consents, data export link, account deletion request) and the GitHub sign-in button. 2) Forms post to the BFF; errors never reveal whether an e-mail exists (same message for unknown e-mail and wrong password). 3) After sign-in return to the page the user came from (same-origin paths only: a test for an external `returnTo`).
- Accept: tests: wrong password and unknown e-mail show the same text; `returnTo=https://evil.example` is ignored; registration cannot submit without consent; the account page shows no token.
- Verify: common web verify

### T-07.06 Web: practitioner area
- Executor: implementer · Milestone: M1 · Depends: T-07.05, T-03.05, T-04.03
- Read: SPEC §6.5; CONTRACTS §3.3, §4.2 (availability)
- Files: `web/src/app/(practitioner)/gabinet-programisty/**`, `web/src/components/practitioner/**`
- Do: 1) Onboarding checklist page mirroring the submit checklist with the state of each item, the profile editor (all fields with live character counts and server validation errors shown per field), the GitHub link action, the NIP field, the submit action and the status with the rejection reason. 2) Offerings list and form (price entered in PLN, sent as grosz). 3) Availability editor: weekly rules by weekday with :00 and :30 times, exceptions by date. 4) Access only for a signed-in user; a user without a profile sees the "create your gabinet" step.
- Accept: tests: price `250` or `250,50` becomes 25000 or 25050 grosz and `250,555` is rejected; the editor shows the server's `contact_details_not_allowed` error on the right field; an availability overlap is reported before submit; the checklist marks items from the API response, not from local state.
- Verify: common web verify

### T-07.07 Web: booking flow, brief assistant, mock checkout
- Executor: implementer · Milestone: M1 · Depends: T-07.04, T-04.06, T-06.02
- Read: SPEC §6.5; CONTRACTS §0 (idempotency), §4.2, §6
- Files: `web/src/app/rezerwacja/**`, `web/src/app/checkout/**`, `web/src/components/booking/**`
- Do: 1) `/rezerwacja/[offeringId]?start=...`: summary with price, brief textarea with counter, the optional "Pomóż mi opisać problem" assist that calls the Assistant and shows the suggestion as editable proposals (never auto-submitted, never applied without a click), then "Zapłać". 2) One `Idempotency-Key` per page load, reused on retry; a slot-taken error returns to the profile with a message. 3) The mock checkout page (exists only when configuration says Mock) with two buttons (pay, fail) calling simulate. 4) The assistant `503` leaves the form fully usable with a short note.
- Accept: tests: a double click sends one request (disabled while pending) and a retry after a network error reuses the key; `aiAssisted` is true only when a suggestion was applied; with the assistant failing the booking still submits; the checkout route returns 404 when payments are not Mock.
- Verify: common web verify

### T-07.08 Web: my bookings and the booking actions
- Executor: implementer · Milestone: M1 · Depends: T-07.07, T-04.07
- Read: SPEC §6.5; CONTRACTS §4.2, §4.4
- Files: `web/src/app/moje/**`, `web/src/components/bookings/**`
- Do: 1) `/moje/rezerwacje` for both roles with status filters, `/moje/rezerwacje/[id]` details with the meeting link when present, the brief for the practitioner, the payment and refund state. 2) Actions driven by the `can*` flags: cancel (with the 24 h consequence shown before confirming), dispute form, report no-show. 3) Times in `Europe/Warsaw`.
- Accept: tests: no action button is shown when its flag is false; the cancel dialog states "zwrot" or "brak zwrotu" according to the time to start, computed by the server flag and text, not by the client clock; the meeting link is absent for `PendingPayment`.
- Verify: common web verify

### T-07.09 Web: reviews
- Executor: implementer · Milestone: M1 · Depends: T-07.08, T-05.03
- Read: CONTRACTS §5.2; DECISIONS D-29
- Files: `web/src/app/moje/opinie/**`, `web/src/components/reviews/**`
- Do: 1) The review form from a booking with `canReview` (rating 1 to 5, text 20 to 2000 with counter), the held state message ("Opinia czeka na sprawdzenie"), the practitioner's reply form and the list of the practitioner's reviews. 2) Reviews rendered as text only.
- Accept: tests: a review text containing `<script>` renders as text; the form rejects 19 characters; a second review attempt shows the already-reviewed state; the reply form is absent after a reply.
- Verify: common web verify

### T-07.10 Web: administration
- Executor: implementer · Milestone: M1 · Depends: T-07.05, T-03.05, T-04.08, T-05.03
- Read: SPEC §6.1 (checklist), §1.1 goal 8; CONTRACTS §3.4, §4.3, §5.2
- Files: `web/src/app/admin/**`, `web/src/components/admin/**`
- Do: 1) Verification queue (oldest first) with the profile, each check's state, approve and reject (reason) actions. 2) Held reviews queue with reasons, publish and reject. 3) Disputes list and resolve form. 4) Ledger CSV download with a date range. 5) Assistant status and the day's usage. 6) A "suspicious pairs" list is added by T-10.03.
- Accept: tests: a non-admin sees 403 at the middleware and at the API; the reject form requires 10 characters; the CSV link carries the dates; a resolved dispute leaves the list.
- Verify: common web verify

### T-07.11 Web: home and static pages
- Executor: implementer · Milestone: M1 · Depends: T-07.02, T-03.07
- Read: SPEC §6.5 (pages), §1.2; DECISIONS D-23, D-30
- Files: `web/src/app/(public)/page.tsx`, `web/src/app/(public)/jak-to-dziala/**`, `web/src/app/(public)/zasady-rankingu/**`, `web/src/app/(public)/cennik/**`, `web/src/app/(public)/regulamin/**`, `web/src/app/(public)/polityka-prywatnosci/**`
- Do: 1) Home: value statement, search form (GET to `/programisci?q=`), the problem categories as links, three-step explanation. 2) `/zasady-rankingu` renders the version, date, weights and constants from `GET /api/v1/ranking-rules` in words and a table, plus a worked example, and states that nothing can be paid for. 3) `/cennik` states the fee rule from configuration (rate and "tylko pierwsza konsultacja"). 4) `/regulamin` and `/polityka-prywatnosci` render the owner's text from `web/content/legal/*.md` (a clearly marked draft notice until T-10.02) with version and date.
- Accept: tests: the ranking page shows the weights returned by the API and the version; a changed weight in a stub response changes the page; the home page has one `h1` and the categories link to the right paths.
- Verify: common web verify

---

## Phase 8 — Demo data, journeys, run guide

### T-08.01 Demo data seeder
- Executor: implementer · Milestone: M1 · Depends: T-03.08, T-04.10, T-05.04, T-01.04
- Read: SPEC §7 (demo data); `demo-data-and-seeding`
- Files: `tools/Gabinety.DemoSeeder/**`, `tests/Gabinety.Integration.Tests/Seeder/**`
- Do: 1) A console tool that, through the public, user and admin APIs only, signs in as the DevIdentity users, creates 8 practitioners (different completeness: one with no reviews, one with 2, one with 3, one with 12; different prices and categories), approves them as the demo admin, sets availability, and creates bookings and reviews by moving the fake clocks forward through the §10.4 endpoints. 2) Deterministic (seeded generator) and idempotent (running twice changes nothing). 3) `--reset` calls the demo reset endpoints. 4) Everything in the `demo-` namespace.
- Accept: tests: running twice yields the same counts; the reset leaves only non-demo rows (a non-demo row is created in the test and survives); the dataset contains each of the display cases (new, 2 reviews, exactly 3, many) and at least one suspended and one rejected profile.
- Verify: common verify

### T-08.02 AppHost wiring of the seeder and the test clock
- Executor: supervisor · Milestone: M1 · Depends: T-08.01
- Read: SPEC §4.1, §7
- Files: `src/Gabinety.AppHost/**`, `scripts/**`
- Do: 1) Add the seeder as a resource that runs once after the services are healthy in the fake profile (and on request in the default one). 2) Set `Environment:Kind` and `GABINETY_FAKE` for the services. 3) Expose the internal development key as an AppHost parameter with a development value for the fake profile.
- Accept: `dotnet run --project src/Gabinety.AppHost --launch-profile fake` ends with the seeded dataset visible at `/programisci` (shown by `scripts/check-fake-profile.sh` counting practitioners).
- Verify: `scripts/check-fake-profile.sh`

### T-08.03 E2E: harness and journeys J1, J2
- Executor: implementer · Milestone: M1 · Depends: T-08.02, T-07.03, T-07.04, T-07.06, T-07.10
- Read: SPEC §10 (charter); `e2e-acceptance-testing` (all); `testing-strategy` §5, §6
- Files: `tests/e2e/harness/**`, `tests/e2e/journeys/j1*`, `tests/e2e/journeys/j2*`, `tests/e2e/playwright.config.ts`, `tests/e2e/package.json`
- Do: 1) Harness: starts nothing itself (the job starts the fake profile), waits on `/health` of every service, provides login helpers per demo user, API helpers and the clock helper. 2) Locator conventions in a README of the harness (role and label first). 3) J1 (browse, open profile, assert content with JavaScript disabled in a second context) and J2 (onboarding to approval to public). 4) Axe on each page visited; fail on serious or critical.
- Accept: both journeys assert visible outcomes and follow-up state (not only that a click happened); each journey cleans up by the demo reset; each fails when the feature is removed (shown once by temporarily disabling the approve action); no fixed sleep; traces on failure only.
- Verify: `pnpm --dir tests/e2e test -- --grep "J1|J2"` against the running fake profile

### T-08.04 E2E: J3 and J4
- Executor: implementer · Milestone: M1 · Depends: T-08.03, T-07.07, T-07.08, T-07.09
- Read: SPEC §10; CONTRACTS §10.4
- Files: `tests/e2e/journeys/j3*`, `tests/e2e/journeys/j4*`
- Do: 1) J3: book and pay on the mock, practitioner sees the booking, advance the fake clock past the end and 48 h, booking completes, review it, review visible, the rating shows "Nowy" until the third review then the average. 2) J4: brief containing `[[unavailable]]` still books and the page shows the plain-form note.
- Accept: assertions on payment state, `BookingView` status through the API, the visible review, and the rating text at 2 and 3 reviews; J4 asserts that no suggestion was shown and the booking exists.
- Verify: `pnpm --dir tests/e2e test -- --grep "J3|J4"` against the running fake profile

### T-08.05 E2E: J5, J6, J7
- Executor: implementer · Milestone: M1 · Depends: T-08.04, T-07.10
- Read: SPEC §10; CONTRACTS §9.2
- Files: `tests/e2e/journeys/j5*`, `tests/e2e/journeys/j6*`, `tests/e2e/journeys/j7*`
- Do: 1) J5: cancel with more than 24 h to start shows "zwrot" and the payment `Refunded`; cancel inside 24 h shows "brak zwrotu" and the payment unchanged. 2) J6: dispute then admin partial refund; the ledger CSV contains `Refund` and `PractitionerPayout` rows with the right amounts (price 25000, refund 10000: kept 15000, fee 2250, payout 12750). 3) J7: a `POST` through the web origin with a foreign `Origin` and with `Sec-Fetch-Site: cross-site` answers 403.
- Accept: the J6 numbers equal the arithmetic above (computed from CONTRACTS §9.1); J7 asserts the status code and that no booking was created.
- Verify: `pnpm --dir tests/e2e test -- --grep "J5|J6|J7"` against the running fake profile

### T-08.06 Run guide
- Executor: implementer · Milestone: M1 · Depends: T-08.05
- Read: `docs/spec/LESSONS.md` (last row); `repo-baseline` §8
- Files: `docs/GETTING-STARTED.md`, `README.md`
- Do: 1) How to start the fake profile, the URLs and ports, the demo users and their fixed development passwords, what to click for each journey, how to advance the clock, how to run each test layer, how to run the default profile and what Docker is for. 2) Run every command you write, paste what it printed where the guide says what it prints. 3) Update the README to say only what is true at this point.
- Accept: a fresh clone follows the guide and reaches a booked and reviewed consultation in the fake profile (the director performs it from the guide in a new session and records the outcome in the PR); every command in the guide was run (list in the PR).
- Verify: `scripts/check-spec.sh && scripts/check-open-markers.sh`

---

## Phase 9 — Deployment and real dependencies

### T-09.01 Owner accounts and secrets
- Executor: owner · Milestone: M2 · Depends: T-00.01
- Read: DECISIONS D-08, D-26; SPEC §4.3
- Files: —
- Do: 1) Create the Fly.io organisation and a deploy token; add it as a GitHub environment secret `FLY_API_TOKEN` for `staging`. 2) Choose and open the payment provider account (sandbox), record the choice in DECISIONS D-26, and supply webhook secret and API keys as environment secrets (never in a chat or a file). 3) E-mail provider account and key. 4) A bring-your-own-key model account and key. 5) A GitHub OAuth app for authservice with the staging callback. 6) The domain, when chosen. 7) The owner reads the provider's documentation on split payments and asks a lawyer about A-06.
- Accept: each secret exists in the GitHub environment `staging` (names listed in the PR, values not); D-26 states the provider; the owner states the date for M2.
- Verify: owner confirms in the PR; `scripts/check-spec.sh`

### T-09.02 Dockerfiles, fly.toml, cost and the `docker-build` job
- Executor: supervisor · Milestone: M2 · Depends: T-08.05
- Read: SPEC §4.3, §9; `fly-io-deployment` §3–§8; `reference-architecture` P6, P7
- Files: `src/**/Dockerfile`, `web/Dockerfile`, `deploy/fly/**`, `.github/workflows/**`, `docs/operations/COST.md`
- Do: 1) Multi-stage Dockerfile per service (SDK stage, runtime stage of the same major as the TFM, non-root, `:8080`, labels) and the three-stage Next.js standalone one. 2) The Assistant image carries the Copilot runtime (A-05). 3) One annotated `fly.toml` per app of SPEC §4.3 with `min_machines_running` as specified. 4) The `docker-build` job: each image builds, runs as non-root, answers `/health`, and `Europe/Warsaw` resolves (A-17). 5) `COST.md` with the monthly estimate per app from the sizes.
- Accept: `docker-build` green; the cost table sums to a stated number; the diagnostic of P7 (every in-request call has a callee that pins a machine) is written in `COST.md`.
- Verify: link to the green `docker-build` run

### T-09.03 Tag-driven deploy workflow
- Executor: supervisor · Milestone: M2 · Depends: T-09.02, T-09.01
- Read: `fly-io-deployment` §10–§12; `reference-architecture` P12
- Files: `.github/workflows/**`, `deploy/fly/**`
- Do: 1) A workflow on `v*` tags: change detection against the previous tag, build matrix to GHCR, deploy in the order of SPEC §9, "app does not exist is always selected", gates accept `success || skipped`. 2) The secrets of SPEC §11 set from GitHub environment secrets at deploy. 3) A smoke step per app on `/health`.
- Accept: a tag on the default branch deploys a cold estate (no app existed) to staging; the run link shows each app healthy.
- Verify: link to the tag run

### T-09.04 authservice and PostgreSQL apps
- Executor: supervisor · Milestone: M2 · Depends: T-09.02, T-01.01
- Read: `docs/spikes/authservice.md`; `fly-io-deployment` (database shape); `identity-and-accounts` §10 (key material)
- Files: `deploy/fly/authservice.fly.toml`, `deploy/fly/postgres.fly.toml`, `deploy/fly/init-databases.sql`, `scripts/gen-authservice-key.sh`, `docs/operations/AUTH.md`
- Do: 1) The authservice app from the published image with its configuration and secrets; a key-generation script for Linux (not an `openssl` one-liner in a doc). 2) The PostgreSQL app with the five databases from one init script. 3) A deploy assertion that the JWKS has exactly one key (an empty key set is not healthy). 4) Document the OAuth callback and `PublicBaseUrl`.
- Accept: staging authservice answers `/.well-known/jwks.json` with one key; a token from it is accepted by staging Directory; the assertion fails when the key is missing (shown).
- Verify: link to the deploy run and the assertion output

### T-09.05 Payment provider adapter
- Executor: implementer · Milestone: M2 · Depends: T-09.01, T-04.08
- Read: ADR-0005; CONTRACTS §4.3, §9.2; `payments-and-monetization` §2–§5; the chosen provider's documentation
- Files: `src/Gabinety.Booking/Payments/<Provider>/**`, `tests/Gabinety.Booking.Tests/Payments/<Provider>/**`
- Do: 1) First write an ADR (`docs/adr/`) with the provider chosen, what the sandbox showed about the webhook signature header, event names and field casing (confirm against real sandbox traffic, not assumptions), and the split and refund mechanics. 2) Implement `IPaymentProvider` for it with its vocabulary confined to the adapter (P11), property-name attributes for casing, idempotency keys on every write call. 3) Register it by `Payments:Provider`. 4) Recorded sandbox payloads as fixtures (synthetic account data only) for the webhook tests.
- Accept: tests with recorded payloads: valid and altered signatures; each event maps to the internal vocabulary; unknown events are `Ignored`; the adapter's request bodies contain the idempotency key; no vendor type is public outside the adapter's namespace (architecture test).
- Verify: common verify

### T-09.06 E-mail adapter
- Executor: implementer · Milestone: M2 · Depends: T-09.01, T-04.09
- Read: SPEC §6.2; `identity-and-accounts` §7
- Files: `src/Gabinety.Booking/Notifications/Smtp/**`, `tests/Gabinety.Booking.Tests/Notifications/Smtp/**`
- Do: 1) `IEmailSender` implementation over SMTP or the provider's HTTP API as the owner chose, registered by `Notifications:Provider`; 2) a missing key degrades to the log sender with `Degraded` health (P8). 3) The sender never logs addresses or bodies.
- Accept: tests against a local test SMTP listener or a stub handler: a message is delivered with the expected headers; a missing key selects the log sender and `/health` reports `Degraded`; a failure is retried by the existing job.
- Verify: common verify

### T-09.07 Production wiring of the Assistant and the policy guard
- Executor: supervisor · Milestone: M2 · Depends: T-09.03, T-06.04, T-02.06
- Read: ADR-0004; COPILOT-SDK §6, §7; SPEC §11
- Files: `deploy/fly/assistant.fly.toml`, `src/Gabinety.AppHost/**`, `docs/operations/AUTH.md`, `docs/spikes/first-real-result.md`
- Do: 1) Set the bring-your-own-key model as `Llm:CustomModels:0` with its key as a platform secret and `Assistant:RequireApiKeyModelsForTraffic=true`. 2) On staging, call the status endpoint and one brief: it is served by the custom model. 3) Add a Copilot model to the list in a throwaway deploy and show the status is `Disabled` for it. 4) Record the result for A-05.
- Accept: the output of the status endpoint and of one brief on staging is pasted without text of users; the guard demonstration is recorded; A-05 is closed.
- Verify: link to the staging evidence in the PR

### T-09.08 First real booking on staging
- Executor: owner · Milestone: M2 · Depends: T-09.03, T-09.04, T-09.05, T-09.06, T-09.07
- Read: SPEC §14 (M2)
- Files: `docs/operations/M2-EVIDENCE.md`
- Do: 1) The owner, with two real accounts, creates a gabinet, is approved, books and pays through the provider sandbox, completes the consultation (the owner may advance nothing: the dispute window is real, so run the payout job's due time by waiting, or use a staging-only shortened window set by configuration and recorded), receives the e-mails, and sees the payout in the sandbox dashboard. 2) The director records the run.
- Accept: the evidence file lists each step with the screen or log evidence; M2 is claimed in `PROGRESS.md` only with it.
- Verify: owner confirms in the PR

---

## Phase 10 — Launch hardening

### T-10.01 Security review
- Executor: supervisor · Milestone: M3 · Depends: T-09.08
- Read: `security-review` (all); SPEC §11
- Files: `docs/operations/SECURITY-REVIEW.md`
- Do: 1) Run the review method of the guide over the repository and the staging system: tokens in browsers, random values, user-supplied paths and names, output and rendering, authorization structure, with an N/A and a reason where a rule does not apply. 2) Each finding in the guide's format, ranked, with a failure scenario. 3) Each `High` and `Medium` finding becomes a task.
- Accept: the ledger exists; no `High` remains open at the time M3 is claimed; findings have task ids.
- Verify: `scripts/check-spec.sh`

### T-10.02 Legal texts and compliance review
- Executor: owner · Milestone: M3 · Depends: T-09.08
- Read: SPEC §12.1 A-06, A-07, A-13; DECISIONS D-08
- Files: `web/content/legal/**`, `docs/operations/LEGAL.md`
- Do: 1) The owner, with a lawyer and an accountant, supplies the terms, the privacy text and the cookie statement for the session cookies, and answers A-06, A-07, A-13 (including the duties of an online marketplace, how the review-verification method is disclosed, how invoices for the fee are issued). 2) The director replaces the draft notice in the pages with the supplied text and version and records the answers.
- Accept: the pages show the supplied text, version and date; `LEGAL.md` records each of the three assumptions as `confirmed` or `changed` with who answered and when.
- Verify: owner confirms in the PR

### T-10.03 Abuse controls and suspicious pairs
- Executor: implementer · Milestone: M3 · Depends: T-07.10, T-05.04
- Read: SPEC §11 (threat); CONTRACTS §10.2; `service-api-patterns` §1
- Files: `src/Gabinety.Reputation/Features/Analytics/**`, `src/Gabinety.Booking/Features/Analytics/**`, `web/src/app/admin/podejrzane/**`, `tests/Gabinety.Reputation.Tests/Analytics/**`
- Do: 1) An admin endpoint listing practitioner-client pairs with more than two bookings and only five-star reviews (Booking exposes pair counts through an internal endpoint; Reputation joins them). 2) Verify each rate limit of CONTRACTS §10.2 with a test that exceeds it. 3) The admin page.
- Accept: tests: the pair list matches a constructed case and ignores a pair with one 4-star review; each limit answers 429 on the next request and recovers after the window (fake clock).
- Verify: common verify

### T-10.04 Performance measurement
- Executor: supervisor · Milestone: M3 · Depends: T-09.08
- Read: SPEC §13
- Files: `scripts/measure-performance.sh`, `docs/operations/PERFORMANCE.md`
- Do: 1) Generate the scaled dataset through the seeder with a factor. 2) Measure every row of SPEC §13 with the stated tool, record command, environment, date and numbers. 3) A miss is reported as a miss and becomes a task.
- Accept: the file has one measured number per row, or "not measured" with the reason; nothing is rounded in favour of the target.
- Verify: link to the measurement run

### T-10.05 Runbook, backups and restore
- Executor: supervisor · Milestone: M3 · Depends: T-09.08
- Read: SPEC §8, §4.3; `fly-io-deployment` (state)
- Files: `docs/operations/RUNBOOK.md`, `deploy/fly/backup/**`
- Do: 1) Daily dump of each database to object storage and a retention rule. 2) Restore into a scratch database and run a row-count and constraint check. 3) The runbook: deploy, rollback, rotate each secret, handle a payment webhook failure burst, a stuck job, GDPR erasure across authservice and the three hooks, a suspended practitioner with future bookings.
- Accept: one restore performed and logged with the counts; the runbook steps are each marked as run or not run.
- Verify: link to the restore log

### T-10.06 Ranking changelog
- Executor: implementer · Milestone: M3 · Depends: T-07.11
- Read: CONTRACTS §9.3; ADR-0008
- Files: `docs/operations/RANKING-CHANGELOG.md`, `tests/Gabinety.Ranking.Tests/Changelog*`
- Do: 1) Create the changelog with the first row for version `2026-11-v1`. 2) A test that fails when `RankingRules` constants or version change without a new row (the test hashes the constants and compares with the hash recorded in the last row).
- Accept: changing a weight in a branch makes the test fail until the changelog row is added (shown once, then reverted).
- Verify: common verify

### T-10.07 M3 checkpoint
- Executor: supervisor · Milestone: M3 · Depends: T-10.01, T-10.02, T-10.03, T-10.04, T-10.05, T-10.06
- Read: SUPERVISOR §9; SPEC §14 (M3); `session-status`
- Files: `docs/spec/PROGRESS.md`, `README.md`
- Do: 1) Compare every claim of the documents with what can be measured now. 2) Update `PROGRESS.md` and the README in one PR; list the gaps as new tasks. 3) Report M3 with each proof or the evidence of failure.
- Accept: the gate line of M3 is claimed with links, or stated as failed with evidence; "Not shown yet" is up to date.
- Verify: `scripts/check-spec.sh`

---

## Waves

Which tasks may run at the same time (disjoint `Files:` sets). The supervisor never starts more sub-sessions than
DECISIONS D-16 allows, whatever this table permits.

| Wave | Tasks | Blocked by |
|---|---|---|
| 0 | T-00.01 → T-00.02 → T-00.03 → T-00.04 (in order, never parallel; T-01.01 runs after T-00.02 beside T-00.03 only if D-16 allows and T-00.03 does not touch `docs/spikes/`) | — |
| 1 | T-01.01, T-01.02 (read-only spikes, parallel with T-00.03) | T-00.02 |
| 2 | T-01.03 (supervisor), T-03.06, T-04.01, T-04.02 (pure libraries, disjoint) | T-00.03 |
| 3 | T-01.04, T-02.01 → T-02.02 → T-02.03, T-02.04 (T-02.05 follows T-06.02) | T-00.03, T-01.01, T-01.02, T-01.03 |
| 4 | Directory chain T-03.01 → T-03.02 → T-03.03 → T-03.04 → T-03.05 → T-03.07 → T-03.08 | T-00.04, T-01.03 |
| 5 | Booking chain T-04.03 → T-04.04 → T-04.05 → T-04.06 → T-04.07 → T-04.08 → T-04.09 → T-04.10 (parallel with wave 4 and 6 while D-16 allows) | T-04.01, T-04.02, T-03.01 (T-04.10 also T-03.08) |
| 6 | Assistant: T-06.01 → T-06.02 ∥ T-06.03 → T-06.04; Reputation: T-05.01 → T-05.02 → T-05.03 → T-05.04 | T-02.03; T-04.10 for T-05.02; T-06.03 for T-05.04 |
| 7 | Web: T-07.01, T-07.02 first (parallel), then T-07.03, T-07.05, T-07.11 (parallel), then T-07.04, T-07.06, T-07.07, T-07.08, T-07.09, T-07.10 as their dependencies merge | T-00.03, T-01.04; the backend tasks named in each Depends |
| 8 | T-08.01 → T-08.02 → T-08.03 → T-08.04 → T-08.05 → T-08.06 | backend and web complete (M1 gate follows T-08.06 and T-02.06) |
| 9 | T-09.01 (owner, early, parallel with anything) ; T-09.02 → T-09.03 ∥ T-09.04 ; T-09.05, T-09.06 (after T-09.01) ; T-09.07 ; T-09.08 | T-08.05 for T-09.02 |
| 10 | T-10.01, T-10.02, T-10.04, T-10.05 (parallel); T-10.03, T-10.06; then T-10.07 | T-09.08 |

Owner actions that should start early because they gate later work: T-00.01, T-00.02, T-09.01 (account lead times), T-02.06 (needs the owner's machine), T-10.02 (a lawyer's calendar).
