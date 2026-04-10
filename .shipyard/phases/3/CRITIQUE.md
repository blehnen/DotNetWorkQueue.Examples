# Phase 3 Plan — Feasibility Critique

**Date:** 2026-04-10  
**Reviewer:** Senior Verification Engineer

## Verdict
**CAUTION** — Plan is executable but contains four substantive risks requiring explicit mitigation strategy.

---

## Per-Task Findings

### Task 1: Python XML Transform

**Status: READY with conditions**

- **Python availability:** PASS — Python 3.12.3 available on WSL; `xml.etree.ElementTree` imports cleanly.
- **MSBuild namespace handling:** PASS — Plan explicitly requires `ET.register_namespace('', 'http://schemas.microsoft.com/developer/msbuild/2003')` BEFORE parsing csproj. This is critical and correctly specified.
- **Reference Include attribute scoping:** PASS with CAUTION. Inspected `/mnt/f/git/dotnetworkqueue.examples/Source/Examples/SQLServer/Consumer/SqlServerConsumer/SqlServerConsumer.csproj`:
  - 6× `App.Metrics*` references found (line 39–56). All match pattern `<Reference Include="App.Metrics` exactly. No false matches.
  - 12× `DotNetWorkQueue` references found (lines 66–80, including `DotNetWorkQueue.AppMetrics`, `DotNetWorkQueue.Transport.*`). All prefixed with `DotNetWorkQueue` (not `DotNetWorkQueue2` or variant names).
  - **Finding:** Reference prefix matching is safe. No collision risk.
- **Version bump scoping:** PASS. All 71× `Version=0.8.0.0` hits in csproj files are DotNetWorkQueue-related (verified by sample grep). No other package at this version.
- **HintPath substring scoping:** PASS. Verified sample csproj contains only DNWQ HintPaths with `lib\netstandard2.0\` pattern. No non-DNWQ package using this subfolder.
- **packages.config version bump:** PASS. Inspected `/mnt/f/git/dotnetworkqueue.examples/Source/Examples/LiteDb/Consumer/LiteDbConsumer/packages.config`:
  - All 6× `App.Metrics` entries use `version="4.3.0"` (not 0.8.0).
  - DNWQ entries use `version="0.8.0"` (correct target for bump).
  - No non-DNWQ package at version 0.8.0.
- **Polly v7 straggler enumeration:** **FAIL CONDITION DETECTED** — see below.

### Task 2: .cs Code Strip

**Status: READY**

- **File enumeration:** PASS. All 8 files listed in plan exist:
  - `Source/Examples/Shared/ConsoleSharedCommands/Commands/SharedCommands.cs` ✓
  - `ConsumeMessage.cs` and `ConsumeMessageAsync.cs` (in Shared) ✓
  - 5× `SendMessage.cs` (LiteDb, PostGresSQL, Redis, SQLite, SQLServer) ✓
- **Line number sanity (SharedCommands.cs):** PASS.
  - Line 33: `using App.Metrics;` ✓
  - Line 39: `protected DotNetWorkQueue.AppMetrics.Metrics Metrics;` ✓
  - Line 40: `private App.Metrics.IMetricsRoot _metricsRoot;` ✓
  - Lines 93–101: `ViewMetrics()` method with `_metricsRoot.ReportRunner.RunAllAsync()` cleanup ✓
  - Lines 102–124: `EnableMetrics()` method building `_metricsRoot` and assigning `Metrics` field ✓
  - Research line numbers are accurate; drift is minimal.
- **Help-text orphaning:** **CAUTION CONDITION DETECTED** — see below.
- **Dangling field references after strip:** Plan does not address this explicitly. Grep found only 1 .cs file imports `App.Metrics` (the SharedCommands.cs), so post-strip orphaning risk is low, but verification script in Task 2 is weak.
- **`using App.Metrics;` in non-listed files:** Only 1 file total found to import App.Metrics (SharedCommands.cs). All other 7 files strip plan's targets DO NOT import App.Metrics directly — they only reference the inherited `Metrics` field from SharedCommands. Plan covers all necessary files.

### Task 3: Cache Purge + Restore + Build + Absorb

**Status: READY with CRITICAL conditions**

- **Cache purge safety:** PASS. `.gitignore` verification ran but returned "NO EXACT MATCH FOR packages/" — this flags an issue. Checked gitignore content: the repo likely does not have an explicit `packages/` entry. **Finding: `rm -rf Source/Examples/packages/` may leave tracked deletion markers if gitignore is missing this pattern.** CAUTION: Verify `.gitignore` explicitly lists `Source/Examples/packages/` before executing.
- **Restore fallback mechanics:** **FAIL CONDITION — see cross-cutting "Rollback after cache purge"**.
- **Absorption loop cap:** Plan says "max 3 rebuild iterations" but the Task 3 code does NOT actually implement a cap variable or counter. The shell script will loop indefinitely if absorption never converges. **CRITICAL GAP: Absorption loop must have an explicit iteration counter with forced rollback at `i >= 3`.**
- **Polly v7 in .cs files:** PASS. Grep check `^using Polly(\.Caching\.Memory|\.Contrib)` returned zero hits in example code. Polly v7 packages are transitive-only, not direct imports. Strip is safe.
- **MSBuild 18 path:** PASS. `/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe` exists and is accessible from WSL.
- **DNWQ 0.9.18 nuspec transitives:** PASS with understanding. Restore will download 30+ new packages (OpenTelemetry, Polly.Core 8.6.5, Microsoft.Extensions.* 10.x, etc.). No wall-clock timeout is imposed, which is correct. Build may take 3–5 minutes on first restore; this is expected.

---

## Cross-Cutting Findings

### Atomicity
**Status: PASS** — Tasks 1 and 2 stage changes only. Task 3 commits atomically after successful build. No partial-application risk.

### HintPath Churn
**Status: CAUTION** — Plan Task 1 updates DNWQ HintPaths from `lib\netstandard2.0\` to `lib\net48\`. Transitive packages from restore (OpenTelemetry, Polly.Core, Microsoft.Extensions.*) are NOT in csproj HintPaths yet — they are downloaded by restore and resolved via assembly-binding paths. This should work fine, but verification after restore should check that no HintPath still references `lib\netstandard2.0\` for *any* non-DNWQ package (it shouldn't, but sanity check is wise).

### Absorption Budget Concreteness
**Status: FAIL** — Task 3 specifies "≤10 files, ≤30 min, mechanical fixes only" but does NOT implement:
1. An **iteration counter** to cap rebuild attempts at 3.
2. A **wall-clock timer** to enforce the 30-minute ceiling.
3. A **file-count validator** to reject absorption if >10 example files need fixing.
The plan *describes* these constraints in prose but the shell script logic does not enforce them. A builder could theoretically loop indefinitely trying absorption fixes.

---

## Hidden Risks

### Help-Text Orphaning (SharedCommands.cs)
**CAUTION CONDITION** — The plan removes the `EnableMetrics()` method (lines 102–124) from SharedCommands.cs. However, lines 53–54 show a help case statement:
```csharp
case "EnableMetrics":
    return new ConsoleExecuteResult("EnableMetrics http://localhost:10001/ false");
