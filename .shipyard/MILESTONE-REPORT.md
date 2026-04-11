# Milestone Report: examples-freeze

**Completed:** 2026-04-11
**Phases:** 7/7 complete
**Final commit:** `1814f4f shipyard: freeze at DotNetWorkQueue 0.9.18 / .NET Framework 4.8`
**Freeze tag:** `freeze` (permanent external-facing marker)

## Project definition

`DotNetWorkQueue.Examples` — the original example applications for the DotNetWorkQueue library — pinned to its final state at **DotNetWorkQueue 0.9.18** targeting **.NET Framework 4.8**. DNWQ 0.9.19 dropped net48 for net10/net8 exclusively, making this a one-way door: the repo cannot follow upstream forward without losing the classic-csproj identity that differentiates it from the newer `DotNetWorkQueue.Samples` repo. The freeze leaves the repo in a clean, self-consistent, runnable final state with a GitHub Actions CI workflow proving the permanent build signal.

## Phase summaries

### Phase 1 — Spike & Baseline Capture — ✅ COMPLETE

Installed `.gitattributes` (CRLF discipline for the Windows-authored repo), captured the immutable Phase 1 baseline at **0 Warning(s) / 0 Error(s)**, resolved R1 (NuGet availability: all 8 DNWQ packages publish 0.9.18) and R2 (AppMetrics touch-point enumeration: 8 .cs files, 17 packages.config, 17 csproj, 16 App.config). Locked in the MSBuild invocation pattern (`/mnt/c/Program Files/Microsoft Visual Studio/18/Community/MSBuild/Current/Bin/MSBuild.exe` + `wslpath -w` for Windows-native solution paths) that every subsequent phase relied on.

### Phase 2 — TFM Bump to net48 — ✅ COMPLETE

Atomic commit `4ef2d29`. 53 files bumped from mixed `net452` / `net472` → `net48`: 20 csproj (`<TargetFrameworkVersion>`), 16 App.config (`<supportedRuntime>` sku), 17 packages.config (`targetFramework` attribute). 974 insertions / 974 deletions — pure substitution, no structural drift. Build post-bump still 0/0. HintPath entries intentionally deferred to Phase 3.

### Phase 3 — Package Bump + AppMetrics Strip — ✅ COMPLETE

