# STACK.md

## Overview

All projects in this repository target .NET Framework 4.7.2 (with the exception of two shared library projects that target .NET Framework 4.5.2). The build model is classic-style `.csproj` with `packages.config` NuGet restore — not SDK-style. Every runnable project is a WinForms executable that hosts an embedded console shell. All DotNetWorkQueue transport packages are pinned at version 0.8.0 with no version drift across transports.

## Metrics

| Metric | Value |
|--------|-------|
| Total .csproj files | 20 |
| Runnable (WinExe) projects | 15 (3 per transport × 5 transports) |
| Shared library projects | 5 |
| packages.config files | 17 |
| DotNetWorkQueue version (all transports) | 0.8.0 |
| Assembly version (SharedAssemblyInfo.cs) | 0.7.1 |
| Solution file | `Source/Examples/Examples.sln` |

## Findings

### .NET Target Framework

- **Runnable projects**: All 15 transport projects target `v4.7.2`
  - Evidence: `Source/Examples/SQLServer/Producer/SqlServerProducer/SqlServerProducer.csproj` — `<TargetFrameworkVersion>v4.7.2</TargetFrameworkVersion>`
  - Evidence: `Source/Examples/LiteDb/Producer/LiteDbProducer/LiteDbProducer.csproj` — confirmed `v4.7.2`
  - Evidence: verified across SQLite, PostgreSQL, Redis consumer/async projects — all `v4.7.2`
- **Shared library projects — mixed target**: Two shared libraries still target `v4.5.2`, not `v4.7.2`
  - `Source/Examples/Shared/ConsoleShared/ConsoleShared.csproj` — `v4.5.2`
  - `Source/Examples/Shared/ConsoleView/ConsoleView.csproj` — `v4.5.2`
  - `Source/Examples/Shared/ExampleMessage/ExampleMessage.csproj` — `v4.5.2`
  - `Source/Examples/Shared/ShellControlV2/ShellControlV2.csproj` — `v4.5.2`
  - `Source/Examples/Shared/ConsoleSharedCommands/ConsoleSharedCommands.csproj` — `v4.7.2`
  - [Inferred] The `v4.5.2` projects are legacy carry-overs; they compile fine under the v4.7.2 runnable projects due to backward compatibility.

### Project Style and NuGet Restore

- **Classic-style `.csproj`** (not SDK-style): all projects use `<Project ToolsVersion=...>` with explicit `<Reference>` and `<HintPath>` entries pointing into a shared `Source/Examples/packages/` folder
  - Evidence: `Source/Examples/SQLServer/Producer/SqlServerProducer/SqlServerProducer.csproj`
- **NuGet restore model**: `packages.config` in every runnable and shared project that has NuGet dependencies (17 files total)
  - Evidence: `Source/Examples/LiteDb/Consumer/LiteDbConsumer/packages.config`, etc.
  - Restore command: `nuget restore Source/Examples/Examples.sln` (not `dotnet restore`)
- **Build command**: `msbuild Source/Examples/Examples.sln /t:Build /p:Configuration=Debug`
  - Evidence: `CLAUDE.md`

### Output Types

- All 15 transport projects: `<OutputType>WinExe</OutputType>`
  - Evidence: `Source/Examples/SQLServer/Producer/SqlServerProducer/SqlServerProducer.csproj`
- Shared projects: `<OutputType>Library</OutputType>`

### DotNetWorkQueue Core + Transport Packages

All packages are uniform at version **0.8.0** across every transport. No version drift detected.

| Package | Version | Used by transports |
|---------|---------|-------------------|
| `DotNetWorkQueue` | 0.8.0 | All |
| `DotNetWorkQueue.AppMetrics` | 0.8.0 | All |
| `DotNetWorkQueue.Transport.Shared` | 0.8.0 | All |
| `DotNetWorkQueue.Transport.RelationalDatabase` | 0.8.0 | SQL Server, PostgreSQL, SQLite |
| `DotNetWorkQueue.Transport.SqlServer` | 0.8.0 | SQL Server only |
| `DotNetWorkQueue.Transport.PostgreSQL` | 0.8.0 | PostgreSQL only |
| `DotNetWorkQueue.Transport.SQLite` | 0.8.0 | SQLite only |
| `DotNetWorkQueue.Transport.LiteDb` | 0.8.0 | LiteDb only |
| `DotNetWorkQueue.Transport.Redis` | 0.8.0 | Redis only |

Evidence for SQL Server: `Source/Examples/SQLServer/Producer/SqlServerProducer/packages.config`
Evidence for PostgreSQL: `Source/Examples/PostGresSQL/Producer/PostGresSQLProducer/packages.config`
Evidence for Redis: `Source/Examples/Redis/Producer/RedisProducer/packages.config`
Evidence for LiteDb: `Source/Examples/LiteDb/Producer/LiteDbProducer/packages.config`
Evidence for SQLite: `Source/Examples/SQLite/Producer/SQLiteProducer/packages.config`

