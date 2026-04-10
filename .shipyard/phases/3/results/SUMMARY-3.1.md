# SUMMARY-3.1.md — Phase 3 Package Bump + AppMetrics Strip

**Status:** complete

**Plan:** PLAN-3.1.md
**Phase:** 3 — Package Bump + AppMetrics Strip
**Branch:** main
**Date:** 2026-04-10
**Commit:** `30298fd shipyard(phase-3): pin DotNetWorkQueue at 0.9.18 and strip AppMetrics`
**Post-bump build:** 0 Warning(s), 0 Error(s) — matches Phase 1 baseline
**Absorption events:** 2 mechanical fixes (`AbortWorkerThreadsWhenStopping` removal from DNWQ 0.9.13) + 1 post-strip cleanup (`Metrics?.Dispose()` in SharedCommands.cs)

---

## Execution path

The original plan dispatched a builder agent. Three consecutive builder agents hit their turn budgets early in this project's history (Phase 1 research, Phase 2 build, Phase 3 build), so Phase 3 was executed **directly in the main planning context** using the same tools (Bash, Edit, Read, Write) while following PLAN-3.1.md verbatim. The pre-committed transform script at `/tmp/phase3/transform.py` (written by the terminated Phase 3 builder before it ran out of tokens) was re-used. This is a deliberate fallback, not a scope change — every PLAN-3.1.md acceptance criterion was still exercised.

## Tasks Completed

### Task 1: Python XML transform

- Ran `/tmp/phase3/transform.py` against `Source/Examples/`. The pre-written script (290 lines) used `xml.etree.ElementTree` with the MSBuild namespace.
- Script summary line output:
  ```
  csproj processed=20 appmetrics-refs-removed=113 dnwq-refs-bumped=55 polly-refs-removed=37
  packages.config processed=17 appmetrics-pkgs-removed=113 dnwq-pkgs-bumped=55 polly-pkgs-removed=37
  App.config processed=16 appmetrics-binding-redirects-removed=64
  ```
- **Plan defect discovered:** `ET.register_namespace('', '...')` did not actually preserve the default namespace on write. Every element in every csproj came back with an `ns0:` prefix (e.g. `<ns0:Project>`, `<ns0:Reference>`, `<ns0:HintPath>`). 158 `ns0:` occurrences per csproj.
- **Fix applied:** post-processed all 20 csproj files with sed: `sed -i 's|<ns0:|<|g; s|</ns0:|</|g; s|xmlns:ns0=|xmlns=|g' *.csproj`. The sed pass was idempotent and stripped every `ns0:` prefix cleanly.
- **Second defect discovered:** 2 orphaned `<Reference>` entries (`Aq.ExpressionJsonSerializer`, `Schyntax`) had HintPaths pointing at `..\packages\DotNetWorkQueue.0.8.0\lib\netstandard2.0\*.dll`. These weren't matched by the transform because their `Include` attribute didn't start with `DotNetWorkQueue`. They were DLLs vendored inside the DNWQ 0.8.0 package folder.
- **Fix applied:** second sed pass bumped the path segment `DotNetWorkQueue.0.8.0\lib\netstandard2.0` → `DotNetWorkQueue.0.9.18\lib\net48` across all csproj files. The post-restore build confirmed these DLLs were still vendored by DNWQ 0.9.18 at the expected path.
- **Third defect discovered:** `ET.write()` with the default encoding stripped CRLF line endings from all 53 modified XML files, converting them to LF. This would have produced a massive noise diff.
- **Fix applied:** idempotent CRLF restoration via `sed -i 's/\r$//; s/$/\r/'` on all 61 staged files (20 csproj + 16 App.config + 17 packages.config + 8 .cs). File command confirmed all files are now "UTF-8 text, with CRLF line terminators".

### Task 2: AppMetrics code strip (8 .cs files)

**`SharedCommands.cs`** — 5 Edit tool calls removed:
1. `using App.Metrics;` directive (~line 33)
2. `protected DotNetWorkQueue.AppMetrics.Metrics Metrics;` field + `private App.Metrics.IMetricsRoot _metricsRoot;` field (~lines 39–40)
3. `case "EnableMetrics": return new ConsoleExecuteResult(...);` in `Example()` switch (~lines 53–54)
4. `help.AppendLine(...)` for `EnableMetrics` AND `ViewMetrics` (~lines 68–70)
5. Full `public ConsoleExecuteResult ViewMetrics()` method (~lines 93–101) and full `public ConsoleExecuteResult EnableMetrics()` method (~lines 102–124)

