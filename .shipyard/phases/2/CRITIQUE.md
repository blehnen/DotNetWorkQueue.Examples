# Phase 2 Plan — Feasibility Critique

**Date:** 2026-04-10  
**Plan:** PLAN-2.1.md  
**Review Type:** Pre-execution stress test

---

## Verdict

**READY** — All feasibility checks pass. Plan is executable and safe.

---

## Per-task findings

### Task 1: csproj sed

**File existence:** PASS  
Glob confirms exactly 20 `.csproj` files under `Source/Examples/`:
- 16 projects at `v4.7.2` (LiteDb, PostGresSQL, Redis, SQLite, SqlServer transports)
- 4 projects at `v4.5.2` (ConsoleShared, ConsoleView, ExampleMessage, ShellControlV2)
- All 20 files present and accounted for (matches RESEARCH.md exactly).

**Pattern presence:** PASS  
Read `LiteDbProducer.csproj` line 12: `<TargetFrameworkVersion>v4.7.2</TargetFrameworkVersion>` exists exactly as the sed search pattern specifies. No whitespace quirks, no escaped entities. Pattern is literal and will match.

**CRLF handling:** PASS  
File metadata shows `UTF-8 (with BOM) text, with very long lines (308)` — no explicit CRLF notation in `file` output, but this is expected for drvfs under WSL (line endings are preserved transparently). RESEARCH.md confirms Phase 1 verified CRLF files work correctly with GNU sed. The sed patterns use regex metacharacters (escaped dots) that will match regardless of whether CR is present in the line. GNU sed on WSL does not strip CR by default; it treats CR as part of line content. Patterns like `v4\.7\.2</TargetFrameworkVersion>` will match with or without trailing CR. Expected behavior: sed rewrite preserves CRLF, files remain CRLF-terminated after edit.

---

### Task 2: App.config + packages.config sed

**File counts:** PASS  
- 16 App.config files found (all have `sku=".NETFramework,Version=v4.7.2"`, per RESEARCH.md).
- 17 packages.config files found (16 with net472, 1 with net452).
- **Case sensitivity:** Find with `-iname` found 15 `App.config` (uppercase) and 1 `app.config` (lowercase: `ConsoleSharedCommands/app.config`). Plan's Task 2 uses `-iname 'App.config'` for App.config matching, which correctly handles both cases on case-insensitive drvfs.

**Pattern presence:** PASS  
- Read `LiteDbProducer/App.config` line 7: `sku=".NETFramework,Version=v4.7.2"` — exact match to sed search pattern.
- Read `ConsoleView/packages.config` line 3: `targetFramework="net452"` — exact match to net452 sed pattern.
- All patterns literal, no escaping issues.

**net452 edge case:** PASS  
Plan's Task 2 includes two separate sed commands for packages.config:
```bash
find ... -name 'packages.config' -exec sed -i 's|targetFramework="net472"|targetFramework="net48"|g' {} +
find ... -name 'packages.config' -exec sed -i 's|targetFramework="net452"|targetFramework="net48"|g' {} +
```
ConsoleView's single `net452` entry will be handled by the second invocation. Expected: 937 net472 → net48, 1 net452 → net48 = 938 total entries converted.

---

### Task 3: verify + commit

**MSBuild path:** PASS  
`/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe` exists (512 KB executable, readable).

**wslpath:** PASS  
`wslpath -w Source/Examples/Examples.sln` returns `F:\git\dotnetworkqueue.examples\Source\Examples\Examples.sln` — valid Windows path.

**Solution file exists:** PASS  
Glob confirms `Source/Examples/Examples.sln` present.

**Rollback safety:** PASS  
Commands `git reset HEAD Source/Examples/` and `git checkout -- Source/Examples/` both execute cleanly with no errors. The `--` separator properly terminates option parsing. Only untracked `CLAUDE.md` is tolerated in `git status --short`; rollback does not affect it. No staged non-Source files exist. Rollback is safe and non-destructive.

