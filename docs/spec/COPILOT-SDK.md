# Gabinety.Llm — using the GitHub Copilot SDK properly

> Normative for the project `src/Gabinety.Llm` and for how `Gabinety.Assistant` uses it. Decision record: ADR-0004.
>
> **Source.** Everything here is taken from the sibling project
> [`konradcinkusz/llm-multi-agent-chess`](https://github.com/konradcinkusz/llm-multi-agent-chess), project `src/Chess.Llm`
> (its `README.md` holds the spike notes, its `CopilotLlmGateway.cs` the implementation). That project is a **reference to port
> from, not a dependency**: a task that builds `Gabinety.Llm` attaches the chess repository read-only (it is public) and ports the
> files named in §11, instead of re-deriving the SDK usage from the SDK's documentation. What the chess project could not verify
> (it had no Copilot login in its sandbox) is listed in §9 and stays an assumption here until the owner's live smoke run (T-02.06).

## 1. The fourteen rules

| # | Rule | Why (from the chess project) |
|---|---|---|
| R1 | `GitHub.Copilot.SDK` is referenced by `Gabinety.Llm.csproj` and by no other project. No code outside the project uses a `GitHub.Copilot` namespace. CI greps for it; an architecture test checks the public surface by reflection | Keeps a vendor SDK, its experimental types and its release cadence out of the domain (P11) |
| R2 | No SDK type appears in a public signature. The public surface is plain records and interfaces (§2) | The same |
| R3 | The package version is pinned in `Directory.Packages.props` (the chess project pins `1.0.28`; T-01.02 re-checks the latest compatible one). Nothing relies on a newer member | SDK 1.0.x still carries experimental types |
| R4 | The runtime is started by a hosted service; `StartAsync` **never throws**: it records `Ready`, `NotAuthenticated` or `Unavailable` and disposes the client when not `Ready` | P8: a missing login must not stop the host |
| R5 | The health check `llm` is **never Unhealthy**: `Ready` and `Fake` are Healthy, the others Degraded. `/health` stays HTTP 200 | P8 and the chess invariant |
| R6 | One conversation is one SDK session, created with **no tools**: `AvailableTools = []`, and `OnPermissionRequest` rejects every request as a second guard | The chess project could not confirm that an empty list means "no tools"; the handler makes it safe either way |
| R7 | The system prompt **replaces** the CLI's own: `SystemMessageMode.Replace` | The default prompt describes a coding agent |
| R8 | Every reply has a timeout that the caller passes, applied twice: as the `SendAndWaitAsync` timeout (its own default is 60 s) and as a linked cancellation. On timeout the session is **aborted** (`AbortAsync`, 5 s budget) and a `TimeoutException` is thrown | `SendAndWait` does not stop in-flight work |
| R9 | Replies of one conversation are serialised by the caller. Usage and reasoning events are collected per reply and reset at its start | The event subscription is per session, not per message |
| R10 | Usage is summed per reply from the usage events. `Unavailable` credits mean "not reported", never "free". Reasoning tokens are already inside output tokens and are not added again | Avoids double counting and fake zeros |
| R11 | Model prices come from the runtime's listing, not from guesses. A model without a price is "price unknown", not zero | Copilot bills AI credits per token for token-billed models |
| R12 | **Traffic caused by the public never runs on the Copilot subscription**: only on bring-your-own-key models (§6). Enforced in code by a policy guard in the Assistant (§7), on by default | The chess invariant `RequireApiKeyModels`; the Copilot terms for serving third parties are unverified (A-04) |
| R13 | The key of a bring-your-own-key model is a platform secret. It is never logged, never in an exception message, never in `ToString()` of an options type. Validation errors name the setting, never its value or a URL with credentials | P5 |
| R14 | The tests use a deterministic `FakeLlmGateway`; none needs a network or a login. The live behaviour is checked by a script the owner runs, which refuses to run under CI | Real calls cost money and cannot run in CI |

## 2. Public surface of `Gabinety.Llm`

```csharp
namespace Gabinety.Llm;

public interface ILlmGateway
{
    LlmGatewayStatus Status { get; }
    Task<IReadOnlyList<LlmModelInfo>> ListModelsAsync(CancellationToken ct);
    Task<ILlmConversation> StartConversationAsync(LlmConversationOptions options, CancellationToken ct);
}

public interface ILlmConversation : IAsyncDisposable
{
    string ModelId { get; }
    Task<LlmReply> AskAsync(string prompt, CancellationToken ct);
}

public enum LlmGatewayState { Ready, NotAuthenticated, Unavailable, Fake }
public sealed record LlmGatewayStatus(LlmGatewayState State, string Detail);

public sealed record LlmModelInfo(string Id, string DisplayName, bool Enabled, LlmTokenPrices? Pricing = null,
    IReadOnlyList<string>? ReasoningEfforts = null, string? DefaultReasoningEffort = null, bool Custom = false);

/// <summary>Prices in AI credits per BatchSize tokens (as the runtime lists them), or USD for a custom model.</summary>
public sealed record LlmTokenPrices(long BatchSize, double? InputCredits, double? OutputCredits, double? CachedInputCredits);

public sealed record LlmConversationOptions(string ModelId, string SystemPrompt, TimeSpan Timeout, string? ReasoningEffort = null);

public sealed record LlmReply(string Text, TimeSpan Latency, int? InputTokens, int? OutputTokens,
    string? Reasoning = null, int? ReasoningTokens = null, int? CachedInputTokens = null, double? AiCredits = null);

public sealed class LlmUnavailableException(string message) : Exception(message);

public static class LlmServiceCollectionExtensions
{
    /// <summary>Registers the gateway chosen by Llm:Provider (Copilot or Fake), the "llm" health check and the options.</summary>
    public static IHostApplicationBuilder AddLlmGateway(this IHostApplicationBuilder builder);
}
```

Differences from the chess project: no `PremiumMultiplier` (the multiplier is null for token-billed models and unused here), no
`ArenaOptions` (the custom-model section is named `Llm:CustomModels`, §6), and no reasoning-effort validation beyond passing the value
through. Everything else keeps its name so a port is a copy with a rename.

## 3. SDK members used (from the chess spike, SDK 1.0.26–1.0.28)

Namespace `GitHub.Copilot` unless noted. Re-verified by T-01.02 against the pinned package; a member that no longer exists is a finding there, not a silent change.

| Need | Member |
|---|---|
| Client | `new CopilotClient(CopilotClientOptions)`, `StartAsync(ct)`, `DisposeAsync()` |
| Options | `CopilotClientOptions.Connection`, `.GitHubToken`. `UseLoggedInUser` stays at its default |
| Connection | `RuntimeConnection.ForStdio(path, args)` for `Copilot:CliPath`; `RuntimeConnection.ForUri(url, connectionToken)` for `Copilot:CliUrl`; `null` = the runtime bundled with the SDK |
| Sign-in | `GetAuthStatusAsync(ct)` → `IsAuthenticated`, `Login`, `StatusMessage` |
| Models | `ListModelsAsync(ct)` → `ModelInfo { Id, Name, Policy?.State, Billing?.TokenPrices, SupportedReasoningEfforts, DefaultReasoningEffort }` |
| Session | `CreateSessionAsync(SessionConfig, ct)` → `CopilotSession`; `SessionConfig.Model`, `.ReasoningEffort`, `.Provider`, `.SystemMessage = new SystemMessageConfig { Mode = SystemMessageMode.Replace, Content }`, `.AvailableTools = []`, `.Streaming = false`, `.OnPermissionRequest` |
| Ask | `session.SendAndWaitAsync(new MessageOptions { Prompt }, timeout, ct)` → `AssistantMessageEvent?`, text in `.Data.Content` |
| Abort | `session.AbortAsync(ct)` |
| Events | `session.On<AssistantUsageEvent>(handler)` and `session.On<AssistantReasoningEvent>(handler)`, each returns `IDisposable` |
| Prices | `ModelInfo.Billing.TokenPrices { BatchSize, InputPrice, OutputPrice, CacheReadPrice, CachePrice (deprecated) }` and `client.Rpc.Models.ListTokenPriceUnitsAsync(ct)` for the unit (`AiCredits` or `NanoAiu`) |
| Teardown | `session.DisposeAsync()` |
| Custom provider | `ProviderConfig { Type, ModelProvider, WireApi, BaseUrl, ApiKey, WireModel }` assigned to `SessionConfig.Provider`, with no Copilot model id |

Experimental types (`Rpc.PermissionDecision`, `Rpc.ModelPolicyState`, `Rpc.TokenPriceUnit`) are marked `GHCP001`. The `#pragma warning disable GHCP001` is scoped to the single method that needs each one, with a comment saying why. The `-warnaserror` build must still pass.

## 4. Session settings (the one place a session is configured)

An internal static class `CopilotSessionSettings` with `ForCopilotModel(options)` and `ForCustomModel(options, custom)`; both return a `SessionConfig` with the values of R6, R7 and `Streaming = false`. It is `internal` so tests reach it through `InternalsVisibleTo` without a runtime. Tests assert: tools list is empty, the permission handler rejects, the system message mode is `Replace` and carries the prompt, the model id is null for a custom model and the provider is set, `ReasoningEffort` is passed through.

## 5. Lifecycle, status and the timeout contract

- `StartAsync`: one linked cancellation with `Copilot:StartupTimeout` (30 s) around start, sign-in check and model listing together. Outcomes: `Ready` ("Signed in as <login>; <n> models."), `NotAuthenticated`, `Unavailable` ("Copilot runtime unavailable: <ExceptionType>: <message>"). The message never contains a token.
- `ListModelsAsync` and `StartConversationAsync` throw `LlmUnavailableException(Status.Detail)` unless `Ready`. The Assistant maps that to `assistant_unavailable`.
- `AskAsync` throws `TimeoutException` on timeout (after the abort), `OperationCanceledException` when the caller cancelled (after the abort), `InvalidOperationException` when the model finished without a message.
- An `ActivitySource` named `Gabinety.Llm` emits `chat <model>` with `gen_ai.request.model` and the usage tags (`gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`). **No prompt, reply or reasoning text is put on a span, a log or a metric.**

## 6. Configuration

| Key | Default | Meaning |
|---|---|---|
| `Llm:Provider` | `Copilot` | `Copilot` or `Fake`; anything else stops the host at startup with a message naming the key |
| `Copilot:CliPath` | SDK-bundled runtime | Copilot CLI executable |
| `Copilot:CliUrl` | unset | already running runtime; wins over `CliPath` |
| `Copilot:GitHubToken` | unset | replaces the logged-in user; secret; never logged |
| `Copilot:StartupTimeout` | 30 s | start, sign-in, listing |
| `Llm:Seed` | `1` | seed for the fake gateway |
| `Llm:FakeLatencyMs` | `0` | simulated delay of the fake gateway |
| `Llm:CustomModels:N:*` | empty | bring-your-own-key models: `Id` (no `@`, no spaces, unique ignoring case), `DisplayName`, `ProviderType` (`anthropic`, `openai` or `azure`), `BaseUrl` (absolute http or https), `Model` (the name the endpoint expects, sent as `ProviderConfig.WireModel`), `WireApi` (needed for `openai` and `azure`, default `chat-completions`), `ApiKey`, and optionally a **complete pair** of prices per million tokens: `UsdPerMillionInput` and `UsdPerMillionOutput` |

Validation at startup (`ValidateOnStart`): every rule above; a bad entry stops the host with a message that names the entry (`Llm:CustomModels[0].Id 'x@y' must not contain '@'`) and never quotes a key or a URL. `ApiKey` is never a plain value: the AppHost reads it from the secret parameter `custom-model-<id>-key` and passes it as `Llm__CustomModels__N__ApiKey`; a plain `ApiKey` in a file stops the AppHost. `CustomModelOptions.ToString()` omits the key (a test asserts it). A custom id equal to a Copilot id is not listed and an error is logged naming it.

`ProviderType` mapping: `anthropic` is the Anthropic Messages API (`https://api.anthropic.com`; `WireApi` is not sent); `openai` is any OpenAI-compatible endpoint; `azure` is Azure OpenAI; the telemetry `ModelProvider` is `anthropic`, `openai` or `azure_openai`.

## 7. The traffic policy guard (in `Gabinety.Assistant`, not in `Gabinety.Llm`)

`Llm` offers mechanism; the Assistant owns policy. An `ITrafficPolicy` decides, for each request that a public user caused, whether a model may be used:

- `Assistant:RequireApiKeyModelsForTraffic` is true (default): only models with `Custom = true` may serve a request caused by a client, a practitioner or a moderator. Otherwise `policy_refused`, answered as `assistant_unavailable`.
- The fake gateway's models count as allowed (they cost nothing and serve tests and the fake profile).
- The setting is read at startup; changing it needs a restart. The status endpoint reports `Disabled` when no allowed model exists.
- A test proves that with the default setting and a Copilot-only model list, `POST /api/v1/assistant/brief` answers `503` and no session was created (a counting fake gateway).
- The owner's own runs (smoke script, evaluations) use a separate entry point that does not go through the public endpoints (T-02.05 and T-02.06, T-06.05).

The policy exists because the Copilot terms for serving other people from a personal subscription are **not verified** (A-04); the owner may relax the default only by a recorded decision.

## 8. Usage and cost (never part of any rating)

Per reply (`UsageTally`): `InputTokens`, `OutputTokens`, `ReasoningTokens`, `CachedInputTokens`, `AiCredits` (nano-AIU ÷ 1 000 000 000, summed; `Unavailable` events add nothing; `Partial` is summed as reported and may undercount). The constant `NanoAiuPerAiCredit` lives in one place (assumption: 1 AI unit = 1 AI credit, unverified live). For a custom model the Assistant estimates USD from the configured price pair. The Assistant writes one `assistant_usage` row per call (CONTRACTS §6) and refuses new work when the sum of `usd_estimate` of the UTC day reaches `Assistant:DailyBudgetUsd` (`over_budget`). Cost, tokens and latency are **never** an input to ranking, ratings, verification or fees (SPEC §5 rule 6).

## 9. Not verified (assumptions carried from the chess project, plus ours)

| # | Item | Settled by |
|---|---|---|
| A-04 | The Copilot terms allow or forbid serving end users from a personal subscription. Treated as forbidden | The owner reads the current terms; ADR amendment |
| A-05 | A bring-your-own-key session needs a running Copilot runtime, and possibly a sign-in; the production image therefore carries the runtime | T-01.02 (spike), T-09.07 (deployed) |
| A-08 | `AvailableTools = []` means "no tools" (the permission handler is the second guard) | T-02.06 live smoke |
| A-09 | `SystemMessageMode.Replace` removes the CLI's own system message | T-02.06 |
| A-10 | `CopilotUsage.TotalNanoAiu` is set on usage events of a real reply; 1 AIU = 1 credit | T-02.06 |
| A-11 | A user-supplied `Copilot:CliPath` needs no extra arguments (`ForStdio(path, [])`); if the CLI needs a server-mode flag the fix is one line in `BuildClientOptions` | T-02.06 |
| A-12 | The runtime bundled with the SDK is downloaded during `dotnet build` (the chess project: about 89 MB, checked against a published SHA-256 and cached under the project's `obj`). The first build, and every cold CI runner, needs internet access and time; CI caches it | T-00.03 measures it |

A behaviour in this table is described in documents and PRs as "assumed" until its task has recorded a real result, then the row is closed with the evidence link.

## 10. `FakeLlmGateway` (deterministic, offline)

Two models, `fake-a` and `fake-b` (`Enabled`, priced at 1.0 input and 4.0 output credits per million tokens; efforts `low`, `medium`, `high`). State `Fake`, detail "Fake gateway". Usage approximated at four characters per token. The reply depends only on the prompt text and `Llm:Seed`:

| Prompt contains (case-insensitive) | Reply |
|---|---|
| the system prompt id `brief/v1` and the words `migrac` or `legacy` or `.net framework` | a valid brief JSON with `problemSlug` `migracja-legacy-dotnet`, `techSlugs` `["dotnet","csharp"]`, confidence `high` |
| `brief/v1` and `ai`, `llm`, `gpt` or `agent` | `integracja-ai`, `["llm"]`, `medium` |
| `brief/v1`, otherwise | `audyt-kodu`, `[]`, `low` |
| `[[invalid]]` | text that is not JSON (tests the repair attempt and the final `503`) |
| `[[slow]]` | waits for the caller's timeout, observing cancellation (tests the abort path) |
| `[[unavailable]]` | throws `LlmUnavailableException` |
| the system prompt id `triage/v1` and `[[hold]]` or `oszust` or `spam` | `{"flag":"hold","reasons":["spam"]}` |
| `triage/v1`, otherwise | `{"flag":"none","reasons":[]}` |

The fake serves `Llm:CustomModels` as well, so the bring-your-own-key path and the policy guard run offline.

## 11. Porting map from the chess project

| Chess file (`src/Chess.Llm/`) | Target (`src/Gabinety.Llm/`) | Changes |
|---|---|---|
| `LlmContracts.cs` | `LlmContracts.cs` | drop `PremiumMultiplier`; namespace |
| `CopilotLlmGateway.cs` | `CopilotLlmGateway.cs` | drop `ArenaOptions` for `LlmOptions`; `Llm:CustomModels`; ActivitySource `Gabinety.Llm` |
| `CopilotSessionSettings.cs` | same | none beyond namespace |
| `CopilotClientHostedService.cs` | same | none |
| `CopilotOptions.cs` | same | none |
| `UsageTally.cs`, `TokenPriceMapping.cs`, `CustomModelPricing.cs` | same | `UsdPerAiCredit` becomes `Llm:UsdPerAiCredit` (default 0.01) |
| `ArenaOptions.cs` (custom model parts) | `LlmOptions.cs`, `CustomModelOptions.cs`, `LlmOptionsValidator.cs` | only the custom-model section and its validation |
| `FakeLlmGateway.cs` | same | replies of §10 instead of chess moves |
| `LlmGatewayHealthCheck.cs`, `LlmServiceCollectionExtensions.cs` | same | health check name `llm` |
| `ReplyReasoning.cs` | `ReplyReasoning.cs` | none (reasoning is only for diagnostics; it is never stored) |
| tests: `Chess.Llm.Tests` | `Gabinety.Llm.Tests` | keep the degraded-path tests (missing CLI path, refused loopback URI end in `Unavailable`, no exception, no network), the usage and price tests, the session-settings tests; add the §10 fake tests and the policy guard tests in `Gabinety.Assistant.Tests` |
| `tests/.../PublicSurfaceTests` | `Gabinety.Architecture.Tests` | reflection over public types: no member exposes a type from an assembly named `GitHub.Copilot*` |
| `scripts/smoke-copilot.sh` | `scripts/smoke-llm.sh` | runs the Assistant's `smoke` command-line verb (not an HTTP endpoint, T-02.05) with one brief; prints model id, latency, tokens, credits; refuses under `CI`; refuses when `Llm:Provider` would be `Fake` |
