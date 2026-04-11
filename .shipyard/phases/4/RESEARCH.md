# RESEARCH — Phase 4 (Code & Repo Hygiene)

Compressed research pass. Phase 4 is LOW-risk mechanical work with every requirement already scoped by filename in ROADMAP.md. The facts below were verified directly against `main` at commit `042706a` (Phase 3 complete) on 2026-04-11 using `git ls-files`, `find`, and `grep`. A full researcher-agent pass would restate these same facts from the same codebase docs, so this note captures the verified state in one place and hands off to the architect.

## Codebase context

- **Repo:** `dotnetworkqueue.examples` — WinForms shell-hosted example apps for the DotNetWorkQueue library across SQL Server, PostgreSQL, SQLite, LiteDb, Redis (see `.shipyard/codebase/STACK.md`, `.shipyard/codebase/ARCHITECTURE.md`).
- **Build system:** .NET Framework 4.8 (post-Phase 2), classic `.csproj` with `packages.config`. Restore via `nuget restore`, build via `msbuild Source/Examples/Examples.sln /p:Configuration=Release`.
- **Current baseline:** 0 warnings / 0 errors in Release as of Phase 3 commit `30298fd`. Preserved into shipyard commit `042706a`.
- **Working tree state:** clean except the untracked `CLAUDE.md` (intentionally kept out of every phase commit).

## Requirement-by-requirement verification

### R10 — LiteDbConsumer namespace fix

- **Target file:** `Source/Examples/LiteDb/Consumer/LiteDbConsumer/Commands/ConsumeMessage.cs`
- **Current state:** `namespace SQLiteConsumer.Commands` at line 30 (verified via `grep -n "^namespace"`).
- **Expected state:** `namespace LiteDbConsumer.Commands`.
- **Cross-repo `using` reference check:** the CLAUDE.md note "Commands appear in the shell by class name" and `STRUCTURE.md` describe the per-project reflection-based command discovery (`Assembly.GetExecutingAssembly()` in `Program.Main` → `ShellControlV2` scans for `IConsoleCommand` implementations). Commands are not imported across projects — each project's command set comes from its own assembly at runtime. A repo-wide grep for `using SQLiteConsumer.Commands;` must be performed by the builder before the edit to prove no cross-reference exists; expected result is zero matches. If any matches appear, the builder must update each reference in the same commit.
- **Blast radius:** single file declaration. No class rename, no file move.

### R12 — SharedAssemblyInfo version bump

- **Target file:** `SharedAssemblyInfo.cs` at repo root (linked into every project per CLAUDE.md).
- **Current state (verified via `grep -E "Assembly(Version|FileVersion|InformationalVersion)"`):**
  ```
  [assembly: AssemblyVersion("0.7.1")]
  [assembly: AssemblyFileVersion("0.7.1")]
  [assembly: AssemblyInformationalVersion("0.7.1")]
  ```
- **Expected state:** all three at `"0.9.18"`.
- **Cross-reference check:** no other file references `0.7.1` as a version string (the builder should grep to confirm before the edit).
- **Blast radius:** three attribute lines in one file. Every project picks up the new version on next build via the shared link; no per-project `AssemblyInfo.cs` bumps required.

### R13 — Orphaned RPC build artifacts

- **Rpc folder discovery (verified via `find Source/Examples -type d -name Rpc`):**
  ```
  Source/Examples/PostGresSQL/Rpc
  Source/Examples/Redis/Rpc
  Source/Examples/SQLite/Rpc
  Source/Examples/SQLServer/Rpc
  ```
