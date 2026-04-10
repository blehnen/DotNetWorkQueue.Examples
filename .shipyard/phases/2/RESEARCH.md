# Phase 2 Research — TFM Bump to net48

## Current csproj TFM inventory

All 20 `.csproj` files confirmed. `<TargetFrameworkVersion>` appears exactly once per file, always in the first (unconditional) `<PropertyGroup>`. No configuration-conditional `<PropertyGroup>` blocks carry a separate `<TargetFrameworkVersion>` — the Debug/Release property groups contain only `<PlatformTarget>`, `<OutputPath>`, etc. Sed will match exactly one line per file per substitution.

| csproj path | Current TFM |
|---|---|
| `Source/Examples/Shared/ConsoleShared/ConsoleShared.csproj` | `v4.5.2` |
| `Source/Examples/Shared/ConsoleView/ConsoleView.csproj` | `v4.5.2` |
| `Source/Examples/Shared/ExampleMessage/ExampleMessage.csproj` | `v4.5.2` |
| `Source/Examples/Shared/ShellControlV2/ShellControlV2.csproj` | `v4.5.2` |
| `Source/Examples/Shared/ConsoleSharedCommands/ConsoleSharedCommands.csproj` | `v4.7.2` |
| `Source/Examples/LiteDb/Producer/LiteDbProducer/LiteDbProducer.csproj` | `v4.7.2` |
| `Source/Examples/LiteDb/Consumer/LiteDbConsumer/LiteDbConsumer.csproj` | `v4.7.2` |
| `Source/Examples/LiteDb/Consumer/LiteDbConsumerAsync/LiteDbConsumerAsync.csproj` | `v4.7.2` |
| `Source/Examples/PostGresSQL/Producer/PostGresSQLProducer/PostGreSQLProducer.csproj` | `v4.7.2` |
| `Source/Examples/PostGresSQL/Consumer/PostGresSQLConsumer/PostGreSQLConsumer.csproj` | `v4.7.2` |
| `Source/Examples/PostGresSQL/Consumer/PostGreSQLConsumerAsync/PostGreSQLConsumerAsync.csproj` | `v4.7.2` |
| `Source/Examples/Redis/Producer/RedisProducer/RedisProducer.csproj` | `v4.7.2` |
| `Source/Examples/Redis/Consumer/RedisConsumer/RedisConsumer.csproj` | `v4.7.2` |
| `Source/Examples/Redis/Consumer/RedisConsumerAsync/RedisConsumerAsync.csproj` | `v4.7.2` |
| `Source/Examples/SQLite/Producer/SQLiteProducer/SQLiteProducer.csproj` | `v4.7.2` |
| `Source/Examples/SQLite/Consumer/SQLiteConsumer/SQLiteConsumer.csproj` | `v4.7.2` |
| `Source/Examples/SQLite/Consumer/SQLiteConsumerAsync/SQLiteConsumerAsync.csproj` | `v4.7.2` |
| `Source/Examples/SQLServer/Producer/SqlServerProducer/SqlServerProducer.csproj` | `v4.7.2` |
| `Source/Examples/SQLServer/Consumer/SqlServerConsumer/SqlServerConsumer.csproj` | `v4.7.2` |
| `Source/Examples/SQLServer/Consumer/SqlServerConsumerAsync/SqlServerConsumerAsync.csproj` | `v4.7.2` |

**Counts:** 16 at `v4.7.2`, 4 at `v4.5.2`. Matches STACK.md exactly (16 = 15 runnable + ConsoleSharedCommands; 4 = ConsoleShared, ConsoleView, ExampleMessage, ShellControlV2).

### TFM-adjacent property flags

| Property | Findings |
|---|---|
| `<TargetFrameworkProfile />` | Present in all 20 csproj as an empty element. Value is blank — this is the standard VS-generated stub for non-Client Profile targets. No action needed; sed does not touch it. |
| `<OldToolsVersion>` | Not present in any csproj. |
| `<PlatformToolset>` | Not present in any csproj. |
| `Condition=` on `<PropertyGroup>` referencing TFM | Not present. The Debug/Release conditional `<PropertyGroup>` blocks contain no `<TargetFrameworkVersion>`. One match per file for sed is confirmed. |
| `<TargetFramework>` (SDK-style) | Not present in any csproj. All files are classic style. |

