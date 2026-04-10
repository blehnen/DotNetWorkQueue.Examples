# CONCERNS.md

## Overview

This repo contains demo/example applications for the DotNetWorkQueue library across five transports (SQL Server, PostgreSQL, SQLite, LiteDb, Redis). The README explicitly labels this as "the original example code" and redirects users to the newer `DotNetWorkQueue.Samples` repo. The codebase is in maintenance mode rather than active development. Concerns are rated relative to the actual risk for demo code — not production systems.

## Metrics

| Metric | Value |
|--------|-------|
| packages.config files | 17 |
| App.config files with committed connection strings | 8+ (all transport projects) |
| App.config files containing LAN IP addresses | 4 (SQL Server x2, PostgreSQL x2) |
| obj/Debug generated files tracked in git | 20 (`.AssemblyAttributes.cs` files) |
| bin/Debug compiled artifacts tracked in git | Hundreds of DLLs (see Glob results) — Rpc projects only |
| TODO / FIXME / HACK comments in .cs files | 0 |
| Hardcoded real passwords or API tokens found | 0 |
| DotNetWorkQueue core version | 0.8.0 |
| SharedAssemblyInfo version | 0.7.1 (mismatches package version 0.8.0) |
| ConsoleView/ConsoleShared target framework | net452 (mismatches main projects at net472) |

---

## Findings

### 1. Technical Debt

- **WinForms console host (not a plain console app)**
  - All example entry points are WinForms applications, not console executables. Each `Program.cs` calls `Application.Run(form)` with `[STAThread]`.
  - Evidence: `Source/Examples/SQLServer/Producer/SqlServerProducer/Program.cs` (lines 28–47)
  - Impact (LOW for demo): Non-obvious to users who expect a command-line app; cannot be run headlessly (e.g., in CI or Docker). Any code reader unfamiliar with WinForms will be confused by the entry point pattern.

- **Reflection-driven command discovery**
  - The shell UI discovers all runnable commands by scanning the executing assembly via `System.Reflection`. Command routing, parameter binding, and help text are all reflection-based.
  - Evidence: `Source/Examples/Shared/ConsoleView/FormMain.cs` (lines 56, 115–125) — calls `ConsoleGetCommands.GetCommands(_commandAssembly)` and dispatches through `ConsoleExecute.Execute`/`ConsoleExecute.ExecuteAsync`.
  - Impact (LOW): Fragile under rename/refactor. Commands silently disappear if method signatures change. No compile-time validation that commands are reachable.

- **Default TripleDES key/IV exposed in example help text**
  - `SharedCommands.EnableDes` has default parameter values of `"aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"` (key) and `"aaaaaaaaaaa="` (IV), and the `Example()` method prints these defaults as sample invocations.
  - Evidence: `Source/Examples/Shared/ConsoleSharedCommands/Commands/SharedCommands.cs` (lines 58, 86)
  - Impact (LOW for demo, MEDIUM as a copy-paste hazard): These are clearly placeholder values, not real keys. However, a developer copying the `EnableDes` call pattern into production could unknowingly use a fixed trivially-guessable key. The example offers no warning that these values must be changed.

- **No automated testing story**
  - There are no test projects, no test runners configured, and no CI pipeline files in the repo. Verification is entirely manual (run the WinForms app, type commands).
  - Evidence: Glob of `**/*.Tests.*` returns nothing; no `.github/workflows/`, no `azure-pipelines.yml`, no `Jenkinsfile`.
  - Impact (LOW): Expected for demo code, but means regressions after package updates go undetected until someone manually runs each project.

- **Duplication of boilerplate across transport variants**
  - Each transport has its own `Commands/ConsumeMessage.cs`, `Commands/ConsumeMessageAsync.cs`, `Commands/QueueCreation.cs`, and `Commands/SendMessage.cs` with near-identical structure. The shared base (`ConsoleSharedCommands`) reduces some duplication but transport-specific files still duplicate large blocks.
  - Evidence: Compare `Source/Examples/SQLServer/Consumer/SqlServerConsumer/Commands/ConsumeMessage.cs` with `Source/Examples/LiteDb/Consumer/LiteDbConsumer/Commands/ConsumeMessage.cs` — identical structure, different namespace and transport registration call.
  - Impact (LOW): Maintenance cost: any fix to the message-handling pattern must be applied to ~10 files.