- **Rpc bin/obj state:** every Rpc project subfolder under those four parents has stale `bin/` and `obj/` on disk. The builder must enumerate them on-demand via `find Source/Examples/*/Rpc -type d \( -name bin -o -name obj \)` at execution time (rather than hardcoding the list from this research pass) because the exact set depends on how many Rpc subprojects exist per transport.
- **Git state:** `git ls-files | grep -E '/(bin|obj)/'` returns 0 results — **nothing tracked**. The delete is a pure working-tree `rm -rf`, no `git rm` verb needed. After the delete, `git status` for the Rpc folders must be unchanged (proving the delete only removed untracked state).
- **Scope guard:** the other 48 non-Rpc `bin/obj` directories on disk are active build output from Phase 3's Release build. They are out of scope for R13 — do not touch them.
- **Rpc source tree scope guard:** the Rpc `.cs`, `.csproj`, `packages.config`, and `App.config` files are **out of scope for Phase 4**. ROADMAP R13 only names the `bin`/`obj` subdirectories.

### R14 — Tracked bin/obj cleanup

- **Current state:** `git ls-files | grep -E '/(bin|obj)/'` returns 0 files. Nothing tracked.
- **`.gitignore` coverage:** per `.shipyard/codebase/CONCERNS.md` the gitignore already contains `[Bb]in/` and `[Oo]bj/`. No gap expected.
- **Effective work:** none. This requirement collapses to a verification assertion — "zero tracked bin/obj; gitignore coverage intact" — that the plan captures as a read-only check in the build gate task.

### R15 — SQL Server + PostgreSQL App.config IP correction

- **File list (verified via `grep -rn "192.168.0.58" Source/Examples --include="App.config" -l`):**
  1. `Source/Examples/PostGresSQL/Consumer/PostGreSQLConsumerAsync/App.config`
  2. `Source/Examples/PostGresSQL/Consumer/PostGresSQLConsumer/App.config`
  3. `Source/Examples/PostGresSQL/Producer/PostGresSQLProducer/App.config`
  4. `Source/Examples/SQLServer/Consumer/SqlServerConsumer/App.config`
  5. `Source/Examples/SQLServer/Consumer/SqlServerConsumerAsync/App.config`
  6. `Source/Examples/SQLServer/Producer/SqlServerProducer/App.config`
- **Match count:** exactly 6 files, one occurrence per file in the `Connection` appSetting value (verified via `grep -rn ... | wc -l`). Matches ROADMAP's "all 6 SQL Server + PostgreSQL producer/consumer projects".
- **Expected edit:** change the `192.168.0.58` octet string to `192.168.0.2` in each of the six files. Leave the rest of the connection string (username, password, port, database name, queue name, other appSettings) untouched.
- **Redis scope guard:** Redis `App.config` files already use `192.168.0.2` (per ROADMAP note and confirmed by the absence of Redis matches in the grep). Do not edit Redis App.config files.
- **SQLite/LiteDb scope guard:** file-based transports, no IP addresses in their `App.config` connection strings. Out of scope.

## Hidden-dependency check

Each requirement touches an orthogonal file set:

| Requirement | Files touched |
|------------|---------------|
| R10 | 1 file: `LiteDbConsumer/Commands/ConsumeMessage.cs` |
| R12 | 1 file: `SharedAssemblyInfo.cs` |
| R13 | 0 files tracked; working-tree deletes under `Source/Examples/*/Rpc/**/{bin,obj}` only |
| R14 | 0 files (verification only) |
| R15 | 6 files: the PG + SQL Server `App.config` set listed above |

No file appears in more than one requirement. R10, R12, R15 can land in any order within the same commit. R13 and R14 are working-tree/git-state checks with no source code impact. The whole phase collapses to a single atomic commit safely.

## Build gate expectations

After the edits land, `nuget restore` + `msbuild Source/Examples/Examples.sln /p:Configuration=Release` must emit **0 warnings / 0 errors**. The Phase 1 baseline ceiling (0/0) still holds — Phase 4 does not touch any build-graph-relevant file. A non-zero warning count is a revert signal, not an acceptance condition.

## Plan handoff

This research pass validates the plan shape already drafted in `CONTEXT-4.md` — single wave, one plan file with three tasks: (1) source edits R10+R12+R15, (2) working-tree cleanup R13+R14, (3) build gate + atomic commit. Architect proceeds directly to `PLAN-4.1.md` without further decomposition.
