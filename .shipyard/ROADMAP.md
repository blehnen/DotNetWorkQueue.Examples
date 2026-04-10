# ROADMAP — examples-freeze

Freeze `DotNetWorkQueue.Examples` at DotNetWorkQueue **0.9.18** / **.NET Framework 4.8**. One-way door. Single focused session, sized in hours.

Each phase is a single atomic commit (or short commit chain) with an explicit revert point. Phases are strictly sequential — the freeze does not parallelize.

---

## Phase 1 — Spike & Baseline Capture — ✅ COMPLETE (2026-04-10)

- **Depends on:** none
- **Delivers:** R1, R2, plus baseline-warning capture (NFR "Baseline warning discipline")
- **Effort:** XS (<1h)
- **Risk:** LOW
- **Actual outcome:** GO for Phase 2. Baseline = 0 warnings, 0 errors. 8 DNWQ packages (not 6) confirmed at 0.9.18. See `.shipyard/phases/1/SPIKE-NOTES.md`.

**Work:**
1. Query NuGet for `DotNetWorkQueue`, `DotNetWorkQueue.Transport.SqlServer`, `.PostgreSQL`, `.SQLite`, `.LiteDb`, `.Redis` at `0.9.18`. Confirm all six publish a matching version. If any transport lags, **STOP** and raise scope question (R1).
2. Grep `Source/Examples/` for `App.Metrics`, `DotNetWorkQueue.AppMetrics`, `using App.Metrics`, `IMetrics`, `IMetricsRoot`. Produce a flat file list to be edited in Phase 3 (R2).
3. Run a clean Release build of `Source/Examples/Examples.sln` against current `main` (net472, DNWQ 0.8.0 working tree). Record the **warning count** in the spike notes. This is the immutable ceiling for R18.

**Success criterion:** A spike-notes file exists listing (a) confirmed 0.9.18 availability for all 6 packages, (b) every file that touches AppMetrics, (c) baseline Release warning count as an integer.

---

## Phase 2 — TFM Bump to net48 (Atomic) — ✅ COMPLETE (2026-04-10)

- **Depends on:** Phase 1
- **Delivers:** R3, R4, R5 (R5 deferred to Phase 3 by design — see CONTEXT-2.md)
- **Effort:** S (1–3h)
- **Risk:** LOW — net472 → net48 is a tiny BCL delta; net452 → net48 on 4 shared libs is the only project-graph change with any meaningful surface, but those projects only consume `ConsoleView`/WinForms BCL types that exist in both targets.
- **Actual outcome:** Atomic commit `4ef2d29`, 53 files changed (20 csproj + 16 App.config + 17 packages.config), 0 warnings / 0 errors post-build — baseline discipline preserved. HintPaths intentionally deferred to Phase 3. See `.shipyard/phases/2/results/SUMMARY-2.1.md`.

**Work:**
1. Update `<TargetFrameworkVersion>` to `v4.8` in all 20 csproj files under `Source/Examples/` (15 runnable + 5 shared, including `ConsoleSharedCommands` from net472 and `ConsoleShared`/`ConsoleView`/`ExampleMessage`/`ShellControlV2` from net452) — R3.
2. Update `<supportedRuntime>` in every `App.config` and `targetFramework="net48"` in every `packages.config` — R4.
3. Audit `<Reference>`/`<HintPath>` entries; remove any that hard-code a non-net48 lib subfolder. NuGet should auto-resolve net48 (or net472 fallback) on next restore — R5.
4. Commit. Run `nuget restore` + `msbuild ... /p:Configuration=Release`.

**Success criterion:** `Source/Examples/Examples.sln` builds clean in Release with zero new warnings vs Phase 1 baseline. Every `.csproj` under `Source/Examples/` declares `v4.8`. Maps to Success Criteria #1.

**What could go wrong (despite LOW rating):** A NuGet package in the dependency graph may not ship a `net48` lib folder, forcing a `net472` fallback that triggers a binding-redirect warning. Mitigation: keep an eye on warning delta; if a single package is the offender, accept the fallback (net48 is binary-compatible with net472) and note it.

---

## Phase 3 — Package Bump + AppMetrics Strip (Atomic) — ✅ COMPLETE (2026-04-10)

- **Depends on:** Phase 2
- **Delivers:** R6, R7, R8, R9, R11
- **Effort:** M (3–6h) — actual: compressed into main-context execution after 3 consecutive builder agent timeouts
- **Risk:** **MEDIUM** — this is the highest-risk phase in the freeze.
- **Actual outcome:** Atomic commit `30298fd`, 61 files changed (20 csproj + 16 App.config + 17 packages.config + 8 .cs). 0 warnings / 0 errors post-build. One non-metrics breaking change absorbed (`AbortWorkerThreadsWhenStopping` removed in DNWQ 0.9.13) + one missed-removal fix (`Metrics?.Dispose()`), all within the absorption budget (3 of 10 files, 1 of 3 iterations). Three plan defects documented in SUMMARY-3.1.md for retrospective (ElementTree namespace bug, `msbuild /t:Restore` no-op for packages.config, orphaned HintPaths for vendored DLLs).

