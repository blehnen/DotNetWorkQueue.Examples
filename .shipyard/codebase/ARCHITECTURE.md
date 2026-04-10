# ARCHITECTURE.md

## Overview

DotNetWorkQueue.Examples is a collection of interactive WinForms-hosted console-emulator applications demonstrating producer/consumer patterns against five transports (SQL Server, PostgreSQL, SQLite, LiteDb, Redis). Every runnable executable follows an identical startup pattern: `Program.Main` boots a WinForms form that reflects over the executing assembly to discover commands, then routes user input to those commands via reflection-based dispatch. Per-transport projects are intentionally thin — they consist almost entirely of a single-line class that supplies a transport-specific type parameter to a shared generic base.

## Metrics

| Metric | Value |
|--------|-------|
| Total .csproj files in solution | 20 (16 runnable + 4 shared libraries) |
| Shared library projects | 5 (ConsoleShared, ConsoleSharedCommands, ConsoleView, ShellControlV2, ExampleMessage) |
| Runnable transport projects | 15 (5 transports × 3 roles: Producer, Consumer, ConsumerAsync) — see matrix caveat below |
| Lines in thinnest per-transport command | 5 lines of logic (SqlServerConsumer `ConsumeMessage.cs`) |
| Lines in thickest shared base | ~330 lines (`ConsumeMessageAsync<TTransportInit>`) |

## Findings

### Shared-Base / Per-Transport Inheritance Pattern

Every transport's consumer and async-consumer command is a one-line subclass that supplies the transport's `*MessageQueueInit` type as the generic parameter to a shared base class:

- **`SharedConsumeMessage<TTransportInit>`** — base for all synchronous consumers (`Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessage.cs`)
- **`ConsumeMessageAsync<TTransportInit>`** — base for all async/scheduler consumers (`Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessageAsync.cs`)
- **`SharedSendCommands`** — non-generic abstract base for all producers; declares `protected abstract Dictionary<string, IProducerBaseQueue> Queues` (`Source/Examples/Shared/ConsoleSharedCommands/Commands/SharedSendCommands.cs:33`)

Both consumer base classes themselves inherit `SharedCommands`, which provides cross-cutting concerns: GZip/TripleDES message interceptors (`EnableGzip`, `EnableDes`), App.Metrics integration (`EnableMetrics`, `ViewMetrics`), and `IDisposable` teardown (`Source/Examples/Shared/ConsoleSharedCommands/Commands/SharedCommands.cs`).

Example — the entire SqlServer sync consumer command file:
```csharp
// Source/Examples/SQLServer/Consumer/SqlServerConsumer/Commands/ConsumeMessage.cs
public class ConsumeMessage : SharedConsumeMessage<SqlServerMessageQueueInit>
{
    public override ConsoleExecuteResult Info => new ConsoleExecuteResult(...);
}
```

The `QueueCreation` command is the **exception**: it is not shared. Each transport has its own `QueueCreation` class implementing `IConsoleCommand` directly (not inheriting a shared base), because transport-specific schema options differ significantly. SQL Server exposes `EnableHoldTransactionUntilMessageCommitted`, `EnablePriority`, `EnableStatus`, `EnableStatusTable`, and full DDL column/constraint customization (`Source/Examples/SQLServer/Producer/SqlServerProducer/Commands/QueueCreation.cs:63-78`). LiteDb exposes a smaller option set with no column/constraint customization (`Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/QueueCreation.cs:61-67`).

### Reflection-Driven Command Discovery

The command discovery pipeline is:

```
Program.Main(Assembly.GetExecutingAssembly())
  -> new ConsoleView.FormMain(assembly)
       -> FormMain_Load
            -> ConsoleGetCommands.GetCommands(assembly)
                 -> reflects over ConsoleShared assembly: finds IConsoleCommand implementations -> default commands
                 -> reflects over executing assembly: finds IConsoleCommand implementations -> transport commands
                 -> for each command class: Activator.CreateInstance(type)
                 -> for each public instance method returning ConsoleExecuteResult: registers as subcommand
                 -> detects async methods via AsyncStateMachineAttribute
  -> shellControl1.SetCommands(list)  // populates tab-completion
  -> ShellControl1OnCommandEntered    // dispatches on user input
       -> ConsoleExecute.IsAsync(cmd) -> routes to ExecuteAsync or Execute
       -> typeInfo.InvokeMember(command.Name, ..., inputArgs) // reflection invocation
```