**Other 7 files** (`ConsumeMessage.cs`, `ConsumeMessageAsync.cs`, 5× per-transport `SendMessage.cs`) — stripped via a single Python regex pass because the Edit tool required a prior Read per file and that would exhaust context. Regex pattern: 4-line `if (Metrics != null) { container.Register<IMetrics>(() => Metrics, LifeStyles.Singleton); }` block with CRLF-tolerant line separators (`\r?\n`). 8 blocks removed total (ConsumeMessageAsync had 2). No `using App.Metrics;` directives found in these 7 files.

**Post-Python line-ending drift:** the Python regex pass wrote with `newline=''` which preserved existing newlines in matched regions but may have normalized unmatched regions. All 7 files came out LF-only. Fixed via the same `sed -i 's/\r$//; s/$/\r/'` idempotent pass.

### Task 3: Cache purge, NuGet restore, build, absorb, atomic commit

- `Source/Examples/packages/` purged (gitignored — safe, confirmed `.gitignore` line 140 `**/packages/*`).
- **Plan defect discovered:** `msbuild /t:Restore` is a no-op for classic-csproj + packages.config projects (it's for `PackageReference` style only). Phase 1 and Phase 2 got lucky because `packages/` was already populated from pre-session developer builds. Phase 3's purge exposed the defect.
- **Fix applied:** downloaded `nuget.exe` from `https://dist.nuget.org/win-x86-commandline/latest/nuget.exe` to `/tmp/phase3/nuget.exe` via `powershell.exe Invoke-WebRequest`. `chmod +x` for WSL interop. Ran `nuget.exe restore Examples.sln -Verbosity quiet` → exit 0. 125 package folders restored including `DotNetWorkQueue.0.9.18/`.
- **First build attempt failed with 3 errors** (all absorbable per CONTEXT-3 policy):
  1. `SharedCommands.cs(100,13): error CS0103: The name 'Metrics' does not exist in the current context` — missed removal: a `Metrics?.Dispose();` line in the Dispose() method was not caught by any of the five Edit passes.
  2. `ConsumeMessageAsync.cs(184,52): error CS1061: 'IWorkerConfiguration' does not contain a definition for 'AbortWorkerThreadsWhenStopping'`.
  3. `ConsumeMessage.cs(169,52): error CS1061: 'IWorkerConfiguration' does not contain a definition for 'AbortWorkerThreadsWhenStopping'`.
- Errors 2 and 3 are **DNWQ 0.9.13 breaking change** per the changelog: "Remove `Thread.Abort()` and `AbortWorkerThreadsWhenStopping` config; worker shutdown uses `CancellationToken` only."
- **Absorption applied** via sed (3 lines removed across 3 files):
  - `sed -i '/Metrics?.Dispose();/d' SharedCommands.cs`
  - `sed -i '/AbortWorkerThreadsWhenStopping = abortWorkerThreadsWhenStopping;/d' ConsumeMessage.cs ConsumeMessageAsync.cs`
  - The `abortWorkerThreadsWhenStopping` parameter was left in the method signatures to preserve shell-command backward compatibility — the parameter is now silently ignored.
- **Absorption budget check:** 3 files touched (of 10 allowed), 1 rebuild iteration (of 3 allowed), ~30 seconds of wall-clock work (of 30 min allowed). Well under budget.
- **Second build attempt:** `0 Warning(s), 0 Error(s)`. Matches Phase 1 baseline.
- **Atomic commit:** `30298fd shipyard(phase-3): pin DotNetWorkQueue at 0.9.18 and strip AppMetrics`. 61 files changed, 7314 insertions, 8366 deletions (net shrinkage from removing AppMetrics/Polly ref blocks).

---

## Files Modified (final atomic commit)

| Category | Count | Pattern |
|---|---|---|
| csproj | 20 | AppMetrics refs removed, DNWQ refs bumped, HintPaths migrated to net48, Polly v7 stragglers removed |
| App.config | 16 | AppMetrics binding redirects removed (runtime hygiene) |
| packages.config | 17 | AppMetrics entries removed, DNWQ entries bumped to 0.9.18, Polly v7 stragglers removed |
| .cs | 8 | AppMetrics code stripped + 3 absorption fixes (Metrics?.Dispose removal, 2× AbortWorkerThreadsWhenStopping assignment removal) |
| **Total** | **61** | |

---

## Decisions Made

### Deviation: executed directly in main context, not via builder agent

Three consecutive builder agent dispatches in this project (Phase 1 research, Phase 2 build, Phase 3 build) hit their internal turn budgets before completing. Phase 3 was executed directly in the main planning context as a result. Every PLAN-3.1.md acceptance criterion was still exercised — the execution path was different but the contract was preserved. The transform.py script (pre-written by the terminated Phase 3 builder) was reused. Documented as a lesson for future phases: for heavy multi-file work, the builder agent model may need to be revisited.

### CRLF discipline required three layers of fix

Three separate tools produced LF-only output in this phase: ElementTree's `ET.write()` (XML files), a Python regex sub (7 .cs files), and the atomic sed fix itself. The idempotent sed pattern `s/\r$//; s/$/\r/` (strip then re-add) cleaned all of them up. The `.gitattributes` from Phase 1 produces noisy "LF will be replaced by CRLF" warnings but does NOT auto-fix the working tree on `git add`. Future phases touching text files via non-native-Windows tools MUST apply this sed fix before staging.

### Orphaned DLL HintPaths (`Aq.ExpressionJsonSerializer`, `Schyntax`) were left as HintPath bumps, not as `<Reference>` deletions

These DLLs are vendored inside the DotNetWorkQueue package's `lib\` folder, not as separate NuGet packages. When DNWQ 0.9.18 was restored, the same DLLs appeared at `packages\DotNetWorkQueue.0.9.18\lib\net48\` (confirmed because the build succeeded after the HintPath bump). Alternative would have been to delete the Reference entries entirely and trust MSBuild assembly resolution — but since the bump worked, we kept the minimal edit.

### AbortWorkerThreadsWhenStopping parameter kept in method signature

Only the assignment line was deleted. The `abortWorkerThreadsWhenStopping` method parameter stays in `ConsumeMessage.cs` and `ConsumeMessageAsync.cs`. Reason: these methods are shell commands discovered via reflection (per the architecture map), and any user who types `ConsumeMessage queueName ... false` at the shell still expects to pass a value for this parameter. Removing the parameter would break shell backward compatibility. The parameter is now silently ignored. An alternative would be to rename it to `_unused` or similar, but that adds noise for zero runtime benefit.

### ViewMetrics() and EnableMetrics() public method removal changes shell surface

The reflection-driven shell no longer exposes `EnableMetrics` and `ViewMetrics` commands at runtime. Users typing either at the shell prompt will now see "Command not found". This is the correct frozen state — the examples no longer demonstrate metrics.

---

## Issues Encountered

### Plan defect 1: `ET.register_namespace` doesn't reliably preserve default namespace on ElementTree write

Symptom: every element in every transformed csproj came back prefixed with `ns0:`. The plan's Python approach assumed register_namespace worked, but it didn't. Mitigation: post-process sed to strip `ns0:` prefixes. Lesson: for XML manipulation of files with default namespaces, lxml is more reliable than ElementTree, or use a text-only approach.

### Plan defect 2: `msbuild /t:Restore` is a no-op for packages.config projects

Symptom: `msbuild ... /t:Restore` printed "Nothing to do" after the cache purge. Build then failed to find any DLLs. Mitigation: downloaded standalone `nuget.exe` (8.2 MB) from dist.nuget.org and used `nuget restore`. Lesson: for classic csproj + packages.config, always use `nuget.exe restore`, NOT `msbuild /t:Restore`.

### Plan defect 3: transform.py missed orphaned HintPaths for vendored DLLs

Symptom: Post-transform grep found `DotNetWorkQueue.0.8.0` still present in 16 csproj files. Root cause: the transform's HintPath bump only fired on `<Reference>` blocks whose `Include` attribute started with `DotNetWorkQueue`, missing `<Reference Include="Aq.ExpressionJsonSerializer">` and `<Reference Include="Schyntax">` whose HintPaths also referenced the DNWQ package folder. Mitigation: separate sed pass that bumped the path segment regardless of Reference Include attribute.

### Non-metrics breaking change absorbed: `AbortWorkerThreadsWhenStopping` removed from `IWorkerConfiguration`

DNWQ 0.9.13 changelog note: "Remove `Thread.Abort()` and `AbortWorkerThreadsWhenStopping` config; worker shutdown uses `CancellationToken` only." The example code still set this property. Mitigation: delete the assignment lines. This was the ONLY non-metrics API break that surfaced during the 0.8.0 → 0.9.18 absorption — the rest of the example API surface (`QueueContainer<TInit>`, `QueueCreationContainer<TInit>`, `IProducerBaseQueue`, `QueueMessage<T, IAdditionalMessageData>`, `QueueConnection`, `ConsumerQueueTypes`) was stable across the 10 minor releases.

### NuGet restore was surprisingly fast

Only ~5 seconds elapsed between the `nuget restore` invocation and its exit-0 return. That's much faster than the "several minutes" the plan predicted. Possible explanations: (a) NuGet global cache (`~/.nuget/packages/` on Windows) was already warm from prior system use, (b) only packages that genuinely needed bumping were re-downloaded, (c) the Polly v7 straggler removal shrunk the actual download set. Either way: the ~4-minute prediction was pessimistic.

### Missing `Metrics?.Dispose()` removal (caught by build)

Task 2 removed `Metrics` field declarations and the `ViewMetrics`/`EnableMetrics` methods but missed a single-line `Metrics?.Dispose();` call in the Dispose() pattern. The build caught it immediately; the absorption guard spent 1 rebuild iteration on the fix.

---

## Verification Results

Re-ran the plan's `## Verification` shell block post-commit:

```
grep -rl 'DotNetWorkQueue\.0\.8\.0' Source/Examples/ --include='*.csproj' --include='packages.config'
→ (empty) DNWQ 0.8.0 CLEAR

grep -rl 'App\.Metrics' Source/Examples/ --include='*.cs' --include='*.csproj' --include='packages.config' --include='App.config' --include='app.config'
→ (empty) APPMETRICS CLEAR

grep -rl 'DotNetWorkQueue\.0\.9\.18' Source/Examples/ --include='*.csproj' | wc -l
→ 16 (all 16 csproj that reference DNWQ are at 0.9.18; the 4 shared library csproj do not reference DNWQ at all)

grep -rlE 'targetFramework="net(472|452)"' Source/Examples/ --include='packages.config'
→ (empty) PHASE 2 TFMs CLEAR (no regression)

Build: 0 Warning(s), 0 Error(s) — matches Phase 1 baseline
```

### Plan acceptance criteria

| Task | Criterion | Result |
|---|---|---|
| 1 | Zero csproj with `<Reference Include="App.Metrics` | PASS |
| 1 | Zero packages.config with `<package id="App.Metrics` | PASS |
| 1 | Zero files reference `DotNetWorkQueue.AppMetrics` | PASS |
| 1 | Zero stale DNWQ 0.8.0 assembly identity in csproj | PASS |
| 1 | Zero `DotNetWorkQueue.0.8.0` folder references in csproj/packages.config | PASS (after orphaned HintPath fix) |
| 1 | Zero Polly v7 stragglers in packages.config | PASS |
| 1 | DNWQ 0.9.18 present in csproj | PASS (16 csproj) |
| 1 | `/tmp/phase3/transform.py` exists, NOT committed, NOT under Source/ | PASS |
| 2 | Zero AppMetrics symbols in any example .cs file | PASS |
| 2 | SharedCommands.cs has no Metrics/_metricsRoot/MetricsBuilder/ReportRunner | PASS |
| 2 | All 8 .cs files staged | PASS |
| 3 | `packages/` regenerated with `DotNetWorkQueue.0.9.18/` | PASS (125 packages) |
| 3 | No `DotNetWorkQueue.0.8.0/` or `App.Metrics.*/` folders | PASS |
| 3 | `msbuild /t:Build` exits 0 | PASS |
| 3 | Warning count ≤ 0 (Phase 1 baseline) | PASS |
| 3 | Error count = 0 | PASS |
| 3 | Exactly one new commit with expected subject | PASS (30298fd) |
| 3 | `git status --short` clean after commit | PASS (only CLAUDE.md untracked) |

---

## Phase 3 Verdict

**GO — Phase 4 may begin.**

- DotNetWorkQueue pinned at 0.9.18 across all 16 runnable projects that reference it.
- AppMetrics fully stripped: 0 references in any .cs, .csproj, packages.config, or App.config file.
- Polly v7 stragglers removed.
- Build baseline preserved: 0 warnings, 0 errors — identical to Phase 1.
- Atomic single commit on main.
- Absorption budget well within policy (3 of 10 files, 1 of 3 iterations, <1 min of 30 min).
- Three plan defects documented for retrospective (ElementTree namespace, MSBuild restore scope, orphaned HintPaths).
- One non-metrics breaking change absorbed (AbortWorkerThreadsWhenStopping).
- No structural breaks surfaced — the absorption policy was the right call.
