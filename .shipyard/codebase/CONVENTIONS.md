# CONVENTIONS.md

## Overview

All hand-authored C# files in this repo follow a uniform MIT license header block, use a reflection-driven command pattern built around `IConsoleCommand`, and share version metadata through a single root-level `SharedAssemblyInfo.cs`. The per-transport projects are intentionally thin subclasses; transport-agnostic logic lives in the `Shared/` projects. There are no linter config files or `.editorconfig` files present — conventions are enforced by example rather than tooling.

## Metrics

| Metric | Value |
|--------|-------|
| Total hand-authored .cs files (excl. obj/) | 90 |
| Projects in solution | 20 |
| Projects linking SharedAssemblyInfo.cs | 18 of 20 |
| Command classes per transport-project | 1–3 (QueueCreation, SendMessage, ConsumeMessage) |
| TODO/FIXME/HACK occurrences | 0 (grep found none) |
| Linter / .editorconfig files | 0 |

## Findings

### File Header Pattern

Every hand-authored .cs file opens with a fixed MIT license comment block. Two distinct variants exist:

- **ConsoleShared variant** — credits John Atten (original CodeProject shell author), uses `//` comment lines with no leading spaces on the body lines.
  - Evidence: `Source/Examples/Shared/ConsoleShared/IConsoleCommand.cs` (lines 1–27)
- **Brian Lehnen variant** — credits Brian Lehnen, copyright range `© 2015-2020`, uses `//` lines with a leading space on each body line.
  - Evidence: `Source/Examples/Shared/ConsoleSharedCommands/Commands/SharedCommands.cs` (lines 1–25)
  - Evidence: `Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/QueueCreation.cs` (lines 1–25)
  - Evidence: `Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/SendMessage.cs` (lines 1–25)
  - Evidence: `Source/Examples/LiteDb/Consumer/LiteDbConsumer/Commands/ConsumeMessage.cs` (lines 1–25)

Both variants are 25–27 lines long. The header ends with a `// ---------------------------------------------------------------------` separator line, followed immediately by `using` directives (no blank line between header and usings in the Lehnen variant).

### Shared Assembly Info

Version, company, and copyright information is centralized in `SharedAssemblyInfo.cs` at the repository root and linked into each project as a compile-time file link.

- Evidence: `SharedAssemblyInfo.cs` — sets `AssemblyCompany`, `AssemblyCopyright`, `AssemblyVersion` (0.7.1), `AssemblyFileVersion`, `AssemblyInformationalVersion`.
- Evidence: 18 of 20 `.csproj` files contain a `<Compile Include=` link to `SharedAssemblyInfo.cs`.
- Per-project `AssemblyInfo.cs` files contain only `AssemblyTitle` and `AssemblyProduct` (two attributes).
  - Evidence: `Source/Examples/LiteDb/Producer/LiteDbProducer/Properties/AssemblyInfo.cs` (lines 27–28)

### IConsoleCommand Interface

The shell command contract is defined in `ConsoleShared`:

```csharp
public interface IConsoleCommand
{
    ConsoleExecuteResult Info { get; }
    ConsoleExecuteResult Help();
    ConsoleExecuteResult Example(string command);
}
```

- Evidence: `Source/Examples/Shared/ConsoleShared/IConsoleCommand.cs` (lines 29–37)

Every command class implements this interface. `Info` is a one-line description used by the shell's command listing. `Help()` returns a multi-line formatted string built with `StringBuilder`. `Example(string command)` returns a sample invocation string for the named subcommand, dispatched via a `switch` statement.

### Command Class Structure

Command classes follow this pattern consistently across all transports:

