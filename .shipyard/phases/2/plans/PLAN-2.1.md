---
phase: phase-2-tfm-bump
plan: 2.1
wave: 1
dependencies: [phase-1]
must_haves:
  - All 20 csproj files under Source/Examples/ target v4.8
  - All 16 App.config files declare sku=".NETFramework,Version=v4.8"
  - All 17 packages.config files use targetFramework="net48"
  - Solution restores and builds in Release with zero errors and zero warnings (Phase 1 baseline)
  - Single atomic commit with subject shipyard(phase-2):
  - SharedAssemblyInfo.cs untouched
  - HintPaths untouched (deferred to Phase 3)
files_touched:
  - Source/Examples/**/*.csproj
  - Source/Examples/**/App.config
  - Source/Examples/**/packages.config
tdd: false
risk: medium
---

# Plan 2.1: Phase 2 TFM Bump to net48

## Context

This plan executes Phase 2 of the `examples-freeze` project: bumping every example
project from .NET Framework 4.7.2 / 4.5.2 to .NET Framework 4.8. The scope, file
counts, exact sed commands, and risk assessment come from
`.shipyard/phases/2/RESEARCH.md`; the execution shape (work on `main` directly, sed
mass-edit, HintPaths deferred, SharedAssemblyInfo untouched, pre-commit build
verification mandatory, 3-task split: csproj stage / config stage / verify+commit)
comes from `.shipyard/phases/2/CONTEXT-2.md`. Phase 1 (spike + baseline build)
completed cleanly with 0 warnings and 0 errors — that becomes the hard acceptance
bar for this phase per the NFR "Baseline warning discipline".

Research confirms no surprises: 16 csproj at v4.7.2, 4 at v4.5.2, 16 App.config at
v4.7.2 sku (15 runnable + `Shared/ConsoleSharedCommands/app.config` lowercase —
the `-iname` sed handles the case-mismatch automatically), 937 package entries at
`net472` across 16 files, 1 entry at `net452` in `ConsoleView/packages.config` —
53 files touched in total. No SDK-style projects, no condition-gated TFM blocks,
no `TargetFrameworks` (plural) to worry about.

## Dependencies

Phase 1 complete (build baseline captured, 0 warnings / 0 errors). No intra-phase
dependencies — this is the only plan in Phase 2, single wave.

## Tasks

<task id="1" files="Source/Examples/**/*.csproj" tdd="false">
  <action>
Mass-edit all 20 .csproj files under Source/Examples/ to replace their
&lt;TargetFrameworkVersion&gt; element with v4.8, then stage (but do NOT commit).
Use the exact sed commands from RESEARCH.md so behavior is reproducible:

```bash
cd /mnt/f/git/dotnetworkqueue.examples

find Source/Examples -name '*.csproj' -exec sed -i \
  's|<TargetFrameworkVersion>v4\.7\.2</TargetFrameworkVersion>|<TargetFrameworkVersion>v4.8</TargetFrameworkVersion>|g' {} +

find Source/Examples -name '*.csproj' -exec sed -i \
  's|<TargetFrameworkVersion>v4\.5\.2</TargetFrameworkVersion>|<TargetFrameworkVersion>v4.8</TargetFrameworkVersion>|g' {} +

git add Source/Examples/**/*.csproj
```

Do NOT touch SharedAssemblyInfo.cs. Do NOT touch HintPaths. Do NOT bump package
versions. Do NOT commit yet — the commit happens at the end of Task 3 after
successful build verification.
  </action>
  <verify>
```bash
# Expect empty output (no stale TFMs left)
grep -rlE '<TargetFrameworkVersion>v4\.(7\.2|5\.2)</TargetFrameworkVersion>' \
  Source/Examples/ --include='*.csproj' || echo "CSPROJ TFM OK"

# Expect 20 matching files
grep -rl '<TargetFrameworkVersion>v4.8</TargetFrameworkVersion>' \
  Source/Examples/ --include='*.csproj' | wc -l

# Expect exactly 20 csproj files staged as modified and nothing else
git status --short | grep -vE '^M  Source/Examples/.*\.csproj$|^\?\? CLAUDE\.md$' | wc -l
git diff --cached --name-only | grep -c '\.csproj$'
```
  </verify>
  <done>
- `grep` for `v4.7.2`/`v4.5.2` TargetFrameworkVersion across Source/Examples returns empty.
- `grep` for `v4.8` TargetFrameworkVersion returns 20 files.
- `git diff --cached --name-only | grep -c '\.csproj$'` returns 20.
- No non-csproj files are modified (only untracked CLAUDE.md is tolerated).
- No commit has been made yet.
  </done>
