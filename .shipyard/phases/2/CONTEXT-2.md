# Phase 2 Context — TFM Bump to net48

Decisions captured during `/shipyard:plan 2` discussion before researcher/architect dispatch.

## Decision: No worktree, no feature branch — work directly on `main`

User selected "Work on main directly" rather than isolating via worktree or branch. Implications for the plan:

- The TFM bump must land as a **single atomic commit** on `main`. No intermediate commits that leave `main` in a half-bumped state (e.g., csproj at net48 but packages.config still at net472).
- If the bump breaks the build, the roll-back mechanism is `git revert` or `git reset --mixed post-build-phase-1` (the checkpoint tag set after Phase 1). Both are cheap because Phase 1 only added `.gitattributes` + markdown.
- The builder must verify the solution still builds clean (0 warnings, 0 errors — matching the Phase 1 baseline) **before** creating the commit, not after. Committing a broken build to `main` is the failure mode to avoid.
- No merge-to-main step is needed; the work is already on `main` once committed.

## Decision: mass-edit via sed/perl, not per-file hand edits

52 files need the same mechanical substitutions. Per-file `Edit` tool calls would be 52+ round-trips with no upside. The plan should use `sed -i` (GNU sed on WSL supports in-place) or equivalent shell-level transforms:

- **csproj**: replace `<TargetFrameworkVersion>v4.5.2</TargetFrameworkVersion>` and `<TargetFrameworkVersion>v4.7.2</TargetFrameworkVersion>` with `<TargetFrameworkVersion>v4.8</TargetFrameworkVersion>`. The shared project `ConsoleSharedCommands` is at net472; the other 4 shared projects (`ConsoleShared`, `ConsoleView`, `ExampleMessage`, `ShellControlV2`) are at net452; the 15 runnable projects are at net472. After the bump all 20 csproj declare v4.8.
- **App.config**: replace `sku=".NETFramework,Version=v4.5.2"` and `sku=".NETFramework,Version=v4.7.2"` with `sku=".NETFramework,Version=v4.8"` in every `<supportedRuntime>` element.
- **packages.config**: replace `targetFramework="net452"` and `targetFramework="net472"` with `targetFramework="net48"` on every `<package>` element.

The builder should run one `git diff --stat` after the transforms and before the commit to sanity-check that the number of files touched matches expectations (20 csproj + 17 packages.config + 15 App.config = up to 52 files — some App.configs may not contain a `<supportedRuntime>` element and will show no change).

## Decision: HintPath entries left alone in Phase 2

`<HintPath>` entries in csproj files point at specific package lib subfolders like `..\..\..\packages\App.Metrics.4.3.0\lib\net461\App.Metrics.dll`. These are literal paths, not wildcards — they do NOT auto-upgrade when the project's TFM changes.

**Analysis:** .NET Framework 4.8 is binary-compatible with net472, net471, net461, net452, and older. A `lib\net461\Foo.dll` reference will continue to load at runtime under a net48 target. Phase 2 can safely leave HintPaths untouched. The only concern would be a package that ships ONLY a `lib\net48\` folder — but Phase 2 is not bumping package versions, so the package set stays exactly what it is today (DNWQ 0.8.0, AppMetrics 4.3.0, etc.), and their current HintPaths already work.

**HintPath cleanup deferred to Phase 3.** When Phase 3 bumps packages to 0.9.18, the HintPaths will need to regenerate regardless — that is the correct time to address them. Phase 2 explicitly does NOT touch them.

## Decision: `SharedAssemblyInfo.cs` is not touched in Phase 2

`SharedAssemblyInfo.cs` at repo root is linked into every project. It contains `AssemblyVersion("0.7.1")` (per the concerns map's version-drift finding). The version bump is deferred to Phase 4 (R12 in PROJECT.md). Phase 2 is strictly TFM-only.

## Decision: pre-commit build verification is mandatory

Before the atomic TFM-bump commit lands, the builder must:

1. Apply all substitutions (csproj + App.config + packages.config).
2. Run `nuget restore` equivalent via `msbuild /t:Restore` on the full solution using the `wslpath -w`-translated Windows path established in Phase 1.
3. Run `msbuild /t:Build /p:Configuration=Release` and parse the summary for warning/error counts.
4. Verify:
   - Build exits 0 (no errors).
   - Warning count ≤ Phase 1 baseline of 0. (Any new warning is a failure.)
5. **If any check fails, the builder STOPS, runs `git reset --hard HEAD` to discard the uncommitted substitutions, and escalates.** Nothing is committed unless all checks pass.

This pre-commit gate is the core safety mechanism given the "work on main directly" choice.

## Decision: no packages/ cache purge in Phase 2

The `Source/Examples/packages/` directory contains cached NuGet binaries. In Phase 2, it stays exactly as it is. `msbuild /t:Restore` with an unchanged package list is a no-op against the cache; only Phase 3's package bump will change what lives there.

## Decision: plan shape — one plan, one wave, at most 3 tasks

The ROADMAP Phase 2 work is R3, R4, R5 — three mechanical substitutions across three file types, plus verification. One plan file (`PLAN-2.1.md`) with three tasks:

1. Apply all substitutions + stage + pre-commit build verification gate
2. (Absorbed into Task 1 commit-atomicity requirement.)
3. (Absorbed into Task 1.)

Actually — since the three substitutions must land in a single atomic commit, they are effectively one task. The "3 tasks" Shipyard cap suggests splitting into:

1. Apply csproj TFM substitutions (staged but NOT committed)
2. Apply App.config + packages.config TFM substitutions (staged alongside task 1 but NOT committed)
3. Run restore + build verification gate, then atomic commit if green

This split gives the plan three auditable checkpoints while still landing a single commit. Architect should confirm.

## Decision: deliverables beyond the code commit

The plan should also produce:

- A SUMMARY-2.1.md documenting the file counts touched, the post-bump build output (warning/error counts, baseline comparison), and any surprises.
- An updated SPIKE-NOTES.md reference or a short "Phase 2 notes" file capturing the actual warning delta (expected: 0, same as Phase 1 baseline).
- No changes to ROADMAP.md or PROJECT.md in this phase. Those get status updates in the build step, not the plan step.