- **SharedAssemblyInfo version mismatches installed packages**
  - `SharedAssemblyInfo.cs` declares `AssemblyVersion("0.7.1")` but every `packages.config` pins `DotNetWorkQueue` and related packages at `0.8.0`.
  - Evidence: `SharedAssemblyInfo.cs` (line 7); `Source/Examples/SQLServer/Consumer/SqlServerConsumer/packages.config` (line 11).
  - Impact (LOW): Cosmetic inconsistency, but confusing to anyone checking the assembly version to infer the library version in use.

---

### 2. Dependencies

- **App.Metrics 4.3.0 — unmaintained upstream**
  - App.Metrics has had no releases since 2021. The project's GitHub shows no active development. Version 4.3.0 is the last published release and is used across all 17 projects.
  - Evidence: `Source/Examples/SQLServer/Consumer/SqlServerConsumer/packages.config` (lines 3–8); same in all other `packages.config` files.
  - Impact (MEDIUM): No known CVE directly in App.Metrics, but the library is effectively abandoned. Any bug in metric emission or reporting will not receive an upstream fix. [Inferred] The App.Metrics HTTP reporting endpoint (`EnableStatus`) that uses an embedded HTTP listener may not handle TLS or authentication, consistent with its abandoned state — suspected, needs verification.

- **Polly.Caching.Memory 3.0.2 — targets old Polly v7 API**
  - `Polly.Caching.Memory` 3.x targets the Polly v7 `ISyncPolicy`/`IAsyncPolicy` API. The repo pins `Polly` at 8.6.5 (v8), which ships a new `ResiliencePipeline` API. The v7-style `Polly.Caching.Memory` package is listed alongside v8 core packages.
  - Evidence: `Source/Examples/SQLServer/Consumer/SqlServerConsumer/packages.config` (lines 50, 53) — `Polly.Caching.Memory` 3.0.2 alongside `Polly.Core` 8.6.5.
  - Impact (MEDIUM): Polly v7 compatibility shims exist in v8, so this will likely compile, but the caching policy integration path is the legacy one. If DotNetWorkQueue 0.8.0 uses the v8 ResiliencePipeline internally, the v7-style caching layer may be bypassed silently. [Inferred] — needs verification against DotNetWorkQueue source.

- **Polly.Contrib.Simmy 0.3.0 — targets Polly v7, abandoned**
  - Simmy 0.3.0 is designed for Polly v7. A Polly v8-compatible rewrite exists as `Polly.Contrib.Simmy` 1.x. The 0.3.0 version has no further releases on the v7 branch.
  - Evidence: `Source/Examples/SQLServer/Consumer/SqlServerConsumer/packages.config` (line 51); `Source/Examples/Redis/Consumer/RedisConsumer/packages.config` — Simmy not present in Redis, only in SQL/Postgres/SQLite/LiteDb projects.
  - Impact (LOW): Chaos/fault injection only; not on the production message path. Silently inoperative if Polly v8 does not call v7 policy wrappers in the same way.

- **Polly.Contrib.WaitAndRetry 1.1.1 — only in SQL Server projects**
  - Present only in SQL Server `packages.config`, absent from Redis, PostgreSQL, SQLite, LiteDb. Likely a transitive dependency difference, not intentional.
  - Evidence: `Source/Examples/SQLServer/Consumer/SqlServerConsumer/packages.config` (line 52).
  - Impact (LOW): Inconsistency between transport configurations; no functional impact for demo use.

- **MsgPack.Cli 1.0.1 — only in Redis projects**
  - Redis transport uses MessagePack serialization via `MsgPack.Cli` 1.0.1. This is the last stable release of the library before the project migrated to `MessagePack` (the separate Yoshifumi Kawai library). No known CVE in 1.0.1, but it is effectively end-of-life for the `MsgPack.Cli` package identity.
  - Evidence: `Source/Examples/Redis/Consumer/RedisConsumer/packages.config` (line 31).
  - Impact (LOW): Demo code; no production message integrity concern here.

- **System.IO.Compression 4.3.0 — very old BCL backport**
  - Only the Redis projects reference this package at 4.3.0 (released 2016). All other packages in the same file are at 10.0.1 or similar modern versions.
  - Evidence: `Source/Examples/Redis/Consumer/RedisConsumer/packages.config` (line 46).
  - Impact (LOW): This is a .NET BCL polyfill; on .NET 4.7.2, the in-box `System.IO.Compression` supersedes it. No practical risk, but the stale version is unnecessary clutter.

