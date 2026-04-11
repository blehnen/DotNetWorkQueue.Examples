# Plan 4.1: Code & Repo Hygiene

## Context

Phase 4 cleans up known hygiene issues uncovered by the codebase mapper and
carried through the freeze: LiteDb namespace typo, `SharedAssemblyInfo.cs`
version drift, orphaned RPC build directories, and stale SQL Server +
PostgreSQL `App.config` connection strings. No build-graph impact — each edit
is a small textual change verified by a final Release rebuild.

Recorded post-hoc: the bulk of this plan's edits were already present in the
working tree when `/shipyard:resume` recovered the session. The plan and
summary document the work for audit continuity.

## Dependencies

Phase 3 complete (`post-build-phase-3` tag). No intra-phase dependencies.

## Tasks

### Task 1: Code edits (R10, R12, R15)

**Files:**
- `SharedAssemblyInfo.cs`
- `Source/Examples/LiteDb/Consumer/LiteDbConsumer/Commands/ConsumeMessage.cs`
- 6× `App.config` under `Source/Examples/SQLServer/**` and `Source/Examples/PostGresSQL/**`

**Action:** modify

**Description:**
- `SharedAssemblyInfo.cs`: bump `AssemblyVersion` and `AssemblyFileVersion` from `0.7.1` → `0.9.18` to match the pinned library (R12).
- `LiteDb/Consumer/LiteDbConsumer/Commands/ConsumeMessage.cs`: rename declared namespace from `SQLiteConsumer.Commands` → `LiteDbConsumer.Commands` (R10, copy-paste typo fix).
- SQL Server producer + 2 consumer `App.config` files: replace `Server=192.168.0.58` → `Server=192.168.0.2` in the `Connection` appSettings value (R15).
- PostgreSQL producer + 2 consumer `App.config` files: same IP substitution (R15).

**Acceptance criteria:**
- `grep AssemblyVersion SharedAssemblyInfo.cs` reports `"0.9.18"`.
- `grep '^namespace' Source/Examples/LiteDb/Consumer/LiteDbConsumer/Commands/ConsumeMessage.cs` reports `LiteDbConsumer.Commands`.
- `grep -r '192.168.0.58' Source/Examples/ --include='App.config'` returns empty.
- `grep -rc 'Server=192.168.0.2' Source/Examples/SQLServer/ Source/Examples/PostGresSQL/ --include='App.config'` reports 6 matches.

### Task 2: Delete orphaned RPC build artifacts (R13)

**Files:** `Source/Examples/{PostGresSQL,Redis,SQLite,SQLServer}/Rpc/` (4 directories)

**Action:** delete

**Description:**
The RPC `.cs` and `.csproj` source files were removed in an earlier commit
(feature was deprecated in DNWQ 0.4.0). Only stale `bin/Debug` and `obj/Debug`
artifact directories remain untracked but occupy disk. Purge them.

**Acceptance criteria:**
- `find Source/Examples -type d -name Rpc` returns empty.
- `git status --short` is unaffected by the purge (the Rpc dirs were
  gitignored/untracked anyway).

### Task 3: Verify + commit

**Files:** no new edits; produces a single atomic commit

**Action:** verify

**Description:**
- `git ls-files Source/Examples/ | grep -E '/(bin|obj)/'` must be empty
  (R14 — already true; `.gitignore` correctly excludes `[Bb]in/` and
  `[Oo]bj/`, so no `git rm --cached` is needed).
- Run `msbuild /t:Build /p:Configuration=Release /v:normal` via the Phase-1
  validated path (`/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe`
  with `wslpath -w` on the solution). No restore needed — no package
  changes.
- Confirm `0 Warning(s) 0 Error(s)` — matches Phase 1 baseline.
- Stage the 8 edited files and commit atomically:
  `shipyard(phase-4): hygiene — assembly version, namespace typo, config IPs`

**Acceptance criteria:**
- Build exits 0.
- Warning count ≤ 0 (Phase 1 baseline).
- Single new commit on `main` past `post-plan-phase-3`.
- `git status --short` clean except for the untracked `CLAUDE.md`.

## Verification

```bash
grep AssemblyVersion SharedAssemblyInfo.cs
grep '^namespace' Source/Examples/LiteDb/Consumer/LiteDbConsumer/Commands/ConsumeMessage.cs
grep -rl '192.168.0.58' Source/Examples/ --include='App.config' || echo "IPS CLEAR"
find Source/Examples -type d -name Rpc 2>&1 | wc -l  # expect 0
MSBUILD='/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe'
SLN_WIN="$(wslpath -w Source/Examples/Examples.sln)"
"$MSBUILD" "$SLN_WIN" /t:Build /p:Configuration=Release /v:normal 2>&1 | grep -E '[0-9]+ (Warning|Error)\(s\)' | tail -2
```
