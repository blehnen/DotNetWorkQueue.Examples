# Project: examples-freeze

Freeze `DotNetWorkQueue.Examples` at DotNetWorkQueue **0.9.18** / **.NET Framework 4.8** — the last version of the upstream library that supports a .NET Framework target. After this pass the repo receives no further updates.

## Description

The upstream `DotNetWorkQueue` library dropped .NET Framework 4.8 and .NET Standard 2.0 in version **0.9.19** (2026-04-07), moving to .NET 10 / .NET 8 exclusively. This examples repo cannot follow forward without abandoning its .NET Framework classic-csproj identity, which is the one thing that differentiates it from the newer `DotNetWorkQueue.Samples` repo.

The goal of this project is to bring the examples to a clean, self-consistent, runnable final state: bumped to .NET Framework 4.8, pinned at DotNetWorkQueue 0.9.18, stripped of the removed `AppMetrics` dependency, tidied of known hygiene issues, verified by a Release build and a real runtime smoke test, and stamped with a GitHub Actions CI workflow so the build signal persists. Readers looking for modern metrics usage, current API surface, or `.NET 10` examples are redirected to `DotNetWorkQueue.Samples`.

This is a **one-way door**. After the freeze commit, the repo is in maintenance-only state — no new features, no further version bumps, no transport additions.

## Goals

1. Bump every runnable project from net472 → **net48**.
2. Bump the 4 shared library projects (`ConsoleShared`, `ConsoleView`, `ExampleMessage`, `ShellControlV2`) from net452 → net48, and `ConsoleSharedCommands` from net472 → net48 — bring the entire solution to a single consistent TFM.
3. Pin all `DotNetWorkQueue.*` NuGet packages at **0.9.18** (the last 4.8-supporting release).
4. Strip all `DotNetWorkQueue.AppMetrics` and `App.Metrics.*` references plus any code that wires into them. `AppMetrics` was removed upstream in 0.9.1; the examples will no longer demonstrate metrics. Readers interested in metrics are pointed at `DotNetWorkQueue.Samples`.
5. Delete orphaned RPC `bin/Debug` and `obj/Debug` artifact directories across all 4 affected transports (RPC source + csproj files are already gone; only stale build outputs remain).
6. Fix known hygiene issues: the LiteDb `namespace SQLiteConsumer.Commands` copy-paste typo, `SharedAssemblyInfo.cs` version bump to 0.9.18 to match the pinned library, purge all committed `bin/` / `obj/` artifacts, confirm `.gitignore` rules cover both going forward.
7. Correct committed `App.config` connection strings for SQL Server and PostgreSQL from `192.168.0.58` → `192.168.0.2` (the actual test host).
8. Update `README.md` with a clear "frozen" banner: pinned version, TFM, reason, and a link to `DotNetWorkQueue.Samples` for current usage.
9. Add a minimal GitHub Actions CI workflow (`windows-latest`, MSBuild + NuGet) that proves the solution builds clean in Release.

## Non-Goals

- **No SDK-style csproj migration.** Classic csproj + `packages.config` stays.
- **No `dotnet build` / `dotnet test` toolchain.** MSBuild.exe + `nuget.exe restore` only.
- **No OpenTelemetry.Metrics port.** Metrics wiring is stripped, not replaced. `DotNetWorkQueue.Samples` already demonstrates the modern metrics story.
- **No Jenkins pipeline.** No Windows build agent is available, and MSBuild-on-Linux-via-Mono is the wrong trade right before a freeze. GitHub Actions only.
- **No new transports, no new commands, no refactor of the shared-base hierarchy.**
- **No new test projects.** Zero automated tests exist today and zero will exist after the freeze. CI is build-only; the SQLite/LiteDb round-trip is a manual gate.
- **Not archiving the GitHub repo.** That is a GitHub-side action outside this project's scope.
- **Not abstracting LAN IPs to placeholders.** They are being corrected to the repo owner's actual test environment, not removed.
- **Not following DotNetWorkQueue 0.9.19+.** One-way door.

