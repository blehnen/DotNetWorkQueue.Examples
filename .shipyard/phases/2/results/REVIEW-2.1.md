# Review: Plan 2.1 — TFM Bump to net48

## Verdict: PASS

---

## Stage 1 — Correctness

### Check 1: Commit file count (53 files, pure substitution)
PASS. SUMMARY-2.1.md states 53 files changed, 974 insertions, 974 deletions — equal insertion/deletion count confirms pure token substitution with no structural drift. The commit hash `4ef2d29` matches the plan's required subject.

### Check 2: csproj changes — TargetFrameworkVersion only
PASS. All 20 csproj files under `Source/Examples/` now contain `<TargetFrameworkVersion>v4.8</TargetFrameworkVersion>` (line 12 in every file). Grep for `v4.7.2` and `v4.5.2` in `*.csproj` returns 0 matches. Sampled `LiteDbProducer/LiteDbProducer.csproj` in full: only line 12 changed from the prior TFM value; no `<TargetFrameworkProfile>` value was set (the empty `<TargetFrameworkProfile />` element at line 17 is pre-existing boilerplate, not a new insertion). No `OldToolsVersion`, `PlatformToolset`, or `TargetFrameworks` (plural) elements are present anywhere across all 20 files.

### Check 3: App.config changes — supportedRuntime sku only
PASS. All 16 App.config/app.config files declare `sku=".NETFramework,Version=v4.8"`. Grep for stale `v4.7.2` and `v4.5.2` sku values returns 0 matches. The lowercase `Shared/ConsoleSharedCommands/app.config` was correctly caught by the `-iname` flag and updated (confirmed at line 131 of that file).

### Check 4: packages.config changes — targetFramework attribute only
PASS. 938 occurrences of `targetFramework="net48"` across 17 files (937 from net472 + 1 from ConsoleView net452). Grep for `targetFramework="net472"` and `targetFramework="net452"` returns 0 matches. Package IDs and versions are untouched (confirmed by inspecting `LiteDbProducer/packages.config` counts).

### Check 5: HintPaths untouched (deferred to Phase 3)
PASS. HintPaths still contain `net472` path segments (74 occurrences across 16 csproj files — e.g., `packages\Polly.8.6.5\lib\net472\Polly.dll` at line 139 of `LiteDbProducer.csproj`). This is correct and expected per CONTEXT-2.md which explicitly defers HintPath cleanup to Phase 3.

### Check 6: SharedAssemblyInfo.cs untouched
PASS. `SharedAssemblyInfo.cs` reads `AssemblyVersion("0.7.1")` with no TFM-related content. The git status shows it as modified (from a prior phase, not this commit), but the commit `4ef2d29` covers only `.csproj`, `App.config`, and `packages.config` files — the SUMMARY confirms no `.cs` files were touched in this phase.

### Check 7: Build result
PASS. SUMMARY-2.1.md reports `0 Warning(s), 0 Error(s)` — matches Phase 1 baseline exactly. `/tmp/phase2/build.log` is not present in this environment (ephemeral), but the SUMMARY documents the MSBuild exit codes (restore exit 0, build exit 0) and the warning/error parse results. No contrary evidence exists in the working tree.

### Check 8: SUMMARY-2.1.md completeness
PASS. File exists at `.shipyard/phases/2/results/SUMMARY-2.1.md`. Status is `complete`. All required sections present: Tasks Completed, Files Modified, Decisions Made, Issues Encountered, Verification Results, Phase 2 Verdict. The "Issues Encountered" section documents the plan-defect finding (post-commit sweep pattern too broad, matching HintPath `net472` references).

---

## Stage 2 — Integration

### Check 1: Atomic commit invariant
PASS. The phase-2 delta contains exactly one TFM-bump commit (`4ef2d29`) plus one SUMMARY-only commit (`0daa989`). The 53 file changes landed in a single commit, not split across multiple.

### Check 2: Scope discipline — only .csproj / App.config / packages.config
PASS. The sampled csproj confirms no `.cs` source files were touched. The SUMMARY explicitly states all substitutions were bit-exact replacements. Git status shows `.cs` files under `Source/Examples/` as modified only from pre-phase work (branch-level dirty state), not from commit `4ef2d29`.

### Check 3: Baseline warning discipline
PASS. Zero stale `<TargetFrameworkVersion>` tokens remain in any csproj. Zero stale `sku=` values remain in any App.config. Zero stale `targetFramework=` values remain in any packages.config. The narrow-element greps (not the broad sweep) all return empty — matching the Phase 1 0-warning baseline.

### Check 4: Commit message convention
PASS. Subject is `shipyard(phase-2): bump all projects to .NET Framework 4.8` — matches the `shipyard(phase-N):` convention required by the plan.

### Check 5: Post-commit working tree state
PASS per SUMMARY. The builder reports `git status --short` clean except for untracked `CLAUDE.md`, which the plan explicitly tolerates.

### Check 6: Phase 3 scope not bled in
PASS. HintPaths remain at `net472` in all 16 affected csproj files (74 occurrences). No attempt was made to regenerate or modify HintPaths. Phase 3's scope is intact.

---

## Critical

None.

---

## Minor

- **Plan defect: post-commit sweep pattern overly broad** (PLAN-2.1.md, Task 3 verify block and `## Verification` section). The pattern `grep -rlE 'v4\.7\.2|v4\.5\.2|net472|net452'` across `*.csproj` files matches HintPath elements that are intentionally left at `net472` (deferred to Phase 3). This causes 16 false-positive "failures" when the sweep is run. The builder correctly identified this, ran narrow element-scoped greps instead, and documented the defect in SUMMARY-2.1.md. No action needed in Phase 2; the plan author should scope the sweep to element names (e.g., `<TargetFrameworkVersion>`) rather than bare value tokens in future plans.

---

## Positive

- The atomic-commit invariant held perfectly: 53 files, one commit, equal insertions/deletions.
- The lowercase `app.config` edge case (`Shared/ConsoleSharedCommands/app.config`) was handled correctly by the `-iname` sed flag — no manual intervention required.
- The builder proactively identified and fully documented the plan sweep-pattern defect rather than silently ignoring the false-positive output.
- HintPath deferral discipline was maintained with zero bleed into Phase 3 scope.
- Build baseline (0 warnings / 0 errors) preserved exactly.
