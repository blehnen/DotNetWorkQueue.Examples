# Phase 5 Research — GitHub Actions CI

Minimal research — phase scope is a single workflow file.

## Reference pattern: DotNetWorkQueue.Samples `.github/workflows/ci.yml`

Read during brainstorming at session start. Shape:
```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
jobs:
  build:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      - name: Restore SampleShared
        run: dotnet restore "Source\Samples\SampleShared\SampleShared.sln"
      - name: Build SampleShared
        run: dotnet build "Source\Samples\SampleShared\SampleShared.sln" -c Debug --no-restore
      # ... one restore + build pair per solution (6 total) ...
```

## What DOES transfer from the Samples pattern

- `windows-latest` runner
- Trigger shape (`push`/`pull_request` to `main`)
- Workflow name `CI`
- `actions/checkout@v4`
- Separating `restore` from `build` for clarity

## What does NOT transfer

- **`actions/setup-dotnet@v4`** — the Samples repo uses SDK-style csproj with `PackageReference`, so `dotnet build` works. This repo uses classic csproj + `packages.config`, which `dotnet build` does not reliably handle. Confirmed by Phase 3: `msbuild /t:Restore` is a no-op for packages.config; `dotnet restore` has the same limitation.
- **`dotnet build`/`dotnet restore` commands** — replaced with `msbuild` and `nuget restore`.
- **Per-solution restore/build pairs** — this repo has a single `Source/Examples/Examples.sln`, not 6 separate solutions.
- **Debug configuration** — this repo is pinned to Release for the freeze.
- **Integration test runs** — this repo has no test projects.

## Required GitHub Actions used

- `actions/checkout@v4` — standard.
- `microsoft/setup-msbuild@v2` — puts `MSBuild.exe` on `PATH`. The `windows-latest` runner already has Visual Studio's MSBuild installed, but this action ensures the version is pinned and PATH is configured.
- `NuGet/setup-nuget@v2` — puts `nuget.exe` on `PATH`. The runner does include nuget.exe but this action locks down the version.

Both actions are official/verified publishers on the GitHub marketplace.

## Known runner environment (from GitHub docs, not verified here)

`windows-latest` (2025) ships:
- Visual Studio 2022 Enterprise
- MSBuild ≥ 17.x
- .NET Framework targeting packs 4.6.2, 4.7.2, 4.8 — **net48 is available**

The workflow does not need a targeting-pack install step.

## No repo files to read or modify beyond the new workflow

- Does not touch source code.
- Does not touch `Source/Examples/Examples.sln` (exists, unchanged).
- Does not touch `Source/Examples/packages/` cache.
- Does not touch `.gitignore` or `.gitattributes`.
