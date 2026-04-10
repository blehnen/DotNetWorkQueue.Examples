# STRUCTURE.md

## Overview

The repository has a single solution (`Source/Examples/Examples.sln`) containing 20 projects: 5 shared libraries and 15 runnable transport-specific executables. All runnable projects are WinForms executables targeting .NET Framework 4.7.2. The directory tree mirrors the logical matrix of `{Transport} × {Role}`, with shared infrastructure collected under `Source/Examples/Shared/`.

## Metrics

| Metric | Value |
|--------|-------|
| Solution file | `Source/Examples/Examples.sln` |
| Total .csproj references in solution | 20 |
| Shared library projects | 5 |
| Transport folders | 5 (LiteDb, PostGresSQL, Redis, SQLServer, SQLite) |
| Runnable projects | 15 (Producer + Consumer + ConsumerAsync per transport) |
| .cs source files (excl. bin/obj) | ~75 |
| Unique NuGet packages (across all packages.config) | ~50+ |

## Findings

### Repository Root Layout

```
/
├── .gitignore
├── CLAUDE.md                        — AI assistant guidance (architecture summary, conventions)
├── LICENSE
├── README.md
├── SharedAssemblyInfo.cs            — Linked into every project; holds version/company attributes
└── Source/
    └── Examples/
        ├── Examples.sln
        ├── packages/                — Shared NuGet package restore folder (all HintPaths resolve here)
        ├── Shared/                  — 5 shared library projects (see below)
        ├── SQLServer/
        ├── PostGresSQL/
        ├── SQLite/
        ├── LiteDb/
        └── Redis/
```

Evidence: `SharedAssemblyInfo.cs` (repo root); `Source/Examples/Examples.sln`

### Shared Projects

All under `Source/Examples/Shared/`:

| Project | Assembly | Purpose |
|---------|----------|---------|
| `ConsoleShared` | `ConsoleShared.dll` | Core shell infrastructure: `IConsoleCommand` interface, `ConsoleGetCommands` (reflection discovery), `ConsoleExecute` (reflection dispatch), `ConsoleCommand` (input parser), `ConsoleMacro` (macro record/replay), `ConsoleFormatting`, `ConsoleExecuteResult`, `ConsoleParseArgument` |
| `ConsoleSharedCommands` | `ConsoleSharedCommands.dll` | Transport-agnostic base command classes: `SharedCommands` (metrics/interceptors base), `SharedSendCommands` (producer abstract base), `SharedConsumeMessage<TInit>` (sync consumer base), `ConsumeMessageAsync<TInit>` (scheduler consumer base), `CreateNotifications` |
| `ConsoleView` | `ConsoleView.dll` | WinForms `FormMain` that accepts `Assembly`, calls `ConsoleGetCommands`, hosts `ShellControlV2`, wires up `LogControl` and `QueueStatusControl` |
| `ShellControlV2` | `ShellControlV2.dll` | Third-party (CodeProject/MIT) console-emulation WinForms control; provides tab-completion, command history, `CommandEntered` event |
| `ExampleMessage` | `ExampleMessage.dll` | DTOs shared by all producers and consumers: `SimpleMessage { Message, RunTimeInMs }`, `SimpleResponse` |

Evidence: `Source/Examples/Shared/ConsoleShared/IConsoleCommand.cs`, `Source/Examples/Shared/ConsoleSharedCommands/Commands/SharedSendCommands.cs`, `Source/Examples/Shared/ConsoleView/FormMain.cs`, `Source/Examples/Shared/ExampleMessage/SimpleMessage.cs`

### Transport × Role Matrix

All 15 combinations exist. Each transport folder follows the same internal layout:

```
{Transport}/
├── Producer/
│   └── {Transport}Producer/
│       ├── Program.cs
│       ├── App.config                    — appSettings key "Connection"
│       ├── {Transport}Producer.csproj
│       ├── packages.config
│       ├── Properties/AssemblyInfo.cs    — links SharedAssemblyInfo.cs
│       └── Commands/
│           ├── QueueCreation.cs          — transport-specific DDL (implements IConsoleCommand directly)
│           └── SendMessage.cs            — extends SharedSendCommands
└── Consumer/
    ├── {Transport}Consumer/
    │   ├── Program.cs
    │   ├── App.config
    │   ├── Commands/ConsumeMessage.cs    — : SharedConsumeMessage<{Transport}MessageQueueInit>
    │   └── ...
    └── {Transport}ConsumerAsync/
        ├── Program.cs
        ├── App.config
        ├── Commands/ConsumeMessage.cs    — : ConsumeMessageAsync<{Transport}MessageQueueInit>
        └── ...
```

Confirmed projects per transport:

| Transport | Folder name | Producer | Consumer (sync) | Consumer (async) |
|-----------|-------------|----------|-----------------|------------------|
| SQL Server | `SQLServer` | `SqlServerProducer` | `SqlServerConsumer` | `SqlServerConsumerAsync` |
| PostgreSQL | `PostGresSQL` | `PostGresSQLProducer` | `PostGresSQLConsumer` | `PostGreSQLConsumerAsync` |
| SQLite | `SQLite` | `SQLiteProducer` | `SQLiteConsumer` | `SQLiteConsumerAsync` |
| LiteDb | `LiteDb` | `LiteDbProducer` | `LiteDbConsumer` | `LiteDbConsumerAsync` |
| Redis | `Redis` | `RedisProducer` | `RedisConsumer` | `RedisConsumerAsync` |