## Requirements

### Investigation & verification (spike)

- **R1.** Confirm DotNetWorkQueue 0.9.18 publishes NuGet packages for every transport in use (`SqlServer`, `PostgreSQL`, `SQLite`, `LiteDb`, `Redis`) at a matching version. If any transport lags, document the highest consistent floor version and raise a scope question before proceeding.
- **R2.** Enumerate every `.cs` file and csproj that references `App.Metrics` or `DotNetWorkQueue.AppMetrics`. Produce a concrete "files to touch for metrics strip" list before making any code edits.

### TFM bump

- **R3.** Update `<TargetFrameworkVersion>` to `v4.8` in all csproj files in `Source/Examples/` (runnable + shared).
- **R4.** Update matching `<supportedRuntime>` in every `App.config` and `targetFramework` in every `packages.config`.
- **R5.** Verify each package's `net48` lib folder is selected by NuGet. Remove any lower-target `HintPath` entries from csproj `<Reference>` items.

### Package bump

- **R6.** Pin `DotNetWorkQueue` and all `DotNetWorkQueue.Transport.*` at 0.9.18 across every `packages.config`.
- **R7.** Let NuGet pull the updated transitive graph on restore. Remove any direct references to packages that are now transitively provided and conflict (in particular the standalone Polly v7 reference flagged by the concerns map).
- **R8.** Remove all `DotNetWorkQueue.AppMetrics` and `App.Metrics.*` package references from every `packages.config` and every csproj `<Reference>` / `<HintPath>` entry.

### Code absorption

- **R9.** Strip every `using App.Metrics` and every AppMetrics DI hookup from example code. Remove the entire metric-registration code path. Producer/consumer wiring otherwise intact.
- **R10.** Fix `Source/Examples/LiteDb/Consumer/LiteDbConsumer/Commands/ConsumeMessage.cs`: declared namespace `SQLiteConsumer.Commands` → `LiteDbConsumer.Commands`.
- **R11.** Absorb any other 0.8.0 → 0.9.18 breaking-change edits that actually hit example code. Based on the changelog read, the only expected hit is metrics. Treat any additional break as a scope-check trigger.

### Repo hygiene

- **R12.** Update `SharedAssemblyInfo.cs` assembly version to `0.9.18` to match the pinned library.
- **R13.** Delete orphaned RPC build artifacts under `Source/Examples/{SqlServer,PostGresSQL,SQLite,Redis}/Rpc/**/{bin,obj}`.
- **R14.** `git rm -r` every committed `bin/` and `obj/` directory in the repo. Confirm `.gitignore` excludes both; add rules if not.
- **R15.** Correct SQL Server + PostgreSQL `App.config` connection strings: `192.168.0.58` → `192.168.0.2`, across all 6 producer/consumer projects.

### Documentation

- **R16.** Rewrite the top of `README.md`: frozen banner naming DotNetWorkQueue 0.9.18 + .NET Framework 4.8, one-paragraph reason (upstream moved to net10/net8 in 0.9.19), link to `DotNetWorkQueue.Samples` for modern usage and metrics. Preserve the existing transport bullet list and license section below.
- **R17.** Update `CLAUDE.md`'s TFM reference from `4.7.2` to `4.8` and note the pin at 0.9.18. Leave everything else.

### Continuous integration

- **R-CI-1.** Add `.github/workflows/ci.yml`. Single job on `runs-on: windows-latest`, triggers on `push` to `main` and `pull_request` to `main`. Steps: `actions/checkout@v4` → `microsoft/setup-msbuild@v2` → `NuGet/setup-nuget@v2` → `nuget restore Source/Examples/Examples.sln` → `msbuild Source/Examples/Examples.sln /p:Configuration=Release`. No test step.
- **R-CI-2.** Mirror structural patterns (trigger shape, branch list) from `DotNetWorkQueue.Samples/.github/workflows/ci.yml` but substitute the MSBuild toolchain for `dotnet build` — the two are not interchangeable for classic csproj + packages.config.
- **R-CI-3.** The freeze commit is the first commit that must pass this workflow green on `main`.

