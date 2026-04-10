# Phase 3 Context — Package Bump + AppMetrics Strip

Decisions captured during `/shipyard:plan 3` discussion before researcher/architect dispatch. This is the highest-risk phase in the freeze (MEDIUM risk, M effort). Context rules here are load-bearing.

## Decision: Non-metrics breakage fallback — absorb small breaks in-phase

The user chose **Option A (Recommended)** from the discussion question. Concrete policy the builder must follow:

### Absorb in-phase ("small" break)

The builder fixes and continues WITHOUT returning to planning if the break is mechanical:

- **Method rename** — e.g., `CreateQueue` → `ProvisionQueue`. Mechanical find/replace across example code.
- **Parameter reorder or addition with a sensible default** — swap arguments, add a `null` or default value.
- **Namespace move** — update `using` directives, no call-site changes.
- **Type alias or small enum rename** — e.g., `ConsumerQueueTypes` → `QueueVariant`. Mechanical.
- **Return type change** when the example discards the return value.
- **Property rename** — e.g., `Options.MaxRetries` → `Options.RetryCount`.

**Budget ceiling for absorption:**
- **≤10 example files touched** by absorption fixes (beyond the expected AppMetrics strip).
- **≤30 minutes of absorption work** — rough wall-clock estimate from the builder's perspective.
- **Zero changes** to the shared-base class hierarchy (`SharedSendCommands`, `SharedConsumeMessage<T>`) beyond the exact parameter/method rename that the break forces. No refactoring, no restructuring.

### Hard rollback ("structural" break)

The builder does `git reset --hard HEAD` (or the staged-rollback sequence from Phase 2) and escalates WITHOUT committing if:

- Signature change affects the `SharedSendCommands` / `SharedConsumeMessage<T>` base contract in a way that requires real refactoring (e.g., a base method is split into two, or a generic constraint changes).
- Removal of a type that has no direct replacement and would require re-architecting the example.
- Generic parameter changes affecting the `TInit` / `SqlServerMessageQueueInit` pattern that ripples through every transport.
- DI container registration API changes affecting `QueueContainer<TInit>` construction.
- Any break requiring **>10 file edits** OR **>30 minutes** of absorption work.
- Any compilation error the builder cannot resolve after one honest absorption attempt.

On rollback, the builder writes a SUMMARY-3.1.md with status `failed`, lists the observed break, and stops. The plan is then re-opened for discussion.

## Decision: Work on `main` directly (unchanged from Phase 2)

Same rationale as Phase 2: the pre-commit build verification gate is the safety mechanism, not a feature branch. If the build fails, the rollback sequence restores `main` cleanly. No worktree.

## Decision: AppMetrics strip + package bump land as ONE atomic commit