No surprises from TFM-adjacent properties.

---

## Current App.config supportedRuntime inventory

15 `App.config` files exist (one per runnable project). Shared library projects (`ConsoleShared`, `ConsoleView`, `ConsoleSharedCommands`, `ExampleMessage`, `ShellControlV2`) have no `App.config` — correct, as libraries don't declare a startup runtime.

| App.config path | Current sku value | Has `<supportedRuntime>`? |
|---|---|---|
| `Source/Examples/LiteDb/Producer/LiteDbProducer/App.config` | `.NETFramework,Version=v4.7.2` | Yes |
| `Source/Examples/LiteDb/Consumer/LiteDbConsumer/App.config` | `.NETFramework,Version=v4.7.2` | Yes |
| `Source/Examples/LiteDb/Consumer/LiteDbConsumerAsync/App.config` | `.NETFramework,Version=v4.7.2` | Yes |
| `Source/Examples/PostGresSQL/Producer/PostGresSQLProducer/App.config` | `.NETFramework,Version=v4.7.2` | Yes |
| `Source/Examples/PostGresSQL/Consumer/PostGresSQLConsumer/App.config` | `.NETFramework,Version=v4.7.2` | Yes |
| `Source/Examples/PostGresSQL/Consumer/PostGreSQLConsumerAsync/App.config` | `.NETFramework,Version=v4.7.2` | Yes |
| `Source/Examples/Redis/Producer/RedisProducer/App.config` | `.NETFramework,Version=v4.7.2` | Yes |
| `Source/Examples/Redis/Consumer/RedisConsumer/App.config` | `.NETFramework,Version=v4.7.2` | Yes |
| `Source/Examples/Redis/Consumer/RedisConsumerAsync/App.config` | `.NETFramework,Version=v4.7.2` | Yes |
| `Source/Examples/SQLite/Producer/SQLiteProducer/App.config` | `.NETFramework,Version=v4.7.2` | Yes |
| `Source/Examples/SQLite/Consumer/SQLiteConsumer/App.config` | `.NETFramework,Version=v4.7.2` | Yes |
| `Source/Examples/SQLite/Consumer/SQLiteConsumerAsync/App.config` | `.NETFramework,Version=v4.7.2` | Yes |
| `Source/Examples/SQLServer/Producer/SqlServerProducer/App.config` | `.NETFramework,Version=v4.7.2` | Yes |
| `Source/Examples/SQLServer/Consumer/SqlServerConsumer/App.config` | `.NETFramework,Version=v4.7.2` | Yes |
| `Source/Examples/SQLServer/Consumer/SqlServerConsumerAsync/App.config` | `.NETFramework,Version=v4.7.2` | Yes |

**Counts:**
- Files with `sku=".NETFramework,Version=v4.7.2"`: **15**
- Files with `sku=".NETFramework,Version=v4.5.2"`: **0**
- Files with a `<supportedRuntime>` but some other TFM: **0**
- Files without a `<supportedRuntime>` element: **0** (all 15 App.config files have one)
- App.config files in shared library projects: **0** (libraries carry no App.config — no-op, nothing to edit)

The CONTEXT-2.md anticipated some App.configs might not have a `<supportedRuntime>` element. In practice all 15 do. Sed touches all 15; none are no-ops.

The v4.5.2 sku variant (`sku=".NETFramework,Version=v4.5.2"`) does not appear anywhere. The sed command for v4.5.2 → v4.8 on App.configs will be a safe no-op across all files.

---

## Current packages.config targetFramework inventory

17 `packages.config` files exist. **Summary counts:**

| targetFramework value | Package-entry count | File count |
|---|---|---|
| `net472` | 937 | 16 |
| `net452` | 1 | 1 |
| Any other value | 0 | 0 |

The single `net452` entry is in `Source/Examples/Shared/ConsoleView/packages.config`:

```xml
<package id="Newtonsoft.Json" version="13.0.4" targetFramework="net452" />
```

`ConsoleView` has only this one package entry. Its csproj declares `v4.5.2`, so `net452` is consistent with its current TFM. Phase 2 will bump it to `net48` along with everything else.

**Per-file net472 counts** (for change-count verification by the builder):