### Verification

- **R18.** Solution builds clean in Release via `nuget restore` + `msbuild`. Zero errors, no new warnings versus a pre-change baseline count captured before any edits.
- **R19.** **Mandatory manual gate**: SQLite (or LiteDb) producer + consumer round-trip smoke test. Launch the WinForms producer, create a queue, send a message; launch the matching consumer, receive the message, verify the shell log shows the round-trip. Capture a transcript or screenshot in the plan's verification notes.
- **R20.** **Best-effort (non-blocking)**: producer ↔ consumer round-trip on SQL Server, PostgreSQL, and Redis against `192.168.0.2`. Each transport either passes, or its failure is documented inline with a one-line note and the freeze still ships.

## Non-Functional Requirements

- **Reversibility** — each work area commits as an atomic, revertable step. If any single step breaks the build in a way that cannot be fixed in ~30 min of iteration, `git revert` that commit and reopen scope before continuing.
- **No new abstractions** — zero refactoring beyond what R1–R20 + R-CI-1–3 call for. New cleanup ideas that surface mid-build go into a deferred-notes file, not into the freeze commit.
- **Baseline warning discipline** — capture the pre-change Release build warning count before any edits. The freeze commit may equal or reduce that count, never exceed it. Any new warning requires explicit acknowledgement and justification.
- **Determinism** — `nuget restore` + `msbuild` from a fresh checkout must reproduce identical build artifacts on the owner's machine, on GitHub Actions `windows-latest`, and for any future reader. No dependence on a specific Visual Studio version or global NuGet cache state.

## Success Criteria

The freeze is complete when all of the following are simultaneously true:

1. Every `.csproj` in `Source/Examples/` declares `v4.8`. Zero projects at net452 or net472.
2. Every `packages.config` references `DotNetWorkQueue` and transports at exactly `0.9.18`. Zero references to `DotNetWorkQueue.AppMetrics` or `App.Metrics.*`.
3. `Source/Examples/Examples.sln` builds clean in Release — zero errors, no new warnings vs baseline.
4. `.github/workflows/ci.yml` exists; its first run on the freeze commit is green.
5. R19 manual SQLite/LiteDb smoke test passes, with transcript captured.
6. R20 remote smoke tests attempted; each transport passes or has a one-line "why we shipped anyway" note.
7. `LiteDbConsumer/Commands/ConsumeMessage.cs` declares `namespace LiteDbConsumer.Commands`.
8. `SharedAssemblyInfo.cs` declares assembly version `0.9.18`.
9. No `bin/` or `obj/` directories tracked in git; `.gitignore` covers both.
10. `README.md` opens with a frozen banner naming pin, TFM, reason, and the link to `DotNetWorkQueue.Samples`.

## Constraints

- **Technical:** Classic csproj + `packages.config` only. MSBuild.exe + `nuget.exe` only. No `dotnet build`. No Mono. Windows-only CI. No new test projects. No SDK-style migration.
- **Upstream:** This is a one-way door. After 0.9.18, the upstream library moves forward without us. No path to 0.9.19+ without abandoning .NET Framework, which is not what this repo is for.
- **Timeline:** Single focused session. If the spike surfaces breaking changes beyond metrics that cost more than half a day to absorb, re-open scope rather than push through.
- **Budget:** Zero cost. GitHub-hosted `windows-latest` runner only. No paid services, no hosted infrastructure beyond what GitHub provides free for public repos.
- **Infrastructure:** R20 remote smoke tests depend on SQL Server, PostgreSQL, and Redis being reachable on `192.168.0.2` at smoke-test time. If any service is down, R20 degrades to "document and skip" rather than blocking the freeze.