1. Implement `IConsoleCommand` (and `IDisposable`).
2. Declare one or two `Lazy<T>` fields for deferred container initialization.
3. Declare a `Dictionary<string, T>` to hold per-queue-name state (creators or running queues).
4. Constructor initializes the `Lazy<T>` and the `Dictionary` — no other setup.
5. `Info` property returns a `ConsoleExecuteResult` with a `ConsoleFormatting.FixedLength(...)` formatted string.
6. `Help()` builds a `StringBuilder`, appends `ConsoleFormatting.FixedLength("MethodName args", "description")` lines, and returns.
7. `Example(string command)` is a `switch` over command name strings returning sample argument strings.
8. Public methods are the shell subcommands (e.g. `CreateQueue`, `RemoveQueue`, `Send`, `StartQueue`).
9. A private `CreateModuleIfNeeded(string queueName)` guard method checks the dictionary before creating a new entry.
10. `Dispose(bool disposing)` iterates the dictionary, disposes all values, clears it, then checks `_lazyField.IsValueCreated` before disposing the container.

Evidence (QueueCreation): `Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/QueueCreation.cs`
Evidence (SendMessage): `Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/SendMessage.cs`
Evidence (SharedConsumeMessage): `Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessage.cs`

### Lazy<T> and Dictionary Pattern

Every command class that manages DotNetWorkQueue objects uses:

```csharp
private readonly Lazy<QueueContainer<TInit>> _queueContainer;
private readonly Dictionary<string, IProducerBaseQueue> _queues;
```

The `Lazy<T>` defers container construction until the first queue operation. The dictionary maps queue names (typed by the user at the shell) to live queue instances. This pattern appears identically in:

- `QueueCreation` (producer): `Lazy<QueueCreationContainer<TInit>>` + `Dictionary<string, TMessageQueueCreation>`
  - Evidence: `Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/QueueCreation.cs` (lines 40–42)
- `SendMessage` (producer): `Lazy<QueueContainer<TInit>>` + `Dictionary<string, IProducerBaseQueue>`
  - Evidence: `Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/SendMessage.cs` (lines 47–49)
- `SharedConsumeMessage<T>` (consumer base): `Lazy<QueueContainer<TTransportInit>>` + `Dictionary<string, IConsumerBaseQueue>`
  - Evidence: `Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessage.cs` (lines 44–46)

### Configuration via App.config

All projects read the transport connection string from `App.config` `appSettings` key `Connection`:

```csharp
ConfigurationManager.AppSettings["Connection"]
```

Evidence: `Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessage.cs` (lines 292–293)
Evidence: `Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/QueueCreation.cs` (line 176)

The `App.config` files contain only this one `appSettings` key, plus `<startup>` and extensive `<assemblyBinding>` redirects. Connection strings committed in the repo point at the original author's dev machine (e.g. `Filename=c:\temp\test.ldb` for LiteDb, `192.168.0.58` for SQL Server / Postgres / Redis).

Evidence (LiteDb): `Source/Examples/LiteDb/Producer/LiteDbProducer/App.config` (line 4)

### Inheritance Hierarchy for Per-Transport Commands

Per-transport `ConsumeMessage` classes are minimal subclasses that supply only the transport `TInit` type parameter and override `Info`:

```csharp
public class ConsumeMessage : SharedConsumeMessage<LiteDbMessageQueueInit>
{
    public override ConsoleExecuteResult Info =>
        new ConsoleExecuteResult(ConsoleFormatting.FixedLength("ConsumeMessage", "Processes messages in a queue"));
}
```

Evidence: `Source/Examples/LiteDb/Consumer/LiteDbConsumer/Commands/ConsumeMessage.cs` (lines 32–35)

This is the thinnest possible subclass — 5 lines of body. All behavior comes from `SharedConsumeMessage<T>` in the shared project.

### Namespace Conventions

Namespaces follow project name, not folder path:

- Shared library: `ConsoleShared`, `ConsoleSharedCommands.Commands`
- Per-transport producers: `LiteDbProducer.Commands`, `RedisProducer.Commands`, etc.
- Per-transport consumers: `SQLiteConsumer.Commands` (note: LiteDb consumer uses `SQLiteConsumer` namespace — apparent copy-paste artifact)
- `Program.cs` files use the transport project namespace directly (e.g. `namespace SQLiteProducer`)

