# Phase 1 Research — Spike & Baseline Capture

Research performed directly in the main planning context (first researcher-agent dispatch hit its turn budget without writing output).

## R1: NuGet 0.9.18 availability — GO ✅

Verified via `powershell.exe Invoke-WebRequest` against `https://api.nuget.org/v3-flatcontainer/{package}/index.json` (WSL WebFetch and curl/wget are blocked by a context-mode hook; PowerShell on the Windows side bypasses it).

| Package | 0.9.18 | Last 3 published |
|---|---|---|
| `DotNetWorkQueue` | ✅ | 0.9.18, 0.9.30, 0.9.31 |
| `DotNetWorkQueue.Transport.SqlServer` | ✅ | 0.9.18, 0.9.30, 0.9.31 |
| `DotNetWorkQueue.Transport.PostgreSQL` | ✅ | 0.9.18, 0.9.30, 0.9.31 |
| `DotNetWorkQueue.Transport.SQLite` | ✅ | 0.9.18, 0.9.30, 0.9.31 |
| `DotNetWorkQueue.Transport.LiteDb` | ✅ | 0.9.18, 0.9.30, 0.9.31 |
| `DotNetWorkQueue.Transport.Redis` | ✅ | 0.9.18, 0.9.30, 0.9.31 |
| `DotNetWorkQueue.Transport.Shared` | ✅ | 0.9.18, 0.9.30, 0.9.31 |
| `DotNetWorkQueue.Transport.RelationalDatabase` | ✅ | 0.9.18, 0.9.30, 0.9.31 |

**Interesting finding:** every package skips 0.9.19–0.9.29 — those versions were never published to NuGet. The changelog mentions 0.9.19 as "drop net48" but it was apparently tagged in git without a corresponding NuGet publish. This means **0.9.18 is simultaneously the last 4.8-compatible release AND the current latest non-net10-only published version**. There is no competing "newer 4.8" release to consider.

**No escalation needed.** The Phase 3 package bump target of `0.9.18` is unambiguous and universally available.

## R2: AppMetrics touch-point inventory

Grep pattern: `App\.Metrics|IMetricsRoot|IMetrics\b|DotNetWorkQueue\.AppMetrics` across `Source/Examples/`. 88 total hits; filtering out the `Source/Examples/packages/` cache (auto-regenerated, not tracked), the **editable-file** hits are:

### `.cs` source files that reference AppMetrics (8 files)

Shared (in `Source/Examples/Shared/ConsoleSharedCommands/Commands/`):
- `SharedCommands.cs`
- `ConsumeMessageAsync.cs`
- `ConsumeMessage.cs`

Per-transport `SendMessage.cs` in each Producer project:
- `Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/SendMessage.cs`
- `Source/Examples/PostGresSQL/Producer/PostGresSQLProducer/Commands/SendMessage.cs`
- `Source/Examples/Redis/Producer/RedisProducer/Commands/SendMessage.cs`
- `Source/Examples/SQLite/Producer/SQLiteProducer/Commands/SendMessage.cs`
- `Source/Examples/SQLServer/Producer/SqlServerProducer/Commands/SendMessage.cs`

**Observation:** No `Consumer`, `ConsumerAsync`, or `QueueCreation` files appear in the list. The AppMetrics wiring is confined to the **Producer → SendMessage** path and the **shared ConsumeMessage** bases. This narrows Phase 3's code-strip scope significantly.

The researcher did not inspect the in-file shape (method names, whether wiring is in constructor/RegisterService/etc.) — that is Phase 3 architect work, not Phase 1 spike work. The file list is the deliverable.

### `packages.config` files referencing `App.Metrics` (17 files)

All 15 runnable project `packages.config` files plus `Shared/ConsoleSharedCommands/packages.config` and `Shared/ConsoleView/packages.config`. Every such file carries 6 `<package id="App.Metrics..." version="4.3.0" targetFramework="net472" />` entries covering: `App.Metrics`, `App.Metrics.Abstractions`, `App.Metrics.Concurrency`, `App.Metrics.Core`, `App.Metrics.Formatters.Ascii`, `App.Metrics.Formatters.Json`.

### `.csproj` files referencing `App.Metrics` (17 files — same projects)

Each csproj holds corresponding `<Reference Include="App.Metrics..." ...>` entries with `<HintPath>..\..\..\packages\App.Metrics.4.3.0\lib\net461\App.Metrics.dll</HintPath>` style paths. Phase 3 must remove both the package references and the csproj `<Reference>` items.

### `App.config` files referencing `App.Metrics` (15 files)

Every runnable project's `App.config` has `<assemblyBinding>` → `<dependentAssembly>` → `<assemblyIdentity name="App.Metrics..."` binding-redirect entries. Phase 3 must strip these too, or leave a cosmetic no-op (the runtime won't use them but the warnings may appear).

### `Source/Examples/packages/` cache (36+ XML docs, not tracked)

These files live in the NuGet package cache and will vanish when Phase 3 does `nuget restore` against a package list that no longer contains AppMetrics. **No edit action required.** The concerns mapper flagged them earlier but they resolve themselves.

## Pre-existing `.gitattributes` / `.gitignore` state

