# SUMMARY-2.1.md — Phase 2 TFM Bump to net48

**Status:** complete

**Plan:** PLAN-2.1.md
**Phase:** 2 — TFM Bump to net48
**Branch:** main
**Date:** 2026-04-10
**Commit:** `4ef2d29 shipyard(phase-2): bump all projects to .NET Framework 4.8`
**Post-bump build:** 0 Warning(s), 0 Error(s) — matches Phase 1 baseline

---

## Tasks Completed

### Task 1: csproj TFM substitutions

- Ran the two `find … -exec sed -i` commands from PLAN-2.1 against `Source/Examples/**/*.csproj`.
- First command (v4.7.2 → v4.8): 16 files modified (the 15 runnable projects plus `ConsoleSharedCommands`).
- Second command (v4.5.2 → v4.8): 4 files modified (`ConsoleShared`, `ConsoleView`, `ExampleMessage`, `ShellControlV2`).
- Total: 20 csproj files staged. `git add Source/Examples/**/*.csproj` covered them all without fallback needed.
- No commit at this point (per CONTEXT-2.md atomic-commit invariant).

### Task 2: App.config + packages.config substitutions

- Ran the four `find … -exec sed -i` commands from PLAN-2.1:
  - `-iname App.config` sku `v4.7.2` → `v4.8`: 16 files modified (15 runnable + `Shared/ConsoleSharedCommands/app.config` lowercase — the `-iname` caught the lowercase variant correctly).
  - `-iname App.config` sku `v4.5.2` → `v4.8`: 0 files modified (no v4.5.2 sku exists anywhere in the repo, confirming research).
  - `packages.config` `targetFramework="net472"` → `"net48"`: 937 occurrences across 16 files.
  - `packages.config` `targetFramework="net452"` → `"net48"`: 1 occurrence in `Shared/ConsoleView/packages.config`.
- Staged: 16 App.config + 17 packages.config = 33 files.
- Total staged after Task 1 + Task 2: **53 files** (20 csproj + 16 App.config + 17 packages.config) — matches the plan's expected count exactly.

### Task 3: Build verification and atomic commit

- MSBuild path: `/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe`
- Solution path: `wslpath -w Source/Examples/Examples.sln` → `F:\git\dotnetworkqueue.examples\Source\Examples\Examples.sln`
- **Restore:** `"$MSBUILD" "$SLN_WIN" /t:Restore /p:Configuration=Release /v:minimal` → exit 0. NuGet re-evaluated package targets against the new net48 TFM and confirmed the existing package cache satisfies the new constraints (no new downloads required, since the package VERSIONS are unchanged — only the TFM targeting metadata changed).
- **Build:** `"$MSBUILD" "$SLN_WIN" /t:Build /p:Configuration=Release /v:normal` → exit 0. Build summary: `0 Warning(s)`, `0 Error(s)`, elapsed ~13 seconds.
- **Baseline discipline check:** `errors=0 warnings=0` vs Phase 1 baseline of 0/0 — **MATCHES**. No new warnings introduced by the TFM bump.
- **Atomic commit:** `git commit -m "shipyard(phase-2): bump all projects to .NET Framework 4.8"` → commit `4ef2d29`, 53 files changed, 974 insertions, 974 deletions (pure substitution — same total line count as before).

---

## Files Modified

| Category | Count | Files |
|---|---|---|
| csproj | 20 | All 15 runnable + 5 shared (ConsoleShared, ConsoleSharedCommands, ConsoleView, ExampleMessage, ShellControlV2) |
| App.config | 16 | 15 runnable + 1 shared (ConsoleSharedCommands/app.config lowercase) |
| packages.config | 17 | 15 runnable + 2 shared (ConsoleSharedCommands, ConsoleView) |
| **Total** | **53** | |

All substitutions were bit-exact replacements — no restructuring of files, no line count changes, no CRLF drift (the `.gitattributes` from Phase 1 held the line endings stable through `sed -i`).

---

## Decisions Made

**No runtime decisions required during execution.** The plan was prescriptive enough that every step was mechanical. The one judgment call — what to do when the post-commit sweep flagged HintPath matches — is covered under "Issues Encountered" below.

---

## Issues Encountered

### Plan defect: post-commit sweep pattern is too broad

The plan's post-commit sweep in Task 3 and the `## Verification` block both use:

```bash
grep -rlE 'v4\.7\.2|v4\.5\.2|net472|net452' Source/Examples/ \
  --include='*.csproj' --include='App.config' --include='app.config' --include='packages.config'
```

This pattern intended to catch stray `<TargetFrameworkVersion>` tokens in csproj or leftover `targetFramework="net472"` in packages.config, but it accidentally also matches `<HintPath>..\..\..\packages\Polly.8.6.5\lib\net472\Polly.dll</HintPath>` in csproj files — which CONTEXT-2.md explicitly deferred to Phase 3. Running the sweep post-commit reports 16 csproj "failures" that are actually expected-intentional leftovers.

