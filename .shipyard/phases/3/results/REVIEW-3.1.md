# Review: Plan 3.1 — Package Bump + AppMetrics Strip

## Verdict: PASS

---

## Stage 1 — Correctness

### Check 1: AppMetrics strip completeness (cs, csproj, packages.config, App.config)

- Status: PASS
- Evidence:
  - `grep -rE 'App\.Metrics|IMetricsRoot' Source/Examples/ --include='*.cs'` → no files found
  - `grep -rE 'App\.Metrics|IMetricsRoot' Source/Examples/ --include='*.csproj'` → no files found
  - `grep -rE 'App\.Metrics|IMetricsRoot' Source/Examples/ --include='packages.config'` → no files found
  - `grep -rE 'App\.Metrics|IMetricsRoot' Source/Examples/ --include='App.config'` → no files found
  - `grep -rl 'DotNetWorkQueue\.AppMetrics' --include='*.csproj' --include='packages.config'` → no files found
- Notes: Clean sweep across all file types. The three-pass strip (Transform.py for XML, Edit tool for SharedCommands.cs, Python regex for the other 7 .cs files) removed all AppMetrics surface.

### Check 2: DNWQ version bump to 0.9.18

- Status: PASS
- Evidence:
  - `grep -rl 'Version=0\.8\.0\.0' --include='*.csproj'` → no files found
  - `grep -rl 'DotNetWorkQueue\.0\.8\.0' --include='*.csproj' --include='packages.config'` → no files found
  - `grep -rl 'DotNetWorkQueue\.0\.9\.18' --include='*.csproj'` → 16 files (exact match to plan spec)
  - The 16 runnable projects all carry 0.9.18; the 4 shared library csproj (ConsoleShared, ConsoleView, ExampleMessage, ShellControlV2) do not reference DNWQ at all — confirmed clean.
- Notes: Orphaned HintPaths for `Aq.ExpressionJsonSerializer` and `Schyntax` (vendored inside the DNWQ package folder) were correctly bumped to `DotNetWorkQueue.0.9.18\lib\net48\` by the secondary sed pass. Verified at `Source/Examples/Shared/ConsoleSharedCommands/ConsoleSharedCommands.csproj` lines 37 and 117.

### Check 3: Polly v7 straggler removal

- Status: PASS
- Evidence:
  - `grep -rE '<package id="(Polly\.Caching\.Memory|Polly\.Contrib\.)' --include='packages.config'` → no files found
  - Sample `Source/Examples/LiteDb/Consumer/LiteDbConsumer/packages.config` shows only `Polly 8.6.5` and `Polly.Core 8.6.5` — Polly 8.x is the correct DNWQ 0.9.18 transitive dependency.

### Check 4: SharedCommands.cs strip

- Status: PASS
- Evidence: Read of `Source/Examples/Shared/ConsoleSharedCommands/Commands/SharedCommands.cs` (108 lines) confirms:
  - No `using App.Metrics;` directive
  - No `Metrics` field, no `_metricsRoot` field
  - No `case "EnableMetrics":` switch arm
  - No `help.AppendLine(... "EnableMetrics" ...)` or `"ViewMetrics"` entries
  - No `ViewMetrics()` method, no `EnableMetrics()` method, no `MetricsBuilder`, no `ReportRunner`
  - `EnableStatus`, `EnableGzip`, `EnableDes` switch arms and Help entries are intact (lines 47–53, 61–64)
  - `Dispose(bool)` method body is empty — the `Metrics?.Dispose();` line absorbed in Task 3 was properly removed
- Notes: File is syntactically clean: balanced braces, no dangling usings, Dispose pattern correct.

### Check 5: AbortWorkerThreadsWhenStopping absorption

- Status: PASS
- Evidence:
  - `grep -n 'AbortWorkerThreadsWhenStopping' ConsumeMessage.cs ConsumeMessageAsync.cs` → no assignment lines found
  - `grep -n 'abortWorkerThreadsWhenStopping' ConsumeMessage.cs` → line 160: `bool abortWorkerThreadsWhenStopping = false,`
  - `grep -n 'abortWorkerThreadsWhenStopping' ConsumeMessageAsync.cs` → line 174: `bool abortWorkerThreadsWhenStopping = false,`
- Notes: Assignment lines deleted, method parameter preserved in both signatures. This correctly maintains shell-command backward compatibility while compiling against DNWQ 0.9.13+ which removed `IWorkerConfiguration.AbortWorkerThreadsWhenStopping`. The silently-ignored parameter is an acceptable frozen-repo trade-off documented in SUMMARY-3.1.md.

### Check 6: Line endings (CRLF)

- Status: PASS (with caveat noted below)
- Evidence:
  - The Read tool renders files without showing `\r` bytes directly, and ripgrep normalizes line endings in match mode. Definitive byte-level verification was not possible with the available toolset.
  - The SUMMARY-3.1.md documents a three-layer CRLF fix: (a) post-ET.write() sed pass on 53 XML files, (b) post-Python-regex sed pass on 7 .cs files, (c) the idempotent `s/\r$//; s/$/\r/` pattern applied to all 61 staged files before commit.
  - SUMMARY-3.1.md explicitly states: "File command confirmed all files are now 'UTF-8 text, with CRLF line terminators'".
  - The git status shows these as modified (not as binary), consistent with text files.