Evidence (namespace mismatch): `Source/Examples/LiteDb/Consumer/LiteDbConsumer/Commands/ConsumeMessage.cs` line 31 — `namespace SQLiteConsumer.Commands` in a LiteDb project.

### WinForms as Console Host (Intentional Pattern)

Every `Program.cs` is a WinForms `[STAThread]` entry point that opens `ConsoleView.FormMain`, passing `Assembly.GetExecutingAssembly()`. There is no `Console.WriteLine` usage in runnable projects — all output goes through the shell control.

```csharp
[STAThread]
static void Main()
{
    Application.EnableVisualStyles();
    Application.SetCompatibleTextRenderingDefault(false);
    using (var form = new ConsoleView.FormMain(Assembly.GetExecutingAssembly()))
    {
        form.Text = "LiteDB Producer";
        Application.Run(form);
    }
}
```

Evidence: `Source/Examples/LiteDb/Producer/LiteDbProducer/Program.cs` (lines 38–48)

The shell reflects over the executing assembly to discover all `IConsoleCommand` implementations, making the command set entirely determined by what classes are compiled into that executable.

### Formatting and Style

- Indentation: 4-space (tabs not observed in sampled files).
- Braces: Allman style (opening brace on its own line) for methods and classes; single-line braces for trivial `if` guards are sometimes used inline.
- `using` directives: not sorted alphabetically; order appears to be BCL first, then third-party, then project-local — but this is inconsistent and not enforced by tooling.
- String interpolation (`$"..."`) used throughout for result messages; no `string.Format` observed in newer code.
- Null-conditional operator (`?.`) used (e.g. `Metrics?.Dispose()`).
- XML doc comments (`/// <summary>`) present on `Dispose()` overloads consistently; absent elsewhere.

### Notable Oddity: Async Lock on StringBuilder

`SendMessage.SendAsync` uses a `private readonly object _asyncStringBuilderLock` to protect a `StringBuilder` in a `foreach` + `await` loop. This is unusual — the lock is per-instance and the method is `async`, so `Task.ConfigureAwait(false)` continuations could run on any thread pool thread. The pattern is functional but non-idiomatic.

Evidence: `Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/SendMessage.cs` (lines 50, 188–215)

## Summary Table

| Item | Detail | Confidence |
|------|--------|------------|
| MIT license header on every .cs | Present in all sampled files (14/14) | Observed |
| Two header variants (Atten vs Lehnen) | Differ in attribution and minor whitespace | Observed |
| SharedAssemblyInfo.cs linked in 18/20 projects | Sets version 0.7.1 | Observed |
| IConsoleCommand: Info + Help() + Example() | Defined in ConsoleShared; implemented by all command classes | Observed |
| Lazy<T> for deferred queue container | Used in all 3 command class types | Observed |
| Dictionary<string, T> for per-queue state | Used in all 3 command class types | Observed |
| App.config `appSettings/Connection` for connection string | Used in all transport projects | Observed |
| WinForms host for console shell | All Program.cs files are STAThread WinForms apps | Observed |
| Namespace mismatch in LiteDb consumer | Uses `SQLiteConsumer.Commands` — copy-paste artifact | Observed |
| No linter / .editorconfig | Conventions enforced by example only | Observed |
| 4-space indentation, Allman braces | Consistent across sampled files | Observed |

## Open Questions

- The 2 projects not linking `SharedAssemblyInfo.cs` — which projects are they and is this intentional?
- The `SQLiteConsumer.Commands` namespace in the LiteDb consumer: is this causing any runtime issues, or is it purely cosmetic (namespace is never used externally)?
- `ConsoleShared` targets `.NETFramework,Version=v4.5.2` (seen in its obj/ folder) while all other projects target 4.7.2 — was this intentional or an oversight?