Evidence:
- `Source/Examples/Shared/ConsoleShared/ConsoleGetCommands.cs:14-44` — two-pass reflection: first the ConsoleShared assembly (default commands), then the passed-in `assembly` (transport-specific commands)
- `Source/Examples/Shared/ConsoleShared/ConsoleGetCommands.cs:54` — `Activator.CreateInstance(commandClass)` — commands are instantiated with no constructor arguments
- `Source/Examples/Shared/ConsoleShared/ConsoleGetCommands.cs:74-78` — async detection via `AsyncStateMachineAttribute`
- `Source/Examples/Shared/ConsoleView/FormMain.cs:56` — `_commandLibraries = ConsoleGetCommands.GetCommands(_commandAssembly)`
- `Source/Examples/Shared/ConsoleView/FormMain.cs:104-126` — async/sync routing and reflection invocation
- `Source/Examples/SQLServer/Producer/SqlServerProducer/Program.cs:41` — `new ConsoleView.FormMain(Assembly.GetExecutingAssembly())`

Commands appear in the shell prefixed by their class name (namespace). The user types `ConsumeMessage.StartQueue examplequeue`. Public methods on the class become the subcommands; `Help()` and `Example(command)` are special by convention.

### Three Producer/Consumer Variants

**Producer** (e.g. `SqlServerProducer`)
- Commands: `QueueCreation` (transport-specific DDL) + `SendMessage` (extends `SharedSendCommands`)
- `SendMessage` creates a `QueueContainer<SqlServerMessageQueueInit>`, calls `_queueContainer.Value.CreateProducer<SimpleMessage>(queue)` to get `IProducerQueue<SimpleMessage>`, then calls `.Send(message, data)` or `.SendAsync(...)` (`Source/Examples/SQLServer/Producer/SqlServerProducer/Commands/SendMessage.cs:296-305`)
- Optional message interceptors (GZip, TripleDES) registered into the container at creation time (`SendMessage.cs:67-97`)

**Consumer** (e.g. `SqlServerConsumer`)
- Single command: `ConsumeMessage : SharedConsumeMessage<SqlServerMessageQueueInit>`
- `CreateModuleIfNeeded` calls `_queueContainer.Value.CreateConsumer(queueConnection)` → `IConsumerQueue`
- `StartQueue` calls `consumerQueue.Start<SimpleMessage>(HandleMessages, notifications)` — the library's built-in thread pool drives message delivery to `HandleMessages` (`Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessage.cs:229-242`)
- Worker configuration (thread count, heartbeat, retry, expiration) is set via console commands before `StartQueue`

**ConsumerAsync** (e.g. `SqlServerConsumerAsync`)
- Single command: `ConsumeMessage : ConsumeMessageAsync<SqlServerMessageQueueInit>`
- Key difference: introduces a **`SchedulerContainer`** alongside `QueueContainer`. Creates `ATaskScheduler` + `ITaskFactory` via `_schedulerContainer.Value.CreateTaskScheduler()` / `CreateTaskFactory(_taskScheduler)` (`Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessageAsync.cs:329-333`)
- `CreateModuleIfNeeded` calls `_queueContainer.Value.CreateConsumerQueueScheduler(queueConnection, _taskFactory)` instead of `CreateConsumer` — this binds the consumer to the shared task scheduler rather than DotNetWorkQueue's internal thread pool
- `StartQueue` calls `_taskScheduler.Start()` first, then `consumerQueue.Start<SimpleMessage>(HandleMessages, notifications)` via `IConsumerQueueScheduler` interface (`ConsumeMessageAsync.cs:260-273`)
- Supports work groups: `AddWorkGroup(queueConnection, type, workGroupName, concurrencyLevel)` calls `_taskScheduler.AddWorkGroup(...)` — this is absent in the sync consumer variant
- The `HandleMessages` callback is identical in both variants; the difference is entirely in wiring, not message processing logic

### Message Flow