- **ConsoleView/ConsoleShared target net452 — mismatched TFM**
  - The shared UI library (`ConsoleView`) and the console shell (`ConsoleShared`) target `net452` while all consuming projects target `net472`. This is visible in the tracked `obj/Debug` attribute files.
  - Evidence: `Source/Examples/Shared/ConsoleView/obj/Debug/.NETFramework,Version=v4.5.2.AssemblyAttributes.cs`; `Source/Examples/Shared/ConsoleShared/obj/Debug/.NETFramework,Version=v4.5.2.AssemblyAttributes.cs`.
  - Impact (LOW): .NET Framework is backward-compatible so net452 assemblies load fine in a net472 host. However, .NET Framework 4.5.2 reached end of support on April 26, 2022. Any security fix to the BCL that only ships for 4.6+ will not be available to code compiled against 4.5.2 unless the runtime polyfills it.

---

### 3. Security

- **LAN IP addresses committed in App.config files**
  - SQL Server and PostgreSQL App.config files contain hardcoded connection strings pointing to a specific LAN host (`192.168.0.58`) and a database server (`192.168.0.2` for Redis).
  - Evidence:
    - `Source/Examples/SQLServer/Consumer/SqlServerConsumer/App.config` (line 4): `Server=192.168.0.58;...Database=TestR;Trusted_Connection=True;max pool size=500`
    - `Source/Examples/SQLServer/Producer/SqlServerProducer/App.config` (line 4): same IP and database
    - `Source/Examples/SQLServer/Consumer/SqlServerConsumerAsync/App.config` (line 4): same
    - `Source/Examples/PostGresSQL/Producer/PostGresSQLProducer/App.config` (line 4): `Server=192.168.0.58;Port=5432;Database=IntegrationTesting;Integrated Security=true;`
    - `Source/Examples/PostGresSQL/Consumer/PostGresSQLConsumer/App.config` (line 4): same PostgreSQL connection
    - `Source/Examples/Redis/Producer/RedisProducer/App.config` (line 4): `192.168.0.2`
  - These are private RFC-1918 addresses, not public IPs, and `Trusted_Connection=True` / `Integrated Security=true` means no password is embedded — Windows auth is used. No password, token, or API key is present in any committed file.
  - Impact (LOW for external exposure, MEDIUM as developer experience issue): Any developer cloning the repo and running without editing `App.config` will attempt to connect to someone else's dev machine. The addresses should be replaced with `localhost` or `.\SQLEXPRESS` style defaults in example code.

- **No hardcoded passwords, tokens, or API keys found**
  - Searched all `.cs` files for `password`, `secret`, `token`, `apikey`, and similar patterns. The only match was the placeholder TripleDES key (`aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa`) documented above.
  - Evidence: Grep across `Source/**/*.cs` — no matches beyond the placeholder DES key.
  - Impact: Not a concern.

- **TripleDES default key/IV in example help output**
  - Covered under Technical Debt above. Reiterating here: the defaults are `aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa` / `aaaaaaaaaaa=` — trivially weak, but presented as sample invocations in the UI.
  - Evidence: `Source/Examples/Shared/ConsoleSharedCommands/Commands/SharedCommands.cs` (lines 58, 86).
  - Impact (MEDIUM as copy-paste hazard): No real credential is committed. Risk is that a reader copies the pattern without understanding that the key must be changed.

---

### 4. Repository Hygiene

- **bin/Debug compiled artifacts committed to git for RPC projects**
  - The `.gitignore` correctly excludes `[Bb]in/` and `[Oo]bj/`, but git status shows many RPC project `bin/Debug/` DLLs as modified (not untracked), meaning they were committed in a prior state and the ignore rule was not applied retroactively.
  - Evidence: Glob of `Source/Examples/**/bin/**` returns hundreds of entries across `PostGresSQL/Rpc`, `Redis/Rpc`, `SQLite/Rpc`, `SQLServer/Rpc` projects (DLLs including `Serilog.Sinks.ColoredConsole.dll`, `Microsoft.IO.RecyclableMemoryStream.dll`, etc.).
  - Impact (MEDIUM): Binary artifacts in git inflate repository size, make diffs meaningless, and can carry outdated/patched-over DLL versions. The `Serilog.Sinks.ColoredConsole.dll` in `bin/Debug/` is notable — this package was deprecated in favor of `Serilog.Sinks.Console` years ago; the committed DLL may predate the migration.

- **obj/Debug generated .cs files committed**
  - MSBuild-generated `AssemblyAttributes.cs` files are tracked in source control for 20 projects.
  - Evidence: `Source/Examples/Shared/ConsoleSharedCommands/obj/Debug/.NETFramework,Version=v4.7.2.AssemblyAttributes.cs` and 19 others (see Glob results above).
  - Impact (LOW): Noise in diffs; regenerated on every build. No functional harm.

