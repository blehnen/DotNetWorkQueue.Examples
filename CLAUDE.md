# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

Example applications for the [DotNetWorkQueue](https://github.com/blehnen/DotNetWorkQueue) library, demonstrating producers and consumers across multiple transports (SQL Server, PostgreSQL, SQLite, LiteDb, Redis). The README notes these are the *original* examples — the newer [DotNetWorkQueue.Samples](https://github.com/blehnen/DotNetWorkQueue.Samples) repo is more straightforward and is the preferred reference for simple usage.

**Frozen as of 2026-04:** the repo is pinned at DotNetWorkQueue 0.9.18 / .NET Framework 4.8 and receives no further updates. DotNetWorkQueue 0.9.19 dropped net48 in favor of .NET 10 / .NET 8 only — see the Samples repo above for modern usage.

## Build / run

- Solution: `Source/Examples/Examples.sln`
- **.NET Framework 4.8**, classic-style `.csproj` with `packages.config` (not SDK-style). Restore via `nuget restore` (not `dotnet restore`).
- Pinned at **DotNetWorkQueue 0.9.18** — the last upstream release that supports net48. Do not bump; 0.9.19 dropped the framework entirely.
- Build with MSBuild / Visual Studio, e.g. `msbuild Source/Examples/Examples.sln /t:Build /p:Configuration=Release`. On WSL, use the full Windows MSBuild path and convert the solution path via `wslpath -w`; `cmd.exe /c msbuild` does not work (not on PATH).
- No automated test suite in this repo — all projects are interactive WinForms console-emulator apps. To "run a single test" means launching one producer/consumer executable and driving it through its shell commands.
- Each runnable project has an `App.config` with an `appSettings` key `Connection` holding the transport connection string. Edit this before running (SQL Server/Postgres/Redis hosts are hardcoded to the original author's dev machine).

## High-level architecture

Every runnable project is a **WinForms host for an embedded console shell** (`ConsoleView.FormMain`, driven by `ShellControlV2`). `Program.Main` does nothing transport-specific — it just boots the form and passes `Assembly.GetExecutingAssembly()`, which the shell reflects over to discover `IConsoleCommand` implementations in the executing project's `Commands/` folder.

The repo is organized as a matrix of `{Transport} × {Producer, Consumer, ConsumerAsync}` projects on top of a small set of shared projects:

```
Source/Examples/
  Shared/
    ConsoleShared         — IConsoleCommand abstractions, shell plumbing
    ConsoleSharedCommands — transport-agnostic base classes: SharedSendCommands, SharedConsumeMessage<TInit>
    ConsoleView           — WinForms FormMain that hosts the shell
    ShellControlV2        — console-emulation WinForms control (modified from CodeProject article)
    ExampleMessage        — SimpleMessage DTO used by all producers/consumers
  SQLServer/ Redis/ SQLite/ PostGresSQL/ LiteDb/
    Producer/<Transport>Producer             — QueueCreation + SendMessage commands
    Consumer/<Transport>Consumer             — ConsumeMessage command (sync)
    Consumer/<Transport>ConsumerAsync        — ConsumeMessage command using shared task scheduler
```

**The per-transport Command classes are intentionally thin.** They inherit the shared base (`SharedConsumeMessage<SqlServerMessageQueueInit>`, `SharedSendCommands`, etc.) and wire in the transport's `*MessageQueueInit` type from `DotNetWorkQueue.Transport.*`. Transport-specific logic that *does* live in the per-transport project is the `QueueCreation` command, because each transport exposes different schema/creation options (SQL Server column types, Redis key layout, etc.).

Key DotNetWorkQueue types flowing through the examples:
- `QueueCreationContainer<TInit>` / `*MessageQueueCreation` — schema/queue provisioning
- `QueueContainer<TInit>` — producer factory
- `QueueConnection` — `{ConnectionString, QueueName}` tuple; the connection string comes from `App.config` appSettings `Connection`
- `IProducerBaseQueue`, `QueueMessage<SimpleMessage, IAdditionalMessageData>` — send path
- `ConsumerQueueTypes` enum — distinguishes plain/RPC/etc. queue variants when the same physical queue is reused

When adding or modifying a command, follow the existing pattern: put transport-agnostic behavior in `Shared/ConsoleSharedCommands` and keep per-transport files as subclasses that only supply the `TInit` type parameter and transport-specific creation options.

## Conventions specific to this repo

- **Do not convert projects to SDK-style / .NET Core.** The whole repo is deliberately pinned to .NET Framework 4.8 + `packages.config` + classic csproj. NuGet references use `<HintPath>..\..\..\packages\...</HintPath>` and assume the shared `Source/Examples/packages` folder.
- `SharedAssemblyInfo.cs` at the repo root is linked into every project; version/company info lives there, not in per-project `AssemblyInfo.cs`.
- Commands appear in the shell by class name. Public methods on an `IConsoleCommand` become shell subcommands (e.g. `QueueCreation CreateQueue myQueue`). `Help()` and `Example(command)` strings are what the user sees at runtime — keep them in sync when changing method signatures.
- Connection strings in committed `App.config` files point at the original author's LAN (`192.168.0.58`, etc.). Expect to edit them locally; don't treat them as secrets or "correct" defaults.