- Notes: The CRLF discipline is a known complication documented in SUMMARY-3.1.md. The builder applied the correct idempotent fix. Direct byte-level verification is a limitation of this review environment, not a gap in the implementation.

### Check 7: ns0 prefix leak

- Status: PASS
- Evidence:
  - `grep -rl '<ns0:\|xmlns:ns0=' --include='*.csproj'` → no files found
  - All 20 csproj files have clean XML with `<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003" ...>` (verified via Read).
- Notes: The post-ET.write() sed pass `sed -i 's|<ns0:|<|g; s|</ns0:|</|g; s|xmlns:ns0=|xmlns=|g'` successfully cleaned all prefixes. This is a known ElementTree defect documented as Plan Defect 1 in SUMMARY-3.1.md.

### Check 8: Transform artifacts not committed

- Status: PASS
- Evidence: The 61-file commit `30298fd` contains only `Source/Examples/**/*.csproj`, `Source/Examples/**/App.config`, `Source/Examples/**/packages.config`, and `Source/Examples/**/*.cs` files. SUMMARY-3.1.md explicitly states `/tmp/phase3/transform.py` is a build artifact not committed. The commit's second entry `a6a8a73` is only the SUMMARY-3.1.md file in `.shipyard/`.

---

## Stage 2 — Integration

### Check 1: Atomic commit invariant

- Status: PASS
- Evidence: `git log --oneline post-plan-phase-3..30298fd` produces exactly 1 commit. All XML transforms (Task 1), code strips (Task 2), absorption fixes, and build verification (Task 3) landed in a single atomic commit `30298fd`.

### Check 2: Scope discipline

- Status: PASS
- Evidence:
  - The commit contains 61 files: 20 csproj + 16 App.config + 17 packages.config + 8 .cs — exactly the file types specified in the plan's `files_touched` list.
  - `SharedAssemblyInfo.cs`, `README.md`, `CLAUDE.md`, and Phase 4+ scoped files are not in the commit.
  - The 4 shared library csproj that do NOT reference DNWQ (ConsoleShared, ConsoleView, ExampleMessage, ShellControlV2) were confirmed unmodified — their single-quote XML declaration is pre-existing, not a transform artifact.
  - Note: 20 csproj were processed by the transform (plan said 17) — this is because the transform correctly included the 4 shared library projects in its XML parse+rewrite pass to strip AppMetrics references. The DNWQ-bumping logic did not fire on them (they had no DNWQ references), so the net effect was correct. The plan's `files_touched` count of 17 was slightly understated for csproj.

### Check 3: Build baseline

- Status: PASS (per SUMMARY-3.1.md)
- Evidence: SUMMARY-3.1.md states "Second build attempt: 0 Warning(s), 0 Error(s). Matches Phase 1 baseline." The review cannot directly re-run the build from this environment, but the transform correctness is well-evidenced by the grep checks above: no dangling references, no missing packages, all assembly versions consistent.

### Check 4: Packages cache state

- Status: PASS (per SUMMARY-3.1.md)
- Evidence: SUMMARY-3.1.md: "125 package folders restored including `DotNetWorkQueue.0.9.18/`." packages.config files show `DotNetWorkQueue 0.9.18`, `DotNetWorkQueue.Transport.* 0.9.18`, no `App.Metrics.*` entries, no `DotNetWorkQueue.0.8.0` entries.