| packages.config | net472 entries |
|---|---|
| `Shared/ConsoleSharedCommands/packages.config` | 94 |
| `SQLServer/Producer/SqlServerProducer/packages.config` | 73 |
| `SQLServer/Consumer/SqlServerConsumer/packages.config` | 73 |
| `SQLServer/Consumer/SqlServerConsumerAsync/packages.config` | 73 |
| `PostGresSQL/Producer/PostGresSQLProducer/packages.config` | 54 |
| `PostGresSQL/Consumer/PostGresSQLConsumer/packages.config` | 54 |
| `PostGresSQL/Consumer/PostGreSQLConsumerAsync/packages.config` | 54 |
| `Redis/Producer/RedisProducer/packages.config` | 54 |
| `Redis/Consumer/RedisConsumer/packages.config` | 54 |
| `Redis/Consumer/RedisConsumerAsync/packages.config` | 54 |
| `SQLite/Producer/SQLiteProducer/packages.config` | 52 |
| `SQLite/Consumer/SQLiteConsumer/packages.config` | 52 |
| `SQLite/Consumer/SQLiteConsumerAsync/packages.config` | 52 |
| `LiteDb/Producer/LiteDbProducer/packages.config` | 48 |
| `LiteDb/Consumer/LiteDbConsumer/packages.config` | 48 |
| `LiteDb/Consumer/LiteDbConsumerAsync/packages.config` | 48 |
| `Shared/ConsoleView/packages.config` | 0 (only net452 entry) |

No mixing of net472 and net452 within a single file. ConsoleView is entirely net452; all others are entirely net472.

---

## Sed commands to execute in the build (copy-ready)

All commands use `-E` (extended regex) and target files exclusively under `Source/Examples/`. The repo root is the working directory for all commands.

### Command 1 — csproj TFM bump

```bash
find Source/Examples -name '*.csproj' \
  -exec sed -i \
    -e 's|<TargetFrameworkVersion>v4\.7\.2</TargetFrameworkVersion>|<TargetFrameworkVersion>v4.8</TargetFrameworkVersion>|g' \
    -e 's|<TargetFrameworkVersion>v4\.5\.2</TargetFrameworkVersion>|<TargetFrameworkVersion>v4.8</TargetFrameworkVersion>|g' \
    {} +
```

Expected changes: 16 lines (v4.7.2) + 4 lines (v4.5.2) = **20 lines across 20 files**.

### Command 2 — App.config supportedRuntime bump

```bash
find Source/Examples -name 'App.config' \
  -exec sed -i \
    -e 's|sku="\.NETFramework,Version=v4\.7\.2"|sku=".NETFramework,Version=v4.8"|g' \
    -e 's|sku="\.NETFramework,Version=v4\.5\.2"|sku=".NETFramework,Version=v4.8"|g' \
    {} +
```

Expected changes: 15 lines (v4.7.2) + 0 lines (v4.5.2) = **15 lines across 15 files**.

### Command 3 — packages.config targetFramework bump

```bash
find Source/Examples -name 'packages.config' \
  -exec sed -i \
    -e 's|targetFramework="net472"|targetFramework="net48"|g' \
    -e 's|targetFramework="net452"|targetFramework="net48"|g' \
    {} +
```

Expected changes: 937 occurrences across 16 files (net472) + 1 occurrence in 1 file (net452) = **938 occurrences across 17 files**.

### CRLF / idempotency notes

- All repo files are CRLF (Windows-authored repo; `.gitattributes` with `* text=auto eol=crlf` committed in Phase 1). GNU sed on WSL reads and rewrites CRLF files correctly without stripping line endings. Verified by reading `LiteDbProducer.csproj` — the file content is intact through WSL's drvfs layer.
- Idempotency: if any file already contains `v4.8` / `net48`, the sed patterns will not match and the file is left untouched. No risk of double-substitution.
- The dot in `v4.8`, `.NETFramework`, and `net48` is escaped with `\.` in the source patterns where it is a regex metacharacter (matching any character). In the replacement strings, the literal dot is used directly — correct behavior.
- `find ... -exec ... {} +` batches arguments; semantically equivalent to running sed once per file but more efficient.

---

## Pre-commit verification command block

Derived directly from Phase 1 SPIKE-NOTES.md (which established the correct wslpath translation).