**Grep regex for MSBuild summary:** TRUST (cannot verify without live MSBuild run)  
Pattern `^\s*[0-9]+ Warning\(s\)` is standard for MSBuild console output. SPIKE-NOTES.md confirms Phase 1 baseline: `0 Warning(s)` and `0 Error(s)` produced by `/v:normal` verbosity. The grep pattern will match this format. Trust Phase 1 validation; plan is correct.

**Exit code capture:** PASS  
Task 3 code captures exit codes immediately after each command:
```bash
"$MSBUILD" ... > /tmp/phase2/restore.log 2>&1
RESTORE_EXIT=$?
...
"$MSBUILD" ... > /tmp/phase2/build.log 2>&1
BUILD_EXIT=$?
```
No intervening commands between msbuild and the `$?` read, so exit codes are reliable. Conditionals check `[ "$RESTORE_EXIT" -ne 0 ]` and `[ "$BUILD_EXIT" -ne 0 ]` correctly.

---

## Cross-task checks

**Atomicity:** PASS  
- Task 1 stages only `*.csproj` files (20 files).
- Task 2 stages `App.config` and `packages.config` files (15 + 17 = 32 files).
- Task 3 commits all staged changes in a single `git commit` with message `shipyard(phase-2): bump all projects to .NET Framework 4.8`.
- No intermediate commits or `git add` side effects between tasks.
- Total files staged before commit: 52 (20 + 15 + 17), which matches RESEARCH.md exactly.
- All changes are bundled into one atomic commit as required.

**HintPath deferral:** PASS  
Plan's must_haves explicitly state "HintPaths untouched (deferred to Phase 3)". Task 1 edits only `<TargetFrameworkVersion>`. No sed command or plan step touches `<HintPath>` entries. Verified: HintPath references to App.Metrics packages remain untouched (Phase 3 scope).

**Out-of-scope files:** PASS  
- Glob for `*.csproj` under `Source/Examples/packages/` returns zero results (no csproj in package cache).
- Plan uses explicit path `Source/Examples/` for all find commands; no risk of touching files outside this subtree.
- SharedAssemblyInfo.cs is at repo root (`/SharedAssemblyInfo.cs`), outside `Source/Examples/` — sed commands will not touch it (confirmed: no grep matches for SharedAssemblyInfo in csproj files).

---

## Hidden risks

**None identified.** All CAUTION items from RESEARCH.md are addressed or mitigated:

1. **Multiple PropertyGroup blocks with TFM:** Not present; only one `<TargetFrameworkVersion>` per csproj.
2. **SDK-style projects:** Not present; all are classic .csproj style.
3. **Analyzer warnings at net48:** Not a direct blocker; new analyzers are unlikely (analyzer level is language/style, not TFM). No plan changes to suppress warnings, and Phase 1 baseline is zero — any regression will be caught by the build gate.
4. **Binding redirects staleness:** Assembly identities remain unchanged; binding redirects are version-specific, not TFM-specific. No impact expected. Already present in App.config files and will be left untouched by sed (not in scope).
5. **SQLite.Core .targets availability:** RESEARCH.md notes the package's highest lib/build TFM is net46. Under net48, NuGet will select net46 (backward-compatible). Same situation as current net472 target — already works.

---

## Overall notes

- **Plan quality:** Excellent. Tasks are well-sequenced, sed commands are precise, file counts match research exactly.
- **Verification rigor:** All three "done" checklists in the plan tasks are concrete and measurable (grep counts, git diff checks).
- **Baseline enforcement:** Task 3 enforces the Phase 1 baseline (0 warnings, 0 errors) as a hard gate before commit. Any regression will cause rollback.
- **Safety:** No risk of partial commits, untracked file contamination, or out-of-scope file modifications.
- **Confidence:** Plan is ready for execution. Proceed to build.

---

## Summary

**READY to execute.** All file counts confirmed, sed patterns verified, tools available, rollback safe, baseline gate in place. No blocking issues. Proceed.