Package bump and code strip are tightly coupled:
- Bumping DNWQ 0.8.0 → 0.9.18 without removing AppMetrics code → build fails (the `DotNetWorkQueue.AppMetrics` package no longer exists as of 0.9.1; linker can't resolve the types).
- Removing AppMetrics code without bumping packages → build succeeds but leaves unused package references and stale HintPaths.

Neither state should ever be committed. The commit lands ONLY after all of package bump + AppMetrics strip + clean build happen together. Same pattern as Phase 2's Task 3 gate.

## Decision: Purge `Source/Examples/packages/` cache before restore

`Source/Examples/packages/` is a NuGet v2 (classic csproj) local package cache. It's gitignored. Purging it before the Phase 3 `msbuild /t:Restore` is the cleanest way to ensure:
- No stale 0.8.0 package folders leftover alongside 0.9.18 folders.
- No stale AppMetrics.4.3.0 folders that might still match lingering HintPaths.
- A fully fresh dependency graph for the verification build.

**Concrete:** `rm -rf Source/Examples/packages/` before Task 3's MSBuild restore call. This is safe — `.gitignore` excludes the folder, no history is lost, and restore regenerates it.

## Decision: csproj edit approach — use a one-shot script, not Edit-per-file or sed-per-file

Classic csproj `<Reference>` removal is multi-line XML surgery. Per-file Edit tool calls would require 17 files × ~5 `<Reference>` blocks each ≈ 85+ tool calls just to delete AppMetrics references, plus more for HintPath version substitutions. That hits the builder turn budget easily.

The builder should write **one shell/Python/PowerShell script** that performs the transforms once and is then run:

1. **AppMetrics removal phase** (multi-line XML):
   - For each csproj: read the file, delete every `<Reference Include="App.Metrics..." ...>...</Reference>` block (could be self-closing or multi-line), write back.
   - For each packages.config: delete every `<package id="App.Metrics..." .../>` line (single-line, easy sed).
   - For each App.config: delete every `<dependentAssembly>...<assemblyIdentity name="App.Metrics..."/>...</dependentAssembly>` block (multi-line XML surgery, optional — binding redirects are runtime-only and won't block the compile).

2. **DNWQ version bump** (literal string substitution):
   - In each csproj: replace `packages\DotNetWorkQueue.0.8.0\lib\net48\` → `packages\DotNetWorkQueue.0.9.18\lib\net48\` (and every transport package). Simple sed works for this.
   - In each csproj `<Reference Include>` assembly identity string: replace `Version=0.8.0.0` → `Version=0.9.18.0` for DNWQ packages. Regex sed.
   - In each packages.config: replace `version="0.8.0"` → `version="0.9.18"` on DNWQ package entries (careful not to touch unrelated packages' versions).

3. **AppMetrics .cs strip** (code-level):
   - 8 files from Phase 1 R2 inventory. The architect should NOT assume the shape — the builder reads each file and removes: `using App.Metrics;`, `using App.Metrics.*;`, any `IMetrics` / `IMetricsRoot` constructor parameters, any DI registration of AppMetrics, and any `metrics.*()` call sites that reference the stripped parameters.
   - This part may warrant per-file Edit tool calls because the shape varies and is not known upfront. The builder should inspect one file first to understand the pattern, then propagate.

**The script approach reduces the risk of turn-budget exhaustion** the previous builders hit. The architect should call out in the plan: "Write the transform script first, then run it, then verify." Rather than: "Edit file 1, edit file 2, ..."

## Decision: transform script language preference — Python

Python is:
- Installed on WSL (standard).
- Has `xml.etree.ElementTree` for robust XML parsing of csproj.
- Easier to audit than a sprawling sed pipeline.
- Produces clear error messages when a transform fails.

The builder should write the transform script to a tempfile (e.g. `/tmp/phase3/transform.py`), run it against the repo, and keep the script as an artifact in `/tmp/phase3/` so the reviewer can inspect it. Do NOT commit the script to the repo.

## Decision: keep the transitive graph stable except for AppMetrics removal

- Do **not** add or remove any direct NuGet dependency beyond AppMetrics.
- Do **not** bump Polly, Serilog, Newtonsoft.Json, or any non-DNWQ package version manually.
- After the restore, the new DNWQ 0.9.18 package will bring in an updated transitive graph (Polly 8.x, Microsoft.Extensions.* 10.x, OpenTelemetry, SimpleInjector 5.x). Those transitive dependencies are resolved by NuGet — the builder does NOT manually touch them in packages.config.
- If the concerns-mapper-flagged "standalone Polly v7" reference is no longer transitively needed, remove it. If it's still needed, leave it.

## Decision: post-restore, verify no orphaned or duplicate packages

After `msbuild /t:Restore`, run a sanity check:

```bash
ls Source/Examples/packages/ | grep -i 'AppMetrics\|DotNetWorkQueue\.0\.8\.0'
```

This should return empty (no AppMetrics folders, no DNWQ 0.8.0 folder). If either appears, the transform missed something — the builder rolls back and investigates.

## Decision: plan shape — 3 tasks max per Shipyard rule, single plan, single wave

Phase 3 has multiple distinct work types (packages.config sub, csproj sub, .cs strip, verify+commit). The Shipyard rule is ≤3 tasks per plan. The work naturally splits as:

1. **Task 1: Write and run the XML/packages.config transform script.** Modifies csproj + packages.config + App.config (binding redirects optional). Stages changes only.
2. **Task 2: Strip AppMetrics code from 8 .cs files.** The builder inspects the first file to understand the pattern, then applies the same transform to the remaining 7. Stages changes only.
3. **Task 3: Purge packages/ cache, restore, build, verify, atomic commit.** This is the gate.

Architect may adjust; this is guidance.

## Decision: deliverables beyond the code commit

- `.shipyard/phases/3/results/SUMMARY-3.1.md` documenting file counts, build outcome, any absorbed breaking changes, and final warning count.
- If absorption happens, the summary's "Decisions Made" section must list every small break absorbed, the file paths touched, and the reasoning.
- The transform script at `/tmp/phase3/transform.py` is kept as a build artifact (not committed) for audit purposes.
- No changes to ROADMAP.md or PROJECT.md in this phase — those are updated during the build-complete step.