```
[Producer side]
  SimpleMessage { Message: string, RunTimeInMs: int }   (ExampleMessage project)
      |
      v
  new QueueMessage<SimpleMessage, IAdditionalMessageData>(msg, data)
      |  data may carry: delay, expiration, priority (AdditionalMessageData.SetDelay/SetExpiration/SetPriority)
      v
  IProducerQueue<SimpleMessage>.Send(message, data)  -- or .SendAsync(), or batch .Send(IEnumerable)
      |
      v
  DotNetWorkQueue transport (SQL Server / Redis / SQLite / LiteDb / PostgreSQL)
      |
      | (queue stored in transport)
      v

[Consumer side -- sync]
  IConsumerQueue.Start<SimpleMessage>(HandleMessages, IWorkerNotification)
      |  DotNetWorkQueue internal thread pool polls transport
      v
  HandleMessages(IReceivedMessage<SimpleMessage> message, IWorkerNotification notifications)
      |  accesses message.Body.RunTimeInMs, message.MessageId
      |  honours notifications.WorkerStopping.CancelWorkToken for graceful shutdown
      v
  (message acknowledged / rolled back by transport)

[Consumer side -- async/scheduler variant]
  ATaskScheduler.Start()
  IConsumerQueueScheduler.Start<SimpleMessage>(HandleMessages, IWorkerNotification)
      |  DotNetWorkQueue feeds work items into the shared ATaskScheduler / ITaskFactory
      v
  (same HandleMessages callback)
```

Evidence: `Source/Examples/Shared/ExampleMessage/SimpleMessage.cs`, `Source/Examples/SQLServer/Producer/SqlServerProducer/Commands/SendMessage.cs:246-277`, `Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessage.cs:303-330`, `Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessageAsync.cs:378-405`

### Container / Dependency Injection Pattern

DotNetWorkQueue uses its own `IContainer` abstraction (not Microsoft.Extensions.DI at the application level). The examples register optional services into it via a `RegisterService(IContainer container)` callback passed to `QueueContainer<TInit>` at construction. This is the only DI customization point exposed to example code. The `SchedulerContainer` used by async consumers accepts its own parallel `RegisterSchedulerService` callback. Both containers are created lazily (`Lazy<T>`) and disposed on `StopQueue` or form close.

Evidence: `Source/Examples/SQLServer/Producer/SqlServerProducer/Commands/SendMessage.cs:61-98`, `Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessageAsync.cs:296-325`

### Macro System

`FormMain` supports a lightweight macro record/replay system: `ConsoleExecute.StartMacroCapture()` records subsequent commands to a file under `<assembly-dir>\Macro\`, and `RunMacro` replays them by injecting commands back into `shellControl1.SendCommand(...)`. This is transport-agnostic and lives entirely in `ConsoleShared`/`ConsoleView`.

Evidence: `Source/Examples/Shared/ConsoleView/FormMain.cs:153-173`

## Summary Table

| Item | Detail | Confidence |
|------|--------|------------|
| WinForms shell host | Every executable is `FormMain` + `ShellControlV2` | Observed |
| Command discovery | Reflection over `IConsoleCommand` types; public methods returning `ConsoleExecuteResult` become subcommands | Observed |
| Async detection | `AsyncStateMachineAttribute` on method | Observed |
| Per-transport consumer is 1 class | `class ConsumeMessage : SharedConsumeMessage<XxxMessageQueueInit>` — body is just `Info` property | Observed |
| Consumer vs ConsumerAsync wiring | Sync: `CreateConsumer` + `IConsumerQueue.Start`; Async: `CreateConsumerQueueScheduler` + `ATaskScheduler` + `IConsumerQueueScheduler.Start` | Observed |
| QueueCreation not shared | Each transport has own `QueueCreation : IConsoleCommand` due to transport-specific DDL options | Observed |
| SendMessage partially shared | `SharedSendCommands` declares abstract `Queues` dict; actual send logic in per-transport `SendMessage` | Observed |
| Message DTO | `SimpleMessage { Message: string, RunTimeInMs: int }` from `ExampleMessage` project | Observed |
| DI registration | Optional callback to `QueueContainer<TInit>` constructor; no Microsoft.Extensions.DI wiring | Observed |
| Connection string source | `ConfigurationManager.AppSettings["Connection"]` from `App.config` in each runnable project | Observed |

## Open Questions

- `SharedSendCommands.SendMethod` returns `"not implemented"` — it appears to be a stub for Linq expression / method queue sends. Whether this was intentionally left unimplemented or is a genuine gap is unclear.
- `SimpleResponse.cs` exists in `ExampleMessage` but no consumer or producer example appears to use RPC response patterns in the current code — its role is unclear without a deeper search.
- The `ConsoleExecutingAssembly.Namespace` field is used as the command prefix in the shell (e.g. `ConsumeMessage.StartQueue`), but the name `Namespace` is misleading — it is the class name, not a .NET namespace. This may confuse future contributors.