```bash
MSBUILD='/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe'
SLN_WIN="$(wslpath -w /mnt/f/git/dotnetworkqueue.examples/Source/Examples/Examples.sln)"
mkdir -p /tmp/phase2

"$MSBUILD" "$SLN_WIN" /t:Restore /p:Configuration=Release /v:minimal \
  > /tmp/phase2/restore.log 2>&1 \
  || { echo "RESTORE FAILED"; tail -20 /tmp/phase2/restore.log; exit 1; }

"$MSBUILD" "$SLN_WIN" /t:Build /p:Configuration=Release /v:normal \
  > /tmp/phase2/build.log 2>&1 \
  || { echo "BUILD FAILED"; tail -40 /tmp/phase2/build.log; exit 1; }

tail -5 /tmp/phase2/build.log | grep -E '[0-9]+ (Warning|Error)\(s\)'
```

**Verbosity note:** `/v:normal` is required. Phase 1 confirmed that `/v:minimal` suppresses the `X Warning(s)  X Error(s)` summary line. The grep on `tail -5` will only find the summary when verbosity is `normal` or higher.

**Baseline gate:** build log must show `0 Warning(s)` and `0 Error(s)`. Any deviation from the Phase 1 baseline of `0 warnings, 0 errors` is a hard stop — run `git reset --hard HEAD` and escalate before committing.

---

## Risks / surprises

| Item | Finding | Action required |
|---|---|---|
| Multiple `<PropertyGroup>` blocks with separate `<TargetFrameworkVersion>` | Not present. Debug/Release conditional blocks contain no `<TargetFrameworkVersion>`. Sed matches exactly 1 line per csproj. | None |
| SDK-style `<TargetFramework>` (without "Version" suffix) | Not present in any csproj. All 20 files are classic style. | None |
| `.targets` / `.props` files in `Source/Examples/` project directories | None found. All `.targets` files are inside `Source/Examples/packages/` (NuGet package cache, not tracked). | None |
| `<AssemblyOriginatorKeyFile>` / `.snk` references | Not present in any csproj. | None |
| `<TargetFrameworkProfile />` (empty element) | Present in all 20 csproj as a VS-generated stub. Empty value means no Client Profile constraint. Sed does not touch this element. | None |
| ConsoleView packages.config has `net452` not `net472` | 1 entry. The sed command for `net452` → `net48` handles it. Expected; consistent with the project's v4.5.2 TFM. | Already covered by Command 3 |
| App.config v4.5.2 sku variant absent | No App.config contains `sku=".NETFramework,Version=v4.5.2"`. The sed command for it is a safe no-op. | None |
| `Stub.System.Data.SQLite.Core.NetFramework` package ships `.targets` files only up to `net46` | This package's highest lib/build TFM in the cache is `net46`. Under net48, NuGet will select the `net46` folder (best available, backward-compatible). This is the same situation as the current net472 target and already works. The ROADMAP Phase 2 risk note covers this. | Verify no new binding-redirect warnings appear in the build log |

---

## Notes for the architect

**3-task plan shape: still correct.** The three substitutions are genuinely independent edit passes (different file globs, different sed patterns) and collapsing them to one task would lose the per-type auditability that `git diff --stat` provides. The CONTEXT-2.md split — (1) csproj edits, (2) App.config + packages.config edits, (3) restore + build + commit — is the right shape.

**File counts for `git diff --stat` sanity check:**
- After Task 1: exactly 20 csproj files changed, 1 line each.
- After Task 2: exactly 15 App.config + 17 packages.config = 32 files changed.
- Total staged before Task 3: 52 files (20 + 15 + 17). This matches the CONTEXT-2.md estimate precisely.

**No per-file hand-editing required.** All substitutions are uniform across their file types. The sed commands above are sufficient; no file needs special-casing.

**ConsoleView note for the architect:** `ConsoleView/packages.config` has only 1 package entry (`Newtonsoft.Json` at `net452`). After Phase 2, it will read `net48`. This is correct and intentional — the project's TFM will also be bumped to v4.8 in the same commit.

**R5 (HintPath audit) deferred to Phase 3** per CONTEXT-2.md. Phase 2 does not touch HintPaths. The packages cache already provides correct net461/net472 fallback folders for net48 consumers; no binding errors are expected from TFM-only bump.