- **`.gitattributes`: does not exist** at repo root. Confirmed via `ls .gitattributes` returning "No such file or directory".
- **`.gitignore`: already excludes `[Bb]in/` and `[Oo]bj/`** via the classic Visual Studio ignore pattern. No new rules needed for R14 — the existing ignore is fine going forward. The only cleanup is `git rm -r --cached` for already-tracked `bin/`/`obj/` directories.
- **Binary file extensions under `Source/Examples/packages/`**: `.dll`, `.exe`, `.pdb`, `.nupkg`, `.xml`, `.nuspec`, `.targets`, `.props`, `.psmdcp`, `.png`, `.md`, `.config`. For a `.gitattributes` `binary` marker set, the load-bearing ones are `.dll`, `.exe`, `.pdb`, `.nupkg`, `.png`. `.xml`, `.md`, `.targets`, `.props`, `.config`, `.nuspec` are text and should follow the `text=auto eol=crlf` default.

### Suggested `.gitattributes` starter content

```gitattributes
# Windows-authored repo; preserve CRLF in the working tree.
* text=auto eol=crlf

# Binary assets that must never be text-normalized.
*.dll binary
*.exe binary
*.pdb binary
*.nupkg binary
*.snk binary
*.png binary
*.ico binary
*.jpg binary
*.gif binary
```

Architect should confirm this is the final rule set when writing the plan. It may need small additions if the researcher missed a binary extension type present in the repo.

## WSL build-tool invocation

- `cmd.exe /c msbuild -version` → **"not recognized"**. MSBuild is NOT on the shell PATH inherited by `cmd.exe` from WSL.
- `cmd.exe /c nuget help` → **"not recognized"**. Same — no nuget.exe on PATH.
- **vswhere.exe exists** at `/mnt/c/Program Files (x86)/Microsoft Visual Studio/Installer/vswhere.exe`.
- **`vswhere.exe -latest -requires Microsoft.Component.MSBuild -find MSBuild\**\Bin\MSBuild.exe`** returns `C:\Program Files\Microsoft Visual Studio\18\Community\MSBuild\Current\Bin\MSBuild.exe` — Visual Studio 2026 Community is installed. MSBuild 18 is present.

### Recommendation to the architect

**Baseline capture IS automatable** from subagents in WSL. The invocation pattern is:

```bash
MSBUILD='/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe'
"$MSBUILD" Source/Examples/Examples.sln /t:Restore /p:Configuration=Release
"$MSBUILD" Source/Examples/Examples.sln /t:Build    /p:Configuration=Release
```

Notes:
- **MSBuild 18 handles `packages.config` restore via `/t:Restore`** (built-in since MSBuild 17.3). No separate `nuget.exe` is required. If restore surprises us, fall back to downloading `nuget.exe` as a Phase 1 sub-task — but expect it to Just Work.
- Warning count can be captured from MSBuild stdout by filtering for `warning ` lines or parsing the summary tail (`XXX Warning(s)`).
- The full path contains spaces; plan tasks must quote it.
- Absolute path rather than bash variable may be clearer in plan files.

## File counts

| Type | Count | Location |
|---|---|---|
| `packages.config` | **17** | 15 runnable + `Shared/ConsoleSharedCommands/` + `Shared/ConsoleView/` |
| `.csproj` | **20** | 15 runnable + 5 shared (`ConsoleShared`, `ConsoleSharedCommands`, `ConsoleView`, `ExampleMessage`, `ShellControlV2`) |
| `App.config` in runnable projects | **15** | one per Producer/Consumer/ConsumerAsync |
| Runnable projects | **15** | 5 transports × 3 variants (Producer, Consumer, ConsumerAsync) |
| Shared projects | **5** | see above |

All 17 `packages.config` files and all 17 `.csproj` files that reference `App.Metrics` will be touched in Phase 3. Phase 2 (TFM bump) touches all 20 csproj + 15 App.config + 17 packages.config — a broader sweep but mechanical.

## Notes for the architect

1. **Line-ending sub-task ordering is load-bearing.** `.gitattributes` must commit BEFORE the baseline warning capture, otherwise capture may happen against a tree that's still drifting. Spike notes file should be written AFTER baseline capture so it can quote the integer.
2. **Baseline capture tooling:** plan tasks can invoke MSBuild directly with the full `/mnt/c/...` path. No user-manual fallback needed. Quote the path.
3. **AppMetrics code strip scope is narrower than feared:** only 8 `.cs` files, and 5 of them are the per-transport `SendMessage.cs` which inherits from `SharedSendCommands`. The actual wiring may live mostly in the shared base (to be verified during Phase 3 research), leaving per-transport files with minimal delta.
4. **No scope escalation from Phase 1.** R1 is green across all 7 packages, R2's file list is bounded, the tooling works. The architect can produce a clean spike plan with no blocking unknowns.
5. **One emergent risk:** the Phase 1 spike now includes a `.gitattributes` sub-task that may trigger real normalization changes via `git add --renormalize .`. If it does, those normalized changes are a one-shot commit that must land on `main` alongside `.gitattributes` (or immediately after) — treat it as an atomic unit so nobody ever checks out a tree where `.gitattributes` is committed but the working tree hasn't been normalized to match. Architect should plan this as a single task with a conditional second commit, not as two separate tasks.