**Why MEDIUM:** Bumping `DotNetWorkQueue` 0.8.0 → 0.9.18 crosses a major version of internal API surface (ten minor releases). The PROJECT.md non-goal R11 explicitly says "treat any additional break as a scope-check trigger" — this phase is where such breaks will surface. Removing AppMetrics is also coupled to the package bump: you cannot remove the AppMetrics package without simultaneously removing the `using App.Metrics` lines and DI registration calls in example code, or every consumer/producer project fails to compile.

**Work (must land as one commit, or one tight chain that ends green):**
1. Across all 17 `packages.config` files: pin `DotNetWorkQueue`, `DotNetWorkQueue.Transport.Shared`, `.RelationalDatabase`, `.SqlServer`, `.PostgreSQL`, `.SQLite`, `.LiteDb`, `.Redis` at exactly `0.9.18` — R6.
2. Remove all `DotNetWorkQueue.AppMetrics` and `App.Metrics.*` entries from every `packages.config`, and any matching `<Reference>`/`<HintPath>` from every `.csproj` — R8.
3. Using the Phase 1 R2 file list, strip `using App.Metrics`, `IMetrics`/`IMetricsRoot` parameters, and the entire metric-registration code path from every example file. Producer/consumer wiring stays intact — R9.
4. Remove the standalone Polly v7 reference (`Polly.Caching.Memory` 3.0.2, `Polly.Contrib.Simmy` 0.3.0, `Polly.Contrib.WaitAndRetry` 1.1.1) flagged in CONCERNS.md if it is no longer transitively required — R7. Let NuGet redo the transitive graph on restore.
5. `nuget restore` + `msbuild ... /p:Configuration=Release`. Iterate compile errors. If absorbing breakage exceeds ~half a day, **revert and re-open scope** (NFR Reversibility + Constraint:Timeline).

**Success criterion:** Zero references to `DotNetWorkQueue.AppMetrics` or `App.Metrics.*` in any tracked file. Every `DotNetWorkQueue.*` package at `0.9.18`. Solution builds clean in Release with warning count ≤ Phase 1 baseline. Maps to Success Criteria #2 + #3.

**What could go wrong:** (a) A 0.8.0 → 0.9.18 API break exists outside metrics — if so, stop and re-open scope per R11. (b) A transport lib genuinely needs `App.Metrics.Abstractions` transitively in 0.9.18 — unlikely given upstream removed it in 0.9.1, but verify before stripping the package. (c) New build warnings appear from the bumped library; the baseline gate says they must be justified or fixed.

---

## Phase 4 — Code & Repo Hygiene

- **Depends on:** Phase 3
- **Delivers:** R10, R12, R13, R14, R15 (R11 is absorbed inside Phase 3 if it triggered)
- **Effort:** S (1–3h)
- **Risk:** LOW — these edits do not interact with the build graph.

**Work:**
1. Fix `Source/Examples/LiteDb/Consumer/LiteDbConsumer/Commands/ConsumeMessage.cs`: rename declared namespace `SQLiteConsumer.Commands` → `LiteDbConsumer.Commands`. Update any `using` statements that follow — R10.
2. Bump `SharedAssemblyInfo.cs` `AssemblyVersion`/`AssemblyFileVersion`/`AssemblyInformationalVersion` from `0.7.1` → `0.9.18` — R12.
3. Delete orphaned RPC build artifacts under `Source/Examples/{SqlServer,PostGresSQL,SQLite,Redis}/Rpc/**/{bin,obj}` — R13.
4. `git rm -r --cached` every committed `bin/` and `obj/` directory in the repo. Verify `.gitignore` already covers `[Bb]in/` and `[Oo]bj/` (CONCERNS.md confirms it does); add rules only if a gap is found — R14.
5. Correct `App.config` connection strings: `192.168.0.58` → `192.168.0.2` in all 6 SQL Server + PostgreSQL producer/consumer projects — R15. Leave the existing Redis `192.168.0.2` as-is.
6. `nuget restore` + `msbuild ... /p:Configuration=Release` to confirm hygiene didn't disturb anything.

**Success criterion:** Solution still builds clean in Release. No `bin/` or `obj/` directories tracked in git (`git ls-files | grep -E '/(bin|obj)/'` returns empty). LiteDb namespace fix verified. SharedAssemblyInfo at 0.9.18. Maps to Success Criteria #7, #8, #9.

---

## Phase 5 — CI Workflow

- **Depends on:** Phase 4 (code must be stable so the first CI run is meaningful)
- **Delivers:** R-CI-1, R-CI-2, R-CI-3
- **Effort:** XS (<1h) for the file; verification depends on push timing
- **Risk:** LOW — the workflow is small and uses well-known marketplace actions. The only failure mode is a runner-environment mismatch with the local build, which is exactly what we want to discover before freezing.