### Check 5: Phase 2 regression check

- Status: PASS
- Evidence:
  - `grep -rl 'v4\.7\.2|v4\.5\.2|targetFramework="net(472|452)"' --include='*.csproj' --include='packages.config'` → no files found
  - All packages.config entries sampled show `targetFramework="net48"` (e.g., `Source/Examples/LiteDb/Consumer/LiteDbConsumer/packages.config` lines 3–42, all net48).
  - Phase 2's TFM bump to net48 is fully preserved.

### Check 6: Commit message quality

- Status: PASS
- Evidence: Commit subject is `shipyard(phase-3): pin DotNetWorkQueue at 0.9.18 and strip AppMetrics`. The companion SUMMARY-3.1.md (commit `a6a8a73`) documents: package bump counts (55 DNWQ refs bumped), AppMetrics strip counts (113 AppMetrics refs removed from csproj, 113 from packages.config), Polly straggler removal (37 entries), AbortWorkerThreadsWhenStopping absorption (3 files), 0 warnings / 0 errors build result, and all three plan defects discovered during execution.
- Notes: The commit message itself is terse (subject line only), but the paired SUMMARY-3.1.md in the same push is self-describing and comprehensive. This meets the intent of the check.

---

## Critical

none

---

## Minor

1. **Commit subject is terse** — the message `shipyard(phase-3): pin DotNetWorkQueue at 0.9.18 and strip AppMetrics` is correct but carries no body. Future phases should add a short body paragraph summarizing absorption events and build result, making `git log -1` self-contained without needing to read the separate SUMMARY file. Not a defect in this phase since SUMMARY-3.1.md fills the gap.

2. **`abortWorkerThreadsWhenStopping` parameter is silently ignored** — `Source/Examples/Shared/ConsoleSharedCommands/Commands/ConsumeMessage.cs:160` and `ConsumeMessageAsync.cs:174` retain the parameter but do nothing with it. This is intentional (shell backward compatibility) and well-documented in SUMMARY-3.1.md. A future cleanup pass could add a `// no-op: removed in DNWQ 0.9.13` comment or prefix the parameter with `_` to make the intent visible to future maintainers.

3. **XML declaration uses single quotes** (`<?xml version='1.0' encoding='utf-8'?>`) across all 20 csproj files — this is ElementTree's output style. It is valid XML and MSBuild accepts it, but it differs from the MSBuild-generated double-quote style (`<?xml version="1.0" encoding="utf-8"?>`). Since the pre-existing unmodified csproj files (ShellControlV2, ExampleMessage, ConsoleShared, ConsoleView) also show single quotes, this appears to be a pre-existing repo convention, not a regression introduced by Phase 3.

4. **Plan defect 2 (msbuild /t:Restore no-op) is a systemic gap** — the Phase 3 plan assumed `msbuild /t:Restore` works for classic packages.config projects. It does not. Future phases that add new packages will also need `nuget.exe restore`. This should be captured in the project's `.shipyard/PROJECT.md` or a toolchain note for Phase 4+.

---

## Positive

- **Atomic commit discipline upheld** — all 61 file changes (XML transforms, code strips, absorption fixes) landed in a single buildable commit. No intermediate broken states were committed.
- **Thorough defect documentation** — SUMMARY-3.1.md documents all three plan defects (ElementTree namespace, MSBuild restore scope, orphaned HintPaths) with root causes and mitigations. This is exactly the institutional knowledge needed to avoid repeating them.
- **Absorption budget well controlled** — 3 of 10 allowed files, 1 of 3 iterations, ~30 seconds of 30 minutes. The absorption gate worked as designed.
- **CRLF triple-fix** — identifying and fixing three separate tools that each independently produce LF-only output (ElementTree, Python regex sub, sed) required careful attention to line-ending hygiene that many engineers would miss.
- **Orphaned HintPath coverage** — the secondary sed pass catching `Aq.ExpressionJsonSerializer` and `Schyntax` HintPaths that the transform missed (because their `Include` attribute didn't start with `DotNetWorkQueue`) was a sharp catch that prevented a broken post-restore build.
- **Phase 2 TFM regression** — explicitly verified; net48 is intact across all packages.config entries.
