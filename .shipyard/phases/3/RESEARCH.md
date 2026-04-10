# Phase 3 Research — Package Bump + AppMetrics Strip

Research performed directly in the main planning context (the researcher agent timed out before writing a file).

## R9: AppMetrics code-shape — STRIP IS BOUNDED

The AppMetrics wiring is concentrated in **one file**. The other 7 files each carry only a 3-line DI registration block.

### `Shared/ConsoleSharedCommands/Commands/SharedCommands.cs` — the load-bearing file

- Line 33: `using App.Metrics;`
- Line 39: `protected DotNetWorkQueue.AppMetrics.Metrics Metrics;`
- Line 40: `private App.Metrics.IMetricsRoot _metricsRoot;`
- Line 95–99 (cleanup / disposal path): a `if (Metrics != null)` block that calls `_metricsRoot.ReportRunner.RunAllAsync()` and awaits the tasks.
- Line 105–123 (enable-metrics method): the `_metricsRoot = new App.Metrics.MetricsBuilder()...Build()` + `Metrics = new DotNetWorkQueue.AppMetrics.Metrics(_metricsRoot)` constructor chain. Presumably wrapped in a public `EnableMetrics()` or similar method.

**Strip scope for this file:** ~25 lines of code. Delete the using, the two field declarations, the cleanup block, and the entire enable-metrics method. Verify no orphaned references to `Metrics` or `_metricsRoot` remain.

### `Shared/ConsoleSharedCommands/Commands/ConsumeMessage.cs` and `ConsumeMessageAsync.cs`

Each has ONE occurrence (lines ~272 and ~307/315 respectively):

```csharp
if (Metrics != null)
{
    container.Register<IMetrics>(() => Metrics, LifeStyles.Singleton);
}
```

That's it. 4 lines to delete from each file. The `Metrics` field reference disappears when `SharedCommands.cs` is stripped (these files inherit from it).

### 5 per-transport `SendMessage.cs` files

Identical pattern. One block around line 68:

```csharp
if (Metrics != null)
{
    container.Register<IMetrics>(() => Metrics, LifeStyles.Singleton);
}
```

4 lines to delete per file × 5 transports = 20 lines.

### Code strip total

| File | Lines to remove |
|---|---|
| `Shared/ConsoleSharedCommands/Commands/SharedCommands.cs` | ~25 |
| `Shared/ConsoleSharedCommands/Commands/ConsumeMessage.cs` | ~4 |
| `Shared/ConsoleSharedCommands/Commands/ConsumeMessageAsync.cs` | ~4 |
| 5× `{Transport}Producer/Commands/SendMessage.cs` | ~20 (4 each) |
| **Total** | **~53 lines** |

Every deletion is mechanical (whole-block removal, no re-plumbing required). The `IMetrics` parameter-injection pattern IS NOT present — metrics are stored on the shared base class as a field, then consumed by derived classes via inheritance. Stripping the shared field removes the reference chain. Clean.

**Per-transport `SendMessage.cs` `using App.Metrics;` directives** — each of the 5 files has one at the top. Strip alongside the registration block.

## R6/R8: csproj Reference shape (canonical: `SqlServerConsumer.csproj`)

Every runnable csproj carries the same pattern. The App.Metrics block and the DNWQ block are contiguous and easy to identify by element name.

### App.Metrics references (6 `<Reference>` blocks per csproj)

```xml
<Reference Include="App.Metrics, Version=4.3.0.0, Culture=neutral, PublicKeyToken=0d5193a913d1b812, processorArchitecture=MSIL">
  <HintPath>..\..\..\packages\App.Metrics.4.3.0\lib\net461\App.Metrics.dll</HintPath>
</Reference>
<Reference Include="App.Metrics.Abstractions, Version=4.3.0.0, Culture=neutral, PublicKeyToken=0d5193a913d1b812, processorArchitecture=MSIL">
  <HintPath>..\..\..\packages\App.Metrics.Abstractions.4.3.0\lib\net461\App.Metrics.Abstractions.dll</HintPath>
</Reference>
<Reference Include="App.Metrics.Concurrency, ...">
<Reference Include="App.Metrics.Core, ...">
<Reference Include="App.Metrics.Formatters.Ascii, ...">
<Reference Include="App.Metrics.Formatters.Json, ...">
```