- **Large number of uncommitted modifications in working tree**
  - The git status at conversation start showed 50+ modified files (`M` prefix) across all transport projects — `App.config`, `.csproj`, `Program.cs`, `AssemblyInfo.cs`, `packages.config` — suggesting a broad package update pass has been done locally but not yet committed.
  - Evidence: git status output (truncated at 2k chars) showing modifications to files in LiteDb, PostGresSQL, Redis, SQLite, SQLServer projects.
  - Impact (LOW): Normal working-tree state during active development. Worth noting for anyone picking up the repo — the working tree does not match HEAD.

- **`Serilog.Sinks.ColoredConsole.dll` in committed bin/ artifacts**
  - `Serilog.Sinks.ColoredConsole` was deprecated by the Serilog maintainers and superseded by `Serilog.Sinks.Console`. The compiled DLL is present in the committed `bin/Debug/` folders for all 8 RPC projects.
  - Evidence: `Source/Examples/Redis/Rpc/RedisRpcConsumerAsync/bin/Debug/Serilog.Sinks.ColoredConsole.dll` (and 7 equivalent paths).
  - Impact (LOW): Deprecated but not removed from the committed artifacts. No CVE in `ColoredConsole`, but the binary is stale.

---

### 5. Drift from Sibling Repo

- **README redirects users to `DotNetWorkQueue.Samples`**
  - The README (line 3) explicitly states: "This is the original example code. It's suggested to look at the sample repo instead, as the samples are more straight forward."
  - Evidence: `README.md` (lines 1–4); link to `https://github.com/blehnen/DotNetWorkQueue.Samples`.
  - Context: This is not a defect but means the repo is in maintenance mode. New features demonstrated in the Samples repo (newer transport options, updated patterns) will not appear here. Consumers of this repo should be aware they may be reading documentation for superseded usage patterns.

---

## Summary Table

| Item | Detail | Severity | Confidence |
|------|--------|----------|------------|
| LAN IPs in committed App.config | `192.168.0.58` in SQL Server + PostgreSQL configs; `192.168.0.2` in Redis config | MEDIUM | Observed |
| bin/Debug DLLs committed for RPC projects | Hundreds of binary artifacts tracked in git, including deprecated `Serilog.Sinks.ColoredConsole.dll` | MEDIUM | Observed |
| App.Metrics 4.3.0 — abandoned upstream | Last release 2021, no active maintenance | MEDIUM | Observed |
| TripleDES placeholder key as copy-paste hazard | Default key `aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa` shown in help output | MEDIUM | Observed |
| Polly v7 caching/Simmy packages alongside Polly v8 | `Polly.Caching.Memory` 3.0.2 + `Polly.Contrib.Simmy` 0.3.0 target old API | MEDIUM | Observed |
| ConsoleView/ConsoleShared target net452 (EOL) | .NET 4.5.2 end of support April 2022; mismatches main projects at net472 | LOW | Observed |
| obj/Debug generated files tracked in git | 20 `AssemblyAttributes.cs` files committed | LOW | Observed |
| SharedAssemblyInfo version 0.7.1 vs packages 0.8.0 | Assembly version does not match installed DotNetWorkQueue version | LOW | Observed |
| WinForms host for demo apps | Cannot run headlessly; non-obvious entry point | LOW | Observed |
| Reflection-driven command discovery | Silent breakage on rename/refactor | LOW | Observed |
| No automated tests or CI | Regressions only caught by manual execution | LOW | Observed |
| Large uncommitted working-tree changes | 50+ modified files not yet committed | LOW | Observed |
| MsgPack.Cli 1.0.1 (Redis only) | End-of-life package identity; no known CVE | LOW | Observed |
| System.IO.Compression 4.3.0 (Redis only) | Stale BCL backport alongside modern packages | LOW | Observed |
| Polly.Contrib.WaitAndRetry absent from non-SQL projects | Inconsistency across transport variants | LOW | Inferred |

## Open Questions

1. Does `DotNetWorkQueue` 0.8.0 internally use the Polly v8 `ResiliencePipeline` API exclusively, or does it retain v7 policy shims? If v7 shims are gone, `Polly.Caching.Memory` 3.0.2 may be non-functional.
2. The `bin/Debug/` artifacts for non-RPC projects are listed as modified in git status but not as tracked binaries in Glob results — are those projects already gitignored correctly and only the Rpc projects escaped the ignore? The `.gitignore` patterns look correct; the Rpc binaries may have been force-added historically.
3. Are there plans to bring the RPC project packages.config files up to parity with the other projects? The RPC projects were not modified in the recent package-update working-tree changes visible in git status.
