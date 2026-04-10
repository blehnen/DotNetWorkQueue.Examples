# TESTING.md

## Overview

This repository has no automated test suite of any kind. There are no test projects, no test frameworks referenced, and no CI pipeline. All verification is manual: a developer launches a producer executable and a consumer executable side-by-side, types shell commands into the embedded WinForms console emulator, and visually confirms that messages flow between them.

## Metrics

| Metric | Value |
|--------|-------|
| Automated test projects | 0 |
| xUnit / NUnit / MSTest references in any file | 0 |
| CI configuration files (.github/, appveyor.yml, azure-pipelines.yml, .travis.yml) | 0 |
| Runnable executable projects | ~15 (5 transports × Producer + Consumer + ConsumerAsync) |
| Test framework packages in any packages.config | 0 |

## Findings

### No Automated Tests

A full-repo search for `xunit`, `nunit`, `mstest`, `[Fact]`, `[Test]`, `[TestMethod]`, and `[TestFixture]` returned zero matches in any `.cs`, `.csproj`, or `packages.config` file. The only hit was a single line in `.gitignore` (listing test result directories as patterns to ignore, which is a template artifact, not evidence of actual test projects).

There are no `*Test*`, `*Spec*`, or `*Tests*` directories or projects anywhere in the solution.

Evidence: `Source/Examples/Examples.sln` — 20 projects, none named with a test suffix.
Evidence: `.gitignore` — only reference to test tooling.

### No CI Pipeline

No continuous integration configuration files exist at any depth in the repository:

- No `.github/workflows/` directory.
- No `appveyor.yml`.
- No `azure-pipelines.yml`.
- No `.travis.yml`.
- No `Makefile` or `Dockerfile`.

The repository has no automated build or validation on push.

### How "Testing" Actually Works

Testing is entirely manual and interactive. The workflow is:

**Step 1 — Edit connection strings.**
Each project's `App.config` contains a single `appSettings` key `Connection`. This must be updated to point at a local instance of the target transport before running.

Evidence: `Source/Examples/LiteDb/Producer/LiteDbProducer/App.config` (line 4) — default value is `Filename=c:\temp\test.ldb;Connection=shared;`

**Step 2 — Launch the Producer executable.**
Build and run the producer project for the transport under test (e.g. `LiteDbProducer.exe`). A WinForms window opens containing an embedded console emulator (`ShellControlV2`). The window title identifies the transport and role (e.g. "LiteDB Producer").

Evidence: `Source/Examples/LiteDb/Producer/LiteDbProducer/Program.cs` (line 44) — `form.Text = "LiteDB Producer"`

**Step 3 — Create and configure a queue via the shell.**
Type commands at the shell prompt. The shell reflects over the executing assembly to find all `IConsoleCommand` implementations and dispatches typed input to their public methods. Typical producer setup sequence:

```
QueueCreation CreateQueue myqueue
SendMessage CreateQueue myqueue 0
SendMessage Send myqueue 10 100 false
```

The `QueueCreation` command provisions the queue schema in the transport. `SendMessage CreateQueue` creates the in-process producer object. `SendMessage Send` enqueues messages.

Evidence: `Source/Examples/Shared/ConsoleSharedCommands/Commands/SharedCommands.cs` — `Example()` methods show canonical command strings.
Evidence: `Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/QueueCreation.cs` — `Help()` and `Example()` document all subcommands.

**Step 4 — Launch the Consumer executable in a second window.**
Run the matching consumer project (e.g. `LiteDbConsumer.exe` or `LiteDbConsumerAsync.exe`) with the same `Connection` value. At the consumer shell:

```
ConsumeMessage CreateQueue myqueue 0
ConsumeMessage StartQueue myqueue
```

**Step 5 — Visual verification.**
Watch the consumer shell for log output confirming messages are being processed. The message handler logs `Processing Message {id}` and `Processed message {id}` via `IWorkerNotification.Log`. If Serilog is enabled (`ConsumeMessage EnableSerilog`), output appears on the console surface.

Evidence: `Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessage.cs` (lines 304–329) — `HandleMessages` method with `LogDebug` calls.

**Optional — Enable interceptors or metrics before sending:**

```
SendMessage EnableGzip
SendMessage EnableDes <key> <iv>
SendMessage EnableMetrics
SendMessage ViewMetrics
```

Evidence: `Source/Examples/Shared/ConsoleSharedCommands/Commands/SharedCommands.cs` (lines 47–124) — `EnableGzip`, `EnableDes`, `EnableMetrics`, `ViewMetrics` methods.

### Producer / Consumer Split

Each transport has up to three runnable projects:

| Project type | Purpose |
|---|---|
| `*Producer` | Creates queues and enqueues messages |
| `*Consumer` | Dequeues and processes messages synchronously |
| `*ConsumerAsync` | Dequeues using the async task-scheduler variant |

A complete manual test of a transport requires running at least the Producer and one Consumer variant in separate processes simultaneously, with the same `Connection` string pointing at the same transport instance.

[Inferred] There is no documented test matrix or checklist — which feature combinations (Gzip + DES, delayed messages, method queues) have been exercised is not recorded anywhere in the repo.

### Per-Transport Connection Prerequisites

Each transport requires a live backing service for end-to-end manual testing:

| Transport | Required service |
|---|---|
| SQL Server | SQL Server instance |
| PostgreSQL | PostgreSQL instance |
| SQLite | Local file path (no server needed) |
| LiteDb | Local file path (no server needed) |
| Redis | Redis instance |

The App.config defaults point at `192.168.0.58` (SQL Server, Postgres, Redis) — the original author's LAN address, which will not be reachable on other machines. LiteDb and SQLite use local file paths (`c:\temp\`) that require the directory to exist.

Evidence: `Source/Examples/LiteDb/Producer/LiteDbProducer/App.config` (line 4)
Evidence: `CLAUDE.md` (line 16) — "SQL Server/Postgres/Redis hosts are hardcoded to the original author's dev machine"

## Summary Table

| Item | Detail | Confidence |
|------|--------|------------|
| Automated test suite | None — zero test projects | Observed |
| xUnit / NUnit / MSTest | No references anywhere in repo | Observed |
| CI pipeline | No CI configuration files exist | Observed |
| Test mechanism | Manual shell commands in WinForms console emulator | Observed |
| Producer+Consumer manual loop | Two executables run side by side; visual output verified | Observed |
| Test coverage tracking | None | Observed |
| Documented test scenarios | None — no test plans or checklists committed | Observed |
| LiteDb / SQLite testable without a server | Yes — file-based transport | Observed |
| SQL Server / Postgres / Redis require live service | Yes — defaults point at author's LAN IP | Observed |

## Open Questions

- Is there a companion integration-test suite in the main `DotNetWorkQueue` library repo that exercises these same transports programmatically?
- Would it be feasible to add a headless/CLI mode to the existing command classes (bypassing WinForms) to enable automated smoke tests in CI?
- Which transport(s) are actively exercised before each release, and by whom?