**Action:** delete all 6 `<Reference>` blocks whose `Include` attribute starts with `App.Metrics`. 17 csproj × 6 blocks = **102 blocks to delete**.

### DotNetWorkQueue references (variable per csproj, ~5-6 blocks)

```xml
<Reference Include="DotNetWorkQueue, Version=0.8.0.0, Culture=neutral, processorArchitecture=MSIL">
  <HintPath>..\..\..\packages\DotNetWorkQueue.0.8.0\lib\netstandard2.0\DotNetWorkQueue.dll</HintPath>
</Reference>
<Reference Include="DotNetWorkQueue.AppMetrics, Version=0.8.0.0, Culture=neutral, processorArchitecture=MSIL">
  <HintPath>..\..\..\packages\DotNetWorkQueue.AppMetrics.0.8.0\lib\netstandard2.0\DotNetWorkQueue.AppMetrics.dll</HintPath>
</Reference>
<Reference Include="DotNetWorkQueue.Transport.RelationalDatabase, Version=0.8.0.0, ...">
<Reference Include="DotNetWorkQueue.Transport.Shared, Version=0.8.0.0, ...">
<Reference Include="DotNetWorkQueue.Transport.SqlServer, Version=0.8.0.0, ...">
```

**Three transforms needed:**

1. **Delete** the `DotNetWorkQueue.AppMetrics` reference block entirely (17 csproj × 1 = 17 blocks).
2. **Bump version** in all remaining DNWQ `<Reference>` `Include` attributes: `Version=0.8.0.0` → `Version=0.9.18.0`.
3. **Bump HintPath** to point at 0.9.18 and the `net48` lib folder:
   - `packages\DotNetWorkQueue.0.8.0\lib\netstandard2.0\` → `packages\DotNetWorkQueue.0.9.18\lib\net48\`
   - Same for each transport package (`DotNetWorkQueue.Transport.*`).

**HintPath subfolder migration (netstandard2.0 → net48):** This is important for correctness. The 0.9.18 packages ship both `lib\net48\` and `lib\netstandard2.0\`, and NuGet would prefer the closer match. Pointing HintPaths at `lib\net48\` explicitly avoids any resolution surprises.

### Python ElementTree transform is viable

Sanity test ran successfully:
```
Total Reference: 88
App.Metrics refs: 6
DNWQ refs: 5
```

Python 3 is installed on WSL. `xml.etree.ElementTree` handles the MSBuild namespace (`http://schemas.microsoft.com/developer/msbuild/2003`). The architect's plan should use a single Python transform script for csproj editing, not per-file `Edit` tool calls.

**Namespace caveat:** all XPath queries must use the namespace prefix (e.g., `.//m:Reference` not `.//Reference`). Writing out the tree preserves the namespace by default if the parser reads it.

## R8: App.config binding-redirect shape (representative: `SqlServerConsumer/App.config`)

```xml
<dependentAssembly>
  <assemblyIdentity name="App.Metrics.Abstractions" publicKeyToken="0d5193a913d1b812" culture="neutral" />
  <bindingRedirect oldVersion="0.0.0.0-4.2.0.0" newVersion="4.2.0.0" />
</dependentAssembly>
<dependentAssembly>
  <assemblyIdentity name="App.Metrics.Core" ... />
<dependentAssembly>
  <assemblyIdentity name="App.Metrics.Concurrency" ... />
<dependentAssembly>
  <assemblyIdentity name="App.Metrics.Formatters.Ascii" ... />
```