All 15 combinations are present. No transport is missing a variant.

Evidence: confirmed by direct file listing — all 15 `.csproj` files present under their respective transport/role directories.

### Naming Conventions

**Folder casing:**
- Transport folders use mixed casing matching the transport brand: `SQLServer`, `PostGresSQL`, `SQLite`, `LiteDb`, `Redis`
- Role folders: `Producer/`, `Consumer/` (title case)
- Project sub-folder names match the project/assembly name exactly

**Project names:**
- Producer: `{Transport}Producer` — e.g. `SqlServerProducer`, `LiteDbProducer`
- Consumer (sync): `{Transport}Consumer` — e.g. `SqlServerConsumer`
- Consumer (async): `{Transport}ConsumerAsync` — e.g. `SqlServerConsumerAsync`
- Notable inconsistency: the PostgreSQL folder is `PostGresSQL` (mixed case) but the project names are `PostGresSQLProducer`, `PostGresSQLConsumer`, `PostGreSQLConsumerAsync` (the async variant drops the second capital S in `SQL`)

Evidence: `Source/Examples/PostGresSQL/Consumer/PostGreSQLConsumerAsync/PostGreSQLConsumerAsync.csproj` vs `Source/Examples/PostGresSQL/Consumer/PostGresSQLConsumer/PostGreSQLConsumer.csproj`

**Namespace alignment:**
- Shared libraries use namespace = assembly name: `ConsoleShared`, `ConsoleSharedCommands.Commands`, `ConsoleView`, `ExampleMessage`
- Per-transport projects use namespace = project name: `SqlServerProducer.Commands`, `LiteDbConsumer.Commands`, etc.

**Command class names:**
- Always `ConsumeMessage` for consumer commands (both sync and async variants)
- Always `SendMessage` for the producer send command
- Always `QueueCreation` for the producer schema command
- These names are also the shell prefix the user types (e.g. `ConsumeMessage.StartQueue`)

### Solution File and Project References

- Solution: `Source/Examples/Examples.sln`
- Contains 20 `.csproj` references (confirmed by grep count)
- NuGet restore: classic `packages.config` style; all packages restore to `Source/Examples/packages/` shared folder
- HintPaths in `.csproj` files use relative paths like `..\..\..\packages\DotNetWorkQueue.x.y.z\lib\net472\DotNetWorkQueue.dll`
- `SharedAssemblyInfo.cs` is added as a linked file (`<Compile Include="..\..\..\..\SharedAssemblyInfo.cs">`) in each project's `Properties/AssemblyInfo.cs` — per-project `AssemblyInfo.cs` contains only assembly-specific attributes (title, GUID)

Evidence: `Source/Examples/SQLServer/Producer/SqlServerProducer/SqlServerProducer.csproj` (HintPath pattern), `Source/Examples/SQLServer/Producer/SqlServerProducer/Properties/AssemblyInfo.cs` (SharedAssemblyInfo link)

### Entry Points

Every runnable project has an identical `Program.cs`:

```csharp
[STAThread]
static void Main()
{
    Application.EnableVisualStyles();
    Application.SetCompatibleTextRenderingDefault(false);
    using (var form = new ConsoleView.FormMain(Assembly.GetExecutingAssembly()))
    {
        form.Text = "<Transport> <Role>";  // e.g. "Sql Producer"
        Application.Run(form);
    }
}
```

The only per-project variation is the `form.Text` title string. All transport-specific behaviour is in the `Commands/` folder classes, discovered at runtime by reflection.

Evidence: `Source/Examples/SQLServer/Producer/SqlServerProducer/Program.cs:38-46`

### Configuration Hierarchy

- No shared/inherited config. Each runnable project has its own `App.config` with:
  ```xml
  <appSettings>
    <add key="Connection" value="..." />
  </appSettings>
  ```
- Connection strings in committed files point to the original author's dev environment (`192.168.0.58` etc.) — must be edited locally before use
- No environment variable substitution or config transforms are present [Inferred: no `.config` transform files observed in project directories]

Evidence: `CLAUDE.md` (connection string note); `Source/Examples/LiteDb/Consumer/LiteDbConsumer/App.config`

## Summary Table

| Item | Detail | Confidence |
|------|--------|------------|
| Solution location | `Source/Examples/Examples.sln` | Observed |
| Project count | 20 (5 shared + 15 runnable) | Observed |
| Transport matrix | 5 transports × 3 roles = 15, all present | Observed |
| Shared code location | `Source/Examples/Shared/` | Observed |
| Version info location | `SharedAssemblyInfo.cs` at repo root, linked into all projects | Observed |
| NuGet style | `packages.config` + shared `packages/` folder | Observed |
| PostgreSQL naming inconsistency | Folder: `PostGresSQL`; async project drops one S vs sync | Observed |
| Entry point pattern | Identical `Program.cs` in all 15 runnable projects | Observed |
| Per-project config | `App.config` with `Connection` appSetting, not shared | Observed |
| Command subfolder | `Commands/` under each runnable project root | Observed |

## Open Questions

- No `packages/` folder content is committed to git (`.gitignore` excludes it). First-time builds require `nuget restore` against NuGet.org — there is no private feed or vendored packages directory in the repo.
- The `SimpleResponse` DTO in `ExampleMessage` is not used by any current example project. It may be a placeholder for RPC examples that were never added.
- `Source/Examples/.vs/` directory is present in the repo (Visual Studio IDE state) — it appears `.gitignore` does not exclude it fully, which may cause spurious diffs for contributors using different VS versions.