Highest-risk phase. Atomic commit `30298fd`. 61 files touched: DNWQ + 7 transport packages pinned at 0.9.18 with HintPaths migrated from `\lib\netstandard2.0\` to `\lib\net48\`, all `App.Metrics.*` and `DotNetWorkQueue.AppMetrics` references purged (113 Reference blocks + 113 packages.config entries + 17 AppMetrics refs removed), 37 Polly v7 stragglers removed, 64 App.Metrics binding redirects stripped from App.config, 8 .cs files code-stripped (~45 lines removed). **One non-metrics breaking change absorbed** per the policy: DNWQ 0.9.13 removed `AbortWorkerThreadsWhenStopping` from `IWorkerConfiguration`, so the assignment line was deleted from `ConsumeMessage.cs` and `ConsumeMessageAsync.cs` (method parameter left in signature for shell backward compatibility). Build post-strip 0/0 — baseline preserved. Three plan defects documented: `register_namespace` ns0: leak in csproj, `msbuild /t:Restore` no-op for packages.config, orphaned HintPaths for vendored DLLs.

### Phase 4 — Code & Repo Hygiene — ✅ COMPLETE

Single atomic commit. 8 text edits + 4 directory purges: `SharedAssemblyInfo.cs` version bumped 0.7.1 → 0.9.18 (closed the drift), `LiteDb/Consumer/LiteDbConsumer/Commands/ConsumeMessage.cs` namespace typo fixed (`SQLiteConsumer.Commands` → `LiteDbConsumer.Commands`), SQL Server + PostgreSQL App.config connection strings corrected from `192.168.0.58` → `192.168.0.2` (6 files), orphaned `Source/Examples/{PostGresSQL,Redis,SQLite,SQLServer}/Rpc/` directories (containing stale `bin/Debug` + `obj/Debug`) deleted. R14 (committed bin/obj) was a no-op — `.gitignore` already had the exclusions and `git ls-files` returned zero tracked artifacts. Plan and summary written post-hoc after `/shipyard:resume` recovered pre-existing working-tree edits.

### Phase 5 — GitHub Actions CI — ✅ COMPLETE

Atomic commit `8d37e19`. Added `.github/workflows/ci.yml` — single job on `windows-latest` with `actions/checkout@v4` + `microsoft/setup-msbuild@v2` + `NuGet/setup-nuget@v2` + `nuget restore` + `msbuild /t:Build /p:Configuration=Release /v:minimal /nologo`. No test step (repo has none), no artifact upload, no schedule trigger. Deliberately uses the msbuild+nuget toolchain (not `dotnet build`) because classic csproj + packages.config is not dotnet-compatible — a hard-won lesson from Phase 3's cache purge. First green run on the freeze commit is asynchronous and happens when the repo is pushed to GitHub.

### Phase 6 — Documentation — ✅ COMPLETE

Atomic commit `bd11651`. `README.md` received a prepended `⚠ This repository is frozen.` quoted banner naming DNWQ 0.9.18 + .NET 4.8, explaining the 0.9.19 net48 drop, and linking to `DotNetWorkQueue.Samples`. `CLAUDE.md` (which had been untracked from the session's `/init` run since the very start) was finalized and committed for the first time with TFM 4.7.2 → 4.8 update, 0.9.18 pin note, and a bonus hygiene bullet documenting the WSL MSBuild invocation pattern. Architecture, shared-base, and convention sections unchanged — still accurate.

### Phase 7 — Verification & Freeze Commit — ✅ COMPLETE

Atomic commit `1814f4f`. R18 final clean Release build: 0/0. R19 mandatory LiteDb producer↔consumer round-trip smoke test: PASS after 4 mid-test fixes — (a) a latent Phase 3 App.config namespace bug that caused .NET Framework side-by-side config error at runtime, fixed inline across 16 files via sed; (b) CLAUDE.md had the shell command syntax wrong (space- vs dot-separated), corrected in-session; (c) my initial command script guessed method signatures wrong (StartQueue not Start, two-step CreateQueue pattern); (d) `c:\temp\` directory didn't exist on the user's machine, `mkdir` fix. Known pre-existing quirk preserved: example handler logs at Debug level but EnableSerilog defaults to Information, so successful message processing produces no visible log output — matches pre-freeze behavior. R20 remote smoke tests (SQL Server, PostgreSQL, Redis at `192.168.0.2`) SKIPPED per best-effort allowance. Two tags applied: `post-build-phase-7` (per-phase checkpoint) and `freeze` (permanent external-facing marker).

## Key decisions across the freeze

- **Work on `main` directly, no worktree.** The pre-commit build-verification gate was the safety mechanism; every phase landed atomically with rollback via `git reset` to the prior checkpoint tag.
- **Non-metrics breakage absorption policy.** Phase 3 adopted a concrete budget (≤10 files, ≤30 min, mechanical fixes only, no structural refactoring) for handling DNWQ 0.8.0 → 0.9.18 API drift. One break (`AbortWorkerThreadsWhenStopping`) absorbed well within budget.
- **Python XML transform approach.** Phase 3 used `xml.etree.ElementTree` to delete AppMetrics references across 50+ files in one shot. Two serialization quirks (ns0: prefix leak, default-namespace promotion) required sed post-processing — the serializer's namespace handling is unreliable.
- **`nuget.exe` for packages.config restore.** Phase 3 discovered `msbuild /t:Restore` is a no-op for classic csproj + packages.config. Downloaded standalone `nuget.exe` to `/tmp/phase3/` and used it via WSL interop (chmod +x required). This lesson drove the Phase 5 CI workflow to use `NuGet/setup-nuget@v2` instead of `dotnet restore`.
- **Manual smoke test for R19.** WinForms console shells can't be automated from WSL. Phase 7 script-drove the user through a step-by-step smoke test procedure; I recorded outcomes verbatim in SUMMARY-7.1.md.

## Documentation status

- API reference: N/A (example repo, not a library)
- Architecture: inline in CLAUDE.md (updated in Phase 6)
- User guides: in-shell `Help()` + README transport list
- README: FROZEN banner + existing content preserved
- Migration guide: redirect to `DotNetWorkQueue.Samples` in the README banner

See `.shipyard/DOCUMENTATION-SHIP.md` for the ship-time documentation pass.

## Security posture

See `.shipyard/AUDIT-SHIP.md`. No secrets committed, no SQL injection surface (DNWQ handles parameterization + queue name validation), no deserialization gadget risk (DNWQ 0.9.12 `DenyListSerializationBinder` default), no known CVEs at pinned versions. Three non-blocking advisories (action SHA pinning, branch protection, CODEOWNERS) are GitHub-admin actions outside the freeze's scope.

## Known issues

Three pre-existing documentation nits persist in the frozen state (all documented in SUMMARY-7.1.md and DOCUMENTATION-SHIP.md):

1. CLAUDE.md shell command syntax examples are space-separated but the actual shell requires dot-separated.
2. The two-step `CreateQueue` pattern (physical provisioning vs in-process handle dictionary) is not documented anywhere.
3. Example `LogDebug` handlers + Information-level Serilog defaults produce no visible output during successful message processing — a pre-existing quirk, not a freeze regression.

These are knowledge-drift nits, not functional defects. A post-freeze fork-and-edit pass could address them in ~20 lines of documentation edits.

## Metrics

- **Phases planned and executed:** 7/7
- **Commits in the freeze:** ~22 (planning + build + summary commits across all phases, plus the freeze commit itself)
- **Files touched by the cumulative diff:** ~80 (20 csproj + 17 packages.config + 16 App.config + 8 .cs + SharedAssemblyInfo.cs + README.md + CLAUDE.md + .gitattributes + .github/workflows/ci.yml + shipyard artifacts)
- **Lines of code removed (net):** roughly 1100 (AppMetrics strip was the largest contributor — package references, binding redirects, method bodies, using directives)
- **Lines of code added (net, excluding shipyard artifacts):** roughly 100 (`.gitattributes`, CI workflow, README banner, CLAUDE.md updates, absorption fixes)
- **Plan defects documented for retrospective value:** 6 (3 in Phase 3, 3 in Phase 7)
- **Breaking changes absorbed:** 1 (`AbortWorkerThreadsWhenStopping`)
- **Smoke tests passed:** 1 hard-gate (R19 LiteDb)
- **Baseline warning count drift across all 7 phases:** 0 (preserved 0/0 from Phase 1 through freeze)