</task>

<task id="2" files="Source/Examples/**/App.config, Source/Examples/**/packages.config" tdd="false">
  <action>
Mass-edit all App.config and packages.config files under Source/Examples/ and stage.
Use the exact sed commands from RESEARCH.md:

```bash
cd /mnt/f/git/dotnetworkqueue.examples

# App.config supportedRuntime sku (use -iname in case of mixed casing)
find Source/Examples -iname 'App.config' -exec sed -i \
  's|sku="\.NETFramework,Version=v4\.7\.2"|sku=".NETFramework,Version=v4.8"|g' {} +

find Source/Examples -iname 'App.config' -exec sed -i \
  's|sku="\.NETFramework,Version=v4\.5\.2"|sku=".NETFramework,Version=v4.8"|g' {} +

# packages.config targetFramework attribute
find Source/Examples -name 'packages.config' -exec sed -i \
  's|targetFramework="net472"|targetFramework="net48"|g' {} +

find Source/Examples -name 'packages.config' -exec sed -i \
  's|targetFramework="net452"|targetFramework="net48"|g' {} +

git add Source/Examples/**/App.config Source/Examples/**/packages.config
```

Do NOT touch any other files. Do NOT commit yet.
  </action>
  <verify>
```bash
# Expect empty — no stale sku values in App.config
grep -rlE 'sku="\.NETFramework,Version=v4\.(7\.2|5\.2)"' \
  Source/Examples/ --include='App.config' --include='app.config' \
  || echo "APP.CONFIG SKU OK"

# Expect empty — no stale targetFramework values in packages.config
grep -rlE 'targetFramework="net(472|452)"' \
  Source/Examples/ --include='packages.config' \
  || echo "PACKAGES.CONFIG TFM OK"

# Expect 16 App.config files now declare v4.8 sku
grep -rl 'sku="\.NETFramework,Version=v4\.8"' \
  Source/Examples/ --include='App.config' --include='app.config' | wc -l

# Expect 938 net48 entries across packages.config files (937 from net472 + 1 from net452)
grep -rE 'targetFramework="net48"' \
  Source/Examples/ --include='packages.config' | wc -l

# Expect 53 files staged in total (20 csproj + 16 App.config + 17 packages.config)
git diff --cached --stat | tail -1
git diff --cached --name-only | wc -l
```
  </verify>
  <done>
- Zero App.config files contain `sku=".NETFramework,Version=v4.7.2"` or `v4.5.2`.
- Zero packages.config files contain `targetFramework="net472"` or `net452"`.
- 16 App.config files contain `sku=".NETFramework,Version=v4.8"`.
- 938 lines across packages.config files contain `targetFramework="net48"`.
- `git diff --cached --name-only | wc -l` reports 53.
- No commit has been made yet.
  </done>
</task>

<task id="3" files="(no source edits; produces a commit and logs under /tmp/phase2/)" tdd="false">
  <action>
Capture the pre-verification diff summary, run NuGet restore and Release build via
the Phase-1 validated MSBuild invocation, enforce the Phase 1 baseline warning
discipline, and commit atomically only if restore + build succeed and the warning
count is not greater than the Phase 1 baseline (0).

