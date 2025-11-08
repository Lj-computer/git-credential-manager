## Quick orientation for AI coding agents

This repository is the cross-platform Git Credential Manager (GCM) written in C# and .NET.
Keep guidance concise and concrete: reference code locations and the project's explicit patterns rather than generic advice.

Key places to look
- `docs/architecture.md` — canonical big-picture with diagrams and the command/provider flow.
- `Git-Credential-Manager.sln` (solution root) and `Directory.Build.props`/`Directory.Build.targets` — centralized build and targeting rules.
- `src/shared/*` — shared libraries and provider implementations (e.g. `src/shared/Atlassian.Bitbucket/BitbucketHostProvider.cs`).
- `src/*/{linux,osx,windows}` — platform-specific helpers and installers.
- Tests are collocated under `*/*.Tests` projects (e.g. `src/shared/*/*.Tests`).

Architecture and patterns to preserve
- GCM centers on a small console entry project (`Git-Credential-Manager`) that registers host providers and runs `Core.Application`. See `docs/architecture.md` and `src/shared/Git-Credential-Manager/Program.cs`.
- Host providers implement `IHostProvider` (or derive from the `HostProvider` base). Typical example: `BitbucketHostProvider` in `src/shared/Atlassian.Bitbucket/BitbucketHostProvider.cs`.
  - `IsSupported(InputArguments)` selects providers by matching host/response.
  - `Get|Store|EraseCredentialAsync` are the primary operations; provider code should prefer the base `HostProvider` helpers and `GetServiceName` for credential keys.
- Commands: `Get`, `Store`, `Erase` are wired to provider selection via the registry and use an `ICommandContext` exposing services: `CredentialStore`, `Settings`, `Streams`, `Terminal`, `Trace`, `HttpClientFactory`, etc. See `docs/architecture.md` for the command flow diagram.
- Async-first: most code uses `async`/`await`. Follow existing async patterns and bubble exceptions to be handled at the entry point.

Build / test / debug workflows (concrete)
- Build the entire solution with dotnet: `dotnet build Git-Credential-Manager.sln` (there is a workspace task labeled `build` that runs this).
- Run tests: `dotnet test Git-Credential-Manager.sln` (workspace task `test`).
- Coverage: the repository has a `test with coverage` task that invokes `--collect 'XPlat Code Coverage'` and a `report coverage - win` task that calls ReportGenerator; see workspace tasks and `.code-coverage/coverlet.settings.xml` for details.
- Platform helpers and installers exist under `src/linux`, `src/osx`, `src/windows` — be mindful of shell scripts and signing steps when changing installer logic.

Project-specific conventions
- Centralized props/targets: prefer adding build settings in `Directory.Build.props`/`Directory.Build.targets` to affect all projects rather than editing many csproj files.
- Provider registration: providers are registered at startup via `Application.RegisterProvider` in the main project — changes to provider ordering can change selection priority. Generic/catch-all provider is registered last.
- Credential keys: service names are derived from the remote URL via `GetServiceName` (no username included) — changing this affects storage/lookup across platforms.
- Secrets handling: tracing exists but MUST filter secrets. Use `ITrace.WriteLineSecrets` where appropriate and avoid logging credentials to normal traces. See `docs/architecture.md` and `BitbucketHostProvider` usage.

Integration points / external dependencies
- OAuth / MSAL: MSAL/`Microsoft.Identity.Client` is used for Microsoft auth — this influences target frameworks (Windows GUI/web popups require .NET Framework or specific MSAL hooks).
- Host provider REST API registries: providers may use an internal `IRegistry<TRestApi>` pattern (see `BitbucketRestApiRegistry`) to abstract API discovery and calls.
- Credential stores: use `ICommandContext.CredentialStore` abstraction — implementations differ per platform. Do not bypass this abstraction.

What to change vs what to avoid
- Change: provider logic, REST API clients, authentication flows, tests for new behavior.
- Avoid: changing the command/provider dispatch model, `GetServiceName` semantics, or removing the centralized `Directory.Build.props` without broad coordination — these are cross-cutting and will break credential lookup or builds.

Search tips and examples to cite in PRs
- To find provider selection code: search for `IHostProvider.IsSupported` and `RegisterProvider`.
- To see how credentials are stored: `GetServiceName`, `ICommandContext.CredentialStore.Get` and `AddOrUpdate` usage (example: `BitbucketHostProvider.StoreCredentialAsync`).
- To inspect a full authentication flow: open `src/shared/Atlassian.Bitbucket/BitbucketHostProvider.cs` and `docs/architecture.md` (both show the refresh vs interactive OAuth flows).

If you edit code
- Add or update unit tests in the corresponding `*.Tests` project. Tests run with `dotnet test` across the solution.
- Run solution build and tests locally before opening PR — CI enforces the same build and test steps.

If anything in this summary looks incomplete or you want more details (specific callers, tests to run, or CI expectations), tell me which area to expand and I will update this file.