### Supporting / Infrastructure Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| `App.Metrics` | 4.3.0 | Metrics instrumentation (via `DotNetWorkQueue.AppMetrics`) |
| `App.Metrics.Abstractions` | 4.3.0 | Metrics abstractions |
| `App.Metrics.Core` | 4.3.0 | Metrics core |
| `App.Metrics.Formatters.Ascii` | 4.3.0 | ASCII metrics output |
| `App.Metrics.Formatters.Json` | 4.3.0 | JSON metrics output |
| `App.Metrics.Reporting.Console` | 4.3.0 | Console metrics reporter (ConsoleSharedCommands only) |
| `Serilog` | 4.3.0 | Structured logging |
| `Serilog.Sinks.Console` | 6.1.1 | Console log sink |
| `Microsoft.Extensions.Logging` | 10.0.1 | Logging abstractions used by DotNetWorkQueue |
| `Microsoft.Extensions.Logging.Abstractions` | 10.0.1 | |
| `Microsoft.Extensions.DependencyInjection` | 10.0.1 | DI container |
| `SimpleInjector` | 5.5.0 | DI container (used internally by DotNetWorkQueue) |
| `Newtonsoft.Json` | 13.0.4 | JSON serialization |
| `OpenTelemetry` | 1.14.0 | Observability |
| `OpenTelemetry.Api` | 1.14.0 | |
| `Polly` | 8.6.5 | Resilience / retry policies |
| `Polly.Core` | 8.6.5 | |

Evidence: `Source/Examples/Shared/ConsoleSharedCommands/packages.config`, `Source/Examples/Shared/ConsoleView/packages.config`

### Transport-Specific Database Driver Libraries

| Transport | NuGet driver package | Version |
|-----------|---------------------|---------|
| SQL Server | `Microsoft.Data.SqlClient` | 6.1.3 |
| SQL Server | `Microsoft.Data.SqlClient.SNI` | 6.0.2 |
| PostgreSQL | `Npgsql` | 8.0.8 |
| SQLite | `System.Data.SQLite.Core` | 1.0.119.0 |
| SQLite | `Stub.System.Data.SQLite.Core.NetFramework` | 1.0.119.0 |
| LiteDb | `LiteDB` | 5.0.21 |
| Redis | `StackExchange.Redis` | 2.10.1 |

Evidence: respective `packages.config` files for each transport producer project.

### Shared Assembly Version

- `SharedAssemblyInfo.cs` at repo root is linked into every project and sets `AssemblyVersion`, `AssemblyFileVersion`, `AssemblyInformationalVersion` to `0.7.1`
  - Evidence: `SharedAssemblyInfo.cs` lines 7-9
  - Note: the NuGet packages in use are version 0.8.0 — the assembly version was not updated to match.

### Console Shell Framework

- All runnable projects use a WinForms host (`ConsoleView.FormMain`) with an embedded console emulator (`ShellControlV2`, derived from a CodeProject article)
- Commands are discovered via reflection over `IConsoleCommand` implementations in the `Commands/` folder
  - Evidence: `CLAUDE.md`, `Source/Examples/Shared/ConsoleShared/Commands/DefaultCommands.cs`

### IDE / Tooling Hints

- No `.editorconfig`, `.vsconfig`, or analyzer packages were found in the repository.
- [Inferred] Projects are intended to be opened with Visual Studio (classic MSBuild / NuGet workflow). No Rider or VS Code configuration files present.

## Summary Table

| Item | Detail | Confidence |
|------|--------|------------|
| Language | C# | Observed |
| .NET Framework (runnable) | 4.7.2 | Observed |
| .NET Framework (4 shared libs) | 4.5.2 | Observed |
| csproj style | Classic (non-SDK) | Observed |
| NuGet restore | packages.config | Observed |
| DotNetWorkQueue version | 0.8.0 (uniform) | Observed |
| App.Metrics version | 4.3.0 | Observed |
| Serilog version | 4.3.0 | Observed |
| SimpleInjector version | 5.5.0 | Observed |
| Solution file | `Source/Examples/Examples.sln` | Observed |
| Assembly version (SharedAssemblyInfo) | 0.7.1 | Observed |
| .editorconfig / analyzers | None found | Observed |
| Build tool | MSBuild / Visual Studio | Observed |

## Open Questions

- Why does `SharedAssemblyInfo.cs` declare version 0.7.1 while the NuGet packages are at 0.8.0? Is the assembly version intentionally behind?
- The four shared library projects (`ConsoleShared`, `ConsoleView`, `ExampleMessage`, `ShellControlV2`) target .NET 4.5.2 — is this intentional for maximum compatibility or an oversight?
- No `.vsconfig` found — is there a minimum Visual Studio version requirement for the WinForms designer used in `ConsoleView` / `ShellControlV2`?
