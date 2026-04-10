---
phase: package-bump-appmetrics-strip
plan: 3.1
wave: 1
dependencies: []
must_haves:
  - DotNetWorkQueue packages pinned at 0.9.18 across all csproj + packages.config
  - AppMetrics references removed from csproj, packages.config, App.config, and .cs files
  - Polly v7 straggler packages removed (Polly.Caching.Memory, Polly.Contrib.Simmy, Polly.Contrib.WaitAndRetry)
  - Single atomic commit after clean restore + build
  - Baseline discipline upheld (errors=0, warnings<=0)
files_touched:
  - Source/Examples/**/*.csproj (17 files)
  - Source/Examples/**/packages.config (17 files)
  - Source/Examples/**/App.config (16 files)
  - Source/Examples/Shared/ConsoleSharedCommands/Commands/SharedCommands.cs
  - Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessage.cs
  - Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessageAsync.cs
  - Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/SendMessage.cs
  - Source/Examples/PostGresSQL/Producer/PostGresSQLProducer/Commands/SendMessage.cs
  - Source/Examples/Redis/Producer/RedisProducer/Commands/SendMessage.cs
  - Source/Examples/SQLite/Producer/SQLiteProducer/Commands/SendMessage.cs
  - Source/Examples/SQLServer/Producer/SqlServerProducer/Commands/SendMessage.cs
tdd: false
risk: high
---

# PLAN 3.1 — Package Bump + AppMetrics Strip (atomic)

## Context

Phase 3 pins `DotNetWorkQueue` at **0.9.18** and strips **AppMetrics** from the example set. This is the highest-risk phase of `examples-freeze` because:

