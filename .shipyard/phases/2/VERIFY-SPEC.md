# Phase 2 Plan — Spec Compliance Review

## Verdict
**PASS** — The plan fully delivers all requirements and conforms to the Shipyard specification.

## Coverage Check

| Req | Criterion | Status | Notes |
|-----|-----------|--------|-------|
| R3 | All 20 csproj files target v4.8 | PASS | Task 1 applies sed to replace v4.7.2→v4.8 (16 files) and v4.5.2→v4.8 (4 files). Verify block confirms 20 files match v4.8 pattern post-edit. |
| R4 | All 15 App.config + all 17 packages.config updated | PASS | Task 2 applies sed for App.config sku and packages.config targetFramework. Verify counts: 15 App.config + 17 packages.config = 32 files. Expect 938 total package entries at net48 (937 from net472 + 1 from net452). |
| R5 | Audit/remove non-net48 HintPaths | DEFERRED | Explicitly called out in plan context (line 24): "HintPath entries left alone in Phase 2" and CONTEXT-2.md justifies deferral: "net48 is binary-compatible with net472/net461/net452; existing HintPaths still resolve at runtime." Phase 3 will clean them when packages bump. Rationale documented. Acceptable per spec. |

## Task Count
**PASS** — 3 tasks, within ≤3 limit. Task division is proper: (1) csproj edits staged, (2) App.config+packages.config edits staged, (3) verify+commit.

## Atomic Commit Invariant
**PASS** — Plan explicitly enforces single atomic commit:
- Task 1 (line 66-68): "Do NOT commit yet — the commit happens at the end of Task 3..."
- Task 2 (line 119): "Do NOT commit yet."
- Task 3 (line 157): "Produces a commit" — commit only occurs here, after successful build verification.
- Rollback on failure (lines 224-226): `git reset HEAD` + `git checkout --` restores pre-edit state.

## Pre-Commit Build Verification
**PASS** — Mandatory gate fully specified in Task 3 (lines 172–222):
- Restore via `/t:Restore` (lines 176–187); exit code checked; rollback on failure.
- Build via `/t:Build /p:Configuration=Release /v:normal` (lines 190–201); exit code checked; rollback on failure.
- Warning/error parsing (lines 204–210): extracts count from MSBuild summary.
- Baseline check (lines 213–218): hard stop if `ERR_COUNT ≠ 0` or `WARN_COUNT > 0` (Phase 1 baseline = 0/0); rollback on violation.
- Commit only after all checks pass (line 221).

## Testable Acceptance Criteria
**PASS** — Each task's `<done>` block is runnable:
- **Task 1:** grep patterns for stale TFMs; wc to count 20 files at v4.8; git diff --cached to count 20 csproj staged. All concrete.
- **Task 2:** grep for stale sku/targetFramework values; wc to count 15 App.config + 938 package lines; git diff --cached --name-only to count 52 total files. All concrete.
- **Task 3:** exit codes from restore/build; grep for baseline counts; git log to confirm commit exists; git status to verify clean tree; post-commit grep sweep for any remaining stale tokens. All concrete.

## Scope Discipline
**PASS** — Plan does NOT include:
- SharedAssemblyInfo.cs version bump (deferred to Phase 4, per line 66 "Do NOT touch SharedAssemblyInfo.cs" and CONTEXT-2.md lines 32–34).
- HintPath cleanup (deferred to Phase 3, per context section and CONTEXT-2.md lines 24–30).
- Package version bumps (excluded from Phase 2 scope; RESEARCH.md line 227 notes ConsoleView packages.config stays at current versions).
- Namespace fixes (Phase 4 task).
- README edits (Phase 6 task).
- CI workflow (Phase 5 task).

## Sed Spot-Check
**PASS** — All sed commands are correct:

1. **csproj commands (Task 1, lines 57–61):**
   - Pattern: `'s|<TargetFrameworkVersion>v4\.7\.2</TargetFrameworkVersion>|<TargetFrameworkVersion>v4.8</TargetFrameworkVersion>|g'`
   - Escaping: `\.` correctly escapes literal dots in XML tag names and version strings. `|` delimiter avoids slash-in-XML issues. Correct.
   - Handles both v4.7.2 (16 files) and v4.5.2 (4 files) via separate sed invocations.

2. **App.config commands (Task 2, lines 103–107):**
   - Pattern: `'s|sku="\.NETFramework,Version=v4\.7\.2"|sku=".NETFramework,Version=v4.8"|g'`
   - Escaping: `\.` in source patterns; literal dot in replacement. Correct.
   - Uses `find ... -iname 'App.config'` for case-insensitive matching (line 103). Good defensive coding, though RESEARCH.md shows all 15 files are lowercase `App.config`.

3. **packages.config commands (Task 2, lines 111–114):**
   - Pattern: `'s|targetFramework="net472"|targetFramework="net48"|g'`
   - No dots; no escaping needed. Handles both net472 (937 entries) and net452 (1 entry). Correct.

All commands use `find ... -exec ... {} +` batching (efficient) and `-i` for in-place editing on CRLF files. RESEARCH.md (lines 165–170) confirms GNU sed on WSL handles CRLF correctly.

## Verification Commands Accuracy
**PASS** — Task 3 verification block includes:
- Pre-commit snapshot of staged changes (lines 168–170).
- Build exit-code capture (lines 178, 193).
- MSBuild summary grep with fallback for missing lines (lines 209–210, defensive with `${VAR:-1}` defaults).
- Baseline comparison (line 213): explicit check `WARN_COUNT -gt 0` triggers rollback.
- Post-commit confirmation (lines 237–245): grep sweep for any remaining stale TFM/targetFramework tokens.

## Notes
- No revisions required. Plan fully meets Shipyard spec compliance.
- CONTEXT-2.md and RESEARCH.md are correctly integrated; plan references them and follows their guidance exactly.
- The 3-task split preserves auditability via per-type `git diff --stat` checkpoints while landing a single atomic commit — matches CONTEXT-2.md decision rationale (lines 54–67).
- Pre-commit build verification is the critical safety gate; it is robust and will catch any breaking changes before main is touched.