```bash
cd /mnt/f/git/dotnetworkqueue.examples
mkdir -p /tmp/phase2

# Snapshot staged changes
git diff --cached --stat > /tmp/phase2/staged-stat.txt
git diff --cached --name-only > /tmp/phase2/staged-files.txt
wc -l /tmp/phase2/staged-files.txt  # expect 53

MSBUILD='/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe'
SLN_WIN="$(wslpath -w Source/Examples/Examples.sln)"

# Restore
"$MSBUILD" "$SLN_WIN" /t:Restore /p:Configuration=Release /v:minimal \
  > /tmp/phase2/restore.log 2>&1
RESTORE_EXIT=$?
echo "restore exit: $RESTORE_EXIT"

if [ "$RESTORE_EXIT" -ne 0 ]; then
  echo "RESTORE FAILED — rolling back"
  tail -60 /tmp/phase2/restore.log
  git reset HEAD Source/Examples/
  git checkout -- Source/Examples/
  exit 1
fi

# Build
"$MSBUILD" "$SLN_WIN" /t:Build /p:Configuration=Release /v:normal \
  > /tmp/phase2/build.log 2>&1
BUILD_EXIT=$?
echo "build exit: $BUILD_EXIT"

if [ "$BUILD_EXIT" -ne 0 ]; then
  echo "BUILD FAILED — rolling back"
  tail -80 /tmp/phase2/build.log
  git reset HEAD Source/Examples/
  git checkout -- Source/Examples/
  exit 1
fi

# Parse warning/error counts from the MSBuild summary
WARN_LINE=$(grep -E '^\s*[0-9]+ Warning\(s\)' /tmp/phase2/build.log | tail -1)
ERR_LINE=$(grep  -E '^\s*[0-9]+ Error\(s\)'   /tmp/phase2/build.log | tail -1)
echo "$WARN_LINE"
echo "$ERR_LINE"

WARN_COUNT=$(echo "$WARN_LINE" | grep -oE '[0-9]+' | head -1)
ERR_COUNT=$(echo  "$ERR_LINE"  | grep -oE '[0-9]+' | head -1)

# Baseline warning discipline: must not exceed the Phase 1 baseline of 0
if [ "${ERR_COUNT:-1}" -ne 0 ] || [ "${WARN_COUNT:-1}" -gt 0 ]; then
  echo "BASELINE VIOLATION — errors=$ERR_COUNT warnings=$WARN_COUNT (baseline=0/0)"
  git reset HEAD Source/Examples/
  git checkout -- Source/Examples/
  exit 1
fi

# Atomic commit
git commit -m "shipyard(phase-2): bump all projects to .NET Framework 4.8"
```

If any step fails, the rollback sequence (`git reset HEAD` + `git checkout --`)
restores the working tree to its pre-plan state and the plan is escalated. No
partial state is ever committed.
  </action>
  <verify>
```bash
# Exits captured above must both be 0
echo "restore exit: $RESTORE_EXIT, build exit: $BUILD_EXIT"

# Baseline warning discipline
grep -E '[0-9]+ (Warning|Error)\(s\)' /tmp/phase2/build.log | tail -2

# Exactly one new commit on main with the expected subject
git log -1 --pretty='%h %s'

# Working tree clean (CLAUDE.md untracked is tolerated)
git status --short | grep -vE '^\?\? CLAUDE\.md$' | wc -l

# Post-commit: no stale TFM tokens remain anywhere in the examples tree
grep -rlE 'v4\.7\.2|v4\.5\.2|net472|net452' Source/Examples/ \
  --include='*.csproj' --include='App.config' --include='app.config' \
  --include='packages.config' || echo "POST-COMMIT SWEEP OK"
```
  </verify>
  <done>
- `/t:Restore` exited 0; `/tmp/phase2/restore.log` saved.
- `/t:Build /p:Configuration=Release` exited 0; `/tmp/phase2/build.log` saved.
- Build summary reports `0 Error(s)` and `0 Warning(s)` — matches Phase 1 baseline.
- Exactly one new commit on `main` with subject `shipyard(phase-2): bump all projects to .NET Framework 4.8`.
- `git status --short` (ignoring untracked CLAUDE.md) is empty.
- Post-commit grep for `v4.7.2|v4.5.2|net472|net452` across csproj / App.config / packages.config under `Source/Examples/` returns empty.
  </done>
</task>

## Verification

```bash
# Post-build checks
MSBUILD='/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe'
SLN_WIN="$(wslpath -w Source/Examples/Examples.sln)"

# All projects declare v4.8
grep -l '<TargetFrameworkVersion>v4.7.2</TargetFrameworkVersion>\|<TargetFrameworkVersion>v4.5.2</TargetFrameworkVersion>' Source/Examples/**/*.csproj 2>/dev/null || echo "CSPROJ OK"

# No net472/net452 in packages.config
grep -rlE 'targetFramework="net(472|452)"' Source/Examples/ --include='packages.config' || echo "PACKAGES.CONFIG OK"

# No v4.7.2/v4.5.2 sku in App.config
grep -rlE 'sku="\.NETFramework,Version=v4\.(7\.2|5\.2)"' Source/Examples/ --include='App.config' --include='app.config' || echo "APP.CONFIG OK"

# Build clean in Release
"$MSBUILD" "$SLN_WIN" /t:Build /p:Configuration=Release /v:normal 2>&1 | grep -E '[0-9]+ (Warning|Error)\(s\)' | tail -2
```

## Rollback

If restore, build, or the baseline warning check fails in Task 3:

```bash
git reset HEAD Source/Examples/
git checkout -- Source/Examples/
```

No commit is ever made in a failure path, so rollback is local-only — nothing to
revert on `main`.