1. DNWQ 0.9.18's nuspec pulls a large transitive graph (see `RESEARCH.md`), so restore may surface version conflicts.
2. AppMetrics removal touches both project metadata (XML) **and** example source code (C#).
3. The package bump and code strip must land in **one atomic commit** — partial application leaves the tree un-buildable.
4. Polly v7 stragglers (37 entries across 16 `packages.config`) must be removed unless example `.cs` code directly imports Polly v7 APIs.

The builder works on `main` directly (no worktree — confirmed in ROADMAP). Working tree is clean except `CLAUDE.md`.

**Required reading before starting:**
- `.shipyard/PROJECT.md` (R6, R7, R8, R9, R11, NFR baseline warning discipline)
- `.shipyard/phases/3/CONTEXT-3.md` (absorption budget, atomic-commit rule, Python transform approach)
- `.shipyard/phases/3/RESEARCH.md` (exact XML shapes, line numbers, transitive graph notes)
- `.shipyard/phases/1/SPIKE-NOTES.md` (MSBuild 18 path + `wslpath -w` requirement)
- `.shipyard/phases/2/results/SUMMARY-2.1.md` (current post-Phase-2 baseline)

## Dependencies

- Phase 2 complete (net48 retarget landed, baseline warnings captured).
- MSBuild 18 (`/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe`) available from WSL.
- `python3` with `xml.etree.ElementTree` (stdlib, no pip install needed).

## Tasks

<task id="1" files="/tmp/phase3/transform.py, Source/Examples/**/*.csproj, Source/Examples/**/packages.config, Source/Examples/**/App.config" tdd="false">
  <action>
Write a Python 3 script at `/tmp/phase3/transform.py` (build artifact — NOT committed) that walks `Source/Examples/` and mutates XML files with `xml.etree.ElementTree`. MSBuild requires the default namespace to be preserved; call `ET.register_namespace('', 'http://schemas.microsoft.com/developer/msbuild/2003')` before parsing any csproj.

**csproj transforms** (applied to every `*.csproj` under `Source/Examples/`):
  1. Delete every `<Reference>` element whose `Include` attribute starts with `App.Metrics.` (match prefix before first comma) — expect 6 per file.
  2. Delete every `<Reference>` element whose `Include` starts with `DotNetWorkQueue.AppMetrics` — expect 1 per file.
  3. For remaining `<Reference>` elements whose `Include` starts with `DotNetWorkQueue` or `DotNetWorkQueue.Transport.`, replace the substring `Version=0.8.0.0` with `Version=0.9.18.0` in the `Include` attribute value.
  4. For each such DNWQ `<Reference>`, locate its child `<HintPath>` and replace the package-folder segment `DotNetWorkQueue.0.8.0` with `DotNetWorkQueue.0.9.18` (and the same pattern for every transport variant — match any segment matching `DotNetWorkQueue(\.Transport\.[^\\]+)?\.0\.8\.0`), AND replace the lib subfolder `\lib\netstandard2.0\` with `\lib\net48\`.
  5. Delete `<Reference>` elements whose `Include` starts with `Polly.Caching.Memory`, `Polly.Contrib.Simmy`, or `Polly.Contrib.WaitAndRetry`.

**packages.config transforms** (applied to every `packages.config` under `Source/Examples/`):
  1. Delete every `<package>` element whose `id` attribute starts with `App.Metrics` (expect 6 per file).
  2. Delete every `<package>` element whose `id` equals `DotNetWorkQueue.AppMetrics`.
  3. For every `<package>` element whose `id` starts with `DotNetWorkQueue` and whose `version` equals `0.8.0`, change `version` to `0.9.18`.
  4. Delete `<package>` elements whose `id` is in `{Polly.Caching.Memory, Polly.Contrib.Simmy, Polly.Contrib.WaitAndRetry}`.

**App.config transforms** (applied to every `App.config` under `Source/Examples/`):
  1. Delete every `<dependentAssembly>` block whose child `<assemblyIdentity>`'s `name` starts with `App.Metrics.`. This is hygiene only (runtime binding redirects, not compile-time), but we do it in the same pass since the XML is already parsed.

**Script requirements:**
  - Use `ET.register_namespace('', 'http://schemas.microsoft.com/developer/msbuild/2003')` BEFORE parsing csproj files so the output preserves the default namespace.
  - For the other file types (packages.config, App.config) use an empty/no namespace.
  - Write files back with `tree.write(path, xml_declaration=True, encoding='utf-8')` for csproj and App.config; `packages.config` uses `encoding='utf-8'` as well.
  - Accept the examples root as a CLI argument: `sys.argv[1]`.
  - Print a summary to stdout on success:
    ```
    csproj processed=<N> appmetrics-refs-removed=<X> dnwq-refs-bumped=<Y> polly-refs-removed=<Z>
    packages.config processed=<N> appmetrics-pkgs-removed=<X> dnwq-pkgs-bumped=<Y> polly-pkgs-removed=<Z>
    App.config processed=<N> appmetrics-binding-redirects-removed=<X>
    ```
  - Exit 0 on success, non-zero on any parse or write error.

**Invocation:**
```bash
mkdir -p /tmp/phase3
# (author the script with Write tool — do NOT place it under Source/ or commit it)
python3 /tmp/phase3/transform.py /mnt/f/git/dotnetworkqueue.examples/Source/Examples/ | tee /tmp/phase3/transform.log
```

After the script runs, stage the XML changes — do NOT commit yet:
```bash
cd /mnt/f/git/dotnetworkqueue.examples
git add 'Source/Examples/**/*.csproj' 'Source/Examples/**/packages.config' 'Source/Examples/**/App.config'
```
  </action>
  <verify>
```bash
cd /mnt/f/git/dotnetworkqueue.examples

# 1. No AppMetrics references in csproj
grep -rl '<Reference Include="App.Metrics' Source/Examples/ --include='*.csproj' || echo "csproj-appmetrics CLEAR"

# 2. No AppMetrics entries in packages.config
grep -rl '<package id="App.Metrics' Source/Examples/ --include='packages.config' || echo "pkgcfg-appmetrics CLEAR"

# 3. No DotNetWorkQueue.AppMetrics anywhere in project metadata
grep -rl 'DotNetWorkQueue\.AppMetrics' Source/Examples/ --include='*.csproj' --include='packages.config' || echo "dnwq-appmetrics CLEAR"

# 4. No stale DNWQ 0.8.0.0 assembly-identity version in csproj
grep -rE 'Version=0\.8\.0\.0' Source/Examples/ --include='*.csproj' | grep -i 'DotNetWorkQueue' || echo "dnwq-0.8.0-assembly CLEAR"

# 5. No stale DNWQ 0.8.0 package folder references in csproj or packages.config
grep -rl 'DotNetWorkQueue\.0\.8\.0' Source/Examples/ --include='*.csproj' --include='packages.config' || echo "dnwq-0.8.0-folder CLEAR"

# 6. No Polly v7 stragglers left in packages.config
grep -rE '<package id="(Polly\.Caching\.Memory|Polly\.Contrib\.)' Source/Examples/ --include='packages.config' || echo "polly-stragglers CLEAR"

# 7. 0.9.18 is actually present
grep -rl 'DotNetWorkQueue\.0\.9\.18' Source/Examples/ --include='*.csproj' | wc -l  # expect > 0
grep -rE 'version="0\.9\.18"' Source/Examples/ --include='packages.config' | wc -l  # expect > 0

# 8. Transform summary
cat /tmp/phase3/transform.log

# 9. Staged changes present
git diff --cached --stat | tail -5
```
  </verify>
  <done>
All seven grep checks above print the "CLEAR" sentinel or zero matches. Counts 7 show non-zero values. The transform log shows expected counts (roughly 17 csproj, 17 packages.config, 16 App.config processed; ~119 AppMetrics Reference blocks removed across csproj; ~102 AppMetrics package entries removed across packages.config). All XML changes are staged (visible in `git diff --cached --stat`) but **no commit has been made**. `/tmp/phase3/transform.py` exists as a build artifact and is NOT inside `Source/` (so it cannot accidentally be staged).
  </done>
</task>

<task id="2" files="Source/Examples/Shared/ConsoleSharedCommands/Commands/SharedCommands.cs, Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessage.cs, Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessageAsync.cs, Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/SendMessage.cs, Source/Examples/PostGresSQL/Producer/PostGresSQLProducer/Commands/SendMessage.cs, Source/Examples/Redis/Producer/RedisProducer/Commands/SendMessage.cs, Source/Examples/SQLite/Producer/SQLiteProducer/Commands/SendMessage.cs, Source/Examples/SQLServer/Producer/SqlServerProducer/Commands/SendMessage.cs" tdd="false">
  <action>
Strip all AppMetrics code from 8 `.cs` files. Use the Read tool to load each file first and confirm exact line ranges — line numbers in RESEARCH.md are approximate. Use the Edit tool for each removal; keep removed blocks contiguous so the builder doesn't need multi-line surgery.

**File 1 — `Source/Examples/Shared/ConsoleSharedCommands/Commands/SharedCommands.cs`** (load-bearing, ~45 lines total):

Read the full file first (confirmed via pre-plan inspection — approximate line numbers below). Identify and remove **every** item listed:

  1. `using App.Metrics;` directive — ~line 33 (single line).
  2. Field `protected DotNetWorkQueue.AppMetrics.Metrics Metrics;` — ~line 39.
  3. Field `private App.Metrics.IMetricsRoot _metricsRoot;` — ~line 40.
  4. `case "EnableMetrics": return new ConsoleExecuteResult("EnableMetrics http://localhost:10001/ false");` — the switch arm inside `Example(string command)` at ~lines 53–54. Remove both lines (case label + return).
  5. Two `help.AppendLine(...)` calls inside `Help()`:
     - `help.AppendLine(ConsoleFormatting.FixedLength("EnableMetrics", "Enables queue metrics"));` — ~lines 68–69 (spans two lines because of the string-argument continuation).
     - `help.AppendLine(ConsoleFormatting.FixedLength("ViewMetrics", "Displays any captured metrics"));` — ~line 70 (single line).
  6. The entire `public ConsoleExecuteResult ViewMetrics()` method — ~lines 93–101. Body contains the `_metricsRoot.ReportRunner.RunAllAsync()` cleanup call. Remove from the `public` keyword through the closing brace, inclusive.
  7. The entire `public ConsoleExecuteResult EnableMetrics()` method — ~lines 102–124. Body contains `_metricsRoot = new App.Metrics.MetricsBuilder()...Build()` and `Metrics = new DotNetWorkQueue.AppMetrics.Metrics(_metricsRoot)`. Remove from the `public` keyword through the closing brace, inclusive.

After removal, Read the file again and grep for `Metrics|_metricsRoot|App\.Metrics|ViewMetrics|EnableMetrics` to confirm nothing remains. The `EnableStatus`, `EnableGzip`, `EnableDes` commands (adjacent in both `Example` and `Help`) must still be intact — they are unrelated and stay.

**Reflection-driven shell impact:** removing the public `EnableMetrics()` and `ViewMetrics()` methods means the shell will no longer expose those commands at runtime. That is correct — the frozen repo has no metrics demo. The per-transport `Program.cs` reflection dispatch will simply not discover them. No additional file updates needed for this.

  After each Edit tool call, re-Read the relevant region to confirm the file still compiles syntactically (balanced braces, no dangling `using` that references a now-removed method).

**Files 2–3 — ConsumeMessage.cs and ConsumeMessageAsync.cs** (in `Source/Examples/Shared/ConsoleSharedCommands/Commands/`):
  Remove ~4-line blocks of the form:
  ```csharp
  if (Metrics != null)
  {
      container.Register<IMetrics>(() => Metrics, LifeStyles.Singleton);
  }
  ```
  `ConsumeMessage.cs` has ONE such block (~line 272). `ConsumeMessageAsync.cs` has TWO such blocks (~lines 307 and 315). Also remove any `using App.Metrics;` directive at the top of each file. Use `grep -n 'Metrics' <file>` first to confirm exact line numbers.

**Files 4–8 — the five `SendMessage.cs` files** (LiteDb, PostGresSQL, Redis, SQLite, SQLServer producers):
  Each has the same shape: one `if (Metrics != null) { container.Register<IMetrics>(() => Metrics, LifeStyles.Singleton); }` block plus a `using App.Metrics;` directive at the top. Grep each file first:
  ```bash
  for f in \
    Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/SendMessage.cs \
    Source/Examples/PostGresSQL/Producer/PostGresSQLProducer/Commands/SendMessage.cs \
    Source/Examples/Redis/Producer/RedisProducer/Commands/SendMessage.cs \
    Source/Examples/SQLite/Producer/SQLiteProducer/Commands/SendMessage.cs \
    Source/Examples/SQLServer/Producer/SqlServerProducer/Commands/SendMessage.cs; do
    echo "=== $f ==="
    grep -n -E 'Metrics|App\.Metrics' "$f"
  done
  ```
  Then Edit each file: remove the `using App.Metrics;` directive and the `if (Metrics != null) { ... }` block. Each file should need 1–2 Edit calls.

**After all 8 files are edited**, run the global grep check below, then stage:
```bash
git add Source/Examples/Shared/ConsoleSharedCommands/Commands/SharedCommands.cs \
        Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessage.cs \
        Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessageAsync.cs \
        Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/SendMessage.cs \
        Source/Examples/PostGresSQL/Producer/PostGresSQLProducer/Commands/SendMessage.cs \
        Source/Examples/Redis/Producer/RedisProducer/Commands/SendMessage.cs \
        Source/Examples/SQLite/Producer/SQLiteProducer/Commands/SendMessage.cs \
        Source/Examples/SQLServer/Producer/SqlServerProducer/Commands/SendMessage.cs
```
No commit yet.
  </action>
  <verify>
```bash
cd /mnt/f/git/dotnetworkqueue.examples

# No AppMetrics symbols anywhere in example .cs files
grep -rE 'App\.Metrics|IMetricsRoot|DotNetWorkQueue\.AppMetrics' Source/Examples/ --include='*.cs' || echo "cs-appmetrics CLEAR"

# Confirm no unqualified `IMetrics` references remain in the 8 touched files
grep -n -E '\bIMetrics\b' \
  Source/Examples/Shared/ConsoleSharedCommands/Commands/SharedCommands.cs \
  Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessage.cs \
  Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessageAsync.cs \
  Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/SendMessage.cs \
  Source/Examples/PostGresSQL/Producer/PostGresSQLProducer/Commands/SendMessage.cs \
  Source/Examples/Redis/Producer/RedisProducer/Commands/SendMessage.cs \
  Source/Examples/SQLite/Producer/SQLiteProducer/Commands/SendMessage.cs \
  Source/Examples/SQLServer/Producer/SqlServerProducer/Commands/SendMessage.cs \
  || echo "IMetrics-refs CLEAR"

# SharedCommands.cs specifically must not contain the removed members
grep -n -E 'Metrics Metrics|_metricsRoot|MetricsBuilder|ReportRunner' \
  Source/Examples/Shared/ConsoleSharedCommands/Commands/SharedCommands.cs \
  || echo "SharedCommands-appmetrics CLEAR"

# All 8 files staged
git diff --cached --name-only | grep -E '\.cs$' | wc -l  # expect 8
```
  </verify>
  <done>
All three grep sentinels print "CLEAR". The `git diff --cached --name-only | grep .cs$ | wc -l` returns 8 (the exact 8 files listed). SharedCommands.cs no longer declares `Metrics`, `_metricsRoot`, the enable-metrics method, or the report-runner cleanup block. All 8 `.cs` files are staged but **not yet committed** (Task 3 handles the atomic commit).
  </done>
</task>

<task id="3" files="Source/Examples/packages/ (purged+regenerated), /tmp/phase3/*.log, git commit" tdd="false">
  <action>
Purge the NuGet package cache, restore + build, enforce baseline discipline, and commit atomically. All staged changes from Tasks 1 and 2 land in ONE commit.

**Step 1 — snapshot + purge:**

Note: `Source/Examples/packages/` is already gitignored via `.gitignore` line 140 (`**/packages/*`), confirmed pre-plan. The `rm -rf` below is safe — nothing is tracked under that path, so the purge cannot discard history.

```bash
cd /mnt/f/git/dotnetworkqueue.examples
mkdir -p /tmp/phase3

git diff --cached --stat  > /tmp/phase3/staged-stat.txt
git diff --cached --name-only > /tmp/phase3/staged-files.txt

# Purge the NuGet package cache so restore regenerates against 0.9.18
rm -rf Source/Examples/packages/
echo "packages/ cache purged"
```

**Step 2 — Polly v7 API usage fast-check:**
```bash
POLLY_CS=$(grep -rE '^using Polly(\.Caching\.Memory|\.Contrib)' Source/Examples/ --include='*.cs' | head -5)
if [ -n "$POLLY_CS" ]; then
  echo "WARNING: example code directly imports Polly v7 packages:"
  echo "$POLLY_CS"
  echo "Polly v7 strip may require absorption or reversal."
fi
```
(If the check fires, Task 1's Polly strip may have broken the build — the build step below will confirm. If it breaks, follow the ABSORPTION DECISION GATE below.)

**Step 3 — restore:**
```bash
MSBUILD='/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe'
SLN_WIN="$(wslpath -w Source/Examples/Examples.sln)"

"$MSBUILD" "$SLN_WIN" /t:Restore /p:Configuration=Release /v:minimal > /tmp/phase3/restore.log 2>&1
RESTORE_EXIT=$?
echo "restore exit: $RESTORE_EXIT"

if [ "$RESTORE_EXIT" -ne 0 ]; then
  echo "RESTORE FAILED — hard rollback"
  tail -80 /tmp/phase3/restore.log
  git reset HEAD Source/Examples/
  git checkout -- Source/Examples/
  # summary.status=failed, exit the plan
  exit 1
fi
```

**Step 4 — build:**
```bash
"$MSBUILD" "$SLN_WIN" /t:Build /p:Configuration=Release /v:normal > /tmp/phase3/build.log 2>&1
BUILD_EXIT=$?
echo "build exit: $BUILD_EXIT"
```

**Step 5 — ABSORPTION DECISION GATE** (runs only if `BUILD_EXIT != 0`):

Extract the error summary:
```bash
grep -E 'error [A-Z]+[0-9]+:' /tmp/phase3/build.log | sort -u > /tmp/phase3/build-errors.txt
ERR_CATEGORIES=$(wc -l < /tmp/phase3/build-errors.txt)
ERR_FILES=$(grep -E 'error [A-Z]+[0-9]+:' /tmp/phase3/build.log \
            | awk -F'[(:]' '{print $1}' | sort -u | wc -l)
echo "BUILD FAILED — $ERR_CATEGORIES unique error codes across $ERR_FILES files"
head -30 /tmp/phase3/build-errors.txt
```

**Per CONTEXT-3.md, the absorption budget is:**
- **≤10 example files** touched
- **≤30 minutes** of mechanical edits
- **Mechanical fixes only**

**Mechanical examples (ABSORB):**
  - Method or property renamed (e.g. `CreateAsync` → `CreateQueueAsync`) — search/replace at call site.
  - Parameter order swapped — reorder arguments at call site.
  - Namespace moved (e.g. `DotNetWorkQueue.X.Y` → `DotNetWorkQueue.Z.X.Y`) — update `using` directive.
  - Property renamed (e.g. `.Count` → `.Length`) — rename at call site.
  - Return type changed but caller ignores the return value — no action or cast.
  - Type alias needed because a class was moved — add a `using Alias = New.Type;`.
  - Enum value renamed — rename at call site.

**Structural examples (HARD ROLLBACK — do NOT absorb):**
  - A required constructor parameter was added (caller doesn't have the new value).
  - An interface contract changed (method signature flipped async/sync, added required members).
  - A whole subsystem was removed or replaced (e.g. "the queue creation API is now fluent instead of imperative").
  - The example needs a new dependency injection registration to compile.
  - Any error that needs design judgment rather than rote substitution.
  - Any error whose fix would cascade into more than 10 files.
  - `ERR_FILES > 10` — automatic rollback regardless of category.

**Hard rollback procedure:**
```bash
git reset HEAD Source/Examples/
git checkout -- Source/Examples/
# packages/ stays purged — next phase will repopulate
# Document in SUMMARY-3.1.md: status=failed, reason=<category>, top-3-errors=<list>
exit 1
```

**Absorption procedure** (shell must enforce the iteration cap — do not rely on memory):

```bash
# Before any absorption: initialize the counter and timer stamp
ABSORPTION_ITER=${ABSORPTION_ITER:-0}
ABSORPTION_START=${ABSORPTION_START:-$(date +%s)}

# Each time the build fails and we decide to absorb, run this guard FIRST:
ABSORPTION_ITER=$((ABSORPTION_ITER + 1))
NOW=$(date +%s)
ELAPSED=$((NOW - ABSORPTION_START))

if [ "$ABSORPTION_ITER" -gt 3 ]; then
  echo "ABSORPTION CAP EXCEEDED (iteration $ABSORPTION_ITER > 3) — hard rollback"
  git reset HEAD Source/Examples/
  git checkout -- Source/Examples/
  exit 1
fi

if [ "$ELAPSED" -gt 1800 ]; then
  echo "ABSORPTION TIMEOUT ($ELAPSED seconds > 1800 = 30 min) — hard rollback"
  git reset HEAD Source/Examples/
  git checkout -- Source/Examples/
  exit 1
fi

if [ "$ERR_FILES" -gt 10 ]; then
  echo "ABSORPTION FILE BUDGET EXCEEDED ($ERR_FILES > 10) — hard rollback"
  git reset HEAD Source/Examples/
  git checkout -- Source/Examples/
  exit 1
fi

echo "ABSORPTION iteration $ABSORPTION_ITER / 3  (elapsed ${ELAPSED}s, files $ERR_FILES)"
```

After the guard passes, perform the fixes:

  1. Examine `/tmp/phase3/build-errors.txt` — confirm every error is mechanical per the `Mechanical examples (ABSORB)` list above.
  2. For each error, use Read + Edit on the failing example file. Do NOT modify anything outside `Source/Examples/`.
  3. `git add` the newly-fixed files.
  4. Loop back to **Step 4** (rebuild). The next iteration re-runs the guard block above; if any of the three caps (iteration, time, file-count) trips, it hard-rollbacks automatically — no judgment needed.
  5. Record every fix in `SUMMARY-3.1.md` under "Decisions Made" with: file, error code, one-line description of the fix.

**Post-rollback consequence (document in SUMMARY-3.1.md if rollback fires):** after the hard rollback, `Source/Examples/packages/` remains empty because Step 1 purged it and the rolled-back csproj state no longer matches any cached package layout. A fresh `msbuild /t:Restore` run against the pre-Phase-3 `main` HEAD will repopulate the cache. The builder does not need to restore manually; it only needs to note the absence in the summary so the reviewer isn't surprised.

**Step 6 — baseline discipline (runs after a successful build):**
```bash
WARN_LINE=$(grep -E '^\s*[0-9]+ Warning\(s\)' /tmp/phase3/build.log | tail -1)
ERR_LINE=$(grep  -E '^\s*[0-9]+ Error\(s\)'   /tmp/phase3/build.log | tail -1)
WARN_COUNT=$(echo "$WARN_LINE" | grep -oE '[0-9]+' | head -1)
ERR_COUNT=$(echo  "$ERR_LINE"  | grep -oE '[0-9]+' | head -1)
echo "warnings=$WARN_COUNT errors=$ERR_COUNT"

if [ "${ERR_COUNT:-1}" -ne 0 ] || [ "${WARN_COUNT:-1}" -gt 0 ]; then
  echo "BASELINE VIOLATION — rolling back"
  git reset HEAD Source/Examples/
  git checkout -- Source/Examples/
  exit 1
fi
```

**Step 7 — atomic commit:**
```bash
git commit -m "shipyard(phase-3): pin DotNetWorkQueue at 0.9.18 and strip AppMetrics"
git log -1 --stat | head -40
git status --short  # expect: only CLAUDE.md untracked
```
  </action>
  <verify>
```bash
cd /mnt/f/git/dotnetworkqueue.examples

# 1. packages/ regenerated with 0.9.18 and no 0.8.0
ls Source/Examples/packages/ | grep -E '^DotNetWorkQueue\.0\.9\.18$' && echo "dnwq 0.9.18 present"
ls Source/Examples/packages/ 2>/dev/null | grep -E '^DotNetWorkQueue\.0\.8\.0$' || echo "dnwq 0.8.0 gone CLEAR"

# 2. No AppMetrics package folders
ls Source/Examples/packages/ 2>/dev/null | grep -E '^App\.Metrics' || echo "appmetrics packages gone CLEAR"

# 3. Build exit + baseline
tail -10 /tmp/phase3/build.log
grep -E '[0-9]+ (Warning|Error)\(s\)' /tmp/phase3/build.log | tail -2

# 4. Exactly one new commit with the expected subject
git log -1 --pretty=format:'%H %s'
git log -1 --pretty=format:'%s' | grep -Fx 'shipyard(phase-3): pin DotNetWorkQueue at 0.9.18 and strip AppMetrics' && echo "commit subject CLEAR"

# 5. Working tree clean (only CLAUDE.md untracked)
git status --short
```
  </verify>
  <done>
`Source/Examples/packages/DotNetWorkQueue.0.9.18/` exists; no `DotNetWorkQueue.0.8.0/` or `App.Metrics.*/` folders remain. `/tmp/phase3/build.log` shows `0 Error(s)` and `Warning(s)` less than or equal to the Phase 1 baseline (zero). `git log -1` shows exactly one new commit with subject `shipyard(phase-3): pin DotNetWorkQueue at 0.9.18 and strip AppMetrics`. `git status --short` is empty except for `CLAUDE.md`. If absorption was needed, every fix is documented in `SUMMARY-3.1.md` under "Decisions Made" with file + error code + one-line rationale.
  </done>
</task>

## Final verification block

```bash
cd /mnt/f/git/dotnetworkqueue.examples

# No stale 0.8.0 references anywhere
grep -rl 'DotNetWorkQueue\.0\.8\.0' Source/Examples/ --include='*.csproj' --include='packages.config' \
  || echo "DNWQ 0.8.0 CLEAR"

# No AppMetrics references anywhere
grep -rl 'App\.Metrics' Source/Examples/ \
  --include='*.cs' --include='*.csproj' --include='packages.config' \
  --include='App.config' --include='app.config' \
  || echo "APPMETRICS CLEAR"

# 0.9.18 present in csproj
grep -rl 'DotNetWorkQueue\.0\.9\.18' Source/Examples/ --include='*.csproj' | wc -l  # expect > 0

# Build clean (re-run to prove determinism)
MSBUILD='/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe'
SLN_WIN="$(wslpath -w Source/Examples/Examples.sln)"
"$MSBUILD" "$SLN_WIN" /t:Build /p:Configuration=Release /v:normal 2>&1 \
  | grep -E '[0-9]+ (Warning|Error)\(s\)' | tail -2
```

Expected: both sentinels print, count > 0 for 0.9.18 match, final build shows `0 Warning(s)` and `0 Error(s)`.

## Rollback

If anything after Task 1 stages but before the Task 3 commit goes sideways:

```bash
cd /mnt/f/git/dotnetworkqueue.examples
git reset HEAD Source/Examples/
git checkout -- Source/Examples/
rm -rf Source/Examples/packages/
# packages/ will be repopulated at the start of the next attempt
```

If the atomic commit has already landed and a downstream problem is discovered:

```bash
git revert HEAD
# or, if nothing has been pushed and we want a clean history:
git reset --hard HEAD~1
rm -rf Source/Examples/packages/
```

`/tmp/phase3/transform.py`, `/tmp/phase3/*.log`, and `/tmp/phase3/build-errors.txt` are retained as forensic artifacts regardless of outcome.