**Compile vs runtime:** binding redirects in App.config are **runtime-only**. They do not affect MSBuild compilation. Stripping them is a hygiene task that is not required for a clean build, but it's a cheap addition to the Python transform script — since the script is already parsing XML, deleting these blocks is a few extra lines.

**Decision for the plan:** strip App.config AppMetrics binding redirects in the same transform pass. They have zero post-strip value and leaving them dangling looks sloppy in the final frozen state.

## DNWQ 0.9.18 library-side facts (nuspec)

Retrieved via `powershell.exe Invoke-WebRequest` against `https://api.nuget.org/v3-flatcontainer/dotnetworkqueue/0.9.18/dotnetworkqueue.nuspec`.

### .NET Framework 4.8 dependency group

```
GuerrillaNtp 3.1.0
Microsoft.CSharp 4.7.0
Microsoft.Extensions.Caching.Memory 9.0.3
Microsoft.IO.RecyclableMemoryStream 3.0.1
Newtonsoft.Json 13.0.4
OpenTelemetry 1.14.0
Polly.Core 8.6.5
SimpleInjector 5.5.0
System.Diagnostics.DiagnosticSource 10.0.1
```

### Notable:

- **NO `App.Metrics` dependency.** Confirmed upstream removal — matches the changelog (0.9.1 removed it). Phase 3's strip is necessary, not optional.
- **`Polly.Core 8.6.5`** (not Polly 7.x). This is the transitive replacement for Phase 2-era Polly.
- **`OpenTelemetry 1.14.0`** is new transitively — the examples did not reference it directly before. It will be pulled in automatically by restore.
- **`Microsoft.Extensions.Caching.Memory 9.0.3`** and **`System.Diagnostics.DiagnosticSource 10.0.1`** imply Microsoft.Extensions.* 9.0.x / 10.0.x modern graph. Restore will download a LOT of new packages — expect `Source/Examples/packages/` to grow substantially from the current 0.8.0 + AppMetrics cache.

### Predicted restore behavior

1. ~30+ new packages will download (Microsoft.Extensions.*, OpenTelemetry, Polly 8, etc.).
2. Existing 0.8.0 DNWQ folders stay in `Source/Examples/packages/` unless manually purged (matches CONTEXT-3.md decision to `rm -rf` the cache first).
3. Build should succeed IF:
   - No example code directly references a type from the removed AppMetrics packages (beyond what Phase 3 already strips).
   - No example code directly references a Polly v7 type (see Polly v7 straggler analysis below).

## Polly v7 straggler — STILL PRESENT, 37 entries across 16 packages.config files

Grep count by project:

| Project | Polly v7 entries |
|---|---|
| `Shared/ConsoleSharedCommands` | 1 |
| `LiteDb/Consumer/LiteDbConsumer` | 1 |
| `LiteDb/Consumer/LiteDbConsumerAsync` | 1 |
| `LiteDb/Producer/LiteDbProducer` | 1 |
| `Redis/Consumer/RedisConsumer` | 2 |
| `Redis/Consumer/RedisConsumerAsync` | 2 |
| `Redis/Producer/RedisProducer` | 2 |
| 5× SQLServer+SQLite+PostGresSQL runnable projects | 3 each = 15 |
| **Total** | **37** |

The packages referenced are `Polly.Caching.Memory 3.0.2`, `Polly.Contrib.Simmy 0.3.0`, `Polly.Contrib.WaitAndRetry 1.1.1` — all Polly v7-era extensions that don't exist in the Polly v8 ecosystem.

### Risk assessment

- **If example code directly calls into these packages** (e.g., `using Polly.Contrib.WaitAndRetry;` or `new MemoryCacheProvider(...)`), removing them breaks the build and Phase 3 must fix or roll back.
- **If they were only transitive inherits from DNWQ 0.8.0 that got pinned at some point**, removing them is safe and makes the graph consistent with the Polly 8 transitive from 0.9.18.

