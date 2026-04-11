# Phase 5 Context — GitHub Actions CI

Phase 5 adds `.github/workflows/ci.yml`. Single file, single job, single purpose: prove `Source/Examples/Examples.sln` builds clean in Release on a fresh `windows-latest` runner. No tests (repo has none).

## Decisions

1. **Triggers:** `push` to `main` and `pull_request` to `main`. No `schedule:` trigger — a frozen repo doesn't need weekly cron runs eating GitHub-hosted minutes forever. User can add one later if they want drift detection.
2. **Runner:** `windows-latest`. Required because this is classic csproj + net48, which does not build on Linux or macOS.
3. **Toolchain:** `microsoft/setup-msbuild@v2` (to register `MSBuild.exe` on PATH) + `NuGet/setup-nuget@v2` (to register `nuget.exe`). **NOT** `actions/setup-dotnet` — the Samples repo uses that, but it wires up `dotnet build` which does not work for classic csproj + packages.config (per the Phase 3 lesson).
4. **Steps:** `checkout → setup-msbuild → setup-nuget → nuget restore → msbuild Release`. No test step. No artifact upload (no value for a frozen repo).
5. **Solution file:** `Source/Examples/Examples.sln` (single solution; the Samples repo has multiple solutions split per transport, so its `ci.yml` has one restore/build pair per solution — this repo collapses to a single pair).
6. **No matrix strategy.** One configuration: `Release`. Debug is not exercised by CI.
7. **File path:** `.github/workflows/ci.yml`. The `.github` directory does not currently exist in this repo — will be created during the build step.

## No user questions captured

Phase 5 had no gray areas worth asking about. The CI shape is fully determined by the toolchain constraints (classic csproj + net48 → must use msbuild + nuget.exe on windows-latest).