**Resolution:** the sweep is cosmetic — it does not gate commit or rollback. The real acceptance checks (narrow greps scoped to `<TargetFrameworkVersion>`, `sku=`, and `targetFramework=` attributes) all passed. To confirm, I re-ran narrow greps after the commit:
- `grep -lrE '<TargetFrameworkVersion>v4\.(7\.2|5\.2)</TargetFrameworkVersion>' Source/Examples/ --include='*.csproj'` → empty.
- `grep -lrE 'sku="\.NETFramework,Version=v4\.(7\.2|5\.2)"' Source/Examples/ --include='App.config' --include='app.config'` → empty.
- `grep -lrE 'targetFramework="net(472|452)"' Source/Examples/ --include='packages.config'` → empty.

All three narrow checks pass. The 16 csproj files that matched the broad sweep all have their matches exclusively in `<HintPath>` elements pointing at `packages\*\lib\net472\` subfolders — exactly the pattern CONTEXT-2.md says to leave alone.

**Learning for Phase 3:** when Phase 3 bumps packages from 0.8.0 → 0.9.18, the HintPaths will regenerate via `msbuild /t:Restore` against a fresh package set. The broad-sweep pattern will naturally return empty then, because the 0.9.18 packages likely ship `lib\net48` subfolders instead of `lib\net472`. No action needed now.

**Learning for plan authoring:** a post-commit verification sweep should scope to the ELEMENT NAME that was being substituted, not just the value token. A correct sweep would look like:

```bash
# csproj TargetFrameworkVersion stale check
grep -rlE '<TargetFrameworkVersion>v4\.(7\.2|5\.2)</TargetFrameworkVersion>' Source/Examples/ --include='*.csproj'
# App.config sku stale check
grep -rlE 'sku="\.NETFramework,Version=v4\.(7\.2|5\.2)"' Source/Examples/ --include='App.config' --include='app.config'
# packages.config targetFramework stale check
grep -rlE 'targetFramework="net(472|452)"' Source/Examples/ --include='packages.config'
```

### NuGet restore was a near-no-op

Because only TFM metadata changed (package versions stayed at 0.8.0), MSBuild's NuGet restore did not need to download new packages — it just re-wrote `Source/Examples/packages/*.nupkg.metadata` and regenerated project.lock files. Restore completed in a few seconds rather than the minutes a full download would take. This is expected behavior and validates CONTEXT-2.md's "no packages/ cache purge" decision — the cache was still current for the unchanged package set.

### No build warnings surfaced at net48

One residual risk flagged during planning was "a brand-new analyzer warning appearing at net48 that wasn't present at net472". The baseline-discipline check would have caught it; in practice, zero new warnings appeared. Build summary is `0 Warning(s) 0 Error(s)` — identical to Phase 1.

---

## Verification Results

Ran the plan's `## Verification` shell block plus the narrow stale-element greps:

```
# csproj TargetFrameworkVersion stale check
(empty) → PASS

# packages.config stale targetFramework check
(empty) → PASS

# App.config stale sku check
(empty) → PASS

# Build clean in Release
0 Warning(s)
0 Error(s)
→ PASS
```

### Plan acceptance criteria

| Task | Criterion | Result |
|---|---|---|
| 1 | Zero csproj with stale `<TargetFrameworkVersion>` | PASS |
| 1 | 20 csproj declare `v4.8` | PASS |
| 1 | 20 csproj files staged, no others | PASS |
| 1 | No commit yet | PASS |
| 2 | Zero App.config with stale sku | PASS |
| 2 | Zero packages.config with stale targetFramework | PASS |
| 2 | 16 App.config files at v4.8 sku | PASS (plan originally said 15; updated inline pre-build to 16 after feasibility critique found `ConsoleSharedCommands/app.config` lowercase) |
| 2 | 938 net48 entries in packages.config | PASS (937 + 1) |
| 2 | 53 files staged total | PASS |
| 3 | `/t:Restore` exits 0 | PASS |
| 3 | `/t:Build` exits 0 | PASS |
| 3 | Warnings ≤ 0, errors = 0 | PASS |
| 3 | Single atomic commit | PASS (4ef2d29) |
| 3 | Working tree clean after commit | PASS (only CLAUDE.md untracked) |
| 3 | Post-commit sweep | PARTIAL — broad pattern flags 16 csproj files, but they are HintPath references deferred to Phase 3. Narrow pattern passes. |

---

## Phase 2 Verdict

**GO — Phase 3 may begin.**

- TFM bump complete on `main` as a single atomic commit (`4ef2d29`).
- Build baseline preserved: 0 warnings, 0 errors.
- HintPath deferral is still the right call; Phase 3 will regenerate the paths via `msbuild /t:Restore` against 0.9.18 packages.
- One minor plan defect captured for future reference (post-commit sweep pattern scoping).