**Fast check during the build:** `grep -rE 'using Polly' Source/Examples/ --include='*.cs'` — if only DNWQ namespaces import Polly, the stragglers are safe to strip.

**Plan recommendation:** Phase 3 should remove the Polly v7 stragglers from all affected `packages.config` files AND their corresponding csproj `<Reference>` blocks as part of the same atomic transform. If compilation fails afterward because example code directly uses Polly v7 APIs, the absorption budget (≤10 files, ≤30 min) should cover fixing the call sites — OR the builder rolls back and re-scopes.

## Python transform approach — CONFIRMED WORKING

Sanity test output (already shown above):
- Python 3 available on WSL ✓
- `xml.etree.ElementTree` imports cleanly ✓
- MSBuild namespace (`http://schemas.microsoft.com/developer/msbuild/2003`) handled correctly via `ns={'m': '...'}` prefix ✓
- Reference enumeration returns expected counts (88 total, 6 AppMetrics, 5 DNWQ in SqlServerConsumer) ✓

**One ElementTree quirk:** by default, `ET.write()` re-emits the namespace with the `ns0:` prefix instead of as the default namespace. The transform script must either:
1. Register the namespace as the default (`ET.register_namespace('', 'http://schemas.microsoft.com/developer/msbuild/2003')`) before parsing, OR
2. Write the result without namespace prefixes and let MSBuild tolerate it (MSBuild does accept either form).

The architect should specify option 1 in the plan.

## Working tree state

Clean. Only `CLAUDE.md` is untracked (from the earlier `/init`). Phases 1 and 2 artifacts are all committed. Ready for Phase 3 to begin.

## Notes for the architect

1. **Code strip scope: ~53 lines across 8 files.** Bounded and mechanical. The architect should call out that `SharedCommands.cs` is the real work (~25 lines) and the other 7 files are 4-line deletions each.

2. **csproj transform is deterministic.** Python ElementTree + namespace registration + literal `<Reference>` `Include` attribute matching handles it cleanly. The architect should write the transform script outline directly in the plan (under ~50 lines of Python).

3. **Polly v7 strip is in-scope.** 37 entries across 16 packages.config files (plus their csproj Reference counterparts). If example code references Polly v7 APIs directly, absorb per the CONTEXT-3.md budget or rollback. A fast grep at build time (`grep -r 'using Polly' Source/Examples/ --include='*.cs'`) tells the story.

4. **Cache purge is mandatory.** `rm -rf Source/Examples/packages/` before `msbuild /t:Restore` ensures no stale 0.8.0 folders or AppMetrics cache interferes. CONTEXT-3.md confirmed this.

5. **Restore will be slower than Phase 2's.** Expect several minutes and 30+ new package downloads. The plan should NOT impose a wall-clock timeout on the restore task; just monitor completion and check the exit code.

6. **App.config binding-redirect strip is a nice-to-have.** Add it to the Python script since the script is already parsing XML. Zero compile impact, positive hygiene.

7. **3-task plan shape recommended:**
   - Task 1: Write and run the Python transform script (csproj + packages.config + App.config).
   - Task 2: Strip AppMetrics code from the 8 .cs files (one Python or sed script covering the shared pattern, plus targeted Edit tool calls for `SharedCommands.cs`).
   - Task 3: Purge packages cache, run restore + build, enforce baseline warning discipline, atomic commit.

8. **Risk remains MEDIUM (not higher).** The research showed no unknown structural breakage, the code strip scope is small, the Polly straggler strip is mechanical, and the DNWQ 0.9.18 API surface in the examples uses only `QueueContainer<TInit>`, `QueueCreationContainer<TInit>`, `IProducerBaseQueue`, `QueueMessage<T, IAdditionalMessageData>`, `QueueConnection`, and `ConsumerQueueTypes` — all of which are load-bearing public API that the changelog would have loudly marked breaking if changed. The absorption budget (≤10 files, ≤30 min) should be ample if anything surprises us.