**Work:**
1. Create `.github/workflows/ci.yml`:
   - Triggers: `push` to `main`, `pull_request` to `main` (mirror trigger shape from `DotNetWorkQueue.Samples/.github/workflows/ci.yml` — R-CI-2).
   - Single job, `runs-on: windows-latest`.
   - Steps: `actions/checkout@v4` → `microsoft/setup-msbuild@v2` → `NuGet/setup-nuget@v2` → `nuget restore Source/Examples/Examples.sln` → `msbuild Source/Examples/Examples.sln /p:Configuration=Release`.
   - **No `dotnet build` step**, no test step (R-CI-1 + non-goal "MSBuild.exe + nuget.exe only").
2. Commit on a branch, push, open a PR to verify the workflow runs green on PR trigger before merging.

**Success criterion:** First workflow run on the freeze commit lands green on `main`. Maps to Success Criteria #4 and R-CI-3.

---

## Phase 6 — Documentation

- **Depends on:** Phase 5 (docs reflect the actual final state, including the CI badge if added)
- **Delivers:** R16, R17
- **Effort:** XS (<1h)
- **Risk:** LOW

**Work:**
1. Rewrite the top of `README.md`: frozen banner naming **DotNetWorkQueue 0.9.18 + .NET Framework 4.8**, one-paragraph reason (upstream moved to net10/net8 in 0.9.19), link to `https://github.com/blehnen/DotNetWorkQueue.Samples` for current usage and metrics. Preserve the existing transport bullet list and license section below — R16.
2. Update `CLAUDE.md`: TFM `4.7.2` → `4.8`, add "pinned at DotNetWorkQueue 0.9.18". Leave everything else untouched — R17.

**Success criterion:** `README.md` opens with the frozen banner and the link to the Samples repo. `CLAUDE.md` shows `4.8` and the 0.9.18 pin. Maps to Success Criterion #10.

---

## Phase 7 — Verification & Freeze Commit

- **Depends on:** Phase 6
- **Delivers:** R18, R19, R20
- **Effort:** S (1–3h), dominated by manual smoke tests
- **Risk:** **MEDIUM** — R19 is the hard manual gate and the only end-to-end runtime check the project receives. R20 depends on remote service availability that is outside this project's control.

**Why MEDIUM:** R19 is **mandatory** and blocks the freeze. If the SQLite/LiteDb round-trip fails, the freeze does not ship and you re-open Phase 3. R20 is best-effort but discovering a transport that cannot connect to `192.168.0.2` after the IP correction in Phase 4 still requires diagnosis to know whether it's a service-down problem or a code-level regression.

**Work:**
1. **R18:** Final `nuget restore` + `msbuild ... /p:Configuration=Release` from a clean checkout. Confirm zero errors and warning count ≤ Phase 1 baseline. Hard gate.
2. **R19 (mandatory):** Launch `LiteDbProducer` (or `SQLiteProducer`); create a queue, send a message. Launch the matching consumer; receive the message. Capture the shell-log transcript or a screenshot into the verification notes. This is a hard gate.
3. **R20 (best-effort):** Attempt producer ↔ consumer round-trip on SQL Server, PostgreSQL, and Redis against `192.168.0.2`. For each transport, record either "PASS" or a one-line "why we shipped anyway" note (e.g., "service unreachable, freeze ships per R20 best-effort clause").
4. Land the freeze commit on `main`. Push. Confirm CI workflow from Phase 5 lands green on the freeze commit (R-CI-3).

**Success criterion:** All 10 PROJECT.md success criteria simultaneously true. R19 transcript captured. R20 results documented per transport. CI green on the freeze commit.

---

## Total Estimated Effort

| Phase | Tier | Hours (mid) |
|-------|------|-------------|
| 1. Spike | XS | 0.5 |
| 2. TFM bump | S | 2 |
| 3. Package bump + metrics strip | M | 4.5 |
| 4. Hygiene | S | 2 |
| 5. CI | XS | 0.5 |
| 6. Docs | XS | 0.5 |
| 7. Verification | S | 2 |
| **Total** | — | **~12h** |

This is a long single session or a comfortable two-session split. The natural break point is between Phase 3 (riskiest, ends green) and Phase 4 (cosmetic).

## Risk Summary

| Phase | Risk | Why |
|-------|------|-----|
| 3. Package bump + metrics strip | **MEDIUM** | DNWQ 0.8.0 → 0.9.18 crosses 10 minor releases; AppMetrics removal coupled to code edits; non-metrics breakage would re-open scope. |
| 7. Verification & freeze commit | **MEDIUM** | R19 is a hard manual gate with no fallback; R20 depends on external service availability. |
| All others | LOW | Mechanical edits or low-surface workflow file. |

## Rationale

The proposed phase shape in the prompt is correct and is followed verbatim. One small clarification: R11 (absorb non-metrics 0.8 → 0.9.18 breakage) is folded into Phase 3 rather than getting its own slot, because in practice it can only be discovered while the package bump is in flight — there is no separate edit pass for it. If R11 turns out to be substantial, the NFR Reversibility + Constraint:Timeline rules say revert Phase 3 and re-open scope rather than push through, so R11 cannot quietly grow into its own phase.

Baseline warning capture is placed at the end of Phase 1 (not "the start of Phase 2" as the prompt outlined) so the spike phase owns the immutable ceiling end-to-end. This is a wording-only difference — the capture still happens before any edit lands.