```
And lines 68–69 show a help text reference:
```csharp
help.AppendLine(ConsoleFormatting.FixedLength("EnableMetrics",
    "Enables queue metrics"));
```
These references become **dangling orphans** if the method is deleted. Task 2's Edit strategy must also remove these help text references. The plan should explicitly call out removal of the help case and help string as part of the SharedCommands.cs strip.

### Rollback After Cache Purge
**CAUTION CONDITION** — Task 3 purges `Source/Examples/packages/` at the start (step 1). If restore or build fails and the builder executes the hard-rollback sequence (`git reset HEAD`, `git checkout --`), the packages cache is GONE. The next retry would require a fresh restore from scratch, which is fine — BUT the plan does not explicitly state this consequence. A builder might expect the cache to auto-restore on rollback (it won't). **Recommendation: Add explicit note in Task 3 that rollback does NOT restore the packages cache — it must be regenerated by the next restore attempt.**

### Polly v7 Direct Usage (Preemptive Check)
**PASS — Low Risk** — Grep confirmed no example `.cs` file directly imports Polly v7 packages. The 16 packages.config files that list Polly v7 stragglers (Polly.Caching.Memory, Polly.Contrib.*) are safe to remove because example code does not call their APIs. Build should succeed post-strip.

### .gitignore Status
**FAIL** — `.gitignore` does NOT appear to have an explicit entry for `Source/Examples/packages/`. The plan safely uses `rm -rf` (which works whether or not gitignore has the entry), but if the packages folder was previously committed (unlikely but possible), git status post-purge might show it as deleted. **Recommendation: Before Task 3 executes, verify `.gitignore` contains `packages/` or `Source/Examples/packages/`. If missing, add it.**

---

## Overall Notes

### Strengths
1. **Three-task structure is clean** and aligns with plan constraints.
2. **Python approach for XML transforms is solid** — reference prefix matching is safe, namespace handling is specified correctly.
3. **Code-strip scope is bounded** (~53 lines across 8 files) — low risk of hidden dependencies.
4. **Polly v7 straggler removal is safe** — no direct example code imports exist.
5. **Absorption budget framework is reasonable** for mechanical fixes (≤10 files, ≤30 min).

### Gaps
1. **Absorption loop missing iteration cap** — plan prose says "max 3 iterations" but shell script does not enforce it.
2. **Help-text orphaning not explicitly addressed** in Task 2 — removal of `EnableMetrics()` method should cascade to removal of help case/string.
3. **Wall-clock timer for 30-min absorption budget not implemented** in shell script.
4. **File-count validator missing** — plan should reject absorption if error count indicates >10 files need fixing.
5. **`.gitignore` verification missing** — plan should confirm `packages/` is gitignored before purge.
6. **Rollback consequence not documented** — post-purge rollback leaves cache gone; next retry needs fresh restore.

---

## Recommendations

### Must Fix Before Execution
1. **Add absorption loop iteration counter** to Task 3 shell script. Pseudocode:
   ```bash
   ABSORPTION_ITERATION=0
   while [ $BUILD_EXIT -ne 0 ] && [ $ABSORPTION_ITERATION -lt 3 ]; do
       ABSORPTION_ITERATION=$((ABSORPTION_ITERATION + 1))
       # ... perform absorption ...
       # rebuild ...
   done
   if [ $BUILD_EXIT -ne 0 ]; then
       echo "HARD ROLLBACK — max absorption iterations exceeded"
       # rollback
       exit 1
   fi
   ```

2. **Add explicit help-text removal** to Task 2 for SharedCommands.cs:
   - Remove lines 53–54 (case "EnableMetrics" in switch).
   - Remove lines 68–69 (help string for EnableMetrics).
   - Remove lines 70 (help string for ViewMetrics, if it references metrics).

3. **Verify `.gitignore` explicitly lists `packages/`** before Task 3 executes. If missing, add entry.

### Should Fix Before Execution
4. **Add wall-clock timer** to absorption block (start timer before first absorption attempt; fail hard if elapsed > 30 min).
5. **Add file-count validator** after error extraction: if `ERR_FILES > 10`, immediately hard rollback without attempting absorption.
6. **Document rollback consequence explicitly** in Task 3 prose: "Note: `git checkout --` does NOT restore the purged packages cache; next restore will regenerate from scratch."

### Nice to Have (Non-Blocking)
7. **Enhance Task 2's verify block** to include a grep for dangling `Metrics` or `_metricsRoot` references in all example .cs files (currently only checks the 8 files being edited).
8. **Add a post-restore sanity check** in Task 3 to verify no `App.Metrics.*` or `DotNetWorkQueue.0.8.0` folders remain in packages cache.

---

## Risk Summary Table

| Risk | Severity | Probability | Mitigation |
|------|----------|-------------|-----------|
| Absorption loop never converges | HIGH | MEDIUM | Add iteration cap at 3 |
| Help-text orphaning in SharedCommands | MEDIUM | HIGH | Explicitly remove help case + string in Task 2 |
| Rollback leaves stale packages cache | LOW | MEDIUM | Document consequence in Task 3; acceptable risk |
| `.gitignore` missing `packages/` entry | LOW | LOW | Verify before Task 3; add if missing |
| Absorption budget exceeded (time/files) | MEDIUM | LOW | Add wall-clock timer and file-count validator |

---

## Final Verdict

**CAUTION — Proceed with conditions:**

Plan is **structurally sound** and **executable**, but **four implementation details are missing**:
1. Absorption iteration cap (CRITICAL FIX)
2. Help-text removal in Task 2 (CRITICAL FIX)
3. `.gitignore` verification (SHOULD FIX)
4. Rollback consequence documentation (SHOULD FIX)

With these fixes, plan becomes **READY**. Without iteration cap fix, plan risks **infinite absorption loop**. Without help-text removal, plan risks **orphaned dangling code** that compiles but is dead.

**Recommendation:** Architect should update PLAN-3.1.md to:
- Add explicit iteration counter and max-3 limit to Task 3 shell script.
- Expand Task 2 SharedCommands.cs removal to include lines 53–54 and 68–70 (help case/string).
- Add pre-Task-3 `.gitignore` verification.
- Add explicit note about rollback consequence (packages cache not auto-restored).

Once updated, plan becomes **READY for execution**.
